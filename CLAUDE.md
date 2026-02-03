# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Go-based Model Context Protocol (MCP) server for Slack workspaces. It provides tools for reading messages, searching, and optionally posting to Slack channels via multiple transport protocols (stdio, SSE, HTTP).

## Build & Development Commands

```bash
# Build for current platform
make build

# Run tests (unit tests only, filtered by ".*Unit.*" in name)
make test

# Run integration tests (filtered by ".*Integration.*" in name)
make test-integration

# Format code
make format

# Tidy go.mod
make tidy

# Download dependencies
make deps

# Build for all platforms (darwin/linux/windows × amd64/arm64)
make build-all-platforms

# Clean build artifacts
make clean
```

## Running the Server

```bash
# Stdio transport (default)
./build/slack-mcp-server -transport stdio

# SSE transport
SLACK_MCP_XOXP_TOKEN=your_token ./build/slack-mcp-server -transport sse

# HTTP transport
SLACK_MCP_XOXP_TOKEN=your_token ./build/slack-mcp-server -transport http
```

## Architecture

### Package Structure

- **cmd/slack-mcp-server/main.go** - Entry point, handles CLI flags (`-transport stdio|sse|http`), initializes provider and server, starts background cache watchers
- **pkg/provider/** - Core API abstraction layer (`api.go`), manages Slack API interactions, user/channel caching
- **pkg/provider/edge/** - Advanced Slack client implementation (derived from rusq/slackdump), handles DM/MPIM support, fast message fetching, search
- **pkg/handler/** - MCP tool handlers:
  - `conversations.go` - Tools: `conversations_history`, `conversations_replies`, `conversations_add_message`, `conversations_search_messages`, `channels_list`, `reactions_add`, `attachment_get_data`
  - `channels.go` - CSV resource generation for channels directory
- **pkg/server/** - MCP server setup, tool registration, middleware
- **pkg/server/auth/** - Bearer token authentication for SSE/HTTP transports
- **pkg/transport/** - HTTP client with custom TLS fingerprinting (uTLS) for Enterprise Slack environments
- **pkg/limiter/** - Rate limiting per Slack API tiers (Tier2, Tier2boost, Tier3)

### Authentication Modes

Three authentication methods (environment variables):
1. **xoxp** - User OAuth token (`SLACK_MCP_XOXP_TOKEN`)
2. **xoxb** - Bot token (`SLACK_MCP_XOXB_TOKEN`) - limited access, no search
3. **xoxc/xoxd** - Browser session tokens (`SLACK_MCP_XOXC_TOKEN` + `SLACK_MCP_XOXD_TOKEN`) - stealth mode

### Key Design Patterns

- Provider pattern: `ApiProvider` abstracts all Slack API operations
- Background cache warming on startup (users, channels)
- Tool middleware for logging and authentication
- Rate limiting per Slack tier specifications
- Custom TLS fingerprinting (`pkg/transport/`) for bypassing enterprise firewalls

### Test Naming Convention

Tests use naming convention for filtering:
- Unit tests: function names must contain "Unit" (e.g., `TestFoo_Unit`)
- Integration tests: function names must contain "Integration" (e.g., `TestBar_Integration`)

## Key Environment Variables

| Variable | Description |
|----------|-------------|
| `SLACK_MCP_XOXP_TOKEN` | User OAuth token |
| `SLACK_MCP_XOXB_TOKEN` | Bot token (limited) |
| `SLACK_MCP_XOXC_TOKEN` / `SLACK_MCP_XOXD_TOKEN` | Browser session tokens |
| `SLACK_MCP_ADD_MESSAGE_TOOL` | Enable message posting (disabled by default) |
| `SLACK_MCP_HOST` / `SLACK_MCP_PORT` | Server binding (default: 127.0.0.1:13080) |
| `SLACK_MCP_LOG_LEVEL` | debug, info, warn, error, panic, fatal |
| `SLACK_MCP_CUSTOM_TLS` | Enable uTLS fingerprinting for Enterprise |
| `SLACK_MCP_GOVSLACK` | FedRAMP-compliant mode |
