# GitHub Copilot Chat Integration - Technical Documentation

## Overview

This document explains how the "Add Commit to Chat" feature works in the vscode-git-graph extension, allowing users to send git commit information directly to GitHub Copilot Chat for AI-powered analysis.

## Problem Statement

The goal was to:
1. **Research** how VS Code's built-in SCM graph integrates with GitHub Copilot Chat
2. **Understand** the "add to chat" feature mechanism
3. **Implement** similar functionality in mhutchie.git-graph extension

## How VS Code's Built-in SCM Graph Works

### VS Code's Architecture
- **SCM Provider API**: VS Code has a Source Control Management (SCM) API that extensions can use
- **Built-in Git Extension**: VS Code includes a built-in git extension (`vscode.git`) that implements the SCM API
- **Timeline View**: Shows git history with commit graph visualization
- **Context Menu Integration**: Allows extensions to add context menu items to SCM views

### Chat Integration Mechanism
VS Code provides the `workbench.action.chat.open` command that:
- Opens the Copilot Chat panel
- Accepts a `query` parameter to pre-fill the chat input
- Can be called from any extension via `vscode.commands.executeCommand()`

## Implementation in mhutchie.git-graph

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     VS Code Extension Host                   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Git Graph Extension (extension.ts)            │  │
│  │                                                        │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │    Git Graph View (gitGraphView.ts)            │  │  │
│  │  │  - WebView Panel Controller                    │  │  │
│  │  │  - Message Handler                              │  │  │
│  │  │  - Data Source Interface                        │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  │                       ↕                               │  │
│  │         Message Passing (postMessage)                 │  │
│  │                       ↕                               │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │         WebView (web/main.ts)                  │  │  │
│  │  │  - Git Graph UI                                │  │  │
│  │  │  - Context Menus                               │  │  │
│  │  │  - User Interactions                           │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                               │
│                       Calls Command ↓                         │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           VS Code Chat API                            │  │
│  │     workbench.action.chat.open                        │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

#### 1. Type Definitions (src/types.ts)

Added new message types for the WebView ↔ Extension communication:

```typescript
export interface RequestAddCommitToChat extends RepoRequest {
    readonly command: 'addCommitToChat';
    readonly commitHash: string;
}

export interface ResponseAddCommitToChat extends ResponseWithErrorInfo {
    readonly command: 'addCommitToChat';
}
```

These types are added to the `RequestMessage` and `ResponseMessage` union types.

#### 2. Configuration (src/config.ts)

Added a new visibility setting for the context menu item:

```typescript
commit: { 
    // ... other options
    addToChat: true  // Controls visibility of "Add Commit to Chat" menu item
}
```

Users can configure this via:
```json
{
    "git-graph.contextMenuActionsVisibility": {
        "commit": {
            "addToChat": true
        }
    }
}
```

#### 3. WebView UI (web/main.ts)

Added context menu item in the commit context menu:

```typescript
{
    title: 'Add Commit to Chat',
    visible: visibility.addToChat,
    onClick: () => {
        sendMessage({ 
            command: 'addCommitToChat', 
            repo: this.currentRepo, 
            commitHash: hash 
        });
    }
}
```

This appears in the right-click menu for any commit in the graph.

#### 4. Message Handler (src/gitGraphView.ts)

The core implementation that handles the request:

```typescript
case 'addCommitToChat':
    try {
        // 1. Fetch commit details from git
        const commitData = await this.dataSource.getCommitDetails(
            msg.repo, 
            msg.commitHash, 
            true
        );
        
        if (commitData.commitDetails) {
            const commit = commitData.commitDetails;
            
            // 2. Format commit information for Copilot Chat
            let chatQuery = `Analyze this Git commit:\n\n`;
            chatQuery += `**Commit:** ${msg.commitHash}\n`;
            chatQuery += `**Author:** ${commit.author} <${commit.authorEmail}>\n`;
            chatQuery += `**Date:** ${new Date(commit.authorDate * 1000).toLocaleString()}\n\n`;
            chatQuery += `**Message:**\n${commit.body}\n\n`;
            
            // 3. Add file changes summary
            if (commit.fileChanges.length > 0) {
                chatQuery += `**Files Changed (${commit.fileChanges.length}):**\n`;
                for (const file of commit.fileChanges) {
                    const changeType = /* map file.type to human-readable string */;
                    chatQuery += `- ${changeType}: ${file.newFilePath}`;
                    if (file.additions !== null && file.deletions !== null) {
                        chatQuery += ` (+${file.additions}, -${file.deletions})`;
                    }
                    chatQuery += '\n';
                }
            }
            
            // 4. Open Copilot Chat with the formatted query
            await vscode.commands.executeCommand('workbench.action.chat.open', {
                query: chatQuery,
                isPartialQuery: false
            });
            
            // 5. Send success response back to WebView
            this.sendMessage({
                command: 'addCommitToChat',
                error: null
            });
        }
    } catch (error) {
        // Handle errors and send error response
        this.sendMessage({
            command: 'addCommitToChat',
            error: error.message
        });
    }
    break;
```

### Data Flow

1. **User Action**: User right-clicks on a commit in Git Graph and selects "Add Commit to Chat"
2. **WebView → Extension**: WebView sends `RequestAddCommitToChat` message with commit hash
3. **Extension Processing**:
   - Fetches complete commit details using DataSource
   - Formats data into markdown with:
     - Commit hash
     - Author information
     - Commit date
     - Full commit message
     - File changes with additions/deletions stats
4. **VS Code Command**: Calls `workbench.action.chat.open` with formatted query
5. **Copilot Chat**: Opens with pre-filled commit analysis prompt
6. **Response**: Extension sends success/error response back to WebView

### Commit Information Format

The formatted commit information sent to Copilot Chat looks like:

```markdown
Analyze this Git commit:

**Commit:** abc123def456789...
**Author:** John Doe <john@example.com>
**Date:** 2/6/2026, 6:30:00 PM

**Message:**
Fix bug in user authentication

This commit resolves an issue where users couldn't log in
with special characters in their passwords.

**Files Changed (3):**
- Modified: src/auth.ts (+15, -5)
- Modified: tests/auth.test.ts (+25, -3)
- Added: docs/security.md (+50, -0)
```

### VS Code Chat API Integration

The integration uses the `workbench.action.chat.open` command which:
- **Command**: `'workbench.action.chat.open'`
- **Parameters**: 
  - `query`: The text to pre-fill in the chat input
  - `isPartialQuery`: Whether the query should auto-submit (false = auto-submit)
- **Requirements**: 
  - Works with VS Code 1.90+ (when Chat API was introduced)
  - Requires GitHub Copilot Chat extension to be installed
  - Falls back gracefully if chat is not available

### Error Handling

The implementation includes comprehensive error handling:

1. **Commit Not Found**: Returns error message if commit details can't be fetched
2. **Chat Command Unavailable**: Catches exception if `workbench.action.chat.open` fails
3. **User Feedback**: Sends error response back to WebView for display

## Key Insights from Research

### Built-in Graph vs mhutchie.git-graph

**VS Code Built-in SCM Graph:**
- Limited graph visualization
- Basic timeline view
- No dedicated "add to chat" button (uses general context attachment)
- Integrated directly into SCM view

**mhutchie.git-graph Extension:**
- Rich, interactive graph visualization
- Full-featured Git operations
- Custom WebView-based UI
- Now has direct "Add Commit to Chat" integration

### Chat Context Mechanisms

Research revealed several ways to add context to Copilot Chat:

1. **Direct Command** (✅ Implemented):
   - `workbench.action.chat.open` with query parameter
   - Simple, reliable, works across VS Code versions

2. **Chat Participants** (Not Implemented):
   - Register custom chat participant with `vscode.chat.createChatParticipant`
   - Requires VS Code 1.90+
   - More complex but allows richer interactions

3. **Context Variables** (Not Implemented):
   - Use `#selection`, `#file`, etc. in queries
   - Limited to predefined variable types

### Why This Approach?

The chosen implementation using `workbench.action.chat.open`:
- ✅ **Simple**: Single command call
- ✅ **Minimal Changes**: No new dependencies or APIs
- ✅ **User-Friendly**: Automatic chat opening with context
- ✅ **Maintainable**: Clear, straightforward code
- ✅ **Flexible**: Easy to extend formatting in the future

## Usage

### For Users

1. Open Git Graph view
2. Right-click any commit in the graph
3. Select "Add Commit to Chat"
4. Copilot Chat opens with commit details pre-loaded
5. Ask questions or request analysis from Copilot

### Configuration

Disable the feature if desired:

```json
{
    "git-graph.contextMenuActionsVisibility": {
        "commit": {
            "addToChat": false
        }
    }
}
```

## Future Enhancements

Potential improvements:
- Add support for comparing two commits in chat
- Include actual diff content for small changes
- Add quick prompts (e.g., "Explain this commit", "Find bugs", "Suggest improvements")
- Integrate with Language Model API for even richer experiences

## References

- [VS Code Chat API Documentation](https://code.visualstudio.com/api/extension-guides/ai/chat)
- [VS Code Language Model API](https://code.visualstudio.com/api/extension-guides/ai/language-model)
- [GitHub Copilot Chat Extension](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat)
- [VS Code SCM API](https://code.visualstudio.com/api/extension-guides/scm-provider)

## Conclusion

This implementation successfully brings GitHub Copilot Chat integration to the popular Git Graph extension, allowing users to leverage AI assistance for understanding their git history directly from the visual commit graph interface.
