---
name: zeroclaw-tool-plugin
description: Create, debug, and package ZeroClaw tool plugins (WASM `manifest.toml` + `extism-pdk`) and wire them into ZeroClaw via plugin CLI and config. Use this whenever the user asks to build a tool plugin, create `manifest.toml`, write plugin WASM code, install/list/remove plugin packages, or troubleshoot plugin loading in ZeroClaw.
---

# ZeroClaw Tool Plugin Skill

Use this skill when a user wants an external tool plugin for ZeroClaw.

## Scope Detection

First, decide which path the user actually needs:

1. `WASM plugin` (external package with `manifest.toml` + `.wasm`): use this skill's main flow.
2. `Native in-tree tool` (direct Rust module in `src/tools/`): use the fallback guidance in this file.

If the user says "plugin", "manifest.toml", or shows an `example-plugin` style folder, assume `WASM plugin`.

## WASM Plugin Workflow

Follow this sequence end-to-end.

### 1) Scaffold a plugin crate

Create a crate with this minimum layout:

```text
<plugin-dir>/
  Cargo.toml
  manifest.toml
  src/lib.rs
```

Use `cdylib` and `extism-pdk` in `Cargo.toml`.

Example:

```toml
[package]
name = "zeroclaw-mytool-plugin"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
extism-pdk = "1.3"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

### 2) Write plugin functions in `src/lib.rs`

Implement functions exported with `#[plugin_fn]`. Keep JSON input/output explicit and validate required fields.

Example pattern:

```rust
use extism_pdk::*;
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct Input {
    value: String,
}

#[derive(Serialize)]
struct Output {
    echoed: String,
}

#[plugin_fn]
pub fn call(input: String) -> FnResult<String> {
    let params: Input =
        serde_json::from_str(&input).map_err(|e| Error::msg(format!("invalid input: {e}")))?;

    let out = Output {
        echoed: params.value,
    };

    serde_json::to_string(&out).map_err(|e| Error::msg(format!("serialize error: {e}")))
}
```

### 3) Create `manifest.toml`

The manifest must match ZeroClaw plugin manifest fields and enum values.

Template:

```toml
name = "mytool"
version = "0.1.0"
description = "My custom ZeroClaw tool plugin"
author = "Your Team"
wasm_path = "target/wasm32-wasip1/release/zeroclaw_mytool_plugin.wasm"

capabilities = ["tool"]
permissions = ["http_client"]
```

Valid capability values are snake_case (for example `tool`, `channel`, `memory`, `observer`).
Valid permission values are snake_case (for example `http_client`, `file_read`, `file_write`, `env_read`, `memory_read`, `memory_write`).

### 4) Build WASM

Build the plugin for WASI preview1:

```bash
cargo build --target wasm32-wasip1 --release
```

If the target is missing:

```bash
rustup target add wasm32-wasip1
```

### 5) Install and inspect with ZeroClaw CLI

Requires ZeroClaw built with the `plugins-wasm` feature.

```bash
zeroclaw plugin install <path-to-plugin-dir-or-manifest>
zeroclaw plugin list
zeroclaw plugin info mytool
```

Remove when needed:

```bash
zeroclaw plugin remove mytool
```

### 6) Ensure plugin loading is enabled in config

In `config.toml`:

```toml
[plugins]
enabled = true
plugins_dir = "~/.zeroclaw/plugins"
auto_discover = true
max_plugins = 50
```

### 7) Verify discovery path alignment

Current code paths use two different roots:

1. `zeroclaw plugin install` uses `workspace_dir/plugins` via `PluginHost::new(&config.workspace_dir)`.
2. Runtime tool loading uses `[plugins].plugins_dir`.

If a plugin installs but does not load, align these directories first (or copy the plugin folder to the configured `plugins_dir`).

### 8) Call out current execution status

At the time of writing, `src/plugins/wasm_tool.rs` still contains a placeholder bridge and returns:

`WASM execution bridge not yet implemented`

So the plugin lifecycle (manifest/discovery/install/list/info) exists, but actual WASM tool execution is not fully wired.

Do not hide this from users. State it early when they ask to run plugin logic end-to-end.

## Troubleshooting Checklist

When plugin load/usage fails, check in this order:

1. `plugins-wasm` feature is enabled in the ZeroClaw build.
2. `manifest.toml` exists and parses.
3. `wasm_path` is correct relative to manifest location.
4. Built artifact exists at `wasm_path`.
5. `capabilities` includes `"tool"`.
6. `[plugins].enabled = true`.
7. Install path and `plugins_dir` point to the same plugin root.

## Fallback: Native Tool (In-Tree)

If the user needs a production-ready callable tool immediately, suggest native in-tree tool implementation:

1. Implement `Tool` in `src/tools/` (`name`, `description`, `parameters_schema`, `execute`).
2. Register it in `src/tools/mod.rs` (`default_tools` or `all_tools_with_runtime`).
3. Add config gating in `src/config/schema.rs` if needed.
4. Add focused tests for schema and execution behavior.

Validation commands:

```bash
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
cargo test
```

## Response Style

When using this skill:

1. Produce concrete file edits, not only theory.
2. Prefer minimal working plugin first, then iterate.
3. Include exact commands to build and verify.
4. Explicitly mention any runtime limitations you observe in current source.
