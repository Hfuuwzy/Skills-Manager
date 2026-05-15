---
name: install-skill
description: Install one or more selected skills from a GitHub URL, GitHub repo path, or local path into the user's central skills repository at ~/.skills/<category>/<skill-name>/. Use this when the user asks to install, add, fetch, or import skills from an external source, especially when a GitHub repository contains multiple SKILL.md files and the user should choose which skills to install. Windows-first design using PowerShell, gh CLI, ConvertFrom-Json (no jq), question pickers, and robocopy. Produces usable skills in the central repo without enabling them for any tool.
---

# install-skill

## Purpose

Fetch one or more selected skills from an external source (GitHub URL, GitHub shorthand, or local folder) and place each under the user's central skills repository:

```
$env:USERPROFILE\.skills\<category>\<skill-name>\
```

The skill ends up self-contained in the central repo. This skill does **not** wire the installed skill into any tool (OpenCode, Claude Code, Codex, Cursor). Linking is the responsibility of the sister skill `link-skills`. Auditing is the responsibility of the sister skill `audit-skills`. Stay in your lane.

## When to Use

Trigger this skill when the user asks to:

- Install / add / fetch / import / pull / get a skill.
- "Install this skill from GitHub: <url>"
- "Add the canvas-design skill from anthropics/skills"
- "Import the skill at D:\work\my-skill into my central repo"

Do NOT use this skill for:

- Enabling a skill for a specific tool. Use `link-skills`.
- Listing / validating / cleaning the central repo. Use `audit-skills`.
- Authoring a brand-new skill from scratch. Use `skill-creator`.

## Inputs

The agent should collect these inputs from the user, asking only for what is missing:

| Input        | Required | Notes                                                                                                                      |
|--------------|----------|----------------------------------------------------------------------------------------------------------------------------|
| `Source`     | yes      | GitHub URL, `<owner>/<repo>[/<subpath>]` shorthand, or absolute local folder path.                                         |
| `Category`   | no       | One of: `ai`, `browser`, `creative`, `document`, `process`, `project`, `research`, `tools`, `writing`. If omitted, infer and confirm. New categories require explicit user confirmation. |
| `SkillName`  | no       | Override for the destination folder name. Defaults to the YAML `name:` field of `SKILL.md`, falling back to the source folder name. Sanitized to `[a-z0-9-]+`. |
| `Branch`     | no       | Optional git branch / ref. Defaults to the repo default branch.                                                            |
| `Subpath`    | no       | Path inside the repo where the skill lives. Inferred from URL `tree/<branch>/<subpath>` or shorthand `<owner>/<repo>/<subpath>`. |

Allowed categories (do not silently invent new ones):

```
ai, browser, creative, document, process, project, research, tools, writing
```

If the inferred or user-supplied category is not in this list, ask the user to confirm creating it before proceeding.

`SkillName` override is allowed only when installing exactly one selected skill. If multiple skills are selected, use each skill's frontmatter `name:` (fallback: folder name) and sanitize independently.

## Resolution Rules

Source resolution is strict and deterministic:

1. **Full GitHub URL**, e.g. `https://github.com/<owner>/<repo>` or `https://github.com/<owner>/<repo>/tree/<branch>/<subpath>`:
   - Parse `<owner>`, `<repo>`, optional `<branch>`, optional `<subpath>`.
   - Prefer `gh repo clone <owner>/<repo> <staging> -- --depth 1 --branch <branch>`.
   - Fallback: `git clone --depth 1 --branch <branch> https://github.com/<owner>/<repo>.git <staging>`.
   - If `<branch>` is unknown, omit `--branch` and rely on the repo default.
   - If `<subpath>` is given, the effective skill root is `<staging>\<subpath>`.

2. **GitHub shorthand**, e.g. `<owner>/<repo>` or `<owner>/<repo>/<subpath>`:
   - Treat as the URL above with no explicit branch.

3. **Local absolute path** that resolves to an existing directory containing (directly or nested) a `SKILL.md`:
   - Stage with robocopy:
     ```powershell
     robocopy "<src>" "<staging>" /MIR /NFL /NDL /NJH /NJS
     ```
   - Robocopy exit codes 0-7 are success; 8+ are failures.

4. **Anything else** (file path, raw zip URL, http URL that is not GitHub, missing path): stop and ask the user to clarify.

Staging directory:

```powershell
$staging = Join-Path $env:TEMP (".skills-staging\" + [System.Guid]::NewGuid().ToString("N"))
New-Item -ItemType Directory -Force -Path $staging | Out-Null
```

## Workflow

Follow these steps in order. Mark each as a todo before executing.

1. **Parse inputs.** Decide whether `Source` is GitHub URL, GitHub shorthand, or local path. Extract `Owner`, `Repo`, `Branch`, `Subpath` where applicable. Quote every path you build.
2. **Create staging dir** under `$env:TEMP\.skills-staging\<guid>`.
3. **Fetch source into staging.**
   - GitHub: `gh repo clone` first, then `git clone --depth 1` fallback. Capture exit code and stderr.
   - Local: `robocopy` mirror. Treat exit code `<= 7` as success.
4. **Resolve effective skill root.** If a `Subpath` was provided, descend into `<staging>\<subpath>`. Verify it exists.
5. **Discover skill candidates.** Locate every case-sensitive `SKILL.md` within depth `<= 4` of the effective root using `Get-ChildItem -LiteralPath <root> -Recurse -Depth 4 -Filter SKILL.md -File | Where-Object { $_.Name -ceq 'SKILL.md' }`. Each parent folder is one candidate skill root. If zero matches, abort.
6. **Build candidate metadata.** For every candidate, read frontmatter (`name:`, `description:`), infer sanitized `SkillName`, infer `Category`, compute relative path, and mark whether `$env:USERPROFILE\.skills\<category>\<skillName>` already exists.
7. **Interactive selection.** If more than one candidate exists, use the native `question` picker to let the user select one or more candidates. If only one candidate exists, show it and ask `Install / Cancel` unless the user already made an explicit single-skill request.
8. **Category confirmation.** For selected candidates, show inferred category and destination in the plan. If category confidence is low or category is new, ask the user to confirm/override before copying. If multiple selected skills need category choices, batch category-confirmation questions where practical.
9. **Compute install plan.** Build a table of `(source relative path, skillName, category, destination, conflict action)` for every selected skill.
10. **Conflict check.** For any destination that already exists and is not empty, run **Conflict Handling** per selected skill.
11. **Final confirmation.** Present the full install plan and ask `Proceed / Change selection / Cancel`. Do not mutate until the user confirms.
12. **Remove nested `.git` directories** from each selected skill root before copying, so the central repo never contains nested git metadata.
13. **Copy each selected skill into place** with robocopy mirror:
    ```powershell
    New-Item -ItemType Directory -Force -Path $dst | Out-Null
    robocopy "$skillRoot" "$dst" /MIR /NFL /NDL /NJH /NJS
    ```
14. **Run Verification (destination only)** for every installed skill. If any check fails, surface the error and stop with the partial result table.
15. **Cleanup staging** with `Remove-Item -Recurse -Force -LiteralPath $staging`.
16. **Verify staging cleanup**: `Test-Path -LiteralPath $staging` must be `$false`.
17. **Report** all installed destination paths, resolved categories, and skill names. Remind the user that enabling/linking into a tool requires running `link-skills`.

## Interactive Candidate Selection

Prefer OpenCode's native `question` tool when a GitHub/local source contains multiple candidate skills. Use plain numbered text only as a fallback when `question` is unavailable.

### Candidate picker rules

- Present candidates by relative path and inferred destination, not just by folder name.
- Label each option as `<relative-path>` or `<frontmatter-name> — <relative-path>` when names would otherwise collide.
- Description should include inferred category, sanitized destination name, and the first ~80 characters of frontmatter `description:`.
- Allow `multiple: true` so the user can install several skills from the same GitHub repo in one run.
- If there are more than about 15 candidates, split them into page tabs in a single `question` call: `Candidates 1/3`, `Candidates 2/3`, etc. Preserve selections across tabs.
- Do not install all candidates by default just because a repo contains many `SKILL.md` files. User selection is required unless the source path directly points to one skill folder.

Example candidate picker:

```text
question({
  questions: [
    {
      header: "Candidates 1/2",
      question: "Which skills from this source should be installed?",
      multiple: true,
      options: [
        { label: "document/pdf", description: "document/pdf -> document/pdf. PDF manipulation skill..." },
        { label: "tools/mcp-builder", description: "tools/mcp-builder -> tools/mcp-builder. Build MCP servers..." }
      ]
    },
    {
      header: "Candidates 2/2",
      question: "Select more skills from this source.",
      multiple: true,
      options: [ /* remaining candidates */ ]
    }
  ]
})
```

For final confirmation:

```text
question({
  questions: [{
    header: "Confirm install",
    question: "Install these skills into ~/.skills?\n<plan table>",
    multiple: false,
    options: [
      { label: "Proceed", description: "Copy selected skills into the central repo." },
      { label: "Change selection", description: "Go back to candidate selection." },
      { label: "Cancel", description: "Do nothing." }
    ]
  }]
})
```

## Category Inference

When the user did not supply `-Category`, infer one by scanning `description:` (and `name:` as a tiebreaker) with these case-insensitive keyword heuristics. Stop at the first match in this order:

| Keywords (any of)                                                | Category   |
|------------------------------------------------------------------|------------|
| `browser`, `playwright`, `selenium`, `puppeteer`, `web automation` | `browser`  |
| `canvas`, `art`, `poster`, `illustration`, `algorithmic art`, `vue`, `react`, `svelte`, `css`, `tailwind`, `ui`, `ux`, `frontend`, `component`, `spring`, `java`, `api`, `rest`, `backend`, `server`, `database`, `sql` | `creative` |
| `subagent`, `research session`, `workflow`, `pipeline`, `cleanup` | `process` |
| `install`, `link`, `audit`, `package`, `manager`, `cli tool`     | `tools`    |
| `rag`, `vector`, `embedding`, `search`, `retrieval`, `llm`, `prompt`, `mcp` | `ai`       |
| `paper`, `literature`, `citation`, `bibtex`, `pubmed`, `arxiv`, `peer review` | `research` |
| `doc`, `documentation`, `blog`, `markdown`, `note`, `report`, `weekly` | `writing`  |
| `pdf`, `docx`, `pptx`, `xlsx`, `spreadsheet`, `presentation`     | `document` |
| project-specific names, internal repos, no other match           | `project`  |

Always echo the inferred category to the user and request confirmation before final placement. If none match, ask the user.

## Conflict Handling

If destination `$dst` already exists and is not empty, present exactly three options:

1. **Overwrite** - mirror staged skill over the existing folder:
   ```powershell
   robocopy "$skillRoot" "$dst" /MIR /NFL /NDL /NJH /NJS
   ```
   Confirm with the user explicitly before doing this. Mirror deletes extra files.
2. **Rename** - suggest `<skillName>-2` (then `-3`, etc., picking the first that does not exist):
   ```powershell
   $candidate = "$skillName-2"
   while (Test-Path -LiteralPath (Join-Path $env:USERPROFILE ".skills\$category\$candidate")) {
       $n = [int]($candidate -replace '.*-(\d+)$','$1') + 1
       $candidate = ($skillName + "-$n")
   }
   ```
3. **Skip** - leave the existing destination unchanged, mark this selected skill as `Skipped (conflict, user chose Skip)`, and continue processing the remaining selected skills. Clean up staging only at the normal cleanup step.

Never silently overwrite. Never merge by hand.

## Windows Constraints

- **PowerShell 5.1** is the assumed shell. All commands must run there. Avoid PowerShell 7-only syntax.
- **No symlinks, no junctions, no hard links.** This skill only copies content. Symlinks/junctions belong to `link-skills`, not this skill, regardless of whether Windows Developer Mode is on.
- **No `jq`.** Parse JSON via `ConvertFrom-Json` if you ever need to (e.g. `gh api` output).
- **`gh` CLI is available** and preferred for GitHub fetches. `git.exe` is the fallback. `curl.exe` is available but unnecessary here.
- **No `rm`.** Use `Remove-Item -LiteralPath` (with `-Recurse -Force` only when justified). Never use bash aliases.
- **Quote every path** with double quotes, especially when it might contain spaces (`$env:USERPROFILE` may include `WZY`, but other accounts could include spaces).
- **`-LiteralPath` everywhere it is accepted.** Avoid `-Path` for user-supplied paths to dodge wildcard expansion.
- **Long paths.** If a destination path exceeds ~240 chars, prefix with the literal `\\?\` extended-length form, for example `\\?\C:\Users\<user>\.skills\<category>\<skill>\...`. Robocopy handles long paths natively; `Copy-Item` may not.
- **Robocopy exit codes.** Any value `<= 7` is success. Treat `>= 8` as failure.

## PowerShell Reference Snippets

These snippets are illustrative and runnable as written on PowerShell 5.1. Adapt variables; do not paste blindly.

### Setup paths

```powershell
$ErrorActionPreference = "Stop"
$skillsRoot = Join-Path $env:USERPROFILE ".skills"
$staging    = Join-Path $env:TEMP (".skills-staging\" + [System.Guid]::NewGuid().ToString("N"))
New-Item -ItemType Directory -Force -Path $staging | Out-Null
```

### Parse a GitHub URL

```powershell
function Resolve-GitHubSource {
    param([Parameter(Mandatory)] [string] $Source)
    $u = $Source.Trim()
    if ($u -match '^https?://github\.com/([^/]+)/([^/]+?)(?:\.git)?(?:/tree/([^/]+)(?:/(.+))?)?/?$') {
        return [pscustomobject]@{
            Owner = $Matches[1]; Repo = $Matches[2]
            Branch = $Matches[3]; Subpath = $Matches[4]
            Kind = "github"
        }
    }
    if ($u -match '^([A-Za-z0-9_.-]+)/([A-Za-z0-9_.-]+)(?:/(.+))?$') {
        return [pscustomobject]@{
            Owner = $Matches[1]; Repo = $Matches[2]
            Branch = $null; Subpath = $Matches[3]
            Kind = "github"
        }
    }
    if (Test-Path -LiteralPath $u -PathType Container) {
        return [pscustomobject]@{ Kind = "local"; Path = (Resolve-Path -LiteralPath $u).Path }
    }
    throw "Unrecognized source: $u"
}
```

### Clone (gh first, git fallback)

```powershell
function Invoke-CloneToStaging {
    param(
        [Parameter(Mandatory)] [string] $Owner,
        [Parameter(Mandatory)] [string] $Repo,
        [string] $Branch,
        [Parameter(Mandatory)] [string] $Staging
    )
    $remote = "$Owner/$Repo"
    $ghArgs = @("repo","clone",$remote,"$Staging","--","--depth","1")
    if ($Branch) { $ghArgs += @("--branch",$Branch) }
    $gh = Get-Command gh.exe -ErrorAction SilentlyContinue
    if ($gh) {
        & $gh.Path @ghArgs
        if ($LASTEXITCODE -eq 0) { return }
    }
    $gitArgs = @("clone","--depth","1")
    if ($Branch) { $gitArgs += @("--branch",$Branch) }
    $gitArgs += @("https://github.com/$Owner/$Repo.git","$Staging")
    & git.exe @gitArgs
    if ($LASTEXITCODE -ne 0) { throw "git clone failed for $remote" }
}
```

### Stage a local folder

```powershell
function Copy-LocalToStaging {
    param(
        [Parameter(Mandatory)] [string] $Src,
        [Parameter(Mandatory)] [string] $Staging
    )
    & robocopy "$Src" "$Staging" /MIR /NFL /NDL /NJH /NJS | Out-Null
    if ($LASTEXITCODE -ge 8) { throw "robocopy staging failed (code $LASTEXITCODE)" }
}
```

### Find SKILL.md

```powershell
function Find-SkillMd {
    # Depth 4 covers common GitHub layouts:
    #   <repo>/SKILL.md
    #   <repo>/<group>/SKILL.md                (e.g. document-skills/SKILL.md)
    #   <repo>/<group>/<skill>/SKILL.md        (e.g. document-skills/pdf/SKILL.md)
    #   <repo>/skills/<group>/<skill>/SKILL.md (e.g. xhyqaq/skill-manager)
    # Case-sensitive match prevents picking up `skill.md` or `Skill.MD`.
    param([Parameter(Mandatory)] [string] $Root)
    $hits = Get-ChildItem -LiteralPath $Root -Recurse -Depth 4 -Filter SKILL.md -File `
        | Where-Object { $_.Name -ceq "SKILL.md" }
    return ,$hits
}
```

### Read frontmatter `name:` and `description:`

```powershell
function Read-SkillFrontmatter {
    param([Parameter(Mandatory)] [string] $SkillMdPath)
    $lines = Get-Content -LiteralPath $SkillMdPath -Encoding UTF8
    if ($lines.Count -lt 2 -or $lines[0].Trim() -ne "---") {
        return [pscustomobject]@{ Name = $null; Description = $null }
    }
    $end = -1
    for ($i = 1; $i -lt $lines.Count; $i++) {
        if ($lines[$i].Trim() -eq "---") { $end = $i; break }
    }
    if ($end -lt 0) { return [pscustomobject]@{ Name = $null; Description = $null } }
    $body = $lines[1..($end-1)]
    $name = $null; $desc = $null
    foreach ($l in $body) {
        if (-not $name -and $l -match '^\s*name\s*:\s*(.+?)\s*$')        { $name = $Matches[1].Trim('"').Trim("'") }
        if (-not $desc -and $l -match '^\s*description\s*:\s*(.+?)\s*$') { $desc = $Matches[1].Trim('"').Trim("'") }
    }
    return [pscustomobject]@{ Name = $name; Description = $desc }
}
```

### Sanitize a skill name

```powershell
function Get-SafeSkillName {
    param([Parameter(Mandatory)] [string] $Raw)
    $s = $Raw.ToLowerInvariant()
    $s = ($s -replace '[^a-z0-9-]', '-')
    $s = ($s -replace '-+', '-').Trim('-')
    if (-not $s) { throw "Skill name reduced to empty after sanitization" }
    return $s
}
```

### Final copy

```powershell
function Copy-IntoCentralRepo {
    param(
        [Parameter(Mandatory)] [string] $SkillRoot,
        [Parameter(Mandatory)] [string] $Category,
        [Parameter(Mandatory)] [string] $SkillName
    )
    $dst = Join-Path $env:USERPROFILE ".skills\$Category\$SkillName"
    New-Item -ItemType Directory -Force -Path $dst | Out-Null
    & robocopy "$SkillRoot" "$dst" /MIR /NFL /NDL /NJH /NJS | Out-Null
    if ($LASTEXITCODE -ge 8) { throw "robocopy install failed (code $LASTEXITCODE)" }
    return $dst
}
```

### Cleanup staging

```powershell
if (Test-Path -LiteralPath $staging) {
    Remove-Item -Recurse -Force -LiteralPath $staging
}
```

## Verification

After copying, run all of these **per installed skill**. Stop and report the first failing skill, but keep already-verified skills in the success column.

1. **Destination exists.**
   ```powershell
   Test-Path -LiteralPath (Join-Path $env:USERPROFILE ".skills\$category\$skillName")
   ```
2. **`SKILL.md` present at destination root.**
   ```powershell
   Test-Path -LiteralPath (Join-Path $dst "SKILL.md")
   ```
3. **Frontmatter parses.** Re-run `Read-SkillFrontmatter` on the destination `SKILL.md`. Both `name:` and `description:` must be non-empty.
4. **No nested `.git`** anywhere under destination:
   ```powershell
   @(Get-ChildItem -LiteralPath $dst -Recurse -Force -Directory |
       Where-Object { $_.Name -eq ".git" }).Count -eq 0
   ```
5. **Category folder is in the allowed list** (or was confirmed as a new category by the user).

When all selected skills have been processed, print a single per-skill summary table:

```
Install summary
---------------
Skill                 Category   Destination                                                       Status
--------------------- ---------- ----------------------------------------------------------------- ------
pdf                   document   C:\Users\WZY\.skills\document\pdf                                 OK
docx                  document   C:\Users\WZY\.skills\document\docx                                OK
mcp-builder           tools      C:\Users\WZY\.skills\tools\mcp-builder                            Skipped (conflict, user chose Skip)
peer-review           research   C:\Users\WZY\.skills\research\peer-review                         Failed (robocopy 16)
```

Status values: `OK`, `Skipped (<reason>)`, `Failed (<reason>)`. Always include the count of OK / Skipped / Failed at the bottom, e.g. `2 OK, 1 Skipped, 1 Failed`.

Then remind the user:

```
Next step: run the link-skills skill to enable the OK rows for OpenCode / Claude / Codex.
```

Never claim success for a skill that is not in the OK column.

## Failure Handling

- **`gh` not found and `git` not found** -> abort with "Install Git for Windows or GitHub CLI to fetch from GitHub."
- **`gh repo clone` fails** -> retry once with `git clone --depth 1`. If still failing, report stderr and stop.
- **Robocopy exit code `>= 8`** (during staging or any per-skill copy) -> report the code together with the source/destination paths. Do not retry blindly. If only some skills failed, keep already-installed skills in place and surface a partial result table.
- **No `SKILL.md` found** under the effective skill root -> abort. Do not invent one.
- **Multiple `SKILL.md` files found** -> this is the normal multi-skill case. Build candidate metadata (Step 6) and run the interactive selection picker (Step 7). Never auto-install all of them; never silently pick the first. If the user picks zero candidates, treat it as Cancel and clean up staging.
- **User picks zero candidates in the picker** -> treat as Cancel: skip Steps 8-14, clean up staging, report no-op.
- **Frontmatter missing `name:`** -> fall back to that candidate's source folder name, sanitize, and warn the user. If sanitization yields an empty string, skip that candidate and report it in the failure column.
- **Two selected candidates sanitize to the same `<category>/<skillName>`** -> stop before any copy and ask the user to disambiguate (rename one, drop one, or assign different categories).
- **Destination conflict** -> apply Conflict Handling per skill. Never silently overwrite, never mass-overwrite.
- **Long path errors** -> prefix paths with `\\?\` and retry the robocopy step for that skill only.
- Always clean up `$staging` in a `finally`-equivalent block (PowerShell `try { ... } finally { ... }`), even on partial failure.

## Examples

### Example 1: Install from a full GitHub URL (single skill)

User: "Install https://github.com/anthropics/skills/tree/main/document-skills/pdf into my central repo."

Agent steps:

1. Parse -> `Owner=anthropics`, `Repo=skills`, `Branch=main`, `Subpath=document-skills/pdf`.
2. Stage at `$env:TEMP\.skills-staging\<guid>`.
3. `gh repo clone anthropics/skills "$staging" -- --depth 1 --branch main`.
4. `$skillRoot = Join-Path $staging "document-skills\pdf"`.
5. `Find-SkillMd $skillRoot` -> finds exactly one `SKILL.md` directly in `pdf\`.
6. Build candidate metadata for the one hit. Frontmatter `name: pdf`, description mentions PDF/forms.
7. Single-candidate path: show the candidate and ask `Install / Cancel`. User picks Install.
8. Heuristic -> category `document`. Confirm with user.
9. Compute install plan, conflict check passes (folder absent). Final `Proceed / Change selection / Cancel` confirmation.
10. Remove nested `.git` from `$skillRoot` (none here) and copy with robocopy.
11. Per-skill verification passes. Cleanup staging. Print 1-row summary table (`1 OK, 0 Skipped, 0 Failed`).

### Example 2: Install from GitHub shorthand (single skill, name override)

User: "Add `xhyqaq/skill-manager/skills/audit-skill` as `audit-skills` under `tools`."

Agent steps:

1. Parse -> `Owner=xhyqaq`, `Repo=skill-manager`, `Subpath=skills/audit-skill`, no branch.
2. Stage and `gh repo clone xhyqaq/skill-manager "$staging" -- --depth 1`.
3. `$skillRoot = Join-Path $staging "skills\audit-skill"`. Verify exists.
4. `Find-SkillMd $skillRoot` -> exactly one `SKILL.md`. Read frontmatter -> `name: audit-skill`.
5. Single-candidate path. Because the user passed `-SkillName audit-skills` and `-Category tools`, skip inference but still ask `Install / Cancel`.
6. Sanitize -> `audit-skills`. Destination: `$env:USERPROFILE\.skills\tools\audit-skills`. The `-SkillName` override is allowed because exactly one candidate is selected.
7. Final confirmation. Copy, per-skill verify, cleanup.

### Example 3: Install from a local folder (single skill)

User: "Install the skill at `D:\work\my-rag-skill` into the central repo."

Agent steps:

1. `Source` resolves as a local path. Confirm directory exists.
2. Robocopy mirror into staging.
3. `Find-SkillMd <staging>` -> exactly one `SKILL.md` at `<staging>\SKILL.md`. Frontmatter `name: rag-helper`, description mentions vectors and retrieval.
4. Single-candidate path. User picks Install.
5. Heuristic -> category `ai`. Confirm with user.
6. Sanitize -> `rag-helper`. Destination: `$env:USERPROFILE\.skills\ai\rag-helper`.
7. Final confirmation. Copy, per-skill verify, cleanup.

### Example 4: Selective multi-skill install from a multi-skill GitHub repo

User: "Install some skills from https://github.com/anthropics/skills."

Agent steps:

1. Parse -> `Owner=anthropics`, `Repo=skills`, no `Subpath`.
2. Stage and `gh repo clone anthropics/skills "$staging" -- --depth 1`.
3. `$skillRoot = $staging`.
4. `Find-SkillMd $skillRoot` -> several hits, e.g. `document-skills/pdf/SKILL.md`, `document-skills/docx/SKILL.md`, `document-skills/pptx/SKILL.md`, `document-skills/xlsx/SKILL.md`, `mcp-builder/SKILL.md`, `webapp-testing/SKILL.md`, ...
5. Build candidate metadata for every hit (relative path, frontmatter `name:`/`description:`, sanitized SkillName, inferred Category, conflict flag).
6. Interactive selection. Because there are >15 candidates, paginate into a single batched `question` call with tabs `Candidates 1/2`, `Candidates 2/2`. User picks `document-skills/pdf`, `document-skills/docx`, `mcp-builder`. `-SkillName` override is rejected here because more than one candidate is selected; each skill keeps its own frontmatter `name:`.
7. Category confirmation, batched: `pdf` -> `document`, `docx` -> `document`, `mcp-builder` -> `tools`. User confirms in one batched question.
8. Compute install plan:
   ```
   Source                       SkillName     Category   Destination                                       Conflict
   document-skills/pdf          pdf           document   ~/.skills/document/pdf                            (none)
   document-skills/docx         docx          document   ~/.skills/document/docx                           Overwrite (existing)
   mcp-builder                  mcp-builder   tools      ~/.skills/tools/mcp-builder                       (none)
   ```
9. Conflict handling: for `docx`, ask Overwrite / Rename / Skip. User picks Overwrite, re-confirms.
10. Final `Proceed / Change selection / Cancel`. User picks Proceed.
11. For each selected skill: remove nested `.git` under that skill root, robocopy `/MIR` into destination, run per-skill verification.
12. Cleanup staging. Print summary table:
    ```
    Install summary
    ---------------
    Skill         Category   Destination                                Status
    ------------- ---------- ------------------------------------------ ------
    pdf           document   C:\Users\WZY\.skills\document\pdf          OK
    docx          document   C:\Users\WZY\.skills\document\docx         OK (overwrite)
    mcp-builder   tools      C:\Users\WZY\.skills\tools\mcp-builder     OK

    3 OK, 0 Skipped, 0 Failed
    ```
13. Remind the user to run `link-skills` to enable any of these for OpenCode / Claude / Codex.

### Example 5: Conflict resolution (Rename)

User: "Install `anthropics/skills/document-skills/pdf` again."

Agent steps:

1. Resolve as before. Single candidate. Destination `$env:USERPROFILE\.skills\document\pdf` already exists.
2. Present the user with Overwrite / Rename / Skip.
3. User picks Rename. Compute next free name: `pdf-2`. Destination becomes `$env:USERPROFILE\.skills\document\pdf-2`.
4. Final `Proceed / Change selection / Cancel`. Copy, per-skill verify, cleanup. Report both the original (untouched) and the new path in the summary table.

### Example 6: Conflict resolution (Overwrite, with confirmation)

User: "Reinstall `audit-skills` from the latest main, overwrite my local copy."

Agent steps:

1. Stage and clone as in Example 2.
2. Single candidate. Conflict detected at `$env:USERPROFILE\.skills\tools\audit-skills`.
3. Re-confirm with the user that Overwrite mirrors and deletes extras.
4. After confirmation, robocopy `/MIR` over the existing folder.
5. Per-skill verify, cleanup. Remind the user that linking is handled by `link-skills`.

---

Boundary reminder: this skill stops after the file is in the central repo. It does not edit `opencode.json`, `~/.claude/skills`, Codex configs, or any tool registry. Hand off to `link-skills` for that.
