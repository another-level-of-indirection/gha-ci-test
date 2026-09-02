# GHA CI Test

[![CI](https://github.com/another-level-of-indirection/gha-ci-test/actions/workflows/ci.yml/badge.svg)](https://github.com/another-level-of-indirection/gha-ci-test/actions/workflows/ci.yml)

A minimal working example of Buzz Projects + relay git + GitHub Actions. This repo exists because we hit real failures trying to wire CI into an older project (`buzz-tui`) and needed a clean place to prove the procedure.

---

## The simple version (Feynman)

Imagine you have **two filing cabinets** for the same project:

1. **Buzz relay** — where Buzz Desktop pushes code. Auth is automatic (your Nostr identity). This is the cabinet Buzz Projects opens when you push from the app.
2. **GitHub** — where GitHub Actions runs. It only sees what lands in *its* cabinet.

A **Buzz PR** is not a GitHub PR. It is a signed note on the relay saying "please merge these changes." GitHub never hears about it unless you also copy the files into GitHub's cabinet.

**The mistake we made:** we told Buzz "this repo lives at the relay" *after* the project had already been announced as GitHub-only. Buzz updated its address book (the Nostr announcement), but nobody ever built the relay filing cabinet. So when we pushed, the server said *repository not found* — not because auth failed, but because there was literally no storage for that address.

**The fix:** create the relay cabinet **first**, on the **first** announcement. Then add GitHub as a mirror. Push to both when you want CI to run.

That is the whole story. Everything below is detail.

---

## Three things that are easy to confuse

### 1. "Initialized" in Buzz ≠ "exists on the relay"

Buzz Projects can create a **local git workspace** on your machine — files on disk, remotes configured, everything looks healthy in the UI. That is *local* initialization.

Relay git storage is separate. The server only creates it when a repo is **announced with a relay clone URL from the start** (or via `buzz repos create` with `--clone` pointing at the relay). If that step never happened, `git push` to the relay URL returns **404 repository not found** even though NIP-98 authentication succeeded.

**Quick check:** `git ls-remote <relay-clone-url>` should list branches. If it 404s, the relay has no pointer for that repo — stop and re-announce relay-first (or use a new repo id).

### 2. A Buzz PR does not trigger GitHub Actions

| Action | What it is | Triggers GHA? |
|--------|-----------|---------------|
| `git push` to GitHub | Normal git push | Yes (`push` / `pull_request` events) |
| `git push` to relay | Buzz-native push | No — GitHub never sees it |
| `buzz pr open` | Signed Nostr event (kind:1618) | No — unless you mirror commits to GitHub |

Nick's workflow must include an explicit step to get commits onto GitHub (manual `git push origin`, a mirror script, or a future Buzz webhook → `repository_dispatch`).

### 3. Buzz Desktop cannot push to GitHub for you

From Buzz Projects, relay push uses NIP-98 automatically. GitHub push from Desktop has **no credentials attached** — public clone/fetch only. You push to GitHub from a normal terminal with `gh` or SSH keys.

---

## Why `buzz-tui` failed but this repo works

We debugged `another-level-of-indirection/buzz-tui` on branch `hybrid-monorepo`:

| Symptom | Cause |
|---------|-------|
| `git push` to relay → 404 | Relay object-store pointer never seeded for `ec2dd863…/buzz-tui` |
| Buzz PR shows diffs anyway | PR diffs can come from local state or GitHub — relay push is not required |
| Two announcements for `d=buzz-tui` | Ian's GitHub-only head + hybrid fork's dual-URL head — Projects may bind to the wrong one |

`gha-ci-test` was created **relay-first** on day one. `git ls-remote` works immediately. Push works. Same relay, same auth — different provisioning history.

**Do not try to retrofit relay git onto a GitHub-only announcement.** Start fresh with a new repo id or ask relay ops to seed the pointer.

---

## Architecture (hybrid pattern)

```
┌─────────────────┐     manual mirror      ┌─────────────────┐
│   Buzz relay    │ ──────────────────────►│     GitHub      │
│  (source of     │   git push origin      │  (build farm —  │
│   truth, PRs,   │                        │   GitHub Actions)│
│   audit trail)  │                        │                 │
└─────────────────┘                        └─────────────────┘
        ▲                                           │
        │ git push relay                            │ workflow runs
        │ (NIP-98, automatic)                       ▼
   Buzz Desktop                              pass/fail badge
```

**Today:** mirror is manual — after `git push relay`, run `git push origin`.

**Eventually:** Buzz channel workflow + `call_webhook` posts CI results back into the repo channel; `diff_posted` trigger can kick GitHub via `repository_dispatch`. That requires an owner/admin to create the webhook workflow in Desktop.

---

## Nick's checklist — confirm CI will work

Run these in order on a **new** test repo before betting a real project on the flow.

### Step 0 — Prerequisites

- Buzz Desktop logged in, member of the target project channel
- `buzz` CLI configured (`BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY`)
- `gh` CLI authenticated to GitHub
- git 2.46+ (for NIP-98 credential helper)

### Step 1 — Create the Buzz repo relay-first

```bash
RELAY="https://lotf.communities.buzz.xyz"
OWNER="<your-64-char-hex-pubkey>"   # from buzz users get, or your key
CHANNEL="<project-channel-uuid>"
REPO_ID="my-ci-test"                # pick a fresh id

buzz repos create \
  --id "$REPO_ID" \
  --name "My CI Test" \
  --channel "$CHANNEL" \
  --clone "${RELAY}/git/${OWNER}/${REPO_ID}"
```

**Pass:** command returns a `buzz://repo?...` link.

**Fail:** if you skip `--clone` with the relay URL, stop — you are repeating the buzz-tui mistake.

### Step 2 — Verify relay storage exists (before any local work)

```bash
git ls-remote "${RELAY}/git/${OWNER}/${REPO_ID}"
```

**Pass:** lists refs (may be empty on a brand-new announce — that is fine, not a 404).

**Fail:** `repository not found` → relay pointer missing. Do not proceed; fix the announcement or use a new `--id`.

### Step 3 — Push initial commit to relay

```bash
mkdir "$REPO_ID" && cd "$REPO_ID"
git init -b main
echo "# CI test" > README.md
git add README.md && git commit -m "init"
git remote add relay "${RELAY}/git/${OWNER}/${REPO_ID}"
git push -u relay main
```

**Pass:** push succeeds without 404.

**Fail:** 404 after step 2 passed → channel ACL issue; confirm you are a channel member and `--channel` was set on announce.

### Step 4 — Create GitHub mirror and push

```bash
gh repo create your-org/"$REPO_ID" --public --source=. --remote=origin --push
```

Or manually:

```bash
gh repo create your-org/"$REPO_ID" --public
git remote add origin "https://github.com/your-org/${REPO_ID}.git"
git push -u origin main
```

**Pass:** `git ls-remote origin` shows `refs/heads/main`.

### Step 5 — Add GitHub Actions and confirm green

Copy `.github/workflows/ci.yml` from this repo (or write any trivial smoke test). Push to GitHub:

```bash
git add .github/workflows/ci.yml
git commit -m "add CI"
git push origin main
```

**Pass:** workflow runs green at `https://github.com/your-org/<repo>/actions`.

**Fail:** workflow never triggers → you pushed only to relay; re-run `git push origin main`.

### Step 6 — Open a Buzz PR (relay side)

```bash
git checkout -b test-branch
echo "test" >> README.md && git add README.md
git commit -m "test PR"
git push relay test-branch

buzz pr open \
  --repo-owner "$OWNER" \
  --repo-id "$REPO_ID" \
  --subject "Test PR" \
  --commit "$(git rev-parse HEAD)" \
  --merge-base "$(git rev-parse main)" \
  --clone "${RELAY}/git/${OWNER}/${REPO_ID}" \
  --clone "https://github.com/your-org/${REPO_ID}.git" \
  --branch-name test-branch \
  --channel "$CHANNEL"
```

**Pass:** Buzz PR appears in Projects with diff.

**Note:** GHA will **not** run from this step alone. That is expected.

### Step 7 — Mirror to GitHub and confirm GHA on the PR branch

```bash
git push origin test-branch
```

Then open a GitHub PR (or let the `pull_request` workflow fire on the branch push).

**Pass:** Actions tab shows a run for the branch.

### Step 8 — Ongoing habit

Every time you push to relay from Buzz Projects:

```bash
git push origin <same-branch>
```

Until automated mirror wiring ships, this is the step that connects Buzz work to GitHub CI.

---

## What this repo contains

| Piece | Purpose |
|-------|---------|
| `.github/workflows/ci.yml` | Minimal smoke test (`test -f README.md`) on push/PR to `main` |
| `relay` remote | `https://lotf.communities.buzz.xyz/git/ec2dd863…/gha-ci-test` |
| `origin` remote | `https://github.com/another-level-of-indirection/gha-ci-test` |

## Links

| | |
|---|---|
| Buzz repo | buzz://repo?owner=ec2dd863d2cf968900bf479839ef793c6c329c7e6abbdec410c104bbee6b5e3b&d=gha-ci-test |
| GitHub | https://github.com/another-level-of-indirection/gha-ci-test |
| Actions | https://github.com/another-level-of-indirection/gha-ci-test/actions |

## Further reading

Workspace guide (agent-maintained): `GUIDES/BUZZ_GITHUB_ACTIONS_CI.md` in the Buzz nest.
