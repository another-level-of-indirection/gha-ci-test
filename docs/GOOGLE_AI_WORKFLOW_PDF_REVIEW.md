# Review: Google AI "Buzz GitHub Workflows" PDF

This document reviews a Google AI Mode search result (shared as a PDF screenshot) that proposes bridging Buzz and GitHub with custom YAML workflows. It compares that proposal to what actually works on Buzz today, based on hands-on testing with this repo (`gha-ci-test`).

**Source:** Google AI Mode search screenshot — not official Buzz documentation. The PDF footer shows a Google Search URL.

---

## What the doc proposes

Two workflows in `.buzz/workflows/github-bridge.yml`:

### Direction A — Buzz → GitHub (auto-mirror on push)

- **Trigger:** `on: git: event: push` on relay branches
- **Runs on:** `buzz-agent-shell`
- **Steps:** `buzz/checkout@v1` → `buzz/shell-run@v1` → `git push github` with PAT → `buzz/channel-message@v1`

### Direction B — GitHub → Buzz

- **Trigger:** `on: webhook: path: /github-receiver-endpoint`
- **Posts** GitHub payload to a Buzz channel via `buzz/channel-message@v1`

---

## What is real vs hallucinated

| PDF element | Status |
|---|---|
| `on: git: event: push` trigger | **Does not exist** — Buzz workflows support `webhook`, `message_posted`, `reaction`, `schedule`, `diff_posted` |
| `buzz/checkout@v1`, `buzz/shell-run@v1` | **Do not exist** — no shell execution or git checkout actions |
| `runs-on: buzz-agent-shell` | **Does not exist** |
| `.buzz/workflows/*.yml` in repo | **Not how workflows work** — they are channel-scoped Nostr events via `buzz workflows create` |
| `buzz/channel-message@v1` | **Does not exist** — real action is `send_message` |
| Custom webhook path `/github-receiver-endpoint` | **Wrong shape** — real URL is `POST /hooks/{workflow_id}` + `X-Webhook-Secret` |
| PAT + `git push` from inside a workflow | **Conceptually right, not implemented** |

The PDF is Google AI inventing a GitHub Actions dialect for Buzz. The syntax and capabilities are fabricated — it will not run if you try to create these workflows.

---

## Where it agrees with the verified approach

The **two-bridge mental model** is correct:

```
Buzz push/PR  →  mirror to GitHub  →  Actions runs
Actions finish  →  webhook  →  result in Buzz channel
```

We proved the **GitHub → Buzz** half works with real syntax:

```yaml
name: CI results to channel
trigger:
  on: webhook
steps:
  - id: post
    action: send_message
    text: |
      CI **{{trigger.status}}** on `{{trigger.ref}}`
      Commit: `{{trigger.sha}}`
      [View run]({{trigger.run_url}})
```

Create via:

```bash
buzz workflows create --channel <repo-channel-uuid> --yaml "$(cat workflow.yaml)"
```

The create response includes `workflow_id` and `webhook_secret` (returned once — save it).

**Webhook URL:** `POST https://lotf.communities.buzz.xyz/hooks/{workflow_id}`

**Auth:** `X-Webhook-Secret: {webhook_secret}` header or `?secret={webhook_secret}` query param.

**GitHub Actions curl step** (add to `.github/workflows/ci.yml`):

```yaml
- name: Notify Buzz
  if: always()
  env:
    BUZZ_HOOK: https://lotf.communities.buzz.xyz/hooks/YOUR_WORKFLOW_ID
    BUZZ_SECRET: ${{ secrets.BUZZ_WEBHOOK_SECRET }}
  run: |
    curl -fsS -X POST "$BUZZ_HOOK" \
      -H "Content-Type: application/json" \
      -H "X-Webhook-Secret: $BUZZ_SECRET" \
      -d "{\"status\":\"${{ job.status }}\",\"ref\":\"${{ github.ref }}\",\"sha\":\"${{ github.sha }}\",\"run_url\":\"${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\"}"
```

This was triggered twice on `gha-ci-test` — CI result messages appeared in the repo channel with `buzz:workflow` tag.

---

## The Buzz → GitHub half (PDF vs reality)

The PDF proposes auto-mirror on relay push using shell actions inside a workflow. That is the gap we already identified.

### What works today

1. **Manual mirror:** `git push origin` after `git push relay` (Desktop cannot push to GitHub)
2. **Near-term signal:** `diff_posted` trigger + `call_webhook` → GitHub `repository_dispatch` (signals Actions; does not copy git objects — still need mirror)

Example `call_webhook` workflow (requires **owner or admin** channel role to create):

```yaml
name: Trigger GitHub CI
trigger:
  on: diff_posted
steps:
  - id: dispatch
    action: call_webhook
    url: https://api.github.com/repos/ORG/REPO/dispatches
    method: POST
    headers:
      Authorization: "Bearer {{secrets.GITHUB_PAT}}"
      Accept: "application/vnd.github+json"
    body: '{"event_type":"buzz-pr","client_payload":{"ref":"main"}}'
```

### What the PDF envisions (not shipped)

- `git push` event trigger on relay branches
- Agent shell execution (`runs-on: buzz-agent-shell`)
- Automatic `git push github` with PAT from inside a workflow

This is a reasonable **product direction**, not shippable configuration today.

---

## Viability verdict

| Approach | Viable today? |
|---|---|
| PDF as written | **No** — syntax and capabilities are invented |
| PDF end-state (auto-mirror + channel notifications) | **Reasonable product direction**, not shippable config |
| Verified pattern (manual mirror + webhook `send_message` + optional `call_webhook` dispatch) | **Yes** — tested on `gha-ci-test` |

---

## Practical difference for CI testing

The PDF implies: push to relay → Buzz workflow auto-syncs to GitHub → Actions fire → results post to channel. **Zero terminal steps.**

Reality today:

1. Push to relay (relay-first repo only — see [README pointer section](../README.md#what-seed-the-pointer-means))
2. `git push origin` manually (Desktop cannot push to GitHub)
3. Actions run on GitHub
4. `curl` to Buzz webhook → `send_message` in channel

Step 2 is the friction the PDF hand-waves away. Until git-push triggers and shell actions ship, no YAML bridge eliminates it.

---

## Caveats from our debugging (apply to any bridge plan)

1. **Relay-first provisioning** — announce with relay clone URL on day one or push 404s (`buzz-tui` lesson)
2. **Buzz PR ≠ GitHub PR** — opening a Buzz PR does not trigger GHA; mirror first
3. **Desktop cannot push to GitHub** — mirror from terminal with `gh`/SSH
4. **`call_webhook` is gated** — only owner/admin can save outbound webhook workflows
5. **Webhook secret is one-time** — copy it from the create response; `workflows get` does not return it

---

## Recommendation

- **Do not** try to implement the PDF YAML — it will fail at workflow create
- **Do** use the verified pattern from this repo's [README](../README.md) and [Nick's checklist](../README.md#nicks-checklist--confirm-ci-will-work)
- **Treat the PDF as a product spec** for what Buzz workflows could become (git event triggers, agent shell, PAT secrets) — worth filing as a feature request if that UX is desired

---

## Suggested rollout on `gha-ci-test`

1. Create the webhook workflow in the `gha-ci-test` repo channel (owner/admin)
2. Add `BUZZ_WEBHOOK_SECRET` to GitHub repo secrets
3. Add the curl notify step to `ci.yml`, push to GitHub
4. Confirm pass/fail posts into the repo channel
5. Later: create the `diff_posted` + `call_webhook` workflow for the reverse direction (owner/admin required)
