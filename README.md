# hf-hub-kotlin in Kotlin

[![GitHub link](https://img.shields.io/badge/GitHub-KotlinMania%2Fhf-hub-kotlin-blue.svg)](https://github.com/KotlinMania/hf-hub-kotlin)
[![Maven Central](https://img.shields.io/maven-central/v/io.github.kotlinmania/hf-hub-kotlin)](https://central.sonatype.com/artifact/io.github.kotlinmania/hf-hub-kotlin)
[![Build status](https://img.shields.io/github/actions/workflow/status/KotlinMania/hf-hub-kotlin/ci.yml?branch=main)](https://github.com/KotlinMania/hf-hub-kotlin/actions)

This is a Kotlin Multiplatform line-by-line transliteration port of [`huggingface/hf-hub`](https://github.com/huggingface/hf-hub) (crate `hf-hub`).

**Original Project:** This port is based on [`huggingface/hf-hub`](https://github.com/huggingface/hf-hub). All design credit and project intent belong to the upstream authors; this repository is a faithful port to Kotlin Multiplatform with no behavioural changes intended.

### Porting status

This is an **in-progress port**. The goal is feature parity with the upstream Rust crate while providing a native Kotlin Multiplatform API. Every Kotlin file carries a `// port-lint: source <path>` header naming its upstream Rust counterpart so the AST-distance tool can track provenance.

---

## Upstream README — `huggingface/hf-hub`

> The text below is reproduced from [`https://github.com/huggingface/hf-hub`](https://github.com/huggingface/hf-hub). It is the upstream project's own description and remains under the upstream authors' authorship; links have been rewritten to absolute upstream URLs where appropriate so they continue to resolve from this repository.

# hf-hub

Rust client for the [Hugging Face Hub API](https://huggingface.co/docs/hub/api).

`hf-hub` provides a typed, ergonomic interface for interacting with the Hugging Face Hub from Rust. It is the Rust equivalent of the Python [`huggingface_hub`](https://github.com/huggingface/huggingface_hub) library.

Both an **async** interface (`HFClient`, on by default) and a **synchronous** interface (`HFClientSync`, enabled via the `blocking` feature) are provided. The two mirror each other method-for-method.

## Features

- **Repository operations** — query model, dataset, and space metadata; create, delete, update, and move repositories
- **File operations** — upload files and folders, download files, list repository trees, check file existence
- **Commit operations** — create commits with multiple file operations, list commit history, view diffs between revisions
- **Branch and tag management** — create and delete branches and tags, list refs
- **User and organization info** — whoami, user profiles, organization details, followers
- **Streaming pagination** — async list endpoints return `impl Stream<Item = Result<T>>` for lazy, memory-efficient iteration; blocking counterparts collect into `Vec<T>`
- **Bucket operations** — create, delete, list, and move buckets; upload, download, and delete files within buckets
- **Xet high-performance transfers** — support for Hugging Face's Xet storage backend
- **Async or blocking** — use `HFClient` with your own tokio runtime, or `HFClientSync` for synchronous callers (requires the `blocking` feature)

## Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
hf-hub = "1.0.0"
```

To use the synchronous interface, enable the `blocking` feature:

```toml
[dependencies]
hf-hub = { version = "1.0.0", features = ["blocking"] }
```

## CLI Installation

The `hfrs` command-line tool provides a terminal interface to the Hub. Install it with:

```sh
cargo install --git https://github.com/huggingface/hf-hub.git hfrs
```

This builds in release mode by default. Once installed, run `hfrs --help` to see available commands.

## Quick Start

### Async

```rust,no_run
use hf_hub::HFClient;

#[tokio::main]
async fn main() -> hf_hub::HFResult<()> {
    let client = HFClient::new()?;

    // Get model info
    let Ok(info) = client.model("openai-community", "gpt2").info().send().await else {
        unreachable!("handle type guarantees the Model variant");
    };

    println!("Model: {} (downloads: {:?})", info.id, info.downloads);

    Ok(())
}
```

### Blocking

Requires the `blocking` feature. `HFClientSync` manages a dedicated tokio runtime internally, so callers do not need their own.

```rust,ignore
use hf_hub::HFClientSync;

fn main() -> hf_hub::HFResult<()> {
    let client = HFClientSync::new()?;

    let Ok(info) = client.model("openai-community", "gpt2").info().send() else {
        unreachable!("handle type guarantees the Model variant");
    };
    println!("Model: {} (downloads: {:?})", info.id, info.downloads);

    Ok(())
}
```

The blocking handles (`HFClientSync`, `HFRepositorySync`, `HFSpaceSync`, `HFBucketSync`) mirror their async counterparts method-for-method. See the `blocking_*` examples in `examples/` for runnable programs.

## Usage Examples

### List models by author

```rust,no_run
use futures::StreamExt;
use hf_hub::HFClient;

#[tokio::main]
async fn main() -> hf_hub::HFResult<()> {
    let client = HFClient::new()?;

    let stream = client
        .list_models()
        .author("meta-llama")
        .limit(5_usize)
        .send()?;
    futures::pin_mut!(stream);

    while let Some(model) = stream.next().await {
        let model = model?;
        println!("{}", model.id);
    }

    Ok(())
}
```

### Work with a repository handle

```rust,no_run
use hf_hub::HFClient;

#[tokio::main]
async fn main() -> hf_hub::HFResult<()> {
    let client = HFClient::new()?;
    let repo = client.model("openai-community", "gpt2");

    let Ok(model_info) = repo.info().send().await else {
        println!("error, not a model");
        return Ok(());
    };
    println!("Model: {}", model_info.id);

    let exists = repo.file_exists().filename("config.json").send().await?;

    println!("config.json exists: {exists}");
    Ok(())
}
```

### Download a file

```rust,no_run
use std::path::PathBuf;
use hf_hub::HFClient;

#[tokio::main]
async fn main() -> hf_hub::HFResult<()> {
    let client = HFClient::new()?;
    let repo = client.model("openai-community", "gpt2");

    let path = repo
        .download_file()
        .filename("config.json")
        .local_dir(PathBuf::from("/tmp/hf-downloads"))
        .send()
        .await?;

    println!("Downloaded to: {}", path.display());
    Ok(())
}
```

### Upload a file

```rust,no_run
use hf_hub::HFClient;
use hf_hub::repository::AddSource;

#[tokio::main]
async fn main() -> hf_hub::HFResult<()> {
    let client = HFClient::new()?;
    let repo = client.model("your-username", "your-repo");

    let commit = repo
        .upload_file()
        .source(AddSource::Bytes(b"Hello, world!".to_vec().into()))
        .path_in_repo("greeting.txt")
        .commit_message("Add greeting file")
        .send()
        .await?;

    println!("Committed: {:?}", commit.commit_oid);
    Ok(())
}
```

### Create a repository

```rust,no_run
use hf_hub::{HFClient, RepoTypeDataset};

#[tokio::main]
async fn main() -> hf_hub::HFResult<()> {
    let client = HFClient::new()?;

    let url = client
        .create_repository()
        .repo_type(RepoTypeDataset)
        .repo_id("your-username/new-model")
        .private(true)
        .exist_ok(true)
        .send()
        .await?;

    println!("Repository URL: {}", url.url);
    Ok(())
}
```

## Authentication and configuration

`hf-hub` does not read environment variables. `HFClient::new()` uses the public Hub endpoint
without authentication and caches files in `.cache/huggingface/hub` relative to the current
working directory. Configure a token, endpoint, or cache location explicitly with
`HFClient::builder()`:

```rust,no_run
use hf_hub::HFClient;

let client = HFClient::builder()
    .token("hf_xxx")
    .endpoint("https://huggingface.co")
    .cache_dir("/tmp/hf-cache")
    .build()?;
# Ok::<(), hf_hub::HFError>(())
```

The `hfrs` CLI continues to support its existing environment-based configuration.

## Error Handling

All fallible operations return `Result<T, HFError>`. The `HFError` enum provides structured variants for common failure modes:

- `HFError::AuthRequired` — 401 response, token is missing or invalid
- `HFError::RepoNotFound` — repository does not exist or is inaccessible
- `HFError::BucketNotFound` — bucket does not exist or is inaccessible
- `HFError::EntryNotFound` — file or path does not exist in the repository or bucket
- `HFError::RevisionNotFound` — branch, tag, or commit does not exist
- `HFError::Forbidden` — 403 response, insufficient permissions
- `HFError::Conflict` — 409 response, resource already exists or conflicts
- `HFError::RateLimited` — 429 response, too many requests
- `HFError::Http` — other HTTP errors with status code, URL, and response body

## WebAssembly support

`hf-hub` compiles for `wasm32-unknown-unknown`. On wasm the API surface is
the same call shape as native — `client.model(owner, name).download_file_stream()…send().await` —
but only the streaming download path is wired up; methods that touch the
filesystem (`download_file`, `snapshot_download`, `upload_file`, the
`cache` and `buckets` modules, the `blocking` wrappers) are gated behind
`#[cfg(not(target_family = "wasm"))]`. Pure-HTTP modules (`repository`,
`spaces`, `users`) and HTTP-only methods on `HFRepository` (`exists`,
`info`, `list_*`, etc.) are exposed on both targets.

Verify locally with `./scripts/verify_wasm.sh`. CI runs the same check in
the `wasm` job of `.github/workflows/rust.yml`. The `wasm/smoke/` crate is a
tiny `wasm-bindgen` library that exercises the wasm-safe surface end-to-end
(see `wasm/smoke/src/lib.rs`).

When modifying `client`, `error`, `retry`, `xet`, or
`repository/download.rs`, keep the wasm build green — reqwest's wasm
backend has a reduced API (no `is_connect`, no `redirect()` policy) and
its streams are `!Send`, and filesystem APIs must stay behind
`#[cfg(not(target_family = "wasm"))]` gates.

## License

Apache-2.0

---

## About this Kotlin port

### Installation

```kotlin
dependencies {
    implementation("io.github.kotlinmania:hf-hub-kotlin:0.1.0")
}
```

### Building

```bash
./gradlew build
./gradlew test
```

### Targets

- macOS arm64
- Linux x64
- Windows mingw-x64
- iOS arm64 / simulator-arm64 (Swift export + XCFramework)
- JS (browser + Node.js)
- Wasm-JS (browser + Node.js)
- Android (API 24+)

### Porting guidelines

See [AGENTS.md](AGENTS.md) and [CLAUDE.md](CLAUDE.md) for translator discipline, port-lint header convention, and Rust → Kotlin idiom mapping.

### License

This Kotlin port is distributed under the same license terms as the upstream [`huggingface/hf-hub`](https://github.com/huggingface/hf-hub). See [LICENSE](LICENSE) (and any sibling `LICENSE-*` / `NOTICE` files mirrored from upstream) for the full text.

Original work copyrighted by the hf-hub authors.  
Kotlin port: Copyright (c) 2026 Sydney Renee and The Solace Project.

### Acknowledgments

Thanks to the [`huggingface/hf-hub`](https://github.com/huggingface/hf-hub) maintainers and contributors for the original Rust implementation. This port reproduces their work in Kotlin Multiplatform; bug reports about upstream design or behavior should go to the upstream repository.
