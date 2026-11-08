# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

MCP Chat: a terminal chat app that talks to Claude through the Anthropic API and gets documents, prompts and tools from a local MCP server over stdio.

## Commands

```bash
uv pip install -e .          # install dependencies (uv venv must exist)
uv run main.py               # start the interactive chat (spawns mcp_server.py itself)
uv run main.py extra_srv.py  # also connect to additional MCP server scripts (each launched with `uv run`)
uv run mcp_server.py         # run the server alone; use this to surface import/startup errors
uv run mcp_client.py         # quick check: connects to the server and prints its tools
```

There are no tests, linting or type checks.

`.env` must define `ANTHROPIC_API_KEY` and `CLAUDE_MODEL` (`main.py` asserts on both). `USE_UV=1` makes `main.py` spawn the doc server with `uv run`; `USE_UV=0` uses `python`.

## Architecture

The flow spans these files:

- **`main.py`**: loads `.env`, then opens one `MCPClient` per server inside an `AsyncExitStack`. The client for `mcp_server.py` is `doc_client`. All clients go into a `clients` dict that is handed to `CliChat` and `CliApp`.
- **`mcp_server.py`**: a `FastMCP` server with in-memory `docs` (a dict, so edits are lost on restart). It exposes:
  - tools `read_doc_contents` and `edit_document`
  - resources `docs://documents` (JSON list of ids) and `docs://documents/{doc_id}` (text)
  - prompts `format` and `summarize`, each taking `doc_id`
- **`mcp_client.py`**: `MCPClient` wraps `stdio_client` + `ClientSession` and is used as an async context manager. `read_resource` parses JSON when the mime type is `application/json` and otherwise returns the text.
- **`core/chat.py`**: `Chat.run` is the agent loop. It calls Claude with tools gathered from **every** client, and on `stop_reason == "tool_use"` runs the tools and loops.
- **`core/tools.py`**: `ToolManager` converts MCP tools to Anthropic tool schemas and sends each `tool_use` block to the first client that has a tool with that name.
- **`core/cli_chat.py`**: `CliChat` handles the two kinds of special input:
  - `@doc_id` mentions: the document content is read via the resource and inlined into the user message.
  - `/command doc_id`: fetches the MCP prompt named `command` and appends its messages to the conversation.
- **`core/cli.py`**: the `prompt_toolkit` UI. Tab completion for `/` uses the server's prompt list, and completion for `@` uses the resource doc ids. Both lists are loaded in `CliApp.initialize`.
- **`core/claude.py`**: a thin wrapper over `Anthropic().messages.create`.

So adding a new `@mcp.prompt` in `mcp_server.py` automatically creates a new `/command` with autocomplete. Prompts must accept a `doc_id` argument, because `CliChat._process_command` always passes `{"doc_id": ...}`.

## Gotchas

- If the server process crashes at startup (for example from a bad import), `uv run main.py` **hangs silently** while waiting for the MCP handshake. Run `uv run mcp_server.py` directly to see the traceback.
- Prompt message classes come from `mcp.server.fastmcp.prompts.base` (not `mcp.server.base`).
- On Windows, "Event loop is closed" / "I/O operation on closed pipe" warnings at exit are harmless asyncio cleanup noise.
