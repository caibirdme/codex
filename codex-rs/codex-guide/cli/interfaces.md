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
```

## Binary Renaming Behavior

Codex is designed to be compiled as a single binary that can behave differently based on how it's invoked. This is achieved through the "arg0 trick" which allows a single binary to serve multiple purposes.

### How It Works

The main `codex` binary can be renamed to different names and will behave differently based on its invocation name:

1. **`codex`** - Main CLI interface with all subcommands
2. **`codex-linux-sandbox`** - Dedicated sandbox execution binary
3. **`codex-apply-patch`** - Dedicated patch application binary

### Binary Behaviors

#### Main `codex` Binary
When invoked as `codex`, it provides the full CLI interface with:
- Interactive TUI mode
- Headless execution (`exec` subcommand)
- Debug modes (`debug` subcommand)
- MCP server mode (`mcp` subcommand)
- Configuration management
- All other CLI subcommands

#### `codex-linux-sandbox` Binary
When invoked as `codex-linux-sandbox`, it bypasses the normal CLI dispatch and directly executes the Linux sandbox functionality:
```bash
# This is equivalent to:
# codex debug landlock [command]
# But runs with different security context
```

#### `codex-apply-patch` Binary
When invoked as `codex-apply-patch`, it handles patch application directly:
```bash
# This is equivalent to:
# codex exec --model o3 "apply_patch [patch-content]"
# But optimized for patch application
```

### Practical Usage

This design allows for:
1. **Reduced binary size** - Single binary with multiple behaviors
2. **Better security isolation** - Different execution contexts for different purposes
3. **Simplified deployment** - One binary to manage instead of multiple
4. **Optimized performance** - Direct execution paths for specialized tasks

### Example Deployment

```bash
# Build the main binary
cargo build --release

# Create symbolic links for different behaviors
ln -s target/release/codex codex-linux-sandbox
ln -s target/release/codex codex-apply-patch

# Use different behaviors
./codex                    # Full CLI interface
./codex-linux-sandbox      # Linux sandbox execution
./codex-apply-patch        # Patch application
```

Note: This behavior is primarily intended for internal use and deployment scenarios. Users typically interact with the main `codex` binary.
