# Agent Architecture Summary

## Overview

The OpenCode agent system is a sophisticated architecture for managing AI-powered task execution with state management, work continuation, and hierarchical session organization.

## Core Concepts

### Agent Types

Agents are defined in `packages/opencode/src/agent/agent.ts:1-251` with three operational modes:

- **`primary`**: Main agents that users interact with directly (e.g., "build", "plan")
- **`subagent`**: Specialized agents invoked by other agents via the task tool (e.g., "general")
- **`all`**: Agents that can function in both roles

### Built-in Agents

Three default agents ship with OpenCode:

1. **general** (subagent)
   - Purpose: General research, codebase exploration, and search tasks
   - Tools: All except todoread/todowrite
   - Full permissions

2. **build** (primary)
   - Purpose: Default agent for code modifications
   - Full tool access and permissions

3. **plan** (primary)
   - Purpose: Planning tasks with restricted execution
   - Limited bash permissions (read-only operations only)
   - Pattern restrictions: `"find * -delete*": "ask"`, `"*": "ask"`

## State Management

### Session State (`packages/opencode/src/session/index.ts:37-75`)

Each session maintains:

```typescript
interface SessionInfo {
  id: string
  parentId?: string              // Hierarchical sessions for subagents
  cwd: string
  mcpServers: McpServer[]
  createdAt: Date
  model: { providerID: string; modelID: string }
  modeId?: string               // Current agent mode
  messages: Message[]           // Conversation history
  revertState?: RevertState     // Undo/rollback support
  summary?: SessionSummary      // File changes tracking
}
```

### State Persistence

State is managed through multiple layers:

1. **ACPSessionManager** (`packages/opencode/src/acp/session.ts:1-68`)
   - In-memory map of active sessions: `Map<string, ACPSessionState>`
   - Session lifecycle: `create()`, `get()`, `setMode()`, `remove()`
   - Model and agent mode switching

2. **Session Storage** (`packages/opencode/src/session/index.ts`)
   - Persistent message and part storage
   - Query methods: `Session.messages()`, `Session.parts()`
   - State mutations: `Session.addPart()`, `Session.updatePart()`

3. **Session Lock** (`packages/opencode/src/session/lock.ts:50-87`)
   - Prevents concurrent agent execution in same session
   - Uses `AbortController` for cancellation
   - Automatic cleanup with `Symbol.dispose`

```typescript
// Exclusive lock pattern
using lock = SessionLock.acquire({ sessionID })
// Work happens here
// Lock automatically released
```

## Work Continuation

### Context Management (`packages/opencode/src/session/compaction.ts:90-200`)

When token limits are approached, the system employs **compaction**:

1. **Detection**: Checks if `tokens > model.contextWindow * 0.8`
2. **Summarization**: Creates compressed summary of old messages
3. **Resume Injection**: Adds synthetic message:
   ```
   "Use the above summary generated from your last session
   to resume from where you left off."
   ```
4. **Continuation**: Agent resumes with compressed context

Implementation in `packages/opencode/src/session/prompt.ts:452-478`:

```typescript
if (SessionCompaction.isOverflow({ tokens, model })) {
  const summaryMsg = await SessionCompaction.run({
    sessionID: input.sessionID,
    providerID: input.providerID,
    modelID: input.model.id,
  })
  const resumeMsg = {
    type: "text",
    text: "Use the above summary...",
    synthetic: true,
  }
  msgs = [summaryMsg, resumeMsg]
}
```

### Pruning (`packages/opencode/src/session/compaction.ts`)

Removes old tool call outputs while preserving:
- Recent 40,000 tokens of work
- Tool call structure for history
- Essential context for continuation

## Agent Execution Flow

### Main Execution Loop (`packages/opencode/src/session/prompt.ts`)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. SessionPrompt.prompt()                                   │
│    ├─ Resolve agent: Agent.get(agentName)                  │
│    ├─ Resolve model: agent.model || Provider.default()     │
│    ├─ Resolve tools: Filter by agent.tools + permissions   │
│    └─ Acquire SessionLock                                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. Get Messages with Compaction Check                       │
│    ├─ Load Session.messages()                              │
│    ├─ Check token count vs context window                  │
│    └─ Apply compaction if needed (summarize + resume)      │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. Stream to AI (Loop while tool-calls returned)           │
│    ├─ Send messages to model                               │
│    ├─ Parse response for tool calls                        │
│    └─ For each tool call:                                  │
│        ├─ Update part status: pending → running            │
│        ├─ Execute tool with context                        │
│        ├─ Stream updates via Bus.publish()                 │
│        └─ Update part status: running → completed/error    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. Post-Processing                                          │
│    ├─ Prune old tool outputs (if > 40K tokens)             │
│    ├─ Store final message + parts                          │
│    ├─ Release SessionLock                                   │
│    └─ Emit SessionPrompt.Event.Idle                        │
└─────────────────────────────────────────────────────────────┘
```

### Tool Execution States

Each tool invocation transitions through states:

- **pending**: Tool call parsed, awaiting execution
- **running**: Tool actively executing
- **completed**: Tool finished successfully
- **error**: Tool encountered an error

State changes are broadcast via `Bus.subscribe(MessageV2.Event.PartUpdated, ...)` in `packages/opencode/src/acp/agent.ts:62-313`.

## Subagent Invocation

### Task Tool (`packages/opencode/src/tool/task.ts:27-104`)

Subagents are invoked hierarchically:

```typescript
async execute(params, ctx) {
  const agent = await Agent.get(params.subagent_type)

  // Create child session with parent reference
  const session = await Session.create({
    parentID: ctx.sessionID,
    title: params.description + ` (@${agent.name} subagent)`,
  })

  // Execute in isolated context
  const result = await SessionPrompt.prompt({
    sessionID: session.id,
    agent: agent.name,
    model: agent.model,
    parts: [{ type: "text", text: params.prompt }],
  })

  // Return condensed output to parent
  return {
    title: params.description,
    output: result.parts.findLast(x => x.type === "text")?.text,
  }
}
```

### Session Hierarchy

```
Parent Session (build agent)
    │
    ├─ uses Task tool
    │
    └─→ Child Session (general subagent)
            │
            ├─ Isolated execution
            ├─ Own message history
            └─ Results bubble up to parent
```

Benefits:
- **Isolation**: Subagent work doesn't pollute parent context
- **Specialization**: Different agents for different tasks
- **Composability**: Agents can invoke other agents recursively

## Permission Management

### Permission Schema (`packages/opencode/src/agent/agent.ts:215-250`)

Permissions are hierarchically merged:

1. Default permissions (allow all)
2. Global config (`~/.config/opencode/opencode.jsonc`)
3. Agent-specific permissions
4. Command-specific overrides

### Permission Types

```typescript
permission: {
  edit: "allow" | "ask" | "deny",
  bash: {
    "pattern*": "allow" | "ask" | "deny"
  },
  webfetch: "allow" | "ask" | "deny"
}
```

### Example: Plan Agent Restrictions

```typescript
{
  bash: {
    "cut*": "allow",
    "grep*": "allow",
    "ls*": "allow",
    "find*": "allow",
    "find * -delete*": "ask",    // Dangerous operations require approval
    "rm*": "ask",
    "*": "ask"                    // Default: ask for everything else
  }
}
```

Wildcard patterns are evaluated in order, with more specific patterns taking precedence.

## Agent Configuration

### Configuration Loading (`packages/opencode/src/config/config.ts:28-123`)

Hierarchy (later overrides earlier):

1. Global: `~/.config/opencode/opencode.jsonc`
2. Project: `opencode.jsonc` in worktree or parent directories
3. Custom agents: `.opencode/agent/*.md` files
4. Environment: `OPENCODE_CONFIG`, `OPENCODE_PERMISSION`

### Markdown-Based Agent Definition

Create `.opencode/agent/my-agent.md`:

```markdown
---
description: Custom agent for specific tasks
mode: subagent
tools:
  bash: false
  read: true
  write: true
temperature: 0.7
top_p: 0.9
permission:
  edit: ask
  bash:
    "git*": allow
    "*": deny
---

You are a specialized agent for [purpose].

Your responsibilities:
- Task 1
- Task 2

Guidelines:
- Guideline 1
- Guideline 2
```

The content after the frontmatter becomes the agent's system prompt.

## Event-Driven Architecture

### Message Bus (`packages/opencode/src/acp/agent.ts:62-313`)

Real-time updates flow through an event bus:

```typescript
// Subscribe to part updates
Bus.subscribe(MessageV2.Event.PartUpdated, (event) => {
  // Stream tool status to ACP clients
  publishPartUpdate(event.part)
})

// Subscribe to permission requests
Bus.subscribe(Permission.Event.Updated, (event) => {
  // Handle user approval/denial
  processPermissionUpdate(event)
})
```

### Event Types

- **MessageV2.Event.PartUpdated**: Tool execution state changes
- **Permission.Event.Updated**: User permission decisions
- **SessionPrompt.Event.Idle**: Agent finished working
- **SessionPrompt.Event.Cancel**: Abort signal received

## Work Sequencing

### Session Lock Mechanics (`packages/opencode/src/session/lock.ts:50-87`)

Ensures only one agent works per session:

```typescript
export function acquire(input: { sessionID: string }) {
  const lock = get(input.sessionID)
  if (lock) throw new LockedError(...)

  const controller = new AbortController()
  state().locks.set(sessionID, {
    controller,
    created: Date.now()
  })

  return {
    signal: controller.signal,
    abort: () => controller.abort(),
    [Symbol.dispose]: () => release(sessionID)
  }
}
```

Usage with automatic cleanup:

```typescript
using lock = SessionLock.acquire({ sessionID: "abc123" })
// Work happens here
// Lock automatically released even if exception thrown
```

### Cancellation

The `AbortController.signal` is passed through the execution stack:
- `SessionPrompt.prompt()` checks signal between tool calls
- Tool implementations receive `signal` in context
- On abort, cleanup happens and `SessionPrompt.Event.Cancel` fires

## Key Implementation Files

| File | Purpose | Key Content |
|------|---------|-------------|
| `packages/opencode/src/agent/agent.ts` | Agent definitions | Schema (1-97), built-in agents (98-130), permissions (215-250) |
| `packages/opencode/src/acp/agent.ts` | ACP protocol | Lifecycle management (504-513), event subscriptions (62-313) |
| `packages/opencode/src/session/prompt.ts` | Execution engine | Main loop (1-1896), compaction (452-478) |
| `packages/opencode/src/session/index.ts` | Session state | State schema (37-75), storage (21-456) |
| `packages/opencode/src/session/lock.ts` | Concurrency control | Lock acquire/release (6-94) |
| `packages/opencode/src/session/compaction.ts` | Context management | Overflow detection (90-200) |
| `packages/opencode/src/acp/session.ts` | Session manager | ACPSessionManager (1-68) |
| `packages/opencode/src/tool/task.ts` | Subagent invocation | Task tool implementation (1-105) |
| `packages/opencode/src/config/config.ts` | Configuration | Config loading (28-123), agent loading (85-86) |

## Advanced Features

### Agent Mode Switching

Switch agents mid-session:

```typescript
async setSessionMode(params: SetSessionModeRequest) {
  const session = this.sessionManager.get(params.sessionId)
  await Agents.get(params.modeId).then((agent) => {
    if (!agent) throw new Error(`Agent not found: ${params.modeId}`)
  })
  this.sessionManager.setMode(params.sessionId, params.modeId)
}
```

This allows:
- Starting in "plan" mode for read-only exploration
- Switching to "build" mode for implementation
- Using custom agents for specialized tasks

### AI-Generated Agents

```typescript
Agent.generate(description: string)
```

Creates agent configurations dynamically based on natural language descriptions.

### Revert State

Sessions can track `revertState` for undo operations:
- Store file states before modifications
- Rollback to previous state on error or user request
- Preserve git history for safety

## Summary

The OpenCode agent system provides:

1. **Flexible Agent Types**: Primary, subagent, and hybrid modes
2. **Robust State Management**: Persistent sessions with hierarchical organization
3. **Intelligent Continuation**: Context compaction and summarization for long-running work
4. **Concurrency Control**: Session locks prevent race conditions
5. **Event-Driven Updates**: Real-time streaming of tool execution status
6. **Permission System**: Granular control over agent capabilities
7. **Hierarchical Execution**: Parent-child sessions for task decomposition
8. **Configuration Flexibility**: Markdown-based agent definitions with inheritance

This architecture enables agents to handle complex, long-running tasks while maintaining state consistency and providing clear visibility into work progress.
