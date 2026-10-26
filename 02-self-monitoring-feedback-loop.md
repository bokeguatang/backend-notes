# 自监控系统的反馈环：一个被 catch 吞掉的 ConcurrentModificationException

> 关键词：AOP 切面、Druid Filter、ThreadLocal、`ConcurrentModificationException`、自采集反馈环
> 场景：给后台写操作接入"后端 SQL 审计日志"

---

## 一、需求背景

合规要求：后台的**主动写操作**（新增/修改/删除）必须留全量审计日志，且要上报到统一的日志平台。

审计粒度要求到 **SQL 级别**——不只是"谁在什么时候点了什么菜单"，而是"这次操作实际执行了哪些 SQL、影响了几行、耗时多少"。

## 二、设计思路

整体方案是 AOP + 连接池采集 + 线程上下文：

```
浏览器请求
  → @ActionLog 注解的 Controller 方法（如 活动库存调整）
  → 切面 @Around 进入
       ├─ 生成 traceId
       └─ 初始化上下文：ThreadLocal（trace_id + SQL 列表）
  → 业务方法执行，SQL 经过连接池
       └─ 采集器检查当前线程有没有上下文
            有 → 采集 SQL / 参数 / 影响行数 / 耗时 → 追加进上下文
  → 切面收尾（正常返回 & 异常都要）
       ├─ 逐条组装审计报文
       ├─ 上报：落库审计表 + 异步发到日志平台的 Kafka topic
       └─ 清理 ThreadLocal
```

采集器的挂载点选在**连接池**（Druid 的 `FilterAdapter`），好处是 **JPA 和 JdbcTemplate 的 SQL 都能覆盖**，不用在每个 DAO 上打点。

采集代码的核心逻辑（示意）：

```java
private void capture(String sql, long rows, long start, boolean ok, String err) {
    OperateLogContext ctx = OperateLogContext.current();
    if (ctx == null) return;            // 非被监控请求，直接跳过（平时零开销）
    ctx.addSql(new SqlRecord(sql, rows, System.currentTimeMillis() - start, ok, err));
}
```

## 三、现象：每次操作只上报了 1 条 SQL

自测时发现一个诡异的现象：

| 预期 | 实际 |
|---|---|
| 一次操作产生 N 条 SQL → 上报 N 条审计日志 | **只上报了 1 条** |
| 上报的应该包含 UPDATE | 上报的往往是**最先执行的那条 SELECT** |
| 无异常 | 日志里每次都有一条 `上报失败: null` |

**三条线索合起来**指向一个方向：

1. 只有第一条 → 循环在第一次迭代后就中断了；
2. `null` 的错误信息 → 异常被 `catch` 住，但**堆栈被吞了**（只记了 message，而 CME 的 message 通常是 null）；
3. 业务功能完全正常，接口不报错 → 问题只影响日志，不影响主流程。

## 四、排查路径

### 第一步：把吞掉的异常打出来

原来是 `log.error("上报失败: " + e.getMessage())`，`ConcurrentModificationException` 的 message 正是 `null`。

改成打印完整堆栈：

```java
log.error("上报失败", e);   // 传异常对象，不要只传 getMessage()
```

立刻看到真相：

```
java.util.ConcurrentModificationException
    at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:909)
    at ...ActionLogAspect.reportBackendOperate(ActionLogAspect.java:xx)
```

### 第二步：谁修改了集合？

加日志统计遍历前后的 size，发现**遍历过程中集合在变大**。

看代码路径：

```
reportBackendOperate(ctx)
  └─ 遍历 ctx.getRecords()          ← 正在 for-each
       └─ 每条组装报文 → handler 上报
            └─ handler 落库：INSERT INTO 审计表 ...
                 └─ ★ 这条 INSERT 也走同一个连接池
                      └─ 采集器发现当前线程仍有上下文（ThreadLocal 还没清）
                           └─ ctx.addSql(这条 INSERT)  ← 往正在遍历的 List 里追加！
```

**根因确认：上报动作本身又触发了一次采集。**

## 五、根因分析：自采集的反馈环

设计时的假设是：

```
采集发生在【业务执行期间】，上报发生在【业务结束之后】—— 两个阶段天然分离
```

这个假设在**"被观测对象包含自己"**的时候就不成立了：

```
观测者（审计上报） → 也是写操作 → 也是被观测对象（采集器覆盖全连接池）
```

于是形成了一个环：**上报 → 写库 → 被采集 → 集合变更 → 遍历中断**。

时序上就是"读列表"和"写列表"发生在同一次遍历里：

```
for (SqlRecord r : ctx.getRecords())   // 迭代器快照了 modCount
        │
        └── 上报 → INSERT 审计表 → ctx.addSql()  // modCount++
                     ↓
        下一次 hasNext() → modCount 不一致 → 抛 CME
```

## 六、修复：两层，缺一不可

### 第一层：遍历用快照，切断"边遍历边追加"

```java
// 修复前
for (SqlRecord r : ctx.getRecords()) { ... }

// 修复后：先复制一份，后续 addSql 不再影响本次循环
for (SqlRecord r : new ArrayList<>(ctx.getRecords())) { ... }
```

`new ArrayList<>(list)` 复制的是引用列表，开销极小（一次操作几十条 SQL 而已）。

### 第二层：从源头排除"日志表自身"的 SQL

```java
private static final Set<String> LOG_TABLES = new HashSet<>(Arrays.asList(
        "TBL_OPERATE_LOG", "TBL_BACKEND_OPERATE_LOG",
        "TBL_BACKEND_FLOW_LOG", "TBL_SYS_ACTION_LOG"));

private void capture(String sql, ...) {
    OperateLogContext ctx = OperateLogContext.current();
    if (ctx == null) return;
    if (isWriteToLogTable(sql)) return;      // ★ 写日志表的 SQL 不采集
    ...
}
```

**为什么只做第一层不够**：不抛异常了，但**数据被污染**——你写日志的那条 INSERT 会被当成"业务 SQL"上报给日志平台。日志表被反复采集，数据量还会随日志增长自我膨胀。

## 七、验证

| 用例 | 修复前 | 修复后 |
|---|---|---|
| 删除操作（1 条 SELECT + 1 条 UPDATE） | 只上报 SELECT | **两条都上报** |
| 库存调整（多条 SQL） | 只上报第一条 | 全部上报 |
| 日志表自身写入 | 被采集上报 | **不再采集** |
| 错误日志 | 每次 `上报失败: null` | 无 |

校验方式：查审计表里该 `traceId` 下是否有 UPDATE 记录、影响行数是否为 1；同时确认日志里不再出现上报失败。

## 八、可复用的经验

1. **自采集系统必须先解决反馈环，再谈采集完整性。**
   监控/审计/埋点系统一旦覆盖了"自己的写入路径"，就会形成环。**观测者要主动把自己排除在被观测范围之外**——这和"日志框架不打印自己的日志"是同一个道理。
2. **不要把异常的 message 当日志。**
   `log.error(msg + e.getMessage())` 会丢掉堆栈，而像 CME 这类异常的 message 就是 `null`，等于什么都没记。**要么传异常对象，要么至少打 `getClass().getSimpleName()`。**
3. **`ConcurrentModificationException` 的典型指纹**：循环提前中断 + 处理结果"只有第一条" + 异常被吞。遇到"只有第一条生效"的现象，优先怀疑这个。
4. **`for-each` 遍历期间任何"可能修改集合"的调用都要警惕**——尤其是那种"看起来只是上报/记日志"的调用，因为它可能间接写库、又间接回到采集逻辑。
5. **采集器用"有没有上下文"做开关，本身是正确的设计**：非被监控请求直接 return，运行时开销接近 0。问题只在于**上下文的生命周期比"业务执行"长**——它一直持续到切面收尾，把上报动作也圈了进去。

```java
// 上下文的生命周期（修复后）
try {
    Object result = pjp.proceed();     // 业务执行：采集器工作
    reportBackendOperate(ctx);         // 上报：快照遍历 + 排除日志表
    return result;
} catch (Throwable t) {
    reportBackendOperate(ctx);         // 异常路径同样上报已采集的 SQL
    throw t;
} finally {
    OperateLogContext.clear();         // ★ 必须清理，避免线程复用串上下文
}
```

---

## 发布前检查

- [x] 表名、字段名已脱敏或改为通用命名
- [x] 未出现真实 IP、Kafka topic、账号密钥
- [x] 未出现客户名称与项目代号
- [x] 代码片段为示意性重写
