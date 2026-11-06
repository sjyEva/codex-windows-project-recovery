# Recovering Missing Local Codex Projects on Windows

[简体中文](README.zh-CN.md)

> **Case study — successful on one Windows profile (November 6, 2026).** This is **not** an official OpenAI repair procedure or a universal recovery script. Start with the read-only checks below; do not edit your live database or global-state JSON until you have independently validated the IDs and have a rollback snapshot.

## Who this guide is for

After a Codex Desktop update, local projects are missing from the **Projects** sidebar, but their source folders and some previous chats still exist. This guide helps you distinguish missing **project data** from missing **desktop project registration**, then explains the evidence and safeguards used in one successful recovery.

**Important:** Restoring the project list is different from restoring each historical chat's project membership. In the case described here, the local projects reappeared, but membership was **not** reconstructed for all 944 threads.

## Start here: safe read-only checks

### 1. Preserve the profile and confirm the profile location

Save your work and fully exit Codex Desktop, VS Code/Codex extensions, and any Codex CLI sessions **before copying live state**. Closing a window may leave background writers running.

In a standalone PowerShell window, check whether `CODEX_HOME` overrides the default profile location:

```powershell
$env:CODEX_HOME
$profile = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME '.codex' }
Write-Output $profile
```

Copy the **entire** profile to a separate, private backup folder, including `state_5.sqlite`, any `-wal`/`-shm` companions, `.codex-global-state.json`, `.bak` files, `sessions`, and `archived_sessions` when present. Do not upload this backup to GitHub. Check that the copied files exist before proceeding.

### 2. Audit a **copied** SQLite database

Use [DB Browser for SQLite](https://sqlitebrowser.org/) or another SQLite client to open the **backup copy**, not the live database. Confirm the real schema first; table names may differ between versions.

Run these read-only queries independently:

```sql
PRAGMA quick_check;

SELECT COUNT(*) AS projects FROM projects;
SELECT COUNT(*) AS project_roots FROM project_roots;
SELECT COUNT(*) AS threads FROM threads;

SELECT
  COUNT(*) AS total_threads,
  COUNT(project_id) AS threads_with_project_id
FROM threads;
```

`PRAGMA quick_check` should return `ok` for a sound basic integrity check; this is **not** proof that all project or chat relationships are correct. `threads.project_id IS NULL` does **not** mean chat content has vanished.

### 3. Compare the desktop global-state registry

Check the **current** JSON and any preserved pre-incident snapshots. The following commands only read the current JSON; they do not display private IDs or paths.

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

If a field is absent or your JSON has a different shape, **stop** rather than assuming this case's schema applies.

### 4. Interpret the results before attempting repair

| Observation | Next action |
|---|---|
| Project rows exist in SQLite; desktop registry has fewer entries | Check legacy-ID mappings and desktop registration; this is the branch used in this case. |
| Project rows and registry both exist, but sidebar is empty | Investigate desktop catalog loading / `project/list` behaviour; rewriting JSON may not help. |
| SQLite is corrupt, incomplete, or missing project rows | Preserve the backup, investigate other snapshots and official support; **do not** reconstruct IDs by guessing. |
| Historical chat assignments are absent | Restore only assignments supported by exact historical evidence; **do not** bulk-assign by `cwd`. |
| You cannot establish unambiguous old-ID-to-new-ID mappings | Do **not** apply the reconstruction described below. |

## What happened in this case

**Environment:** Windows Codex Desktop `26.1029.61003`; audited and recovered November 6, 2026.

| Evidence before repair | Count |
|---|---:|
| SQLite project rows / project-root rows | 27 / 30 |
| SQLite threads | 944 |
| Threads with non-NULL `project_id` | 0 |
| JSON `local-projects` entries | 3 |
| JSON `project-order` entries | 2 |
| JSON `thread-project-assignments` entries | 11 |
| Local / remote assignment objects | 2 / 9 |
| Valid, unique idempotency-key-to-project pairs | 27 |
| Idempotency rows pointing to absent project IDs | 4 |

The copied database returned `ok` for `PRAGMA quick_check`. At audit time, 23 projects had all roots accessible, two had partial root accessibility, and two had none; none was deleted solely because a path was unavailable.

### How project identity was validated

The following data sources were checked **together**, rather than treating names or raw IDs as interchangeable:

- `projects.id`, `projects.name`, `projects.position`, and `project_roots.path` in SQLite;
- `project_idempotency_keys(key, project_id)` in SQLite;
- `projects.metadata`, including `codex.localProjectId` where present;
- JSON `local-projects` and `app-server-project-id-by-legacy-project-id-by-host`.

For your **own** copied database, these read-only checks can flag orphaned references and projects with no or multiple valid key rows:

```sql
SELECT
  COUNT(*) AS all_key_rows,
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

The second query may expose project IDs in your local SQL client: **do not post those results publicly**. Zero returned rows are only a uniqueness check, not proof of legacy-ID semantics.

In this particular profile, **each of the 27 existing projects had exactly one unique valid idempotency-key/project pairing**. Three pairings matched existing JSON mappings. Two projects additionally had `codex.localProjectId` metadata matching that mapping chain. Four orphan idempotency rows were excluded. **Do not assume idempotency keys are legacy IDs in every Codex version** without equivalent cross-checks.

The 11 JSON chat-assignment values were **objects**, not raw project-ID strings (`projectKind`, `projectId`, sometimes `hostId`). Both **local** chat IDs existed in the local SQLite database and uniquely mapped to valid projects. The nine **remote** chat IDs did not occur in that database; they were preserved as remote assignments, not converted into local chats.

### A non-obvious `project-order` issue

One of the original two `project-order` entries did **not** appear in local project IDs or SQLite, but was referenced by the nine **remote** assignments. Deleting it as an “invalid ID” would have been unsafe. The recovery candidate therefore preserved both original order entries, then appended the missing **local** project IDs in SQLite `position` order.

## How the recovery was performed — **not a universal apply procedure**

1. A **staged candidate**, stored outside the live profile, was constructed from the copied SQLite data and the latest JSON. Three existing project entries remained unchanged; 24 missing entries were added only after identity and root-path checks.
2. The candidate expanded `local-projects` **3 → 27**, the legacy-to-app-server mapping **4 → 28**, and `project-order` **2 → 28** (27 local projects plus the preserved remote-referenced entry).
3. Every existing thread-assignment object (two local, nine remote), unrelated JSON setting, and migration flag was preserved unchanged. No new historical chat memberships were inferred from `cwd`.
4. The candidate was checked against the current JSON and copied database using hashes and structural assertions. When the live JSON changed, the stale candidate was rejected and regenerated; the safety check was **not** bypassed.
5. Before any write, the recovery tool required possible profile writers to exit. The first apply attempt stopped because **`code.exe` (VS Code)** was still running. VS Code was exited normally; the tool was then run from standalone PowerShell.
6. A fresh, hash-verified pre-apply snapshot was created. The controlled operation atomically replaced **only** `.codex-global-state.json`. It did **not** edit SQLite, thread rows, session files, or migration flags.
7. Codex Desktop was reopened, and the user confirmed that the missing local projects were visible again.

**Why there is no copy-paste `--apply` command here:** The actual scripts had hard-coded expectations for this profile (27 projects, the observed schema, specific mapping invariants, and a particular ordering convention). Publishing those scripts as a universal fixer could overwrite someone else's valid state. A reusable recovery tool needs independent testing against multiple versions and synthetic profiles, plus a verified backup, dry-run diff, writer detection, and rollback path. Until then, use the process above as a **validation checklist**, not an instruction to write these fields into your own profile.

## Outcome and verification

| Field | Before | After controlled replacement |
|---|---:|---:|
| JSON local project entries | 3 | 27 |
| JSON order entries | 2 | 28 |
| JSON legacy-ID mappings | 4 | 28 |
| Preserved local chat assignments | 2 | 2 |
| Preserved remote chat assignments | 9 | 9 |

The user reported that the missing projects reappeared. The apply log also confirmed that the live SQLite project/root inventory matched the audited copy and that a rollback snapshot was saved. **A full subsequent restart-and-persistence verification was not recorded in the evidence presented here.**

This operation did **not** repopulate `threads.project_id` for the 944 local threads and should not be described as restoration of every historical chat's project membership.

## Post-recovery checklist and rollback principles

- Reopen Codex Desktop and check the Projects list.
- Open representative restored projects and an older chat, if accessible.
- Verify that local and remote chat entries remain present.
- Quit and restart the desktop app to check persistence; if the list disappears again, collect diagnostics before taking further action.
- Keep both the original profile backup and pre-apply snapshot. If the application behaves incorrectly, **stop** and compare the saved state with the live one; do not repeatedly run the apply operation or blindly overwrite newer data.
- Rollback should be performed only with all writers stopped and after another backup of the current state. Restoring one JSON file may be insufficient if other profile files have changed since the snapshot.

## Limits, privacy, and related reports

This is a **single-profile, unsupported workaround**, not an official OpenAI fix or proof that the underlying desktop migration issue is resolved. Schema and ID semantics can vary by release, and some users report that the SQLite project catalog and `project/list` are healthy while the Desktop sidebar remains empty. In that situation, registry reconstruction alone may not help.

Do **not** publish or share your raw `.codex` folder, database, state JSON, thread IDs, project IDs, local paths, account IDs, or private hashes. Do not reset migration flags, bulk-assign chats from `cwd`, or patch the live SQLite database as a first resort.

Related upstream discussion: [openai/codex#42739 — Local projects disappear from sidebar after Windows desktop update](https://github.com/openai/codex/issues/42739).
