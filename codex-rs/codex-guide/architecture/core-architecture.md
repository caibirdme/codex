# Core Architecture

The core architecture of Codex CLI is built around the `codex-core` crate, which contains the main business logic and serves as the foundation for all other components.

## Core Components

### Config System
- **Location**: `core/src/config.rs`
- **Purpose**: Manages all configuration options including model selection, sandbox policies, and provider settings
- **Key Features**:
  - TOML-based configuration with CLI overrides
  - Profile support for different configurations
  - Environment variable support
  - Layered configuration (defaults < config.toml < CLI overrides)

### Conversation Management
- **Location**: `core/src/conversation_manager.rs`
- **Purpose**: Manages the lifecycle of AI conversations
- **Key Features**:
  - Message history management
  - Context window optimization
  - Conversation state persistence

### Model Provider System
- **Location**: `core/src/model_provider_info.rs`
- **Purpose**: Abstracts different AI model providers
- **Key Features**:
  - Support for OpenAI, Azure, Ollama, and custom providers
  - Unified API interface
  - Retry logic and error handling
  - Streaming support

### Execution Environment
- **Location**: `core/src/exec_env.rs`
- **Purpose**: Manages the execution context for commands
- **Key Features**:
  - Working directory management
  - Environment variable handling
  - Process spawning and management

### Safety and Security
- **Location**: `core/src/safety.rs`
- **Purpose**: Implements security policies and safety checks
- **Key Features**:
  - Command validation
  - Policy enforcement
  - Risk assessment

## Data Flow Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   CLI/TUI       │────│   Core Logic     │────│   AI Provider   │
│   Interface     │    │   (codex-core)   │    │   (OpenAI/etc)  │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                │
                                │
                       ┌──────────────────┐
                       │   Sandbox        │
                       │   Execution      │
                       └──────────────────┘
```

## Key Architectural Patterns

### 1. Configuration as Code
All configuration is strongly typed and validated at runtime, with clear precedence rules.

### 2. Plugin Architecture
MCP servers can be dynamically loaded to extend functionality.

### 3. Security First
All command execution is sandboxed by default, with explicit user approval for sensitive operations.

### 4. Streaming Architecture
Real-time streaming of AI responses and command output for responsive user experience.

### 5. Modular Design
Clear separation of concerns with well-defined interfaces between components.

## Crate Structure

### Core Crates

#### `core/`
- Main business logic
- Configuration management
- Conversation handling
- Model provider abstraction
- Safety and security checks

#### `cli/`
- Main CLI entry point
- Subcommand routing
- Argument parsing
- Exit status handling

#### `exec/`
- Headless execution mode
- Non-interactive command processing
- JSON output support

#### `tui/`
- Terminal User Interface
- Real-time chat interface
- Command approval prompts
- File search functionality

#### `mcp-client/` and `mcp-server/`
- MCP protocol client implementation
- MCP protocol server implementation
- Tool and resource discovery

#### `execpolicy/`
- Execution policy enforcement
- Command validation
- Security policy checking

#### `linux-sandbox/`
- Linux-specific sandbox implementation
- Landlock LSM integration

#### `chatgpt/`
- ChatGPT-specific integration
- Authentication handling
- API client implementation

### Supporting Crates

#### `common/`
- Shared utilities and helpers
- Common data structures
- Cross-crate functionality

#### `protocol/`
- Shared protocol definitions
- Message formats
- API contracts

#### `apply-patch/`
- Patch application utilities
- Diff processing
- File modification tools

#### `mcp-types/`
- MCP protocol type definitions
- Schema validation
- Protocol compatibility

## Component Interactions

### 1. CLI → Core
The CLI layer parses user input and passes it to the core logic, which handles configuration loading and conversation management.

### 2. Core → Model Providers
The core logic communicates with model providers to generate AI responses, handling authentication, rate limiting, and error recovery.

### 3. Core → Execution Environment
When commands are generated, the core logic passes them to the execution environment for safe execution within the sandbox.

### 4. Core ↔ MCP
The core interacts with MCP servers to extend functionality, retrieving tools and resources as needed.

### 5. Core ↔ Sandbox
All command execution goes through the sandbox system to ensure security and prevent unauthorized system access.

## Design Principles

### 1. Separation of Concerns
Each component has a well-defined responsibility, making the system maintainable and testable.

### 2. Extensibility
The architecture supports adding new model providers, execution policies, and MCP servers without modifying core logic.

### 3. Security by Design
Security measures are integrated at every level, from input validation to execution sandboxing.

### 4. Performance Optimization
Efficient caching, streaming, and resource management ensure responsive user experience.

### 5. Error Resilience
Robust error handling and graceful degradation ensure the system remains functional even when parts fail.
