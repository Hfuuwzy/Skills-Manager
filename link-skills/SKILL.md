---
name: link-skills
description: Enable skills from the central repository at ~/.skills/ for a specific tool by linking or copying them to the right discovery directory. Supports four targets - OpenCode global, Claude global, project-level OpenCode (.opencode/skills), project-level Claude-compatible (.claude/skills). Windows-first auto mode prefers junctions (mklink /J) for local directories, including cross-drive local NTFS paths, and falls back to robocopy /MIR only when a junction is not appropriate. Use this when user asks to enable, link, attach, expose, or activate a skill for OpenCode/Claude in either global or project scope.
---

# link-skills

## Purpose

Take skills that already live in the user's central skill repository at `$env:USERPROFILE\.skills\<category>\<skill-name>\` and make them discoverable by a specific tool (OpenCode or Claude) at either global or project scope.

Auto mode picks the right Windows mechanism per `(skill, target)` pair:

- **Any local directory path on this machine** -> directory **junction** (`mklink /J`). No admin, no Dev Mode, works across local drive letters too.
- **Network / UNC / non-local path where junction is not appropriate** -> mirrored **copy** (`robocopy /MIR`). Works without privilege, but is a snapshot, not live.
- **Explicit override** -> directory **symlink** (`mklink /D`) only when the user specifically asks for `-Mode symlink` and Developer Mode (or admin) is available.

Edits in the central repo are visible immediately at the destination for both junctions and symlinks. Robocopy targets must be re-linked after central updates.

This skill is the "publish" half of the workflow. Its sister skills are:

- `install-skill` - places new skills *into* the central repo at `$env:USERPROFILE\.skills\`.
- `audit-skills` - health checks linked / copied skills.

## When to Use

Trigger this skill when the user says any of:

- "enable skill X for OpenCode / Claude"
- "link skill X globally / for this project"
- "attach / expose / activate X for OpenCode"
- "add my custom skills to `.opencode/skills`" or `.claude/skills`
- "publish all `tools/*` skills to my OpenCode global config"

Do **not** trigger this skill for:

- Authoring or editing a skill (use `skill-creator`).
- Importing a brand new skill into the repo (use `install-skill`).
- Diagnosing broken links (use `audit-skills`).

## Targets Supported

Exactly four targets. Use these names verbatim when talking to the user.

| # | Target name        | Destination path                                              | Scope    | Tool                |
|---|--------------------|---------------------------------------------------------------|----------|---------------------|
| 1 | OpenCode-Global    | `$env:USERPROFILE\.config\opencode\skills`                    | Global   | OpenCode            |
| 2 | Claude-Global      | `$env:USERPROFILE\.claude\skills`                             | Global   | Claude (and compat) |
| 3 | OpenCode-Project   | `<repo-root>\.opencode\skills`                                | Project  | OpenCode            |
| 4 | Claude-Project     | `<repo-root>\.claude\skills`                                  | Project  | Claude (and compat) |

`<repo-root>` is resolved by walking up from the current working directory until a folder is found that contains any of: `.git`, `package.json`, `pom.xml`, or `AGENTS.md`. If no such ancestor exists, **ask the user for an absolute project root path** before continuing - do not guess.

OpenCode discovers from `~/.config/opencode/skills`, `~/.claude/skills`, `.opencode/skills`, and `.claude/skills`. Choosing target 2 or 4 also makes the skill available to Claude Code if installed.

## Project Scope Boundary

For project-level targets, this skill manages **only** these directories:

- `<repo-root>\.opencode\skills\`
- `<repo-root>\.claude\skills\`

Never initialize or manage a package environment for project skills:

- Do **not** run `npm init`, `npm install`, `bun install`, or similar package-manager commands.
- Do **not** create or modify `.opencode\package.json`, `.opencode\package-lock.json`, `.opencode\bun.lock`, `.opencode\node_modules`, or `.opencode\.gitignore`.
- Do **not** create `.opencode\plugins\` or `.opencode\tools\` for this skill.

If those files already exist, treat them as **OpenCode plugin/npm artifacts**, not as skill requirements. Mention them only as informational context and leave them untouched. Project skills require only `<repo-root>\.opencode\skills\<skill-name>\SKILL.md` (or the Claude-compatible `.claude\skills` equivalent).

## Inputs

Collect (asking the user only for what is missing):

- `Skill` - one of:
  - `<category>/<skill-name>` (e.g. `tools/link-skills`)
  - `<category>/*` (whole category)
  - `*` (every skill in the central repo)
  - a list of any of the above
- `Target` - one of `OpenCode-Global`, `Claude-Global`, `OpenCode-Project`, `Claude-Project`. Multiple allowed.
- `Mode` (advanced optional override) - `junction` | `symlink` | `copy` | `auto` (default `auto`). Do **not** ask for this in the normal picker flow; only honor it when the user explicitly requests a mode (for example "use symlink" or "force copy").
- `ProjectRoot` (only required for `*-Project` targets when auto-detection fails).

Always echo the resolved inputs back to the user before doing anything destructive.

## Discovery

The central repo root is **always** `$env:USERPROFILE\.skills`. Never accept a different root for this skill.

Layout assumed: `~/.skills/<category>/<skill-name>/SKILL.md` (depth 2).

Discovery must be **fresh at runtime**. Every time the user enters `link-skills`, and every time they choose or re-choose skill selection, rescan `$env:USERPROFILE\.skills` from disk before building picker options. Do **not** reuse:

- OpenCode's already-loaded `<available_skills>` metadata.
- A hard-coded category/skill index from this SKILL.md, README, prior audit output, or a previous run.
- A `$skills` / `$categories` variable populated before an `install-skill` run or before the current picker stage.

Reason: users often run `install-skill` immediately before `link-skills`; newly copied folders must appear in `Individual skills` pages and `By category` tabs without restarting OpenCode.

Discovery procedure:

1. List categories: `Get-ChildItem -LiteralPath $env:USERPROFILE\.skills -Directory`.
2. For each category, list skill folders: `Get-ChildItem -LiteralPath <category> -Directory`.
3. Keep only folders that contain a `SKILL.md` file at their root.
4. Filter out:
   - Folders whose name starts with `.` or `_`.
   - The folder `tools\link-skills` itself when generating the *interactive selection list* (it can still be linked when the user names it explicitly).
5. If a folder lacks `SKILL.md`, **warn** the user (`"<category>/<name>: SKILL.md missing - skipped"`) but continue.

Pull the description from the YAML frontmatter `description:` field of each `SKILL.md` (read the first ~30 lines, find the `description:` key).

The discovered list is the only source of truth for interactive options. Sort by `<category>/<skill-name>` after scanning, then derive page counts, category tab names, and option labels from that sorted list.

## Interactive Selection UX

Prefer OpenCode's native `question` tool instead of asking the user to manually type skill paths. Use plain numbered text only as a fallback when the runtime does not expose `question`.

### Batched picker sequence

Minimize back-and-forth, but keep the order intuitive: choose **what** first, then choose **where**. Prefer one `question` call with multiple `questions` tabs for each stage whenever the options are known up front. Do **not** ask one page at a time when the pages can be batched into tabs.

Before constructing any skill picker in this sequence, run the Discovery procedure again and build a fresh in-memory list. The scope question itself may be static, but every option after the scope answer must come from the latest filesystem scan.

1. **Select scope first** (one `question` call, single choice): `All skills`, `By category`, `Individual skills`, `Type manually`.
2. **If scope is `All skills`**: rescan now, expand to the full fresh discovered list; no skill-picker follow-up is needed.
3. **If scope is `Type manually`**: ask once for the manual text (`*`, `<category>/*`, `<category>/<skill>`, or comma-separated values), then resolve those tokens against a fresh scan and warn for missing entries.
4. **If scope is `Individual skills`**: rescan now, then immediately present all discovered skills in one batched picker call split into page tabs:
   - `Skills 1/3`, `Skills 2/3`, `Skills 3/3`, etc.
   - Each page tab uses `multiple: true` and contains about 15 skill options.
   - The user should be able to select across all page tabs and submit once.
   - Do **not** ask for a keyword/filter first.
5. **If scope is `By category`**: rescan now, then immediately present one batched picker call with one tab per discovered category:
   - Use only categories that currently exist under `~/.skills` and contain at least one valid skill with root `SKILL.md`.
   - Each category tab uses `multiple: true` and lists the individual skills in that category.
   - Selecting the `By category` scope does **not** install whole categories automatically; the user still chooses specific skills inside category tabs.
   - If a category has more than about 15 skills, split that category into tabs like `research 1/2`, `research 2/2`.
6. **Select targets after skills are selected** (one `question` call, `multiple: true`): `OpenCode-Global`, `Claude-Global`, `OpenCode-Project`, `Claude-Project`.
7. **Final confirmation** (one final `question` call): show the resolved plan and ask `Proceed`, `Change selection`, `Cancel`. If the user chooses `Change selection`, discard the old discovered list and restart at Step 1 or Step 4/5 with a fresh scan.

### Pagination and batching rules

- Use about 15 real skill options per tab/page.
- For `Individual skills`, pagination is count-based over the freshly sorted full skill list. Do **not** pack multiple category groups into one trailing tab if that makes the tab exceed about 15 options.
- Prefer multiple tabs in one `question` call over sequential page-navigation questions.
- Use labels like `Skills 1/3` or `research 2/3` for page headers.
- Preserve selections across page/category tabs; union all selected skills after submit.
- Do not use keyword filtering in the normal flow; batched pagination is the default long-list solution.
- Only use sequential follow-up questions when the answer changes the available option set and the new options could not be constructed beforehand.

### Dynamic option construction guardrails

- Recompute total page count from the freshly discovered skill count: `ceil(count / 15)`. Never reuse old `Skills 1/4` / `Skills 4/4` labels after the discovered count changes.
- Recompute category tabs from the fresh scan. If `install-skill` just added `document/new-skill`, `By category` must include `document` and that skill immediately.
- Omit empty categories. Do not show category tabs based on the allowed-category list alone.
- If a selected skill disappears between selection and execution, stop before mutation and show `Missing from ~/.skills after selection: <category>/<skill>`; ask the user to reselect from a fresh scan.
- If new skills appear between selection and execution, do not silently add them to the current plan; they will appear if the user chooses `Change selection`.

Do **not** show a Mode picker in the normal flow. Default to `auto`. Only ask about mode when the user explicitly requests a mode override or the planned target is unusual enough that `auto` cannot decide safely.

### `question` tool shape

Use concise headers and short labels. Example:

```text
question({
  questions: [{
    header: "Scope",
    question: "How should skills be selected?",
    multiple: false,
    options: [
      { label: "All skills", description: "Link every skill in ~/.skills." },
      { label: "By category", description: "Choose skills inside category tabs." },
      { label: "Individual skills", description: "Choose from paginated skill tabs." },
      { label: "Type manually", description: "Enter *, category/*, or category/skill." }
    ]
  }]
})
```

For individual skills, ask all pages at once:

```text
question({
  questions: [
    {
      header: "Skills 1/3",
      question: "Select skills from page 1.",
      multiple: true,
      options: [ /* first ~15 skills */ ]
    },
    {
      header: "Skills 2/3",
      question: "Select skills from page 2.",
      multiple: true,
      options: [ /* next ~15 skills */ ]
    },
    {
      header: "Skills 3/3",
      question: "Select skills from page 3.",
      multiple: true,
      options: [ /* remaining skills */ ]
    }
  ]
})
```

For category selection, ask all category tabs at once:

```text
question({
  questions: [
    // Example only. Replace these with categories from Get-CentralSkillsFresh.
    { header: "creative", question: "Select creative skills.", multiple: true, options: [ /* creative skills */ ] },
    { header: "tools", question: "Select tools skills.", multiple: true, options: [ /* tools skills */ ] },
    { header: "research", question: "Select research skills.", multiple: true, options: [ /* research skills */ ] }
  ]
})
```

After the skill selection is complete, ask targets:

```text
question({
  questions: [{
    header: "Targets",
    question: "Where should selected skills be linked?",
    multiple: true,
    options: [
      { label: "OpenCode-Global", description: "~/.config/opencode/skills" },
      { label: "Claude-Global", description: "~/.claude/skills" },
      { label: "OpenCode-Project", description: "<repo>/.opencode/skills" },
      { label: "Claude-Project", description: "<repo>/.claude/skills" }
    ]
  }]
})
```

For final confirmation:

```text
question({
  questions: [{
    header: "Confirm link",
    question: "Create these links/copies?\n<plan table>",
    multiple: false,
    options: [
      { label: "Proceed", description: "Apply the plan." },
      { label: "Change selection", description: "Go back and choose skills or targets again." },
      { label: "Cancel", description: "Do nothing." }
    ]
  }]
})
```

### Fallback behavior

If `question` is unavailable, render a numbered menu in plain text using the same options and the same pagination rules. Accept typed numbers, `*`, `<category>/*`, `<category>/<skill>`, or comma-separated values.

Always present the planned actions (mode + source -> destination, per skill, per target) and ask for explicit confirmation before any filesystem mutation.

## Linking Modes (Junction vs Symlink vs Copy vs Skip)

There are four operation modes per `(skill, target)` pair:

### Junction (preferred for local directories)

- Created via `cmd /c mklink /J "<targetPath>" "<sourcePath>"`.
- Does **not** require admin or Windows Developer Mode.
- Edits in the central repo are visible immediately at the destination.
- Works for **directories** on local Windows paths, including cross-drive local NTFS paths.

### Directory Symlink (explicit override only)

- Created via `cmd /c mklink /D "<targetPath>" "<sourcePath>"`.
- Also works across drives, but is usually unnecessary here because `mklink /J` already covers local directory links.
- Requires Windows Developer Mode **on** (or running as admin). Without Dev Mode, `mklink /D` fails with "You do not have sufficient privilege".
- Edits in the central repo are visible immediately at the destination, just like junctions.
- Detection: check `(Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock' -ErrorAction SilentlyContinue).AllowDevelopmentWithoutDevLicense -eq 1`.

### Copy (fallback when junction is not appropriate, or user explicitly requests)

- Performed via `robocopy "<sourcePath>" "<targetPath>" /MIR /NFL /NDL /NJH /NJS`.
- `/MIR` mirrors the source - including deletions on subsequent runs.
- Works for network / UNC / non-local targets, no admin or Dev Mode required.
- Disadvantage: must be re-run after every central-repo update; not a "live" link.

### Skip

- Used when destination is already a junction or symlink to the exact same source. Report `Already linked, no-op.` and continue.

## Windows Decision Tree

For each `(skill, target)` pair:

1. Resolve `sourcePath = $env:USERPROFILE\.skills\<category>\<skill-name>`.
2. Resolve `targetParent` (one of the four target paths). Ensure it exists; create with `New-Item -ItemType Directory -Path <targetParent> -Force` if missing.
3. Resolve `targetPath = <targetParent>\<skill-name>`.
4. Detect environment **once** per session if `symlink` might be used:
   - `$devMode = ((Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock' -ErrorAction SilentlyContinue).AllowDevelopmentWithoutDevLicense -eq 1)`.
5. Determine `Mode`:
   - If user passed an explicit `-Mode junction|symlink|copy`, honor it (warn if their choice is impossible: e.g. `symlink` without Dev Mode/admin).
   - Else (`auto`):
     - If either path is network / UNC / otherwise unsuitable for junction -> `copy`.
     - Else -> `junction`.
6. Inspect `targetPath` (see Conflict Handling) and decide: create / skip / overwrite-after-confirm.
7. Execute the planned action (see PowerShell Reference Snippets).
8. Verify (see Verification).

## Workflow

Follow these steps in order:

1. **Discover** skills as described in `Discovery`.
2. **Collect skill selection first** using `question` pickers as described in `Interactive Selection UX`. Do not force the user to manually type paths unless they choose `Type manually` or `question` is unavailable.
3. **Collect targets after skills are selected** using a `question` picker. For project targets, walk up from `Get-Location` looking for `.git`, `package.json`, `pom.xml`, or `AGENTS.md`. If not found, ask for an absolute path.
4. **Plan**: build a table of `(skill, target, mode, conflict-action)` rows. Print it.
5. **Confirm with `question`**: show the plan and ask `Proceed / Change selection / Cancel`. Do not proceed unless the user selects `Proceed` or explicitly says yes.
6. **Execute** row by row. Stop on first error and report.
7. **Verify** every successful row (see Verification).
8. **Summarize**: print a final table of `OK / Skipped / Failed`.

## Conflict Handling

For each `targetPath`, before mutating:

- Use `Get-Item -LiteralPath $targetPath -ErrorAction SilentlyContinue` then check `.LinkType` and `.Target`. `LinkType` may be `'Junction'`, `'SymbolicLink'`, or `$null` (regular directory).
- Cases:

| Existing state at targetPath                                   | Action                                                                                                 |
|----------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Does not exist                                                 | Proceed with chosen mode.                                                                              |
| Junction or SymbolicLink whose target == sourcePath            | Report `Already linked, no-op.` Skip.                                                                  |
| Junction or SymbolicLink whose target != sourcePath            | Ask: **Overwrite / Skip / Cancel**.                                                                    |
| Regular directory                                              | Ask: **Overwrite-with-link / Overwrite-with-copy-mirror / Skip / Cancel**.                             |
| Regular file with same name                                    | Ask: **Overwrite / Skip / Cancel** (highly unusual - flag prominently).                                |

To overwrite (only after explicit user confirmation), choose the removal command by destination type:

```powershell
$existing = Get-Item -LiteralPath $targetPath -Force
switch ($existing.LinkType) {
    'Junction'     { cmd /c rmdir "$targetPath" }      # deletes link only, source untouched
    'SymbolicLink' { cmd /c rmdir "$targetPath" }      # deletes link only, source untouched
    default        { Remove-Item -LiteralPath $targetPath -Recurse -Force }  # regular dir
}
```

Then create the junction / symlink / copy as planned.

**Never** call `rm`. **Never** auto-overwrite. **Never** use `Remove-Item -Recurse` on a junction or symlink; use `cmd /c rmdir "<path>"` instead. Always re-confirm with the user verbally before issuing any removal.

## Permissions Note

- **Junction** (`mklink /J`): works for any local user on Windows 10/11 with **no admin** and **no Developer Mode**. It is the default for local directory targets, including cross-drive local NTFS paths.
- **Directory symlink** (`mklink /D`): requires **Developer Mode on** (Settings → Privacy & security → For developers) or an elevated/admin shell. Usually unnecessary here, but available as an explicit override. Detection: `(Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock' -ErrorAction SilentlyContinue).AllowDevelopmentWithoutDevLicense -eq 1`.
- **Robocopy mirror**: needs no special permissions for user-owned directories. Used when a junction is not appropriate (for example network / UNC targets) or when the user explicitly chooses copy mode.
- If a `mklink /J` call fails with "Access is denied", the cause is almost always a stale entry at the destination, not a privilege issue. Inspect the destination, remove appropriately (use `cmd /c rmdir` for an existing link), and retry.
- If a `mklink /D` call fails with "You do not have sufficient privilege", Dev Mode is off (or registry not readable). Either enable Dev Mode, run the shell as admin, or fall back to copy mode.

## PowerShell Reference Snippets

All snippets are copy-pasteable for PowerShell 5.1, use `-LiteralPath`, and quote every path. Replace placeholders in angle brackets.

### Resolve paths

```powershell
$skillsRoot   = Join-Path $env:USERPROFILE '.skills'
$category     = '<category>'
$skillName    = '<skill-name>'
$sourcePath   = Join-Path $skillsRoot (Join-Path $category $skillName)

# Targets:
$targets = @{
    'OpenCode-Global'  = Join-Path $env:USERPROFILE '.config\opencode\skills'
    'Claude-Global'    = Join-Path $env:USERPROFILE '.claude\skills'
    'OpenCode-Project' = Join-Path '<repo-root>' '.opencode\skills'
    'Claude-Project'   = Join-Path '<repo-root>' '.claude\skills'
}
$targetParent = $targets['<Target>']
$targetPath   = Join-Path $targetParent $skillName
```

### Freshly discover central skills

Run this immediately before building `All skills`, `Individual skills`, or `By category` options. Do not cache the result across picker stages.

```powershell
function Get-SkillDescription {
    param([Parameter(Mandatory)] [string] $SkillMd)
    $lines = Get-Content -LiteralPath $SkillMd -TotalCount 30 -Encoding UTF8
    foreach ($line in $lines) {
        if ($line -match '^\s*description\s*:\s*(.+?)\s*$') {
            return $Matches[1].Trim('"').Trim("'")
        }
    }
    return ''
}

function Get-CentralSkillsFresh {
    param([string] $SkillsRoot = (Join-Path $env:USERPROFILE '.skills'))

    if (-not (Test-Path -LiteralPath $SkillsRoot -PathType Container)) {
        throw "Central skills repo not found: $SkillsRoot"
    }

    $rows = New-Object System.Collections.Generic.List[object]
    $categories = Get-ChildItem -LiteralPath $SkillsRoot -Directory |
        Where-Object { -not $_.Name.StartsWith('.') -and -not $_.Name.StartsWith('_') }

    foreach ($category in $categories) {
        $skillDirs = Get-ChildItem -LiteralPath $category.FullName -Directory |
            Where-Object { -not $_.Name.StartsWith('.') -and -not $_.Name.StartsWith('_') }
        foreach ($skillDir in $skillDirs) {
            $skillMd = Join-Path $skillDir.FullName 'SKILL.md'
            if (-not (Test-Path -LiteralPath $skillMd -PathType Leaf)) {
                Write-Warning ("{0}/{1}: SKILL.md missing - skipped" -f $category.Name, $skillDir.Name)
                continue
            }
            # Hide link-skills from interactive pickers to avoid self-linking noise.
            if ($category.Name -eq 'tools' -and $skillDir.Name -eq 'link-skills') { continue }

            $rows.Add([pscustomobject]@{
                Id          = ("{0}/{1}" -f $category.Name, $skillDir.Name)
                Category    = $category.Name
                SkillName   = $skillDir.Name
                SourcePath  = $skillDir.FullName
                Description = Get-SkillDescription -SkillMd $skillMd
            })
        }
    }
    return @($rows | Sort-Object Id)
}
```

### Find repo root by walking up

```powershell
function Find-RepoRoot {
    param([string]$Start = (Get-Location).Path)
    $dir = Get-Item -LiteralPath $Start
    while ($null -ne $dir) {
        foreach ($marker in '.git','package.json','pom.xml','AGENTS.md') {
            if (Test-Path -LiteralPath (Join-Path $dir.FullName $marker)) {
                return $dir.FullName
            }
        }
        $dir = $dir.Parent
    }
    return $null  # caller should ask the user
}
```

### Ensure target parent exists

```powershell
if (-not (Test-Path -LiteralPath $targetParent)) {
    New-Item -ItemType Directory -Path $targetParent -Force | Out-Null
}
```

### Detect local-vs-network target and Dev Mode (only for explicit symlink)

```powershell
function Test-IsUncPath {
    param([string]$Path)
    return $Path.StartsWith('\\')
}

$targetIsUnc = Test-IsUncPath -Path $targetParent

# Dev Mode is only needed if the user explicitly forces symlink mode.
$devMode = ((Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock' -ErrorAction SilentlyContinue).AllowDevelopmentWithoutDevLicense -eq 1)
```

### Inspect existing destination

```powershell
$existing = Get-Item -LiteralPath $targetPath -ErrorAction SilentlyContinue
if ($existing) {
    $linkType        = $existing.LinkType   # 'Junction', 'SymbolicLink', or $null
    $isLink          = ($linkType -in 'Junction','SymbolicLink')
    $linkTarget      = if ($isLink) { ($existing.Target | Select-Object -First 1) } else { $null }
}
```

### Create a junction (default for local paths)

```powershell
cmd /c mklink /J "$targetPath" "$sourcePath"
# Verify
(Get-Item -LiteralPath $targetPath).LinkType  # expect 'Junction'
```

### Create a directory symlink (explicit override, requires Dev Mode)

```powershell
# Pre-flight: confirm Dev Mode is on, otherwise mklink /D fails with privilege error.
if (-not $devMode) { throw "Symlink requires Windows Developer Mode. Enable in Settings -> Privacy & security -> For developers." }

cmd /c mklink /D "$targetPath" "$sourcePath"
# Verify
(Get-Item -LiteralPath $targetPath).LinkType  # expect 'SymbolicLink'
```

### Mirror copy (UNC / non-local fallback, or user-requested)

```powershell
robocopy "$sourcePath" "$targetPath" /MIR /NFL /NDL /NJH /NJS
# robocopy exit codes 0..7 are success; >=8 is failure.
if ($LASTEXITCODE -ge 8) { throw "robocopy failed with exit code $LASTEXITCODE" }
```

### Auto-mode helper (encapsulates the decision tree)

```powershell
function Invoke-LinkSkill {
    param(
        [Parameter(Mandatory)] [string]$SourcePath,
        [Parameter(Mandatory)] [string]$TargetPath,
        [ValidateSet('auto','junction','symlink','copy')] [string]$Mode = 'auto'
    )
    $targetParent = Split-Path -Parent $TargetPath
    if (-not (Test-Path -LiteralPath $targetParent)) {
        New-Item -ItemType Directory -Path $targetParent -Force | Out-Null
    }
    $targetIsUnc = $targetParent.StartsWith('\\')
    $devMode = ((Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock' -ErrorAction SilentlyContinue).AllowDevelopmentWithoutDevLicense -eq 1)

    if ($Mode -eq 'auto') {
        if ($targetIsUnc) { $Mode = 'copy' }
        else              { $Mode = 'junction' }
    }

    switch ($Mode) {
        'junction' {
            cmd /c mklink /J "$TargetPath" "$SourcePath" | Out-Null
        }
        'symlink' {
            if (-not $devMode) { throw "Symlink requires Developer Mode (or admin)." }
            cmd /c mklink /D "$TargetPath" "$SourcePath" | Out-Null
        }
        'copy' {
            robocopy "$SourcePath" "$TargetPath" /MIR /NFL /NDL /NJH /NJS | Out-Null
            if ($LASTEXITCODE -ge 8) { throw "robocopy failed: $LASTEXITCODE" }
        }
    }
    return $Mode
}
```

### Remove (only after confirmation - branch by destination type)

```powershell
# Junctions, symlinks and regular directories need DIFFERENT removal commands.
$existing = Get-Item -LiteralPath $targetPath -Force
switch ($existing.LinkType) {
    'Junction' {
        # cmd rmdir removes the junction reparse point only; the source is untouched.
        cmd /c rmdir "$targetPath"
    }
    'SymbolicLink' {
        # Directory symlinks are also removable via cmd rmdir; the source is untouched.
        cmd /c rmdir "$targetPath"
    }
    default {
        # Regular directory: Remove-Item -Recurse is safe.
        Remove-Item -LiteralPath $targetPath -Recurse -Force
    }
}
```

## Verification

After every successful action, verify:

1. **Existence**: `Test-Path -LiteralPath $targetPath` is `$true`.
2. **Listing**: `Get-ChildItem -LiteralPath $targetParent | Where-Object Name -eq $skillName` returns one row.
3. **Junction marker** (when mode was junction):

   ```powershell
   (Get-Item -LiteralPath $targetPath).LinkType   # expect 'Junction'
   (Get-Item -LiteralPath $targetPath).Target     # expect path == $sourcePath
   ```
4. **SKILL.md readable from destination**:

   ```powershell
   Get-Content -LiteralPath (Join-Path $targetPath 'SKILL.md') -TotalCount 5
   ```
   First line should be `---` (YAML frontmatter).
5. **Tool discovery hint** - mention to the user:
   - For OpenCode-Global, the skill lives at `~/.config/opencode/skills/<skill-name>/SKILL.md`.
   - For Claude-Global, the skill lives at `~/.claude/skills/<skill-name>/SKILL.md`.
   - For project targets, the skill lives at `<repo-root>\.opencode\skills\<skill-name>\SKILL.md` or `<repo-root>\.claude\skills\<skill-name>\SKILL.md`.

## Examples

### Example 1 - Enable a single skill globally for OpenCode

User: "Enable `tools/link-skills` for OpenCode globally."

Plan:

```
tools/link-skills  ->  C:\Users\WZY\.config\opencode\skills\link-skills   [junction, same volume C:]
```

Confirm, then:

```powershell
$src = Join-Path $env:USERPROFILE '.skills\tools\link-skills'
$dstParent = Join-Path $env:USERPROFILE '.config\opencode\skills'
if (-not (Test-Path -LiteralPath $dstParent)) {
    New-Item -ItemType Directory -Path $dstParent -Force | Out-Null
}
cmd /c mklink /J "$(Join-Path $dstParent 'link-skills')" "$src"
```

Verify, then summarize.

### Example 2 - Enable an entire category for both Claude and OpenCode in a project

User: "Link all `creative/*` skills into this project for both OpenCode and Claude."

1. Resolve repo root via `Find-RepoRoot`.
2. Discover all `creative/*` folders containing `SKILL.md`.
3. Build plan: 2 targets × N skills, each row defaulting to junction for local directories or copy for UNC / non-local targets.
4. Confirm, execute, verify.

### Example 3 - Cross-drive local linking

User: "Link `research/literature-review` to a project on `E:\work\paper`."

`sourcePath` is on `C:`, `targetParent` is on `E:`. Because both are local directory paths, `auto` still resolves to a junction:

```powershell
cmd /c mklink /J "$targetPath" "$sourcePath"
```

This is a real live link. Edits in `~/.skills/...` are immediately visible at the target.

Only if the target were UNC / network / otherwise unsuitable for junction would `auto` fall back to:

```powershell
robocopy "$sourcePath" "$targetPath" /MIR /NFL /NDL /NJH /NJS
```

The user can also force a specific mode with `-Mode junction|symlink|copy` to bypass the auto decision.

### Example 4 - Conflict: existing junction to a different source

Destination already exists as a junction pointing to an old path. Ask:

```
Overwrite / Skip / Cancel ?
```

If Overwrite, remove the existing junction first with `cmd /c rmdir "$targetPath"` (this deletes the junction reparse point only, never the source), then create the new junction with `cmd /c mklink /J "$targetPath" "$sourcePath"`.

## Anti-Patterns

- Linking the **same skill** both globally and project-locally with **different versions** - causes confusing precedence.
- Linking a folder that is **not** under `$env:USERPROFILE\.skills` - this skill operates only on the central repo. For external folders, use `install-skill` first.
- Defaulting to **symbolic links** (`mklink` without `/J`, or `New-Item -ItemType SymbolicLink`) - they require Developer Mode or admin and will fail silently for many users.
- Calling `rm` (forbidden in this system) or `Remove-Item` without explicit user confirmation, or without `-LiteralPath`.
- Auto-overwriting a regular directory at the destination - always confirm first.
- Hardcoding `C:\Users\<name>\...` paths - always derive from `$env:USERPROFILE`.
- Using `Remove-Item -Recurse -Force` on a junction. PS 5.1 behavior has been inconsistent across versions and can risk traversal. Use `cmd /c rmdir "<path>"` for junctions and reserve `Remove-Item -Recurse` for regular directories.
- Forgetting to create the target parent directory before `mklink /J` - the parent must exist, the leaf must not.
