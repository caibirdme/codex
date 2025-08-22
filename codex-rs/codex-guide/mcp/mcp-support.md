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
