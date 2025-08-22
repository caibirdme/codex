# Sandbox Security System

The Codex CLI implements a comprehensive sandbox security system to safely execute AI-generated commands while protecting the host system.

## Security Philosophy

- **Principle of Least Privilege**: Commands run with minimal necessary permissions
- **Defense in Depth**: Multiple layers of security controls
- **User Control**: Explicit user approval for sensitive operations
- **Transparency**: Clear visibility into what commands will do

## Sandbox Modes

### 1. Read-Only Mode (Default)
- **Permissions**: Read any file, no writes, no network
- **Use Case**: Safe exploration and analysis
- **Security Level**: High

### 2. Workspace-Write Mode
- **Permissions**: Read any file, write to workspace and temp directories, no network
- **Use Case**: Development and editing tasks
- **Security Level**: Medium

### 3. Danger-Full-Access Mode
- **Permissions**: Full system access
- **Use Case**: Trusted environments (containers, VMs)
- **Security Level**: None (use with caution)

## Platform-Specific Implementations

### macOS (Seatbelt)
- **Technology**: Apple's Seatbelt (sandbox-exec)
- **Implementation**: `core/src/seatbelt.rs`
- **Features**:
  - File system restrictions
  - Network access control
  - Process execution limits
  - Read-only Git repositories

### Linux (Landlock)
- **Technology**: Linux Landlock LSM
- **Implementation**: `linux-sandbox/src/landlock.rs`
- **Features**:
  - File system access control
  - Network restrictions
  - Process capabilities

### Windows
- **Status**: Basic implementation (limited sandboxing)
- **Approach**: Process restrictions and file system ACLs

## Security Components

### 1. Execution Policy (`execpolicy/`)
- **Purpose**: Defines what commands are allowed
- **Features**:
  - Command whitelisting/blacklisting
  - Argument validation
  - Path restrictions
  - Network access control

### 2. Command Validation
- **Location**: `core/src/is_safe_command.rs`
- **Purpose**: Pre-execution safety checks
- **Checks**:
  - Malicious command patterns
  - Dangerous file operations
  - Network requests
  - System modifications

### 3. Approval System
- **Location**: `core/src/approval.rs`
- **Purpose**: User approval for sensitive operations
- **Levels**:
  - **Never**: No approval required
  - **On Failure**: Approve when commands fail
  - **On Request**: Approve when model requests
  - **Untrusted**: Approve all non-whitelisted commands

## Security Configuration

### Basic Configuration
```toml
# ~/.codex/config.toml
sandbox_mode = "read-only"
approval_policy = "untrusted"
```

### Advanced Configuration
```toml
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
writable_roots = ["/tmp", "/var/tmp", "/home/user/projects"]
network_access = false
exclude_tmpdir_env_var = false
exclude_slash_tmp = false
```

## Security Features

### 1. File System Protection
- **Read-Only Access**: Prevents unauthorized modifications
- **Path Restrictions**: Limits accessible directories
- **Symbolic Link Protection**: Prevents symlink attacks
- **Hidden File Protection**: Protects sensitive files

### 2. Network Security
- **Network Isolation**: Blocks outbound connections by default
- **DNS Protection**: Prevents DNS-based attacks
- **Port Restrictions**: Limits network access

### 3. Process Security
- **Process Isolation**: Runs commands in isolated environment
- **Resource Limits**: Prevents resource exhaustion
- **Privilege Dropping**: Runs with minimal privileges

### 4. Environment Security
- **Environment Filtering**: Removes sensitive environment variables
- **Path Sanitization**: Cleans PATH and other variables
- **Secret Protection**: Filters API keys and tokens

## Security Monitoring

### 1. Audit Logging
- **Location**: `~/.codex/log/`
- **Content**: All executed commands and their outcomes
- **Retention**: Configurable log retention

### 2. Security Events
- **Blocked Commands**: Logged when commands are blocked
- **Policy Violations**: Recorded for security analysis
- **User Approvals**: Tracked for compliance

## Testing Security

### 1. Debug Mode
```bash
# Test sandbox on macOS
codex debug seatbelt ls -la

# Test sandbox on Linux
codex debug landlock ls -la
```

### 2. Security Validation
```bash
# Check current security settings
codex config show | grep -E "(sandbox|approval)"

# Validate configuration
codex config validate
```

## Best Practices

### 1. Development Environment
```toml
# Safe for development
sandbox_mode = "workspace-write"
approval_policy = "on-failure"
```

### 2. Production Environment
```toml
# Maximum security
sandbox_mode = "read-only"
approval_policy = "untrusted"
```

### 3. CI/CD Environment
```toml
# Automated execution
sandbox_mode = "danger-full-access"
approval_policy = "never"
```

## Security Considerations

### 1. Trust Model
- **Trusted**: User explicitly trusts a directory
- **Untrusted**: Default for new directories
- **Verification**: Git repository state affects trust

### 2. Escalation Paths
- **Explicit Approval**: User must approve sensitive operations
- **Context Awareness**: Model understands security implications
- **Rollback Capability**: Can undo changes if needed

### 3. Incident Response
- **Immediate Blocking**: Dangerous commands are blocked
- **User Notification**: Clear explanation of security decisions
- **Recovery Options**: Ways to proceed safely
