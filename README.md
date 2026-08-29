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
| **New formula** | Me, by hand | Scaffold the file here, push to `main`, then trigger the source repo's workflow to fill in the real version |
| **Version bump** | GitHub Actions | The source repo's release workflow pushes the new `url` + `sha256` into this repo |

Steady state is that I almost never edit this repo directly. Each tool's own repo owns its
version number, so the release workflow *there* is what rewrites the formula *here*.

### Adding a new formula (the manual part)

1. Create `Formula/<name>.rb` from the scaffolding below. The file basename must match the
   class name in CamelCase — `Formula/my-tool.rb` defines `class MyTool` — or `brew` silently
   won't see it.
2. Point `url`/`sha256` at a release that already exists, so the formula is installable from
   the moment it lands. Get the hash with:
   ```sh
   curl -sL https://github.com/tockrock/<repo>/archive/refs/tags/v0.1.0.tar.gz | shasum -a 256
   ```
3. Check it locally before pushing (see [Local checks](#local-checks)).
4. Commit and push to `main`.
5. Add the bump workflow to the tool's own repo (below), then trigger it — from then on that
   repo maintains this file.

### Scaffolding

```ruby
class MyTool < Formula
  desc "Does the thing"        # no leading article, don't repeat the formula name
  homepage "https://github.com/tockrock/my-tool"
  url "https://github.com/tockrock/my-tool/archive/refs/tags/v0.1.0.tar.gz"
  sha256 "0000000000000000000000000000000000000000000000000000000000000000"
  license "MIT"                # `brew audit --new` requires this

  depends_on "go" => :build

  def install
    system "go", "build", *std_go_args(ldflags: "-s -w")
  end

  test do
    assert_match "0.1.0", shell_output("#{bin}/my-tool --version")
  end
end
```

The `test do` block is the only test suite a tap has — keep it meaningful, since it's what
runs if anything ever validates this formula.

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
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: tockrock/homebrew-personal
          token: ${{ secrets.HOMEBREW_TAP_TOKEN }}
      - name: Rewrite url and sha256
        env:
          TAG: ${{ github.event.release.tag_name || github.ref_name }}
        run: |
          set -euo pipefail
          VERSION="${TAG#v}"
          URL="https://github.com/${{ github.repository }}/archive/refs/tags/${TAG}.tar.gz"
          SHA=$(curl -fsSL "$URL" | shasum -a 256 | cut -d' ' -f1)
          F=Formula/my-tool.rb
          sed -i -E "s|^  url \".*\"|  url \"${URL}\"|" "$F"
          sed -i -E "s|^  sha256 \".*\"|  sha256 \"${SHA}\"|" "$F"
          git config user.name  github-actions
          git config user.email github-actions@github.com
          git commit -am "my-tool ${VERSION}"
          git push
```

**`HOMEBREW_TAP_TOKEN` is the part that's easy to forget.** The default `GITHUB_TOKEN` only
has rights to the repo it runs in, so it cannot push here. Each source repo needs a
fine-grained PAT with **Contents: read and write** on `tockrock/homebrew-personal`, stored as
a repo secret. If pushes start failing with a 403, the PAT has most likely expired.

## Local checks

The working copy is itself a valid tap, so run these against the file path directly:

```sh
brew style ./Formula/<name>.rb                         # lint (--fix autocorrects)
brew audit --strict --new ./Formula/<name>.rb          # required for a formula that's new
brew install --build-from-source ./Formula/<name>.rb   # real build
brew test ./Formula/<name>.rb                          # run the test do block
```

`--new` is the strictest audit and only applies the first time; later bumps just need
`brew audit --strict`.
