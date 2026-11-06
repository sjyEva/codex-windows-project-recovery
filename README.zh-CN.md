# Windows Codex Desktop 本地项目消失：诊断与恢复案例

[English](README.md)

> **2026 年 11 月 6 日的一次个人恢复案例。** 这不是 OpenAI 官方修复方案，也不是适用于所有版本的通用恢复脚本。请先做只读检查，不要直接修改正在使用的 SQLite 或 JSON。

## 适用场景

Codex Desktop 更新后，Projects 侧边栏中的部分或全部**本地项目**不显示，但源文件夹和部分历史聊天仍然存在。

本案例帮助你区分：项目数据本身是否丢失，还是桌面端的项目注册状态与数据库不一致。**恢复项目列表不等于恢复每条历史聊天的项目归属。**

## 一、先做只读检查

### 1. 备份与确认配置目录

保存工作，正常退出 Codex Desktop、使用 Codex 扩展的 VS Code 及 Codex CLI，然后再复制配置文件，避免后台进程在复制时写入数据库。

在**独立的 PowerShell** 中检查 `CODEX_HOME` 是否覆盖默认位置：

```powershell
$env:CODEX_HOME
$profile = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME '.codex' }
Write-Output $profile
```

将整个 `.codex` 配置目录复制到另一个私有目录，并保留 `state_5.sqlite`、可能存在的 `-wal` / `-shm`、`.codex-global-state.json`、`.bak`、`sessions` 和 `archived_sessions`。**不要把这些原始文件上传 GitHub。**

### 2. 检查备份 SQLite

使用 [DB Browser for SQLite](https://sqlitebrowser.org/) 打开**备份副本**，先确认表结构，再逐条运行：

```sql
PRAGMA quick_check;

SELECT COUNT(*) AS projects FROM projects;
SELECT COUNT(*) AS project_roots FROM project_roots;
SELECT COUNT(*) AS threads FROM threads;

SELECT COUNT(*) AS total_threads,
       COUNT(project_id) AS threads_with_project_id
FROM threads;
```

`PRAGMA quick_check` 返回 `ok` 是基础完整性信号，但并不证明项目关联一定正确。`threads.project_id` 全为 NULL 也不代表聊天内容已经消失。

### 3. 检查全局 JSON

下面只统计当前注册数量，不输出真实项目 ID 或本机路径：

```powershell
$state = Get-Content -LiteralPath (Join-Path $profile '.codex-global-state.json') -Raw -Encoding UTF8 | ConvertFrom-Json
$local = $state.PSObject.Properties['local-projects'].Value
$order = $state.PSObject.Properties['project-order'].Value
$assignments = $state.PSObject.Properties['thread-project-assignments'].Value

[pscustomobject]@{
  LocalProjects = @($local.PSObject.Properties).Count
  OrderEntries = @($order).Count
  AssignmentEntries = @($assignments.PSObject.Properties).Count
} | Format-List
```

如果字段不存在或结构不同，不要把本案例的 JSON 结构硬套到你的版本。

### 4. 根据结果决定方向

| 观察结果 | 应采取的方向 |
|---|---|
| SQLite 有项目，JSON 注册数量明显更少 | 检查新旧 ID 映射、项目注册状态 |
| SQLite 与 JSON 都有项目，但侧边栏不显示 | 重点排查桌面端加载与 `project/list`，重写 JSON 可能无效 |
| SQLite 损坏或缺少项目记录 | 保留备份，寻找历史快照和官方支持，不要猜测 ID |
| 聊天关联缺失 | 仅恢复有明确历史证据的关联，不能单凭 `cwd` 推断 |
| 无法验证唯一、正确的旧 ID 映射 | **停止写入，不应套用本次恢复方法** |

## 二、本案例的审计证据

环境：Windows Codex Desktop `26.1029.61003`，2026 年 11 月 6 日完成恢复。

| 恢复前数据 | 结果 |
|---|---:|
| SQLite 项目 / 根目录 | 27 / 30 |
| SQLite 历史会话 | 944 |
| `project_id` 非空的会话 | 0 |
| JSON 本地项目注册 | 3 |
| JSON 排序记录 | 2 |
| JSON 聊天关联 | 11 |
| Local / Remote 关联 | 2 / 9 |
| 已验证唯一 key—有效项目配对 | 27 |
| 指向不存在项目的幂等记录 | 4 |

数据库 `PRAGMA quick_check` 返回 `ok`。23 个项目的根目录全部可访问，2 个部分可访问，2 个不可访问；没有因为路径不可访问就删除项目。

### 验证项目身份

同时交叉核对 SQLite 的 `projects.id`、`project_roots.path`、`project_idempotency_keys(key,project_id)`、`projects.metadata` 和 JSON 的 `local-projects`、`app-server-project-id-by-legacy-project-id-by-host`。

在这个 profile 中，27 个现存项目各自拥有一个唯一有效的幂等 key；3 条对应当前 JSON 的有效映射，另外 4 条孤立记录被排除。**这不意味着所有 Codex 版本的幂等 key 都是旧项目 ID。**

可以对你自己的**数据库副本**执行：

```sql
SELECT COUNT(*) AS all_key_rows,
       SUM(CASE WHEN p.id IS NULL THEN 1 ELSE 0 END) AS orphan_rows,
       COUNT(DISTINCT CASE WHEN p.id IS NOT NULL THEN k.project_id END) AS projects_with_keys
FROM project_idempotency_keys AS k
LEFT JOIN projects AS p ON p.id = k.project_id;

SELECT p.id, COUNT(k.key) AS valid_keys_for_project
FROM projects AS p
LEFT JOIN project_idempotency_keys AS k ON k.project_id = p.id
GROUP BY p.id
HAVING COUNT(k.key) <> 1;
```

第二条可能在本机显示项目 ID，**不要公开查询输出**。即使返回 0 行，也只证明“每个项目恰好有一个有效 key”，不能单独证明 ID 的语义。

### Local 与 Remote 不能混淆

11 条 JSON 关联值实际上是包含 `projectKind`、`projectId`（部分带 `hostId`）的对象：2 条 Local 的聊天 ID 能在本地 SQLite 找到并映射到现有项目，9 条 Remote 的聊天 ID 不在本地 SQLite 中，因此只保留、不强制转换或本地归类。

特别重要：原 `project-order` 的第二个 ID 虽然不是本地项目，但被这 9 条 Remote 关联引用，**不能当成无效 ID 删除**。

## 三、这次如何恢复（非通用写入教程）

1. 完整备份，并对数据库和 JSON 做只读检查。
2. 核对 27 个数据库项目的唯一身份映射，排除无效与冲突记录。
3. 在备份目录中生成候选 JSON，保留原有 3 个项目，补入经过核验的 24 个缺失项目。
4. 原样保留 2 条 Local、9 条 Remote 关联、迁移标志和其他设置。
5. 保留原 `project-order` 两个条目，再按数据库 `position` 补入缺失的 Local ID。最终排序为 28 条（27 Local + 1 Remote）。
6. 当 JSON 哈希变化时，拒绝旧候选，重新从最新状态生成；不绕过安全校验。
7. 第一次实际写入因 `code.exe` 仍在运行而被阻止。正常关闭 VS Code 后，在独立 PowerShell 中执行受控操作。
8. 写入前创建哈希校验的回退快照，最终**只原子替换 `.codex-global-state.json`**，不修改 SQLite、会话文件和迁移标志。
9. 重新打开 Codex，用户确认项目已重新显示。

本案例的写入脚本包含针对这一 profile 的项目数量、映射结构和排序方式的限制，因此**不提供一条所有人都能直接执行的 `--apply` 命令**。如要发布通用工具，需要在多个版本和合成测试配置上充分验证候选生成、并发写入检测、备份和回退机制。

## 四、恢复结果与局限

| 字段 | 写入前 | 写入后 |
|---|---:|---:|
| `local-projects` | 3 | 27 |
| `project-order` | 2 | 28 |
| 项目 ID 映射 | 4 | 28 |
| 保留 Local 关联 | 2 | 2 |
| 保留 Remote 关联 | 9 | 9 |

用户确认本地项目重新显示。写入日志也确认实时 SQLite 项目目录与已审计副本一致，并保存了替换前回退快照。**现有材料没有记录完整重启后的持久性验收**，不能宣称已经验证过多次重启。

这次**没有**为 944 条会话重新填写 `threads.project_id`，不能理解为全部历史聊天分组都恢复了。

### 恢复后检查与回退原则

检查项目列表、部分项目能否打开、旧聊天能否访问，并在完整退出和重新启动后检查是否仍然存在。保留原始备份与写入前快照。如果出现异常，先停止进一步写入，比较当前状态与快照；不能不加区分地重复执行或覆盖新数据。需要回退时，应当先退出所有写入进程，并对回退时的当前状态再次备份。

## 五、安全提示与相关 Issue

这是**个人环境中的非官方恢复方法**，不是 OpenAI 的通用修复方案，也不能证明更新导致的所有故障都有相同根因。即使 SQLite 和 `project/list` 都正常，桌面端仍可能显示为空。

不要公开 `.codex` 原始目录、数据库、状态 JSON、项目和聊天 ID、账号信息、路径、哈希或聊天内容；不要根据 `cwd` 批量分组聊天，也不要随意重置迁移标志。

相关讨论：[openai/codex#42739](https://github.com/openai/codex/issues/42739)。
