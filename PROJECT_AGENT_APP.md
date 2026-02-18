# Project Agent OS: The OpenClaw Frontend Interface

## Executive Summary
**Agent OS** is not a chat app; it is an Integrated Development Environment (IDE) for Agency. Current interfaces (Discord, Slack, standard chat UIs) treat AI agents as chatbots—linear, text-based conversational partners. This fundamental mismatch limits utility. Agents are recursive, multi-modal, and asynchronous. They think, browse, code, and execute tools in background loops. 

This project aims to build a **local-first, cross-platform (Mobile + Desktop) interface** specifically designed for high-bandwidth human-agent collaboration. It prioritizes observability ("Brain View"), intervention (Pause/Edit), and non-linear communication (Infinite Threading).

## Core Philosophy: "Agent OS" vs. Chat App

| Feature | Standard Chat App (Discord/Slack) | Agent OS (The Goal) |
| :--- | :--- | :--- |
| **Atomic Unit** | Message (Text Bubble) | Action (Thought + Tool + Output) |
| **State** | Ephemeral, Server-Side | Local-First, Persisted, Sync-Aware |
| **Structure** | Linear Stream | Recursive/Tree-based (Infinite Threading) |
| **Visibility** | Final Output Only | "Glass Box" (Live Thinking, Token Count, Context) |
| **Control** | Reply / Reaction | Pause, Intervene, Edit Memory, Fork Session |
| **Input** | Keyboard Dominant | Voice-First (PTT), Multimodal (Screen/Cam) |

## Tech Stack & Libraries (Lazy Genius Audit)

We are not reinventing the wheel. We are assembling a Ferrari from parts.

### 1. Chat UI: `flutter_chat_ui` (Flyer Chat)
*   **Verdict:** **ADOPT & EXTEND.**
*   **Why:** Building a chat UI from scratch is a trap. `flutter_chat_ui` is the most mature, customizable package.
*   **The "Lazy" Win:** It handles the infinite scroll, keyboard avoidance, and basic bubbles perfectly.
*   **The "Genius" Tweak:** We leverage its `custom` message type heavily.
    *   Standard text -> Markdown bubble.
    *   Tool use -> Custom "Terminal" widget.
    *   Thought trace -> Custom "Accordion" widget.
    *   *Note:* We will use `flutter_markdown` with `syntax_highlighter` for code blocks inside the bubbles.

### 2. State Management: `riverpod` (Hooks)
*   **Verdict:** **ADOPT.**
*   **Why:** `bloc` is too much boilerplate for a rapid iteration cycle. `riverpod` allows dynamic provider generation (`family` providers), which is critical for handling **N** parallel agents.
*   **Architecture:**
    *   `agentProvider(id)`: Returns the state of a specific agent session.
    *   `activeAgentIdProvider`: Controls which agent is currently visible.
    *   This makes "switching agents" as instant as changing a variable.

### 3. Local DB: `drift` (over `sqflite`)
*   **Verdict:** **ADOPT.**
*   **Why:** `sqflite` is raw SQL (error-prone). `drift` provides type-safe Dart APIs and reactive streams.
*   **The Win:** We can bind the Chat UI directly to a `watch()` stream from the DB. When an agent writes a message in the background, the UI updates automatically without manual refresh logic.

### 4. Notifications: `flutter_local_notifications`
*   **Verdict:** **ADOPT.**
*   **Why:** Industry standard.
*   **Use Case:** Critical for multi-agent workflows. When Agent A finishes a long task (e.g., "Researching API"), the user gets a system notification: *"Agent A: Task Complete. Click to review."*

### 5. Markdown & Code: `flutter_markdown`
*   **Verdict:** **ADOPT.**
*   **Config:** Must use custom builders for `code` blocks to support syntax highlighting and "Copy" buttons.

---

## Multi-Agent Architecture

The "Agent OS" treats agents like Discord Servers or Browser Tabs, not just a single chat thread.

### The "Session" Model
*   **Concept:** A **Session** is a persistent container for an agent's lifecycle.
*   **State:** Each Session has:
    *   **ID:** UUID.
    *   **Context:** Vector store + Short-term memory.
    *   **Status:** `Idle`, `Thinking`, `Tooling`, `Paused`, `AwaitingInput`.
    *   **Socket:** Dedicated WebSocket connection to the backend.

### Parallel Execution (The "Background" Layer)
Agents must be **independent processes**.
*   **Scenario:** You ask Agent A to "Scrape this website" (takes 2 mins). You immediately switch to Agent B to ask a quick question.
*   **Implementation:**
    *   The `AgentService` in Flutter holds a `Map<String, AgentController>`.
    *   Background agents continue to receive WebSocket events.
    *   If a background agent emits a `NeedsUserAction` event (e.g., "Login required"), it triggers a `local_notification` and adds a "Badge" (red dot) to its icon in the sidebar.

### UI Layout: "The Omnibar"
*   **Left Sidebar (The Agent Dock):**
    *   Vertical list of avatars (like Discord servers).
    *   **Active:** Highlighted.
    *   **Busy:** Spinner overlay on avatar.
    *   **Alert:** Red dot/badge.
*   **State Switching:**
    *   User clicks Agent B.
    *   `activeAgentId` updates.
    *   Main Chat View rebuilds instantly using `agentProvider(agentB_ID)`.
    *   Scroll position is preserved per-agent.

---

## Architecture

```mermaid
graph TD
    subgraph Client_Device [Client (Flutter)]
        UI[UI Layer]
        LocalDB[(SQLite + Vector)]
        SyncEngine[Sync Engine / CRDT]
        
        UI --> LocalDB
        UI --> SyncEngine
        SyncEngine <--> LocalDB
    end

    subgraph Server_Infrastructure [Agent Server]
        Gateway[API Gateway / WebSocket]
        Auth[Auth Service]
        AgentRuntime[OpenClaw Runtime]
        MasterDB[(Postgres)]
        
        Gateway <--> SyncEngine
        Gateway --> Auth
        Gateway --> AgentRuntime
        AgentRuntime --> MasterDB
    end

    subgraph Agent_Brain [Agent Logic]
        LLM[LLM Inference]
        Tools[Tool Executor]
        Memory[Context Manager]
        
        AgentRuntime <--> LLM
        AgentRuntime <--> Tools
        AgentRuntime <--> Memory
    end
```

## UX/UI Concept

### 1. The Main Feed (Center)
- **Not just bubbles:** Messages are "Blocks".
- **Streaming Thoughts:** `<think>` tags render as collapsible, streaming accordions. You see the agent planning in real-time (faded text), then the final answer in bold.
- **Tool Widgets:** When an agent runs `browser.snapshot`, the image renders inline. When it runs `exec`, a mini-terminal window appears in the chat stream.

### 2. The "Brain View" (Right Sidebar / Drawer)
- **HUD for the Agent:**
  - **Context Window:** Visual bar showing 75% utilized (e.g., 90k/128k tokens).
  - **Scratchpad:** A persistent sticky note area shared between user and agent.
  - **Active Sub-agents:** List of background tasks (e.g., "Researching API docs - 45%").
  - **File Browser:** View the agent's current workspace directory.

### 3. Infinite Threading
- Click any message to "drill down". The UI slides, and that message becomes the new root.
- Allows branching conversations without polluting the main timeline.
- Essential for debugging: "Why did you do this?" -> Click tool output -> Start thread -> "Analyze this error".

### 4. Input & Control
- **PTT (Push-to-Talk):** Large floating button. Hold to speak. Whisper-grade transcription.
- **Intervention:** A global "PAUSE" button. Freezes the agent execution loop immediately. Allows the user to inspect state, edit the next prompt, or cancel the action.

## Feature Roadmap

### Phase 1: MVP (The "Glass Box")
- [ ] **Core Chat:** Markdown rendering, code highlighting.
- [ ] **Streaming:** Real-time token streaming via WebSocket.
- [ ] **Thought Visualization:** Collapsible `<think>` blocks.
- [ ] **Basic Auth:** Connect to user's OpenClaw instance.
- [ ] **Terminal Widget:** Read-only view of tool execution logs.

### Phase 2: The "Commander" (Control)
- [ ] **Input:** Voice PTT integration.
- [ ] **Files:** Drag-and-drop file upload/injection into context.
- [ ] **Intervention:** Pause/Stop signal handling.
- [ ] **Sidebar:** "Brain View" showing basic token usage and active model.

### Phase 3: The "OS" (Offline & Sync)
- [ ] **Local DB:** SQLite integration for full history search.
- [ ] **Sync:** Replay queue for offline actions.
- [ ] **Widgets:** Interactive tool widgets (e.g., click a button in chat to trigger an agent function).
- [ ] **Multi-Agent:** UI for visualizing multiple sub-agents working in parallel.
