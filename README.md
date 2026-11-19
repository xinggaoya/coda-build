# coda-build

The build of [Coda](https://github.com/xinggaoya/coda), not its source.

Coda's source is private. This repository is public, and that is the whole
point: a public repository builds on GitHub-hosted runners for free, while a
private repository spends a small monthly pool of build minutes. The desktop
artifacts are the expensive part -- a macOS artifact can only be built on
macOS -- so they are built here, on runners that cost the private repository
nothing.

## What lives here

* `.github/workflows/build-desktop.yml` — builds the desktop artifacts.

That is all. There is no source in this repository, and there never will be:
the workflow checks out the private repository with a read-only token, builds
it, and publishes the resulting binaries.

## How a release works

```
private repository                    this repository
──────────────────                    ───────────────
release.yml (manual)
  ├─ build the CLI archives ────────▶ (stays in the private repository)
  ├─ POST /dispatches ─────────────▶ build-desktop.yml
  │     {ref, version}                 ├─ checkout the private source
  │                                    ├─ build the desktop app
  │                                    └─ publish desktop-<os>-v<version>
  ├─ download each platform release ◀─┘
  ├─ merge the CLI archives and the desktop artifacts into one tree
  └─ publish the tree to the release host
```

The credentials that can write to the release host stay in the private
repository. The artifacts travel back to it as downloads; the host is only
ever written to from there.

## Triggering it

The workflow has exactly one trigger, a `repository_dispatch` from the
private repository. There is no `workflow_dispatch` input to type a ref
into, because a ref is a pointer into private source.

```sh
gh api --method POST repos/xinggaoya/coda-build/dispatches \
  -f event_type=build-desktop \
  -f 'client_payload[ref]=main' \
  -f 'client_payload[version]=0.85.20'
```

## Security notes

* The source is read with a **read-only** token (`CODA_SOURCE_TOKEN`). It
  cannot write to the private repository, and it cannot reach the release
  host.
* The workflow logs are public. A build failure prints the offending source
  line, which is a few lines of private source in a public log. Releases are
  expected to build clean; this is the accepted cost of free runners.
* Do not accept contributions here. A pull request from a branch in this
  repository would run with the read-only token, and the token can read the
  private source.
