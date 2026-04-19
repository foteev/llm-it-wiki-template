# IT Wiki System — Full Implementation Guide

> A dual-vault, git-backed, Obsidian-powered knowledge base for IT teams.
> Designed to work with human contributors and AI assistants in strict collaboration.
> Fork this, adapt names to your company, and run it.

---

## Table of Contents

1. [Philosophy](#1-philosophy)
2. [Architecture Overview](#2-architecture-overview)
3. [Repository Layout](#3-repository-layout)
4. [Bootstrapping from Scratch](#4-bootstrapping-from-scratch)
5. [Conventions](#5-conventions)
6. [Page Templates](#6-page-templates)
7. [AGENTS.md — IT Vault](#7-agentsmd--it-vault)
8. [AGENTS.md — Users Vault](#8-agentsmd--users-vault)
9. [sync-wiki.sh](#9-sync-wikish)
10. [Human Workflow](#10-human-workflow)
11. [AI Assistant Rules Summary](#11-ai-assistant-rules-summary)
12. [Lint & Quality Checklist](#12-lint--quality-checklist)
13. [Design Decisions & Rationale](#13-design-decisions--rationale)

---

## 1. Philosophy

Most IT knowledge bases fail in one of two ways:

**Failure Mode A — Over-formalized:** A SharePoint graveyard. Nobody can find anything. Nobody updates anything because updating is slow and discouraged. Knowledge rots silently.

**Failure Mode B — Over-casual:** A Slack channel. Everything is fast and searchable for 30 days, then gone forever. Institutional knowledge evaporates when people leave.

This system is designed to avoid both. The core ideas are:

**1. Separation of Concern — two vaults, two audiences.**
Technical documentation (runbooks, infra, configs) is kept in `it-vault`, maintained by IT staff. Employee-facing guides (how-to, known issues) are kept in `users-vault`, readable by everyone. They reference each other but do not copy each other.

**2. Raw sources are sacred.**
Every configuration file, export, or script that informs a wiki page is stored in `raw/` and never modified. The wiki is derived from raw evidence. This prevents documentation from drifting away from reality.

**3. AI drafts. Humans approve.**
An AI assistant can draft runbooks at high speed from chat discussions or source files. But the AI never commits, never sets a runbook to `status: active`, and explicitly flags uncertainty. The human is the quality gate — not a bottleneck on content creation, but the final authority on correctness.

**4. Provenance over prose.**
Every runbook must link to what it was derived from via `sources:` in YAML frontmatter. Every operation must be logged. The index must be updated. These rules are enforced by the AI and checked by humans.

---

## 2. Architecture Overview

```
Obsidian/                        ← Workspace root (not a git repo itself)
├── it-vault/                    ← Git repo #1 (IT team only)
│   ├── AGENTS.md                ← AI operating rules for this vault
│   ├── README.md                ← Human onboarding for IT staff
│   ├── sync-wiki.sh             ← Sync script for both vaults
│   ├── index.md                 ← Master index of all wiki pages
│   ├── log.md                   ← Operational change log
│   ├── raw/                     ← Immutable source files (never edit)
│   ├── wiki/                    ← All authored documentation
│   └── templates/               ← Frontmatter templates
│
└── users-vault/                 ← Git repo #2 (all employees, read-only)
    ├── AGENTS.md                ← AI operating rules for this vault
    ├── README.md                ← Human guide for employees
    ├── index.md                 ← Index of all guides
    ├── log.md                   ← Change log
    └── wiki/                    ← Guides, known issues, glossary
```

Both vaults are **independent git repositories** hosted in Azure DevOps (or GitHub/GitLab). The sync script handles committing and pushing both in a single step.

**Cross-vault references** use pseudolinks (not filesystem paths) to avoid broken links:
- From `users-vault` to `it-vault`: `@IT:infrastructure/runbooks/reset-mfa`
- From `it-vault` to `users-vault`: `@USERS:how-to/sharepoint/finding-archived-files`

---

## 3. Repository Layout

### it-vault (full)

```
it-vault/
├── AGENTS.md
├── README.md
├── sync-wiki.sh
├── index.md
├── log.md
│
├── raw/
│   ├── configs/
│   │   ├── firewalls/           ← NGFW config exports (XML/JSON)
│   │   ├── NAS/                 ← NAS appliance config backups
│   │   └── servers/             ← Server config snapshots
│   └── scripts/
│       ├── ad/                  ← Directory Services automation scripts
│       ├── cloud/               ← Cloud platform automation scripts
│       └── document-platform/   ← Document management platform scripts
│
├── wiki/
│   ├── infrastructure/
│   │   ├── servers/
│   │   │   └── virtual-machines/  ← One .md per VM / server
│   │   ├── storage/               ← NAS / cloud storage entities
│   │   ├── network/
│   │   │   └── firewalls/         ← One .md per firewall appliance
│   │   ├── monitoring/            ← SIEM, log management, alerting
│   │   ├── virtualization/
│   │   └── runbooks/
│   │       ├── network/           ← Network provisioning runbooks
│   │       └── document-platform/ ← Document platform maintenance runbooks
│   │
│   ├── applications/
│   │   ├── erp/                   ← ERP system entities and runbooks
│   │   ├── low-code-platform/     ← Low-code / automation platform entities
│   │   └── runbooks/              ← App-specific runbooks
│   │
│   ├── cloud/
│   │   ├── cloud-vms/             ← Cloud VM entities
│   │   ├── cloud-storage/         ← Object / blob storage entities
│   │   └── document-platform/     ← Cloud document platform entity
│   │
│   ├── security/
│   │   └── identity/              ← Service accounts, MFA, access policies
│   │
│   ├── service-desk/
│   │   ├── runbooks/              ← User-triggered incident runbooks
│   │   └── known-issues.md
│   │
│   ├── automation/
│   │   └── runbooks/              ← Scheduled/automated job runbooks
│   │
│   └── projects/
│       ├── active/                ← In-progress IT projects
│       └── completed/             ← Finished IT projects
│
└── templates/
    ├── TPL-Entity.md
    ├── TPL-Runbook.md
    ├── TPL-Project.md
    └── TPL-Decision.md
```

### users-vault (full)

```
users-vault/
├── AGENTS.md
├── README.md
├── index.md
├── log.md
│
├── wiki/
│   ├── how-to/
│   │   ├── document-platform/   ← e.g. finding-archived-files.md
│   │   ├── vpn/                 ← VPN client connection guides
│   │   └── email/               ← Email client tips, shared mailboxes
│   │
│   ├── known-issues/            ← Current outages/workarounds
│   └── glossary/                ← Plain-language IT terms
│
└── templates/
    └── TPL-HowTo.md
```

---

## 4. Bootstrapping from Scratch

### Prerequisites
- Git installed with SSH keys configured for your remote (Azure DevOps / GitHub)
- Obsidian installed and opened on the workspace folder
- Bash available (WSL, Git Bash on Windows, or native on macOS/Linux)
- An AI assistant that can read `AGENTS.md` (Antigravity, Claude, GPT, Gemini)

### Step 1: Create the workspace

```bash
mkdir Obsidian && cd Obsidian

# Clone or create both repos as siblings
git clone ssh://git@your-remote/it-vault
git clone ssh://git@your-remote/users-vault
```

Or initialize them fresh:
```bash
mkdir it-vault && cd it-vault && git init && cd ..
mkdir users-vault && cd users-vault && git init && cd ..
```

### Step 2: Scaffold directory structure

```bash
# it-vault
cd it-vault
mkdir -p raw/configs raw/scripts
mkdir -p wiki/infrastructure/{servers/virtual-machines,storage,network/firewalls,monitoring,runbooks/{network,sharepoint}}
mkdir -p wiki/applications/{erp,power-platform,runbooks}
mkdir -p wiki/cloud/{azure-vms,azure-storage,sharepoint}
mkdir -p wiki/security/identity
mkdir -p wiki/service-desk/runbooks
mkdir -p wiki/automation/runbooks
mkdir -p wiki/projects/{active,completed}
mkdir -p templates

# users-vault
cd ../users-vault
mkdir -p wiki/{how-to,known-issues,glossary}
mkdir -p templates
```

### Step 3: Create core files

Create `index.md`, `log.md`, `AGENTS.md`, `README.md` for each vault from the content in this document (Sections 7–8 for AGENTS.md; adapt Section 3 layouts for index.md skeleton).

Create templates from Section 6.

Copy `sync-wiki.sh` from Section 9 into the root of `it-vault`.

```bash
chmod +x it-vault/sync-wiki.sh
```

### Step 4: Open in Obsidian

Open Obsidian → **Open folder as vault** → select the `Obsidian/` workspace root.

> **Note:** Open the `Obsidian/` parent as the vault root (not `it-vault/` or `users-vault/`) so both vaults appear as folders and wikilinks can resolve across them.

### Step 5: Configure your AI assistant

Point your AI assistant (Antigravity, Claude Projects, GPT Custom Instructions) at `AGENTS.md` in each vault. The assistant should read the file at the beginning of each session. The rules define its behavior entirely — no additional prompting is needed.

---

## 5. Conventions

### File Naming
- **Always kebab-case**: `reset-user-mfa.md`, `ngfw-waw.md`
- **Never spaces or underscores in wiki pages**
- **Raw files may retain original names**: `VSH-FS02_Syno_ConfBkp.db`

### YAML Frontmatter
Every wiki page must have a YAML frontmatter block. See templates in Section 6. Key rules:

| Property | Rule |
|----------|------|
| `type` | Required. `entity`, `runbook`, `project`, `decision` |
| `status` | AI always sets new runbooks to `draft`. Only humans change to `active`. |
| `sources` | Use `"[[raw/path/to/file]]"` format for Obsidian clickable links |
| `credentials` | Use `{{SECRET:vault/item-name}}` placeholder only. Never real values. |
| `tested-by` | Left blank by AI. Filled in by the human who verified the runbook. |
| `reviewed-by` | Left blank by AI. Filled in by an approver. |

### Internal Links
Use Obsidian `[[wikilinks]]` for all internal references:
```
[[wiki/infrastructure/storage/vsh-fs02-nas|VSH-FS02 NAS]]
```

Always include a display name after the `|` pipe for readability.

### Cross-Vault Pseudolinks
These are **not functional Obsidian links** — they are navigation hints for humans and AI context:
```
@IT:infrastructure/runbooks/network/enable-china-access
@USERS:how-to/sharepoint/finding-archived-files
```
Omit the `wiki/` prefix in pseudolinks.

### Tags
Kebab-case only: `#active-directory`, `#sharepoint`, `#powershell`

### Code Blocks
Always specify the language:
````
```powershell
Get-Item -Path "C:\example"
```
````

### Runbook Classification

| Trigger Type | Location |
|-------------|----------|
| User reports incident / needs help | `wiki/service-desk/runbooks/` |
| IT maintenance, provisioning, lifecycle | `wiki/infrastructure/runbooks/` or `wiki/applications/runbooks/` |
| Automated/scheduled job documentation | `wiki/automation/runbooks/` |

---

## 6. Page Templates

### TPL-Entity.md
```yaml
---
type: entity
entity-type: server | service | application | network-device | cloud-platform
name: "{{name}}"
status: active | deprecated | planned
environment: production | staging | dev
location: azure-westeurope | on-prem | warsaw-dmz
owner: "{{team}}"
tags: [tag1, tag2]
sources: ["[[raw/path/to/source]]"]
credentials: "{{SECRET:vault/item}}"
reviewed-by: 
last-reviewed: 
---

# Overview
Provide a high-level description of what this entity is and its role in the organization.

# Technical Details
- **Architecture:** Link to diagram or description
- **IP Address:** `x.x.x.x`
- **DNS Hostnames:** `hostname.domain.local`
- **Dependencies:**
  - [[related-entity]]

# Support & Contact
- **Owner Team:**
- **Primary Contact:**
- **Emergency Escalation:**

## Related Docs
- [[wiki/...]]
```

### TPL-Runbook.md
```yaml
---
type: runbook
title: "{{title}}"
category: infrastructure | service-desk | applications | automation
difficulty: easy | medium | hard
estimated-time: "5 min"
tags: [tag1, tag2]
status: draft
sources: ["[[raw/scripts/category/script.ps1]]"]
tested-by: 
last-tested: 
reviewed-by: 
review-date: 
pseudolink: "@USERS:how-to/category/guide-name"
---

# Problem Statement
Briefly describe the issue this runbook solves and why it exists.

# Prerequisites
- [ ] Prerequisite 1
- [ ] Prerequisite 2

# Instructions
1. Step one
2. Step two
   - **Command:**
     ```powershell
     example command
     ```
3. Step three

# Verify
Describe exactly how to confirm the runbook succeeded.
- [ ] Verification check 1
- [ ] Verification check 2

## Related Docs
- [[wiki/...]]
```

### TPL-Project.md
```yaml
---
type: project
title: "{{title}}"
status: active | on-hold | completed | cancelled
priority: high | medium | low
start-date: YYYY-MM-DD
target-date: YYYY-MM-DD
owner: "{{name}}"
reviewed-by: 
tags: [project, tag]
sources: ["[[raw/...]]"]
---

# Project: {{title}}

## Objective
What are we trying to achieve and why?

## Success Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Stakeholders
- **Lead:**
- **Team:**
- **External:**

## Resources
- Architecture: [[wiki/...]]
- Scripts: [[raw/scripts/...]]

## Progress Log
| Date | Milestone | Status |
|------|-----------|--------|
| YYYY-MM-DD | Project Start | ✅ |
```

### TPL-Decision.md (Architecture Decision Record)
```yaml
---
type: decision
title: "ADR: {{title}}"
date: YYYY-MM-DD
status: proposed | accepted | superseded | deprecated
deciders: [person1, person2]
tags: [adr, architecture]
---

# ADR: {{title}}

## Context and Problem Statement
What is the issue we are solving? Why now?

## Decision Drivers
- Driver 1
- Driver 2

## Considered Options
1. Option A
2. Option B

## Decision Outcome
Chosen option: **Option A**, because [justification].

### Consequences
- **Good:** Advantage
- **Bad:** Trade-off
```

### TPL-HowTo.md (users-vault)
```yaml
---
type: how-to
title: "{{title}}"
category: sharepoint | vpn | email | general
tags: [tag1]
status: published
reviewed-by:
last-reviewed:
it-source: "@IT:service-desk/runbooks/category/runbook-name"
---

# How To: {{title}}

## What This Guide Covers
One sentence describing what the user will accomplish.

## Before You Start
- You need: [software / access / role]

## Steps
1. Step one
2. Step two

> [!TIP]
> Helpful hint here.

## It Didn't Work
If something goes wrong, [open a support ticket](YOUR_TICKETING_SYSTEM_URL) and describe what happened.
```

---

## 7. AGENTS.md — IT Vault

Save this file as `it-vault/AGENTS.md`. This is the operating contract for your AI assistant.

```markdown
# IT Wiki Schema

## Identity
You are the IT Knowledge Base maintainer for [COMPANY].

## Architecture
- `raw/` — immutable sources. Never modify.
- `wiki/` — your workspace. Create, update, cross-link here.
- `templates/` — use for new pages.
- `index.md` — update after every operation.
- `log.md` — append after every operation.
- Users vault at `../users-vault/` — draft user-facing content there.
- Sync tool: `sync-wiki.sh` — remind the human to run this after they commit.

## Conventions
- [[wikilinks]] for internal references
- YAML frontmatter on every page (see templates)
- **Clickable Sources:** In frontmatter, use `sources: ["[[raw/path/to/file.ext]]"]` to ensure Obsidian property navigation works.
- Tags: kebab-case (#active-directory)
- Files: kebab-case (reset-user-mfa.md)
- Code blocks: always specify language
- Pseudolinks: @USERS:path and @IT:path for cross-vault refs (omit 'wiki/' prefix)

## Credential Security
- NEVER store passwords, API keys, tokens, or secrets.
- NEVER suggest real URLs for password managers (e.g., `bwl://`, `op://`).
- Use placeholders only: `{{SECRET:vault/item-name}}`.
- Document purpose and permissions, not the credentials.

## Hallucination Prevention
- New runbooks: always `status: draft`.
- Never set `tested` or `approved` — only humans do that.
- Prefer exact commands over descriptions.
- Include "Verify" step in every runbook.
- If unsure: > [!WARNING] Unverified — test before use.
- **NEVER perform autonomous Git commits. Stop and wait for human review.**

## Retrieval & Reasoning
Universal protocol for answering ANY query (Access, How-to, Historical, Architectural):

1. **Tier 1: The Truth (Wiki Search)**
   - Check `index.md` for the most recent updates related to the query.
   - Search `wiki/` for explicit documentation, entities, or runbooks related to the query.
   - If found: Provide the documented answer and link the source.

2. **Tier 2: The Evidence (Raw Ingest)**
   - Audit `raw/` files (configs, chat exports, meeting notes, tickets) for factual mentions or historical context.
   - Use `grep` or `Select-String` to find "fragmented knowledge" that hasn't been wikified yet.

3. **Tier 3: The Logic (Architectural Deduction)**
   - If Tiers 1 & 2 fail: Use core system blueprints to deduce the most likely state.
   - Example (Network): Deduce path via Subnet/VLAN mapping.
   - Example (Process): Deduce steps based on similar existing SOPs or compliance rules.

4. **Tier 4: The Mirror (Brutal Honesty)**
   - If deduction is impossible or weak: **Do not guess and do not validate assumptions.**
   - Call out the "Ghost Entity" — e.g., "This [User/Server/Process] is a ghost; it is documented nowhere."
   - State exactly where the knowledge gap is and what evidence is required to bridge it.

## Continuous Ingestion
- **Proactive Wikification:** If a chat discussion contains new technical facts, decisions, or troubleshooting steps, proactively ask: "Should I add this to the wiki as a new entry or runbook?"
- **Resource Requests:** If a discussion involves configurations (JSON, XML, YAML) or documentation that exists outside the wiki, ask: "Can you provide the source file or config to be stored in `raw/` for better grounding?"

## Assisted Drafting
1. When asked to "draft from source" or "draft from chat":
2. Generate markdown using correct templates.
3. Identify entities → `wiki/infrastructure/`, `wiki/applications/`, `wiki/cloud/`, `wiki/service-desk/` etc.
4. **Runbook Classification Rule:**
   - **Trigger = Incident/Troubleshooting (User Help)** → Draft in `wiki/service-desk/runbooks/`.
   - **Trigger = Maintenance/Provisioning/lifecycle** → Draft in `wiki/infrastructure/runbooks/`, `wiki/applications/runbooks/` etc.
5. If user-relevant → draft a version for the Users vault in the `../users-vault/` path.
5. Add `@USERS:` and `@IT:` pseudolinks to both files.
6. Present paths and content to the human. Wait for explicit approval.
7. Remind the human to run `sync-wiki.sh` after they have reviewed and committed. Never run it yourself.

## Lint
- [ ] Orphan pages
- [ ] Missing entity pages
- [ ] Runbooks with last-tested > 90 days
- [ ] Runbooks still in `draft` after 7 days
- [ ] Projects past target-date still active
- [ ] Missing frontmatter
- [ ] Credentials or secrets in any page
- [ ] Broken pseudolinks
```

---

## 8. AGENTS.md — Users Vault

Save this file as `users-vault/AGENTS.md`.

```markdown
# IT Wiki Schema (Users Vault)

## Identity
You are the IT Help Center assistant. Your goal is to provide clear,
human-readable troubleshooting steps and "how-to" guides for company employees.

## Architecture
- `wiki/how-to/subfolder` — step-by-step guides.
- `wiki/known-issues/` — current outages and workarounds.
- `wiki/glossary/` — simple definitions of IT terms.
- IT vault at `../it-vault/` — references for the IT team.

## Conventions
- Use plain language. Avoid jargon.
- Use numbered lists for steps.
- Use `> [!TIP]` for helpful hints.
- Pseudolinks: @IT:path to link to technical docs in the IT vault (omit 'wiki/' prefix).

## Security
- NEVER include passwords, links to secrets, or technical infra details.
- NEVER mention internal IP addresses, server names, or firewall rules.
- If a user asks for access or a password, direct them to the ticketing system.

## Ingest Rules
- Only process content pushed by the IT team.
- Ensure all pages have proper titles and tags.

## Continuous Improvement
- **Identify Gaps:** If a user explains a workaround or if you solve a complex
  issue that isn't documented, ask the IT team: "Should I draft a new 'how-to'
  guide for this in the Help Center?"

## User Access Queries
- If a user asks "Can I access X?", "How do I reach Y?":
- **Search first:**
  1. Search for related 'index.md' updates related to the query.
  2. Only answer if there is an explicit `wiki/how-to` guide.
- **Protocol:**
  1. Search for explicit how-to guides.
  2. If missing: Refuse to guess technical details.
  3. Redirect to Help Desk: "Your access to this resource may require specific
     permissions. Please check with your manager or open a ticket."
```

---

## 9. sync-wiki.sh

Save in `it-vault/sync-wiki.sh`. Run it **after** you have reviewed and committed changes in each vault.

```bash
#!/bin/bash
# sync-wiki.sh - Transactional Sync for Dual-Vault IT Wiki
# Usage: ./sync-wiki.sh
# Run from inside it-vault/ — it will find both sibling repos automatically.

WORKSPACE_ROOT=$(cd "$(dirname "$0")/.." && pwd)

sync_repo() {
    local REPO_NAME=$1
    echo "--- Checking $REPO_NAME ---"

    if [ ! -d "$WORKSPACE_ROOT/$REPO_NAME" ]; then
        echo "⚠️  Error: Directory $REPO_NAME not found at $WORKSPACE_ROOT."
        return
    fi

    cd "$WORKSPACE_ROOT/$REPO_NAME" || return

    # 1. Pull latest to avoid conflicts
    git pull --rebase

    # 2. Check for changes
    if [[ -n $(git status -s) ]]; then
        echo "📦 Found changes in $REPO_NAME. Committing..."
        git add .
        git commit -m "sync: $(date +%Y-%m-%d_%H:%M:%S)"

        # 3. Push
        if git push; then
            echo "✅ $REPO_NAME pushed successfully."
        else
            echo "❌ Error pushing $REPO_NAME. Resolve conflicts and retry."
        fi
    else
        echo "🍃 No changes in $REPO_NAME."
    fi
}

sync_repo "it-vault"
sync_repo "users-vault"

cd "$WORKSPACE_ROOT"
echo "-----------------------------------"
echo "✅ Workflow complete."
```

> **Windows note:** Run with Git Bash (`sh ./sync-wiki.sh`) or WSL. PowerShell is not compatible with this script.

---

## 10. Human Workflow

### Daily: Answering IT Questions

1. Open Obsidian. The AI assistant has access to the vault via the open files and directory context.
2. Ask the AI your question in natural language.
3. The AI follows the 4-Tier retrieval protocol: Wiki → Raw → Deduction → Brutal Honesty.
4. If the answer is undocumented, the AI will say so and ask whether to create a new entry.

### Adding New Knowledge (from chat or incident)

1. Discuss the issue or change with the AI.
2. Say: *"Draft a runbook from this discussion."* or *"Wikify this."*
3. The AI proposes file path(s) and content. Review them.
4. Approve, edit, or reject. The AI waits for explicit confirmation before writing.
5. After writing, place any relevant source files in `raw/`.
6. Commit your changes → run `./sync-wiki.sh`.

### Adding a New Entity (server, device, service)

1. Place any config backup or export in `raw/configs/<category>/`.
2. Say: *"Draft an entity page for [entity name] from this config."*
3. Review the draft. The AI fills in what it can from the source; it leaves the rest blank rather than guessing.
4. Set `reviewed-by` and `last-reviewed` after you verify technical details.
5. Link it from `index.md`. Commit → sync.

### Periodic Maintenance (monthly)

Run the lint checklist (Section 12) with the AI:

> "Run a lint audit on the wiki."

The AI will check `index.md` for broken links, flag runbooks still in `draft` older than 7 days, identify pages without frontmatter, and report any pseudolinks that point nowhere.

---

## 11. AI Assistant Rules Summary

This table is a quick reference for humans on what the AI will and will not do.

| Action | AI Behavior |
|--------|------------|
| Draft a runbook | ✅ Yes — always sets `status: draft` |
| Set a runbook to `active` | ❌ Never — humans only |
| Set `tested-by` or `approved` | ❌ Never — humans only |
| Store a password or token | ❌ Never — uses `{{SECRET:...}}` placeholder |
| Commit to git | ❌ Never — reminds human to do it |
| Run `sync-wiki.sh` | ❌ Never — reminds human to do it |
| Add `> [!WARNING]` when uncertain | ✅ Yes — always |
| Guess when documentation is missing | ❌ Never — calls it a "Ghost Entity" |
| Propose cross-vault content in `users-vault` | ✅ Yes — when content is user-relevant |
| Update `index.md` and `log.md` | ✅ Yes — after every write operation |

---

## 12. Lint & Quality Checklist

Run this manually or ask the AI to audit. Check after any significant content change.

### Structural
- [ ] Every new page is linked from `index.md`
- [ ] Every new page has a `log.md` entry
- [ ] No wiki pages exist outside of `wiki/` (no accidental files in root)

### Frontmatter
- [ ] All pages have YAML frontmatter
- [ ] All runbooks have `sources:` linking to their `raw/` counterpart
- [ ] No page contains real credentials, IPs in `credentials:`, or secret tokens
- [ ] `status: draft` runbooks older than 7 days are flagged for review
- [ ] `last-tested` not older than 90 days for active runbooks

### Content
- [ ] Every runbook has a `# Verify` section
- [ ] All code blocks specify a language
- [ ] All wikilinks use display names (`[[path|Display Name]]`)
- [ ] All pseudolinks use the correct format (`@IT:` or `@USERS:`)
- [ ] No orphan pages (pages that are not linked to from anywhere)

### Security
- [ ] No real passwords, API keys, or PATs in any file
- [ ] No real URLs for password vaults (e.g. `bitwarden://`, `1password://`)
- [ ] `raw/` files are not modified (verify via `git diff raw/`)

---

## 13. Design Decisions & Rationale

### Why Obsidian?
Obsidian stores everything as plain markdown files. No vendor lock-in. The vault survives the death of any software. Git handles versioning. The AI reads plain files natively.

### Why Two Separate Git Repositories?
Keeping `it-vault` and `users-vault` as separate repos enforces the access boundary at the git level. `it-vault` can contain sensitive technical detail (network addresses, firewall rules, diagnostic scripts). `users-vault` is safe to give broader read access.

### Why `raw/` is Immutable
If a runbook was written from a specific version of a config file, that config becomes auditable evidence. Modifying `raw/` would break the chain of provenance. Updated configs should be added as new files (with dates or versions in the name).

### Why the AI Never Commits
The commit is the human's signature. It means "I reviewed this, I believe it's correct, and I am responsible for it." An AI that auto-commits would undermine the entire quality model of the system.

### Why `status: draft` is the Invariant Default
AI-generated runbooks can look authoritative but contain errors. The `draft` status is a mandatory speed bump that forces a human to consciously change it — implying they have read, tested, and approved the content. Without this, the AI's confidence would set the quality bar, not the human's.

### Why `sync-wiki.sh` Lives Inside `it-vault`
The script manages both vaults and therefore must be version-controlled. Storing it in the parent directory (outside any repo) meant it was invisible to git and could be silently changed or lost. Placing it inside `it-vault` makes it part of the IT team's managed tooling.

---

*This implementation guide is itself version-controlled. To update it: edit, commit, and push like any other wiki page. The AI can assist in keeping it up to date.*
