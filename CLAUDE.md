# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A personal Homebrew tap (`ssh://git@github.com/tockrock/homebrew-personal.git`), described by
the README as "Tools for myself". As of this writing the repo contains no formulae yet, so
there is no build, lint, or test setup beyond what `brew` itself provides.

Because the GitHub repo is named `homebrew-personal` under the `tockrock` user, Homebrew resolves
it as the tap `tockrock/personal`:

```sh
brew tap tockrock/personal
brew install tockrock/personal/<formula>
```

## Tap layout Homebrew expects

Homebrew discovers files by directory and by class name, so placement and naming are not
stylistic choices:

- Formulae go in `Formula/<name>.rb` (`HomebrewFormula/` and the repo root also work, but
  `Formula/` is the convention to follow here). Casks go in `Casks/<name>.rb`.
- The file basename must match the class name in CamelCase: `Formula/my-tool.rb` defines
  `class MyTool < Formula`. A mismatch makes the formula invisible to `brew`.
- A tap is a plain git repo — `brew tap` clones it and `brew update` pulls it. There is no build
  or publish step; pushing to `main` is the release.

## Who owns which edits

This is the load-bearing convention in this repo, and it is not visible from the files here:

- **Version bumps are automated and arrive from elsewhere.** Each tool's own repo runs a
  release workflow that checks out this tap, rewrites its formula's `url` and `sha256`, and
  pushes. Those commits are authored by `github-actions`. Do not hand-edit a `url`, `sha256`,
  or version in an existing formula — the source repo is the authority, and a manual edit is
  either redundant or will be clobbered by the next release.
- **New formulae are scaffolded here by hand**, pushed to `main`, and only then handed over to
  the source repo's workflow. This is the one time editing a formula directly is correct.

So: a request to "update <tool> to version X" is usually a task for that tool's repo, not this
one. Ask before hand-editing a version here. The README documents the workflow and carries the
scaffolding template plus the source-repo workflow YAML — keep the two files in agreement when
either changes.

Cross-repo pushes need a fine-grained PAT with Contents: read/write on this repo, stored in
each source repo as the `HOMEBREW_TAP_TOKEN` secret; the default `GITHUB_TOKEN` cannot reach
across repos. A 403 on push from a source repo's workflow means an expired or missing PAT.

## Working on a formula

Once formulae exist, the local checkout is itself a usable tap, so iterate against it directly:

```sh
brew install --build-from-source ./Formula/<name>.rb   # build the local file
brew audit --strict --new ./Formula/<name>.rb          # required checks for a new formula
brew style ./Formula/<name>.rb                         # RuboCop lint, --fix to autocorrect
brew test ./Formula/<name>.rb                          # run the formula's `test do` block
brew reinstall --build-from-source ./Formula/<name>.rb # re-run a build after editing
```

`brew audit` and `brew style` are the lint gate; the `test do` block inside each formula is the
only test suite a tap has, and `brew test` on a single file is how you run one in isolation.

Formula edits that change the download need a matching `sha256`. Get it with
`shasum -a 256 <file>` on the fetched tarball, or let a failed install print the actual hash.
