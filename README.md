# homebrew-personal

Tools for myself.

```sh
brew tap tockrock/personal
brew install tockrock/personal/<formula>
```

The repo is named `homebrew-personal`, so Homebrew refers to the tap as `tockrock/personal`.
There is no build or release step — this repo *is* the distribution channel, and pushing to
`main` publishes immediately to anyone who has tapped it.

## How formulae get updated

Two different paths, and it matters which one you're on:

| | Who does it | How |
|---|---|---|
| **New formula** | Me, by hand | `brew create` the scaffolding here, push to `main`, then add the bump workflow to the tool's repo |
| **Version bump** | GitHub Actions | The tool's release workflow rewrites `url` + `sha256` here and pushes |

Steady state is that I almost never edit this repo directly. Each tool's own repo owns its
version number, so the release workflow *there* is what rewrites the formula *here*.

This is deliberately a *push* model. Homebrew's own update path (`brew livecheck` / `brew bump`
/ `brew bump-formula-pr`) is *pull* — built for maintainers packaging software they don't
control, so it has to poll to discover releases. I own these upstreams, so the release event
is the trigger, and nothing has to guess a version. It would only be worth switching to
`brew bump --tap=tockrock/personal` if this tap ever packaged someone else's tool.

### Adding a new formula (the manual part)

Use `brew create` rather than writing the file from scratch — it fetches the tarball, computes
the `sha256`, and fills in `desc`/`homepage` from the GitHub API:

```sh
brew tap tockrock/personal            # only needed once per machine
brew create --tap=tockrock/personal \
  --set-name my-tool --set-version 0.1.0 \
  https://github.com/tockrock/my-tool/releases/download/v0.1.0/my-tool.tar.gz
```

- **Set `--set-name` / `--set-version` explicitly.** Release assets here are named
  `my-tool.tar.gz` with the version only in the *tag*, so Homebrew's auto-derivation from the
  URL has nothing to work with and will guess wrong.
- **`brew create` writes to Homebrew's clone**, at
  `/opt/homebrew/Library/Taps/tockrock/homebrew-personal` — not to a separate working copy
  elsewhere. Commit and push from wherever the tap's git remote actually lives.
- It opens the new file in `$EDITOR` (`nova --wait`) and blocks until the window is closed.
- Add `--no-fetch` to skip the download and get a bare skeleton with no hash — for when the
  release doesn't exist yet. Add `--go`, `--rust`, `--cmake`, etc. for a build-from-source
  template instead of the binary pattern below.

Then:

1. Trim the generated file down to the binary pattern below. The file basename must match the
   class name in CamelCase — `Formula/my-tool.rb` defines `class MyTool` — or `brew` silently
   won't see it.
2. Check it locally (see [Local checks](#local-checks)).
3. Commit and push to `main`.
4. Add the bump workflow to the tool's own repo — from then on that repo maintains this file.

### The shape these formulae take

Prebuilt binary from a GitHub release, no build step:

```ruby
class MyTool < Formula
  desc "Does the thing"        # no leading article, don't repeat the formula name
  homepage "https://github.com/tockrock/my-tool"
  url "https://github.com/tockrock/my-tool/releases/download/v0.1.0/my-tool.tar.gz"
  sha256 "0000000000000000000000000000000000000000000000000000000000000000"

  depends_on :macos => :sequoia
  depends_on "some-dependency"
  depends_on "tockrock/personal/another-tool"   # tap-local deps need the full name

  def install
    bin.install "my-tool"
  end

  test do
    system "#{bin}/my-tool", "--help"
  end
end
```

`license` is intentionally omitted — `brew audit --new` wants it, but that audit is optional
for a personal tap. The `test do` block is the only test suite a tap has.

### The bump workflow (lives in the tool's repo, not here)

On release, the source repo rewrites its own formula in this tap and pushes:

```yaml
name: Bump Homebrew formula
on:
  release:
    types: [published]
  workflow_dispatch:          # so a new formula can be filled in on demand

jobs:
  bump:
    runs-on: macos-latest     # see note below
    steps:
      - uses: actions/checkout@v4
        with:
          repository: tockrock/homebrew-personal
          token: ${{ secrets.HOMEBREW_TAP_TOKEN }}
          path: tap
      - name: Bump url and sha256
        env:
          TAG: ${{ github.event.release.tag_name || github.ref_name }}
        run: |
          set -euo pipefail
          URL="https://github.com/${{ github.repository }}/releases/download/${TAG}/my-tool.tar.gz"
          SHA=$(curl -fsSL "$URL" | shasum -a 256 | cut -d' ' -f1)
          brew tap tockrock/personal "${GITHUB_WORKSPACE}/tap"
          brew bump-formula-pr --write-only --commit --no-audit \
            --url="$URL" --sha256="$SHA" \
            tockrock/personal/my-tool
      - name: Push
        working-directory: tap
        run: git push
```

`brew bump-formula-pr --write-only` does the file edit with no fork, no PR, and no GitHub API
— `--commit` adds the commit, and pushing is left to the next step. It parses the formula
properly rather than pattern-matching on indentation, and clears a stale `revision` if one is
present. A `sed -i -E "s|^  url \".*\"|  url \"${URL}\"|"` over the two lines works too and
needs no Homebrew on the runner; it's just easier to get subtly wrong.

Two things that are easy to forget:

- **`HOMEBREW_TAP_TOKEN`.** The default `GITHUB_TOKEN` only has rights to the repo it runs in,
  so it cannot push here. Each source repo needs a fine-grained PAT with **Contents: read and
  write** on `tockrock/homebrew-personal`, stored as a repo secret. A 403 on push means the PAT
  expired or was never added to that repo.
- **`macos-latest`, not `ubuntu-latest`.** These formulae carry `depends_on :macos`, and
  Homebrew on Linux can refuse to load a macOS-only formula — which `bump-formula-pr` has to do
  before editing it. The `sed` approach doesn't care and runs anywhere.

## Local checks

The working copy is itself a valid tap, so run these against the file path directly:

```sh
brew style ./Formula/<name>.rb                         # lint (--fix autocorrects)
brew audit --strict ./Formula/<name>.rb                # add --new for a first-time formula
brew install --build-from-source ./Formula/<name>.rb   # real install
brew test ./Formula/<name>.rb                          # run the test do block
brew uninstall <name> && brew install <name>           # verify a bump end to end
```
