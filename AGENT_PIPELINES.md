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

## 2. RAG (Knowledge Base) Pipeline

The Retrieval-Augmented Generation pipeline for answering questions using uploaded documents and knowledge bases.

### Flow Diagram

```
+------------------+     +----------------------+     +------------------------+
|   User Query     | --> | Check Knowledge      | --> | Query Rewrite          |
+------------------+     | Base Enabled         |     | (if chat history)      |
                         +----------------------+     +------------------------+
                                                               |
                                                               v
+------------------+     +----------------------+     +------------------------+
| AI Response with | <-- | Knowledge Base QA    | <-- | Semantic Search        |
| Context          |     | Prompt Injection     |     | (Vector Similarity)    |
+------------------+     +----------------------+     +------------------------+
```

### Pipeline Stages

#### Stage 1: Knowledge Base Check
**Location:** `src/store/chat/slices/aiChat/actions/rag.ts`

```
User Message Received
    |
    v
Check hasEnabledKnowledge()
    |
    +-- No --> Standard Chat Pipeline
    |
    +-- Yes --> Continue to Query Rewrite
```

#### Stage 2: Query Rewrite (Optional)
**Location:** `src/store/chat/slices/aiChat/actions/rag.ts` - `internal_rewriteQuery()`

**Condition:** Only if there's chat history (follow-up questions)

```
Original User Query + Chat History
    |
    v
+------------------------------------------+
| chainRewriteQuery Prompt:                |
| "Given the following conversation and a  |
| follow-up question, rephrase the follow  |
| up question to be a standalone question" |
+------------------------------------------+
    |
    v
AI Model Call (streaming)
    |
    v
Standalone Query for Better Retrieval
```

**Prompts Used:**
- `chainRewriteQuery` (packages/prompts/src/chains/rewriteQuery.ts)

**Data Flow:**
```
Input: {
  query: "What about the other features?",
  context: ["How does React work?", "React uses a virtual DOM..."]
}

Output: "What are the other features of React besides the virtual DOM?"
```

#### Stage 3: Semantic Search
**Location:** `src/services/rag/index.ts`

```
Rewritten Query (or Original)
    |
    v
+------------------------------------------+
| Embedding Generation                     |
| Convert query to vector representation   |
+------------------------------------------+
    |
    v
+------------------------------------------+
| Vector Similarity Search                 |
| Search across:                           |
| - Knowledge base files                   |
| - User uploaded files                    |
+------------------------------------------+
    |
    v
Retrieved Chunks with Similarity Scores
[
  { text: "...", similarity: 0.92, fileId: "...", pageNumber: 3 },
  { text: "...", similarity: 0.87, fileId: "...", pageNumber: 1 },
  ...
]
```

#### Stage 4: Context Injection
**Location:** `packages/prompts/src/prompts/knowledgeBaseQA/`

```
Retrieved Chunks + Original Query + Knowledge Base Metadata
    |
    v
+------------------------------------------+
| knowledgeBaseQAPrompts Builder:          |
|                                          |
| 1. knowledgePrompts(knowledge)           |
|    - List knowledge bases with metadata  |
|                                          |
| 2. chunkPrompts(chunks)                  |
|    - Format retrieved chunks with        |
|      similarity scores                   |
|                                          |
| 3. userQueryPrompt(query, rewriteQuery)  |
|    - Include both raw and rewritten      |
|                                          |
+------------------------------------------+
    |
    v
Inject into System Message
```

**Generated Context Example:**
```xml
<knowledge_base_qa_info>
You are also a helpful assistant good answering questions related to React/TypeScript.

<knowledge_base_anwser_instruction>
- Note that passages might not be relevant to the question, please only use relevant passages.
- If there is no relevant passage, please answer using your knowledge.
- Answer should use the same original language as the question.
</knowledge_base_anwser_instruction>

<knowledge_bases>
<knowledge id="kb1" name="React Docs" type="file" fileType="md">Official React documentation</knowledge>
</knowledge_bases>

<retrieved_chunks>
<chunk fileId="f1" fileName="hooks.md" similarity="0.92" pageNumber="3">
The useState hook allows you to add state to functional components...
</chunk>
</retrieved_chunks>

<user_query>
<raw_query>What about the other features?</raw_query>
<rewrite_query>What are the other features of React besides the virtual DOM?</rewrite_query>
</user_query>
</knowledge_base_qa_info>
```

#### Stage 5: AI Response Generation
**Location:** Standard chat pipeline with enhanced context

```
Enhanced System Message + Chat History + User Message
    |
    v
AI Model Call (streaming)
    |
    v
Response with Knowledge Base Context
```

### Data Flow Summary

| Stage | Input | Output | Prompt Used |
|-------|-------|--------|-------------|
| Query Rewrite | query + history | standalone query | `chainRewriteQuery` |
| Semantic Search | query | chunks with similarity | (embedding model) |
| Context Injection | chunks + metadata | XML context | `knowledgeBaseQAPrompts` |
| Response Generation | context + messages | AI response | (main model) |

---

## 3. Translation Pipeline

The translation pipeline handles message translation with automatic language detection.

### Flow Diagram

```
+------------------+     +----------------------+     +------------------------+
|   Message to     | --> | Language Detection   | --> | Translation Request    |
|   Translate      |     | (Parallel)           |     | (Streaming)            |
+------------------+     +----------------------+     +------------------------+
                                  |                            |
                                  v                            v
                         +----------------------+     +------------------------+
                         | Detect Source        |     | Translate to Target    |
                         | Language (locale)    |     | Language               |
                         +----------------------+     +------------------------+
                                  |                            |
                                  +-------------+--------------+
                                                |
                                                v
                                       +----------------+
                                       | Update Message |
                                       | with translate |
                                       | metadata       |
                                       +----------------+
```

### Pipeline Stages

#### Stage 1: Translation Request
**Location:** `src/store/chat/slices/translate/action.ts` - `translateMessage()`

```
User clicks "Translate" on message
    |
    v
Get message content by ID
    |
    v
Initialize translate metadata:
{ content: '', from: '', to: targetLang }
```

#### Stage 2: Language Detection (Parallel)
**Location:** `src/store/chat/slices/translate/action.ts`

**Runs in parallel with translation**

```
Message Content
    |
    v
+------------------------------------------+
| chainLangDetect Prompt:                  |
| "你是一名精通全世界语言的语言专家，       |
| 你需要识别用户输入的内容，以国际标准      |
| locale 进行输出"                         |
+------------------------------------------+
    |
    v
AI Model Call
    |
    v
Detected Locale (e.g., "zh-CN", "en-US")
```

**Prompts Used:**
- `chainLangDetect` (packages/prompts/src/chains/langDetect.ts)

**Data Flow:**
```
Input: "这是一段中文文本"
Output: "zh-CN"
```

#### Stage 3: Translation (Parallel)
**Location:** `src/store/chat/slices/translate/action.ts`

```
Message Content + Target Language
    |
    v
+------------------------------------------+
| chainTranslate Prompt:                   |
| "You are a professional translator.      |
| Translate the input text to {targetLang}"|
|                                          |
| Rules:                                   |
| - Preserve technical terms               |
| - Maintain formatting                    |
| - Use natural expressions                |
+------------------------------------------+
    |
    v
AI Model Call (streaming)
    |
    v
Translated Text (streamed to UI)
```

**Prompts Used:**
- `chainTranslate` (packages/prompts/src/chains/translate.ts)

**Data Flow:**
```
Input: {
  content: "这是一段中文文本",
  targetLang: "en-US"
}

Output: "This is a Chinese text"
```

#### Stage 4: Result Aggregation
**Location:** `src/store/chat/slices/translate/action.ts`

```
Language Detection Result + Translation Result
    |
    v
Update message translate metadata:
{
  content: translatedText,
  from: detectedLocale,
  to: targetLang
}
    |
    v
Save to database
```

### Data Flow Summary

| Stage | Input | Output | Prompt Used |
|-------|-------|--------|-------------|
| Language Detection | message content | locale code | `chainLangDetect` |
| Translation | content + targetLang | translated text | `chainTranslate` |
| Aggregation | both results | translate metadata | - |

### Parallel Execution

Both language detection and translation run concurrently for better performance:

```
translateMessage(id, targetLang)
    |
    +---> chatService.fetchPresetTaskResult(chainLangDetect)  ----+
    |                                                              |
    +---> chatService.fetchPresetTaskResult(chainTranslate)  ----+
                                                                   |
                                                                   v
                                                        Merge Results
```

---

## 4. Group Chat Pipeline

The group chat pipeline orchestrates multi-agent conversations with a supervisor managing which agents speak and when.

### Flow Diagram

```
+------------------+     +----------------------+     +------------------------+
|   User Message   | --> | Create User Message  | --> | Supervisor Decision    |
+------------------+     +----------------------+     | (Debounced)            |
                                                      +------------------------+
                                                               |
        +------------------------------------------------------+
        |
        v
+-------------------+     +----------------------+     +------------------------+
| Supervisor LLM    | --> | Parse Tool Calls     | --> | Execute Agent          |
| (with tools)      |     | (trigger_agent, etc) |     | Responses              |
+-------------------+     +----------------------+     +------------------------+
        |                                                      |
        |                          +---------------------------+
        |                          |
        v                          v
+-------------------+     +------------------------+
| Update Todo List  |     | For Each Agent:        |
| (if productive)   |     | - Build system prompt  |
+-------------------+     | - Filter messages      |
                          | - Generate response    |
                          +------------------------+
                                   |
                                   v
                          +------------------------+
                          | Trigger Next           |
                          | Supervisor Decision    |
                          | (if more agents)       |
                          +------------------------+
```

### Pipeline Stages

#### Stage 1: User Message Creation
**Location:** `src/store/chat/slices/aiChat/actions/generateAIGroupChat.ts` - `sendGroupMessage()`

```
User types message in group chat
    |
    v
Check for @mentions (extract agent IDs)
    |
    v
Create user message with:
{
  content: message,
  role: 'user',
  groupId: groupId,
  targetId: targetMemberId (if DM)
}
    |
    v
Save to database
```

#### Stage 2: Supervisor Decision (Debounced)
**Location:** `src/store/chat/slices/aiChat/actions/generateAIGroupChat.ts` - `internal_triggerSupervisorDecision()`

**Debounce Thresholds:**
- Fast: 1500ms
- Medium: 5000ms (default)
- Slow: 8000ms

```
Messages + Group Config + Agent List
    |
    v
+------------------------------------------+
| contextSupervisorMakeDecision Builder:   |
|                                          |
| 1. groupSupervisorPrompts(messages)      |
|    - Format messages with author/target  |
|                                          |
| 2. buildSupervisorPrompt(context)        |
|    - Build orchestration instructions    |
|    - Include todo list (if productive)   |
|    - Include agent roster                |
|                                          |
+------------------------------------------+
    |
    v
+------------------------------------------+
| Supervisor Tools Available:              |
| - trigger_agent (group message)          |
| - trigger_agent_dm (DM to agent/user)    |
| - wait_for_user_input (pause)            |
| - create_todo (productive mode)          |
| - finish_todo (productive mode)          |
+------------------------------------------+
    |
    v
AI Model Call (JSON mode)
    |
    v
Parse tool calls into decisions
```

**Prompts Used:**
- `buildSupervisorPrompt` (packages/prompts/src/prompts/groupChat/)
- `groupSupervisorPrompts` (packages/prompts/src/prompts/chatMessages/)

**Supervisor Response Example:**
```json
[
  { "tool_name": "create_todo", "parameter": { "content": "Research topic", "assignee": "agent1" } },
  { "tool_name": "trigger_agent", "parameter": { "id": "agent1", "instruction": "Research the topic" } },
  { "tool_name": "trigger_agent", "parameter": { "id": "agent2", "instruction": "Wait for research" } }
]
```

#### Stage 3: Execute Agent Responses
**Location:** `src/store/chat/slices/aiChat/actions/generateAIGroupChat.ts` - `internal_executeAgentResponses()`

**For each decision in supervisor response:**

```
Decision { id: agentId, instruction, target }
    |
    v
+------------------------------------------+
| Response Order:                          |
| - Sequential: Process one by one         |
|   with 1500ms delay between              |
| - Parallel: Process all simultaneously   |
+------------------------------------------+
    |
    v
internal_processAgentMessage(groupId, agentId, targetId, instruction)
```

#### Stage 4: Individual Agent Response
**Location:** `src/store/chat/slices/aiChat/actions/generateAIGroupChat.ts` - `internal_processAgentMessage()`

```
Agent Config + Messages + Instruction
    |
    v
+------------------------------------------+
| filterMessagesForAgent(messages, agentId)|
|                                          |
| Rules:                                   |
| - See all group messages (no targetId)   |
| - See DMs where agent is target          |
| - See DMs agent sent                     |
| - Hide other DMs (show "***")            |
+------------------------------------------+
    |
    v
+------------------------------------------+
| buildGroupChatSystemPrompt:              |
|                                          |
| 1. Agent's base system role              |
| 2. Group chat guidelines                 |
| 3. Group member roster                   |
| 4. Current instruction from supervisor   |
| 5. Target info (if DM)                   |
+------------------------------------------+
    |
    v
Prepare messages with author tags:
<author_name_do_not_include_in_your_response name="AgentName" id="agent1" />
    |
    v
AI Model Call (streaming)
    |
    v
+------------------------------------------+
| If tool call detected:                   |
| - Execute tool                           |
| - Call same agent again (follow-up)      |
|                                          |
| Else:                                    |
| - Save response                          |
| - Continue to next agent                 |
+------------------------------------------+
```

**Prompts Used:**
- `buildGroupChatSystemPrompt` (packages/prompts/src/prompts/groupChat/)
- `filterMessagesForAgent` (packages/prompts/src/prompts/chatMessages/)

#### Stage 5: Loop Back to Supervisor
**Location:** `src/store/chat/slices/aiChat/actions/generateAIGroupChat.ts`

```
All agents in current batch responded
    |
    v
Check maxResponseInRow limit
    |
    +-- Exceeded --> Wait for user input
    |
    +-- Not exceeded --> Trigger supervisor decision again (debounced)
```

### Data Flow Summary

| Stage | Input | Output | Prompt Used |
|-------|-------|--------|-------------|
| User Message | user input | message record | - |
| Supervisor Decision | messages + agents | tool calls | `buildSupervisorPrompt` |
| Parse Decisions | tool calls | decision list | - |
| Agent Response | filtered messages | agent message | `buildGroupChatSystemPrompt` |
| Loop Control | response count | continue/stop | - |

### Supervisor Tool Calls

| Tool | Purpose | Parameters |
|------|---------|------------|
| `trigger_agent` | Make agent speak to group | `{ id, instruction }` |
| `trigger_agent_dm` | Agent sends DM | `{ id, instruction, target }` |
| `wait_for_user_input` | Pause conversation | `{ reason? }` |
| `create_todo` | Add todo item | `{ content, assignee }` |
| `finish_todo` | Complete todo | `{ index }` |

### Scene Modes

**Casual Mode:**
- No todo management
- Only trigger_agent, trigger_agent_dm, wait_for_user_input tools

**Productive Mode:**
- Full todo management
- All tools available including create_todo, finish_todo

---
