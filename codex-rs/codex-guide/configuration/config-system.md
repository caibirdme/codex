# Configuration System

The Codex CLI configuration system is designed to be flexible and powerful, supporting multiple configuration sources with clear precedence rules.

## Configuration Sources

### 1. Default Values
Built-in defaults for all configuration options.

### 2. Configuration File (`~/.codex/config.toml`)
Primary configuration file using TOML format.

### 3. CLI Overrides
Command-line arguments can override any configuration value.

### 4. Environment Variables
Specific environment variables can override configuration.

### 5. Profiles
Named configuration sets that can be switched at runtime.

## Configuration Structure

### Top-Level Configuration
```toml
model = "o3"
model_provider = "openai"
approval_policy = "untrusted"
sandbox_mode = "read-only"
disable_response_storage = false
```

### Model Providers
```toml
[model_providers.openai]
name = "OpenAI"
base_url = "https://api.openai.com/v1"
env_key = "OPENAI_API_KEY"
wire_api = "chat"

[model_providers.ollama]
name = "Ollama"
base_url = "http://localhost:11434/v1"
```

### Profiles
```toml
[profiles.fast]
model = "gpt-4o-mini"
approval_policy = "never"

[profiles.safe]
model = "o3"
approval_policy = "on-failure"
sandbox_mode = "workspace-write"
```

### MCP Servers
```toml
[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem"]
env = { "ALLOWED_DIRS" = "/home/user/projects" }

[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]
env = { "GITHUB_TOKEN" = "your-token" }
```

### Sandbox Configuration
```toml
[sandbox_workspace_write]
writable_roots = ["/tmp", "/var/tmp"]
network_access = false
exclude_tmpdir_env_var = false
exclude_slash_tmp = false
```

### Shell Environment Policy
```toml
[shell_environment_policy]
inherit = "core"
exclude = ["AWS_*", "AZURE_*"]
set = { CI = "1" }
include_only = ["PATH", "HOME", "USER"]
```

## Configuration Precedence

1. **CLI Arguments** (highest priority)
   - `--model o3`
   - `--config key=value`
   - `--profile fast`

2. **Profile Settings**
   - Settings from active profile

3. **Configuration File**
   - Settings from `~/.codex/config.toml`

4. **Default Values** (lowest priority)
   - Built-in defaults

## Environment Variables

### CODEX_HOME
Specifies the directory for Codex configuration and state files.
```bash
export CODEX_HOME=/custom/path/.codex
```

### OPENAI_API_KEY
API key for OpenAI services.
```bash
export OPENAI_API_KEY=sk-...
```

### Model Provider Environment Variables
Each provider can specify its own environment variable for API keys:
- `OPENAI_API_KEY`
- `AZURE_OPENAI_API_KEY`
- `MISTRAL_API_KEY`
- etc.

## Configuration Validation

All configuration is validated at runtime:
- Model names are checked against known models
- URLs are validated for format
- File paths are checked for existence
- Environment variables are verified

## Dynamic Configuration Updates

Some configuration can be updated during runtime:
- Model selection
- Approval policies
- Sandbox modes

## Configuration Examples

### Basic Configuration
```toml
model = "o3"
approval_policy = "on-failure"
sandbox_mode = "workspace-write"
```

### Advanced Configuration
```toml
model = "o3"
model_provider = "openai"
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[model_providers.openai]
name = "OpenAI"
base_url = "https://api.openai.com/v1"
env_key = "OPENAI_API_KEY"
wire_api = "chat"
request_max_retries = 4
stream_max_retries = 10
stream_idle_timeout_ms = 300000

[profiles.fast]
model = "gpt-4o-mini"
approval_policy = "never"

[profiles.safe]
model = "o3"
approval_policy = "on-failure"
sandbox_mode = "read-only"

[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem"]
env = { "ALLOWED_DIRS" = "/home/user/projects" }

[history]
persistence = "save-all"
