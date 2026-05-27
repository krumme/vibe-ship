# vibe-ship

A [Claude Code](https://claude.com/claude-code) skill that turns "I want to ship this change" into a single command. Installs as `/ship` — so you can say "ship this" (or type `/ship`) and Claude runs the whole end-to-end workflow with **two unskippable approval gates**: an intent check before commit, and a merge check after the cold review only when the reviewer flagged anything. Clean reviews auto-merge; everything else surfaces the gate so you can decide.

```
write code → /ship → intent gate (you approve) → branch + commit + push + PR
        → cold AI review → (clean → auto-merge / feedback → merge gate) → cleanup → done
```

## Who this is for

- **Solo developers** who work on their own GitHub projects and don't have a teammate to review every PR.
- **Small teams** who want a lightweight discipline layer on top of their existing workflow without setting up branch protection, required reviewers, or CI gates.
- **Non-developers shipping code with Claude** who want to move fast but not skip the parts of code review that actually catch bugs.

It assumes your repo auto-deploys on push to `main` (Railway, Render, Vercel, Fly, etc.) and that you have `gh` installed.

## Why it's helpful

Vibe-coding is fast, but speed alone produces sloppy commits, accidental secrets, half-broken doc links, and "fix later" debt that piles up. `vibe-ship` keeps the speed and adds back **just enough** discipline:

- **One gate by default, two when it matters.** You confirm intent once, in plain English, before anything ships. The cold review either auto-merges on a clean APPROVE or surfaces a second gate when the reviewer flagged anything — no clicking through six prompts to land a one-line fix, but also no silently steamrolling over reviewer notes.
- **Both gates are structured, not free-form.** Each one is a Claude Code question prompt with explicit options, so the pause is a visible artifact in the session — it can't be silently skipped even when the agent is operating autonomously.
- **Cold AI review with no verdict-shopping.** A fresh Claude subagent reads the diff with zero session context. It doesn't know what you intended; it reads the code as a reviewer would. If it requests changes, the skill addresses them and pushes the fix — it does **not** re-dispatch the reviewer hoping for a different verdict. Reviewer hygiene by design.
- **Audit trail on every PR.** The cold reviewer's full verdict gets posted as a PR comment. Months later, you can see what was checked, what was flagged, and what was fixed.
- **Fast reverts when prod breaks.** If a smoke test fails after merge, the revert flow skips the cold review and ships immediately. Speed matters when prod is down.
- **One ask, not nagging.** First time you ship from a project, the skill checks for an "If main breaks" rollback section in `CLAUDE.md` and offers to add one. If you decline, it doesn't re-prompt.

## Trigger phrases

After Claude has just written code:

- `"ship this"` / `"ship it"` / `"deploy this"` / `"push it up"` / `"send it"`
- Or ask Claude to fix/build something and ship it
- Or type `/ship` explicitly

## Install

The skill lives at a fixed path on each machine: `~/.claude/skills/ship/SKILL.md`.

```bash
# 1. Clone this repo somewhere outside ~/.claude/skills
git clone https://github.com/krumme/vibe-ship.git ~/vibe-ship

# 2. Copy the skill into place
mkdir -p ~/.claude/skills/ship
cp ~/vibe-ship/ship/SKILL.md ~/.claude/skills/ship/SKILL.md
```

Verify:

```bash
ls ~/.claude/skills/ship/
# should show: SKILL.md
```

Restart Claude Code (or wait for the next conversation) and `/ship` is available.

### Stay on the latest version

If you'd rather always be on the latest, **symlink** instead of copy:

```bash
rm ~/.claude/skills/ship/SKILL.md
ln -s ~/vibe-ship/ship/SKILL.md ~/.claude/skills/ship/SKILL.md
```

Then `cd ~/vibe-ship && git pull` is all you need to update.

## Per-project setup

`vibe-ship` reads two optional sections from each project's `CLAUDE.md`:

### `## If main breaks` (recommended)

A short runbook for rolling back a bad commit. The skill prompts you to add one the first time it ships in a project that doesn't have one. Example for Railway:

```markdown
## If main breaks

Open Railway → Deployments → click the previous successful deploy → Redeploy.

Never commit directly to `main` to "fix it fast." A bad hotfix is more likely to make the outage worse than to fix it.
```

### `## Smoke test` (optional)

If you want `/ship` to remind you to verify the deploy after merge, add a section with the **prod** command:

```markdown
## Smoke test

curl https://your-service.up.your-host.example.com/health
```

If this section is absent, `/ship` prints `Shipped PR #N, you're back on main.` after the merge and stops. It will **not** fish for `smoke-test.sh` or similar — script existence doesn't imply a prod check, and skipping cleanly beats prescribing a useless step.

## Requirements

- [Claude Code](https://claude.com/claude-code) installed (CLI, desktop, or IDE extension)
- [`gh` CLI](https://cli.github.com/) installed and authenticated (`gh auth status` succeeds)
- Project is a git repo with a GitHub remote named `origin`
- You can push to the repo

## Credits

Built by [Kurt Krumme](https://github.com/krumme) and Claude (Anthropic).

## License

[MIT](LICENSE).
