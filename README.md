# Branch Sync Skill

## Overview

Branch Sync 是一个 AI 辅助代码同步与功能迁移 Skill。

它解决了传统 Git Merge 难以处理的场景：

* 多平台分支同步（Develop → Fusion）
* 平台差异代码保留
* 指定提交范围同步
* 跨项目功能迁移
* Feign / Config / Service 等高风险变更控制
* 自动依赖影响分析
* 编译验证与回滚保护

相比简单的 Merge 或 Cherry-pick，Branch Sync 更关注：

> 理解业务变更意图，并将变更安全地迁移到目标分支或目标项目，而不是机械复制代码。

---

# Supported Scenarios

## 1. Cross-Branch Sync（项目内跨分支同步）

适用于：

* develop → fusion
* dev → release
* platformA → platformB

等共享代码基线但存在平台差异的分支。

### Typical Examples

* 将 develop 分支的新功能同步到 fusion 分支
* 将 Bug Fix 同步到客户定制分支
* 将部分提交迁移到生产维护分支
* 保留平台特有逻辑的同时同步业务代码

---

## 2. Cross-Project Migration（跨项目功能迁移）

适用于：

* 从旧项目迁移模块
* 从其他产品线迁移功能
* 复制成熟业务能力到新项目

### Typical Examples

* 功能迁移


---

# Core Capabilities

## Commit Range Locking

首先锁定同步范围：

```bash
git log --after=<time>
git log --before=<time>
```

只同步指定提交中的变更。

避免：

* 历史差异被误同步
* 长期分支漂移导致误判

---

## Difference Classification

所有差异会被分类：

| 类型                    | 处理方式 |
| --------------------- | ---- |
| Business Change       | 同步   |
| Target-only Code      | 保留   |
| Historical Difference | 忽略   |
| Conflict Area         | 用户确认 |

确保：

* 不覆盖平台特有代码
* 不误同步历史差异
* 不自动解决业务冲突

---

## Risk Assessment

根据文件类型自动评估风险：

| 类型         | 风险等级   |
| ---------- | ------ |
| DTO        | Low    |
| VO         | Low    |
| Enum       | Low    |
| Mapper     | Medium |
| Controller | Medium |
| Service    | Medium |
| Feign      | High   |
| Config     | High   |
| Scheduler  | High   |

高风险文件：

* 必须展示保护策略
* 必须获得用户确认

---

## Sync Planning

在修改代码前生成完整同步计划：

| File        | Operation | Risk   |
| ----------- | --------- | ------ |
| UserDTO     | New       | Low    |
| UserService | Merge     | Medium |
| DeviceFeign | Merge     | High   |

支持：

* New
* Merge
* Skip
* Delete

确保用户明确知道将发生什么。

---

## Dependency Impact Scan

自动分析：

* DTO字段变更
* 方法签名变更
* 枚举变更
* 接口变更

查找所有调用方：

```java
UserService.getUser(Long)
↓
UserController
UserFacade
UserServiceTest
```

提前发现影响范围。

---

## Test Impact Verification

自动修复：

* 单元测试
* Mock配置
* 构造器注入
* 参数变化
* 返回值断言

减少同步后的编译失败。

---

## Method-Level Verification

对关键方法进行逐行比对：

```java
calculate()
queryData()
exportReport()
```

确保：

* 核心逻辑一致
* 平台逻辑保留
* 无遗漏同步

---

## Sync Coverage Verification

统计：

| 项目        | 数量 |
| --------- | -- |
| Commit文件数 | N  |
| 已同步       | N  |
| 跳过        | N  |
| 失败        | N  |

目标：

```text
Coverage = 100%
```

避免遗漏文件。

---

## Rollback Protection

同步前自动创建：

```bash
git branch backup-sync-xxxx
```

或：

```bash
git tag sync-before-xxxx
```

出现问题可立即回滚：

```bash
git reset --hard sync-before-xxxx
```

---

# Execution Modes

## Quick Sync

适用于：

* ≤ 5 文件
* 简单变更
* 无配置修改

执行：

```text
Step1
Step2
Step3
Step5
Step8
```

特点：

* 快速
* 低成本
* 适合小型同步

---

## Standard Sync

默认模式。

适用于：

* 普通业务同步
* 中等规模变更

执行完整流程。

---

## Strict Sync

适用于：

* 核心业务模块
* Feign变更
* Config变更
* 大规模同步

额外执行：

* 风险评估
* 调用链分析
* 全量方法校验
* 覆盖率检查

确保最高可靠性。

---

# Workflow

## Cross-Branch Sync

```text
Lock Commit Range
        ↓
Risk Assessment
        ↓
Difference Analysis
        ↓
Sync Plan
        ↓
User Confirmation
        ↓
File-by-File Sync
        ↓
Dependency Scan
        ↓
Test Verification
        ↓
Rollback Point
        ↓
Compile
        ↓
Method Verification
        ↓
Coverage Check
        ↓
Sync Report
```

---

## Cross-Project Migration

```text
Source Analysis
        ↓
Dependency Matching
        ↓
Migration Plan
        ↓
User Confirmation
        ↓
File Migration
        ↓
Compile
        ↓
Verification
        ↓
Migration Report
```

---

# Output Artifacts

同步完成后自动生成：

```text
sync-report-YYYYMMDD-HHmmss.md
```

或：

```text
migration-report-YYYYMMDD-HHmmss.md
```

包含：

* 提交列表
* 文件清单
* 风险评估
* 依赖分析
* 编译结果
* 覆盖率统计
* 回滚信息
* 人工待办事项

---

# Benefits

相比传统 Git Merge：

✅ 保留平台差异代码

✅ 理解业务逻辑而非机械复制

✅ 风险分级控制

✅ 自动依赖分析

✅ 自动测试修复

✅ 编译验证

✅ 覆盖率检查

✅ 回滚保护

✅ 迁移报告归档

---

# Best For

* 多平台产品线
* SaaS 定制版本
* 长期维护分支
* 大型 Java 项目
* 微服务系统
* 企业级代码迁移

Branch Sync 的目标不是替代 Git，而是成为 Git Merge 与人工代码审查之间的智能中间层，让跨分支同步和跨项目迁移变得可控、可追踪、可回滚。
