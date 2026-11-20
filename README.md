# tools

Scripts for the Haskell packages of [jprupp](https://github.com/jprupp) on
GitHub, such as haskoin-core, haskoin-node and haskoin-store. Each works on
the git repository of the directory that it is run from, so one copy serves
every package.

## Install

Clone the repository and put it on your `PATH`:

```sh
git clone https://github.com/jprupp/tools.git ~/Source/tools
export PATH="$HOME/Source/tools:$PATH"   # in ~/.bashrc
```

## The packages they expect

One package at the root of its repository, described by `package.yaml`
(hpack) and built with [Stack](https://haskellstack.org), released from the
branch `master`, which is on GitHub as `origin`. Its `CHANGELOG.md` has a
heading for each version, `## [1.2.3]` or `## 1.2.3`, followed by anything.

## Scripts

### `haskell-release`

Releases the version in `package.yaml` to Hackage, with its documentation.
Bump the version, write its CHANGELOG entry, commit and push, then run it.
It stops at the first step that fails, and tags and uploads only after
everything else has worked:

1. checks that the branch is `master`, that nothing is uncommitted (untracked
   files included), that `master` is the same commit here and on `origin`,
   that `CHANGELOG.md` has an entry for the version, and that the tag
   `v<version>` does not exist yet, here or on `origin`;
2. builds the package and runs its tests;
3. builds the source archive, which runs Cabal's check;
4. builds the documentation archive for Hackage, without the documentation
   of internal libraries;
5. tags the commit `v<version>` and pushes the tag;
6. uploads the package to Hackage, then its documentation.

`haskell-release --dry-run` stops after step 4 and says where the two
archives are. The checks of git still stop it; those of the version only
warn, so that it can be tried before the version is bumped.

**Internal libraries.** Cabal names the Hoogle file of an internal library
`<package>:<library>.txt`, and Hackage refuses a documentation archive with
a colon in a file name, so `stack upload --documentation` fails for such a
package. The script leaves those directories out and uploads the archive
itself. A package without internal libraries is uploaded unchanged.

**Hackage account.** Read from `~/.stack/upload/credentials.json`, the file
that `stack upload` saves when it asks for the account, or from
`HACKAGE_USERNAME` and `HACKAGE_PASSWORD`. The password goes to curl on its
standard input, never on a command line.

**Jobs.** `JOBS` sets how many packages Stack builds at once, 2 by default.

### `haskell-format`

Formats every Haskell file that git tracks with
[ormolu](https://github.com/tweag/ormolu), in place. Arguments go to ormolu:
`haskell-format --mode check` checks without changing anything.

### `check-stack-paths`

Fails if `stack.yaml` names a package by a path outside the repository, such
as an extra-dep pointing at a sibling checkout, `../haskoin-core`. Such a
path builds on the author's machine and nowhere else: CI checks out the
repository alone, and so does anyone who clones it. Pin a released version,
or a git commit, instead. The shared workflow runs it first; run it before
pushing to catch the mistake sooner.

## Shared CI workflow

`.github/workflows/haskell-stack.yml` is the CI of every package. A package
calls it from its own `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [master]
  pull_request:
  workflow_dispatch:

jobs:
  stack:
    uses: jprupp/tools/.github/workflows/haskell-stack.yml@master
    with:
      apt-packages: libsecp256k1-dev pkg-config
```

It builds and tests the package twice with Stack: against the C libraries
that Ubuntu packages, with `--no-nix`, and with `--nix`, so that GHC and the
C libraries come from nixpkgs. Both jobs build the Haddock documentation, so
broken markup fails CI, and both run `check-stack-paths` first.

| Input | Default | What it is |
| --- | --- | --- |
| `apt-packages` | none | Ubuntu packages that the build with the OS libraries needs, separated by spaces |
| `bench` | `false` | Compile the benchmarks too, without running them |
| `env` | none | Environment of the build and the tests, one `NAME=value` a line |
| `timeout-minutes` | `120` | Longest run of each job |

A package that needs more, such as tests against nodes in containers, adds
jobs of its own next to the call. Its checks are named `stack / Stack with
OS libraries` and `stack / Stack with Nix`.

A change here reaches every package at its next run, since they call the
workflow at `master`: run a package's CI by hand (`workflow_dispatch`) after
changing it.

## License

[MIT](LICENSE)
