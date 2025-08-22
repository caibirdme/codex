# Execution Flow

This document describes the complete execution flow of Codex CLI from startup to command completion.

## High-Level Flow

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌─────────────┐
│   Startup   │───│ Configuration│───│   Session   │───│  Execution  │
│   & Init    │   │    Load      │   │   Setup     │   │   & Output  │
└─────────────┘    └──────────────┘    └─────────────┘    └─────────────┘
```

## Detailed Execution Flow

### 1. Startup Phase

#### Process Initialization
```rust
// cli/src/main.rs
fn main() -> Result<()> {
    let args = Cli::parse();
    init_logging()?;
    run_command(args)
}
```

#### Argument Parsing
- Parse CLI arguments using `clap`
- Determine execution mode (TUI, exec, debug, etc.)
- Set up configuration overrides

#### Logging Setup
- Initialize tracing subscriber
- Configure log levels based on environment
- Set up log rotation

### 2. Configuration Loading

#### Configuration Resolution
```rust
// core/src/config.rs
let config = Config::load_with_cli_overrides(
    cli_overrides,
    config_overrides
)?;
```

#### Configuration Sources (in order)
1. **Defaults**: Built-in default values
2. **Config File**: `~/.codex/config.toml`
3. **CLI Overrides**: `--config key=value`
4. **Environment**: `CODEX_HOME`, `OPENAI_API_KEY`

#### Profile Resolution
- Check for `--profile` argument
- Load profile configuration
- Merge with base configuration

### 3. Session Setup

#### Working Directory
```rust
// Resolve working directory
let cwd = match args.cwd {
    Some(path) => path,
    None => env::current_dir()?,
};
```

#### Environment Setup
- Set up shell environment policy
- Filter sensitive environment variables
- Configure sandbox parameters

#### MCP Server Initialization
- Parse MCP server configurations
- Initialize connection managers
- Cache tool/resource information

### 4. Model Provider Setup

#### Provider Selection
- Resolve model provider based on configuration
- Validate API credentials
- Set up connection parameters

#### Model Configuration
- Set context window size
- Configure reasoning parameters
- Set up streaming options

### 5. Conversation Initialization

#### History Loading
- Load conversation history from `~/.codex/history.jsonl`
- Apply retention policies
- Set up conversation context

#### Project Documentation
- Search for `AGENTS.md` in workspace
- Load project-specific instructions
- Set up context window for documentation

### 6. Execution Modes

#### Interactive Mode (TUI)
```rust
// tui/src/main.rs
let app = App::new(config);
app.run()?;
```

**Flow**:
1. Initialize terminal UI
2. Set up event loop
3. Handle user input
4. Process AI responses
5. Manage command execution

#### Headless Mode (exec)
```rust
// exec/src/main.rs
let processor = EventProcessor::new(config);
processor.run(prompt)?;
```

**Flow**:
1. Process input prompt
2. Send to AI model
3. Handle streaming response
4. Execute commands
5. Output results

### 7. Command Execution Flow

#### Command Generation
1. **Prompt Processing**: AI analyzes user prompt
2. **Context Building**: Include relevant files and history
3. **Command Generation**: AI suggests commands to execute

#### Security Validation
```rust
// core/src/safety.rs
fn validate_command(cmd: &str) -> SafetyResult {
    // Check against known dangerous patterns
    // Validate file paths
    // Check network access requirements
}
```

#### Approval Process
1. **Policy Check**: Check approval policy
2. **User Prompt**: Display command for approval
3. **User Decision**: Approve, reject, or modify
4. **Execution**: Run command in sandbox

#### Sandbox Execution
```rust
// core/src/exec.rs
let result = match sandbox_policy {
    SandboxPolicy::ReadOnly => execute_readonly(cmd),
    SandboxPolicy::WorkspaceWrite => execute_workspace_write(cmd),
    SandboxPolicy::DangerFullAccess => execute_unrestricted(cmd),
};
```

### 8. Output Processing

#### Stream Processing
- Real-time output streaming
- Progress indicators
- Error handling and recovery

#### Result Formatting
- **TUI**: Rich terminal formatting
- **Exec**: JSON or plain text output
- **Logging**: Structured logging for debugging

### 9. Cleanup and Persistence

#### History Management
- Save conversation to history file
- Apply retention policies
- Compress old conversations

#### Resource Cleanup
- Close MCP connections
- Clean up temporary files
- Reset terminal state

## Error Handling Flow

### 1. Configuration Errors
```rust
match Config::load() {
    Ok(config) => continue,
    Err(e) => {
        log_error!(e);
        show_user_friendly_error(e);
        exit(1);
    }
}
```

### 2. Network Errors
- **Retry Logic**: Exponential backoff for API calls
- **Fallback**: Switch to alternative providers
- **User Notification**: Clear error messages

### 3. Command Execution Errors
- **Sandbox Errors**: Handle permission denials
- **Command Failures**: Provide meaningful error messages
- **Recovery**: Suggest alternative approaches

### 4. MCP Server Errors
- **Connection Failures**: Graceful degradation
- **Tool Errors**: Fallback to local tools
- **Timeout Handling**: Configurable timeouts

## Performance Optimization

### 1. Caching
- **Configuration**: Cache parsed configuration
- **MCP Tools**: Cache tool definitions
- **Model Responses**: Cache for similar prompts

### 2. Streaming
- **Real-time Output**: Immediate feedback to users
- **Progress Indicators**: Show ongoing operations
- **Partial Results**: Display intermediate results

### 3. Resource Management
- **Memory Limits**: Prevent memory exhaustion
- **Connection Pooling**: Reuse connections
- **Garbage Collection**: Clean up unused resources

## Debugging Execution Flow

### 1. Debug Logging
```bash
# Enable debug logging
RUST_LOG=debug codex

# Enable specific module logging
RUST_LOG=codex_core=debug,codex_exec=trace codex
```

### 2. Execution Tracing
```bash
# Trace command execution
RUST_LOG=codex::exec=trace codex exec "your command"

# Trace MCP interactions
RUST_LOG=codex::mcp=debug codex
```

### 3. Performance Profiling
```bash
# Profile execution time
time codex exec "complex operation"

# Memory usage
/usr/bin/time -v codex exec "memory intensive task"
```

## Extension Points

### 1. Custom Model Providers
- Implement `ModelProviderInfo` trait
- Add to configuration system
- Register with provider registry

### 2. Custom MCP Servers
- Implement MCP protocol
- Add to configuration
- Test with MCP inspector

### 3. Custom Execution Policies
- Implement approval policies
- Add to policy system
- Configure in settings

## Monitoring and Observability

### 1. Metrics Collection
- **Command Execution**: Success/failure rates
- **API Usage**: Model provider statistics
- **Performance**: Response times and throughput

### 2. Health Checks
- **MCP Server Health**: Connection status
- **Model Provider Health**: API availability
- **System Health**: Resource usage

### 3. Alerting
- **Error Rates**: High error rate alerts
- **Performance**: Slow response alerts
- **Security**: Policy violation alerts
