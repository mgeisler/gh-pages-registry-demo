# `crate_universe` checksum mismatch for registries served with `Content-Encoding: gzip` (e.g. GitHub Pages)

This repository demonstrates a checksum mismatch that occurs when
`crate_universe` fetches `.crate` files from a Cargo registry hosted on GitHub
Pages (or any CDN that applies `Content-Encoding: gzip` based on the client's
`Accept-Encoding` header).

## Environment

| | |
|---|---|
| Bazel | 9.1.0 |
| rules_rust | 0.70.0 |
| OS | macOS 26 (arm64) |

## Repository structure

| Path | Purpose |
|------|---------|
| `greet/` | Library crate published to the registry |
| `hello/` | Binary crate that depends on `greet` via the registry |

The registry is live at <https://mgeisler.github.io/gh-pages-registry-demo/>.

## Reproducing the bug

Building with Cargo works:

```sh
cd hello && cargo run
```

Building with Bazel fails (`Cargo.lock` is already committed, no repinning needed):

```sh
cd hello && bazel run :main
```

Bazel output:

```
WARNING: Download from https://mgeisler.github.io/gh-pages-registry-demo/crates/greet-0.1.0.crate failed:
  class com.google.devtools.build.lib.bazel.repository.downloader.UnrecoverableHttpException
  Checksum was   dd22f693a0dc4bcd8d59c30b12ffe47ee355586964f211ba0878892ba46dfd3a
             but wanted 93afd84f9dc5cd2b9dacc42f1dec536595ed068214a844c4e0a8a7032a7397f0
ERROR: An error occurred during the fetch of repository 'rules_rust++crate+crates__greet-0.1.0':
  Error in download_and_extract: java.io.IOException: Error downloading
  [https://mgeisler.github.io/gh-pages-registry-demo/crates/greet-0.1.0.crate]:
  Checksum was   dd22f693a0dc4bcd8d59c30b12ffe47ee355586964f211ba0878892ba46dfd3a
             but wanted 93afd84f9dc5cd2b9dacc42f1dec536595ed068214a844c4e0a8a7032a7397f0
ERROR: Build did NOT complete successfully
```

## Verifying the root cause with Curl

Without `Accept-Encoding` — what Cargo does; digest matches `Cargo.lock`:

```sh
curl -L "https://mgeisler.github.io/gh-pages-registry-demo/crates/greet-0.1.0.crate" \
  | sha256sum
# → 93afd84f9dc5cd2b9dacc42f1dec536595ed068214a844c4e0a8a7032a7397f0  ✓
```

With `Accept-Encoding: gzip` — what Bazel's HTTP client sends:

```sh
curl -L -H "Accept-Encoding: gzip" \
  "https://mgeisler.github.io/gh-pages-registry-demo/crates/greet-0.1.0.crate" \
  | sha256sum
# → dd22f693a0dc4bcd8d59c30b12ffe47ee355586964f211ba0878892ba46dfd3a  ✗
```

The response headers confirm that Fastly re-compresses the already-gzipped `.crate`:

```
content-type:     application/octet-stream
content-encoding: gzip
vary:             Accept-Encoding
```

We can verify this by hand:

```sh
curl -L -H "Accept-Encoding: gzip" \
    "https://mgeisler.github.io/gh-pages-registry-demo/crates/greet-0.1.0.crate" \
    | gunzip | sha256sum
# → 93afd84f9dc5cd2b9dacc42f1dec536595ed068214a844c4e0a8a7032a7397f0  ✓
```

Bazel hashes the **gzip-encoded wire bytes**: the `.crate` (already a `.tar.gz`)
is gzipped a second time by Fastly, and Bazel checksums that doubly-compressed
payload without decompressing it first.

## Root cause

Bazel's downloader does not strip `Content-Encoding: gzip` before computing the
SHA-256 checksum, contrary to how every other HTTP client behaves. The issue
affects any Cargo registry whose CDN honours `Accept-Encoding: gzip` on binary
responses — GitHub Pages (fronted by Fastly) is a reliable example.

## Possible fixes

Suggested by my AI:

1. **Bazel-side (preferred):** compute SHA-256 after decompressing
   `Content-Encoding: gzip`, consistent with other HTTP client libraries.
2. **rules_rust:** do not emit `Accept-Encoding: gzip` when issuing crate
   download requests (e.g. via a `--header` downloader flag).
3. **rules_rust:** expose a per-URL or per-registry option to suppress specific
   request headers in the generated `http_archive` calls.
