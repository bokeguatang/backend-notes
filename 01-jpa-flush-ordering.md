# JPA 延迟落库 + 原生 SQL：一次"没有报错"的数据丢失

> 关键词：`FlushMode.AUTO`、原生 SQL、`UPDATE` 影响 0 行、静默失败
> 环境：Spring Boot 1.5 + JPA(Hibernate) + 外部 jar 提供的实体

---

## 一、需求背景

系统原有一个用户表，实体类 `SysUser` **来自外部依赖 jar**（公司公共用户中心），我们无法修改它。

接到的需求是：给账号增加两个业务字段——**账号生效时间**和**账号失效时间**（由上游统一认证平台下发），并在登录时校验。

```
需求：账号带 EFFECTDATE / EXPIREDATE，登录时判断是否已生效/已过期
约束：SysUser 实体在外部 jar 里，没有这两个字段 → JPA 无法映射
```

## 二、方案选择

因为实体改不了，JPA 侧走不通，只能用**原生 SQL** 直接操作数据库列：

```java
// ① 保存用户（走 JPA）
sysUserRepository.save(newUser);

// ② 用原生 SQL 补写实体不具备的两个字段
batchNativeSql(
    "update tbl_sys_user set ACCOUNT_EFFECT_DATE = ?, ACCOUNT_EXPIRE_DATE = ? where USER_ID = ?",
    effectDate, expireDate, userId);
```

看起来没什么问题，代码能编译、能跑通、日志里也没有异常。

## 三、现象：数据没写进去，但没有任何报错

上线自测时发现：**账号能创建成功，但这两个字段的值始终是 NULL。**

排查过程：

| 步骤 | 结果 |
|---|---|
| 看应用日志 | 无任何 ERROR / WARN |
| 看 SQL 日志 | UPDATE 语句确实打印了 |
| 看影响行数 | `executeUpdate()` 返回 **0** |
| 直接去数据库查 | 该 `USER_ID` 的记录**是存在的**（说明 save 成功），但两个字段为空 |

**关键矛盾**：记录明明存在，为什么 UPDATE 的影响行数是 0？

## 四、根因：flush 时机

问题出在 **JPA 的延迟落库**上。

```
sysUserRepository.save(newUser)   ← 只是把实体放进持久化上下文（Persistence Context）
                                    默认 FlushMode.AUTO：真正的 INSERT 会延迟到
                                    ① 事务提交前，或
                                    ② 下一次查询前，才执行
batchNativeSql("update ...")      ← 这是原生 SQL，直接走 JDBC 发给数据库
                                    ★ 它不触发 Hibernate 的 flush
```

所以执行顺序实际是：

```
1. save(newUser)          → 实体进内存，数据库里还没有这行
2. update ... where USER_ID = ? → 数据库里查不到这行 → 影响 0 行（不报错）
3. 事务提交               → INSERT 才真正执行，这时才插入记录（两个字段自然为空）
```

**为什么"没有报错"**：`UPDATE ... WHERE 条件不匹配` 是合法的 SQL 语义，不是错误。JDBC 返回 `0`，Hibernate 也不认为有问题。这就是典型的**静默失败**——代码正确、SQL 正确、事务正确，只是顺序错了。

## 五、修复

在原生 SQL 之前，主动触发一次 flush。最简单的方式是**执行一次查询**（Hibernate 在查询前会保证持久化上下文与数据库一致，从而 flush 未落库的改动）：

```java
// 保存用户
sysUserRepository.save(newUser);

// ★ 前置查询，强制触发 AUTO flush，让 INSERT 真正落库
findList("from SysUser a where a.id = ?", userId);

// 再执行原生 SQL 更新实体不具备的字段
batchNativeSql(
    "update tbl_sys_user set ACCOUNT_EFFECT_DATE = ?, ACCOUNT_EXPIRE_DATE = ? where USER_ID = ?",
    effectDate, expireDate, userId);
```

更显式的写法（语义更清楚，推荐）：

```java
entityManager.flush();   // 明确表达"我要把当前改动刷到数据库"
```

并把这段落库逻辑用 `try-catch` 隔离，失败只记日志，不影响主流程（账号本身要能正常创建）。

## 六、验证

| 用例 | 结果 |
|---|---|
| 新建账号，传入生效/失效时间 | 两个字段正确落库 |
| 更新账号（变更有效期） | 字段被更新 |
| 上游未下发时间字段 | 不覆盖原值（保留原状态） |
| 构造"未生效"账号登录 | 登录被拒，提示"账号未生效" |
| 构造"已过期"账号登录 | 登录被拒，提示"账号已过期" |

## 七、可复用的经验

1. **混用 JPA 与原生 SQL 时，必须清楚 flush 时机。**
   `save()` 不等于"已经写进数据库"，它只是"登记了一个待写入的改动"。
2. **`UPDATE` 影响 0 行且无异常时，先怀疑"数据是否真的已经落库"，而不是怀疑 SQL 本身。**
   这类问题的排查顺序建议是：
   ```
   影响行数 → 表里有这行吗 → 这行是什么时候写进去的 → 我的更新为什么没匹配上
   ```
3. **原生 SQL 是"旁路"**：它绕过持久化上下文，因此既看不到未 flush 的改动，也不会触发 flush。
4. **外部依赖的实体扩展，是存量系统的常见困境**：
   jar 里的实体改不了、不能加字段，就只能在"自己这一侧"找解法——原生 SQL、DTO 转换、或新增扩展表。这次我们选了最简单、影响面最小的方案。
5. **静默失败比抛异常更危险。** 抛异常至少能被发现；影响 0 行这种问题，只有**主动核对数据**才能发现。凡是"更新类"操作，都应该关注影响行数。

---

## 发布前检查

- [x] 已移除具体表名/字段名中的业务含义（`tbl_sys_user` 等为通用命名）
- [x] 未出现真实 IP、账号、密钥
- [x] 未出现客户名称与项目代号
- [x] 代码片段为示意性重写，非原文件复制
