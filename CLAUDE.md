# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

`chatgpt2md` is a Rust CLI tool that converts a ChatGPT export (ZIP or `conversations.json`) into organized Markdown files, builds a full-text search index with Tantivy, and runs an MCP server so Claude can search and retrieve that history natively.

## Commands

```bash
# Build debug binary
cargo build

# Build optimized release binary (output: ./target/release/chatgpt2md)
cargo build --release

# Run tests
cargo test

# Lint
cargo clippy

# Format
cargo fmt

# Check without building
cargo check
```

There are no tests in the repo yet; `cargo test` will compile but find nothing to run.

Releases are triggered by pushing a `v*` tag. The CI workflow (`.github/workflows/release.yml`) cross-compiles for macOS (arm64 + x86_64), Linux (x86_64), and Windows (x86_64 MSVC), then publishes a GitHub Release with the tarballs/zips.

## Architecture

The project has five source files with a clear single-responsibility split:

### `src/main.rs`
CLI entry point using `clap`. Implements a compatibility shim: if the first argument looks like a file path rather than a subcommand name, it inserts `"convert"` before it so `chatgpt2md export.zip` works as shorthand for `chatgpt2md convert export.zip`.

### `src/convert.rs`
Handles the `convert` subcommand. Accepts a `.zip` or `.json` input and:
1. Extracts `conversations.json` from the ZIP (or reads it directly)
2. Walks the conversation tree — ChatGPT stores conversations as a node graph with `mapping` + `current_node`; the code follows parent pointers from `current_node` to the root and reverses the result to get chronological order
3. Renders each conversation to Markdown with YAML frontmatter (`title`, `source`, `created`, `updated`, `message_count`)
4. Organizes output into `YYYY/MM/` subdirectories (or flat, or `undated/`)
5. Collects `ConversationMeta` structs and passes them to `index::build_index`

Content extraction handles four content types: `text`, `code`, `multimodal_text`, and gracefully skips unknown types. Images become `*[Image]*`, file attachments become `*[File: name]*`.

### `src/index.rs`
Wraps [Tantivy](https://github.com/quickwit-oss/tantivy) to build and query a full-text search index. The schema stores `id`, `title`, `date`, `year`, `month`, `message_count`, `file_path` (all stored), plus `body` (indexed but not stored, to save space).

- `build_index` — always recreates the index from scratch (removes existing directory first)
- `SearchIndex::search` — full-text query over `title` + `body` fields
- `SearchIndex::get_by_id` — exact `TermQuery` on the `id` field
- `SearchIndex::list_by_date` — `BooleanQuery` with `Must` clauses on `year` and/or `month` fields; falls back to `AllQuery` with no filters

### `src/mcp.rs`
Implements the MCP server using the `rmcp` crate (with `#[tool_router]` and `#[tool]` macros). Exposes three tools to Claude:

| Tool | What it does |
|---|---|
| `search_conversations` | Full-text search, up to 100 results |
| `get_conversation` | Reads the `.md` file from disk by ID (tries absolute path, falls back to relative-to-chats-dir) |
| `list_conversations` | Date-filtered listing, up to 200 results |

The server runs on stdio transport via `tokio` and blocks on `service.waiting()`.

### `src/install.rs`
Handles the `install` subcommand. For **Claude Desktop**: reads/creates the JSON config file at the OS-appropriate path and merges in the `chatgpt-history` MCP server entry. For **Claude Code**: shells out to `claude mcp add --transport stdio --scope user ...`. Both use the resolved absolute paths of the binary, index, and chats directory.

## Key conventions

- All user-facing output goes to `stderr` (`eprintln!`); stdout is reserved for the MCP server's JSON-RPC messages.
- `std::process::exit(1)` is used for unrecoverable errors in the CLI path (no `anyhow` or similar).
- The Tantivy index lives at `<output_dir>/.index/` by default; the chats and index paths are always passed explicitly to `serve` and `install`.
- `body` is not stored in the index (only indexed) to reduce disk footprint — so full conversation text is always read from the `.md` file on disk.
