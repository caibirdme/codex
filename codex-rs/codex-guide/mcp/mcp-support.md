# Model Context Protocol (MCP) Support

Codex CLI provides comprehensive support for the Model Context Protocol (MCP), functioning as both an MCP client and server.

## MCP Overview

The Model Context Protocol is a standardized way for AI applications to connect with external tools and data sources. It enables:
- **Tool Integration**: Access to external tools and APIs
- **Resource Access**: Structured access to files, databases, and services
- **Real-time Data**: Live data from external sources
- **Extensibility**: Easy addition of new capabilities

## MCP Client Mode

### Configuration
MCP servers are configured in `~/.codex/config.toml`:

```toml
[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem"]
env = { "ALLOWED_DIRS" = "/home/user/projects" }

[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]
env = { "GITHUB_TOKEN" = "your-token" }

[mcp_servers.postgres]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-postgres"]
env = { "DATABASE_URL" = "postgresql://..." }
```

### Available MCP Servers

#### 1. Filesystem Server
- **Package**: `@modelcontextprotocol/server-filesystem`
- **Purpose**: File operations and directory browsing
- **Configuration**:
  ```toml
  [mcp_servers.filesystem]
  command = "npx"
  args = ["-y", "@modelcontextprotocol/server-filesystem"]
  env = { "ALLOWED_DIRS" = "/home/user/projects:/tmp" }
  ```

#### 2. GitHub Server
- **Package**: `@modelcontextprotocol/server-github`
- **Purpose**: GitHub API integration
- **Configuration**:
  ```toml
  [mcp_servers.github]
  command = "npx"
  args = ["-y", "@modelcontextprotocol/server-github"]
  env = { "GITHUB_PERSONAL_ACCESS_TOKEN" = "ghp_..." }
  ```

#### 3. PostgreSQL Server
- **Package**: `@modelcontextprotocol/server-postgres`
- **Purpose**: Database operations
- **Configuration**:
  ```toml
  [mcp_servers.postgres]
  command = "npx"
  args = ["-y", "@modelcontextprotocol/server-postgres"]
  env = { "DATABASE_URL" = "postgresql://user:pass@localhost/db" }
  ```

#### 4. Web Search Server
- **Package**: `@modelcontextprotocol/server-brave-search`
- **Purpose**: Web search integration
- **Configuration**:
  ```toml
  [mcp_servers.brave-search]
  command = "npx"
  args = ["-y", "@modelcontextprotocol/server-brave-search"]
  env = { "BRAVE_API_KEY" = "your-key" }
  ```

### MCP Client Architecture

#### Components
- **MCP Connection Manager**: `core/src/mcp_connection_manager.rs`
- **MCP Tool Call Handler**: `core/src/mcp_tool_call.rs`
- **Message Processor**: `mcp-client/src/mcp_client.rs`

#### Features
- **Lazy Loading**: MCP servers loaded on-demand
- **Caching**: Tool and resource information cached
- **Error Handling**: Graceful handling of server failures
- **Security**: Sandboxed execution of MCP tool calls

## Managing Multiple MCP Clients

### Client Lifecycle Management

Codex manages multiple MCP clients through the `McpConnectionManager` which:
1. **Spawns** clients concurrently for all configured servers
2. **Initializes** each client with proper capabilities
3. **Maintains** active connections for the session lifetime
4. **Handles** failures gracefully with error reporting

### Connection Pooling and Concurrency

- **Concurrent Startup**: All MCP servers are started concurrently for faster initialization
- **Connection Reuse**: Active clients are reused across tool calls within a session
- **Timeout Handling**: Configurable timeouts for server responses
- **Resource Management**: Proper cleanup of connections when sessions end

### Server Name Resolution

Each MCP server is identified by a unique name from the configuration:
```toml
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]

[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem"]
```

The system resolves tool calls using the fully-qualified name format: `server_name__tool_name`

### Communication Protocol

#### Tool Call Flow
1. **Tool Resolution**: Codex identifies the correct server based on tool name
2. **Message Serialization**: Arguments are serialized to JSON
3. **Network Communication**: Tool call sent via stdio to the MCP server
4. **Result Processing**: Server response is parsed and returned to the model
5. **Event Notification**: Begin/end events are sent to the session

#### Example Communication Flow
```mermaid
sequenceDiagram
    participant Model
    participant Codex
    participant MCP_Server
    
    Model->>Codex: Tool call: github__list_issues
    Codex->>MCP_Server: stdio call: list_issues
    MCP_Server->>Codex: JSON response
    Codex->>Model: Processed result
```

### Error Handling and Recovery

#### Connection Failures
- **Startup Failures**: Failed servers are logged but don't prevent other servers from starting
- **Runtime Failures**: Individual tool calls fail gracefully with error messages
- **Timeout Handling**: Requests that exceed timeout limits are cancelled

#### Retry Logic
- **Automatic Retries**: Failed tool calls may be retried based on server configuration
- **Backoff Strategy**: Exponential backoff for repeated failures
- **Fallback Mechanisms**: Alternative approaches when primary servers fail

## Communication Patterns

### Synchronous Communication
Most MCP tool calls are synchronous:
- Request is sent to the MCP server
- Response is awaited before continuing
- Timeout limits prevent hanging operations

### Asynchronous Communication
For long-running operations:
- Events are streamed back to the session
- Progress updates are provided in real-time
- Results are collected and returned when complete

### Streaming Support

Some MCP servers support streaming responses:
- **Progress Updates**: Real-time status updates
- **Partial Results**: Intermediate results during long operations
- **Error Notifications**: Immediate error reporting

### Security Considerations

#### Isolation
- Each MCP server runs in its own process space
- Network access is controlled by sandbox policies
- File system access is restricted by configuration

#### Authentication
- Environment variables are used for sensitive credentials
- Token-based authentication for external services
- Secure credential handling throughout the pipeline

### Performance Optimization

#### Caching Strategies
- **Tool Metadata**: Cached tool definitions to avoid repeated queries
- **Server Capabilities**: Cached server capabilities for faster resolution
- **Resource Information**: Cached resource listings for frequent access

#### Lazy Initialization
- Servers are only started when needed
- Tool lists are fetched on-demand
- Connections are established only when required

### Monitoring and Debugging

#### Logging
- **Connection Events**: Server startup/shutdown events
- **Tool Calls**: Detailed logs of tool invocations
- **Errors**: Comprehensive error reporting with stack traces

#### Metrics Collection
- **Response Times**: Performance metrics for each tool call
- **Success Rates**: Availability statistics for each server
- **Resource Usage**: Memory and CPU consumption tracking

### Configuration Best Practices

#### Server Selection
- **Minimal Permissions**: Configure servers with least privilege
- **Environment Isolation**: Separate environments for different use cases
- **Health Checks**: Monitor server availability regularly

#### Resource Management
- **Memory Limits**: Set appropriate memory constraints
- **Timeout Values**: Configure reasonable timeout thresholds
- **Retry Policies**: Define appropriate retry strategies

### Advanced Usage Patterns

#### Multi-Server Coordination
Multiple MCP servers can work together:
- **Data Aggregation**: Combine results from multiple sources
- **Cross-Server Operations**: Complex workflows spanning multiple servers
- **Conditional Execution**: Choose servers based on context or conditions

#### Dynamic Server Management
- **Runtime Addition**: Servers can be added during session
- **Configuration Updates**: Hot reload of server configurations
- **Server Replacement**: Seamless replacement of failing servers

### Troubleshooting Guide

#### Common Issues
1. **Server Not Found**: Verify server name in configuration
2. **Permission Denied**: Check file permissions and environment variables
3. **Connection Timeout**: Increase timeout values or optimize server performance
4. **Tool Not Found**: Verify tool name and server connectivity

#### Diagnostic Commands
```bash
# Check available tools
codex --config mcp_servers.debug.command="echo debug"

# Enable verbose logging
RUST_LOG=mcp=debug codex

# Test specific server
codex --config mcp_servers.test.command="your-test-command"
```

## MCP Server Mode

### Running as MCP Server
```bash
# Start Codex as MCP server
codex mcp

# Test with MCP Inspector
npx @modelcontextprotocol/inspector codex mcp
```

### MCP Server Capabilities

#### 1. Tools
- **execute_command**: Execute shell commands
- **apply_patch**: Apply code patches
- **read_file**: Read file contents
- **write_file**: Write to files
- **list_directory**: List directory contents

#### 2. Resources
- **Project files**: Access to workspace files
- **Git information**: Repository status and history
- **Environment**: System information
- **Configuration**: Current settings

#### 3. Prompts
- **Code review**: Automated code review
- **Refactoring**: Code refactoring suggestions
- **Testing**: Test generation
- **Documentation**: Documentation generation

### MCP Server Configuration

#### Server Setup
```json
{
  "mcpServers": {
    "codex": {
      "command": "codex",
      "args": ["mcp"],
      "env": {}
    }
  }
}
```

#### Client Usage
```javascript
// Example client configuration
const client = new MCPClient({
  server: {
    command: "codex",
    args: ["mcp"]
  }
});
```

## MCP Protocol Implementation

### Protocol Versions
- **Supported**: 2025-03-26, 2025-06-18
- **Location**: `mcp-types/src/lib.rs`

### Message Types
- **Initialize**: Server initialization
- **Tools/List**: List available tools
- **Tools/Call**: Execute tool calls
- **Resources/List**: List available resources
- **Resources/Read**: Read resource contents
- **Prompts/List**: List available prompts
- **Prompts/Get**: Get prompt templates

### Error Handling
- **Connection Errors**: Handle server disconnections
- **Tool Errors**: Handle tool execution failures
- **Validation Errors**: Handle invalid requests
- **Timeout Errors**: Handle slow responses

## Advanced MCP Usage

### Custom MCP Servers

#### Python Example
```python
# custom_mcp_server.py
from mcp.server import Server

server = Server("custom-server")

@server.tool()
def custom_tool(input: str) -> str:
    """Custom tool description"""
    return f"Processed: {input}"

if __name__ == "__main__":
    server.run()
```

#### Configuration
```toml
[mcp_servers.custom]
command = "python"
args = ["custom_mcp_server.py"]
```

### MCP Server Development

#### Server Requirements
- **Protocol Compliance**: Must follow MCP specification
- **Security**: Must handle security appropriately
- **Error Handling**: Must provide meaningful error messages
- **Performance**: Must respond within reasonable time limits

#### Testing
```bash
# Test with MCP Inspector
npx @modelcontextprotocol/inspector your-server-command

# Test with Codex
codex --config mcp_servers.test.command="your-server-command"
```

## MCP Best Practices

### 1. Server Selection
- **Use Official Servers**: Prefer official MCP servers
- **Security Review**: Review server code before use
- **Minimal Permissions**: Grant minimal necessary permissions

### 2. Configuration Management
- **Environment Variables**: Use env vars for sensitive data
- **Path Restrictions**: Limit accessible paths
- **Resource Limits**: Set appropriate resource limits

### 3. Error Handling
- **Graceful Degradation**: Handle server failures gracefully
- **User Feedback**: Provide clear error messages
- **Fallback Options**: Have fallback strategies

### 4. Performance Optimization
- **Caching**: Cache tool/resource information
- **Lazy Loading**: Load servers on-demand
- **Connection Pooling**: Reuse connections when possible

## Troubleshooting MCP

### Common Issues

#### 1. Server Not Starting
```bash
# Check server command
which npx
node --version

# Test server manually
npx -y @modelcontextprotocol/server-filesystem
```

#### 2. Connection Issues
```bash
# Check logs
tail -f ~/.codex/log/*.log

# Enable debug logging
RUST_LOG=debug codex
```

#### 3. Permission Issues
```bash
# Check file permissions
ls -la ~/.codex/config.toml

# Check environment variables
env | grep -E "(PATH|NODE|NPM)"
```

### Debug Mode
```bash
# Enable MCP debugging
RUST_LOG=mcp=debug codex

# Test specific server
codex --config mcp_servers.test.command="echo test"
