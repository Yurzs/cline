# Testing MCP Roots Support

## Manual Testing Guide

This guide explains how to test the MCP Roots support implementation in Cline.

## Prerequisites

1. An MCP server that supports the roots capability
2. Cline extension with the roots implementation installed
3. A VS Code workspace (single or multi-root)

## Test Scenarios

### Scenario 1: Single Workspace Root

**Setup:**
1. Open a single folder in VS Code
2. Configure an MCP server in Cline's settings

**Expected Behavior:**
- When the MCP server connects, it can call `roots/list` and receive:
  ```json
  {
    "roots": [
      {
        "uri": "file:///path/to/workspace",
        "name": "workspace"
      }
    ]
  }
  ```

### Scenario 2: Multi-Root Workspace

**Setup:**
1. Create a multi-root workspace in VS Code (File > Add Folder to Workspace)
2. Save the workspace
3. Configure an MCP server

**Expected Behavior:**
- The MCP server receives multiple roots:
  ```json
  {
    "roots": [
      {
        "uri": "file:///path/to/workspace1",
        "name": "workspace1"
      },
      {
        "uri": "file:///path/to/workspace2",
        "name": "workspace2"
      }
    ]
  }
  ```

### Scenario 3: Workspace Change Notification

**Setup:**
1. Start with a workspace open
2. Connect an MCP server
3. Add a new folder to the workspace

**Expected Behavior:**
- The MCP server receives a `notifications/roots/list_changed` notification
- When the server re-requests `roots/list`, it receives the updated list

### Scenario 4: Server Connects Before Workspace Manager

**Setup:**
1. Start Cline with an MCP server configured
2. The workspace manager may not be initialized yet

**Expected Behavior:**
- The MCP server can still connect successfully
- Calling `roots/list` returns an empty array `[]` initially
- Once the workspace manager is initialized, the server receives a `notifications/roots/list_changed` notification
- Subsequent `roots/list` calls return the actual workspace roots

## Creating a Test MCP Server

Here's a simple example of an MCP server that uses roots (in Python):

```python
import asyncio
from mcp.server import Server
from mcp.types import Root

async def main():
    server = Server("test-roots-server")
    
    @server.list_roots()
    async def list_roots() -> list[Root]:
        # This would be implemented by the client
        pass
    
    # When initialized, request roots from client
    @server.on_initialize()
    async def on_initialize():
        roots = await server.request_roots_list()
        print(f"Received roots: {roots}")
        
        # Listen for root changes
        @server.on_roots_list_changed()
        async def on_roots_changed():
            roots = await server.request_roots_list()
            print(f"Roots changed: {roots}")
    
    await server.run()

if __name__ == "__main__":
    asyncio.run(main())
```

## Verification Checklist

- [ ] MCP server can successfully call `roots/list`
- [ ] Response contains correct workspace URIs
- [ ] Response includes workspace names (optional)
- [ ] URIs are properly formatted as `file://` URIs
- [ ] Multi-root workspaces return multiple roots
- [ ] Server receives `notifications/roots/list_changed` when workspace changes
- [ ] Disabled servers do not receive notifications
- [ ] Server can request roots immediately after connection

## Debugging

To debug the roots implementation:

1. Check the Cline output logs for MCP-related messages
2. Look for logs like:
   - "Setting up notification handlers for server"
   - "Sent roots/list_changed notification to [server]"
3. Verify that the workspace manager is properly initialized
4. Check that the server declares roots capability support

## Common Issues

### Empty Roots List
- **Cause**: Workspace manager not yet initialized
- **Solution**: Wait for workspace manager initialization or trigger it by opening a task

### No Change Notifications
- **Cause**: Previous and new roots are identical
- **Solution**: This is expected behavior - notifications only sent when roots actually change

### Server Not Receiving Notifications
- **Cause**: Server is disabled or client connection failed
- **Solution**: Check server status and connection
