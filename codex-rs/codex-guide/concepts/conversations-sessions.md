# Conversations and Sessions

Understanding the distinction between conversations and sessions is crucial for grasping how Codex maintains context and state during interactions.

## Session

A **Session** represents a single, continuous interaction with the Codex system. It's the highest-level organizational unit that encompasses all the state and configuration needed for a complete interaction.

### Key Characteristics of Sessions

1. **Persistent State**: Maintains configuration, environment settings, and session-level context
2. **Unique Identifier**: Each session has a unique UUID that distinguishes it from other sessions
3. **Configuration**: Contains the base configuration that applies to all interactions within the session
4. **Lifecycle**: Created at startup and destroyed when the session ends
5. **Scope**: All operations within a session share the same working directory, approval policies, and sandbox settings

### Session Components

In the code, a Session is defined as:

```rust
pub(crate) struct Session {
    session_id: Uuid,                    // Unique identifier for the session
    tx_event: Sender<Event>,             // Channel for sending events to consumers
    mcp_connection_manager: McpConnectionManager,  // MCP server connections
    notify: Option<Vec<String>>,         // Notification command configuration
    rollout: Mutex<Option<RolloutRecorder>>,  // History recording
    state: Mutex<State>,                 // Mutable session state
    codex_linux_sandbox_exe: Option<PathBuf>,  // Sandbox executable path
    user_shell: shell::Shell,            // User's preferred shell
    show_raw_agent_reasoning: bool,      // Whether to show raw reasoning
}
```

### Session Lifecycle

1. **Creation**: When `Codex::spawn()` is called, a new session is initialized
2. **Operation**: Multiple turns and interactions occur within the session
3. **Shutdown**: Session ends when `Op::Shutdown` is processed

## Conversation

A **Conversation** refers to the exchange of messages between the user and the AI model within a session. It's the actual dialogue that takes place during an interaction.

### Key Characteristics of Conversations

1. **Message History**: Stores the complete history of user prompts and AI responses
2. **Context Window**: Manages the context window to ensure relevant information is available
3. **Turn-based**: Each user input and AI response constitutes a turn
4. **State Management**: Maintains the conversation state for context awareness
5. **Persistence**: Can be saved to history files for later reference

### Conversation Components

The conversation is managed through:

```rust
// Conversation state maintained in Session
struct State {
    approved_commands: HashSet<Vec<String>>,  // Commands pre-approved for execution
    current_task: Option<AgentTask>,          // Current task being processed
    pending_approvals: HashMap<String, oneshot::Sender<ReviewDecision>>,  // Approval requests
    pending_input: Vec<ResponseInputItem>,    // Input waiting to be processed
    history: ConversationHistory,             // Message history
}
```

### Conversation Flow

1. **User Input**: User provides a prompt or instruction
2. **AI Processing**: Model analyzes the input and generates a response
3. **Action Generation**: Model may request tool calls or command execution
4. **Execution**: Commands are executed (with approval if needed)
5. **Feedback Loop**: Results are fed back to the model for continued interaction
6. **Continuation**: New user input continues the conversation

## Relationship Between Sessions and Conversations

### One-to-Many Relationship

- **One Session** can contain **Multiple Conversations** (or turns)
- Each time a user submits a new input, it starts a new turn within the same session
- The conversation history is preserved throughout the session

### Example Scenario

```
Session Start (Session ID: abc-123-def-456)
├── Turn 1: "Fix the bug in main.py"
│   ├── AI Response: "I'll help fix that bug"
│   ├── Command Execution: "grep -n 'bug' main.py"
│   └── Result: Found 3 instances of 'bug'
├── Turn 2: "Can you also update the error handling?"
│   ├── AI Response: "I'll update the error handling for those instances"
│   ├── Command Execution: "apply_patch" with changes
│   └── Result: Changes applied successfully
└── Session End
```

## Session vs Conversation in Practice

### Session-Level Properties
- Working directory (cwd)
- Approval policy
- Sandbox policy
- Model configuration
- MCP server connections
- Notification settings

### Conversation-Level Properties
- Message history
- Current context
- Turn-by-turn state
- Pending approvals
- Approved commands

## Technical Implementation

### Session Management in Core

In `core/src/codex.rs`, the session management works as follows:

```rust
// Session creation
let session = Session::new(configure_session, config, auth, tx_event).await?;

// Session maintains conversation history
let mut state = State {
    history: ConversationHistory::new(),
    // ... other state
};

// Adding to conversation history
sess.record_conversation_items(&items_to_record).await;
```

### Conversation History Management

The conversation history is managed through:

```rust
// ConversationHistory struct
pub struct ConversationHistory {
    // Stores the conversation history
    items: Vec<ResponseItem>,
    // Tracks assistant text for real-time updates
    assistant_text: String,
}

// Methods for managing conversation
impl ConversationHistory {
    pub fn contents(&self) -> Vec<ResponseItem> { /* ... */ }
    pub fn record_items(&mut self, items: &[ResponseItem]) { /* ... */ }
    pub fn append_assistant_text(&mut self, text: &str) { /* ... */ }
}
```

## Practical Implications

### For Users
- **Session**: Think of it as a "session" in a terminal - everything you do in one session is connected
- **Conversation**: Think of it as the actual dialogue - each prompt/response cycle is a conversation turn

### For Developers
- **Session**: Configuration and state that persists across multiple interactions
- **Conversation**: The actual message exchange that gets processed by the AI model

### For Security
- Session-level policies (sandbox, approval) apply to all conversations within that session
- Conversation history is preserved for context but can be cleared or managed

## Example Usage

```rust
// Start a new session
let codex = Codex::spawn(config, auth).await?;

// First conversation turn
let submission_id = codex.submit(Op::UserTurn {
    items: vec![InputItem::Text { text: "Write a hello world program".to_string() }],
    // ... other parameters
}).await?;

// Continue with another turn in the same session
let submission_id = codex.submit(Op::UserTurn {
    items: vec![InputItem::Text { text: "Make it in Python".to_string() }],
    // ... same session context
}).await?;
```

This design allows Codex to maintain context across multiple interactions while keeping the session configuration consistent, enabling a natural conversation flow that feels like a continuous interaction rather than discrete commands.
