# Prompt Construction

The prompts that Codex concatenates before the first interaction with the LLM include:

1. **Base Instructions** - The main `core/prompt.md` file that defines the agent's behavior, capabilities, and guidelines
2. **User Instructions** - Custom instructions provided by the user in `AGENTS.md` or via CLI flags
3. **Tool Instructions** - Specialized instructions for tools like `apply_patch` when needed
4. **MCP Tool Definitions** - Tool specifications from MCP servers that are integrated into the tool list
5. **Environment Context** - Information about the current working directory, sandbox settings, and approval policies

## Prompt Construction Process

The prompt construction process involves:

1. Starting with the base instructions from `core/prompt.md`
2. Adding any user-specific instructions from `AGENTS.md` or CLI overrides
3. Including specialized tool instructions for models that need them
4. Integrating available tools from MCP servers
5. Adding environment context information

The prompt is constructed in the `Prompt::get_full_instructions()` method in `core/src/client_common.rs`, which combines the base instructions with any model-specific tool instructions.

## Prompt Components

### Base Instructions (`core/prompt.md`)
This is the primary instruction set that defines the agent's behavior, capabilities, and guidelines. It includes:
- Agent personality and tone guidelines
- Task execution rules
- Planning and reasoning guidelines
- Code quality and testing guidelines
- Sandbox and approval behavior
- Tool usage guidelines

### User Instructions
These are custom instructions provided by the user in `AGENTS.md` or via CLI flags that are appended to the base instructions.

### Tool Instructions
Specialized instructions for specific tools:
- `apply_patch` tool instructions (when needed for specific models)
- MCP tool definitions from external servers

### Environment Context
Information about the current execution environment:
- Working directory
- Sandbox policy settings
- Approval policy settings
- Shell environment information

## Implementation Details

The prompt construction is handled by the `Prompt` struct in `core/src/client_common.rs`. The key method is `get_full_instructions()` which:

1. Gets the base instructions from `BASE_INSTRUCTIONS` constant (which is loaded from `core/prompt.md`)
2. Checks if the model needs special apply patch instructions
3. Combines all sections with newlines between them
4. Returns the complete prompt as a `Cow<str>` for efficient memory usage

```rust
pub(crate) fn get_full_instructions(&self, model: &ModelFamily) -> Cow<'_, str> {
    let base = self
        .base_instructions_override
        .as_deref()
        .unwrap_or(BASE_INSTRUCTIONS);
    let mut sections: Vec<&str> = vec![base];
    if model.needs_special_apply_patch_instructions {
        sections.push(APPLY_PATCH_TOOL_INSTRUCTIONS);
    }
    Cow::Owned(sections.join("\n"))
}
```

This ensures that the prompt is properly constructed with all necessary context for the model to understand its role and capabilities.
