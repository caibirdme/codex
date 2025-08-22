# Codex CLI - Project Overview

This guide provides a comprehensive overview of the Codex CLI Rust implementation. Codex is a tool that allows users to interact with AI models to perform tasks like code editing, analysis, and automation through natural language instructions.

## What is Codex?

Codex CLI is a standalone, native executable that provides an interface to AI models for code-related tasks. It's designed to be zero-dependency and can be installed via npm or downloaded directly from GitHub releases.

## Key Features

- **Natural Language Interface**: Communicate with AI models using natural language instructions
- **Secure Execution**: Sandboxed execution of commands to prevent unintended system access
- **Model Context Protocol (MCP) Support**: Integration with external tools and services
- **Multiple Execution Modes**: Interactive TUI, headless execution, and programmatic usage
- **Flexible Configuration**: Extensive configuration options through TOML files and CLI flags
- **Cross-platform**: Works on macOS, Linux, and Windows

## Architecture Overview

The project is organized as a Cargo workspace with multiple crates, each serving a specific purpose in the overall system architecture.

## Main Components

1. **core/** - Business logic and main application functionality
2. **cli/** - Command-line interface with subcommands
3. **exec/** - Headless CLI for automation
4. **tui/** - Terminal User Interface
5. **mcp-server/** - MCP server implementation
6. **mcp-client/** - MCP client implementation
7. **protocol/** - Shared protocol definitions
8. **apply-patch/** - Patch application utilities
9. **execpolicy/** - Execution policy enforcement
10. **linux-sandbox/** - Linux sandbox implementation
11. **chatgpt/** - ChatGPT integration
12. **common/** - Common utilities and shared code
