# Learning git, by doing it

A hands-on walkthrough of git for someone who's never used it. Work down
this page a section at a time: run the commands, look at what git says
back. Explanations are in *italics* under each block.

By the end you'll have made commits, branched, hit a merge conflict and
resolved it, pushed to GitHub, and seen automated checks block a bad
commit.

This repo also has **git hooks** already switched on via the
[pre-commit framework](https://pre-commit.com/), so some of your commits
will get checked — and occasionally rejected. That's deliberate. Section 6
and [HOOKS.md](HOOKS.md) explain how it all works; until then, just note
when it happens.

## Prerequisites (macOS)

Do these **before** you start — the last step downloads a few hundred MB
and you don't want to sit watching it.

### 1. Git

```sh
git --version
```

*If that prints a version, you're done. macOS doesn't ship git on its own
— it comes with Apple's Command Line Tools, and running the command above
on a fresh Mac pops up a dialog offering to install them. Accept it, or
run `xcode-select --install` yourself. Takes a few minutes.*

*Apple's git is fine for everything here. If you'd rather have a newer
one, `brew install git` after step 3.*

### 2. Tell git who you are

Every commit is permanently stamped with a name and email:

```sh
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

*Don't skip this. Git won't stop you — if you haven't set them it quietly
invents something from your computer's username and hostname (like
`jay@Jays-MacBook-Pro.local`), warns once, and commits anyway. That junk
address is then baked into the history for good, and fixing it afterwards
means rewriting commits. Two seconds now saves that.*

*`--global` means "for every repo on this machine" — you do this once,
ever. Drop the flag inside a repo to override it just there (handy if you
use a different address for work).*

*Use the email tied to your GitHub account, otherwise GitHub won't
connect your commits to your profile.*

Check it took:

```sh
git config --global --list
```

While you're here, make new repos default to a `main` branch rather than
the older `master`:

```sh
git config --global init.defaultBranch main
```

### 3. Homebrew

The standard package manager for macOS — the easiest way to get the
remaining tool.

```sh
brew --version
```

*Not installed? Follow the one-liner at <https://brew.sh>. It'll ask for
your password and take a few minutes.*

### 4. pre-commit

This is what runs the repo's hooks.

```sh
brew install pre-commit
pre-commit --version
```

**You do not need to install Python separately.** People often assume you
do, because `pre-commit` is a Python tool and some of the hooks are Python
too. But Homebrew installs its own Python alongside it, and macOS already
ships a `python3` for the rest. Nothing for you to manage.

*(You also don't need Go, despite one of the hooks being a Go program —
`pre-commit` downloads a private copy of whatever each hook needs.)*

*If you'd rather not use Homebrew, `pipx install pre-commit` works too —
that route does expect a Python you've installed yourself.*

## 0. Setup

Clone the repo and switch the hooks on:

```sh
git clone https://github.com/jay-withers/git-demo
cd git-demo
pre-commit install --install-hooks
```

*That last command does two things: writes the hook scripts into this
clone, and downloads an isolated environment for each one. **The first
run takes a few minutes and pulls down a few hundred MB** — it's cached
afterwards, so it only happens once per machine.*

Have a look around before changing anything:

```sh
git log --oneline    # the history so far
git status           # clean — nothing changed yet
```

*This repo lives at <https://github.com/jay-withers/git-demo>. Keep it
open in a browser tab — it's useful to see the same commits show up there
as we go.*

## 1. The mental model

*Three things to keep straight: the **working tree** (files on disk right
now), the **staging area** (what you've told git to include in the next
commit), and the **history** (snapshots you've saved). Git makes you
choose what goes in a commit — that's the staging area's whole job, and
it's the part that trips people up most.*

*Every commit is also stamped with the identity you set in the
prerequisites. Confirm it's what you expect:*

```sh
git config --get user.name
git config --get user.email
```

*Also worth knowing up front: `git log`, `git status` and `git diff` never
change anything. They only tell you where you are. When you're lost, they
are always the right first move.*

## 2. Your first commit

```sh
git log
git status            # clean — nothing changed yet
```

*`git log` is the history: who changed what, when, and why. Right now
there's one commit — the files that were already here.*

Now make a change and watch it appear:

```sh
echo "- my first change" >> notes.md
git status
```

*`notes.md` is now **modified**. Git noticed, but hasn't recorded
anything — nothing is saved until you commit.*

```sh
git diff              # exactly what changed
git add notes.md      # stage it: "this goes in the next commit"
git status            # now it's staged, not just modified
```

Now commit it — but deliberately with a sloppy message, to meet the first
hook:

```sh
git commit -m "my first commit"
```

*→ **blocked.** One of this repo's hooks read the commit message and
rejected it. It wants the form `type: description`, so that the history
stays skimmable later. Section 6 covers where that rule comes from; for
now, just give it what it wants:*

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

Now let's see what a teammate gets. Clone it into a different folder —
this is exactly what anyone else on the project would do:

```sh
git clone https://github.com/jay-withers/git-demo /tmp/git-demo-clone
cd /tmp/git-demo-clone
git log --oneline --graph        # the full history came with it
ls .git/hooks/ | grep -v sample  # ...but no hooks are active here!
cd /Users/jay/Git/Me/git-demo
```

*Important detail: the clone has the hook **config** (it's a tracked file)
but the hooks aren't running there — anything inside `.git/` never
travels. That's why section 0 had you run `pre-commit install`, and why
anyone joining a project has to do the same. Which brings us to...*

## 6. The hooks

Those checks that kept interrupting you — here's what they are.

```sh
ls .git/hooks/            # git ships samples; none of them run by default
cat .pre-commit-config.yaml
```

The shape of it (see [HOOKS.md](HOOKS.md) for the full version):

- A git hook is a script git runs at a set moment — before a commit is
  created, after you type a message, before a push.
- Hand-writing them in `.git/hooks/` is a pain: that directory is never
  committed, so nobody else on the project gets your hooks, and you'd be
  reinventing checks everyone needs.
- So `pre-commit` reads a **tracked** config of published, version-pinned
  hooks, installs each in its own environment, and writes the shims into
  `.git/hooks/` for you.

Which means every interruption so far now has a name:

| What happened | Which hook |
| --- | --- |
| `"my first commit"` rejected | `conventional-pre-commit` (commit-msg stage) |
| trailing spaces fixed for you | `trailing-whitespace` |
| conflict markers caught | `check-merge-conflict` |

There's one more you haven't hit yet, and it's usually the one that sells
the whole idea — the secret scanner:

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
