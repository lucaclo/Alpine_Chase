---
description: Plan, build, test and open a PR for a GitHub issue
argument-hint: <issue-number>
---

Please analyze and fix GitHub issue #$ARGUMENTS. Follow the project workflow in `context/workflow.md`.

# PLAN

1. Use `gh issue view $ARGUMENTS` to get the issue details
2. Understand the problem described in the issue
3. Ask clarifying questions if necessary
4. Understand the prior art for this issue
   - Search the scratchpads (`scratchpads/`) for previous thoughts related to the issue
   - Search PRs to see if you can find history on this issue (`gh pr list --state all --search ...`)
   - Search the codebase for relevant files
   - Read `context/plan.md` for the agreed stack and architecture — do not re-litigate decisions made there
5. Think harder about how to break the issue down into a series of small, manageable tasks.
6. Document your plan in a new scratchpad
   - Path: `scratchpads/issue-<number>-<short-issue-name>.md`
   - Include a link to the issue in the scratchpad

# CREATE

- Create a new branch for the issue (e.g. `issue-<number>-<short-name>`), branched from an up-to-date `main`
- Solve the issue in small, manageable steps, according to your plan
- Commit your changes after each step using a descriptive commit message that references the issue number (e.g. "Add user login route #12")
- After every commit, immediately push to the remote branch with:
  `git push origin <branch-name>`
  This ensures work is never lost if the session ends unexpectedly.
- Do not batch multiple steps into a single commit — one logical change = one commit
- Stay within the issue's "Out of Scope" boundaries

# TEST

- Use Puppeteer via MCP to test the changes if you have made changes to the UI
- Write tests (using the project's test framework) that describe the expected behavior of your code
- Run the full test suite to ensure you haven't broken anything
- Run the linter
- If the tests are failing, fix them
- Ensure that all tests are passing before moving on to the next step
- Tick off each Acceptance Criterion from the issue and confirm it is met

# DEPLOY

- Open a PR with `gh pr create`, including `Closes #$ARGUMENTS` in the body, and request a review
- Update `context/progress.md` with the current issue status
- Tell me the PR link. CI (GitHub Actions: tests + linter) must pass before merge. After I merge, I will run `/clear` and start the next issue.
