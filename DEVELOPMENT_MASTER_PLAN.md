# rpackit Master Development Plan

> **Status:** This is the historical 0.1 target specification captured at
> project inception. It preserves the original product design and milestone
> intent; it is not the current execution checklist or API reference.
> [`ROADMAP.md`](ROADMAP.md) is the current-state source of truth. See it and
> the `rpackit` package README for implemented, copy-pasteable workflows. In
> particular,
> `build_desktop()`, `build_static()`, `build_server()`, and `init_release()`
> remain planned APIs as of rpackit 0.1.2.

**Version:** 0.1 draft  
**Date:** 2026-05-07  
**Purpose:** Preserve the original target design for architectural context.
Current planning, implementation claims, and delivery order come from
`ROADMAP.md`, the versioned contracts, and the owning implementation repository.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Product Identity](#2-product-identity)
3. [Core Scope](#3-core-scope)
4. [Build Targets](#4-build-targets)
5. [Non-Goals](#5-non-goals)
6. [Competitive and Technical Context](#6-competitive-and-technical-context)
7. [GitHub Organization Structure](#7-github-organization-structure)
8. [Repository Responsibilities](#8-repository-responsibilities)
9. [Portable R Runtime Strategy](#9-portable-r-runtime-strategy)
10. [rpackit R Package Architecture](#10-rpackit-r-package-architecture)
11. [Desktop Runtime Architecture](#11-desktop-runtime-architecture)
12. [Static Website Architecture](#12-static-website-architecture)
13. [Dynamic Server Bundle Architecture](#13-dynamic-server-bundle-architecture)
14. [Platform-Aware Development Rules](#14-platform-aware-development-rules)
15. [Codex + Superpowers Development Workflow](#15-codex--superpowers-development-workflow)
16. [Public Repository Hygiene](#16-public-repository-hygiene)
17. [Repository-Level AGENTS.md Templates](#17-repository-level-agentsmd-templates)
18. [GitHub Issues, Labels, and PR Templates](#18-github-issues-labels-and-pr-templates)
19. [Contracts and Schemas](#19-contracts-and-schemas)
20. [Milestones](#20-milestones)
21. [First 30-Day Execution Plan](#21-first-30-day-execution-plan)
22. [First Public Demo](#22-first-public-demo)
23. [Codex Prompts](#23-codex-prompts)
24. [Quality Gates](#24-quality-gates)
25. [Future Roadmap](#25-future-roadmap)
26. [Reference Links](#26-reference-links)

---

# 1. Executive Summary

`rpackit` is an open-source organization and toolchain for packaging interactive R applications into portable artifacts.

The core idea is:

```text
R app in.
Correct portable artifact out.
```

The initial focus is Shiny apps. Later, the organization can support additional R app styles or even a new R-native application framework. The first product should not try to replace Shiny. It should solve the practical shipping problem for existing R/Shiny users.

The target workflow proposed by this historical plan was:

```r
rpackit::check_app("myapp")
rpackit::build_desktop("myapp", mode = "portable")
rpackit::build_static("myapp")
rpackit::build_server("myapp")
rpackit::init_release("myapp")
```

The initial artifacts are:

```text
Portable desktop app:
  Windows .exe/.msi/portable zip
  macOS .app/.dmg
  Linux .AppImage/.deb

Static website:
  shinylive/webR export when compatible

Dynamic website:
  Docker-based Shiny server bundle
```

The organization should be designed from day one for multi-repo, multi-platform, agent-assisted development. Codex should be used heavily, but public repositories should remain professional and free from unnecessary AI-tool fingerprints.

---

# 2. Product Identity

## 2.1 Organization Name

```text
rpackit
```

## 2.2 Package Name

```text
rpackit
```

## 2.3 Taglines

Primary:

```text
Pack and ship R apps.
```

Secondary:

```text
Write R. Pack it. Ship everywhere.
```

Technical:

```text
Portable runtimes and build tools for R applications.
```

## 2.4 Public Description

```text
rpackit builds portable runtimes for R applications across desktop, static web, and server targets. It starts with Shiny apps and is designed to support broader R app frameworks in the future.
```

## 2.5 Positioning

Do not position `rpackit` as “a Shiny replacement.”

Position it as:

```text
The missing shipping layer for R apps.
```

Shiny is the first supported app type because it already has a user base.

---

# 3. Core Scope

## 3.1 Initial Supported App Type

```text
Shiny apps only
```

Supported layouts:

```text
app.R
ui.R + server.R
global.R
www/
R/
data/
renv.lock
DESCRIPTION
```

## 3.2 Initial Supported Targets

```text
1. Portable desktop app
2. Static website when compatible
3. Dynamic server bundle
```

## 3.3 Initial Supported Platforms

For development and artifact building:

```text
Windows x86_64
macOS arm64
macOS x86_64
Linux x86_64
```

For portable R runtime in the first stage:

```text
Windows x86_64 first
macOS arm64 / x86_64 second
```

## 3.4 Guiding Principle

```text
Do not invent one universal runtime.
Build a target-aware shipper.
```

---

# 4. Build Targets

## 4.1 Target A: Portable Desktop App

This is the flagship target.

Architecture:

```text
Shiny app
  ↓
rpackit build desktop
  ↓
Tauri shell + bundled portable R + bundled R packages + app files
  ↓
native desktop artifact
```

Expected outputs:

```text
Windows:
  MyApp-setup.exe
  MyApp.msi
  MyApp-portable.zip

macOS:
  MyApp.app
  MyApp.dmg

Linux:
  MyApp.AppImage
  MyApp.deb
```

Runtime behavior:

```text
User opens desktop app
  ↓
Tauri shell starts
  ↓
bundled Rscript starts launcher.R
  ↓
launcher.R starts shiny::runApp()
  ↓
Shiny listens on 127.0.0.1 random port
  ↓
Tauri WebView opens local app URL with token
  ↓
closing app kills R process
```

Key requirements:

```text
No system R required for end user
No manual package installation required for end user
No public network binding
Random localhost port
Session token
Process cleanup on exit
Clear error UI if app fails to launch
```

## 4.2 Target B: Static Website

This target uses `shinylive`/`webR`.

Architecture:

```text
Shiny app
  ↓
rpackit build static
  ↓
shinylive::export()
  ↓
static web assets
  ↓
GitHub Pages / Netlify / Cloudflare Pages / any static server
```

This target is compatibility-gated. Apps using heavy native packages, non-browser-compatible dependencies, system calls, large local files, or Bioconductor-heavy workflows should usually be blocked or warned.

Expected output:

```text
site/
  index.html
  shinylive/
  webR/
  app/
```

## 4.3 Target C: Dynamic Server Bundle

This target uses Docker.

Architecture:

```text
Shiny app
  ↓
rpackit build server
  ↓
Docker build context
  ↓
Linux server / cloud VM / ShinyProxy / container runtime
```

Expected output:

```text
server/
  Dockerfile
  docker-compose.yml
  app/
  renv.lock
  README-deploy.md
```

This target is not the main novelty; direct Docker workflows already exist. The value is that server output is one target in a unified build matrix.

---

# 5. Non-Goals

Do not include these in the initial implementation:

```text
Plumber support
custom Shiny replacement
Kubernetes deployment
AWS/GCP/Azure deployment
automatic macOS notarization in v0.1
single-file self-extracting EXE
full Bioconductor demonstration app
Seurat/large single-cell demo app
Linux portable R runtime
R package submission to CRAN before architecture stabilizes
```

Future support is possible, but early versions must remain focused.

---

# 6. Competitive and Technical Context

## 6.1 Shiny

Shiny is the starting app framework. `rpackit` does not replace it initially; it packages it.

## 6.2 shinylive/webR

`shinylive` enables static web hosting by running R/Shiny in the browser via webR. This is ideal for compatible lightweight Shiny apps but not for all R packages or large bioinformatics workflows.

## 6.3 RDesk

RDesk is an important Windows-only desktop R app framework. It validates the market but does not eliminate the need for `rpackit`, because `rpackit` focuses on:

```text
cross-platform builds
Shiny compatibility
desktop + static web + server matrix
portable runtime infrastructure
CI-based release pipelines
```

## 6.4 Tauri

Tauri is the desktop shell layer for `rpackit`. Users should not write Rust. `rpackit` should generate Tauri projects and use mostly static Rust boilerplate.

## 6.5 Docker/Shiny Server/ShinyProxy

Docker is the dynamic server portability layer. `rpackit` should generate good Docker build contexts, but it should not pretend to replace Docker expertise in v0.x.

## 6.6 Python Packaging Inspiration

Python has strong packaging analogues:

```text
PyInstaller:
  package Python app + interpreter + dependencies into user-runnable artifact

BeeWare Briefcase:
  one codebase → multiple platform artifacts

Pyodide/PyScript:
  browser-side Python runtime analogous to webR/shinylive

Docker:
  dynamic server deployment
```

`rpackit` should be the R ecosystem equivalent:

```text
R app source
  ↓
rpackit check
  ↓
desktop / static web / server artifact
```

---

# 7. GitHub Organization Structure

Create the GitHub organization:

```text
rpackit
```

## 7.1 Initial Repositories

```text
.github
roadmap
rpackit
portable-r
portable-r-windows
rpackit-examples
```

## 7.2 Later Repositories

```text
portable-r-macos
rpackit-tauri
rpackit-actions
rpackit-docs
rpackit-framework
```

## 7.3 Recommended Repository Creation Order

```text
1. .github
2. roadmap
3. rpackit
4. portable-r
5. portable-r-windows
6. rpackit-examples
7. portable-r-macos
8. rpackit-tauri
9. rpackit-actions
10. rpackit-docs
11. rpackit-framework
```

---

# 8. Repository Responsibilities

## 8.1 `.github`

Organization profile and shared community defaults.

```text
.github/
  profile/
    README.md
  ISSUE_TEMPLATE/
  PULL_REQUEST_TEMPLATE.md
  CODE_OF_CONDUCT.md
  SECURITY.md
  FUNDING.yml
```

Organization profile README draft:

```markdown
# rpackit

**Write R. Pack it. Ship everywhere.**

rpackit builds portable runtimes for R apps across desktop, static web, and server targets.

## What we build

- Portable desktop apps: Tauri + bundled R + bundled packages
- Static web apps: shinylive/webR when compatible
- Dynamic server apps: Dockerized Shiny apps for Linux servers
- Portable R runtimes: maintained Windows and macOS R distributions

## Core repositories

- `rpackit`: R package and CLI
- `portable-r`: portable R runtime index and metadata
- `portable-r-windows`: Windows portable R builds
- `portable-r-macos`: macOS portable R builds
- `rpackit-examples`: example apps
```

## 8.2 `roadmap`

Central truth repo.

```text
roadmap/
  README.md
  DEVELOPMENT_MASTER_PLAN.md
  ARCHITECTURE.md
  ROADMAP.md
  CONTRACTS.md
  PLATFORMS.md
  AGENT_POLICY.md
  PUBLIC_REPO_HYGIENE.md
  CODE_REVIEW.md
  DECISIONS.md
  REPO_MAP.md
  contracts/
  prompts/
```

## 8.3 `rpackit`

Main R package and CLI.

```text
rpackit/
  DESCRIPTION
  NAMESPACE
  R/
  tests/
  inst/templates/
  scripts/
  AGENTS.md
```

## 8.4 `portable-r`

Runtime metadata index.

```text
portable-r/
  versions.json
  metadata/
  schemas/
  scripts/
```

Large runtime archives must be released through GitHub Releases, not committed to git.

## 8.5 `portable-r-windows`

Windows portable R builder.

```text
portable-r-windows/
  scripts/
    build.ps1
    patch-rprofile.ps1
    verify.ps1
    package.ps1
  tests/
  .github/workflows/release.yml
  AGENTS.md
```

## 8.6 `portable-r-macos`

macOS portable R builder.

```text
portable-r-macos/
  scripts/
    build.sh
    patch-framework.sh
    verify.sh
    codesign.sh
    notarize.sh
    package.sh
  tests/
  .github/workflows/release.yml
  AGENTS.md
```

## 8.7 `rpackit-tauri`

Tauri templates.

```text
rpackit-tauri/
  templates/
    shiny-portable/
    shiny-static/
  frontend/
    launcher.js
    loading.html
    error.html
  src-tauri/
    minimal-main.rs
  AGENTS.md
```

## 8.8 `rpackit-actions`

Reusable workflows.

```text
rpackit-actions/
  .github/workflows/
    r-package-check.yml
    portable-r-verify.yml
    tauri-build.yml
    docker-build.yml
    docs-build.yml
```

## 8.9 `rpackit-examples`

Example apps.

```text
rpackit-examples/
  hello-shiny/
  csv-viewer/
  ggplot-dashboard/
  shinylive-compatible/
  bioc-result-viewer/
  not-shinylive-compatible/
```

## 8.10 `rpackit-docs`

Documentation website.

```text
rpackit-docs/
  index.qmd
  getting-started.qmd
  portable-desktop.qmd
  static-web.qmd
  server-bundle.qmd
  platform-guide.qmd
  codex-development.qmd
```

---

# 9. Portable R Runtime Strategy

## 9.1 Purpose

Portable R is the foundation of reliable desktop packaging.

It provides:

```text
known R versions
known folder layout
known package library behavior
known executable paths
known relocation behavior
known checksums
```

## 9.2 Supported Versions

Initial:

```text
R 4.5.x
R 4.4.x
```

Potential later:

```text
R 4.3.x
```

## 9.3 Supported Platforms

Initial:

```text
Windows x86_64
macOS arm64
macOS x86_64
```

Future:

```text
Linux x86_64
Linux arm64
```

For Linux dynamic server target, Docker is preferred over portable Linux R.

## 9.4 Windows Layout

```text
portable-r/
  R/
    bin/
      R.exe
      Rscript.exe
    etc/
      Rprofile.site
    library/
    modules/
    include/
  metadata.json
  LICENSES/
```

Suggested `Rprofile.site`:

```r
.local_lib <- file.path(R.home(), "library")
.libPaths(.local_lib)
```

## 9.5 macOS Layout

```text
portable-r/
  R.framework/
    Resources/
      bin/
        R
        Rscript
      library/
      etc/
        Rprofile.site
  metadata.json
  LICENSES/
```

## 9.6 Verification Requirements

Every runtime artifact must pass:

```text
Rscript runs
R.home() points inside runtime
.libPaths() points inside runtime
package installation writes inside runtime
runtime can be moved to another path
path with spaces works
metadata validates against schema
sha256 matches release artifact
```

Windows-specific:

```text
Rscript.exe works from PowerShell
no admin privileges required
no registry dependency assumed
```

macOS-specific:

```text
Rscript works after relocation
R.framework load paths are relocatable
signing/notarization status documented
```

---

# 10. rpackit R Package Architecture

## 10.1 Target Public API

```r
rpackit::doctor()
rpackit::check_app(app_dir)
rpackit::build_desktop(app_dir, mode = "portable", r_version = NULL)
rpackit::build_static(app_dir)
rpackit::build_server(app_dir)
rpackit::init_release(app_dir)
```

## 10.2 `doctor()`

Purpose:

```text
Inspect local machine and report supported development/build tasks.
```

Expected output:

```text
Platform: macOS arm64
R: 4.6.1
Node: found
Rust/Cargo: found
Tauri CLI: found
Docker: found
GitHub CLI: authenticated
Codex: found

Supported tasks:
- R package development
- macOS portable R verification
- macOS desktop artifact testing

Unsupported tasks:
- Windows portable R artifact verification
- Windows installer verification
```

## 10.3 `check_app()`

Purpose:

```text
Inspect a Shiny app and recommend supported build targets.
```

Detect:

```text
app.R or ui.R/server.R
global.R
www/
renv.lock
DESCRIPTION
library()/require() calls
Bioconductor packages
system()/system2()
reticulate usage
large data extensions
static web risk
desktop bundle risk
server suitability
```

Output target matrix:

```text
portable desktop: yes/no/maybe
static web: yes/no/maybe
dynamic server: yes/no/maybe
```

Example:

```text
Detected app type: Shiny

Packages:
  shiny
  ggplot2
  DESeq2
  Rsamtools

Target matrix:
  portable desktop: yes
  static web: no
  dynamic server: yes

Recommendation:
  Use build_desktop() for offline users.
  Use build_server() for shared Linux deployment.
  Do not use build_static() because Rsamtools requires native runtime support.
```

## 10.4 `build_desktop()`

Steps:

```text
1. Run check_app().
2. Resolve platform and architecture.
3. Resolve portable R version.
4. Download portable R artifact.
5. Verify SHA256.
6. Unpack into build/resources/R.
7. Copy app into build/resources/app.
8. Restore/install packages into bundled library.
9. Generate launcher.R.
10. Generate Tauri project.
11. Build platform-specific artifact.
12. Write build manifest.
```

## 10.5 `build_static()`

Steps:

```text
1. Run check_app().
2. Fail or warn if app has static-web blockers.
3. Call shinylive::export().
4. Write site/ output.
5. Optionally generate GitHub Pages workflow.
```

## 10.6 `build_server()`

Steps:

```text
1. Run check_app().
2. Copy app into server/app.
3. Copy renv.lock if available.
4. Generate Dockerfile.
5. Generate docker-compose.yml.
6. Generate README-deploy.md.
7. Optionally run docker build.
```

## 10.7 `init_release()`

Generates:

```text
.github/workflows/rpackit-release.yml
```

Workflow should build:

```text
Windows desktop artifact
macOS desktop artifact
Linux desktop artifact
static web artifact
server Docker build context/image
```

---

# 11. Desktop Runtime Architecture

## 11.1 Generated Build Layout

```text
build/
  src-tauri/
    src/main.rs
    tauri.conf.json
    capabilities/
  frontend/
    index.html
    launcher.js
    loading.html
    error.html
  resources/
    R/
    app/
      app.R
      www/
      renv.lock
    launcher.R
    rpackit.json
```

## 11.2 Runtime Sequence

```text
User opens desktop app
  ↓
Tauri shell starts
  ↓
launcher.js starts bundled Rscript
  ↓
launcher.R starts shiny::runApp()
  ↓
Shiny listens on 127.0.0.1 random port
  ↓
Tauri WebView opens local URL with token
  ↓
User interacts with app
  ↓
Closing window kills R process
```

## 11.3 Security Requirements

```text
Bind only to 127.0.0.1
Use random port
Use random token
Never bind 0.0.0.0
Kill R process on exit
Show readable errors if R fails
Do not expose arbitrary local file paths to frontend
```

## 11.4 Launcher Command

```text
Rscript resources/launcher.R \
  --app resources/app \
  --port <random_port> \
  --token <random_token>
```

## 11.5 `launcher.R` Responsibilities

```text
parse args
set .libPaths() to bundled runtime library
validate app directory
start shiny::runApp()
host=127.0.0.1
write readiness signal
write logs
exit cleanly
```

---

# 12. Static Website Architecture

`rpackit::build_static()` uses `shinylive`.

```text
Shiny app
  ↓
shinylive::export()
  ↓
site/
  index.html
  shinylive assets
  webR runtime
  app files
```

Static compatibility checks should block or warn for:

```text
packages unavailable in webR
system calls
local binaries
large native dependencies
large data files
Bioconductor-heavy workflows
```

---

# 13. Dynamic Server Bundle Architecture

`rpackit::build_server()` creates Docker infrastructure.

```text
server/
  Dockerfile
  docker-compose.yml
  app/
  renv.lock
  README-deploy.md
```

Docker image includes:

```text
Linux base image
R version
system dependencies
R packages
Shiny app
startup command
```

Initial server output should be a Docker build context, not a full cloud deployment system.

---

# 14. Platform-Aware Development Rules

## 14.1 Platform Roles

```text
macOS:
  portable-r-macos
  .app/.dmg
  macOS Tauri builds
  codesign/notarization later

Windows:
  portable-r-windows
  .exe/.msi
  Rtools
  PowerShell scripts

Linux:
  Docker/server target
  AppImage/deb
  CI simulation
  container builds

Any:
  R package logic
  docs
  metadata schemas
  examples
```

## 14.2 Required Platform Behavior

Agents must:

```text
detect current platform before platform-specific work
stop if task requires another platform
never claim native verification unless tests ran on required OS
write platform evidence in PRs
```

## 14.3 `scripts/detect-platform.R`

```r
platform <- tolower(Sys.info()[["sysname"]])
machine <- Sys.info()[["machine"]]

normalized <- switch(
  platform,
  "darwin" = "macos",
  "windows" = "windows",
  "linux" = "linux",
  platform
)

cat(sprintf("platform=%s\narch=%s\n", normalized, machine))
```

## 14.4 `scripts/require-platform.R`

```r
args <- commandArgs(trailingOnly = TRUE)
required <- args[[1]]

platform <- tolower(Sys.info()[["sysname"]])
normalized <- switch(
  platform,
  "darwin" = "macos",
  "windows" = "windows",
  "linux" = "linux",
  platform
)

if (!identical(normalized, required)) {
  stop(sprintf(
    "This task requires platform '%s', but current platform is '%s'. Rerun this task on %s.",
    required, normalized, required
  ), call. = FALSE)
}

cat(sprintf("OK: running on required platform '%s'\n", required))
```

---

# 15. Codex + Superpowers Development Workflow

## 15.1 Operating Model

```text
Main Codex session:
  architect/planner/coordinator

Repo Codex sessions:
  implementers/reviewers

GitHub Issues:
  task packets

GitHub PRs:
  implementation packets

CONTRACTS.md:
  source of truth for cross-repo interfaces
```

## 15.2 Superpowers Role

Use Superpowers to enforce:

```text
brainstorming
writing plans
using git worktrees
test-driven development
requesting code review
finishing development branch
```

Superpowers should be installed in Codex on macOS, Windows, and Linux.

## 15.3 Main Planning Prompt

```text
Use Superpowers.

You are the rpackit planning architect.

Read:
- roadmap/DEVELOPMENT_MASTER_PLAN.md
- roadmap/ARCHITECTURE.md
- roadmap/CONTRACTS.md
- roadmap/PLATFORMS.md
- roadmap/AGENT_POLICY.md
- roadmap/PUBLIC_REPO_HYGIENE.md

Do not edit implementation repositories directly.

Your job:
1. Maintain architecture.
2. Maintain contracts.
3. Break milestones into repo-specific issues.
4. Ensure each issue has:
   - target repo
   - required platform
   - files likely touched
   - acceptance criteria
   - required tests
   - cross-repo dependencies
5. Keep public repository hygiene.
```

## 15.4 Repo Coding Prompt

```text
Use Superpowers.

You are working in the <repo-name> repository.

Before coding:
1. Read AGENTS.md.
2. Read the assigned GitHub issue.
3. Run scripts/detect-platform.R or the repo's doctor script.
4. Compare current platform with the issue label.

If the platform is wrong:
- stop before coding
- report current platform
- report required platform
- tell me which machine/session to use

If the platform is correct:
- create a git worktree
- follow TDD where applicable
- implement only the assigned issue
- run required tests
- open a PR with an Agent Handoff section

Do not commit agent transcripts, prompt logs, scratch files, or AI-tool attribution.
```

---

# 16. Public Repository Hygiene

## 16.1 Policy

```markdown
# Public Repository Hygiene

We use automation internally, but public repositories should read as professional, reviewed software projects.

## Rules

- Do not commit agent transcripts, prompt logs, or scratch files.
- Do not add "Generated by Codex/Claude/Copilot" comments.
- Do not add AI/tool co-author trailers unless explicitly required.
- Commit messages should describe the technical change, not the tool.
- PRs should summarize problem, approach, tests, and risk.
- All merged code is reviewed and owned by the maintainer.
```

## 16.2 `.gitignore`

```gitignore
# Agent scratch/session files
.codex/
.claude/
.agent/
.agent-scratch/
.session/
SESSION_SUMMARY.md
SESSION_TRANSCRIPT.md
*.transcript.md
*.prompt.md
*.agent.md
.agent-notes.md

# Local planning
.local/
.notes/
scratch/
tmp/
```

## 16.3 Commit Message Policy

Good:

```text
Add portable R metadata validator
Fix Windows runtime path normalization
Add tests for path-with-spaces extraction
```

Avoid:

```text
Codex generated metadata validator
AI implementation of Windows builder
Claude fixes tests
```

## 16.4 Public Hygiene Check Script

```bash
#!/usr/bin/env bash
set -euo pipefail

bad_patterns=(
  "Generated by Codex"
  "Generated by Claude"
  "Generated by Copilot"
  "Co-authored-by:.*Codex"
  "Co-authored-by:.*Copilot"
  "SESSION_TRANSCRIPT"
  "prompt log"
)

for pattern in "${bad_patterns[@]}"; do
  if git diff --cached --name-only -z | xargs -0 grep -nEi "$pattern" 2>/dev/null; then
    echo "Blocked: staged files contain agent/tooling fingerprint: $pattern"
    exit 1
  fi
done
```

---

# 17. Repository-Level AGENTS.md Templates

## 17.1 `rpackit/AGENTS.md`

```markdown
# rpackit agent instructions

This repository contains the main R package and CLI.

## Required reading

Before coding, read:

- ../roadmap/DEVELOPMENT_MASTER_PLAN.md
- ../roadmap/CONTRACTS.md
- ../roadmap/PLATFORMS.md
- ../roadmap/PUBLIC_REPO_HYGIENE.md

## Startup checklist

Run:

```bash
Rscript scripts/detect-platform.R
Rscript -e "sessionInfo()"
```

If the issue has a platform-specific label, verify that this machine matches the required platform before coding.

## Style

- Use explicit namespacing.
- Use cli::cli_abort(), cli::cli_warn(), cli::cli_inform().
- Use testthat 3e.
- Do not use network access in tests.
- Mock external tools in tests.
- Prefer file.path() or fs::path().
- Do not hardcode path separators.

## Required checks

Run:

```bash
Rscript -e "devtools::test()"
Rscript -e "devtools::check()"
```

If these cannot run, explain exactly why in the PR handoff.
```

## 17.2 `portable-r-windows/AGENTS.md`

```markdown
# portable-r-windows agent instructions

This repository builds portable R runtimes for Windows.

## Platform policy

Build and verification tasks require Windows native PowerShell.

Docs and metadata edits may be done on any platform, but final verification must run on Windows.

## Startup checklist

Run:

```powershell
Rscript scripts/require-platform.R windows
$PSVersionTable.PSVersion
[System.Runtime.InteropServices.RuntimeInformation]::OSDescription
```

If not on Windows, stop and ask the user to rerun the task on Windows.

## Rules

- Do not commit built R runtimes.
- Publish runtime archives through GitHub Releases.
- Every artifact must include metadata.json.
- Every artifact must have a sha256.
- Test path-with-spaces behavior.
- Test moved-directory behavior.
```

## 17.3 `portable-r-macos/AGENTS.md`

```markdown
# portable-r-macos agent instructions

This repository builds portable R runtimes for macOS.

## Platform policy

Build and verification tasks require macOS.

## Startup checklist

Run:

```bash
Rscript scripts/require-platform.R macos
uname -s
uname -m
sw_vers
```

If not on macOS, stop and ask the user to rerun the task on macOS.

## Rules

- Do not commit built R runtimes.
- Publish runtime archives through GitHub Releases.
- Verify relocation after moving the runtime directory.
- Document signing/notarization status.
```

---

# 18. GitHub Issues, Labels, and PR Templates

## 18.1 Labels

Platform labels:

```text
platform:any
platform:windows
platform:macos
platform:linux
platform:ci
platform:cross-platform
```

Component labels:

```text
component:portable-r
component:desktop
component:static-web
component:server
component:tauri
component:r-package
component:docs
```

Type labels:

```text
type:feature
type:bug
type:contract
type:docs
type:ci
type:test
type:refactor
```

Priority labels:

```text
priority:p0
priority:p1
priority:p2
priority:p3
```

## 18.2 Issue Template

```markdown
## Summary

Describe the task.

## Target repository

- [ ] .github
- [ ] roadmap
- [ ] rpackit
- [ ] portable-r
- [ ] portable-r-windows
- [ ] portable-r-macos
- [ ] rpackit-tauri
- [ ] rpackit-examples
- [ ] rpackit-actions
- [ ] rpackit-docs

## Required platform

- [ ] any
- [ ] windows
- [ ] macos
- [ ] linux
- [ ] ci-matrix

## Files likely touched

- 

## Acceptance criteria

- 

## Tests required

- 

## Contract impact

- [ ] no contract change
- [ ] updates CONTRACTS.md
- [ ] updates JSON schema

## Dependencies

- 
```

## 18.3 PR Template

```markdown
## Summary

What changed and why?

## Changes

- 
- 

## Tests

Commands run:

```text
```

## Platform verification

Current platform:
- [ ] macOS
- [ ] Windows
- [ ] Linux
- [ ] GitHub Actions only
- [ ] Not platform-specific

Required platform:
- [ ] any
- [ ] macOS
- [ ] Windows
- [ ] Linux
- [ ] CI matrix

## Contract impact

- [ ] No contract impact
- [ ] CONTRACTS.md updated
- [ ] JSON schema updated

## Agent handoff

### Issue
rpackit/<repo>#<number>

### Contract impact
Contract changed: yes/no

### Follow-up
List any follow-up tasks.
```

---

# 19. Contracts and Schemas

## 19.1 Portable R Metadata Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Portable R Metadata",
  "type": "object",
  "required": [
    "schema_version",
    "r_version",
    "platform",
    "arch",
    "artifact_url",
    "sha256",
    "archive_format",
    "r_home",
    "rscript",
    "library"
  ],
  "properties": {
    "schema_version": { "const": "1" },
    "r_version": { "type": "string" },
    "platform": { "enum": ["windows", "macos", "linux"] },
    "arch": { "enum": ["x86_64", "arm64"] },
    "artifact_url": { "type": "string" },
    "sha256": { "type": "string" },
    "archive_format": { "enum": ["zip", "tar.zst", "tar.gz"] },
    "r_home": { "type": "string" },
    "rscript": { "type": "string" },
    "library": { "type": "string" }
  }
}
```

## 19.2 Desktop Resource Layout Contract

```text
resources/
  R/
  app/
  launcher.R
  rpackit.json
```

## 19.3 Desktop Launcher Contract

```text
launcher.R --app <path> --port <port> --token <token>
```

## 19.4 Artifact Naming Contract

```text
portable-r-{platform}-{arch}-{r_version}.{zip|tar.zst}
```

Examples:

```text
portable-r-windows-x86_64-4.6.1.zip
portable-r-macos-arm64-4.6.1.tar.zst
portable-r-macos-x86_64-4.6.1.tar.zst
```

---

# 20. Milestones

These are the original target milestones, not live completion status. See
`ROADMAP.md` for implemented work and the current delivery order.

## Milestone 0: Organization Foundation

Repos:

```text
.github
roadmap
rpackit
portable-r
portable-r-windows
rpackit-examples
```

Deliverables:

```text
organization profile README
DEVELOPMENT_MASTER_PLAN.md
ARCHITECTURE.md
CONTRACTS.md
PLATFORMS.md
AGENT_POLICY.md
PUBLIC_REPO_HYGIENE.md
CODE_REVIEW.md
AGENTS.md in each repo
issue templates
PR templates
```

Done when:

```text
Every initial repo has README.md, LICENSE, AGENTS.md, .gitignore, and basic issue/PR templates.
```

## Milestone 1: Portable R Windows

Deliverables:

```text
portable-r metadata schema v1
portable-r-windows build script
portable-r-windows verify script
R 4.6.1 Windows portable artifact
GitHub Release artifact
portable-r index entry
```

Acceptance criteria:

```text
Rscript.exe runs
R.home() points inside runtime
.libPaths() points inside runtime
artifact can be moved and still run
path with spaces works
metadata validates against schema
sha256 matches artifact
```

## Milestone 2: rpackit Checker

Deliverables:

```text
rpackit::doctor()
rpackit::check_app()
Shiny app detection
renv.lock detection
library()/require() detection
basic target matrix
```

Acceptance criteria:

```text
check_app() reports:
- portable desktop suitability
- static web suitability
- server suitability
```

## Milestone 3: Portable Desktop Prototype, Windows

Deliverables:

```text
rpackit::build_desktop(..., mode = "portable")
portable-r-windows integration
Tauri project generation
hello-shiny packaged app
```

Acceptance criteria:

```text
Built app runs on Windows without system R installed.
```

## Milestone 4: Release Workflow

Deliverables:

```text
rpackit::init_release()
GitHub Actions workflow
Windows artifact build
artifact upload
placeholder macOS/Linux jobs
```

Acceptance criteria:

```text
One GitHub Actions workflow can build and upload Windows artifact.
```

## Milestone 5: Static Website Target

Deliverables:

```text
rpackit::build_static()
shinylive integration
static web compatibility warnings
example static site
```

Acceptance criteria:

```text
hello-shiny builds to static site.
incompatible app gives clear diagnostic.
```

## Milestone 6: macOS Portable R

Deliverables:

```text
portable-r-macos arm64 artifact
portable-r-macos x86_64 artifact
rpackit macOS integration
basic macOS desktop app build
```

Acceptance criteria:

```text
Built macOS app runs using bundled R.
```

## Milestone 7: Server Bundle

Deliverables:

```text
rpackit::build_server()
Dockerfile generation
docker-compose.yml
README-deploy.md
server example
```

Acceptance criteria:

```text
docker build works.
docker run exposes Shiny app.
```

## Milestone 8: Linux Desktop Artifact

Deliverables:

```text
Tauri AppImage build
Linux desktop integration
CI job
```

Acceptance criteria:

```text
AppImage runs on target Linux environment.
```

---

# 21. First 30-Day Execution Plan

This section records the initial schedule proposed in May 2026. It is retained
for historical context and must not be used as the current work queue.

## Week 1: Organization and Standards

Create:

```text
rpackit organization
.github repo
roadmap repo
rpackit repo
portable-r repo
portable-r-windows repo
rpackit-examples repo
```

Write:

```text
DEVELOPMENT_MASTER_PLAN.md
ARCHITECTURE.md
CONTRACTS.md
PLATFORMS.md
AGENT_POLICY.md
PUBLIC_REPO_HYGIENE.md
```

Add:

```text
AGENTS.md
README.md
LICENSE
.gitignore
PR template
issue template
```

## Week 2: Windows Portable R

Implement in `portable-r-windows`:

```text
scripts/build.ps1
scripts/patch-rprofile.ps1
scripts/verify.ps1
metadata generation
release workflow draft
```

Goal:

```text
Create first portable-r-windows-x86_64-4.6.1.zip artifact.
```

## Week 3: rpackit Checker

Implement in `rpackit`:

```text
doctor()
check_app()
detect_shiny_app()
detect_packages()
detect_renv()
recommend_targets()
```

Add tests.

## Week 4: Desktop Prototype

Implement in `rpackit`:

```text
resolve_portable_r()
download_portable_r()
verify_portable_r()
render_tauri_template()
build_desktop()
```

Goal:

```text
hello-shiny → portable Windows desktop app
```

---

# 22. First Public Demo

Commands:

```bash
git clone https://github.com/rpackit/rpackit-examples
cd rpackit-examples/hello-shiny

Rscript -e 'rpackit::check_app(".")'
Rscript -e 'rpackit::plan_dependencies(".")'
Rscript -e 'rpackit::prepare_desktop(".", runtime_dir = "/path/to/portable-r")'
```

Current implemented result:

```text
dist/desktop-resources/resources/
  R/
  app/
  launcher.R
  rpackit.json
```

Current verified claim:

```text
The hello-shiny resources run with the bundled Windows R and bundled package
library, without consulting a system R installation. The launcher serves the
app on 127.0.0.1 and passes an HTTP smoke test.
```

The future native-executable acceptance target remains
`dist/HelloShiny-setup.exe`; it is not yet an implemented output.

---

# 23. Codex Prompts

## 23.1 Main Planning Prompt

```text
Use Superpowers.

We are creating the rpackit GitHub organization.

Mission:
rpackit builds portable runtimes for R apps across desktop, static web, and server targets.

Initial scope:
- Shiny apps only
- portable desktop app
- static website when compatible
- dynamic server bundle
- no Plumber support for now
- no custom Shiny replacement yet

Read and maintain:
- DEVELOPMENT_MASTER_PLAN.md
- ARCHITECTURE.md
- CONTRACTS.md
- PLATFORMS.md
- AGENT_POLICY.md
- PUBLIC_REPO_HYGIENE.md

Your role:
You are the planning architect.
Do not implement code directly unless asked.
Create repo-specific GitHub issues with:
- target repository
- required platform
- files likely touched
- acceptance criteria
- tests required
- cross-repo dependencies

First task:
Generate the initial repository setup issues for:
1. .github
2. roadmap
3. rpackit
4. portable-r
5. portable-r-windows
6. rpackit-examples

Follow the public repository hygiene policy.
Do not include agent transcripts or AI-tool references in public-facing docs.
```

## 23.2 Repo Coding Prompt

```text
Use Superpowers.

You are working in the <repo-name> repository.

Before coding:
1. Read AGENTS.md.
2. Read the assigned GitHub issue.
3. Run scripts/detect-platform.R or the repo's doctor script.
4. Compare current platform with the issue label.

If the platform is wrong:
- stop before coding
- report current platform
- report required platform
- tell me which machine/session to use

If the platform is correct:
- create a git worktree
- follow TDD where applicable
- implement only the assigned issue
- run required tests
- open a PR with an Agent Handoff section

Do not commit agent transcripts, prompt logs, scratch files, or AI-tool attribution.
```

## 23.3 Review Prompt

```text
Use Superpowers.

You are reviewing a PR in the rpackit organization.

Read:
- the assigned issue
- AGENTS.md
- roadmap/CONTRACTS.md
- roadmap/CODE_REVIEW.md
- roadmap/PUBLIC_REPO_HYGIENE.md

Review for:
1. Contract compliance
2. Platform correctness
3. Tests
4. Public repo hygiene
5. Error handling
6. Maintainability

Classify findings as:
- Critical
- Major
- Minor

Critical issues block merge.
```

---

# 24. Quality Gates

## 24.1 Required for All PRs

```text
Tests added or updated
Platform verification included
No agent artifacts committed
No AI/tool attribution accidentally included
No generated binary artifacts committed
No hardcoded absolute paths
No hardcoded path separators
```

## 24.2 R Package PRs

```text
devtools::test()
devtools::check()
no network access in tests
mock external tools
explicit namespacing
cli error style
```

## 24.3 Portable Runtime PRs

```text
metadata validates
sha256 generated
path-with-spaces test
moved-directory test
no binary committed to git
release workflow uses artifact upload
```

## 24.4 Tauri PRs

```text
minimal Rust
template parameterized
no hardcoded app name
no hardcoded port
random token support
process cleanup
clear error screen
```

---

# 25. Future Roadmap

## 25.1 After v1

Potential future additions:

```text
Plumber/API app support
Quarto/report app packaging
rpackit-native app framework
macOS signing/notarization automation
Windows code signing
Linux portable R
cloud deployment presets
ShinyProxy config generation
Posit Connect integration
Bioconductor-aware dependency planner
RStudio addin
VS Code extension
```

## 25.2 Future Framework Direction

Do not start here. Later, `rpackit-framework` could provide:

```text
transport-independent R app runtime
desktop-aware architecture
job/progress system
file-oriented UI primitives
httpuv transport
Tauri IPC transport
webR transport
server transport
```

This is future work after `rpackit` has adoption as a packaging tool.

---

# 26. Reference Links

These sources informed the plan and should be reviewed when implementing:

- Codex `AGENTS.md` project instructions: https://developers.openai.com/codex/guides/agents-md
- AGENTS.md convention: https://agents.md/
- Tauri GitHub Action: https://github.com/tauri-apps/tauri-action
- Tauri cross-platform build guide: https://tauri.app/v1/guides/building/cross-platform
- Tauri GitHub pipeline docs: https://v2.tauri.app/distribute/pipelines/github/
- shinylive documentation: https://posit-dev.github.io/r-shinylive/
- shinylive CRAN page: https://cran.r-project.org/package=shinylive
- GitHub organization profile README docs: https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/customizing-your-organizations-profile
- GitHub organization README changelog: https://github.blog/changelog/2021-09-14-readmes-for-organization-profiles/
- shiny2docker vignette: https://cran.r-project.org/web/packages/shiny2docker/vignettes/introduction.html
- Portable R Windows prior art: https://github.com/portable-r/portable-r-windows
- Posit/RStudio portable R builds prior art: https://github.com/rstudio/r-builds
- Superpowers methodology: https://github.com/obra/superpowers

---

# Final Principle

```text
Do not build a universal runtime.
Build the R app shipper that chooses and packages the right runtime.
```

```text
R app in.
Correct portable artifact out.
```

```text
Contracts, issues, PRs, and tests are the communication layer.
Codex is the worker, not the source of truth.
```
