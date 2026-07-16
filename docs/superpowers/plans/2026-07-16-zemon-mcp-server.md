# zemon-mcp MCP Server Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `zemon-mcp` library crate and a `zemon mcp` subcommand that serve the zemon monitoring surface to AI agents as an MCP (Model Context Protocol) server over stdio.

**Architecture:** A thin `rmcp`-based adapter over `zemon-core`. Each MCP tool delegates to a small, testable handler function returning `Result<String, ZemonError>` (a JSON string), which the `#[tool]` method wraps into an MCP result. A single lazily-opened, reconnecting Zenoh session (held in `ServerState`) is reused across calls. The `zemon mcp` subcommand resolves config via the existing `resolve_config` and runs the server.

**Tech Stack:** Rust, tokio, `rmcp` + `rmcp-macros` (official Rust MCP SDK), `serde`/`serde_json`, `schemars`, `zenoh` (via `zemon-core`).

## Global Constraints

- Rust edition/toolchain: match the existing workspace (`cargo build` must pass on Rust 1.75+).
- Single shipped binary: MCP logic lives in the `zemon-mcp` **library** crate; the only new binary entry point is the `zemon mcp` subcommand inside the existing `zemon` binary. Do **not** add a second `[[bin]]`.
- Reuse `zemon-core` functions and the CLI's serde output shapes (`zemon_core::output::to_collection_json`, `TopicInfo`, `NodeInfo`, `ZenohMessage`, etc.). Do not re-implement Zenoh logic.
- Errors surface as `zemon_core::error::ZemonError` internally and map to MCP tool errors preserving `kind`.
- English in code; Korean allowed only in design docs.
- Commit messages: `feat(mcp):`, `test(mcp):`, `chore:`.
- Reuse the resolved config from `zemon_core::config::resolve_config` (do not re-read env ad hoc).

**rmcp API note (verify in Task 1 — REQUIRED):** rmcp resolves to **2.2.0**, whose exact
API may differ from the code sketches below (which follow the older 0.16-era README).
The code in this plan encodes the *structure* (a server struct with `#[tool]` methods, a
`ToolRouter`, a `#[tool_handler] impl ServerHandler`, and `serve(stdio())`), not guaranteed
symbol names. **The authoritative source is the installed crate's own examples**: after
`cargo add`, read `~/.cargo/registry/src/*/rmcp-2.2.0/examples/` (and `src/` for exact
signatures) and copy the real v2 patterns for: the tool router/handler macros, the
`Parameters<T>` wrapper, `CallToolResult` success construction, the text-content constructor
(`Content::text` vs `ContentBlock::text`), the stdio transport entry point, and the
`ErrorData`/`McpError` constructors. Task 1 pins these against a compiling hello-world; every
later task follows the exact pattern Task 1 establishes. Where a name below differs from the
resolved crate, the crate wins — keep the structure identical.

---

## File Structure

```
crates/zemon-mcp/
  Cargo.toml                 # new crate: rmcp + zemon-core + serde + tokio
  src/lib.rs                 # pub: serve_stdio(config), re-exports; wires router + handler
  src/state.rs               # ServerState: config + lazy/reconnecting session
  src/error.rs               # ZemonError -> rmcp McpError mapping
  src/handlers.rs            # testable per-tool functions returning Result<String, ZemonError>
  src/server.rs              # ZemonMcpServer struct, #[tool_router]/#[tool] methods, #[tool_handler]
  tests/pure_tools.rs        # integration tests for no-session tools (keyexpr, config_show)
crates/zemon-cli/
  Cargo.toml                 # add zemon-mcp dependency
  src/cli.rs                 # add `Command::Mcp`
  src/main.rs                # add Mcp arm -> zemon_mcp::serve_stdio(resolved.config)
README.md                    # document `zemon mcp`
```

Responsibilities:
- `state.rs` owns session lifecycle only.
- `handlers.rs` owns "call core, serialize to JSON" logic — pure of rmcp types, so unit-testable.
- `server.rs` owns rmcp wiring only (schemas, wrapping handler output into `CallToolResult`).
- `error.rs` owns the single error-mapping function.

---

## Task 1: Crate skeleton, `zemon mcp` subcommand, and one pure tool over stdio

**Files:**
- Create: `crates/zemon-mcp/Cargo.toml`
- Create: `crates/zemon-mcp/src/lib.rs`
- Create: `crates/zemon-mcp/src/server.rs`
- Create: `crates/zemon-mcp/src/error.rs`
- Create: `crates/zemon-mcp/src/handlers.rs`
- Modify: `Cargo.toml` (workspace `members`)
- Modify: `crates/zemon-cli/Cargo.toml`
- Modify: `crates/zemon-cli/src/cli.rs`
- Modify: `crates/zemon-cli/src/main.rs`

**Interfaces:**
- Produces: `zemon_mcp::serve_stdio(config: zemon_core::config::ZemonConfig) -> anyhow::Result<()>`
- Produces: `zemon_mcp::handlers::keyexpr_json(a: &str, b: &str) -> Result<String, ZemonError>`
- Produces: `zemon_mcp::error::to_mcp_error(e: ZemonError) -> rmcp::ErrorData`
- Consumes: `zemon_core::keyexpr::compare(a, b) -> Result<KeyExprRelation, ZemonError>`

- [ ] **Step 1: Confirm the resolved rmcp version and API names**

Run: `cargo add rmcp --dry-run` (from repo root) and note the resolved version. Then check the docs for that version at `https://docs.rs/rmcp` for the exact names of: the stdio transport (`rmcp::transport::stdio`), the tool-content constructor (`Content::text` or `ContentBlock::text`), and `CallToolResult::success`. Use those exact names in the code below.

- [ ] **Step 2: Add the crate to the workspace**

Edit root `Cargo.toml` `members` to include `"crates/zemon-mcp"`.

Create `crates/zemon-mcp/Cargo.toml`:

```toml
[package]
name = "zemon-mcp"
version = "0.1.0"
edition = "2021"

[dependencies]
zemon-core = { path = "../zemon-core" }
# rmcp resolves to 2.x (confirmed 2.2.0). The `server` feature pulls in the tool
# macros and stdio transport in this line's feature set.
rmcp = { version = "2", features = ["server", "macros", "transport-io"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
schemars = "0.8"
anyhow = "1"
tracing = "0.1"
```

- [ ] **Step 3: Write the failing test for the first pure handler**

Create `crates/zemon-mcp/src/handlers.rs`:

```rust
//! Per-tool logic, deliberately free of rmcp types so it is unit-testable.
//! Every function returns a JSON string on success (the exact shape the CLI
//! `--json` mode produces) or a `ZemonError` on failure.

use zemon_core::error::ZemonError;

/// `keyexpr` tool: pure key-expression relationship comparison (no session).
pub fn keyexpr_json(a: &str, b: &str) -> Result<String, ZemonError> {
    let relation = zemon_core::keyexpr::compare(a, b)?;
    serde_json::to_string(&relation)
        .map_err(|e| ZemonError::internal(format!("serialize keyexpr result: {e}")))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn keyexpr_json_reports_inclusion() {
        let json = keyexpr_json("a/*", "a/b").unwrap();
        let v: serde_json::Value = serde_json::from_str(&json).unwrap();
        assert_eq!(v["a_includes_b"], true);
        assert_eq!(v["b_includes_a"], false);
    }

    #[test]
    fn keyexpr_json_rejects_invalid_expr() {
        assert!(keyexpr_json("a/**/**/c", "b").is_err());
    }
}
```

- [ ] **Step 4: Run the test to verify it fails to compile (module not wired)**

Run: `cargo test -p zemon-mcp keyexpr_json`
Expected: FAIL — `zemon-mcp` has no `lib.rs` / module yet.

- [ ] **Step 5: Create the error mapping**

Create `crates/zemon-mcp/src/error.rs`:

```rust
//! Map the core error taxonomy onto MCP tool errors, preserving `kind`.

use rmcp::ErrorData as McpError;
use zemon_core::error::ZemonError;

/// Convert a `ZemonError` into an MCP tool error. The human message is carried
/// verbatim and the stable `kind` is attached as structured data so agents see
/// the same taxonomy the CLI exposes.
pub fn to_mcp_error(e: ZemonError) -> McpError {
    let data = serde_json::json!({ "kind": e.kind_str() });
    McpError::internal_error(e.to_string(), Some(data))
}
```

Note: if `ZemonError` has no `kind_str()`, add one in `crates/zemon-core/src/error.rs` returning the snake_case kind (the same value used by `to_json`). Verify the exact `McpError` constructor name for the resolved rmcp version (Step 1); `internal_error(message, data)` is the documented shape.

- [ ] **Step 6: Create the server with the first tool**

Create `crates/zemon-mcp/src/server.rs`:

```rust
use rmcp::handler::server::router::tool::ToolRouter;
use rmcp::handler::server::wrapper::Parameters;
use rmcp::model::{CallToolResult, Content};
use rmcp::{schemars, tool, tool_handler, tool_router, ServerHandler};

use crate::error::to_mcp_error;
use crate::handlers;
use crate::state::ServerState;
use std::sync::Arc;

#[derive(Clone)]
pub struct ZemonMcpServer {
    pub(crate) state: Arc<ServerState>,
    tool_router: ToolRouter<ZemonMcpServer>,
}

#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct KeyexprParams {
    /// First key expression (A).
    pub a: String,
    /// Second key expression (B).
    pub b: String,
}

#[tool_router]
impl ZemonMcpServer {
    pub fn new(state: Arc<ServerState>) -> Self {
        Self { state, tool_router: Self::tool_router() }
    }

    #[tool(description = "Compare two key expressions (intersect/include). Pure, no network.")]
    fn keyexpr(&self, Parameters(p): Parameters<KeyexprParams>) -> Result<CallToolResult, rmcp::ErrorData> {
        let json = handlers::keyexpr_json(&p.a, &p.b).map_err(to_mcp_error)?;
        Ok(CallToolResult::success(vec![Content::text(json)]))
    }
}

#[tool_handler]
impl ServerHandler for ZemonMcpServer {}
```

- [ ] **Step 7: Create `state.rs` (minimal for now) and `lib.rs`**

Create `crates/zemon-mcp/src/state.rs`:

```rust
//! Server-wide state: the resolved config and the shared Zenoh session.
//! (Session lifecycle is fleshed out in Task 2.)

use zemon_core::config::ZemonConfig;

pub struct ServerState {
    pub config: ZemonConfig,
}

impl ServerState {
    pub fn new(config: ZemonConfig) -> Self {
        Self { config }
    }
}
```

Create `crates/zemon-mcp/src/lib.rs`:

```rust
pub mod error;
pub mod handlers;
pub mod server;
pub mod state;

use anyhow::Result;
use rmcp::{transport::stdio, ServiceExt};
use std::sync::Arc;
use zemon_core::config::ZemonConfig;

/// Run the MCP server over stdio until the client disconnects.
pub async fn serve_stdio(config: ZemonConfig) -> Result<()> {
    let state = Arc::new(state::ServerState::new(config));
    let server = server::ZemonMcpServer::new(state);
    let service = server.serve(stdio()).await?;
    service.waiting().await?;
    Ok(())
}
```

- [ ] **Step 8: Run the pure-handler test to verify it passes**

Run: `cargo test -p zemon-mcp keyexpr_json`
Expected: PASS (2 tests).

- [ ] **Step 9: Wire the `zemon mcp` subcommand**

In `crates/zemon-cli/Cargo.toml` add under `[dependencies]`:

```toml
zemon-mcp = { path = "../zemon-mcp" }
```

In `crates/zemon-cli/src/cli.rs`, add a variant to `enum Command` (place it right after `Config`):

```rust
    /// Run an MCP (Model Context Protocol) server over stdio for AI agents
    Mcp,
```

In `crates/zemon-cli/src/main.rs`, add an arm in `run` (before the `Command::Tui` arm). It consumes the already-resolved config:

```rust
        Command::Mcp => {
            zemon_mcp::serve_stdio(config)
                .await
                .map_err(|e| ZemonError::internal(format!("mcp server: {e}")))?;
        }
```

- [ ] **Step 10: Build the whole workspace**

Run: `cargo build`
Expected: PASS (workspace compiles, single `zemon` binary).

- [ ] **Step 11: Smoke-test tool discovery over stdio**

Run (sends `initialize` then `tools/list` as line-delimited JSON-RPC):

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"smoke","version":"0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' \
  | ./target/debug/zemon mcp
```

Expected: JSON-RPC responses on stdout; the `tools/list` result contains a tool named `keyexpr`. (If the exact `initialize` params differ for the resolved rmcp version, adjust to match its handshake.)

- [ ] **Step 12: Commit**

```bash
git add crates/zemon-mcp crates/zemon-cli Cargo.toml
git commit -m "feat(mcp): zemon-mcp crate skeleton + zemon mcp subcommand + keyexpr tool"
```

---

## Task 2: Lazy, reconnecting shared session + first session tool (`discover`)

**Files:**
- Modify: `crates/zemon-mcp/src/state.rs`
- Modify: `crates/zemon-mcp/src/handlers.rs`
- Modify: `crates/zemon-mcp/src/server.rs`

**Interfaces:**
- Produces: `ServerState::session(&self) -> Result<Arc<zenoh::Session>, ZemonError>` — opens on first use, reconnects if the cached session was cleared.
- Produces: `ServerState::invalidate_session(&self)` — drops the cached session so the next call reopens.
- Produces: `handlers::discover_json(session: &zenoh::Session, key_expr: &str) -> Result<String, ZemonError>`
- Consumes: `zemon_core::session::open_session(&ZemonConfig) -> Result<zenoh::Session, ZemonError>`, `zemon_core::discover::discover(&Session, &str)`, `zemon_core::output::to_collection_json`.

- [ ] **Step 1: Replace `state.rs` with the session-holding version**

```rust
//! Server-wide state: resolved config + one shared, lazily-opened Zenoh session
//! reused across tool calls. A dropped session is reopened on the next call.

use std::sync::Arc;
use tokio::sync::Mutex;
use zemon_core::config::ZemonConfig;
use zemon_core::error::ZemonError;

pub struct ServerState {
    pub config: ZemonConfig,
    session: Mutex<Option<Arc<zenoh::Session>>>,
}

impl ServerState {
    pub fn new(config: ZemonConfig) -> Self {
        Self { config, session: Mutex::new(None) }
    }

    /// Return the shared session, opening it on first use. Callers that observe
    /// a connection error should call `invalidate_session` so the next call
    /// reopens.
    pub async fn session(&self) -> Result<Arc<zenoh::Session>, ZemonError> {
        let mut guard = self.session.lock().await;
        if let Some(s) = guard.as_ref() {
            return Ok(s.clone());
        }
        let session = zemon_core::session::open_session(&self.config).await?;
        let session = Arc::new(session);
        *guard = Some(session.clone());
        Ok(session)
    }

    /// Drop the cached session so the next `session()` reopens it.
    pub async fn invalidate_session(&self) {
        *self.session.lock().await = None;
    }
}
```

Note: confirm `zemon_core::session::open_session` returns an owned `zenoh::Session` (it does — `open_session(&config) -> Result<Session, ZemonError>`). If `Session` is already `Clone`/cheaply shareable, `Arc` is still fine and keeps the cache type uniform.

- [ ] **Step 2: Write the failing test for `discover_json` serialization shape**

Add to `crates/zemon-mcp/src/handlers.rs` `tests` module a test that does not need a live session by asserting the empty-collection contract of `to_collection_json` indirectly. First add the handler:

```rust
/// `discover` tool: list active keys/topics for `key_expr`.
pub async fn discover_json(
    session: &zenoh::Session,
    key_expr: &str,
) -> Result<String, ZemonError> {
    let topics = zemon_core::discover::discover(session, key_expr)
        .await
        .map_err(ZemonError::from)?;
    zemon_core::output::to_collection_json(&topics)
        .map_err(|e| ZemonError::internal(format!("serialize discover result: {e}")))
}
```

Add a serialization-contract test that exercises `to_collection_json` with the same type, no network:

```rust
#[test]
fn discover_json_uses_count_items_envelope() {
    let empty: Vec<zemon_core::types::TopicInfo> = vec![];
    let json = zemon_core::output::to_collection_json(&empty).unwrap();
    let v: serde_json::Value = serde_json::from_str(&json).unwrap();
    assert_eq!(v["count"], 0);
    assert!(v["items"].as_array().unwrap().is_empty());
}
```

Note: `zemon_core::discover::discover` returns `color_eyre::Result`; if `ZemonError::from(color_eyre::Report)` is not available, wrap with `.map_err(|e| ZemonError::internal(e.to_string()))` instead. Confirm which conversion exists (the CLI uses `?` against `ZemonError`, so a `From<Report>` path exists) and use it consistently.

- [ ] **Step 3: Run the test to verify it fails**

Run: `cargo test -p zemon-mcp discover_json_uses_count_items_envelope`
Expected: FAIL to compile until the handler + import compile, then PASS once Step 2 code is in. (If it compiles and passes immediately, that is acceptable — the deliverable is the handler.)

- [ ] **Step 4: Add the `discover` tool to the server**

In `crates/zemon-mcp/src/server.rs`, add inside the `#[tool_router] impl`:

```rust
    #[tool(description = "List active keys/topics matching a key expression.")]
    async fn discover(
        &self,
        Parameters(p): Parameters<DiscoverParams>,
    ) -> Result<CallToolResult, rmcp::ErrorData> {
        let session = self.state.session().await.map_err(to_mcp_error)?;
        match handlers::discover_json(&session, &p.key_expr).await {
            Ok(json) => Ok(CallToolResult::success(vec![Content::text(json)])),
            Err(e) => {
                if e.is_connection() {
                    self.state.invalidate_session().await;
                }
                Err(to_mcp_error(e))
            }
        }
    }
```

And add the params struct near `KeyexprParams`:

```rust
#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct DiscoverParams {
    /// Key expression to filter (default "**").
    #[serde(default = "default_key_expr")]
    pub key_expr: String,
}

fn default_key_expr() -> String { "**".to_string() }
```

Note: `ZemonError::is_connection()` — add this predicate to `zemon-core/src/error.rs` if absent (returns `true` for the `connection` kind). It is used to decide session invalidation on drop.

- [ ] **Step 5: Build and run existing tests**

Run: `cargo test -p zemon-mcp`
Expected: PASS (pure tests still green; new serialization test green).

- [ ] **Step 6: Manual session smoke test (requires a local router)**

Start `zenohd` in another terminal, then:

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"smoke","version":"0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"discover","arguments":{"key_expr":"**"}}}' \
  | ./target/debug/zemon mcp
```

Expected: a `tools/call` result whose content text parses to `{"count":N,"items":[...]}`.

- [ ] **Step 7: Commit**

```bash
git add crates/zemon-mcp crates/zemon-core
git commit -m "feat(mcp): lazy reconnecting session + discover tool"
```

---

## Task 3: Pure/config tools — `config_show`, and register `info`

**Files:**
- Modify: `crates/zemon-mcp/src/handlers.rs`
- Modify: `crates/zemon-mcp/src/server.rs`
- Create: `crates/zemon-mcp/tests/pure_tools.rs`

**Interfaces:**
- Produces: `handlers::config_show_json() -> Result<String, ZemonError>`
- Produces: `handlers::info_json(session: &zenoh::Session, mode: ConnectMode) -> Result<String, ZemonError>`
- Consumes: `zemon_core::config::resolve_config`, `zemon_core::info::session_info`.

- [ ] **Step 1: Write the failing test for `config_show_json`**

Add to `handlers.rs`:

```rust
/// `config_show` tool: the effective, allow-listed configuration (no session).
pub fn config_show_json() -> Result<String, ZemonError> {
    let resolved = zemon_core::config::resolve_config(Default::default())
        .map_err(|e| ZemonError::invalid_input(e.to_string()))?;
    serde_json::to_string(&resolved.effective)
        .map_err(|e| ZemonError::internal(format!("serialize effective config: {e}")))
}
```

Add its test:

```rust
#[test]
fn config_show_json_is_allow_listed_effective_config() {
    let json = config_show_json().unwrap();
    let v: serde_json::Value = serde_json::from_str(&json).unwrap();
    // Effective view exposes the allow-list only; never raw Zenoh secrets.
    assert!(v.get("endpoint").is_some());
    assert!(v.get("connect_timeout").is_some());
    assert!(v.get("password").is_none());
}
```

- [ ] **Step 2: Run the test to verify it fails, then passes after Step 1 code lands**

Run: `cargo test -p zemon-mcp config_show_json`
Expected: FAIL (missing fn) → after adding the fn, PASS.

- [ ] **Step 3: Add `info_json` handler**

```rust
/// `info` tool: current session details.
pub async fn info_json(
    session: &zenoh::Session,
    mode: zemon_core::config::ConnectMode,
) -> Result<String, ZemonError> {
    let detail = zemon_core::info::session_info(session, mode)
        .await
        .map_err(|e| ZemonError::internal(e.to_string()))?;
    serde_json::to_string(&detail)
        .map_err(|e| ZemonError::internal(format!("serialize info: {e}")))
}
```

- [ ] **Step 4: Register `config_show` and `info` tools**

In `server.rs` add inside the router impl:

```rust
    #[tool(description = "Show the effective, allow-listed configuration. No network.")]
    fn config_show(&self) -> Result<CallToolResult, rmcp::ErrorData> {
        let json = handlers::config_show_json().map_err(to_mcp_error)?;
        Ok(CallToolResult::success(vec![Content::text(json)]))
    }

    #[tool(description = "Show current Zenoh session information.")]
    async fn info(&self) -> Result<CallToolResult, rmcp::ErrorData> {
        let session = self.state.session().await.map_err(to_mcp_error)?;
        match handlers::info_json(&session, self.state.config.mode).await {
            Ok(json) => Ok(CallToolResult::success(vec![Content::text(json)])),
            Err(e) => {
                if e.is_connection() { self.state.invalidate_session().await; }
                Err(to_mcp_error(e))
            }
        }
    }
```

- [ ] **Step 5: Add the pure-tools integration test (no zenohd needed)**

Create `crates/zemon-mcp/tests/pure_tools.rs`:

```rust
//! End-to-end checks for tools that need no network session.

use zemon_mcp::handlers;

#[test]
fn keyexpr_and_config_show_work_without_a_session() {
    let ke = handlers::keyexpr_json("a/*", "a/b").unwrap();
    assert!(serde_json::from_str::<serde_json::Value>(&ke).unwrap()["a_includes_b"]
        .as_bool()
        .unwrap());

    let cfg = handlers::config_show_json().unwrap();
    assert!(serde_json::from_str::<serde_json::Value>(&cfg).unwrap()["endpoint"].is_object());
}
```

- [ ] **Step 6: Run all zemon-mcp tests**

Run: `cargo test -p zemon-mcp`
Expected: PASS (unit + `pure_tools` integration).

- [ ] **Step 7: Commit**

```bash
git add crates/zemon-mcp
git commit -m "feat(mcp): config_show + info tools and pure-tools integration test"
```

---

## Task 4: Read tools — `query`, `nodes`, `liveliness`

**Files:**
- Modify: `crates/zemon-mcp/src/handlers.rs`
- Modify: `crates/zemon-mcp/src/server.rs`

**Interfaces:**
- Produces: `handlers::query_json(session, key_expr, payload: Option<&str>, timeout: Duration, limit: Option<usize>) -> Result<String, ZemonError>`
- Produces: `handlers::nodes_json(session: &zenoh::Session) -> Result<String, ZemonError>`
- Produces: `handlers::liveliness_json(session: &zenoh::Session, key_expr: &str) -> Result<String, ZemonError>`
- Consumes: `zemon_core::query::get(session, key_expr, payload, timeout, limit)`, `zemon_core::registry::query_admin_nodes(session)`, `zemon_core::discover::query_liveliness(session, key_expr)`.

- [ ] **Step 1: Add the three handlers**

```rust
use std::time::Duration;

/// `query` tool: send a GET and collect replies (bounded by `timeout`/`limit`).
pub async fn query_json(
    session: &zenoh::Session,
    key_expr: &str,
    payload: Option<&str>,
    timeout: Duration,
    limit: Option<usize>,
) -> Result<String, ZemonError> {
    let replies = zemon_core::query::get(session, key_expr, payload, timeout, limit)
        .await
        .map_err(|e| ZemonError::internal(e.to_string()))?;
    zemon_core::output::to_collection_json(&replies)
        .map_err(|e| ZemonError::internal(format!("serialize query result: {e}")))
}

/// `nodes` tool: one snapshot of discovered Zenoh nodes (admin space).
pub async fn nodes_json(session: &zenoh::Session) -> Result<String, ZemonError> {
    let nodes = zemon_core::registry::query_admin_nodes(session)
        .await
        .map_err(|e| ZemonError::internal(e.to_string()))?;
    zemon_core::output::to_collection_json(&nodes)
        .map_err(|e| ZemonError::internal(format!("serialize nodes result: {e}")))
}

/// `liveliness` tool: query current liveliness tokens.
pub async fn liveliness_json(
    session: &zenoh::Session,
    key_expr: &str,
) -> Result<String, ZemonError> {
    let tokens = zemon_core::discover::query_liveliness(session, key_expr)
        .await
        .map_err(|e| ZemonError::internal(e.to_string()))?;
    zemon_core::output::to_collection_json(&tokens)
        .map_err(|e| ZemonError::internal(format!("serialize liveliness result: {e}")))
}
```

- [ ] **Step 2: Add a serialization test for the query envelope**

```rust
#[test]
fn query_result_uses_count_items_envelope() {
    let empty: Vec<zemon_core::types::ZenohMessage> = vec![];
    let json = zemon_core::output::to_collection_json(&empty).unwrap();
    let v: serde_json::Value = serde_json::from_str(&json).unwrap();
    assert_eq!(v["count"], 0);
}
```

- [ ] **Step 3: Run the test**

Run: `cargo test -p zemon-mcp query_result_uses_count_items_envelope`
Expected: PASS.

- [ ] **Step 4: Register the three tools**

In `server.rs`, add params structs:

```rust
#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct QueryParams {
    /// Key expression to query.
    pub key_expr: String,
    /// Optional payload to include in the GET.
    #[serde(default)]
    pub payload: Option<String>,
    /// Timeout in milliseconds (default 5000).
    #[serde(default = "default_query_timeout_ms")]
    pub timeout_ms: u64,
    /// Return at most N replies.
    #[serde(default)]
    pub limit: Option<u64>,
}
fn default_query_timeout_ms() -> u64 { 5000 }

#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct LivelinessParams {
    #[serde(default = "default_key_expr")]
    pub key_expr: String,
}
```

Add the tools (each invalidates the session on a connection error, like `discover`):

```rust
    #[tool(description = "Send a Zenoh GET query and collect replies.")]
    async fn query(&self, Parameters(p): Parameters<QueryParams>) -> Result<CallToolResult, rmcp::ErrorData> {
        let session = self.state.session().await.map_err(to_mcp_error)?;
        let r = handlers::query_json(
            &session, &p.key_expr, p.payload.as_deref(),
            std::time::Duration::from_millis(p.timeout_ms),
            p.limit.map(|n| n as usize),
        ).await;
        self.finish(r).await
    }

    #[tool(description = "List discovered Zenoh nodes (one snapshot).")]
    async fn nodes(&self) -> Result<CallToolResult, rmcp::ErrorData> {
        let session = self.state.session().await.map_err(to_mcp_error)?;
        self.finish(handlers::nodes_json(&session).await).await
    }

    #[tool(description = "Query current liveliness tokens.")]
    async fn liveliness(&self, Parameters(p): Parameters<LivelinessParams>) -> Result<CallToolResult, rmcp::ErrorData> {
        let session = self.state.session().await.map_err(to_mcp_error)?;
        self.finish(handlers::liveliness_json(&session, &p.key_expr).await).await
    }
```

- [ ] **Step 5: Add the shared `finish` helper (dedupe the success/invalidate pattern)**

In `server.rs`, add a non-tool method to the `impl ZemonMcpServer` block (outside `#[tool_router]` if the macro requires tool-only methods there; otherwise inside is fine):

```rust
impl ZemonMcpServer {
    async fn finish(&self, r: Result<String, zemon_core::error::ZemonError>) -> Result<CallToolResult, rmcp::ErrorData> {
        match r {
            Ok(json) => Ok(CallToolResult::success(vec![Content::text(json)])),
            Err(e) => {
                if e.is_connection() { self.state.invalidate_session().await; }
                Err(to_mcp_error(e))
            }
        }
    }
}
```

Then refactor the `discover` and `info` tools from Tasks 2–3 to use `self.finish(...)`.

- [ ] **Step 6: Build and test**

Run: `cargo test -p zemon-mcp && cargo build`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add crates/zemon-mcp
git commit -m "feat(mcp): query, nodes, liveliness tools + finish() helper"
```

---

## Task 5: Config-driven tools — `scout`, `doctor`

**Files:**
- Modify: `crates/zemon-mcp/src/handlers.rs`
- Modify: `crates/zemon-mcp/src/server.rs`

**Interfaces:**
- Produces: `handlers::scout_json(config: &ZemonConfig, per_port_timeout: Duration, port_range: (u16, u16)) -> Result<String, ZemonError>`
- Produces: `handlers::doctor_json(config: &ZemonConfig, timeout: Duration) -> Result<String, ZemonError>`
- Consumes: `zemon_core::scout::scout_port_range`, `zemon_core::doctor::run(config, timeout) -> DoctorReport`.

- [ ] **Step 1: Confirm the scout entry point**

Run: `grep -nE "pub async fn scout" crates/zemon-core/src/scout.rs` and note whether the port-range scan is `scout_port_range(config, start, end, per_port_timeout)` or similar. Mirror the argument order the CLI `Command::Scout` arm uses in `crates/zemon-cli/src/main.rs`. Use that exact call below (the placeholder call is `scout_port_range`).

- [ ] **Step 2: Add the handlers**

```rust
/// `scout` tool: multicast scan of a port range (does its own transient sessions).
pub async fn scout_json(
    config: &zemon_core::config::ZemonConfig,
    per_port_timeout: Duration,
    port_range: (u16, u16),
) -> Result<String, ZemonError> {
    let results = zemon_core::scout::scout_port_range(config, port_range.0, port_range.1, per_port_timeout)
        .await
        .map_err(|e| ZemonError::internal(e.to_string()))?;
    zemon_core::output::to_collection_json(&results)
        .map_err(|e| ZemonError::internal(format!("serialize scout result: {e}")))
}

/// `doctor` tool: connection diagnostics. Opens its own session; returns a report.
pub async fn doctor_json(
    config: &zemon_core::config::ZemonConfig,
    timeout: Duration,
) -> Result<String, ZemonError> {
    let report = zemon_core::doctor::run(config, timeout).await;
    serde_json::to_string(&report)
        .map_err(|e| ZemonError::internal(format!("serialize doctor report: {e}")))
}
```

Note: if `scout_port_range`'s real signature differs, adapt the call and the tuple. If `DoctorReport` is not `Serialize`, add `#[derive(Serialize)]` to it in `crates/zemon-core/src/doctor.rs` (the CLI already emits it as `--json`, so a serializer exists — reuse the CLI's path).

- [ ] **Step 3: Add a doctor serialization test**

```rust
#[tokio::test]
async fn doctor_json_is_serializable() {
    // Uses a bogus endpoint + tiny timeout so it fails fast without a router,
    // but still produces a serializable report object.
    let mut config = zemon_core::config::ZemonConfig::default();
    config.endpoint = "tcp/127.0.0.1:1".to_string();
    let json = doctor_json(&config, Duration::from_millis(200)).await.unwrap();
    let v: serde_json::Value = serde_json::from_str(&json).unwrap();
    assert!(v.is_object());
}
```

- [ ] **Step 4: Run the test**

Run: `cargo test -p zemon-mcp doctor_json_is_serializable`
Expected: PASS (report serializes even when checks fail).

- [ ] **Step 5: Register the tools**

In `server.rs`:

```rust
#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct ScoutParams {
    /// Inclusive multicast port range start (default 7446).
    #[serde(default = "default_scout_start")] pub start_port: u16,
    /// Inclusive multicast port range end (default 7546).
    #[serde(default = "default_scout_end")] pub end_port: u16,
    /// Per-port timeout in milliseconds (default 1000).
    #[serde(default = "default_scout_timeout_ms")] pub per_port_timeout_ms: u64,
}
fn default_scout_start() -> u16 { 7446 }
fn default_scout_end() -> u16 { 7546 }
fn default_scout_timeout_ms() -> u64 { 1000 }

#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct DoctorParams {
    /// Overall diagnostic deadline in milliseconds (default 5000).
    #[serde(default = "default_doctor_timeout_ms")] pub timeout_ms: u64,
}
fn default_doctor_timeout_ms() -> u64 { 5000 }
```

```rust
    #[tool(description = "Scout multicast ports for Zenoh nodes (no router needed).")]
    async fn scout(&self, Parameters(p): Parameters<ScoutParams>) -> Result<CallToolResult, rmcp::ErrorData> {
        let r = handlers::scout_json(
            &self.state.config,
            std::time::Duration::from_millis(p.per_port_timeout_ms),
            (p.start_port, p.end_port),
        ).await;
        // scout uses transient sessions, so no shared-session invalidation.
        r.map(|json| CallToolResult::success(vec![Content::text(json)])).map_err(to_mcp_error)
    }

    #[tool(description = "Diagnose the connection: config, session, connectivity checks.")]
    async fn doctor(&self, Parameters(p): Parameters<DoctorParams>) -> Result<CallToolResult, rmcp::ErrorData> {
        let r = handlers::doctor_json(&self.state.config, std::time::Duration::from_millis(p.timeout_ms)).await;
        r.map(|json| CallToolResult::success(vec![Content::text(json)])).map_err(to_mcp_error)
    }
```

- [ ] **Step 6: Build and test**

Run: `cargo test -p zemon-mcp && cargo build`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add crates/zemon-mcp crates/zemon-core
git commit -m "feat(mcp): scout and doctor tools"
```

---

## Task 6: Bounded streaming — `sub_snapshot`

**Files:**
- Modify: `crates/zemon-mcp/src/handlers.rs`
- Modify: `crates/zemon-mcp/src/server.rs`

**Interfaces:**
- Produces: `handlers::sub_snapshot_json(session, key_expr, count: Option<usize>, duration: Option<Duration>, max_payload_bytes: Option<usize>) -> Result<String, ZemonError>`
- Consumes: `zemon_core::subscriber::subscribe(session, key_expr, tx)`, `zemon_core::types::ZenohMessage`.

- [ ] **Step 1: Add the bounded-collect handler**

```rust
use tokio::sync::mpsc;

/// `sub_snapshot` tool: subscribe and collect messages until `count` is reached
/// or `duration` elapses (at least one bound is required), then return a batch.
pub async fn sub_snapshot_json(
    session: &zenoh::Session,
    key_expr: &str,
    count: Option<usize>,
    duration: Option<Duration>,
    max_payload_bytes: Option<usize>,
) -> Result<String, ZemonError> {
    if count.is_none() && duration.is_none() {
        return Err(ZemonError::invalid_input(
            "sub_snapshot requires at least one of `count` or `duration`".to_string(),
        ));
    }
    let (tx, mut rx) = mpsc::unbounded_channel();
    let handle = zemon_core::subscriber::subscribe(session, key_expr, tx)
        .await
        .map_err(|e| ZemonError::internal(e.to_string()))?;

    let deadline = duration.map(|d| tokio::time::Instant::now() + d);
    let mut collected: Vec<zemon_core::types::ZenohMessage> = Vec::new();
    loop {
        if let Some(max) = count {
            if collected.len() >= max { break; }
        }
        let recv = match deadline {
            Some(dl) => match tokio::time::timeout_at(dl, rx.recv()).await {
                Ok(msg) => msg,
                Err(_) => break, // duration elapsed
            },
            None => rx.recv().await,
        };
        match recv {
            Some(msg) => collected.push(msg),
            None => break, // subscriber closed
        }
    }
    handle.abort();

    // Apply payload capping to the JSON view if requested (mirrors CLI --max-payload-bytes).
    let json = match max_payload_bytes {
        Some(cap) => zemon_core::output::to_collection_json_limited(&collected, cap),
        None => zemon_core::output::to_collection_json(&collected),
    }
    .map_err(|e| ZemonError::internal(format!("serialize sub snapshot: {e}")))?;
    Ok(json)
}
```

Note: confirm `to_collection_json_limited`'s exact signature (it exists in `output.rs`); if its capping semantics differ (e.g. it caps the per-item payload view, not the collection), match how the CLI `sub --max-payload-bytes` uses it and adjust. If it is not a drop-in, cap by mapping each `ZenohMessage` to its `to_view_capped(cap)` before serializing.

- [ ] **Step 2: Write a bound-validation test (no network)**

```rust
#[tokio::test]
async fn sub_snapshot_requires_a_bound() {
    // We cannot open a real session here, but the bound check happens before
    // any session use, so pass a dummy via a helper that returns early.
    // Instead, assert the pure precondition by calling the validation directly:
    let err = super::sub_snapshot_bound_error(None, None);
    assert!(err.is_some());
    assert!(super::sub_snapshot_bound_error(Some(1), None).is_none());
}
```

Extract the precondition so it is testable without a session:

```rust
pub(crate) fn sub_snapshot_bound_error(count: Option<usize>, duration: Option<Duration>) -> Option<ZemonError> {
    if count.is_none() && duration.is_none() {
        Some(ZemonError::invalid_input(
            "sub_snapshot requires at least one of `count` or `duration`".to_string(),
        ))
    } else {
        None
    }
}
```

Then have `sub_snapshot_json` call `if let Some(e) = sub_snapshot_bound_error(count, duration) { return Err(e); }` instead of the inline check.

- [ ] **Step 3: Run the test**

Run: `cargo test -p zemon-mcp sub_snapshot_requires_a_bound`
Expected: PASS.

- [ ] **Step 4: Register the tool**

```rust
#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct SubSnapshotParams {
    /// Key expression to subscribe to.
    pub key_expr: String,
    /// Stop after N messages.
    #[serde(default)] pub count: Option<u64>,
    /// Stop after this many milliseconds.
    #[serde(default)] pub duration_ms: Option<u64>,
    /// Cap each payload preview to N bytes in the JSON view.
    #[serde(default)] pub max_payload_bytes: Option<u64>,
}
```

```rust
    #[tool(description = "Subscribe and collect a bounded snapshot of messages (count and/or duration).")]
    async fn sub_snapshot(&self, Parameters(p): Parameters<SubSnapshotParams>) -> Result<CallToolResult, rmcp::ErrorData> {
        let session = self.state.session().await.map_err(to_mcp_error)?;
        let r = handlers::sub_snapshot_json(
            &session, &p.key_expr,
            p.count.map(|n| n as usize),
            p.duration_ms.map(std::time::Duration::from_millis),
            p.max_payload_bytes.map(|n| n as usize),
        ).await;
        self.finish(r).await
    }
```

- [ ] **Step 5: Manual smoke test (with zenohd + a publisher)**

Start `zenohd`; then in a third terminal run `./target/debug/zemon pub test/hello '{"n":1}'` a few times while the snapshot collects:

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"smoke","version":"0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"sub_snapshot","arguments":{"key_expr":"test/**","duration_ms":2000}}}' \
  | ./target/debug/zemon mcp
```

Expected: within ~2s a `{"count":N,"items":[...]}` result.

- [ ] **Step 6: Commit**

```bash
git add crates/zemon-mcp
git commit -m "feat(mcp): bounded sub_snapshot tool"
```

---

## Task 7: Write tool — `pub`

**Files:**
- Modify: `crates/zemon-mcp/src/handlers.rs`
- Modify: `crates/zemon-mcp/src/server.rs`

**Interfaces:**
- Produces: `handlers::pub_json(session, key_expr, value, att: Option<&str>) -> Result<String, ZemonError>`
- Consumes: `zenoh::Session::put`, `zemon_core::output::publish_accepted_json`.

- [ ] **Step 1: Confirm the CLI publish path**

Run: `sed -n '/Command::Pub/,/publish_accepted_json/p' crates/zemon-cli/src/main.rs` and mirror exactly how it builds `session.put(...)`, attaches `--att`, awaits, and calls `publish_accepted_json(key_expr, bytes)`. Reuse that logic verbatim in the handler.

- [ ] **Step 2: Add the handler**

```rust
/// `pub` tool: publish a value to a key expression (test injection).
pub async fn pub_json(
    session: &zenoh::Session,
    key_expr: &str,
    value: &str,
    att: Option<&str>,
) -> Result<String, ZemonError> {
    let mut builder = session.put(key_expr, value.to_string());
    if let Some(att) = att {
        builder = builder.attachment(att.to_string());
    }
    builder
        .await
        .map_err(|e| ZemonError::internal(format!("publish failed: {e}")))?;
    zemon_core::output::publish_accepted_json(key_expr, value.len())
        .map_err(|e| ZemonError::internal(format!("serialize pub result: {e}")))
}
```

Note: match `publish_accepted_json`'s real signature (Step 1) — it may take `(key_expr, bytes)` or a struct. Adjust the `.attachment(...)` call to the exact builder API the CLI uses.

- [ ] **Step 3: Add a serialization test for the accepted envelope**

```rust
#[test]
fn pub_accepted_envelope_shape() {
    let json = zemon_core::output::publish_accepted_json("test/x", 5).unwrap();
    let v: serde_json::Value = serde_json::from_str(&json).unwrap();
    assert_eq!(v["ok"], true);
    assert_eq!(v["key_expr"], "test/x");
    assert_eq!(v["bytes"], 5);
}
```

- [ ] **Step 4: Run the test**

Run: `cargo test -p zemon-mcp pub_accepted_envelope_shape`
Expected: PASS. (If the field names differ, correct the assertions to the real shape produced by `publish_accepted_json`.)

- [ ] **Step 5: Register the tool**

```rust
#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct PubParams {
    /// Key expression to publish to.
    pub key_expr: String,
    /// Value payload (string).
    pub value: String,
    /// Optional attachment metadata (string).
    #[serde(default)] pub att: Option<String>,
}
```

```rust
    #[tool(description = "Publish a value to a key expression (test injection; mutates the network).")]
    async fn r#pub(&self, Parameters(p): Parameters<PubParams>) -> Result<CallToolResult, rmcp::ErrorData> {
        let session = self.state.session().await.map_err(to_mcp_error)?;
        self.finish(handlers::pub_json(&session, &p.key_expr, &p.value, p.att.as_deref()).await).await
    }
```

Note: `pub` is a Rust keyword; use `r#pub` for the method name, or rename the method to `publish` and set the tool's exposed name explicitly via `#[tool(name = "pub", description = "...")]` if the resolved rmcp version supports a `name` argument. Confirm in Step 1 of Task 1 which is available and prefer an explicit `name = "pub"`.

- [ ] **Step 6: Build and test**

Run: `cargo test -p zemon-mcp && cargo build`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add crates/zemon-mcp
git commit -m "feat(mcp): pub tool"
```

---

## Task 8: Error-mapping polish, tool-list assertion, and docs

**Files:**
- Modify: `crates/zemon-mcp/src/error.rs`
- Create/Modify: `crates/zemon-mcp/tests/pure_tools.rs`
- Modify: `README.md`

**Interfaces:**
- Consumes: everything above.

- [ ] **Step 1: Assert the error mapping preserves `kind`**

Add to `error.rs` a test:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use zemon_core::error::ZemonError;

    #[test]
    fn maps_kind_into_error_data() {
        let mcp = to_mcp_error(ZemonError::invalid_input("bad".to_string()));
        // The message is preserved and the kind rides along in data.
        let rendered = serde_json::to_value(&mcp).unwrap();
        assert!(rendered.to_string().contains("invalid_input"));
    }
}
```

Note: adjust the assertion to the resolved rmcp `ErrorData` serialization (Step 1, Task 1). The invariant to preserve: the `kind` string is present in the serialized error.

- [ ] **Step 2: Run the test**

Run: `cargo test -p zemon-mcp maps_kind_into_error_data`
Expected: PASS.

- [ ] **Step 3: Add an in-memory tool-list assertion (no zenohd)**

Confirm rmcp exposes an in-memory/duplex transport for the resolved version. If it does, add to `tests/pure_tools.rs` a test that constructs the server, serves it over the in-memory transport, and asserts `tools/list` includes all 11 tool names (`keyexpr, config_show, discover, info, query, nodes, liveliness, scout, doctor, sub_snapshot, pub`). If the in-memory transport API is not readily usable, instead assert the tool count by unit-constructing `ZemonMcpServer` and inspecting its `ToolRouter` (`server.tool_router.list_all()` or the documented accessor), and document the manual stdio `tools/list` check from Task 1 Step 11 as the protocol-level verification.

```rust
#[test]
fn all_expected_tools_are_registered() {
    use std::sync::Arc;
    let state = Arc::new(zemon_mcp::state::ServerState::new(Default::default()));
    let server = zemon_mcp::server::ZemonMcpServer::new(state);
    let names = server.list_tool_names(); // add this accessor in server.rs
    for expected in ["keyexpr","config_show","discover","info","query","nodes","liveliness","scout","doctor","sub_snapshot","pub"] {
        assert!(names.iter().any(|n| n == expected), "missing tool {expected}");
    }
}
```

Add `list_tool_names` to `server.rs`:

```rust
impl ZemonMcpServer {
    pub fn list_tool_names(&self) -> Vec<String> {
        self.tool_router.list_all().into_iter().map(|t| t.name.to_string()).collect()
    }
}
```

Note: confirm the `ToolRouter` accessor name for the resolved version (`list_all`, `list`, or iterating `tools()`); use the real one.

- [ ] **Step 4: Run the tool-list test**

Run: `cargo test -p zemon-mcp all_expected_tools_are_registered`
Expected: PASS (11 tools).

- [ ] **Step 5: Document `zemon mcp` in the README**

Add a section after the TUI section:

```markdown
## MCP server (for AI agents)

`zemon mcp` runs an MCP (Model Context Protocol) server over stdio, exposing the
read-only monitoring surface (plus `pub` for test injection) as typed tools. It
binds to one network for its lifetime, resolved from the same flags/env as every
other command.

Register it with an MCP client, e.g.:

    { "command": "zemon", "args": ["mcp"] }

Tools: `discover`, `query`, `nodes`, `liveliness`, `scout`, `doctor`, `keyexpr`,
`info`, `config_show`, `sub_snapshot` (bounded: `count` and/or `duration_ms`), and
`pub`. Responses use the same JSON shapes as the CLI `--json` mode. Streaming is
bounded (snapshot); there is no server-push in this version.
```

- [ ] **Step 6: Full workspace verification**

Run: `cargo build && cargo test --workspace`
Expected: PASS (all crates, including the existing suites, remain green).

- [ ] **Step 7: Commit**

```bash
git add crates/zemon-mcp README.md
git commit -m "feat(mcp): error-kind mapping, tool-registry test, and docs"
```

---

## Self-Review Notes (spec coverage)

- Persistent lazy/reconnecting session → Task 2 (`ServerState::session`/`invalidate_session`), used by every session tool via `finish()`.
- Bounded snapshot streaming → Task 6 (`sub_snapshot`, count/duration).
- Read-only tools + `pub` → Tasks 1–7 (11 tools total; no queryable serve / capture / replay / tui).
- Reuse CLI serde shapes → all handlers use `to_collection_json` / the CLI's serializers.
- Config via `resolve_config` → Task 1 subcommand + `config_show` (Task 3).
- stdio transport → Task 1 (`serve(stdio())`).
- Error taxonomy preserved → Task 1 `to_mcp_error` + Task 8 test.
- Single binary → `zemon mcp` subcommand; `zemon-mcp` is a library only.
- Testing: pure tools unit + integration (`keyexpr`, `config_show`); serialization-contract tests for envelopes; manual stdio smoke tests for session tools; tool-registry test.

## After implementation

Open a PR from `feat/12-zemon-mcp` to `master`, referencing issue #12. Note in the PR that live streaming, `queryable serve`, `capture`/`replay`, and non-stdio transports are intentionally deferred (see the design spec's *Out of scope*).
