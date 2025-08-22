# Event System

The Codex CLI implements a sophisticated event-driven architecture that facilitates communication between different components of the system. This system allows for real-time interaction, proper state management, and seamless integration with both CLI and TUI interfaces.

## Overview

The event system in Codex is built around a producer-consumer pattern where:
- **Producers** generate events during various operations (AI responses, command execution, approvals, etc.)
- **Consumers** listen for and process these events to update the UI or handle state changes
- **Events** carry structured data that describes what happened and what should happen next

## Core Event Types

### 1. Session Events
These events relate to session lifecycle and configuration:

```rust
pub enum EventMsg {
    SessionConfigured(SessionConfiguredEvent),
    ShutdownComplete,
    TaskStarted,
    TaskComplete(TaskCompleteEvent),
    TurnAborted(TurnAbortedEvent),
    TokenCount(TokenUsage),
    // ... other events
}
```

### 2. AI Interaction Events
Events related to AI model responses and reasoning:

```rust
pub enum EventMsg {
    AgentMessage(AgentMessageEvent),
    AgentMessageDelta(AgentMessageDeltaEvent),
    AgentReasoning(AgentReasoningEvent),
    AgentReasoningDelta(AgentReasoningDeltaEvent),
    AgentReasoningSectionBreak(AgentReasoningSectionBreakEvent),
    AgentReasoningRawContent(AgentReasoningRawContentEvent),
    AgentReasoningRawContentDelta(AgentReasoningRawContentDeltaEvent),
    // ... other AI-related events
}
```

### 3. Execution Events
Events related to command execution and sandbox operations:

```rust
pub enum EventMsg {
    ExecCommandBegin(ExecCommandBeginEvent),
    ExecCommandEnd(ExecCommandEndEvent),
    PatchApplyBegin(PatchApplyBeginEvent),
    PatchApplyEnd(PatchApplyEndEvent),
    ExecApprovalRequest(ExecApprovalRequestEvent),
    ApplyPatchApprovalRequest(ApplyPatchApprovalRequestEvent),
    // ... other execution events
}
```

### 4. System Events
General system events and notifications:

```rust
pub enum EventMsg {
    Error(ErrorEvent),
    BackgroundEvent(BackgroundEventEvent),
    TurnDiff(TurnDiffEvent),
    McpListToolsResponse(McpListToolsResponseEvent),
    // ... other system events
}
```

## Event Flow Architecture

### Producer Side (Core Logic)
The core logic components generate events as they process operations:

1. **Input Processing**: User input is converted to submission events
2. **AI Processing**: Model responses generate message and reasoning events
3. **Command Execution**: Command execution generates begin/end events
4. **Approval Requests**: Security checks generate approval events
5. **System Events**: Error conditions and state changes

### Consumer Side (Interfaces)
Different interfaces consume events in different ways:

1. **CLI Interface**: Processes events for headless execution
2. **TUI Interface**: Renders events to terminal UI
3. **External Consumers**: Applications that integrate with Codex

## Event Queue System

### Channels and Communication
Codex uses asynchronous channels for event communication:

```rust
// Core structure in Codex
pub struct Codex {
    next_id: AtomicU64,
    tx_sub: Sender<Submission>,   // Submission channel
    rx_event: Receiver<Event>,    // Event receiver
}

// Session structure
pub struct Session {
    session_id: Uuid,
    tx_event: Sender<Event>,      // Event sender
    // ... other fields
}
```

### Submission Flow (Clarified)
The submission flow in Codex works as follows:

1. **User Input** → Submitted via `Codex::submit()` method
2. **Submission Channel** → `tx_sub` channel receives the submission
3. **Processing Loop** → `submission_loop` function processes the submission
4. **Operation Handling** → Different operations trigger different behaviors:
   - `Op::UserInput` → Adds to pending input for current task
   - `Op::UserTurn` → Starts a new task with new context
   - `Op::ExecApproval` → Handles command execution approval
   - `Op::PatchApproval` → Handles patch application approval
5. **Event Generation** → Operations generate events that are sent via `tx_event` channel
6. **Consumer Reception** → `rx_event` receiver gets events for processing

### Detailed Submission Flow Example

```rust
// When user submits input:
let submission_id = codex.submit(Op::UserInput { items }).await?;

// This goes through:
// 1. Codex::submit() → creates Submission with unique ID
// 2. Submission sent to tx_sub channel
// 3. submission_loop receives it
// 4. Operation is processed based on Op variant
// 5. Events are sent via tx_event channel to consumers
// 6. Consumers receive events via rx_event receiver
```

### Event Processing Loop
```rust
async fn submission_loop(
    sess: Arc<Session>,
    turn_context: TurnContext,
    config: Arc<Config>,
    rx_sub: Receiver<Submission>,
) {
    // Infinite loop processing submissions
    while let Ok(sub) = rx_sub.recv().await {
        match sub.op {
            Op::Interrupt => {
                // Handle interruption
            }
            Op::UserInput { items } => {
                // Add to pending input for current task
                // If no current task, start new one
            }
            Op::UserTurn { items, .. } => {
                // Start new task with fresh context
                // Process AI response and generate events
            }
            Op::ExecApproval { id, decision } => {
                // Handle approval decision
                // Notify session of approval
            }
            Op::Shutdown => {
                // Graceful shutdown
                break;
            }
            // ... other operations
        }
    }
}
```

## Event Consumption Patterns

### 1. Real-time Streaming
The TUI interface consumes events in real-time to provide immediate feedback:

```rust
// TUI event processing
loop {
    let event = codex.next_event().await?;
    match event.msg {
        EventMsg::AgentMessage(msg) => {
            // Update UI with AI message
        }
        EventMsg::ExecCommandBegin(begin) => {
            // Show command execution start
        }
        EventMsg::ExecCommandEnd(end) => {
            // Show command execution result
        }
        // ... other event handlers
    }
}
```

### 2. Headless Processing
The exec interface processes events for automation:

```rust
// Headless event processing
while let Ok(event) = codex.next_event().await {
    match event.msg {
        EventMsg::AgentMessage(msg) => {
            // Log or output message
        }
        EventMsg::TaskComplete(_) => {
            // Exit when task completes
            break;
        }
        EventMsg::Error(err) => {
            // Handle errors
            eprintln!("Error: {}", err.message);
        }
        // ... other event handlers
    }
}
```

### 3. Approval Handling
Approval events are handled through a special channel:

```rust
// Approval request handling
let rx_approve = sess.request_command_approval(
    sub_id.clone(),
    call_id.clone(),
    command,
    cwd,
    reason,
).await;

// Wait for user decision
match rx_approve.await.unwrap_or_default() {
    ReviewDecision::Approved => {
        // Continue with execution
    }
    ReviewDecision::Denied => {
        // Cancel execution
    }
}
```

## Event Lifecycle

### 1. Event Creation
Events are created in various parts of the system:
- AI response processing
- Command execution
- Approval systems
- Error handling

### 2. Event Transmission
Events are transmitted through:
- `tx_event` channels from session to consumers
- `tx_sub` channels from user to core processing
- MCP server communication

### 3. Event Consumption
Consumers process events:
- TUI renders events to terminal
- CLI processes events for automation
- Logging systems capture events
- External integrations consume events

## Event Serialization and Format

Events follow a structured format:

```rust
pub struct Event {
    pub id: String,        // Unique identifier for the event
    pub msg: EventMsg,     // The actual event message
}
```

The `EventMsg` enum contains all possible event types with their associated data, ensuring type safety and clear contract between producers and consumers.

## Benefits of the Event System

### 1. Decoupling
Components communicate through events rather than direct function calls, reducing tight coupling.

### 2. Scalability
The async channel-based system scales well with concurrent operations.

### 3. Flexibility
Different consumers can process the same events in different ways.

### 4. Real-time Feedback
Users receive immediate feedback through streaming events.

### 5. Error Handling
Errors are propagated as events, making them easy to handle consistently.

## Example Event Flow

Here's a typical flow of events during a command execution:

1. **User Input** → `UserInput` operation → `tx_sub` channel
2. **AI Processing** → Generates `AgentMessage` events → `tx_event` channel
3. **Command Request** → Generates `ExecCommandBegin` event → `tx_event` channel
4. **Approval Request** → Generates `ExecApprovalRequest` event → `tx_event` channel
5. **User Approval** → `ExecApproval` operation → `tx_sub` channel
6. **Command Execution** → Generates `ExecCommandEnd` event → `tx_event` channel
7. **Completion** → `TaskComplete` event → `tx_event` channel

This event-driven architecture enables Codex to provide responsive, real-time interaction while maintaining clear separation of concerns between components.
