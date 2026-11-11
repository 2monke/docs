---
status: shipped
collections: []
features: []
---


# Vercel Log Error Parser & GitHub Issue Worker

Out-of-band automated runtime error detection and GitHub Issue dispatcher for production monitoring and AI agent triage. Shipped 2026-09-26.

## Scope & Objective

- **Audience**: Autonomous coding agents (Antigravity, GitHub Copilot) and developers.
- **Zero Runtime Overhead**: The web application does **not** make extra network calls, install heavy client SDKs, or run telemetry loops. The application logs normally (`console.error`), which Vercel runtime captures automatically.
- **Out-of-Band Sync**: A scheduled GitHub Action (or local CLI script) runs on a cron schedule, queries Vercel's REST API for runtime logs, parses errors, deduplicates against existing issues, and automatically opens GitHub Issues labeled `agent-task`.
- **100% Free**: Operates entirely within GitHub Actions free tier (720 min/month vs 2,000 min allowance) and Vercel's standard REST API.

---

## Architecture

```
┌──────────────────────────────────────┐
│  Next.js 15 Application on Vercel    │
│  (Normal runtime: console.error)     │
└──────────────────┬───────────────────┘
                   │
                   ▼ (Captured automatically by Vercel)
┌──────────────────────────────────────┐
│  Vercel Runtime Logs / Events        │
└──────────────────┬───────────────────┘
                   │
                   │ GET /v2/deployments/{id}/events (Vercel REST API)
                   ▼
┌────────────────────────────────────────────────────────┐
│  GitHub Action / Worker Script                         │
│  (scripts/check-vercel-errors.ts)                      │
│                                                        │
│  1. Resolves latest production deployment              │
│  2. Fetches logs over the lookback window (e.g. 60m)   │
│  3. Filters for 5xx status codes, exceptions, errors   │
│  4. Generates stable fingerprint (hash of error+frame) │
│  5. Queries GitHub for existing open issues            │
│  6. Creates GitHub Issue for new unique errors:        │
│     - Labels: ['bug', 'vercel-error', 'agent-task']    │
│     - Embeds stack trace & agent prompt instructions   │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│  GitHub Issues (labeled 'agent-task')                  │
│  Autonomous AI agent (Antigravity) picks up the issue,  │
│  reproduces with a test in tests/int/, and submits PR! │
└────────────────────────────────────────────────────────┘
```

---

## File Map

- `scripts/check-vercel-errors.ts` — Standalone worker script that connects to Vercel REST API, parses log errors, computes SHA-256 fingerprints, deduplicates against existing issues, and creates GitHub issues with agent instructions.
- `.github/workflows/check-vercel-errors.yml` — Scheduled GitHub Actions workflow (runs hourly, with manual `workflow_dispatch` trigger).
- `tests/int/check-vercel-errors.int.spec.ts` — Integration tests verifying log parsing, error fingerprinting, deduplication matching, and GitHub payload formatting.
- `package.json` — Added `"check:vercel-errors": "tsx scripts/check-vercel-errors.ts"` script.

---

## Setup & Required Secrets

To activate the automated GitHub Action, add two secrets to your GitHub repository (**Settings $\rightarrow$ Secrets and variables $\rightarrow$ Actions**):

1. **`VERCEL_TOKEN`**: A Vercel Personal Access Token generated at [Vercel Account Tokens](https://vercel.com/account/tokens).
2. **`VERCEL_PROJECT_ID`**: Found in your Vercel project settings (**Settings $\rightarrow$ General $\rightarrow$ Project ID**).
3. **`VERCEL_TEAM_ID`** *(Optional)*: Required only if your Vercel project belongs to a team/organization instead of a personal account.

> **Note on `GITHUB_TOKEN`**: The workflow uses GitHub's built-in `${{ secrets.GITHUB_TOKEN }}` with `issues: write` permission. No manual personal access token is required for GitHub.

---

## Manual Execution & Local Testing

You can run the script locally anytime to inspect recent production errors:

```bash
# Dry run: Inspect errors without creating GitHub issues
VERCEL_TOKEN="your_token" VERCEL_PROJECT_ID="your_project_id" DRY_RUN="true" npm run check:vercel-errors

# Real run: Query past 2 hours and open issues
VERCEL_TOKEN="your_token" VERCEL_PROJECT_ID="your_project_id" GITHUB_TOKEN="ghp_xxx" LOOKBACK_MINUTES="120" npm run check:vercel-errors
```

---

## AI Agent Integration

Issues created by this worker include structured metadata and an instruction prompt for autonomous agents:

- **Labeling**: `bug`, `vercel-error`, `agent-task`.
- **Deduplication Tag**: An embedded HTML comment `<!-- vercel-error-fingerprint: <hash> -->` ensures that repeated occurrences within log windows never spam the issue tracker.
- **Workflow**: When an issue is created, you can prompt Antigravity in chat:
  > *"Review issue #X labeled `agent-task`, reproduce the error with a test, and fix it."*
  Antigravity fetches the issue via GitHub MCP / REST API, analyzes the stack trace, writes a test in `tests/int/`, applies the fix, and opens a Pull Request.
