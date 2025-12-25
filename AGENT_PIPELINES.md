# LobeChat Agent Pipelines Documentation

This document describes all possible agent workflow schemes in LobeChat, including the sequence of prompt calls, data transformations, and the flow of information between pipeline stages.

## Table of Contents

1. [Simple Chat Pipeline](#1-simple-chat-pipeline)
2. [RAG (Knowledge Base) Pipeline](#2-rag-knowledge-base-pipeline)
3. [Translation Pipeline](#3-translation-pipeline)
4. [Group Chat Pipeline](#4-group-chat-pipeline)
5. [Topic Summary Pipeline](#5-topic-summary-pipeline)
6. [Tool Calling Pipeline](#6-tool-calling-pipeline)
7. [Memory Compression Pipeline](#7-memory-compression-pipeline)
8. [Agent Creation Pipeline](#8-agent-creation-pipeline)

---

## 1. Simple Chat Pipeline

The basic conversation flow where a user sends a message and receives an AI response.

### Flow Diagram

```
+------------------+     +----------------------+     +--------------------+
|   User Message   | --> | Context Engine       | --> | AI Model Request   |
+------------------+     | Pipeline             |     +--------------------+
                         +----------------------+              |
                                  |                            v
                                  |                   +--------------------+
                                  |                   | Stream Response    |
                                  v                   +--------------------+
                         +----------------------+              |
                         | 1. SystemRoleInjector|              v
                         +----------------------+     +--------------------+
                                  |                   | Update UI &        |
                                  v                   | Save to DB         |
                         +----------------------+     +--------------------+
                         | 2. ToolSystemRole    |
                         |    Provider          |
                         +----------------------+
                                  |
                                  v
                         +----------------------+
                         | 3. HistorySummary    |
                         |    Provider          |
                         +----------------------+
                                  |
                                  v
                         +----------------------+
                         | 4. Files Context     |
                         |    Injection         |
                         +----------------------+
```

### Pipeline Stages

#### Stage 1: Message Creation
**Location:** `src/store/chat/slices/aiChat/actions/generateAIChat.ts`

```
User Input
    |
    v
Create User Message Record
    |
    v
Dispatch to Message Store
```

#### Stage 2: Context Engine Processing
**Location:** `packages/context-engine/src/`

```
Raw Messages Array
    |
    +--> SystemRoleInjector
    |         |
    |         v
    |    Inject agent's system role as first message
    |    (if not already present)
    |
    +--> ToolSystemRoleProvider
    |         |
    |         v
    |    Append tool-specific instructions if:
    |    - Tools are enabled
    |    - Model supports function calling
    |
    +--> HistorySummaryProvider
    |         |
    |         v
    |    Inject conversation summary if:
    |    - Topic has historySummary
    |    - Long conversation detected
    |
    +--> FilesContextProvider
              |
              v
         Inject file/image context if:
         - User uploaded files
         - Images in conversation
```

#### Stage 3: API Request Formation
**Location:** `src/services/chat/index.ts`

```
Processed Messages
    |
    v
Build ChatStreamPayload:
{
  messages: [...processedMessages],
  model: agentConfig.model,
  provider: agentConfig.provider,
  temperature: agentConfig.temperature,
  tools: enabledTools (if any),
  ...otherParams
}
    |
    v
Send to AI Provider API
```

#### Stage 4: Response Streaming
**Location:** `src/store/chat/slices/aiChat/actions/generateAIChat.ts`

```
Stream Chunks
    |
    +--> Text Chunks
    |         |
    |         v
    |    Append to message content
    |    Update UI in real-time
    |
    +--> Tool Call Chunks
    |         |
    |         v
    |    Parse tool call JSON
    |    Trigger tool execution
    |
    +--> Finish Signal
              |
              v
         Save final message to DB
         Update message metadata
```

### Data Passed Between Stages

| From | To | Data |
|------|-----|------|
| User Input | Message Creation | `{ content, files, role: 'user' }` |
| Message Creation | Context Engine | `UIChatMessage[]` |
| Context Engine | API Request | `{ messages, model, provider, tools, ... }` |
| API Response | UI Update | `{ type: 'text'/'tool', text/tool_calls }` |

---
