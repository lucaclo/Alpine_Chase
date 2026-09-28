# Issue Generation Prompt (Phase 4)

Fill in the blank and run this in Claude Code once `context/brainstorm.md` and `context/plan.md` exist.

---

I want to build ___. Refer to @context/brainstorm.md, @context/plan.md and all that you have planned.

Your job is to convert this into a complete, ordered set of GitHub issues using the GitHub CLI.
Do not write any code. Do not create any files. Only create GitHub issues.

Make sure this is an MVP first, with later issues building on it in iterations.

---

## STEP 1 — THINK BEFORE YOU ACT

Before creating a single issue:

1. Read all requirements fully
2. Think harder about the correct ordering and dependencies across the entire project
3. Identify every place where a broad feature should be split into multiple atomic issues
4. Write out your full proposed issue list as a numbered outline with one-line descriptions
5. Stop and wait for my approval of the outline before creating anything

---

## STEP 2 — ORDERING RULES

Issues must be created in this exact sequence:

**Issue #1: Set up the test suite**
- Unit and integration tests
- Testing framework fully configured and running locally

**Issue #2: Set up CI via GitHub Actions**
- Run the full test suite on every commit
- Run the linter on every commit
- Block merges to main if either fails
- The workflow job must be named `ci` (`jobs: ci:`) so branch protection can require it

**Issue #3: Set up Puppeteer MCP server for browser-based UI testing**
- Claude Code should be able to open a real browser and click through the app to test UI changes

All subsequent issues must be ordered strictly by dependency.
No issue should require code that hasn't already been merged in a prior issue.
When two issues have no dependency on each other, order the more foundational one first.

---

## STEP 3 — WHAT MAKES A GOOD ISSUE

**ATOMIC:** Each issue touches exactly one feature, one model, or one route.
- If an issue touches two models → split it into two issues.
- If an issue touches two routes → split it into two issues.
- If an issue would take more than one focused coding session → split it.

**SPECIFIC:** The title is a concrete action verb phrase.
- Good: "Add password reset flow via Devise mailer"
- Bad: "Auth stuff" or "User authentication"

**SELF-CONTAINED:** A developer with zero prior context on this project could read this issue alone and know exactly what to build, what files to touch, what the success criteria are, and what not to do.
This is critical — Claude Code will work on each issue from a completely cold start with no memory of previous issues.

---

## STEP 4 — REQUIRED ISSUE BODY FORMAT

Every issue must contain exactly these five sections:

```markdown
## Goal
One sentence. What does this issue accomplish and why does it matter to the app?

## Acceptance Criteria
- [ ] Written as "User can..." or "System does..." statements
- [ ] Each bullet is specific and independently verifiable
- [ ] Minimum 3 bullets, maximum 7
- [ ] If you need more than 7, the issue must be split

## Implementation Notes
- Which specific files, models, controllers, routes, or components are created or modified
- Which libraries, gems, or packages to use (be specific about versions if relevant)
- Which libraries or approaches to explicitly avoid and why
- Any edge cases or error states that must be handled
- Any architectural decisions that have already been made and must not be re-litigated

## Dependencies
List the issue numbers that must be merged before work on this issue can begin.
Write "None" if this is a foundational issue.

## Out of Scope
List 2-3 related things that might seem like they belong here but must NOT be
done in this issue. This prevents scope creep and keeps Claude focused.
```

---

## STEP 5 — CREATE THE ISSUES

Once I have approved your outline, create each issue using:

```
gh issue create --title "..." --body "..."
```

Format the body in markdown matching the five sections above.
Create them in order, one at a time.
After creating all issues, run `gh issue list` and show me the full list so I can confirm everything was created correctly.

---

## QUALITY CHECK

Before submitting your outline for my approval, verify every proposed issue against this checklist:

- [ ] Does this issue touch more than one model or route? If yes, split it.
- [ ] Could a developer with zero context understand exactly what to build? If no, add more detail.
- [ ] Does this issue depend on something that isn't in a prior issue? If yes, reorder.
- [ ] Does the title contain a specific action verb? If no, rewrite it.
- [ ] Are there more than 7 acceptance criteria? If yes, split the issue.
