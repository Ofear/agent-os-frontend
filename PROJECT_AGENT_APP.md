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

## Tech Stack Recommendation

### Frontend: Flutter (Dart)
*Why?*
- **Rendering Performance:** Skia/Impeller engine guarantees 60/120fps "native" feel, critical for dense UI (streaming tokens + rendering terminal widgets simultaneously).
- **Multi-Platform:** True single codebase for iOS, Android, macOS, Windows, and Linux.
- **Fidelity:** Agents output markdown, code, and raw data. Flutter's control over every pixel allows for custom "Terminal Widgets" and "Code Editors" that perform better than React Native's bridge implementations.

### Backend / Protocol: gRPC + WebSockets (with CRDTs)
*Why?*
- **Streaming:** gRPC is built for streaming structured data (e.g., token-by-token generation + metadata).
- **Sync:** We need offline support. A standard REST API is insufficient.
- **Protocol:** Use a custom protocol or extend an existing standard (like Matrix, but likely too heavy). A lightweight sync engine using **CRDTs (Conflict-free Replicated Data Types)** ensures that if the user edits a prompt offline while the agent is replying, the state merges correctly when reconnected.

### Database: SQLite (Local) + PGLite/Postgres (Server)
*Why?*
- **Local-First:** The app must work offline. Agents are tools; tools must be available instantly.
- **SQLite:** The industry standard for local edge storage. Fast, reliable.
- **Vector Extensions:** Use `sqlite-vec` locally for fast semantic search over recent agent history without hitting the server.

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
