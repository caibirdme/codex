# Sessions in Codex

Sessions in Codex represent distinct, isolated interactions with the AI system. Understanding how sessions work is important for grasping how Codex manages state and configuration.

## What is a Session?

A **Session** in Codex is a complete interaction context that maintains:
- Configuration settings (model, approval policies, sandbox settings)
- Working directory context
- Conversation history within that session
- MCP server connections
- User authentication state
- Session-specific state like approved commands

## Session Lifecycle

### Creation
Sessions are created when:
1. The Codex CLI starts up (via `Codex::spawn()`)
2. Each CLI invocation creates a new session
3. TUI and exec interfaces both create their own sessions

### Destruction
Sessions are destroyed when:
1. The CLI process terminates
2. An explicit shutdown operation occurs
3. The session reaches its natural end

## Session ID Purpose and Usage

### Why Session IDs Exist
The session ID serves several important purposes:

1. **Identification**: Uniquely identifies each session for tracking and debugging
2. **History Management**: Associates conversation history with specific sessions
3. **State Isolation**: Ensures different sessions maintain completely separate states
4. **Persistence**: Allows for session resumption and replay capabilities
5. **Logging**: Enables correlation of events and logs with specific sessions

### Where Session IDs Are Used

#### 1. Event System
Session IDs are included in events to associate them with the correct session:
```rust
// In SessionConfiguredEvent
SessionConfiguredEvent {
    session_id,  // UUID identifying this session
    model,
    history_log_id,
    history_entry_count,
}
```

#### 2. History Recording
Session IDs are used to store conversation history in separate files:
```rust
// In rollout recording
let session_id = Uuid::new_v4();  // Generated for each session
RolloutRecorder::new(&config, session_id, user_instructions.clone())
```

#### 3. Model Client Identification
The session ID is passed to the model client to maintain session context:
```rust
let client = ModelClient::new(
    config.clone(),
    auth.clone(),
    provider.clone(),
    model_reasoning_effort,
    model_reasoning_summary,
    session_id,  // Passed to model client
);
```

#### 4. History Lookup
Session IDs enable retrieval of specific session histories:
```rust
// When looking up history entries
let entry_opt = tokio::task::spawn_blocking(move || {
    crate::message_history::lookup(log_id, offset, &config)
})
```

## Session Management in Different Interfaces

### TUI Interface
The TUI creates a single session per application run:
```rust
// In tui/src/app.rs
let conversation_manager = Arc::new(ConversationManager::default());
// This creates one session that persists throughout the TUI session
```

### Exec Interface
The exec interface creates a session for each execution:
```rust
// In exec/src/cli.rs
// Each exec command runs in its own session context
// Configuration is passed through CLI arguments
```

### CLI Subcommands
Each subcommand typically creates its own session:
- `codex exec` - Creates a new session for headless execution
- `codex` (TUI) - Creates a session for interactive use
- `codex mcp` - Creates a session for MCP server mode

## Can Users Switch Between Sessions?

### Direct Session Switching
**No**, Codex does not provide a direct way for users to switch between sessions during a single CLI invocation. Each CLI process operates within a single session context.

### Session Isolation
Sessions are completely isolated from each other:
- Different sessions have different configurations
- Conversation histories are separate
- MCP server connections are independent
- Security contexts are distinct

### Multiple Concurrent Sessions
Users can run multiple instances of Codex simultaneously, each with its own session:
```bash
# Session 1: Development work
codex --profile dev

# Session 2: Production work  
codex --profile prod

# Session 3: Testing
codex --sandbox danger-full-access
```

Each command runs in its own session with its own configuration and state.

## Resuming Historical Sessions

### Session Persistence and Resume Capability

Codex supports resuming previous sessions through its experimental resume functionality. This feature allows users to continue conversations from where they left off in a previous session.

### How Session Resumption Works

1. **Session Recording**: Codex automatically records session data to JSONL files in `~/.codex/sessions/`
2. **Resume Path**: The `experimental_resume` configuration option points to a specific session file
3. **State Restoration**: When resuming, Codex restores:
   - Previous conversation history
   - Session configuration
   - Approved commands
   - MCP server connections

### Using Session Resume

#### Via Command Line Flag
```bash
# Resume a specific session by providing the path to its rollout file
codex --experimental-resume /home/user/.codex/sessions/2025/08/21/rollout-2025-08-21T10-30-45-12345678-1234-5678-1234-567812345678.jsonl
```

#### Via Configuration File
In `~/.codex/config.toml`:
```toml
[experimental]
resume = "/home/user/.codex/sessions/2025/08/21/rollout-2025-08-21T10-30-45-12345678-1234-5678-1234-567812345678.jsonl"
```

### Important Notes About Session Resumption

1. **Experimental Feature**: The resume functionality is experimental and may change in future versions
2. **File Format**: Resume requires the full path to a valid JSONL session file
3. **Session Compatibility**: The resumed session must be compatible with the current Codex version
4. **State Preservation**: Resumed sessions preserve conversation history and approved commands
5. **Security Context**: The security context (sandbox settings) is restored from the original session

### Limitations

- Session resumption is limited to the same Codex installation
- Not all session state may be perfectly preserved across different versions
- The feature is primarily intended for debugging and development scenarios

## Session Configuration

### Persistent Settings
Session configuration includes:
- Model selection
- Approval policies
- Sandbox settings
- Working directory
- MCP server configurations
- Environment variables

### Runtime Overrides
Users can override session settings via:
```bash
# Override model for a single session
codex --model o3

# Override sandbox for a single session
codex --sandbox workspace-write

# Override approval policy for a single session
codex --config approval_policy="never"
```

## Session vs Conversation Clarification

### Session (Higher Level)
- **Scope**: Entire CLI process or application lifetime
- **Persistence**: Maintains configuration and state across multiple interactions
- **Isolation**: Complete separation from other sessions
- **Management**: Controlled by CLI arguments and configuration files

### Conversation (Lower Level)
- **Scope**: Individual user prompts and AI responses
- **Persistence**: Within session context, but resets between sessions
- **Context**: Maintains message history for context awareness
- **Management**: Automatic within session boundaries

## Practical Implications

### For Users
1. **Single Session per CLI Invocation**: Each `codex` command runs in one session
2. **Configuration Persistence**: Settings apply to all interactions in that session
3. **Isolated Environments**: Different sessions don't interfere with each other
4. **State Management**: Conversation history is maintained within a session

### For Developers
1. **Session Creation**: Managed through `Codex::spawn()` in core
2. **State Management**: Session holds all mutable state
3. **Event Flow**: Events flow through session channels
4. **Resource Management**: Sessions manage MCP connections and other resources

## Example Scenarios

### Scenario 1: Single Session Usage
```bash
# Start one session with development settings
codex --profile dev --sandbox workspace-write

# Multiple interactions in same session
# All use same configuration and context
# Conversation history is preserved
```

### Scenario 2: Multiple Session Usage
```bash
# Session 1: Development work
codex --profile dev --sandbox workspace-write

# Session 2: Production deployment (different session)
codex --profile prod --sandbox read-only

# Session 3: Testing (different session)
codex --profile test --dangerously-bypass-approvals-and-sandbox
```

Each session operates independently with its own configuration and state.

## Session Limitations

### No Runtime Session Switching
There is no built-in mechanism to switch between sessions while a CLI process is running. Users must:
1. Terminate the current session
2. Start a new session with desired configuration

### Resource Constraints
Each session consumes system resources:
- Memory for conversation history
- Network connections for MCP servers
- Process resources for sandboxing

## Future Considerations

While Codex doesn't currently support session switching, the architecture is designed to:
- Support multiple concurrent sessions
- Allow for session management APIs
- Enable session persistence and restoration

This design choice ensures clarity and prevents confusion that could arise from session state conflicts.
