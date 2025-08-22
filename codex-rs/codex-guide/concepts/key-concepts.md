# Key Concepts

This document explains fundamental concepts that are essential for understanding how Codex CLI works.

## 1. Model Context Protocol (MCP)

The Model Context Protocol is a standardized way for AI applications to connect with external tools and data sources. It enables:

- **Tool Integration**: Access to external tools and APIs
- **Resource Access**: Structured access to files, databases, and services
- **Real-time Data**: Live data from external sources
- **Extensibility**: Easy addition of new capabilities

### MCP Servers
MCP servers provide tools and resources that Codex can use. Examples include:
- Filesystem servers for reading/writing files
- Database servers for querying data
- Web search servers for internet information
- Version control servers for Git operations

### MCP Clients
Codex acts as an MCP client, connecting to MCP servers to extend its functionality.

## 2. Sandboxing

Sandboxing is a security mechanism that isolates command execution to prevent unauthorized system access.

### Sandbox Modes
- **Read-Only**: Can read files but cannot write or access network
- **Workspace-Write**: Can read/write in workspace and temp directories, but no network access
- **Danger-Full-Access**: Full system access (use with extreme caution)

### Security Benefits
- Prevents malicious code execution
- Protects sensitive files and directories
- Controls network access
- Limits resource consumption

## 3. Approval Policies

Approval policies determine when users are prompted to approve command execution.

### Policy Types
- **Never**: No approval required for any commands
- **On Failure**: Approve when commands fail
- **On Request**: Approve when model requests it
- **Untrusted**: Approve all non-whitelisted commands

### Trust Model
- **Trusted Directories**: Explicitly trusted workspaces
- **Untrusted Directories**: Default for new workspaces
- **Git Repositories**: Special handling for Git-managed projects

## 4. Configuration Layers

Codex uses a layered configuration system with clear precedence:

1. **Defaults**: Built-in default values
2. **Config File**: `~/.codex/config.toml`
3. **CLI Overrides**: Command-line arguments
4. **Environment Variables**: System environment settings

### Configuration Profiles
Profiles allow switching between different sets of configuration options:
- Development profiles with more permissive settings
- Production profiles with strict security
- CI/CD profiles optimized for automation

## 5. Conversation Management

Codex maintains conversation state to provide context-aware responses.

### Message History
- Stores user prompts and AI responses
- Manages context window size
- Applies retention policies

### Project Documentation
- Loads `AGENTS.md` files from projects
- Includes project-specific instructions
- Enhances context for AI responses

## 6. Execution Flow

The execution flow describes how Codex processes user requests:

1. **Input Processing**: Parse user prompt and context
2. **AI Analysis**: Generate response and suggested commands
3. **Safety Check**: Validate proposed commands
4. **Approval**: Request user approval when needed
5. **Execution**: Run commands in sandboxed environment
6. **Output**: Return results to user

## 7. Model Providers

Different AI models and providers are supported through a unified interface.

### Supported Providers
- OpenAI (Chat Completions and Responses APIs)
- Azure OpenAI Service
- Ollama (local models)
- Custom providers

### Provider Configuration
Each provider has its own configuration options:
- Base URLs for API endpoints
- Authentication methods
- Rate limiting settings
- Retry policies

## 8. Tool Integration

Codex can integrate with various tools through MCP to extend its capabilities.

### Common Tools
- **File Operations**: Read, write, and manipulate files
- **Code Analysis**: Static analysis and linting
- **Version Control**: Git operations and repository management
- **Build Systems**: Package management and build automation

### Tool Execution
Tools are executed in a secure environment with appropriate permissions and constraints.

## 9. Security Architecture

Codex implements defense-in-depth security measures:

### Multi-layered Security
1. **Input Validation**: Sanitize all user inputs
2. **Command Validation**: Check proposed commands for safety
3. **Execution Sandboxing**: Isolate command execution
4. **Approval Systems**: Human oversight for sensitive operations
5. **Audit Logging**: Track all activities for security monitoring

## 10. Performance Optimization

Codex is designed for efficient operation:

### Caching Strategies
- Configuration caching
- Tool and resource information caching
- Model response caching for similar prompts

### Streaming Architecture
- Real-time output streaming
- Progressive rendering of results
- Immediate feedback to users

### Resource Management
- Memory usage optimization
- Connection pooling for external services
- Garbage collection and cleanup
