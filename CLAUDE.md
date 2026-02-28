# CLAUDE.md

## n8n Workflow Development

This project uses n8n for workflow automation. When working on any n8n-related task, follow these guidelines:

### Use the n8n-mcp MCP Server

The `n8n-mcp` MCP server is configured and provides direct access to the n8n instance. Always use it for:
- **Node discovery**: `search_nodes`, `get_node` (with detail levels: minimal, standard, full)
- **Validation**: `validate_node`, `validate_workflow`, `n8n_validate_workflow`
- **Workflow management**: `n8n_create_workflow`, `n8n_update_partial_workflow`, `n8n_get_workflow`, `n8n_list_workflows`, `n8n_delete_workflow`
- **Auto-fixing**: `n8n_autofix_workflow` for common issues
- **Testing**: `n8n_test_workflow` to execute and verify workflows
- **Templates**: `search_templates`, `get_template`, `n8n_deploy_template`
- **Documentation**: `tools_documentation` for tool usage reference

### Use the n8n Skills

Seven skills are installed in `~/.claude/skills/` that provide expert guidance. Lean on them:

| Skill | When to Use |
|-------|-------------|
| `n8n-mcp-tools-expert` | Choosing and using MCP tools correctly (use first) |
| `n8n-workflow-patterns` | Designing workflow architecture (webhook, HTTP, DB, AI, scheduled) |
| `n8n-node-configuration` | Configuring nodes with correct properties and operations |
| `n8n-expression-syntax` | Writing `{{}}` expressions, accessing `$json`/`$node` variables |
| `n8n-validation-expert` | Interpreting and fixing validation errors |
| `n8n-code-javascript` | Writing JavaScript in Code nodes (`$input`, `$helpers`, DateTime) |
| `n8n-code-python` | Writing Python in Code nodes (standard library only) |

### Workflow for n8n Tasks

1. **Understand the goal** - clarify what the workflow should do
2. **Pick a pattern** - use `n8n-workflow-patterns` to choose the right architecture
3. **Find nodes** - use `search_nodes` via MCP to discover the right nodes
4. **Configure nodes** - use `get_node` for property details, `n8n-node-configuration` for guidance
5. **Write expressions** - use `n8n-expression-syntax` for correct `{{}}` patterns
6. **Build the workflow** - use `n8n_create_workflow` or `n8n_update_partial_workflow`
7. **Validate** - always run `n8n_validate_workflow` after changes
8. **Fix issues** - use `n8n-validation-expert` or `n8n_autofix_workflow`
9. **Test** - use `n8n_test_workflow` to verify

### Key Reminders

- Always validate workflows after making changes
- Webhook data lives under `$json.body` (common gotcha)
- Use `n8n_update_partial_workflow` for incremental edits rather than full rewrites
- Check `tools_documentation` via MCP if unsure about any tool's parameters
- Gmail sendAndWait with `approvalType: "double"` outputs `{ data: { approved: true/false } }` - use boolean check on `$json.data.approved`, NOT string check on `$json.response`
- Gmail Trigger may not extract body from forwarded/multipart emails - use a Gmail Get node after the trigger to re-fetch full content
- nodeType format differs between tools: `nodes-base.*` for search/validate vs `n8n-nodes-base.*` for workflow creation
