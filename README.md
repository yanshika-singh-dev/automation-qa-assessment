# Task 2 — n8n API Integration Workflow

## Overview
**Workflow:** `Task2_Workflow_[YourName].json`  
**Purpose:** Hourly GitHub "Morning Brief" — fetches top automation repos, enriches the #1 result with its README, and posts a digest to Discord.

---

## APIs Used

| API | Endpoint | Why |
|-----|----------|-----|
| **GitHub Search API** | `GET /search/repositories?q=topic:automation&sort=stars` | Free, no auth required for read; rich structured data (stars, language, description) |
| **GitHub Contents API** | `GET /repos/{owner}/{repo}/readme` | Second API call to enrich the top result with its actual README text |
| **Discord Webhook** | `POST {webhookUrl}` | Free, no bot setup needed — a single URL is all that's required |

---

## Workflow Steps

```
Schedule (1h) → GitHub Search → Transform: Top 5 → IF Stars ≥ 1000
                                                        ├─ TRUE  → Fetch README → Merge → Build Payload → Discord
                                                        └─ FALSE → Tag RISING  → Merge ↗
                                                        
Any node fails → Build Error Payload → Post Error to Discord
```

1. **Schedule Trigger** fires every hour.
2. **GitHub Search** fetches the top 10 repos tagged `automation` with 100+ stars.
3. **Transform (Code node)** slices to top 5, picks only the fields we need (name, stars, language, URL).
4. **IF node** splits on `stars >= 1000`:
   - **True (HOT):** fetches the raw README via a second GitHub API call and prepends a 300-char snippet to the embed.
   - **False (RISING):** skips the README fetch (saves API quota) and tags the repo as RISING.
5. **Merge** reunites both branches.
6. **Build Discord Payload (Code node)** assembles a rich Discord embed with 🔥/📈 badges.
7. **Send to Discord** POSTs the embed. On failure the error output routes to **Build Error Payload → Post Error to Discord** so failures are never silent.

---

## Error Handling

- `GitHub Search Repos` and `Send to Discord` both have **`onError: continueErrorOutput`** — if either fails, execution branches to the error path instead of stopping.
- The error path builds a separate Discord embed with the raw error message so the team is notified immediately.
- The `Fetch README` node also has error output wired to the same handler.

---

## Credentials Setup (n8n)

| Credential Name | Type | Value |
|-----------------|------|-------|
| `GitHub Token` | HTTP Header Auth | Header: `Authorization`, Value: `token ghp_XXXX` |
| `Discord Webhook URL` | HTTP Query Auth | Header `webhookUrl` = your Discord webhook URL |

> **Never hard-code secrets.** Both credentials are stored in n8n's Credentials store and referenced by name in nodes.

---

## Bonus: Uptime Monitor (`Bonus_UptimeMonitor.json`)

- **Trigger:** Every 5 minutes (+ daily summary at 08:00 UTC).
- **Pings** `https://demo.realworld.io` with a 10-second timeout.
- **Measures** response time in the Code node.
- **IF node:** healthy = status 200 AND response < 5000 ms.
  - ✅ Healthy → silent console log (no Discord noise).
  - ❌ Down/slow → posts an alert embed to Discord with status code, response time, and timestamp.
- **Timeout/error** (app completely unreachable) is caught via `continueErrorOutput` and routed through the same alert path.
- **Daily summary** at 08:00 posts a digest — in production this would query a log store; the template shows the pattern.
