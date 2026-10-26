# 技术复盘 · 索引

> 这里放的是我在实习期间做**存量 Java 系统改造**时，踩过并解决的几个问题。
> 每篇都是「现象 → 排查路径 → 根因 → 修法 → 可复用的经验」，代码片段已脱敏。
> 内容偏向"真实故障复盘"，不是教程搬运。

## 文章列表

| # | 标题 | 关键词 | 一句话 |
|---|---|---|---|
| 1 | [JPA 延迟落库 + 原生 SQL：一次"没有报错"的数据丢失](./01-jpa-flush-ordering.md) | `FlushMode.AUTO`、原生 SQL、静默失败 | UPDATE 影响 0 行却不报错，问题不在 SQL |
| 2 | [自监控系统的反馈环：一个被 catch 吞掉的 ConcurrentModificationException](./02-self-monitoring-feedback-loop.md) | AOP、Druid Filter、ThreadLocal | 日志上报自己又被采集，循环中断导致日志静默少记 |
| 3 | [互相加锁导致的 ABBA 死锁：为什么"重试"不如"排序"](./03-lock-ordering-deadlock.md) | 死锁、固定顺序加锁、幂等 | 打破循环等待，从根上消除而不是靠超时重试 |

## 背景说明

- 场景均来自**运营商级存量系统**（Spring Boot 1.5 / 2.6、Java 8、MyBatis/JPA 混用），不是新搭的 Demo 项目；
- 文中涉及的业务字段名、表名、IP、密钥均已脱敏或改写；
- 每篇末尾附「发布前检查」，确保不含敏感的客户与生产信息。
