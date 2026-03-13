# pleaseai Workflow

![License](https://img.shields.io/badge/license-Proprietary-red)

This repository contains the [Work Please](https://github.com/pleaseai/work-please) orchestrator configuration for the `pleaseai` organization.

Work Please is an autonomous task runner that polls a GitHub Projects board for active issues and automatically launches a Claude Code agent for each issue. The agent handles the entire lifecycle — branch creation, code implementation, test execution, and PR opening — without human intervention.

---

## Prerequisites

| Requirement | Version |
|-------------|---------|
| [Bun](https://bun.sh) | ≥ 1.0 |
| [Claude Code CLI](https://github.com/anthropics/claude-code) | latest |
| [Infisical CLI](https://infisical.com/docs/cli/overview) | latest |
| GitHub App (installation token) | `contents`, `pull_requests`, `issues`, `projects` permissions |

---

## Installation

### Option A — Install globally with Bun

```bash
bun add -g @pleaseai/work
```

### Option B — Build from source

```bash
git clone https://github.com/pleaseai/work-please.git
cd work-please
bun install
bun run build
```

---

## Environment Configuration

Secrets are managed via [Infisical](https://infisical.com). Register the following secrets in your Infisical project:

| Secret Key | Description |
|------------|-------------|
| `GITHUB_APP_ID` | GitHub App ID |
| `GITHUB_APP_PRIVATE_KEY` | GitHub App Private Key (PEM format) |
| `GITHUB_APP_INSTALLATION_ID` | GitHub App Installation ID |

Work Please uses these credentials to automatically generate an installation token. Token issuance is handled by the orchestrator — no separate script required.

### Infisical CLI Login

```bash
infisical login
```

After registering the three secrets in your Infisical project, run via `infisical run` and they will be injected automatically.

---

## Running

Clone this repository to your preferred location:

```bash
git clone https://github.com/pleaseai/workflow.git
cd workflow
```

Start the orchestrator with secrets injected via Infisical:

```bash
# Run with Infisical secret injection (recommended)
infisical run -- work-please WORKFLOW.md

# Enable HTTP status dashboard on port 6001
infisical run -- work-please --port 6001
```

The orchestrator polls GitHub Projects board #2 every 30 seconds and dispatches agents for issues in **Todo** or **In Progress** status.

---

## HTTP Dashboard

When started with `--port 6001` (already configured in `WORKFLOW.md`), a status dashboard is available at:

```
http://localhost:6001
```

It shows current agent status, retry queue, token usage, and rate limit information.

---

## Configuration Overview (`WORKFLOW.md`)

| Setting | Value |
|---------|-------|
| Tracker | [GitHub Projects v2 (project #2)](https://github.com/orgs/pleaseai/projects/2/views/1) |
| Active statuses | `Todo`, `In Progress` |
| Terminal statuses | `Done`, `Cancelled` |
| Poll interval | 30 seconds |
| Workspace root | `~/workspaces` |
| Max concurrent agents | 5 |
| Max turns per agent | 20 |
| Claude permission mode | `bypassPermissions` |
| HTTP server port | 6001 |

The `before_run` hook runs before each agent session starts:

```bash
# Inject latest secrets from Infisical (including GitHub App token refresh)
eval "$(infisical export --format=dotenv-export)"
git fetch origin
git rebase origin/main || git rebase --abort
```

This ensures each agent session starts with the latest secrets from Infisical and a workspace that is up to date with `main`.

For the full configuration reference, see the [Work Please documentation](https://github.com/pleaseai/work-please).

---

## How It Works

```
Poll (every 30 seconds)
  └─ Fetch active issues from GitHub Projects
       └─ Filter out issues with blockers
            └─ Dispatch Claude Code agent per issue (up to 5 concurrent)
                 └─ Agent: create branch → implement → test → commit → open PR
                      └─ Move issue to "In Review" when PR is created
```

1. **Polling** — Work Please queries the GitHub Projects v2 board for issues in `Todo` or `In Progress` status.
2. **Dispatch** — For each eligible issue, a dedicated workspace directory and Claude Code agent session are created under `~/workspaces`.
3. **Agent execution** — The agent reads the issue, creates a feature branch, implements the changes, runs tests, commits, and opens a PR.
4. **Retry** — If an agent exits without completing, Work Please retries with exponential backoff and a restart prompt.
5. **Cleanup** — Workspaces for issues that move to a terminal status (`Done`, `Cancelled`) are automatically cleaned up.
