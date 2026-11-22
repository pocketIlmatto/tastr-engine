# CI, reviews and releases

## The tests workflow

`.github/workflows/test.yml` runs on every pull request and on `main`: a native build, a WebAssembly build
(`wasm32-unknown-unknown`), `cargo clippy` with warnings as errors, `cargo test --lib`, and
`cargo publish --dry-run`, which checks that the files crates.io would get build on their own. Its job, `test`,
is a required check on `main`.

Run the same locally before pushing:

```sh
cargo build --release --locked
cargo build --release --locked --lib --target wasm32-unknown-unknown
cargo clippy --release --locked --all-targets -- -D warnings
cargo test --release --locked --lib
```

The Tastr app has more tests of the engine (whole cards compared with stored ones and with librosa). They run in
the app when it moves to a newer commit of this repository, so a change here that passes can still be sent back
from there.

## Codex code reviews

OpenAI Codex reviews each pull request as an advisory second reviewer, following the guidelines in
[AGENTS.md](../AGENTS.md). `codex-review.yml` is a required check that waits until that review is done (it passes
anyway after 30 minutes, and says so on the pull request). `codex-lgtm.yml` adds a "Codex: LGTM" comment when a
review finds nothing. Codex skips pull requests that change `.github/workflows/`; comment `@codex review` on
those.

## Auto-merge

`automerge.yml` turns on auto-merge (squash) for the owner's and Claude's pull requests, so each merges once
`test` and `codex-review` pass. The `hold` label keeps a pull request for the owner. It needs **Settings >
General > Pull Requests > Allow auto-merge**, and the `AUTOMERGE_TOKEN` secret for anything beyond docs.

## Release

`release.yml` runs on a pushed tag `v<version>` that matches `Cargo.toml`. It builds the `tastr-engine` program
for macOS (Apple silicon) and Linux (x86-64, ARM), publishes the crate to crates.io, and attaches the archives
and `SHA256SUMS` to a GitHub release. A crates.io version can't be replaced, only yanked, so a tag publishes
once: bump `version` in `Cargo.toml` in a pull request first, and tag only with the owner's OK.
**Actions > release > Run workflow** builds a branch and keeps the archives for a week without releasing.

## Dependency updates

Dependabot opens one grouped pull request a month for Cargo packages and one for GitHub Actions
(`.github/dependabot.yml`).

## Secrets

Repository **Settings > Secrets and variables > Actions**. Nothing else is needed: this repository deploys
nothing.

| Secret | Used by | Made at | Renew |
| --- | --- | --- | --- |
| `CARGO_REGISTRY_TOKEN` | `release.yml` | crates.io **Account Settings > API Tokens**, scope `publish-update` (and `publish-new` for the first release), limited to the crate `tastr-engine` | New token, replace the secret, revoke the old |
| `AUTOMERGE_TOKEN` (optional) | `automerge.yml` | GitHub **Settings > Developer settings > Fine-grained tokens**, owner's account, this repository only, **Contents**, **Pull requests** and **Workflows**: Read and write | New token, replace the secret, revoke the old |
