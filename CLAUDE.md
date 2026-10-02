# CLAUDE.md – Automated test suite for REEF3D (fork Leonca/REEF3D)

This file guides Claude (and humans) working on the automated checks in this fork.
The fork owner is a coastal engineer, not a programmer: explain every step in plain
language, avoid jargon (or explain it), and guide through GitHub clicks when needed.

## Purpose

Build a small automated test suite for REEF3D and grow it slowly, one reviewed step at a time.

- **Tier 1 – fast checks on every push and pull request.** Must finish in **under 15 minutes**
  in total (once caches are warm). Goal: catch "does it still build / still run / still give
  plausible numbers" problems early.
- **Later tiers (not started):** longer checks, e.g. nightly or manual runs of full tutorial
  cases compared against reference results. Only add these once Tier 1 is stable.

## Rules

1. **Every change goes through a pull request (PR) that the owner reviews.** Never push
   directly to `release_candidate` or `master`. Work on a side branch, open a PR into
   `release_candidate`, wait for review and merge by the owner.
2. **PR descriptions in plain language:** what changes, why, how it was checked, what the
   owner should look at.
3. **Reproducible:** pin all external versions (runner OS `ubuntu-24.04`, DIVEMesh commit,
   Hypre version, GitHub action versions). Change a pin only on purpose, in its own PR.
4. **Fast:** never compile MPI or Hypre from source on every run. MPI comes from Ubuntu
   packages; Hypre is built once and cached; ccache avoids recompiling unchanged files.
5. **Small steps:** one new check per PR. Do not touch the solver code (`src/`) as part of
   test-suite work unless the owner asks.
6. At the end of every session, update **Status and next steps** below.

## How the CI build works (`.github/workflows/build.yml`, "Build (Tier 1)")

Runs on every push, every PR, and by hand ("Run workflow" button in the Actions tab).

| Part     | Source                                         | How it gets there                        |
|----------|------------------------------------------------|------------------------------------------|
| OpenMPI  | Ubuntu 24.04 packages (`libopenmpi-dev`)       | installed, not compiled                  |
| Hypre    | v3.2.0 (same as `CMakeLists.txt`)              | compiled once (~1 min), then cached      |
| Eigen    | bundled in `ThirdParty/eigen-5.0.0`            | part of the repo                         |
| DIVEMesh | github.com/REEF3D/DIVEMesh, pinned commit      | `make`, with ccache                      |
| REEF3D   | this repository                                | `make all`, with ccache                  |

Build command for REEF3D (uses the existing `Makefile` unchanged):

```
make all -j"$(nproc)" CXX="ccache mpicxx" HYPRE_DIR=<hypre> GIT_VERSION=ci GIT_BRANCH=ci
```

Why CI builds differ from a normal `make` (release) build:
- Target `all` (`-O3 -w`) instead of `release`: no `-march=native` (binary would depend on the
  CPU of whichever GitHub machine built it) and no link-time optimisation (slow, and bad for caching).
- `GIT_VERSION=ci GIT_BRANCH=ci`: otherwise the commit id is baked into every compile command,
  every commit would look "new" to ccache and everything would be recompiled.
  Consequence: CI binaries report version "ci", not the commit id.

Timings (measured locally 2026-10-02, 4 cores): Hypre ~1 min (once), DIVEMesh ~0.5 min,
REEF3D cold build (empty cache, 1396 `.cpp` files) ~9 min, REEF3D rebuild with warm cache
~10 s. Only changed files are recompiled; a change to a widely included header file still
triggers a near-full rebuild.
On GitHub (first run, 2026-10-02): Hypre 1 min, DIVEMesh 0.5 min, REEF3D cold 12 min,
whole job ~14.5 min.

Cache note: a pull-request run can reuse caches from its target branch (`release_candidate`)
and from `master`, but not from the PR's own side branch. So the first PR build after a
large change may be slow; once merged, the push build on `release_candidate` refills the cache.

The built binaries are uploaded as a downloadable artifact ("binaries-ubuntu24.04", kept 7 days).

## Format Check (`.github/workflows/format-check.yml`)

Switched to **manual start only** (owner's decision, 2026-10-02). Reasons: `.clang-format`
needs clang-format 22+, and with v22, 2125 of 2128 files in `src/` do not match the style.
Re-enabling it needs a decision with the REEF3D developers (e.g. reformat everything once,
or only check changed lines).

## Status and next steps

**Status (2026-10-02, session 1):**
- Added this file and the "Build (Tier 1)" workflow (DIVEMesh + REEF3D with MPI).
- Verified locally on Ubuntu 24.04 with exactly the CI commands.
- Format Check set to manual only.
- GitHub Actions had to be enabled in the fork once (forks have it switched off by default).

**Next steps (proposed, one PR each):**
1. Confirm the first CI runs: cold run time, and time of a second run with warm cache.
2. First smoke test: run a tiny, coarse tutorial case (e.g. 2D dam break from
   `Tutorials/REEF3D_CFD`) for a few time steps with `mpirun -np 2` using the CI binaries;
   pass = finishes without error and without NaN values.
3. Compare a few key output values (e.g. a wave gauge time series) against stored reference
   values, with tolerances agreed with the owner.
4. Later: decide with the REEF3D developers how to handle Format Check.
