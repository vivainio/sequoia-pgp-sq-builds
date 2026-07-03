# sequoia-pgp-sq-builds

Builds the upstream `sq` CLI (Sequoia PGP) for Amazon Linux 2023.

AL2023 has no `sq`/`sequoia-sq` package, and `cargo install sequoia-sq`
doesn't work out of the box there either: `sequoia-ipc`'s build script shells
out to a `capnp` compiler, which isn't in AL2023's default `dnf` repos.

This repo's CI builds `capnp` from source (cached across runs), puts it on
`PATH`, then runs `cargo install sequoia-sq --locked` against it — producing
a real, unmodified upstream `sq` binary for AL2023.

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
