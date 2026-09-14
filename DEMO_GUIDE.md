# Git Hooks Demo — Presenter Guide
> Read this top-to-bottom once to LEARN, then use Part 2 as your live script.
> Total demo time: ~20-25 minutes. Audience: beginners.

---

## PART 0 — Learn in 5 minutes (for you, the presenter)

### 1. What is a Git hook?
A **script that Git runs automatically** when something happens: before/after commit, before push, etc.

You use them to **enforce rules automatically** instead of reminding people:
- "Don't commit broken code" → `pre-commit` checks it
- "Write good commit messages" → `commit-msg` checks it
- "Don't push WIP / broken tests" → `pre-push` checks it
- "Log every commit somewhere" → `post-commit` does it

### 2. The only 4 things you must know

| # | Fact | What to say on stage |
|---|------|----------------------|
| 1 | Hooks live in `.git/hooks/` and are **local only** (not pushed to GitHub). | "That's why this project keeps versioned copies in `.githooks/` + a `setup.sh` installer — the standard workaround." |
| 2 | Hooks are just **executable files** named after events (`pre-commit`, `commit-msg`, ...). Exit code matters: `0` = allow, non-zero = block. | "If a hook exits 1, Git stops. That's the whole magic." |
| 3 | **Client hooks** (this demo) run on your laptop. **Server hooks** (`pre-receive`, `update`) run on GitHub/GitLab — you can't demo those locally, just mention them. | "Today is 100% client-side, no server needed." |
| 4 | Any hook can be bypassed with `--no-verify` (commit/push). That's intentional — for emergencies — and a good Q&A point. | "Hooks guide, they don't jail you." |

### 3. The 4 hooks in this project

```
git add  ──▶  pre-commit (BLOCKING: whitespace, TODO, big files, syntax)
               ──▶  commit-msg (BLOCKING: must look like "feat: ...")
                    ──▶  commit created ──▶ post-commit (INFO ONLY: can't block)
                                                  ...
git push ──▶  pre-push (BLOCKING: no direct push to main, no WIP, tests must pass)
```

Open each hook file before the demo so you can explain one line if asked:
- `.githooks/pre-commit` — uses `git diff --cached` (staged files only!)
- `.githooks/commit-msg` — reads `$1` (file with the message), regex check
- `.githooks/post-commit` — uses `git rev-parse`, `git log -1`, appends to `.commit_log.txt`
- `.githooks/pre-push` — uses `git rev-parse --abbrev-ref HEAD` (branch name) + runs `python3 -m unittest discover -s tests`

### 4. Setup (do this BEFORE the audience arrives)

```bash
cd git-hooks-demo2
./setup.sh
# expected: ✅ Installed pre-commit, commit-msg, post-commit, pre-push

git log --oneline
# expected: 3 commits (feat: initial..., chore: fix gitignore..., fix: improve...)

python3 -m unittest discover -s tests -v
# expected: 4 tests OK
```

If anything is broken: `ls -l .git/hooks/ | grep -v sample` — all 4 hooks must be `-rwxr-xr-x` (executable). If not, re-run `./setup.sh`.

---

## PART 1 — Live Demo Script (copy-paste commands)

> Keep two terminals open: one for commands, one with `cat .githooks/<hook>` to show code when asked.
> Narrator lines are in **bold**.

### ACT 0 — Hook tour (2 min)

**"Git hooks are scripts Git runs for you. Let me show where they live:"**

```bash
ls .git/hooks/ | grep -v sample
ls .githooks/
cat setup.sh
```

Say: *"`.git/hooks/` is local-only and not committed, so teams keep versioned hooks in `.githooks/` and install with `setup.sh`. That's industry practice."*

Show the lifecycle diagram (from Part 0, section 3) on a slide or whiteboard.

---

### ACT 1 — `pre-commit`: catch bad code BEFORE it enters history (5 min)

**"Pre-commit runs when you type `git commit` but BEFORE the commit exists. Exit 1 = blocked."**

**Demo 1a — block trailing whitespace + TODO (the 'wow' moment):**

```bash
# create a bad file on purpose
printf 'x = 1   \n# TODO: fix this later\n' > demo_fail.py
cat demo_fail.py
git add demo_fail.py
git commit -m "feat: add failing demo"
```

Expected output (point at each line):
```
🔍 [pre-commit] Running checks...
  ❌ BLOCKED: Found trailing whitespace...
  ❌ BLOCKED: Found TODO/FIXME...
⛔ [pre-commit] Commit BLOCKED.
```

Say: *"Two bugs caught in 1 second — without human review. The whitespace check uses `git diff --cached --check`, the TODO check greps staged `.py` files."*

**Fix it live:**

```bash
printf 'x = 1\nprint(x)\n' > demo_fail.py
git add demo_fail.py
git commit -m "feat: add demo file"
# ✅ passes pre-commit + commit-msg, then post-commit prints a celebration
git reset --hard HEAD~1  # clean up so repo stays tidy (explain this removes the demo commit)
rm demo_fail.py
```

> If someone asks "what about unstaged files?": answer — *"`git diff --cached` means only staged files are checked. Unstaged changes are ignored — that's why you must `git add` first."*

**Demo 1b (optional, 30 sec) — block syntax errors:**
```bash
echo "def broken(:" > broken.py
git add broken.py
git commit -m "feat: add broken file"
# ❌ BLOCKED: Syntax error
git reset HEAD broken.py; rm broken.py
```

---

### ACT 2 — `commit-msg`: enforce message style (4 min)

**"Commit-msg receives the message as a file ($1) and validates it. We enforce Conventional Commits."**

```bash
echo "hello" > hello.txt
git add hello.txt

git commit -m "added stuff"
# ⛔ BLOCKED + prints required format + examples

git commit -m "fix bug"
# ⛔ BLOCKED (no colon, too short)

git commit -m "docs: add hello file"
# ✅ passes → post-commit fires
```

Say: *"Why bother? `git log --oneline` becomes readable, changelogs can be auto-generated, CI can decide version bumps from `feat:` vs `fix:`."*

Clean up: `git reset --hard HEAD~1; rm hello.txt` (or keep it — your choice, just say what you're doing).

Show the regex if asked:
```bash
cat .githooks/commit-msg
# PATTERN="^(feat|fix|docs|test|chore|refactor): .{5,}$"
```

---

### ACT 3 — `post-commit`: automation AFTER success (3 min)

**"Post-commit cannot block — the commit already exists. Use it for notifications, logging, stats."**

```bash
git log --oneline -3
cat .commit_log.txt
```

Point out: every successful commit from Acts 1-2 appended a line like:
```
2026-09-14 05:57:34 | 5dde412 | Demo User | feat: initial calculator demo
```

Show the code:
```bash
cat .githooks/post-commit
```

Say: *"Real-world uses: send a Slack message, trigger a local build, update a counter. Ours just logs — safe and visible."*

> Gotcha to mention: `post-commit` runs even with `--no-verify`? No — `--no-verify` skips pre-commit/commit-msg but post-commit still runs. Good Q&A trap.

---

### ACT 4 — `pre-push`: last gate before sharing code (5 min)

**"Pre-push runs on `git push`. It's your last chance — ideal for slow checks you don't want on every commit (full tests)."**

**Demo 4a — block direct push to main:**

```bash
git branch --show-current
# main
.git/hooks/pre-push
# ❌ BLOCKED: Pushing directly to 'main' is not allowed
```

Say: *"We run the hook directly because there's no real remote — same script Git would call on push. In a real repo, `git push` triggers it."*

**Demo 4b — pass on a feature branch + run tests:**

```bash
git checkout -b feat/add-multiply
.git/hooks/pre-push
# ✅ passes: Not on main, No WIP, All 4 tests passed
git checkout main
git branch -D feat/add-multiply
```

**Demo 4c — block WIP (intermediate, memorable):**

```bash
git checkout -b feat/wip-demo
echo "# temp" >> app.py
git add app.py
git commit --no-verify -m "wip stuff"
.git/hooks/pre-push
# ❌ BLOCKED: Found WIP in recent commits
git reset --hard HEAD~1
git checkout main
git branch -D feat/wip-demo
```

Say: *"Three layers: branch policy + hygiene (no WIP) + correctness (tests). That's the intermediate pattern companies actually use."*

**Demo 4d (optional, if time) — block on failing tests:**
```bash
git checkout -b feat/break-tests
# break one test temporarily:
sed -i 's/return a + b/return a + b + 999/' app.py
.git/hooks/pre-push
# ❌ BLOCKED: Tests failed
git checkout -- app.py
git checkout main
git branch -D feat/break-tests
```

---

### ACT 5 — Bypass + wrap-up (2 min)

```bash
git commit --help | grep -A2 no-verify
```

Say: *"Every blocking hook can be skipped with `--no-verify`. Show it once, then say: 'with great power...' — emergencies only, and it leaves no trace, so teams pair it with server-side checks."*

**Closing slide / whiteboard:**

| Hook | When | Can block? | Demo rule |
|------|------|-----------|-----------|
| pre-commit | `git commit`, before | ✅ | no whitespace/TODO/big/syntax-error |
| commit-msg | `git commit`, message check | ✅ | must be `type: description` |
| post-commit | after commit created | ❌ | log + summary |
| pre-push | `git push`, before | ✅ | no main, no WIP, tests pass |

**Key takeaways (say these verbatim):**
1. "Hooks are local scripts — exit 0 allows, non-zero blocks."
2. "Version them in `.githooks/` + installer, because `.git/hooks/` isn't committed."
3. "Keep pre-commit fast, push slow checks to pre-push."
4. "Client hooks guide; server hooks enforce — use both in production."

---

## PART 2 — Cheat sheet (keep open during Q&A)

```bash
./setup.sh                                  # install hooks
ls -l .git/hooks/ | grep -v sample          # verify
cat .githooks/pre-commit                    # show code
git diff --cached --check                   # manual whitespace check
git commit --no-verify -m "msg"             # bypass pre-commit/commit-msg
git push --no-verify                        # bypass pre-push
chmod +x .git/hooks/pre-commit              # fix "permission denied"
git log --oneline -5                        # readable history
cat .commit_log.txt                         # post-commit log
```

**If a demo step fails unexpectedly:**
- `git status --short` — is the right file staged?
- `git log --oneline -3` — did post-commit actually run?
- Re-run `./setup.sh` — did you edit `.githooks/` but forget to reinstall? (`.git/hooks/` is the live copy!)
- Windows audience? Hooks need Git Bash + `chmod +x`. Mention it.

**Likely questions + one-line answers:**
- "Do hooks sync via push?" → No. Share via `.githooks/` + docs, or tools like pre-commit.com / Husky.
- "Can I use Python/Node instead of bash?" → Yes — any executable; shebang decides (`#!/usr/bin/env python3`).
- "Server hooks?" → `pre-receive`/`update` on GitHub Enterprise, or Branch Protection Rules on github.com (the hosted equivalent).
- "Slow tests in pre-commit?" → Don't. Keep pre-commit <5s; slow suite goes in pre-push or CI.
- "How to debug?" → `echo` everywhere + run `.git/hooks/<name>` manually.

---

## PART 3 — Practice checklist (do tonight, 15 min)

- [ ] Run `./setup.sh` from scratch in a fresh clone
- [ ] Trigger each BLOCK once (whitespace, bad message, main push, WIP) without looking at notes
- [ ] Explain `git diff --cached` vs `git diff` in one sentence
- [ ] Explain `$1` in commit-msg in one sentence
- [ ] Say why post-commit can't block in one sentence
- [ ] Break a test and watch pre-push catch it, then restore

You're ready when you can do all six without notes. Good luck!
