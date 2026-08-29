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
git log --oneline                    # one commit: the scaffolding
git status                           # clean
pre-commit install --install-hooks   # already done; just confirms hooks are live
```

*The hook environments are already downloaded and cached, so nothing will
stall mid-demo.*

*This repo lives at <https://github.com/jay-withers/git-demo>, so you can
show the same commits in the GitHub UI as you go — useful for connecting
"the thing I typed" to "the thing they've seen in a browser".*

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

## 2. Your first commit

```sh
git log
git status            # clean — nothing changed yet
```

*`git log` is the history: who changed what, when, and why. Right now
there's one commit — the files that were already here.*

Now make a change and watch it appear:

```sh
echo "- a line my mate added" >> notes.md
git status
```

*`notes.md` is now **modified**. Git noticed, but hasn't recorded
anything — nothing is saved until you commit.*

```sh
git diff              # exactly what changed
git add notes.md      # stage it: "this goes in the next commit"
git status            # now it's staged, not just modified
```

Let the commit-message hook bite (this is the hook teaser):

```sh
git commit -m "my first commit"
```

*→ **blocked.** Something checked the commit message and didn't like it.
Park that thought — we'll come back to it. It wants a `type: description`
format:*

```sh
git commit -m "docs: add a line to notes"
git log --oneline
```

*Note what scrolled past — a list of checks, all passing. That's the hooks.*

## 3. Staged vs unstaged

```sh
echo "- and another" >> notes.md
git diff              # unstaged changes
git add notes.md
git diff              # empty now — it's staged
git diff --cached     # there it is
git commit -m "docs: add another line"
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

*Worth showing first, since it makes "a branch is a pointer" concrete: in
a repo with **no** commits, `git branch foo` fails with `not a valid
object name` — there's no commit for the pointer to point at. We have
commits now, so:*

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

*Everything so far has been entirely on this laptop — no internet
involved. A **remote** is just another copy of the repo somewhere else.
GitHub is, at heart, exactly that: a git repo that lives somewhere
everyone can reach.*

```sh
git remote -v         # already pointing at GitHub
git status            # "ahead of origin/main by N commits"
```

*That "ahead by N" is git telling you those commits exist only here. Show
them the GitHub page — the new commits aren't there yet.*

```sh
git push
```

*Refresh the GitHub page. Now they are. That's the whole idea: commit
locally as often as you like, push when you want to share.*

Now show what a teammate sees — have them clone it themselves if they've
got git installed, or do it yourself in another folder:

```sh
git clone https://github.com/jay-withers/git-demo /tmp/git-demo-clone
cd /tmp/git-demo-clone
git log --oneline --graph        # the full history came with it
ls .git/hooks/ | grep -v sample  # ...but no hooks are active here!
cd /Users/jay/Git/Me/git-demo
```

*Important detail: the clone has the hook **config** (it's a tracked file)
but the hooks aren't running — anything inside `.git/` never travels. A
teammate has to run `pre-commit install --install-hooks` once. That's the
natural segue into...*

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

## Cleanup / resetting for a second run

Remove the throwaway clone:

```sh
rm -rf /tmp/git-demo-clone
```

To run the demo again from scratch, roll the repo back to just the
scaffolding commit. `b972082` is the initial commit — check with
`git log --oneline` that it's still the first one.

```sh
cd /Users/jay/Git/Me/git-demo
git checkout main
git reset --hard b972082
git branch -D feature/greeting 2>/dev/null
git push --force-with-lease origin main
```

*`reset --hard` throws away uncommitted work and any commits after that
point — which is what you want here, but it's the one command in this
runbook that genuinely destroys things. Don't demo it casually.*
