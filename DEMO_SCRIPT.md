# Git demo script

A runbook for walking a git beginner through the basics, live. You type
these; narrate what each command does before you run it. Talking points
are in *italics*.

This repo already has hooks wired up via the
[pre-commit framework](https://pre-commit.com/) (see [HOOKS.md](HOOKS.md)) —
your mate will feel them firing as you go. Don't explain them properly
until step 6; the payoff is better once they've been bitten by one.

## 0. Before they sit down

```sh
cd /Users/jay/Git/Me/git-demo
git log --oneline              # nothing — this repo is a blank slate
pre-commit install --install-hooks   # should already be done; confirms hooks are live
```

*The hook environments are already downloaded and cached, so nothing will
stall mid-demo.*

## 1. Identity & concepts

```sh
git config --get user.name
git config --get user.email
```

*A commit records who made it — that comes from this config.*

*Three things to keep straight: the **working tree** (files on disk right
now), the **staging area** (what you've told git to include in the next
commit), and the **history** (snapshots you've saved). Git makes you
choose what goes in a commit — that's the staging area's whole job.*

*One heads-up: this repo has some automated checks wired into git itself.
You'll see them fire. We'll dig into them at the end.*

## 2. First commit

```sh
git status
```

*Everything's untracked — git doesn't know about any of it yet.*

Let the commit-message hook bite (this is the hook teaser):

```sh
git add .
git commit -m "first commit"
```

*→ **blocked.** Something checked the commit message and didn't like it.
Park that thought. It wants a `type: description` format:*

```sh
git commit -m "feat: initial demo files"
git log
```

*Note what scrolled past — a list of checks, all passing. That's the hooks.*

## 3. Change → diff → commit

```sh
echo "- a second line" >> notes.md
git status
git diff
git add notes.md
git diff              # empty now — it's staged
git diff --cached     # there it is
git commit -m "docs: add a second line to notes"
git log --oneline
```

*`git diff` shows unstaged changes; `git diff --cached` shows staged ones.
That's the clearest way to see the staging area is a real, separate place.*

### A hook that fixes your work instead of just complaining

```sh
printf 'a line with trailing spaces   \n' >> notes.md
git add notes.md
git commit -m "docs: add another line"
```

*→ **blocked**, but look: `files were modified by this hook`. It didn't
just refuse, it stripped the trailing whitespace for you. The fix is
sitting in your working tree — you just have to stage it and go again:*

```sh
git diff              # the hook's own fix, unstaged
git add notes.md
git commit -m "docs: add another line"
```

## 4. Branching & merging

```sh
git switch -c feature/greeting
echo "Hello from the feature branch" >> notes.md
git add notes.md
git commit -m "feat: add a greeting"
git switch main
git log --oneline     # the greeting commit isn't here
```

*A branch is just a movable pointer at a commit. Switching branches
rewrites the files in your working tree to match.*

Now make both sides diverge, so the merge is a real one:

```sh
echo "Hello from main" >> notes.md
git add notes.md
git commit -m "docs: add a line on main"
git merge feature/greeting
```

*→ **conflict.** Both branches changed the same part of the same file, so
git stops and asks you to decide. Open `notes.md` and you'll see the
markers:*

```sh
git status
cat notes.md
```

Now demo the conflict-marker hook — try committing without resolving:

```sh
git add notes.md
git commit -m "fix: merge branches"
```

*→ **blocked**: `Merge conflict string '<<<<<<<' found`. A hook caught
that you'd staged the conflict markers themselves. Genuinely easy mistake;
nice to have a safety net.*

Resolve it properly (edit `notes.md` to keep both lines, delete the
`<<<<<<<`, `=======`, `>>>>>>>` lines), then:

```sh
git add notes.md
git commit -m "fix: resolve merge conflict"
git log --oneline --graph --all
```

*See the two lines joining back up — that's a merge commit, the only kind
with two parents.*

## 5. Remotes

*GitHub is, at heart, just a git repo that lives somewhere everyone can
reach. Let's build a stand-in locally — no account needed.*

```sh
git init --bare /tmp/git-demo-remote.git
git remote add origin /tmp/git-demo-remote.git
git push -u origin main
```

Now show what a teammate sees:

```sh
git clone /tmp/git-demo-remote.git /tmp/git-demo-clone
cd /tmp/git-demo-clone
git log --oneline --graph        # the full history came with it
ls .git/hooks/ | grep -v sample  # ...but no hooks are active here!
cd /Users/jay/Git/Me/git-demo
```

*Important detail: the clone has the hook **config** (it's a tracked file)
but the hooks aren't running — the bits that live inside `.git/` never
travel. A teammate has to run `pre-commit install --install-hooks` once.
That's the natural segue into...*

## 6. The hooks (open the hood)

```sh
ls .git/hooks/            # git ships samples; none run by default
cat .pre-commit-config.yaml
```

Walk through [HOOKS.md](HOOKS.md) together. The shape of it:

- A git hook is a script git runs at a set moment (before a commit, after
  a message, before a push).
- Hand-writing them in `.git/hooks/` is a pain: they're not shared, and
  you'd be reinventing checks everyone needs.
- So `pre-commit` reads a **tracked** config of published, version-pinned
  hooks, installs each in its own environment, and wires up the shims.

Everything they've already been bitten by, named:

| What happened | Which hook |
| --- | --- |
| `"first commit"` rejected | `conventional-pre-commit` (commit-msg stage) |
| trailing spaces fixed for you | `trailing-whitespace` |
| conflict markers caught | `check-merge-conflict` |

One they haven't seen yet — the secret scanner. This is usually the one
that sells it:

```sh
printf 'aws_access_key_id = AKIA4T7WZQ2JXKPLMN3R\n' > creds.txt
git add creds.txt
git commit -m "chore: add credentials"
```

*→ **blocked** by `gitleaks`, with the secret redacted in the output. That
check is the entire reason a lot of teams adopt this.*

```sh
git reset creds.txt && rm creds.txt
```

Then the escape hatches, so they know hooks aren't a prison:

```sh
pre-commit run --all-files          # run everything over the whole repo
SKIP=gitleaks git commit -m "..."   # skip one hook
git commit --no-verify -m "..."     # skip all of them
```

*Worth being honest here: `--no-verify` exists and is sometimes the right
call. Hooks are a fast feedback loop, not a security boundary — anything
that really matters gets enforced again in CI, where nobody can skip it.*

## Cleanup (after the demo)

```sh
rm -rf /tmp/git-demo-remote.git /tmp/git-demo-clone
```

To reset this repo to a blank slate and run the whole thing again:

```sh
cd /Users/jay/Git/Me/git-demo
rm -rf .git && git init -b main && pre-commit install --install-hooks
git remote remove origin 2>/dev/null
```
