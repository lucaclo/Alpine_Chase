# Project Workflow (Robbie's Workflow)

This is the workflow for the **entire** Alpine_Chase project. Every phase of work must follow it.
Check which phase we're in before starting anything (see `context/progress.md`).

---

## Phase 0 — Repo setup

1. Create a GitHub repo.
2. Protect `main` so CI must pass before merging (only possible on a **public** repo, or a paid plan for private repos):
   1. Repo on GitHub → **Settings**
   2. Left sidebar → **Branches**
   3. **Add branch protection rule**
   4. Branch name pattern: `main`
   5. Check **Require status checks to pass before merging**
   6. In the search box, type `ci` — the job name in `.github/workflows/ci.yml` (`jobs: ci:`)
   7. Select it → **Save changes**

   Note: the `ci` check only appears in the search box after the CI workflow has run at least once (i.e. after Issue #2 is merged). Come back and finish this step then.

## Phase 1 — Brainstorm (no code)

1. Brainstorm with Claude. Claude asks as many questions as possible to nail down the specifics.
2. Claude generates a prompt outlining the full project spec (include everything) → save to `context/brainstorm.md`.

## Phase 2 — Plan (Claude Code, **plan mode is critical**)

Feed `context/brainstorm.md` into Claude Code in plan mode and ask it to "Plan…" with "Think hard". Outline:

- **Tech stack** — language, frontend, database, auth/login, emails, cloud/VMs, object storage, AI models
- **Technical architecture**
- **Data flow** (delegate most of this work)
- **API**

Save the approved plan to `context/plan.md`.

## Phase 3 — CLAUDE.md

Run `/init` to generate/extend `CLAUDE.md`. **Do not bloat it.** Keep the pointer to this `context/` folder.

## Phase 4 — GitHub issues

Use the prompt in `context/issue-generation-prompt.md` to turn the plan into detailed, atomic GitHub issues.

- First issues are always: (1) set up test suite, (2) set up CI via GitHub Actions, (3) set up Puppeteer MCP for browser UI testing.
- Build an **MVP first**, then iterate on top of it.

## Phase 5 — Build loop (per issue)

1. Run `/process-issue <number>` (defined in `.claude/commands/process-issue.md`) — plan → create → test → open PR.
2. CI (GitHub Actions) runs the full test suite + linter; merge to `main` is blocked if either fails.
3. Approve and merge the PR.
4. Run `/clear`, then start the next issue.
5. Rinse and repeat.

## Phase 6 — New bugs / features

Brainstorm them, then add them as new GitHub issues (same five-section format as Phase 4). Never fix ad hoc outside of an issue.

---

## Standing rules (apply at all times)

- One issue = one branch = one PR.
- One logical change = one commit; reference the issue number (e.g. `Add user login route #12`).
- Push to the remote branch after **every** commit.
- All tests must pass locally before opening a PR.
- Use Puppeteer via MCP to test any UI change.
- Plans for each issue live in `scratchpads/`.
