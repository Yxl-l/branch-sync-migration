---
name: branch-sync
description: AI-assisted cross-branch code synchronization and cross-project code migration. Trigger whenever the user wants to sync code changes between two branches (e.g. develop-xx to fusion-xx), merge specific commits across branches, replicate business logic changes from one platform branch to another, or migrate a feature from another project. Also use when the user mentions "branch sync", "cross-branch", "sync to fusion/develop", "merge from branch", "migrate from project", "cross-project migration", "迁移", "port feature", or needs to move code between projects or branches while preserving platform-specific differences.
---

# Code Synchronization & Migration

Two modes of operation:

1. **Cross-Branch Sync** — sync changes between branches within the same project
2. **Cross-Project Migration** — migrate a feature from another project, fully automated

## Mode Selection

At the start, ask the user:

> 本次操作是哪种类型？
> 1. **项目内跨分支同步** — 同一个项目两个分支之间同步代码变更
> 2. **跨项目功能迁移** — 从另一个项目迁移某个功能到当前项目

Then proceed to the corresponding workflow below.

---

## Execution Mode

After determining the operation type, determine execution mode:

### Quick Sync (快速同步)

适用于：
- ≤ 3 commits
- ≤ 5 files
- 无核心配置变更

执行步骤：
Step1 → Step2 → Step3 → Step5 → Step8

跳过：
- Method-level verification
- Configuration consistency check
- Full test impact analysis

### Standard Sync (标准同步)

默认模式。执行全部标准流程。

### Strict Sync (严格同步)

适用于：
- 核心业务模块
- Feign/API 变更
- Service 核心逻辑变更
- Config 变更

额外执行：
- Risk Assessment — 评估每个变更的风险等级（高/中/低），高风险变更需用户二次确认
- Dependency Impact Scan — 扫描被变更方法的所有调用方，标记受影响范围
- Full Method Verification — 对所有变更方法逐行比对，而非抽查
- Sync Coverage Check — 生成覆盖率报告，列出所有源分支变更项的同步状态（已同步/已跳过/需人工确认）

### 自动选择规则

If the user does not specify, automatically choose:

- ≤ 5 files → Quick
- 6 ~ 20 files → Standard
- \> 20 files → Strict

After the mode is determined, inform the user:
> 本次同步将使用 **[模式名]** 模式，执行步骤：[步骤列表]。如需切换模式请告知。

---

# Mode 1: Cross-Branch Sync (项目内跨分支同步)

Sync business code changes from a source branch to a target branch that has platform-specific differences. The target branch must receive all business changes while preserving its own unique code.

## When to Use

- Two branches share the same codebase but have platform/environment differences
- Source branch has new commits that need to be replicated to target
- Target branch has code that must NOT be overwritten during sync
- User wants controlled, file-by-file synchronization, not a blind git merge

## Prerequisites

Before starting, confirm with the user:

1. **Source branch name** (e.g. `develop-xx`) — where the changes originated
2. **Target branch name** (e.g. `fusion-xx`) — where changes need to go
3. **Commit time range** on source branch — what's in scope for this sync
4. **Known branch differences** — what the target branch has that's unique and must be preserved

## Workflow

Execute these steps in order. Do not skip steps or combine them.

### Step 1: Lock the commit range on the source branch

Switch to the source branch and extract the diff for the specified time range.

```bash
git checkout <source-branch>
```

Identify commits in the time range:

```bash
git log --oneline --after="<start-time>" --before="<end-time>"
```

Get the unique list of changed files:

```bash
git diff --name-only <first-commit>^..<last-commit>
```

Present the user with a clean checklist:

```
Files to sync (N files):
1. path/to/FileA.java — brief description of what changed
2. path/to/FileB.java — brief description of what changed
...
```

**Why this matters**: Without a locked commit range, AI will confuse historical differences between branches with the changes that actually need syncing.

### Step 1.5: Risk Assessment

After generating the file list, classify risk level for each file based on its type:

| Type | Risk |
|------|------|
| DTO | Low |
| VO | Low |
| Enum | Low |
| Mapper | Medium |
| Controller | Medium |
| Service | Medium |
| Feign Client | High |
| Config | High |
| Scheduler | High |

Generate a **Risk Report** and present to the user:

```markdown
## Risk Report

| File | Risk | Reason |
|------|------|--------|
| UserDTO.java | Low | Data object only |
| UserService.java | Medium | Business logic |
| DeviceFeign.java | High | External dependency |
```

**Risk-based processing rules**:

- **Low** — Auto process, no extra confirmation needed
- **Medium** — Include in sync plan, proceed with standard review
- **High** — Must explicitly show preservation strategy (which target-only code will be kept, how external calls will be adapted). **Require user confirmation before modification**

If any file is classified as **High**, stop and present the preservation strategy to the user:

> ⚠️ 以下高风险文件需要确认保护策略：
> 1. `DeviceFeign.java` (High) — 外部依赖，将保留目标分支的 Feign 接口定义和 URL 配置。是否同意？

**Do not proceed with High-risk files until the user confirms.**

> 💡 此步骤在 Quick Sync 模式下跳过，在 Standard Sync 模式下仅标记风险，在 Strict Sync 模式下必须完成完整报告并获得用户确认后方可继续。

### Step 2: Switch to target branch and analyze differences

```bash
git checkout <target-branch>
```

For each file in the checklist, compare source vs target:

```bash
git diff <source-branch>..<target-branch> -- path/to/FileA.java
```

Build a **difference report** for each file, categorizing diffs as:
- **Business changes to sync** — new logic, bug fixes, feature additions from the commit range
- **Target-only code to preserve** — platform-specific imports, different API calls, unique conditional logic
- **Unrelated differences** — formatting, comments, pre-existing differences outside the commit range

Present the report to the user for confirmation before making any changes.

### Step 2.5: Generate Sync Plan

Before modifying any file, consolidate the difference report and risk assessment into a concrete **Sync Plan**.

For each file, generate:

```markdown
## Sync Plan

| File | Operation | Business Changes | Target Code To Preserve | Risk |
|------|-----------|-----------------|------------------------|------|
| UserService.java | Merge | Add cache logic | Fusion permission check | Medium |
| UserDTO.java | New | Full copy from source | — | Low |
| DeviceFeign.java | Merge | New API endpoint | Preserve platform endpoint | High |
| EventTypeEnum.java | Skip | — | Target has equivalent | Low |
```

**Operation types**:
- **New** — File does not exist on target, copy from source (may need adaptation)
- **Merge** — File exists on both branches, merge business changes into target
- **Skip** — No business changes need syncing (target already has equivalent)
- **Delete** — File was removed in source, confirm before removing on target

Present the plan to the user and wait for confirmation:

> 以上是同步计划，确认后开始执行文件同步。

**Do not modify any file before the user confirms the plan.**

### Step 3: Sync files one by one

#### Diff Classification Rules

Every difference between source and target must belong to exactly one category:

| Category | Description | Action |
|----------|-------------|--------|
| **Business Change** | New feature, bug fix, logic correction, performance optimization | ✓ Sync |
| **Target-only Code** | Platform adaptation, environment-specific logic, fusion/develop exclusive code | ✓ Preserve |
| **Historical Difference** | Existing branch divergence before the locked commit range | ✓ Ignore |
| **Conflict Area** | Both branches changed the same logic differently | ⚠ Stop and ask user |

When processing each file, classify every diff before deciding whether to apply it:

1. **Business Change** — These are the changes from the locked commit range. Apply them to the target file.
2. **Target-only Code** — Code that exists only on the target branch for platform reasons. Never overwrite.
3. **Historical Difference** — Pre-existing differences outside the commit range. Leave untouched.
4. **Conflict Area** — When the same method/logic was modified on both branches. **Stop immediately**, present both versions to the user, and ask which to keep.

**Conflict handling**:

> ⚠️ 冲突检测：`ClassName.methodName` 在两个分支上都有修改。
>
> **源分支版本**：
> ```
> <source code snippet>
> ```
>
> **目标分支版本**：
> ```
> <target code snippet>
> ```
>
> 请选择：1) 使用源分支版本  2) 保留目标分支版本  3) 手动合并

**Do not auto-resolve conflicts. Always ask the user.**

---

#### Execution

Process **one file at a time**. For each file:

1. Read the current target branch version
2. Read the source branch version
3. Identify the specific business changes from the locked commit range (not all differences)
4. Apply only those business changes to the target file
5. Verify target-only code is still intact

After each file, briefly report what was synced and what was preserved.

**Critical rules**:
- Never overwrite target-only code
- Never sync unrelated differences that existed before the commit range
- If unsure whether a diff is business or platform-specific, ask the user
- Do not sync all diffs blindly — understand the intent behind each change

### Step 3.5: Dependency Impact Scan

After all file syncs are complete, analyze the synchronized files for downstream impact.

#### Detect signature changes

Scan all synced files for changes in:

- DTO field additions/removals/type changes
- Service interface method signature changes (parameters, return type)
- Constructor parameter changes
- Enum value additions/removals
- Public method signature changes

#### Identify impacted files

For each signature change, trace callers and references in the **target branch** to find all impacted files:

```markdown
## Impact Report

| Changed Component | Change Type | Impacted Files |
|-------------------|-------------|----------------|
| UserService.getUser(Long) | Method signature | UserController, UserFacade, UserServiceTest |
| UserDTO | DTO field change | UserVO, UserMapper, UserExportService |
| EventTypeEnum | Enum addition | EventConstants, EventConfig |
```

#### Process impacts

- If impacted files were already in the sync plan → verify they are correctly updated
- If impacted files were **not** in the sync plan → flag them and determine if they need updates
- If impacted files are target-only code → check compatibility, update if safe, ask user if uncertain

> ⚠️ 以下文件未在同步计划中但受签名变更影响，可能需要更新：
> 1. `UserFacade.java` — 调用了 `UserService.getUser(Long)`，已变更为 `getUser(Long, String)`
>
> 是否自动修复这些受影响的文件？

**Do not skip this step** — undetected impact is the primary cause of post-sync compilation failures and runtime errors.

### Step 4: Test Impact Verification

Based on the Dependency Impact Scan (Step 3.5), verify and fix all affected test code.

#### Check

Scan test files for changes needed in:

1. **Unit Tests** — method calls, parameter lists, return value assertions
2. **Integration Tests** — bean wiring, constructor injection, context configuration
3. **Mock Definitions** — `@Mock` annotations, `when().thenReturn()` setups, mock return values
4. **Constructor Injection** — new dependencies requiring `@InjectMocks` or manual instantiation
5. **Interface Changes** — test classes implementing or referencing changed interfaces

#### Fix

Common fixes to apply:

- Method parameter mismatches (e.g. `getUser(Long)` → `getUser(Long, String)`)
- Missing mocks for new dependencies
- Missing `@Mock` / `@InjectMocks` for new injected fields
- Changed assertions (field additions/removals in returned objects)
- Import path updates for moved/renamed classes

#### Report

After fixing all test files, generate a summary:

```markdown
## Test Fix Report

| Test File | Action | Details |
|-----------|--------|---------|
| UserServiceTest | Updated mock signature | `getUser(Long)` → `getUser(Long, String)` |
| UserControllerTest | Added new field assertion | Assert `result.getType()` is not null |
| UserFacadeTest | Added new mock | `@Mock UserValidator validator` |
```

### Step 4.5: Rollback Protection

Before compilation, create a rollback point on the target branch.

#### Recommended approach

```bash
git checkout <target-branch>

# Option A: backup branch
git branch backup-sync-<timestamp>

# Option B: tag (lightweight, easier to find later)
git tag sync-before-<timestamp>
```

#### Purpose

If the sync introduces issues (compilation failures, runtime errors, behavioral regressions), the entire sync can be reverted immediately:

```bash
# Rollback to the saved point
git checkout <target-branch>
git reset --hard backup-sync-<timestamp>
# or
git reset --hard sync-before-<timestamp>
```

Record the rollback point in the final sync report so any team member can identify and revert if needed.

### Step 5: Compile and verify

```bash
mvn clean compile -DskipTests
```

If compilation fails, fix errors immediately — they indicate missed syncs or incompatible changes. Repeat until clean compile succeeds.

### Step 6: Method-level verification

For critical business logic, do a line-by-line comparison between branches:

```
Compare <source-branch> and <target-branch> for:
- ClassName.methodName — verify logic is identical
```

Read the method from both branches and diff them. Report any discrepancies. This catches cases where AI thinks it synced correctly but missed something.

Focus on methods that:
- Were explicitly modified in the commit range
- Contain complex logic (conditionals, loops, calculations)
- Have platform-adjacent code (where it's easy to accidentally overwrite target-specific code)

### Step 6.5: Sync Coverage Verification

Verify that every file from the locked commit range (Step 1) has been accounted for.

#### Generate coverage report

```markdown
## Sync Coverage Report

| Item | Count |
|------|-------|
| Files in Commit Range | N |
| Files Processed | N |
| Files Skipped | N |
| Files Failed | N |

**Coverage**: Processed / Total = **X%**
**Target**: 100%
```

#### Status definitions

- **Processed** — File was synced (Merge), created (New), or intentionally skipped (Skip) per the sync plan
- **Skipped** — File was determined to not need syncing (target already equivalent), documented with reason
- **Failed** — File sync attempted but encountered errors or unresolved conflicts

#### If coverage is not 100%

List all missing files explicitly:

```markdown
## Uncovered Files

| File | Reason | Action Needed |
|------|--------|---------------|
| SomeService.java | Sync failed due to conflict | Manual resolution required |
| SomeConfig.java | Not processed | Unknown reason — investigate |
```

**Do not proceed to the final report until coverage reaches 100% or every uncovered file is acknowledged by the user.**

### Step 7: Configuration consistency check

Verify that synced changes to configuration classes, constants, enums, and properties are consistent:

1. Check `application.yml` / `application.properties` for new/changed config keys
2. Check constant/enum classes for new values
3. Check `TableName`, `ColumnName` type classes for new entries
4. Verify config values make sense for the target platform environment

### Step 8: Output Sync Report File

Write a markdown report file to the project root as a permanent record of this sync task.

**File path**: `<project-root>/sync-report-<YYYYMMDD-HHmmss>.md`

**File content template**:

```markdown
# 跨分支同步报告

> 生成时间: YYYY-MM-DD HH:mm:ss

## 基本信息

| 项目 | 内容 |
|------|------|
| 同步类型 | 跨分支同步 (Mode 1) |
| 源分支 | <source-branch> |
| 目标分支 | <target-branch> |
| 提交范围 | <first-commit> ~ <last-commit> |
| 提交数量 | N 个提交 |
| 涉及文件数 | N 个文件 |

## 同步的提交列表

| # | Commit Hash | 提交信息 |
|---|------------|---------|
| 1 | abc1234 | fix: 修复越限事件查询逻辑 |
| 2 | def5678 | feat: 新增事件导出功能 |
| ... | ... | ... |

## 文件同步明细

| # | 文件路径 | 操作 | 同步内容 | 保留的目标独有代码 |
|---|---------|------|---------|------------------|
| 1 | service/impl/XxxServiceImpl.java | 同步 | 新增查询方法、修复计算逻辑 | 保留了平台特有的Feign调用 |
| 2 | controller/XxxController.java | 同步 | 新增导出接口 | 保留了平台特有路径前缀 |
| 3 | model/XxxDO.java | 新建 | 全新文件，从源分支复制 | — |
| ... | ... | ... | ... | ... |

## 编译结果

✅ `mvn clean compile -DskipTests` 通过 / ❌ 失败 (附错误)

## 方法级验证结果

| 类名.方法名 | 验证结果 | 备注 |
|------------|---------|------|
| XxxServiceImpl.queryData | ✅ 一致 | — |
| XxxServiceImpl.calculate | ⚠️ 差异 | 保留目标分支特有的精度处理逻辑 |

## 配置变更

| 配置文件 | 变更内容 |
|---------|---------|
| application.yml | 新增 feature.xxx.enabled: true |
| TableName.java | 新增 XXX_TABLE 常量 |

## 需要人工处理的事项

- [ ] ...
- [ ] ...

## 风险评估结果

| 文件 | 风险 | 状态 |
|------|------|------|
| UserDTO.java | Low | Auto Processed |
| UserService.java | Medium | Completed |
| DeviceFeign.java | High | User Confirmed |

## 影响分析

| 修改项 | 影响文件 |
|--------|---------|
| UserService.getUser | UserController, UserFacade |
| UserDTO | UserVO, UserMapper |
| EventTypeEnum | EventConstants |

## 同步覆盖率

| 项目 | 数量 |
|------|------|
| Commit 文件数 | N |
| 已同步 | N |
| 跳过 | N |
| 失败 | N |

**覆盖率**: X%

## 回滚信息

| 类型 | 名称 |
|------|------|
| Backup Branch | `backup-sync-<timestamp>` |
| Tag | `sync-before-<timestamp>` |

回滚命令：`git reset --hard sync-before-<timestamp>`

## 同步质量评分

| 项目 | 满分 | 得分 | 说明 |
|------|------|------|------|
| 覆盖率 | 30 | | 100% 覆盖得满分，每遗漏 1 个文件扣 5 分 |
| 编译通过 | 30 | | 一次通过得满分，每多一轮修复扣 10 分 |
| 方法校验 | 20 | | 全部一致得满分，每处差异扣 5 分 |
| 配置检查 | 10 | | 无遗漏得满分，每遗漏 1 项扣 5 分 |
| 风险处理 | 10 | | 所有 High 风险已确认得满分，未确认每项扣 5 分 |
| **总分** | **100** | | | ...
```

## Important Notes

| Concern | Guidance |
|---------|----------|
| Lock commit range first | Always determine the exact commit range before doing anything else. Without this, historical differences get treated as sync targets. |
| One file at a time | Never try to sync all files at once. File-by-file processing catches issues early and makes rollback easy. |
| Define diff rules upfront | Know which code is target-only before starting. If the user hasn't told you, ask. |
| Understand, don't copy | AI should understand what the change does and replicate the intent, not mechanically copy-paste diffs. |
| Don't forget tests | If business signatures changed, tests must follow. Compilation will catch most issues. |
| Verify at method level | AI can miss things it thinks it synced. Method-by-method comparison is the safety net. |
| Environment vs code | If an API returns empty data after sync, check whether it's an environment/config issue (different database, different service URLs) before touching code again. |
| Output report file | Always write the sync report to a markdown file. This is the permanent archive of what was done. |

## Error Handling

- **Merge conflict detected**: Do not force-resolve. Show the user the conflict and ask which version to keep.
- **Target file doesn't exist**: The file is new in source. Copy it over, but check if it needs platform-specific adjustments (imports, API calls, config references).
- **Source file was deleted**: Confirm with the user before deleting on target — the target may still need it for platform-specific reasons.
- **Compilation fails after sync**: Read the error, identify the root cause, fix only the sync-related issue. Do not refactor surrounding code.

---

# Mode 2: Cross-Project Migration (跨项目功能迁移)

Fully automated feature migration from a source project to the current project. The AI reads the source project directly, analyzes dependencies against the current project, generates a migration plan, and executes with adaptation — all in one session.

**Core principle**: One AI, two projects on the local filesystem. No manual copy-paste between sessions. The user only needs to say **what** to migrate and **where** the source project is.

## When to Use

- Migrating a feature/module from one project to another
- Source and target projects share similar architecture but have different dependencies, utils, or service calls
- Both projects are accessible on the local filesystem

## Prerequisites

Before starting, confirm with the user:

1. **Source project path** (e.g. `E:\yxl\java\SomeOtherProject`) — the project to migrate FROM
2. **Feature description** (e.g. "越限事件分析功能") — what to migrate
3. **Core files** (optional, e.g. "EventAnalysisService, EventAnalysisController") — helps narrow scope if the user already knows

The current working directory is always the **target project** (the project receiving the feature).

## Workflow

Execute these steps in order. Do not skip steps or combine them.

### Step 1: Source Project Analysis

Read and analyze the source project to understand the full scope of the feature.

#### 1a. Locate and read core files

Use the user-provided core file names to search in the source project:

```bash
find <source-project-path> -name "EventAnalysisService.java" -type f
```

Read each core file. From these files, trace the dependency graph to find ALL related files:

- Service interfaces → implementations → dependencies
- Controller → service calls → request/response models
- Models/Entities/DTOs/VOs/BOs referenced by the core code
- Utility classes directly called by the feature code
- Constants and enums referenced
- Configuration classes used

#### 1b. Expand to complete file list

Build the **full file list** by following imports and call chains. Present to user:

```markdown
## 迁移文件清单（共 N 个文件）

| # | 文件路径 (源项目相对路径) | 类型 | 说明 |
|---|-------------------------|------|------|
| 1 | service/IEventAnalysisService.java | Service接口 | 事件分析接口定义 |
| 2 | service/impl/EventAnalysisServiceImpl.java | Service实现 | 核心业务逻辑 |
| 3 | controller/EventAnalysisController.java | Controller | REST入口 |
| 4 | model/EventDO.java | DO | 事件数据对象 |
| 5 | entity/EventQueryDTO.java | DTO | 查询参数 |
| 6 | entity/EventResultVO.java | VO | 返回结果 |
| 7 | constant/EventTypeEnum.java | 枚举 | 事件类型 |
| ... | ... | ... | ... |

### 外部依赖清单

#### 工具类调用
- `SomeUtils.parseData()` — 数据解析
- `DateUtils.formatTimestamp()` — 时间格式化
- `ModelServiceUtils.queryModels()` — 数据平台查询

#### 服务/Feign调用
- `DeviceDataService.getDeviceData()` — 设备数据服务
- `AuthService.getUser()` — 认证服务

#### 常量/枚举引用
- `EventTypeEnum` — 事件类型枚举
- `TableName.EVENT_TABLE` — 事件表名常量

#### 基类/框架
- 继承 `BaseController` — 基类控制器
- 使用 `CommonManagerException` — 业务异常
```

**Ask the user to confirm**:
> 以上是要迁移的完整文件清单和外部依赖。是否需要增减？确认后我开始在当前项目中逐个匹配依赖。

---

### Step 2: Dependency Matching (源项目 → 目标项目)

For each external dependency found in Step 1, search the **current (target) project** to find equivalents or determine what's missing.

#### 2a. Match dependencies one by one

For each dependency, search the target project codebase:

| 源项目依赖 | 搜索方式 | 结果 |
|-----------|---------|------|
| 工具类 `SomeUtils` | `grep -r "class SomeUtils" <target-src>` | ✅ 有 / ❌ 没有 / ⚠️ 类似 |
| Feign `DeviceDataService` | `grep -r "interface DeviceDataService" <target-src>` | ✅ 有 / ❌ 没有 |
| 枚举 `EventTypeEnum` | `grep -r "enum EventTypeEnum" <target-src>` | ✅ 有 / ⚠️ 值不同 |
| 常量 `TableName.EVENT_TABLE` | `grep -r "EVENT_TABLE" <target-src>` | ✅ 有 / ❌ 表名不同 |
| 基类 `BaseController` | `grep -r "class BaseController" <target-src>` | ✅ 有 / ❌ 没有 |

Read the matched target files to compare signatures and values in detail.

#### 2b. Build dependency mapping table

```markdown
## 依赖匹配结果

| 源项目依赖 | 目标项目对应 | 状态 | 适配说明 |
|-----------|------------|------|---------|
| `SomeUtils.parseData()` | `ParseDataUtil.parse()` | ✅ 有 | 方法签名不同，需适配参数 |
| `DateUtils.formatTimestamp()` | `DateUtils.formatTimestamp()` | ✅ 完全一致 | 无需适配 |
| `ModelServiceUtils.queryModels()` | `ModelServiceUtils.queryModelByCondition()` | ⚠️ 方法名不同 | 方法名和参数需替换 |
| `DeviceDataService.getDeviceData()` | — | ❌ 没有 | 需要用 `DataQueryService.queryData()` 替代 |
| `AuthService.getUser()` | `AuthService.getUser()` | ✅ 完全一致 | 包路径不同，需改import |
| `EventTypeEnum` | `EventLimitType` | ⚠️ 枚举不同 | 值映射: OVER_LIMIT(1) → OVER("over") |
| `TableName.EVENT_TABLE` | `TableName.PQ_EVENT_DATA` | ⚠️ 表名不同 | 常量值不同 |
| `BaseController` | — | ❌ 没有 | Controller不继承基类 |
| `CommonManagerException` | `CommonManagerException` | ✅ 有 | 包路径可能不同 |
```

#### 2c. Ask user about unresolvable dependencies

For dependencies marked ❌ where no obvious alternative exists in the target project:

> 以下依赖在当前项目中没有找到对应实现，需要你确认：
> 1. `DeviceDataService` — 当前项目没有这个 Feign 客户端。是新建还是用其他服务替代？
> 2. `SomeSpecialUtil` — 当前项目没有这个工具类。是一起迁移过来还是用其他方式实现？

**Do not proceed** until all ❌ dependencies are resolved.

---

### Step 3: Migration Plan Generation

Based on the file list (Step 1) and dependency mapping (Step 2), generate a concrete migration plan.

#### 3a. Per-file migration instructions

For each file, read the source file content and determine the exact adaptations needed:

```markdown
## 迁移方案

### 文件 1: IEventAnalysisService.java
- **操作**: 直接复制到 `com.cet.pq.common.service.IEventAnalysisService`
- **适配**: 包声明改为目标项目包路径

### 文件 2: EventAnalysisServiceImpl.java
- **操作**: 复制后适配，放到 `com.cet.pq.common.service.impl.EventAnalysisServiceImpl`
- **适配点**:
  1. `SomeUtils.parseData()` → `ParseDataUtil.parse()`
  2. `DeviceDataService.getDeviceData(query)` → `DataQueryService.queryData(new QueryCondition(query))`
  3. `EventTypeEnum.OVER_LIMIT` → `EventLimitType.OVER`
  4. `TableName.EVENT_TABLE` → `TableName.PQ_EVENT_DATA`
  5. 去掉 `extends BaseController`，直接写

### 文件 3: EventTypeEnum.java
- **操作**: 不迁移，使用目标项目已有的 `EventLimitType`
- **说明**: 所有引用处改为使用 `EventLimitType`

### 文件 4: EventQueryDTO.java
- **操作**: 复制后适配，放到 `com.cet.pq.common.entity.EventQueryDTO`
- **适配点**:
  1. `@JsonProperty` 字段名匹配目标项目数据库列名
  2. 添加 `modelLabel` 字段（目标项目 DO 规范）

### 文件 5: EventAnalysisController.java
- **操作**: 复制后适配，放到 `com.cet.pq.common.controller.EventAnalysisController`
- **适配点**:
  1. API路径: `/api/event` → `/pqcommonplugin/api/event`
  2. 返回值 `Result` 包路径修正
  3. 去掉 `BaseController` 继承
  4. Swagger `@Api` tags 调整

### 需要额外修改的已有文件
- `application.yml` — 新增 `feature.event.enabled: true` 配置项
- （如有新的 Feign 客户端需要注册）
```

#### 3b. Present plan and get approval

> 以上是完整的迁移方案，包含 N 个文件的迁移指令和适配点。确认后我开始逐文件执行迁移。

Wait for user confirmation before proceeding.

---

### Step 4: File-by-File Migration Execution

Process **one file at a time**. For each file in the migration plan:

#### Execution process

1. **Read source file** from the source project path
2. **Determine target location** based on target project's package structure
3. **Apply all adaptations** listed in the plan:
   - Package declaration → target project's package
   - Import statements → target project's equivalents
   - Utility class calls → target project's replacements
   - Feign client calls → target project's service clients
   - Constants/enums → target project's values with correct mapping
   - Exception types → target project's exception class
   - Return types → target project's response wrapper
   - `@JsonProperty` annotations → target project's naming convention
   - `modelLabel` field → correct value for target project
   - Controller paths → target project's URL conventions
   - Base class inheritance → remove or replace
   - Swagger annotations → target project's API group
4. **Create the file** at the target location
5. **Report**: Brief summary of what was created and what was adapted

**Critical rules**:
- Read the source file completely before writing — never copy partially
- Every import must be resolved against the target project — no source project imports left behind
- If the source file has dependencies not covered in the plan, stop and ask the user
- Match the target project's coding style (Lombok usage, naming conventions, annotation style)

#### After all files are created

1. Check if any existing files need modifications (new config entries, new Bean registrations, etc.)
2. Apply those modifications

---

### Step 5: Compile and Fix

```bash
mvn clean compile -DskipTests
```

If compilation fails:
1. Read each error message carefully
2. Identify if it's a missing import, wrong method signature, type mismatch, etc.
3. Fix only the migration-related issue — do not refactor surrounding code
4. Re-compile until clean

Common post-migration compilation issues:
- Missing imports (source project package vs target project package)
- Method signature differences between source and target utility classes
- Type mismatches from enum/constant value changes
- Missing `@Autowired` / `@Resource` for new dependencies

---

### Step 6: Verification

#### 6a. Method-level spot check

For critical business logic methods, read both the source file and the migrated file, compare line-by-line:

- Focus on methods with complex logic (conditionals, loops, calculations)
- Focus on methods near adapted code (where it's easy to accidentally change behavior)
- Verify the business logic is preserved despite adaptations

Report any discrepancies to the user.

#### 6b. Configuration consistency check

1. Verify new config entries are in the correct `application.yml`
2. Verify new constants/enums are consistent with target project conventions
3. Verify `TableName`/`ColumnName` references point to valid target project tables
4. Check if new API paths need gateway registration (inform the user)

---

### Step 7: Output Migration Report File

Write a markdown report file to the target project root as a permanent record of this migration task. Also present the key content to the user in the conversation.

**File path**: `<target-project-root>/migration-report-<YYYYMMDD-HHmmss>.md`

**File content template**:

```markdown
# 跨项目迁移报告

> 生成时间: YYYY-MM-DD HH:mm:ss

## 基本信息

| 项目 | 内容 |
|------|------|
| 迁移类型 | 跨项目功能迁移 (Mode 2) |
| 源项目路径 | <source-project-path> |
| 目标项目路径 | <target-project-path> |
| 迁移功能 | <功能名称> |
| 迁移文件数 | N 个文件 |

## 已迁移文件

| # | 源文件 (相对路径) | 目标位置 (完整包路径) | 操作 | 适配点数 | 适配说明 |
|---|-----------------|---------------------|------|---------|---------|
| 1 | service/IEventAnalysisService.java | com.cet.pq.common.service.IEventAnalysisService | 新建 | 1 | 包路径 |
| 2 | service/impl/EventAnalysisServiceImpl.java | com.cet.pq.common.service.impl.EventAnalysisServiceImpl | 新建+适配 | 5 | 工具类替换、枚举映射等 |
| 3 | controller/EventAnalysisController.java | com.cet.pq.common.controller.EventAnalysisController | 新建+适配 | 4 | 路径前缀、基类、Swagger |
| ... | ... | ... | ... | ... | ... |

## 未迁移文件（使用目标项目已有实现）

| 源文件 | 目标项目对应 | 原因 |
|--------|------------|------|
| EventTypeEnum.java | EventLimitType | 目标项目已有，值做了映射 |

## 依赖映射

| 源项目依赖 | 目标项目对应 | 状态 | 适配方式 |
|-----------|------------|------|---------|
| SomeUtils.parseData() | ParseDataUtil.parse() | ✅ 有 | 方法替换，参数格式调整 |
| DeviceDataService.getDeviceData() | DataQueryService.queryData() | ✅ 替代 | 整体替换 + 参数对象转换 |
| EventTypeEnum | EventLimitType | ⚠️ 适配 | 枚举值映射 |
| TableName.EVENT_TABLE | TableName.PQ_EVENT_DATA | ⚠️ 适配 | 常量值不同 |
| BaseController | — | ❌ 无 | Controller 不继承基类 |

## 编译结果

✅ `mvn clean compile -DskipTests` 通过 / ❌ 失败 (附错误信息)

## 方法级验证结果

| 类名.方法名 | 源项目 | 目标项目 | 验证结果 | 备注 |
|------------|--------|---------|---------|------|
| EventAnalysisServiceImpl.queryEvents | ✓ | ✓ | ✅ 一致 | 工具类调用已适配 |
| EventAnalysisServiceImpl.calculateResult | ✓ | ✓ | ✅ 一致 | 枚举值已映射 |

## 配置变更

| 配置文件 | 变更内容 |
|---------|---------|
| application.yml | 新增 feature.event.enabled: true |

## 需要人工处理的事项

- [ ] `application.yml` 中新增的配置项需要在各环境配置中补充
- [ ] 新增 API 路径需要在网关注册
- [ ] 枚举映射关系 (EventTypeEnum ↔ EventLimitType) 需要与业务方确认
- [ ] 启动项目后手动测试迁移的功能接口
- [ ] 检查数据库表结构是否匹配（如有新表需建表）
```

After writing the file, tell the user:

> 📄 迁移报告已保存到 `migration-report-XXXXXXXX-XXXXXX.md`，包含完整的文件清单、依赖映射、编译结果和待办事项。

---

## Important Notes

| Concern | Guidance |
|---------|----------|
| Read source, write target | Always read from source project path, always write to current (target) project. Never mix paths. |
| One file at a time | Process files sequentially. This catches issues early and makes rollback easy. |
| Resolve all dependencies before writing | Complete Steps 1-3 (analyze + match + plan) before creating any files. Don't start writing code with unresolved ❌ dependencies. |
| Understand, don't copy | Read and understand what the source code does, then recreate with target project conventions. Not mechanical copy-paste. |
| Ask when unsure | If a dependency can't be matched or a code pattern is ambiguous, ask the user. Don't guess. |
| Preserve business logic | The goal is identical business behavior with target project's technical stack. Never change the business logic itself. |
| Target project conventions | All migrated code must follow the target project's coding standards (see CLAUDE.md for conventions). |
| Output report file | Always write the migration report to a markdown file. This is the permanent archive of what was done. |

## Error Handling

- **Source file not found**: Ask the user to verify the source project path and file names.
- **Target project missing a critical dependency with no alternative**: Stop and ask the user. Offer options: create the dependency in target project, use a different approach, or skip the feature that requires it.
- **Compilation fails after migration**: Read the error, trace it to a specific adaptation point, fix it. Common causes are import paths, method signature differences, and type mismatches.
- **Behavioral difference after migration**: Compare source and target file line-by-line around the adapted code. Find where the adaptation changed behavior and fix it.
- **Data model mismatch**: If table names, column names, or data types differ between projects, ask the user to confirm the mapping. Don't assume target project uses the same schema.
