# CLI Interfaces

Codex CLI provides multiple command-line interfaces for different use cases, all accessible through the main `codex` command.

## Main CLI (`cli/`)

The main CLI provides subcommands for different modes of operation.

### Basic Usage
```bash
codex [OPTIONS] [SUBCOMMAND]
```

### Common Options
- `--model <MODEL>`: Specify AI model to use
- `--profile <PROFILE>`: Use configuration profile
- `--sandbox <MODE>`: Set sandbox mode (read-only, workspace-write, danger-full-access)
- `--cd <PATH>`: Set working directory
- `--config <KEY=VALUE>`: Override configuration values

### Subcommands

#### 1. Interactive Mode (Default)
```bash
codex
```
Launches the interactive TUI interface.

#### 2. Headless Execution
```bash
codex exec "your prompt here"
```
Runs Codex non-interactively until completion.

#### 3. Debug Mode
```bash
# macOS
codex debug seatbelt [--full-auto] [COMMAND...]

# Linux
codex debug landlock [--full-auto] [COMMAND...]
```
Tests sandbox behavior for specific commands.

#### 4. Shell Completions
```bash
codex completion bash
codex completion zsh
codex completion fish
```
Generates shell completion scripts.

#### 5. MCP Server Mode
```bash
codex mcp
```
Runs Codex as an MCP server.

## TUI Interface (`tui/`)

The Terminal User Interface provides an interactive experience with:
- Real-time chat interface
- File search with `@` prefix
- Command approval prompts
- Progress indicators
- Conversation history

### TUI Features
- **File Search**: Type `@` to trigger fuzzy filename search
- **Command History**: Navigate previous commands with arrow keys
- **Approval Prompts**: Interactive approval for commands
- **Status Indicators**: Real-time status of operations

## Headless CLI (`exec/`)

The headless CLI is designed for automation and scripting.

### Usage
```bash
codex-exec [OPTIONS] [PROMPT]
```

### Options
- `--json`: Output in JSON format
- `--model <MODEL>`: Specify AI model
- `--sandbox <MODE>`: Set sandbox mode
- `--approval-policy <POLICY>`: Set approval policy

### Examples
```bash
# Basic usage
codex-exec "Update all dependencies in package.json"

# JSON output for scripting
codex-exec --json "List all Python files in the project"

# With specific model
codex-exec --model o3 "Refactor the authentication module"
```

## Configuration CLI

### View Current Configuration
```bash
codex config show
```

### Edit Configuration
```bash
# Opens editor for ~/.codex/config.toml
codex config edit
```

### Validate Configuration
```bash
codex config validate
```

## Environment Setup

### Installation
```bash
# Via npm
npm i -g @openai/codex@native

# Via GitHub releases
# Download from https://github.com/openai/codex/releases
```

### Basic Configuration
```bash
# Set up configuration directory
mkdir -p ~/.codex

# Create basic config
cat > ~/.codex/config.toml << EOF
model = "o3"
approval_policy = "on-failure"
sandbox_mode = "workspace-write"
EOF
```

### Environment Variables
```bash
# Set API key
export OPENAI_API_KEY=sk-your-key-here

# Custom config directory
export CODEX_HOME=/custom/path/.codex
```

## Usage Examples

### Interactive Development
```bash
# Start interactive session
codex

# Start in specific directory
codex --cd /path/to/project

# Use specific profile
codex --profile fast
```

### Automation Scripts
```bash
#!/bin/bash
# update-deps.sh
PROJECT_DIR="/path/to/project"
codex-exec --cd "$PROJECT_DIR" --json "Update all dependencies and run tests"
```

### CI/CD Integration
```bash
# In CI pipeline
codex-exec --approval-policy never --sandbox danger-full-access \
  "Generate changelog from git commits"
```

## Advanced CLI Usage

### Batch Processing
```bash
# Process multiple files
find . -name "*.py" -exec codex-exec --json "Review {} for security issues" \;
```

### Integration with Other Tools
```bash
# Pipe output to other tools
codex-exec "Generate test cases for src/main.py" | tee tests/test_main.py
```

### Custom Profiles
```bash
# Create profile for specific use case
cat > ~/.codex/config.toml << EOF
[profiles.security-audit]
model = "o3"
approval_policy = "never"
sandbox_mode = "read-only"

[profiles.development]
model = "gpt-4o"
approval_policy = "on-failure"
sandbox_mode = "workspace-write"
EOF

# Use profile
codex --profile security-audit "Audit for security vulnerabilities"
