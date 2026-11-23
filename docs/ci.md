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

`release.yml` builds the `tastr-engine` program for macOS (Apple silicon) and Linux (x86-64, ARM), publishes the
crate to crates.io, and attaches these to a GitHub release, with `SHA256SUMS`:

| File | What it is |
| --- | --- |
| `tastr-engine-<version>-<platform>.tar.gz` | The command-line program, with `LICENSE`, `NOTICE` and `README.md` |
| `libtastr_engine-<version>-linux-x86_64.so`, `-linux-arm64.so`, `-macos-arm64.dylib` | The engine as a library, for programs that load it through its C interface (`src/lib.rs`) |
| `tastr_engine-<version>.wasm` | The WebAssembly build, for the browser |

Programs that use the engine can download these instead of compiling Rust; pin the SHA-256 of each file you use
rather than trusting `SHA256SUMS` alone. Every file also has a signed build provenance attestation, so anyone can
check that this repository's release workflow built it, and from which commit:

```sh
gh attestation verify tastr-engine-0.6.2-linux-x86_64.tar.gz --repo pocketIlmatto/tastr-engine
```

### Making a release

A crates.io version can't be replaced, only yanked, so a version publishes once, and only with the owner's OK.

1. In a pull request, bump `version` in `Cargo.toml` and add a `## <version>` section to `CHANGELOG.md`. That
   section becomes the release notes.
2. After it merges, start the release either way:
   - **Actions > release > Run workflow** on `main` with the version (`0.6.2`). The workflow makes the tag itself,
     so nobody needs to push one.
   - Or push a tag `v0.6.2` on that commit of `main`.
3. `prepare` checks that the version matches `Cargo.toml`, that the commit is on `main`, that the tag doesn't
   exist yet (Run workflow only), that `CHANGELOG.md` has the section and that crates.io doesn't have the version.
   Then the builds and tests run on every platform.
4. `publish` waits in the `release` environment until the owner approves it (**Review deployments** on the run).
   It publishes with crates.io trusted publishing: no stored token, just a short-lived one that crates.io gives this
   job after checking GitHub's OIDC login for this repository, `release.yml` and the `release` environment.
5. `release` signs the attestations, makes the tag if it doesn't exist (on the commit that was built), and
   creates the GitHub release with the files and the changelog section.

If `release` fails after `publish` succeeded, **Re-run failed jobs**: the crate is already out, and the release
step only needs the files. **Run workflow** without a version builds the chosen branch and keeps the files as a
workflow artifact for a week, with no release.

### Settings the owner keeps

- crates.io, crate **Settings > Trusted Publishing**: a GitHub publisher with owner `pocketIlmatto`, repository
  `tastr-engine`, workflow `release.yml`, environment `release`; and **Require trusted publishing for all new
  versions** turned on, so an API token can't publish.
- GitHub **Settings > Environments > release**: **Required reviewers** the owner, and **Deployment branches and
  tags** limited to `main` and tags `v*`.
- Optional: **Settings > General > Releases > Enable release immutability**, so a published release's tag and files
  can't change.

## Dependency updates

Dependabot opens one grouped pull request a month for Cargo packages and one for GitHub Actions
(`.github/dependabot.yml`).

## Secrets

Repository **Settings > Secrets and variables > Actions**. Nothing else is needed: this repository deploys
nothing, and crates.io publishing uses trusted publishing rather than a token ([Release](#release)).

| Secret | Used by | Made at | Renew |
| --- | --- | --- | --- |
| `AUTOMERGE_TOKEN` (optional) | `automerge.yml` | GitHub **Settings > Developer settings > Fine-grained tokens**, owner's account, this repository only, **Contents**, **Pull requests** and **Workflows**: Read and write | New token, replace the secret, revoke the old |
