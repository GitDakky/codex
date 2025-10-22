# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Codex CLI is a coding agent that runs locally. This is a monorepo containing:
- **Rust workspace** (`codex-rs/`): Native CLI implementation
- **TypeScript SDK** (`sdk/typescript/`): Embeddable SDK for workflows and apps
- **Documentation** (`docs/`): User-facing documentation

## Development Commands

### Prerequisites
- Node.js 22+ and pnpm 10.8.1+
- Rust (version specified in `rust-toolchain.toml`)
- `just` command runner
- `cargo-nextest` for faster test runs: `cargo install cargo-nextest`
- `cargo-insta` for snapshot tests: `cargo install cargo-insta`

### Common Commands

#### Rust (in `codex-rs/` directory)

```bash
# Run Codex CLI
just codex [args]
cargo run --bin codex -- [args]

# Run headless exec mode
just exec [args]

# Run specific binary
cargo run -p codex-tui [args]

# Format (run automatically after changes)
just fmt

# Lint and fix issues (scope with -p when possible)
just fix -p codex-tui
just fix  # workspace-wide (slower)

# Run tests for specific package
cargo test -p codex-tui
cargo nextest run -p codex-core

# Run all tests
just test
cargo nextest run --no-fail-fast

# Snapshot test workflow
cargo test -p codex-tui                    # Run tests to generate snapshots
cargo insta pending-snapshots -p codex-tui # Check pending snapshots
cargo insta show -p codex-tui <file.snap>  # Preview specific snapshot
cargo insta accept -p codex-tui            # Accept all new snapshots
```

#### TypeScript (in repository root or sdk/typescript)

```bash
# Install dependencies
pnpm install

# Format
pnpm format        # Check
pnpm format:fix    # Auto-fix

# SDK-specific (in sdk/typescript/)
pnpm build         # Build SDK
pnpm test          # Run tests
pnpm lint          # Lint code
pnpm lint:fix      # Auto-fix lint issues
```

#### Testing Strategy

1. **Project-specific tests**: Run tests for the specific package you changed (e.g., `cargo test -p codex-tui`)
2. **Shared crate changes**: If changes are in `core`, `common`, or `protocol`, run full test suite: `cargo test --all-features`
3. **Approval**: Ask user before running workspace-wide `just fix` or complete test suite

## Architecture

### Rust Crates Structure

The `codex-rs/` workspace is organized into focused crates:

- **`core/`**: Business logic for Codex agent. Main state machine, tool execution, and conversation management.
  - `codex.rs`: Core agent logic and state machine
  - `config.rs`: Configuration parsing and management
  - `tools/`: Tool implementations (bash, file operations, etc.)
  - `mcp_connection_manager.rs`: MCP server integration
  - `auth.rs`: Authentication and session management

- **`cli/`**: Main CLI multitool that provides all Codex commands via subcommands

- **`tui/`**: Full-screen terminal UI built with [Ratatui](https://ratatui.rs/)
  - Uses snapshot tests extensively to validate rendered output
  - See `tui/styles.md` for styling conventions

- **`exec/`**: Headless CLI for automation (`codex exec`)

- **`protocol/`**: Protocol definitions for communication between components

- **`mcp-server/`**: Allows Codex to run as an MCP server for other clients
- **`rmcp-client/`**: MCP client implementation for connecting to MCP servers

- **Supporting crates**:
  - `backend-client/`: API client for Codex backend
  - `git-tooling/`: Git operations and integration
  - `apply-patch/`: Patch application logic
  - `linux-sandbox/`, `process-hardening/`: Sandboxing implementations
  - `chatgpt/`: ChatGPT-specific authentication

### TypeScript SDK

The SDK (`sdk/typescript/`) wraps the bundled Codex binary and exchanges JSONL events over stdin/stdout. It provides:
- Thread management for multi-turn conversations
- Streaming event APIs
- Structured output via JSON schema
- Image attachment support

## Coding Conventions

### Rust

All crate names are prefixed with `codex-` (e.g., the `core` folder's crate is `codex-core`).

**Code Style:**
- Always inline format! args when possible (`format!("{foo}")` not `format!("{}", foo)`)
- Collapse if statements when reasonable
- Use method references over closures when possible
- Prefer comparing entire objects in tests rather than field-by-field
- Do not use unsigned integers even if the number cannot be negative
- Always use `pretty_assertions::assert_eq` in tests for clearer diffs

**Never modify:**
- Code related to `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR` or `CODEX_SANDBOX_ENV_VAR`
- These are set by the sandbox environment and tests check for them to skip when appropriate

### TUI Styling (Ratatui)

Follow conventions from `codex-rs/tui/styles.md`:

**Colors:**
- Default foreground for most text (use `.reset()` to restore)
- Cyan: user input tips, selection, status indicators
- Green: success and additions
- Red: errors, failures, deletions
- Magenta: Codex-related content
- Avoid: blue, yellow, black, white, custom colors

**Styling helpers:**
- Use Stylize trait: `"text".dim()`, `"text".bold()`, `"text".cyan()`, etc.
- Basic spans: `"text".into()`
- Chaining: `url.cyan().underlined()`
- Computed styles: `Span::styled(text, style)` is OK for runtime-computed styles
- Prefer compact forms that stay on one line after rustfmt

**Text wrapping:**
- Use `textwrap::wrap` for plain strings
- Use helpers in `tui/src/wrapping.rs` for ratatui Lines (e.g., `word_wrap_lines`)
- Use `prefix_lines` helper from `line_utils` for prefixing lines

### Snapshot Tests

This repo uses `insta` for snapshot testing, especially in TUI. When making intentional UI changes:
1. Run tests to generate `*.snap.new` files
2. Review changes by reading the `.snap.new` files directly
3. Accept only if changes are correct: `cargo insta accept -p codex-tui`

### Integration Tests (core)

Use utilities in `core_test_support::responses`:
- `mount_sse*` helpers return `ResponseMock` for asserting outbound POST bodies
- Use `ResponseMock::single_request()` or `ResponseMock::requests()`
- `ResponsesRequest` exposes helpers: `body_json`, `input`, `function_call_output`, etc.
- Build SSE payloads with `ev_*` constructors

Example pattern:
```rust
let mock = responses::mount_sse_once(&server, responses::sse(vec![
    responses::ev_response_created("resp-1"),
    responses::ev_function_call(call_id, "shell", &serde_json::to_string(&args)?),
    responses::ev_completed("resp-1"),
])).await;

codex.submit(Op::UserTurn { ... }).await?;

let request = mock.single_request();
// Assert using request helpers
```

## Configuration

Codex uses `~/.codex/config.toml` for configuration. Key areas:
- `sandbox_mode`: Controls sandboxing level (read-only, workspace-write, danger-full-access)
- `mcp_servers`: MCP server connections
- `notify`: Desktop notifications on turn completion
- Model selection and API settings

See `docs/config.md` for complete reference.

## MCP (Model Context Protocol)

Codex supports MCP in two ways:
1. **As MCP client**: Connects to MCP servers on startup (configured in `config.toml`)
2. **As MCP server**: Run `codex mcp-server` to expose Codex to other MCP clients

## Sandboxing

Platform-specific sandboxing:
- **macOS**: Seatbelt (`codex-rs/core/src/seatbelt.rs`)
- **Linux**: Landlock + Seccomp (`codex-rs/linux-sandbox/`)

Test sandboxed commands: `codex sandbox macos [COMMAND]` or `codex sandbox linux [COMMAND]`

## Contributing Notes

- External contributions are primarily accepted for bugs and security fixes
- New features require proposal via issue and team approval first
- Run all checks before opening PR: `cargo test && cargo clippy --tests && cargo fmt -- --config imports_granularity=Item`
- Update docs in `docs/` folder if changing user-facing behavior
- All contributors must sign the CLA (automated via PR comment)

## Package Manager

This project uses **pnpm** (not npm). Node.js 22+ required.
- Install: `npm install -g pnpm@10.8.1` or `corepack enable && corepack prepare pnpm@10.8.1 --activate`
- Workspace commands: `pnpm --filter <package> <command>`
- Run in all packages: `pnpm -r <command>`

See `PNPM.md` for migration details.
