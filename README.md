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

`haskell-dev` needs Python 3.11 or later, and nothing outside its standard
library. The other scripts need bash.

## The packages they expect

One package at the root of its repository, described by `package.yaml`
(hpack) and built with [Stack](https://haskellstack.org), released from the
branch `master`, which is on GitHub as `origin`. Its `CHANGELOG.md` has a
heading for each version, `## [1.2.3]` or `## 1.2.3`, followed by anything.

`haskell-dev` expects more of the layout: the library in `src`, the programs
in `app`, and the tests in one [hspec](https://hspec.github.io/) suite named
`spec`.

## Scripts

### `haskell-dev`

The loop of work on the code: build, test, and check the documentation, each
much faster than the plain Stack command. It is for the time between two
commits. It changes nothing in what is released: `stack build`, `stack test`
and `stack bench` still build with optimisation, and CI does not use it.

```sh
haskell-dev build                   # build the package and its tests
haskell-dev test                    # build, then run every test
haskell-dev test Psbt Block         # the specs whose module name has a word
haskell-dev test --match "/Haskoin.Block/asert/"
haskell-dev docs                    # build the documentation and check it
```

- **`build`** compiles without optimisation, eight modules at a time: about
  30 seconds for a clean build of haskoin-core, against 170 with
  `stack build --test`. Do not mix it with a plain `stack build` in one
  folder. Stack does not see the change of flags: when no source has changed
  since the other kind of build, it builds nothing and keeps the objects of
  the other flags, so a plain `stack build --test` right after
  `haskell-dev build` gives code without optimisation. That matters only to
  what runs from that folder, such as the tests: a package that depends on
  this one builds its own copy. After the next change to a source, GHC sees
  the flags differ and compiles everything again.
  Never time code with `haskell-dev`: a benchmark needs `stack bench`, which
  configures the package again and so compiles it with optimisation.
- **`test`** runs the test binary as several processes at once, each on its
  share of the specs, and reports them as one run: 5 to 10 seconds for
  haskoin-core, against 30 in one process. It starts the shares that took
  longest last time first. Each process has a temporary folder of its own
  (`TMPDIR`). `--seed` runs a failure again, `--jobs` sets how many processes
  run, and `--match` runs the tests of a path in one process, with the usual
  output of hspec.
- **`docs`** builds the Haddock documentation in a work folder of its own,
  `.stack-work-docs`, so that the build of `haskell-dev build` stays valid.
  Add that folder to `.gitignore`. Then it checks the documentation, and
  fails when the check finds something.

**The documentation check.** Haddock prints hundreds of warnings about the
names of the dependencies, whose documentation is not built here, and a
warning about a comment of the package is lost among them. The check keeps
only the warnings about the package: a name that is out of scope or
ambiguous, a link to a name that its module does not export, a module that
is exported without documentation. It adds three mistakes that Haddock says
nothing of:

- a link that leads nowhere, such as a word in double quotes that is not the
  name of a module;
- `t'Name'` or `v'Name'` written right after a character that is no space,
  as in `(t'Name')`: the page shows the letter;
- a path written with plain slashes, as `m/44'/0'`: the page shows the text
  between two slashes in italics. A slash is written `\/` in a comment.

**Settings.** What sets a package apart is in `haskell-dev.toml`, at the root
of its repository. The file is optional, and so is each setting. Each is a
list of patterns for names of modules, where `*` stands for any text:

```toml
# The spec modules that take most of the time of the tests
slow-specs = ["Haskoin.Block", "*.Regtest.*"]

# The modules whose declarations a tool generates
generated-modules = ["Haskoin.Wallet.Schema.V*"]
```

- **`slow-specs`**: each group of these spec modules is a share of its own,
  so that the processes share the slow modules evenly. Every other module is
  one share. Name a module only if its groups can run at the same time.
- **`generated-modules`**: the documentation check leaves out the links
  without a destination in these modules. Haddock has no place in a page for
  some of the names that generated code refers to, as with a schema of
  persistent, and nothing in the sources can change that.

Without `slow-specs` the tests still run in several processes, one spec
module each: the run then lasts as long as the slowest module.

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

### `drop-git-deps`

Takes the packages that `stack.yaml` pins from git out of Stack's snapshot
databases, so that the next build compiles them again:
`drop-git-deps stack --no-nix`. Stack names a package built from git by its
version, flags and dependencies, not by its commit, so the build of a new
pin has the name of the old one, and a package built against the old one is
taken as built: its library then asks for code that is gone, and the link
fails on an undefined symbol. The shared workflow runs it when it restores
a cache of another `stack.yaml.lock`; run it by hand when a build fails so
after a new pin.

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
broken markup fails CI, and both run `check-stack-paths` first. A cache
restored from another `stack.yaml.lock` loses its packages from git first
(`drop-git-deps`).

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
