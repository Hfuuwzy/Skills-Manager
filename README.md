# Skill Manager

管理 OpenCode / Claude Code / Codex CLI 的 skills 安装和链接。

## 解决什么问题

当你积累了大量 skills 后，会面临以下痛点：

1.  **Skills 散落各处** — 每个项目的 `.opencode/skills/` 或 `.claude/skills/` 里都有副本，更新一个 skill 需要逐个项目同步
2.  **新项目配置繁琐** — 每次开新项目都要手动复制或重新下载 skills
3.  **没有统一管理** — 不知道自己装了哪些 skills，分类混乱，找不到想用的

**Skill Manager 的方案**：用 `~/.skills/` 作为唯一的 source of truth，按分类组织所有 skills，然后通过软链接按需挂载到各个项目。安装一次，处处可用。

## 包含的 Skills

| Skill | 用途 |
|-------|------|
| `install-skill` | 从 GitHub URL、本地路径安装 skill 到 `~/.skills/<category>/` |
| `link-skills` | 将 `~/.skills` 中的 skill 软链接或复制到项目或全局目录 |

### install-skill

- 支持从 GitHub URL、GitHub 简写格式（`owner/repo`）、本地路径安装
- 自动发现仓库中的多个 SKILL.md，支持交互式选择
- 自动推断 skill 分类（ai、browser、creative、document、process、project、research、tools、writing）
- Windows 优先设计，使用 PowerShell、gh CLI、robocopy
- 支持冲突处理（覆盖/重命名/跳过）

### link-skills

- 支持四个目标位置：
  - **OpenCode-Global**: `~/.config/opencode/skills`
  - **Claude-Global**: `~/.claude/skills`
  - **OpenCode-Project**: `<repo-root>/.opencode/skills`
  - **Claude-Project**: `<repo-root>/.claude/skills`
- 自动模式：优先使用目录联接（junction），无需开发者模式
- 支持网络/UNC 路径的 robocopy 镜像复制
- 交互式选择：全部技能、按分类、单个选择、手动输入

## 安装

### 方法 1：手动下载

```powershell
# 用 gh CLI 下载到 ~/.skills/tools/
gh repo clone Hfuuwzy/Skills-Manager "$env:TEMP\skill-manager-tmp"
Move-Item "$env:TEMP\skill-manager-tmp\install-skill" "$env:USERPROFILE\.skills\tools\"
Move-Item "$env:TEMP\skill-manager-tmp\link-skills" "$env:USERPROFILE\.skills\tools\"
Remove-Item -Recurse -Force "$env:TEMP\skill-manager-tmp"
```

### 方法 2：已有 install-skill 时

```powershell
# 如果你有其他方式已安装 install-skill
/install-skill https://github.com/Hfuuwzy/Skills-Manager
```

### 链接到全局 CLI

将这两个 skill 链接到 OpenCode 或 Claude 的全局目录：

```powershell
# OpenCode 全局
$ocGlobal = "$env:USERPROFILE\.config\opencode\skills"
if (-not (Test-Path $ocGlobal)) { New-Item -ItemType Directory -Path $ocGlobal -Force }
cmd /c mklink /J "$ocGlobal\install-skill" "$env:USERPROFILE\.skills\tools\install-skill"
cmd /c mklink /J "$ocGlobal\link-skills" "$env:USERPROFILE\.skills\tools\link-skills"

# Claude 全局
$claudeGlobal = "$env:USERPROFILE\.claude\skills"
if (-not (Test-Path $claudeGlobal)) { New-Item -ItemType Directory -Path $claudeGlobal -Force }
cmd /c mklink /J "$claudeGlobal\install-skill" "$env:USERPROFILE\.skills\tools\install-skill"
cmd /c mklink /J "$claudeGlobal\link-skills" "$env:USERPROFILE\.skills\tools\link-skills"
```

## 使用

### 安装新 skill

在 OpenCode 或 Claude 中：

```
/install-skill https://github.com/anthropics/skills
```

支持以下格式：
- 完整 URL: `https://github.com/owner/repo`
- 带分支和子路径: `https://github.com/owner/repo/tree/main/subfolder`
- GitHub 简写: `owner/repo` 或 `owner/repo/subpath`
- 本地路径: `D:\work\my-skill`

### 链接已安装的 skill

```
/link-skills
```

交互式流程：
1. 选择范围：全部技能 / 按分类选择 / 单个选择 / 手动输入
2. 选择目标：OpenCode 全局 / Claude 全局 / 项目级 OpenCode / 项目级 Claude
3. 确认并执行

## 目录结构

```
~/.skills/                  # 统一技能库（source of truth）
├── ai/                     # AI 相关（RAG、LLM、MCP 等）
├── browser/                # 浏览器自动化类
├── creative/               # 设计与可视化类
├── document/               # 文档处理类（PDF、Word、Excel、PPT）
├── process/                # 工作流方法论类
├── project/                # 项目特定 skills
├── research/               # 学术研究类
├── tools/                  # 工具类
│   ├── install-skill/
│   └── link-skills/
└── writing/                # 写作与文档类
```

## 平台支持

| 平台 | 状态 | 备注 |
|------|------|------|
| Windows | ✅ | 完整支持，PowerShell 5.1+ |
| macOS | ⚠️ | 需要调整路径（`$env:USERPROFILE` -> `$HOME`）|
| Linux | ⚠️ | 需要调整路径和命令 |

### Windows 说明

本工具主要为 Windows 设计：

- 使用 **PowerShell 5.1** 语法
- 无需 `jq`，使用 `ConvertFrom-Json` 解析 JSON
- 优先使用 **目录联接（junction）** 而非符号链接，无需管理员权限或开发者模式
- 跨驱动器链接也使用 junction，完全支持
- 使用 `robocopy` 进行本地镜像复制

## 依赖

- `gh` — GitHub CLI（推荐，用于从 GitHub 下载 skills）
- `git` — Git for Windows（gh 的 fallback）
- `robocopy` — Windows 内置，用于文件复制

## 与其他工具的区别

本项目的 skills 与 [xhyqaq/skill-manager](https://github.com/xhyqaq/skill-manager) 的区别：

| 项目 | 设计重点 | 环境 |
|------|----------|------|
| 本项目 | Windows 优先，详细的 SKILL.md 文档，支持复杂交互选择 | PowerShell 5.1+ |
| xhyqaq/skill-manager | 跨平台兼容，简洁实现 | Bash/macOS/Linux |

两个项目可以互补使用，根据你的主要工作环境选择。

## License

MIT
