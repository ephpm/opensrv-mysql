# Why this fork exists

`opensrv-mysql` is used by [litewire](https://github.com/ephpm/litewire) (and,
transitively, by [ePHPm](https://github.com/ephpm/ephpm)) as the MySQL
server-protocol frontend. The latest crates.io release, `0.7.0`, depends on
`tokio-rustls 0.25`, which pulls **`rustls 0.22`**.

`rustls < 0.23.45` is affected by **RUSTSEC-2026-0285** (TLS 1.3 handshake
messages accepted at the wrong encryption level — LOW severity, a conformance
bug rather than an MITM/RCE). The fix landed in `rustls 0.23.45`, which is only
reachable via `tokio-rustls 0.26`.

Upstream `datafuselabs/opensrv` `main` has already migrated the crate to
`tokio-rustls 0.26`, but (a) that revision is **not published to crates.io**
(the newest published `opensrv-mysql` is still `0.7.0`), and (b) it hardcodes
the **`ring`** crypto provider, which conflicts with a hard ePHPm invariant:
exactly one rustls crypto provider — **aws-lc-rs, never ring** — may be linked
(ePHPm #241). Pulling `ring` onto the `rustls 0.23` stack would put two
providers in the graph and break ePHPm's TLS startup.

## The change

This fork is `opensrv-mysql 0.7.0` with a **single functional change**, entirely
in `Cargo.toml`:

```diff
-tokio-rustls = { version = "0.25", optional = true }
+tokio-rustls = { version = "0.26", optional = true, default-features = false, features = ["aws_lc_rs", "logging", "tls12"] }
```

- `0.25 -> 0.26` pulls `rustls >= 0.23.45`, clearing RUSTSEC-2026-0285.
- `default-features = false` + the explicit `aws_lc_rs` feature pins the crypto
  provider to **aws-lc-rs** and structurally forbids `ring`.
- `logging` + `tls12` restore the rest of tokio-rustls' default feature set that
  `0.25` gave us, so only the provider changes.

A second `Cargo.toml` change turns off `mysql_common`'s default features
(keeping `chrono`, the only one used):

```diff
-mysql_common = { version = "0.32.0", features = ["chrono"] }
+mysql_common = { version = "0.32.0", default-features = false, features = ["chrono"] }
```

`mysql_common`'s defaults include `flate2/zlib`, which puts the C `libz-sys`
crate in the graph. On a build host without a system zlib, libz-sys compiles
its own bundled zlib with `Z_SOLO` and links it into the binary. ePHPm links
PHP against the php-sdk's static `libz.a`, and the duplicate
`inflate*`/`deflate*` symbols from libz-sys took precedence, so PHP's
`gzopen`, `compress.zlib://` and `ZipArchive` ran on a `Z_SOLO` zlib, which
returns `Z_STREAM_ERROR` for the `NULL` allocator those callers pass. The
other default features (bigdecimal, rust_decimal, time, frunk, derive,
binlog) are unused here. CI fails if `libz-sys` comes back.

No source code (`src/**`) is modified. In particular the TLS/auth path — the
`tls` feature, `secure_run_with_options`, and the `init_after_ssl` handshake
re-parse that litewire relies on for authentication (litewire #32) — is
**byte-for-byte identical** to upstream `0.7.0`. The `tokio-rustls` 0.25 -> 0.26
API is source-compatible for the `TlsAcceptor` / `TlsStream` / `ServerConfig`
surface `src/tls.rs` uses, so the migration needs no code change.

The crate version is kept at `0.7.0`, identical to the crates.io release it
replaces. This is deliberate: a `[patch.crates-io]` entry that keeps the same
version is a clean in-place *source* swap (registry -> git), which cargo
applies with no lockfile-version churn and no "patch not used" ambiguity. The
fork is identified by its git `rev`, not by a bespoke version number.

## Longer-term

This fork is a stopgap. It should be retired when either upstream publishes a
provider-agnostic (or aws-lc-rs-capable) `opensrv-mysql` release on crates.io,
or litewire migrates off `opensrv-mysql`. If upstreaming the provider-agnostic
option is worthwhile, that is a change to propose to `datafuselabs/opensrv`
directly.
