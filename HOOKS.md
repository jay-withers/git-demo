# Git hooks in this repo

This repo uses the [**pre-commit**](https://pre-commit.com/) framework to
manage its git hooks. This doc explains what that means, what's installed,
and how to set the same thing up in a new repo.

## What a git hook is

A git hook is just a script git runs at a particular moment — before a
commit is created, after you type a commit message, before a push, and so
on. Every repo already has a `.git/hooks/` directory; run `ls .git/hooks/`
and you'll see a pile of `*.sample` files git ships as examples. None of
them run unless you strip the `.sample` suffix.

Two problems with managing hooks that way by hand:

1. `.git/hooks/` lives inside `.git`, which is **never committed or
   pushed**. A hook you write there only ever affects your own clone.
2. You end up hand-rolling and maintaining shell scripts for checks that
   thousands of other projects also need.

## What pre-commit does about it

`pre-commit` solves both. You declare the checks you want in a **tracked**
[`.pre-commit-config.yaml`](.pre-commit-config.yaml), each one pointing at
a published, version-pinned hook repo. The tool then:

- writes small shim scripts into `.git/hooks/` for you (`pre-commit install`),
- clones each hook repo and installs it in its own isolated environment,
  so hooks can be written in Python, Go, Node, whatever — you don't need
  those toolchains set up yourself,
- runs the right hooks at the right git stage, only on the files you staged.

So the *config* is shared and reviewable in the repo, and the *hooks
themselves* are off-the-shelf and versioned.

### The one manual step per clone

The config is tracked, but the shims in `.git/hooks/` are not — they can't
be, they live inside `.git`. So after cloning this repo you must run once:

```sh
pre-commit install --install-hooks
```

Until you do, the config sits there doing nothing. (You can prove this:
clone the repo and check `.git/hooks/` — only `.sample` files.) Projects
usually put that line in their README or a `make setup` target.

If you don't have the tool yet, see the
[prerequisites](README.md#prerequisites-macos) — short version on macOS is
`brew install pre-commit`, and you don't need to install Python or Go
yourself even though the hooks are written in them.

## What's installed here

All of these come from published repos — see
[`.pre-commit-config.yaml`](.pre-commit-config.yaml) for the exact pinned
versions.

| Hook | Repo | What it does |
| --- | --- | --- |
| `trailing-whitespace` | [pre-commit-hooks](https://github.com/pre-commit/pre-commit-hooks) | Strips trailing spaces. **Fixes the file and aborts** — re-stage and commit again. |
| `end-of-file-fixer` | pre-commit-hooks | Ensures files end in exactly one newline. Also auto-fixes. |
| `check-merge-conflict` | pre-commit-hooks | Blocks committing leftover `<<<<<<<` / `=======` / `>>>>>>>` markers. Only active while the repo is genuinely mid-merge. |
| `check-added-large-files` | pre-commit-hooks | Blocks files over 500 kB — keeps the repo from bloating. |
| `check-yaml` | pre-commit-hooks | Validates YAML syntax (including this repo's own config). |
| `detect-private-key` | pre-commit-hooks | Blocks committing an SSH/TLS private key. |
| `gitleaks` | [gitleaks](https://github.com/gitleaks/gitleaks) | Broad secret scanning — API keys, cloud credentials, tokens. |
| `conventional-pre-commit` | [conventional-pre-commit](https://github.com/compilerla/conventional-pre-commit) | Runs at the **commit-msg** stage: rejects messages that aren't [Conventional Commits](https://www.conventionalcommits.org/) format (`feat: ...`, `fix: ...`, `docs: ...`). |

The first seven run at the `pre-commit` stage (before the commit is
created); the last runs at `commit-msg` (after you've typed the message).
`default_install_hook_types` in the config is what tells `pre-commit
install` to wire up both stages.

`gitleaks` reads its own [`.gitleaks.toml`](.gitleaks.toml), which extends
the default ruleset with one narrow allowlist: this repo's README quotes a
deliberately fake AWS key (because the walkthrough shows gitleaks blocking
one), and without the exception the doc describing the check would trip
the check. Real repos need this too, for test fixtures and example config.

## Useful commands

```sh
pre-commit run --all-files     # run every hook over the whole repo, not just staged files
pre-commit autoupdate          # bump the pinned `rev:`s to the latest releases
pre-commit install --install-hooks   # enable hooks in this clone (do this after cloning)

SKIP=gitleaks git commit -m "..."    # skip one specific hook, just this once
git commit --no-verify -m "..."      # skip ALL hooks — use sparingly and honestly
```

## Other stages you can hook into

This repo only uses `pre-commit` and `commit-msg`, but the framework
supports the rest of git's hook points too — `pre-push` (e.g. run the test
suite before anything leaves your machine), `post-checkout` and
`post-merge` (e.g. remind people to reinstall dependencies after switching
branches or pulling). You opt in by adding `stages: [pre-push]` to a hook
and including that stage in `default_install_hook_types`.

## Setting this up in a new repo

```sh
brew install pre-commit           # or: pipx install pre-commit
cd your-repo
# write a .pre-commit-config.yaml (copy this repo's as a starting point)
pre-commit install --install-hooks
git add .pre-commit-config.yaml
git commit -m "chore: add pre-commit hooks"
```

Everyone who clones the repo afterwards runs `pre-commit install
--install-hooks` once, and they're on the same checks as you.
