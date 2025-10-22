# MCP Roots Support Implementation

## Overview
This document describes the implementation of MCP (Model Context Protocol) Roots support in Cline, as specified in the [MCP Specification](https://modelcontextprotocol.io/specification/2025-06-18/client/roots).

## What are MCP Roots?
MCP Roots allow MCP clients to inform servers about the workspace directories (roots) that are available. This helps servers understand the context in which they're operating, enabling them to provide better, context-aware services.

## Implementation Details

### 1. Client Capability Declaration
In `src/services/mcp/McpHub.ts`, the client now declares the `roots` capability when creating an MCP client connection:

```typescript
const client = new Client(
    {
        name: "Cline",
        version: this.clientVersion,
    },
    {
        capabilities: {
            roots: {
                listChanged: true,
            },
        },
    },
)
```

This tells MCP servers that:
- The client supports the roots feature
- The client will notify servers when the roots list changes (`listChanged: true`)

### 2. Request Handler for `roots/list`
The client implements a request handler that responds to `roots/list` requests from servers:

```typescript
client.setRequestHandler(ListRootsResultSchema as any, async () => {
    const roots = this.getRootsList()
    return {
        roots,
    }
})
```

The `getRootsList()` method converts workspace roots to the MCP format:

```typescript
private getRootsList(): Array<{ uri: string; name?: string }> {
    if (!this.workspaceManager) {
        return []
    }

    const roots = this.workspaceManager.getRoots()
    return roots.map((root) => ({
        uri: `file://${root.path}`,
        name: root.name,
    }))
}
```

### 3. Notification on Roots Change
When workspace roots change, the client notifies all connected MCP servers:

```typescript
async setWorkspaceManager(workspaceManager: WorkspaceRootManager | undefined): Promise<void> {
    const previousRoots = this.getRootsList()
    this.workspaceManager = workspaceManager
    const newRoots = this.getRootsList()

    // Only notify if roots actually changed
    if (!deepEqual(previousRoots, newRoots)) {
        await this.notifyRootsChanged()
    }
}

private async notifyRootsChanged(): Promise<void> {
    const roots = this.getRootsList()

    for (const connection of this.connections) {
        if (connection.server.disabled || !connection.client) {
            continue
        }

        try {
            await connection.client.notification({
                method: "notifications/roots/list_changed",
                params: undefined,
            })
        } catch (error) {
            console.error(`Failed to send roots/list_changed notification to ${connection.server.name}:`, error)
        }
    }
}
```

### 4. Integration with Workspace Manager
The `Controller` class has been updated to pass the `WorkspaceRootManager` to the `McpHub`:

```typescript
// In Controller constructor
this.mcpHub = new McpHub(
    () => ensureMcpServersDirectoryExists(),
    () => ensureSettingsDirectoryPath(),
    ExtensionRegistryInfo.version,
    telemetryService,
    undefined, // Initial value, will be set later
)

// When workspace manager is initialized
async ensureWorkspaceManager(): Promise<WorkspaceRootManager | undefined> {
    if (!this.workspaceManager) {
        this.workspaceManager = await setupWorkspaceManager({...})
        await this.mcpHub.setWorkspaceManager(this.workspaceManager)
    }
    return this.workspaceManager
}
```

## Benefits
1. **Context Awareness**: MCP servers can now understand which workspace directories are available, enabling better context-aware operations
2. **Dynamic Updates**: Servers are automatically notified when workspace roots change
3. **Standard Compliance**: Implements the official MCP specification for roots support

## Testing
MCP servers that support roots can now:
- Request the list of workspace roots via `roots/list`
- Receive notifications when the roots list changes
- Use this information to provide better, context-aware services

## Example Use Cases
1. **File System Servers**: Can limit operations to known workspace roots
2. **Version Control Servers**: Can discover repositories in workspace roots
3. **Build Tool Servers**: Can identify project directories automatically
4. **Documentation Servers**: Can index documentation across all workspace roots
