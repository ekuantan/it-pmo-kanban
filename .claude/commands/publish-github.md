---
description: Security-scan, push to GitHub, write README + About, enable GitHub Pages and set up CI/CD
argument-hint: <owner/repo or repo URL> [public|private] [branch]
---

Publish this project to GitHub end to end. Repo info from the user: **$ARGUMENTS**

Parse the arguments as: `owner/repo` (or an `https://github.com/owner/repo(.git)` URL), optional visibility (default `public`, since free-plan GitHub Pages needs a public repo), optional branch (default `main`). If no repo was given, check `git remote get-url origin`; if there is none either, ask the user for the repo before doing anything.

Work through the steps below **in order** and keep a task list. Use `gh` for all GitHub operations. Read `CLAUDE.md` first and respect its hard constraints (single `index.html`, no build step, no persistence, only the FormSubmit network call).

## 0. Preflight

- `gh auth status` must succeed. If not, stop and tell the user to run `gh auth login` themselves (never handle tokens or passwords).
- `git status --short`, `git branch --show-current`, `git remote -v`, `git log --oneline -5`.
- Check whether the repo exists: `gh repo view <owner/repo> --json name,visibility,description,homepageUrl,defaultBranchRef`.

## 1. Security scan (gate: nothing leaves the machine until this passes)

The user listed this later, but it runs **before** the first push. Scan every file that would be pushed (`git ls-files` plus untracked files that would be added), and the git history (`git log -p --all`).

Look for:
- Secrets: API keys, tokens (`ghp_`, `github_pat_`, `sk-`, `AKIA`, `xox[bp]-`, `AIza`), private keys (`-----BEGIN .* PRIVATE KEY-----`), passwords, connection strings, `.env` files, `*.pem`, `*.key`, `*.p12`, credentials JSON.
- Personal data: email addresses, phone numbers, real names, internal hostnames or IPs, absolute local paths (e.g. `C:\Users\...`).
- Network calls: every `https?://`, `fetch(`, `XMLHttpRequest`, `navigator.sendBeacon`, `WebSocket`, `<script src`, `<link href`, `@import`, `<img src`. The **only** allowed outbound call is `FORMSUBMIT_ENDPOINT`. Report the email address it contains and ask the user whether it is OK for that address to be public.
- Storage APIs that CLAUDE.md forbids: `localStorage|sessionStorage|indexedDB|document\.cookie`.
- Real bank logos, trademarks or anything imitating an official system (CLAUDE.md says the bank is fictitious).
- Stray files that should not ship: untracked folders (e.g. `artifact/`), editor/OS junk, large binaries, `.claude/settings.local.json`.

Use the Grep tool for these patterns, plus `gitleaks detect --source . -v` if `gitleaks` is installed (don't install it unprompted).

Then:
- Create or update `.gitignore` to cover `.env*`, `*.pem`, `*.key`, `.DS_Store`, `Thumbs.db`, `.claude/settings.local.json`, and any stray untracked folders the user does not want published.
- Print a findings table (file:line, what, severity, proposed action).
- If anything sensitive is found: **stop and ask** before fixing. If it is in git history, explain that it needs a history rewrite and that a leaked secret must be rotated; do not rewrite history or force-push without explicit approval.
- Never stage with `git add -A` or `git add .` blindly. Add files by name after the scan.

## 2. README

Create or update `README.md` (keep any existing content the user wrote). Include:
- Title and one-paragraph summary, stating clearly that it is a demo/training board for a **fictitious** bank.
- Live demo link: `https://<owner>.github.io/<repo>/` (case: lowercase owner).
- Features (drawn from `index.html` and CLAUDE.md: columns, filters, drag-and-drop plus keyboard move, add-task form with validation, email notification via FormSubmit, no persistence).
- How to run locally (double-click `index.html`, or `python -m http.server 8000`).
- How the FormSubmit email notification works, including the one-time activation email and where to change `FORMSUBMIT_ENDPOINT`.
- Tech constraints (vanilla, single file, no external resources, no storage).
- CI/CD and deployment section describing the workflows from step 6.
- A CI status badge: `![CI](https://github.com/<owner>/<repo>/actions/workflows/ci.yml/badge.svg)`.

No placeholder text left behind, no invented facts.

## 3. Push the code

- Commit the README, `.gitignore`, and workflow files (step 6 files can be written now so they ship in the same push) with a clear message ending in the Co-Authored-By line from the session's attribution reminder.
- Show the user what will be pushed (`git status`, `git diff --stat origin/<branch>..HEAD` or the file list for a new repo) and the target repo + visibility, then push.
- If the repo doesn't exist: `gh repo create <owner/repo> --<visibility> --source . --remote origin --push`.
- If it exists: set/verify `origin`, then `git push -u origin <branch>`. If the push is rejected, stop and report; never `--force` unless the user explicitly says so.

## 4. Repo "About" section

```bash
gh repo edit <owner/repo> --description "<one-line summary>" --add-topic kanban --add-topic project-management --add-topic vanilla-js --add-topic github-pages --add-topic demo
```

Keep the description under ~120 characters and say it is a demo for a fictitious bank. Leave the homepage for step 7, once Pages is live.

## 5. GitHub Pages

Configure Pages to deploy from GitHub Actions:

```bash
gh api -X POST repos/<owner>/<repo>/pages -f build_type=workflow
```

If it already exists (HTTP 409), switch it instead with `gh api -X PUT repos/<owner>/<repo>/pages -f build_type=workflow`. Then read the URL with `gh api repos/<owner>/<repo>/pages --jq .html_url`. If Pages is refused because the repo is private on a free plan, tell the user and ask whether to make it public; don't change visibility on your own.

## 6. CI/CD with GitHub Actions

Keep `.github/workflows/pages.yml` (deploy) and add `.github/workflows/ci.yml` that runs on `push` and `pull_request` to the branch, with `permissions: contents: read`, and checks:

1. **Constraint check**: fails if `index.html` matches `localStorage|sessionStorage|indexedDB|document\.cookie|!important|alert\(|confirm\(`, or if any `https?://` URL appears other than on the `FORMSUBMIT_ENDPOINT` line.
2. **No external resources**: fails on `<script src=`, `<link ... href="http`, `@import`, or `<img src="http` in `index.html`.
3. **Secret scan**: `gitleaks/gitleaks-action@v2` with `fetch-depth: 0` on checkout (needs `GITHUB_TOKEN` env only; no licence needed for personal accounts).
4. **HTML sanity**: confirm `index.html` exists, starts with `<!DOCTYPE html>`, and has matching `<script>`/`</script>` and `<style>`/`</style>` counts. No npm, no build step.

Make deployment depend on CI: either merge the CI job into `pages.yml` as a job the `deploy` job `needs:`, or trigger `pages.yml` with `on: workflow_run: workflows: [CI], types: [completed]` plus `if: github.event.workflow_run.conclusion == 'success'`. Pick one, keep `workflow_dispatch`, and keep `pages.yml` publishing only `index.html` (+ `.nojekyll`).

Run the constraint checks locally first so the first CI run passes. If these files changed after the push in step 3, commit and push again.

## 7. Link Pages in the About section and verify

```bash
gh repo edit <owner/repo> --homepage "https://<owner>.github.io/<repo>/"
```

Then watch the runs to completion: `gh run list --limit 5` and `gh run watch <id> --exit-status` (run in the background if long). If a run fails, read `gh run view <id> --log-failed`, fix, push, and re-check. Finally open the live URL in the built-in browser and confirm the board loads.

## Final report

Summarise: repo URL, visibility, Pages URL, About description/topics/homepage, workflows and their latest run status, security-scan findings and what was done about each, and anything left for the user (e.g. clicking the FormSubmit activation email, rotating a leaked secret).
