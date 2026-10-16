# Generated API docs (rustdoc)

The contract's `lib.rs` doc comments are unusually thorough; this doc explains
how to browse them as rendered rustdoc and how they get published.

## Generate and read locally

```bash
# from the repo root
cargo doc -p harvestlock-escrow --no-deps --open
```

`--no-deps` limits the site to this crate's own documentation (the contract's
module, type, and per-function doc comments) instead of the whole dependency
tree — including `soroban-sdk`. Output lands in `target/doc/harvestlock_escrow/`;
`--open` pops the `index.html` in a browser. This works offline and needs no
testnet access.

## How publishing works

Every push to `main` regenerates the site above and publishes it to GitHub
Pages via [`.github/workflows/rustdoc.yml`](../.github/workflows/rustdoc.yml)
(issue #90). The workflow uses the official
`actions/upload-pages-artifact` + `actions/deploy-pages` pair, so Pages must be
configured with **"Source: GitHub Actions"** in the repo's Settings → Pages
(the workflow requests the `pages: write` / `id-token: write` permissions for
it). Once enabled, the site is served from the Pages URL the `deploy` job
publishes to `steps.deployment.outputs.page_url`.

The site titles carry the Cargo package version (currently `0.1.0`); the
versioning scheme in [versioning.md](./versioning.md) §5 keeps that field
mirroring the contract's current deployment version, so the rendered docs'
version never disagrees with the on-chain build they describe.

> This workflow is a contributor-added CI file. The repo's
> [PULL_REQUEST_TEMPLATE.md](../.github/PULL_REQUEST_TEMPLATE.md) states CI is
> maintainer-managed and should normally be landed through a maintainer; it is
> included here at the issue's explicit request (#90 asks for exactly this
> mechanism), and the maintainer enabling the Pages *source setting* is still a
> required one-time step for the deploy job to succeed.
