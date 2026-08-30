# Learning git, by doing it

A hands-on walkthrough of git for someone who's never used it. Work down
this page a section at a time: run the commands, look at what git says
back. Explanations are in *italics* under each block.

By the end you'll have made commits, branched, hit a merge conflict and
resolved it, pushed to GitHub, and seen automated checks block a bad
commit.

This repo also has **git hooks** already switched on via the
[pre-commit framework](https://pre-commit.com/), so some of your commits
will get checked — and occasionally rejected. That's deliberate. Section 7
and [HOOKS.md](HOOKS.md) explain how it all works; until then, just note
when it happens.

## Prerequisites (macOS)

Do these **before** you start — the last step downloads a few hundred MB
and you don't want to sit watching it.

### Git

```sh
git --version
```

*If that prints a version, you're done. macOS doesn't ship git on its own
— it comes with Apple's Command Line Tools, and running the command above
on a fresh Mac pops up a dialog offering to install them. Accept it, or
run `xcode-select --install` yourself. Takes a few minutes.*

*Apple's git is fine for everything here. If you'd rather have a newer
one, `brew install git` once you've got Homebrew below.*

### Tell git who you are

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

### Homebrew

The standard package manager for macOS — the easiest way to get the
remaining tool.

```sh
brew --version
```

*Not installed? Follow the one-liner at <https://brew.sh>. It'll ask for
your password and take a few minutes.*

### pre-commit

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

## 1. Getting the repo onto your machine

This repo lives at <https://github.com/jay-withers/git-demo>. Open it in a
browser tab now — it's useful to see the same commits show up there as we
go. What you're looking at is the copy on GitHub; the next step makes you
your own.

### Pick somewhere to put it

Clone into a folder you'll be able to find again. Anywhere in your home
directory is fine:

```sh
mkdir -p ~/Projects
cd ~/Projects
```

*`git clone` creates a **new folder** named after the repo, inside
whatever directory you're sitting in — so run it from the parent, not from
an empty folder you've already made for it. Otherwise you end up with
`~/Projects/git-demo/git-demo`.*

### Clone it

```sh
git clone https://github.com/jay-withers/git-demo
```

*You should see `Cloning into 'git-demo'...` and a few lines of counting
objects. Because this repo is public you don't need a GitHub account or a
password for this step — cloning over HTTPS just downloads it.*

*A clone is not a download of the current files. It's a **full copy of the
repo**: every commit, every branch, the whole history, on your disk. That's
why the rest of this walkthrough works offline — `git log`, `git diff`,
branching, committing all happen locally, with no network involved.*

Move into it:

```sh
cd git-demo
ls -a
```

*Alongside the files you'd expect there's a `.git` directory. **That** is
the repository — the history and all of git's bookkeeping. The visible
files are just the version of them git currently has checked out for you.
Delete `.git` and you're left with an ordinary folder; leave it alone and
you can go back to any commit ever made.*

### Look around before changing anything

```sh
git log --oneline    # the history: who changed what, and why
git status           # clean — nothing changed yet
git remote -v        # "origin" — the GitHub URL you cloned from
```

*`git status` saying `nothing to commit, working tree clean` means your
files match the last commit exactly. `origin` is the name git
automatically gave the place you cloned from — section 6 comes back to
it.*

### Switch the hooks on

```sh
pre-commit install --install-hooks
```

*This does two things: writes the hook scripts into this clone, and
downloads an isolated environment for each one. **The first run takes a
few minutes and pulls down a few hundred MB** — it's cached afterwards, so
it only happens once per machine.*

*You have to run this in every fresh clone. Section 6 shows why.*

### One caveat about pushing

*Everything up to section 6 is local, so you can follow along as-is. But
`git push` needs write access to the repo you cloned, and you won't have
it on someone else's. If you want section 6 to work, click **Fork** on the
GitHub page first to get your own copy, and clone that URL instead —
`https://github.com/YOUR-USERNAME/git-demo`. Pushing to your own fork also
means GitHub will prompt you to log in the first time; a browser popup or
a personal access token, depending on your setup.*

## 2. The mental model

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

## 3. Your first commit

Make a change and watch git notice:

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
stays skimmable later. Section 7 covers where that rule comes from; for
now, just give it what it wants:*

```sh
git commit -m "docs: add a line to notes"
git log --oneline
```

*Note what scrolled past — a list of checks, all passing. That's the hooks.*

## 4. Staged vs unstaged

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
git commit -m "docs: add a messy line"
```

*→ **blocked**, but look: `files were modified by this hook`. It didn't
just refuse, it stripped the trailing whitespace for you. The fix is
sitting in your working tree — you just have to stage it and go again:*

```sh
git diff              # the hook's own fix, unstaged
git add notes.md
git commit -m "docs: add a messy line"
```

## 5. Branching & merging

*One thing that makes "a branch is a pointer" concrete: in a repo with
**no** commits, `git branch foo` fails with `not a valid object name` —
there's no commit for the pointer to point at. We have commits, so:*

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

Try committing without resolving it:

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

## 6. Remotes

*Every commit you've made so far exists only on this laptop. Cloning
pulled the history down, but nothing has gone back the other way. A
**remote** is just another copy of the repo somewhere else — GitHub is, at
heart, exactly that.*

```sh
git remote -v         # "origin" — where you cloned from
git status            # "ahead of origin/main by N commits"
```

*That "ahead by N" is git telling you those commits exist only here. Open
the GitHub page and you won't find them.*

```sh
git push
```

*Refresh the GitHub page — there they are. That's the whole idea: commit
locally as often as you like, push when you want to share.*

Now let's see what a teammate gets. Clone it into a different folder —
this is exactly what anyone else on the project would do:

```sh
git clone https://github.com/jay-withers/git-demo /tmp/git-demo-clone
cd /tmp/git-demo-clone
git log --oneline --graph        # the full history came with it
ls .git/hooks/ | grep -v sample  # ...but no hooks are active here!
cd -                             # back to your clone
```

*Important detail: the clone has the hook **config** (it's a tracked file)
but the hooks aren't running there — anything inside `.git/` never
travels. That is why section 1 had you run `pre-commit install`, and why
anyone joining a project has to do the same. Which brings us to...*

## 7. The hooks

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

Hooks aren't a prison, though. The escape hatches:

```sh
pre-commit run --all-files          # run everything over the whole repo
SKIP=gitleaks git commit -m "..."   # skip one hook
git commit --no-verify -m "..."     # skip all of them
```

*Worth being clear: `--no-verify` exists and is sometimes the right
call. Hooks are a fast feedback loop, not a security boundary — anything
that really matters gets enforced again in CI, where nobody can skip it.*

## Cleanup

Remove the throwaway clone from section 6:

```sh
rm -rf /tmp/git-demo-clone
```

Everything else you did lives in your own clone. To start over, delete it
and clone again — nothing you did affects anyone else unless you pushed.

### Resetting the shared repo (owner only)

*Only relevant if you own this repo and want to run the walkthrough again
from a clean history. It rewrites the published `main`, so don't do it
while anyone else has a clone they care about.*

```sh
git switch main
git reset --hard b972082         # the initial scaffolding commit
git branch -D feature/greeting
git push --force-with-lease origin main
```

*`reset --hard` permanently discards uncommitted work and every commit
after that point. It's the one genuinely destructive command on this page
— check `git log --oneline` first, and be sure you mean it.*
