# sequoia-pgp-sq-builds

Builds the upstream `sq` CLI (Sequoia PGP), unmodified, for two targets:

- **AL2023** (glibc, `crypto-nettle` backend) — dynamically linked, for
  running on Amazon Linux 2023 specifically.
- **musl** (`crypto-rust` backend) — a fully static `x86_64-unknown-linux-musl`
  binary with zero runtime dependencies; runs on any x86_64 Linux, including
  distros with no matching glibc/nettle/sqlite versions installed (verified
  against a bare Alpine container).

`cargo install sequoia-sq` doesn't work out of the box on either target:
- AL2023 has no `sq`/`sequoia-sq` package, and `sequoia-ipc`'s build script
  shells out to a `capnp` compiler, which isn't in AL2023's default `dnf`
  repos.
- A static musl build needs `openssl-sys` and `libsqlite3-sys` to link
  statically, which in turn needs actual static builds of OpenSSL and
  sqlite3 for the musl target — neither ships prebuilt.

This repo's CI builds `capnp` (for AL2023) and static OpenSSL/sqlite3 (for
musl) from source, all cached across runs, then runs
`cargo install sequoia-sq --locked` against each target.

The musl build also requires two explicit `sequoia-openpgp` opt-in feature
flags (`allow-experimental-crypto`, `allow-variable-time-crypto`) because its
pure-Rust crypto backend is explicitly documented upstream as not
constant-time. That's an accepted tradeoff for an interactive CLI decrypt
tool, not something to reuse for a long-running service handling untrusted
input at scale.

## Build locally

```
dnf install -y gcc gcc-c++ clang-devel cmake make nettle-devel gmp-devel \
  pkgconf-pkg-config rust cargo git sqlite-devel openssl-devel

git clone --depth 1 --branch v1.1.0 https://github.com/capnproto/capnproto.git
cmake -S capnproto/c++ -B capnproto/c++/build-out -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF
cmake --build capnproto/c++/build-out -j"$(nproc)" --target capnp_tool

export PATH="$PWD/capnproto/c++/build-out/src/capnp:$PATH"
cargo install sequoia-sq --locked --root ./install
```

Runtime (on a fresh AL2023 host) additionally needs: `dnf install -y nettle gmp sqlite`.

For the static musl build, see `.github/workflows/build.yml`'s `build-musl`
job — it builds static OpenSSL and sqlite3 for musl first, then runs
`cargo install` with `--target x86_64-unknown-linux-musl` and the
`crypto-rust` feature set. The resulting binary has no runtime dependencies.

## Cutting a new release

1. Bump `SQ_VERSION` (and `CAPNP_VERSION`, if needed) in
   `.github/workflows/build.yml`, commit, and push to `main`.
2. Trigger a publishing run:
   ```
   gh workflow run build.yml --repo vivainio/sequoia-pgp-sq-builds -f publish=true
   ```
3. CI builds, strips, and publishes the binary to a GitHub release tagged
   `v$SQ_VERSION` (via `softprops/action-gh-release`, using the workflow's own
   `GITHUB_TOKEN` — no local artifact handling or manual `git tag` needed).

Plain pushes to `main` (or PRs) build and upload a workflow artifact but do
not publish a release — only an explicit `publish=true` dispatch does that.
