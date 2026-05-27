---
name: ship
description: Ship a code change end-to-end on a GitHub repo. After writing code, runs branch → commit → push → PR → cold AI review → squash-merge → cleanup with two unskippable approval gates (intent check before commit; merge check after cold review whenever the verdict has any feedback). A clean APPROVE with zero findings is the only path that auto-merges. Optional smoke test if the project's CLAUDE.md has a `## Smoke test` section. Auto-invoke when the user says "ship this", "ship it", "deploy this", "push it up", "send it", or asks to fix/build something and ship it. Also available as `/ship`.
---

# /ship — end-to-end ship workflow

Designed for solo developers and small teams on GitHub repos that auto-deploy on push to `main` (Railway, Render, Vercel, Fly, etc.). Tuned for users who want a single command that runs the whole ship-it workflow — branch, commit, push, cold AI review, merge, cleanup — without losing review discipline along the way.

## Hard rules

- **Two gates, both UNSKIPPABLE, both via `AskUserQuestion`.**
  - **Step 4 (intent gate)** fires every single time, before any commit. It is NOT a clarifying question — do not skip it because the session is in autonomous mode, because the user said "no questions," because the user already approved a plan that contains this work, or because the change feels obvious.
  - **Step 7 (merge gate)** fires whenever the cold reviewer returns anything other than a clean APPROVE (verdict APPROVE + zero findings). Any [NOTE], any [BLOCKER], any COMMENT verdict → user gets to decide whether to merge, fix-then-merge, or abandon. A clean APPROVE is the only path that auto-merges.
  - Both gates use `AskUserQuestion` so the pauses are visible artifacts in the transcript. If you reach step 8 without the intent gate having fired in the current ship cycle, OR without the merge gate having fired when the verdict was non-clean, you have violated the protocol.
- **Cold review means cold**: the review subagent receives ONLY the PR URL and the review criteria below. Never share the prior conversation, the author's intent, alternatives considered, or any session context. The reviewer must read the diff fresh.
- **No re-dispatch on REQUEST CHANGES**: address the blockers, push the fix, proceed to merge. Re-dispatching invites verdict-shopping.
- **No cold review on reverts**: when prod is broken, speed matters.
- **One ask per session**: if you ask the user about the rollback procedure and they decline or answer, don't ask again this session. (The intent gate in step 4 is separate from this rule — it always fires.)

---

## Step 1 — Pre-flight

Run these checks. On any failure, print the missing piece in plain English and stop:

```bash
git rev-parse --is-inside-work-tree   # inside a git repo
git remote get-url origin             # has a GitHub remote
gh auth status                        # gh is authenticated
```

Verify there's something to ship:
- `git status --porcelain` shows uncommitted changes, OR
- on a non-main branch with `git log @{u}..HEAD` (or `git log origin/main..HEAD` if branch isn't pushed yet) showing unpushed commits

If on `main` with a clean tree and nothing unpushed: print "Nothing to ship." and stop.

## Step 2 — First-run check: "If main breaks" section

Look for a rollback runbook in the project's CLAUDE.md:

```bash
grep -i -E "if main breaks|^## rollback|^## revert|redeploy-previous|redeploy previous" CLAUDE.md
```

If no match and you haven't already asked this session, pause and ask the user:

> I don't see an "If main breaks" section in CLAUDE.md for this project. Can I add one? What's the rollback procedure here? (Railway example: "Open Railway → Deployments → click the previous successful deploy → Redeploy.")

On their answer, append to CLAUDE.md (it'll be staged as part of the same commit later):

```markdown
## If main breaks

{user's revert procedure, verbatim}

Never commit directly to `main` to "fix it fast." A bad hotfix is more likely to make the outage worse than to fix it.
```

If they decline, continue without modifying CLAUDE.md. Don't re-prompt this session.

## Step 3 — Branch setup

- **On `main`**:
  ```bash
  git checkout main && git pull
  git checkout -b <branch-name>
  ```
  Branch name derivation:
  - Fixes → `fix/short-description` (e.g. `fix/sync-lock-bug`)
  - New features → `add-short-description` (e.g. `add-help-endpoint`)
  - Doc-only → `docs/short-description`
  - Keep it under ~40 chars. Either style (with or without slash) is fine.
- **Already on a feature branch with the user's prior work**: stay on it.

## Step 4 — Intent check (THE approval gate)

This step is **mandatory and unskippable** — see the hard rule at the top of this skill. Even if the user has told you not to stop for clarifying questions, even if they pre-approved a plan that includes this work, you still run this gate. It is the only place you ever pause inside `/ship`, so honoring it is non-negotiable.

**Implementation: use `AskUserQuestion`, not "print and wait."** A structured tool call leaves a visible artifact in the transcript, which makes the gate auditable and harder to accidentally skip. Free-form "approve? (y/n)" prompts have been skipped in practice when session-level instructions tell the agent to work autonomously — the structured form prevents that drift.

**Before calling the tool**, print the intent paragraph to the terminal so the user has it in front of them when they answer:

```
What this does
==============
{1–3 sentences explaining what the change accomplishes and why.
Non-technical language. No file paths, no code snippets, no jargon.
This is what gets shown to a colleague who didn't write the change.}

Branch: {branch-name}
Files changing: {comma-separated list}
```

**Then call `AskUserQuestion`** with this exact shape:

- `question`: `"Approve this PR's intent paragraph?"`
- `header`: `"Ship gate"`
- `multiSelect`: `false`
- `options`:
  - `label`: `"Approve and proceed"`, `description`: `"Commit, push, open PR, run cold review."`
  - `label`: `"Push back with feedback"`, `description`: `"Adjust the code or the paragraph and re-ask."`

The user picks one of those, or uses the free-form "Other" field to write specific feedback.

Handle the response:

- **"Approve and proceed"** (and no contradictory notes) → proceed to step 5.
- **"Push back with feedback"**, or **"Other" with text**, or any answer that includes notes asking for changes → incorporate the feedback (adjust code, adjust the paragraph, or both), reprint the paragraph, and call `AskUserQuestion` again. Loop until approval is clean.

## Step 5 — Commit, push, open PR

Stage only the relevant files (never `git add -A` or `git add .`). The commit message focuses on *why*, derived from the approved intent paragraph:

```bash
git add <specific-files-changed>
git commit -m "$(cat <<'EOF'
<one-line subject (≤72 chars)>

<optional 1–2 sentence body explaining why>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
git push -u origin <branch>
```

Open the PR with the approved intent paragraph in the body:

```bash
gh pr create --title "<concise title, <70 chars>" --body "$(cat <<'EOF'
## What this does

<approved intent paragraph verbatim>

## Notes for review

- <CLAUDE.md updates, env var changes, breaking changes — anything reviewers need to know>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Capture the PR URL and PR number from the `gh pr create` output.

## Step 6 — Cold review

Dispatch a subagent of type `general-purpose` with EXACTLY this prompt (substitute `{PR_URL}` only — do NOT add session context, do NOT mention what the author intended, do NOT mention this is from a prior conversation):

```
You are reviewing a pull request. You have no context from any prior
conversation. Read the diff fresh.

PR URL: {PR_URL}

Steps:
1. `gh pr view {PR_URL}` and `gh pr diff {PR_URL}` to see the change
2. `gh pr list --state open` to spot overlap with other open work
3. If env vars are referenced in the diff, verify `.env.example` reflects them
4. If imports are added, verify `requirements.txt` or `package.json` is updated
5. If CLAUDE.md exists, check the change doesn't contradict it
6. If any `.md` file (e.g. `CLAUDE.md`, `README.md`) is in the diff, list every GitHub issue/PR reference (matching `#<number>`, `issues/<number>`, or `pull/<number>`) in the **post-PR state** of that file. For each, substitute the actual number and run `gh issue view <number>` (or `gh pr view <number>`). Flag any pointing at closed state. Stale references in a doc the author is already touching are well within scope.
7. If the PR description includes `Closes #<number>` or `Fixes #<number>` (the GitHub closing keywords), substitute the actual number and run `grep -rn "#<number>\|issues/<number>\|pull/<number>" --include="*.md" .` in the repo. Flag any other markdown file still referencing that number as future / open work — those references will be orphaned the moment this PR merges.

Flag (blockers):
- Secrets or credentials in the diff
- Env var added/renamed without updating `.env.example`
- New dependency not in the manifest
- CLAUDE.md describes behaviour the code no longer matches
- Breaking change to an HTTP endpoint or data contract
- Logging that could capture personal info or message content
- Conflict with another open PR
- Obviously broken code (syntax, undefined references)
- Markdown file in the diff references a GitHub issue or PR that is currently closed
- PR description closes an issue (Closes / Fixes #N) but other markdown files in the repo still reference it as future or open work

Ignore:
- Code style and naming preferences
- Missing tests
- Architectural opinions
- Over-engineering takes

Output format:
VERDICT: APPROVE | REQUEST CHANGES | COMMENT

Findings (omit if none):
- [BLOCKER] one line, specific, actionable
- [NOTE] one line

Do not restate what the code does. Keep total response under 200 words.
```

When the subagent returns, post the full verdict block to the PR for the audit trail:

```bash
gh pr comment <PR-number> --body "$(cat <<'EOF'
{verdict block verbatim}
EOF
)"
```

Also print the verdict to the terminal so the user sees it immediately.

## Step 7 — Verdict handling

The rule: **clean APPROVE → auto-merge. Any feedback at all → second gate.**

A "clean APPROVE" means verdict is APPROVE *and* the Findings block is empty (no [BLOCKER] and no [NOTE] items). Anything else — APPROVE with notes, any REQUEST CHANGES, any COMMENT — triggers the second `AskUserQuestion` gate. This gate is structurally identical to step 4: a visible artifact in the transcript so the agent can't silently steamroll past reviewer feedback.

**Clean APPROVE (verdict APPROVE + zero findings)** → proceed to step 8 automatically. No second gate, the cold review was a clean win.

**All other verdicts** → run the second gate.

### Second gate (only fires on non-clean verdicts)

First, print a paragraph to the terminal that (a) relays what the cold reviewer said and (b) explains what you plan to do about it. Plain language, no jargon. Example shape:

```
Cold review came back with {verdict}. {One sentence summary of the blockers/notes.}

My plan: {what you'd do if approved — e.g. "merge as-is", "address the
typo flag then merge", "fix the two blockers, push, then merge".}
```

Then call `AskUserQuestion`. The exact options depend on the verdict:

- **APPROVE + notes:**
  - `"Merge as-is"` — "Notes are advisory; ship the PR."
  - `"Address notes then merge"` — "Fix the notes, push, then merge."
- **REQUEST CHANGES** (always has blockers):
  - `"Address blockers and merge"` — "Fix the blockers, push, then merge."
  - `"Push back on verdict"` — "Disagree with one or more blockers; merge as-is."
- **COMMENT** (ambiguous):
  - `"Merge"` — "Treat as APPROVE, ship the PR."
  - `"Address feedback then merge"` — "Fix what the reviewer raised, push, then merge."
  - `"Abandon"` — "Close the PR without merging."

Use `header: "Merge gate"` for all three. `multiSelect: false`. The user can also pick "Other" to write specific instructions.

Then act on the answer:

- **Merge as-is / Merge / Push back on verdict** → proceed to step 8.
- **Address ... then merge** → fix the issues in code, commit with a message like `Address review: <one-line>` (same `Co-Authored-By` trailer), push to the same branch, then proceed to step 8. **Do NOT re-dispatch the cold reviewer** — the original verdict comment plus the fix commit IS the audit trail. Re-dispatching invites verdict-shopping.
- **Abandon** → stop. Leave the PR open for the user to close manually or pick up later. Do not delete the branch.
- **Other with custom instructions** → follow them. If they describe fixing then merging, address the issues and proceed to step 8.

## Step 8 — Merge, cleanup, done (or hand off to smoke test)

```bash
gh pr merge <PR-number> --squash --delete-branch
git checkout main && git pull
git branch -D <branch>      # capital D — squash-merged branches aren't recognized as merged by lowercase -d
git remote prune origin     # optional, tidy
```

If `git branch -D` errors with `error: branch '<name>' not found`, that's expected — `gh` already cleaned it up. Move on.

**Smoke test is opt-in per project.** It only runs if the project's `CLAUDE.md` has an explicit `## Smoke test` section that names the command:

```bash
grep -A 20 -i "^## smoke test" CLAUDE.md
```

- **Section exists** → extract the command, go to Step 9.
- **Section does not exist** → print and stop:
  ```
  Shipped PR #<N>, you're back on main.
  ```

Do NOT fish for script names (`test-followup.sh`, `smoke-test.sh`, etc.) — script existence does not imply a prod smoke test. Do NOT prompt the user to add a smoke test. Do NOT mention that smoke testing isn't configured. If the user wants smoke testing wired into `/ship` for a project, they ask for it explicitly or add the section themselves.

## Step 9 — Smoke test hand-off and wait

Only runs if Step 8 found a `## Smoke test` section. Print:

```
Railway is deploying PR #<N>. Wait 60–90 seconds, then run:

  {smoke-test-command}

Report back when it's done.
```

The user runs the smoke test outside the session and reports back.

**Pass** → print:
```
Shipped PR #<N>, smoke test passed, you're back on main.
```
Done.

**Fail** → print the failure summary the user gave, then ask: **"Revert? (y/n)"**

On `y` (revert):
```bash
# Create a revert commit on a new branch
git checkout -b revert-pr-<N>
git revert --no-edit <merge-commit-sha>      # use the squash-merge SHA from main
git push -u origin revert-pr-<N>

# Open and immediately merge the revert PR — no cold review, speed matters
gh pr create --title "Revert PR #<N>" --body "Smoke test failed against prod after PR #<N>. Reverting to redeploy previous version. No cold review per ship-skill protocol — speed matters when prod is broken."
gh pr merge <revert-PR-number> --squash --delete-branch

# Local cleanup
git checkout main && git pull
git branch -D revert-pr-<N> 2>/dev/null || true
```

Print:
```
Reverted PR #<N>, previous version redeploying now.
```

On `n` (don't revert): stop. Leave the user to debug. Don't auto-revert.

---

## What to do when this skill is invoked but Claude hasn't written code yet

If the user invokes `/ship` (or says a trigger phrase) and there's nothing to ship and Claude hasn't just written code in the session, ask the user what they wanted to ship before running pre-flight. Don't fabricate a change.

## What this skill does NOT do

- It does not test code before commit (no unit tests, no linters). The cold reviewer catches obvious-broken code; the smoke test (when configured in CLAUDE.md) catches deploy regressions.
- It does not write tests for the change. If the user wants tests, they ask separately.
- It does not handle stacked PRs. If you're on a branch that depends on another open PR, surface that to the user and let them decide.
- It does not run on repos without a GitHub remote. Local-only repos can't open PRs.
