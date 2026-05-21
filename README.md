# Task 2 — n8n API Integration Workflow

## Overview
**Workflow:** `Task2_Workflow.json`  
**Purpose:** Hourly GitHub "Morning Brief" — fetches top automation repos, enriches the #1 result with its README, and posts a digest to Discord.

---

## APIs Used

| API | Endpoint | Auth |
|-----|----------|------|
| **GitHub Search API** | `GET /search/repositories?q=topic:automation&sort=stars` | None (public API) |
| **GitHub Contents API** | `GET /repos/{owner}/{repo}/readme` | None (public API) |
| **Discord Webhook** | `POST {webhookUrl}` | Webhook URL (stored directly in node) |

---

## Workflow Steps

```
Schedule (1h) → GitHub Search → Transform: Top 5 → IF Stars ≥ 1000
                                                        ├─ TRUE  → Fetch README → Merge → Build Payload → Discord
                                                        └─ FALSE → Tag RISING  → Merge ↗
                                                        
Any node fails → Build Error Payload → Post Error to Discord
```

1. **Schedule Trigger** fires every hour.
2. **GitHub Search** fetches the top 10 repos tagged `automation` with 100+ stars. No auth required — GitHub's public search API allows unauthenticated requests.
3. **Transform (Code node)** slices to top 5, picks only the fields we need (name, stars, language, URL).
4. **IF node** splits on `stars >= 1000`:
   - **True (HOT):** fetches the raw README via a second GitHub API call and prepends a 300-char snippet to the embed.
   - **False (RISING):** skips the README fetch and tags the repo as RISING.
5. **Merge** reunites both branches.
6. **Build Discord Payload (Code node)** assembles a rich Discord embed with 🔥/📈 badges.
7. **Send to Discord** POSTs the embed via Discord Webhook URL.

---

## Error Handling

- `GitHub Search Repos`, `Fetch README`, and `Send to Discord` all have **`onError: continueErrorOutput`** — if any fails, execution branches to the error path instead of stopping.
- The error path builds a separate Discord embed with the raw error message so the team is notified immediately.
- **Proven working** — error alerts were successfully delivered to Discord during testing.

---

## Credentials Setup

| Node | Type | Value |
|------|------|-------|
| `Send to Discord` | URL field | Discord Webhook URL pasted directly |
| `Post Error to Discord` | URL field | Same Discord Webhook URL |
| `GitHub Search Repos` | Authentication | None (public API) |
| `Fetch README` | Authentication | None (public API) |

> GitHub's public REST API allows up to 60 unauthenticated requests per hour — sufficient for this hourly workflow.

---

## Bonus: Uptime Monitor (`Bonus_UptimeMonitor.json`)

- **Trigger:** Every 5 minutes (+ daily summary at 08:00 UTC).
- **Pings** `https://demo.realworld.io` with a 10-second timeout.
- **Measures** response time in the Code node.
- **IF node:** healthy = status 200 AND response < 5000 ms.
  - ✅ Healthy → silent console log (no Discord noise).
  - ❌ Down/slow → posts an alert embed to Discord with status code, response time, and timestamp.
- **Timeout/error** caught via `continueErrorOutput` and routed through the same alert path.
- **Daily summary** at 08:00 posts a digest to the same Discord channel.
