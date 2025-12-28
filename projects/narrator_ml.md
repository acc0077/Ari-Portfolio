# NarratorML — Multi-Agent Story Design System

> Client work. Client identity intentionally not disclosed.

## Project Metadata

**Status:** Production deployment (client delivery)  
**Deployment:** Railway (two services: Flask UI + FastAPI Agent API)  
**Core Tech:** OpenAI Agents SDK, FastAPI, Flask, SSE streaming, Clerk auth, Supabase client (reserved/optional persistence)  
**Primary Focus:** Advanced agent orchestration for story design (macro → blueprint → scenes → QA → export)  
**Role:** Lead implementer for agent system + UI integration + production deployment

---

## Executive Summary

NarratorML is a production-deployed, multi-agent system for interactive story design. It supports iterative creation of a story’s **macro structure** (style/world/characters), **outline** (arcs/beats), and **beat-driven scene generation**, with real-time streaming to a clean web UI and exportable PDF artifacts.

The system runs as **two separate Railway services**:

1) **Flask UI Service**
   - Serves web UI (login + main app)
   - Auth via Clerk (JWT validation)
   - Proxies to the FastAPI backend (API key protected)
   - Provides PDF export (`/download_pdf`)

2) **FastAPI Agent API Service**
   - Runs agent orchestration using the OpenAI Agents SDK
   - Streams Server-Sent Events (SSE) for real-time UX
   - Endpoints: `/agent`, `/scene`, `/cancel`
   - Maintains a structured `StoryContext` across runs

---

## Demo

- Short walkthrough:  
  https://drive.google.com/file/d/1ll_8Mhh5Y4NQ2o_u4Wj360sikX_IdRJ2/view?usp=sharing

The video shows:
- Macro story setup (style / world / characters)
- Blueprint editing (arcs & beats)
- Beat-driven scene generation with live streaming
- QA + continuity tracking
- PDF export

---

## Architecture Overview

### Why two services?
- Keeps the UI secure and responsive (auth + rendering)
- Isolates long-running streaming agent workloads in the API service
- Simplifies service-to-service access control (API key boundary)

---

## System Guarantees & Design Invariants

NarratorML enforces several hard invariants to keep agent behavior safe, auditable, and predictable:

- **No silent state mutation**  
  Story state (`StoryContext`) is only mutated via explicit tool calls.
  Agents are forbidden from claiming changes without invoking tools.

- **Read-only vs write-only separation**
  - Orchestration agents may mutate state via tools.
  - Scene generation agents are strictly read-only and prose-only.

- **Deterministic continuity tracking**
  Facts, hooks, and motifs carry explicit lifecycle state
  (`open`, `closed`, `contradicted`, `dropped`) and salience scores.

- **Graceful interruption**
  All streaming operations support cancellation and still emit a final
  structured snapshot to avoid corrupted sessions.

These invariants were enforced intentionally for client trust and long-running creative sessions.


---

## OpenAI Agents SDK Context Pattern (Why `StoryContext` Matters)

NarratorML uses the Agents SDK **context** pattern: a developer-defined object passed into `Runner.run()` / `Runner.run_streamed()` and made available to agents and tools throughout the run (including `RunContextWrapper` inside tool calls). This enables a shared, structured “working memory” beyond chat history.  
See Agents SDK docs: Context is passed to agents/tools/handoffs as dependency injection + state. (https://openai.github.io/openai-agents-python/agents/) 

---

## Core Data Model: `StoryContext`

`StoryContext` is the canonical, structured state container for the story:

- **StyleFingerprint**
  - tone, pacing, sentence structure, dialogue style, vocabulary level, tags, etc.
- **World**
  - name, description, locations, rules
- **Characters**
  - character roster with ID + description/traits/backstory/role
- **Blueprint**
  - arcs + beats (both sequence-numbered)

`StoryContext` supports `extra = "allow"` to attach temporary fields without breaking schema.

---

## Streaming Protocol (SSE)

The FastAPI service streams responses as **Server-Sent Events**:

- `event: <type>`
- `data: <payload>`
- blank line delimiter

Observed event types:
- `info` — agent switches, tool calls, status updates
- `message` — incremental assistant text (character streaming)
- `scenemessage` — incremental streaming for scene prose
- `final_output` — final JSON snapshot (context + items + scenes)

Every successful request ends with a `final_output` event containing the structured response payload.

---

## FastAPI Agent API

### Authentication (UI → API)
All API endpoints require:
- header: `x-api-key: <NARRATOR_API_KEY>`
- mismatched key returns `401`

### Cancellation model
- `/cancel` adds `stream_id` to an in-memory `CANCELLED` set
- active streams check cancellation and exit cleanly, still returning a final snapshot

---

## Endpoint: `/agent` — Story Orchestration

**Purpose:** Macro + blueprint orchestration (NOT scene writing).  
**Streaming:** SSE ending with a `FinalEvent`.

**Inputs**
- `user_input`
- `input_items` (conversation history as `{role, content}`)
- `story_context` (nullable)
- optional `stream_id`

**Behavior**
- Appends user message to `input_items`
- Runs `story_orchestrator_agent(context)` with streaming
- Streams:
  - agent switches (`🧠 Now using agent: ...`)
  - tool calls (`🔧 Calling tool ...`, `✅ Tool completed ...`)
  - text deltas (`message`)
- Compacts history at the end (see Memory section)
- Returns final JSON snapshot via `final_output`:
  - updated `input_items`
  - updated `story_context`

---

## Endpoint: `/scene` — Scene Orchestration + Generation + QA

**Purpose:** Beat-driven scene generation and/or scene chat, with continuity management.  
**Streaming:** SSE ending with a `SceneFinalEvent`.

**Inputs**
- `user_input`
- `input_items`
- `story_context` (nullable)
- `scenes` (carry-forward list of `SceneOutput`)
- optional `stream_id`

**High-level flow**
1) Classify user message: `scene` vs `chat`
2) If `scene`:
   - determine target beats (Beat Identification agent)
   - for each beat:
     - build beat prompt (including previous scene handoff)
     - build continuity registry (facts/hooks/motifs)
     - stream scene prose (`scenemessage`)
     - post-process into `SceneAnalysis` (summary + facts/hooks/motifs + handoff)
     - run Style QA (`QAStyleReport`)
     - run Narrative QA + continuity updates (`NarrativeQAReport`)
     - apply continuity updates back into the registry objects
     - append structured `SceneOutput` to `scenes[]`
3) If `chat`:
   - stream conversational response (`message`)

Final payload includes:
- compacted `input_items`
- updated `story_context`
- full accumulated `scenes[]`


## Scene Orchestration Routing (Classification → Scene Pipeline vs Chat)

The `/scene` endpoint is intentionally multi-modal: it can either (a) run the full beat-driven scene generation pipeline, or (b) respond conversationally about the story and previously generated scenes.

To decide which mode to run, NarratorML uses a small **Scene Orchestration Classification Agent**:

### `scene_orchestration_classification_agent`
- **Input:** latest user message (and conversation history)
- **Output:** `MessageClassificationResponse` with:
  - `message_classification: "scene" | "other"`

**Classification contract**
- `scene` → user is requesting prose generation or rewrite (e.g., “write a scene”, “redo the last scene”, “add more description”)
- `other` → user is asking questions, giving feedback, or discussing macro structure (world/characters/style/blueprint)

This classification is the first gate in `/scene`:

- If `scene` → run beat identification → per-beat scene writing → post-processing → QA → continuity updates → persist in `scenes[]`
- If `other` → run a dedicated chat agent and stream normal `message` deltas

### `scene_orchestration_chat_agent`
When classification is `other`, the system uses a “chat-only” agent that:
- answers **general** questions about the story helpfully
- if the user asks about **macro structure** (world, characters, writing style, blueprint), it instructs them to switch to the **Macro tab**
- receives a **read-only StoryContext snapshot** (style/world/characters/blueprint) for reference

This separation prevents accidental scene generation when the user is merely discussing structure or giving notes, while keeping the UX fluid inside a single `/scene` streaming endpoint.


---

## Agents & Tools

### Story Orchestrator Agent
Top-level agent that routes between:
- `edit_macro_structure` tool: world / characters / style
- `edit_story_blueprint` tool: arcs/beats CRUD

**Invariant (“Golden Rule”)**
> The story never changes unless a tool is called.

### Macro Story Structure Tools
Tooling to mutate:
- `StyleFingerprint`
- `World`
- `Characters` (create/update/delete by `character_id`)

### Blueprint Tooling
Single `update_blueprint(...)` tool supports:
- upsert arcs + beats
- delete arcs and beats
- move beats across arcs (remove from other arcs)
- renumber arcs/beats globally to avoid collisions

### Scene Writing Agent
Pure prose generator:
- read-only context snapshot included in instructions
- `tool_choice="none"`

### Post-processing + QA
- Scene analysis: summary, handoff, facts/hooks/motifs
- Style QA: score against `StyleFingerprint`
- Narrative QA: continuity checks + status updates to registry items

---

## Conversation Memory Compaction

To avoid runaway prompt growth:
- keep a tail window with at least **6 user turns**
- if history exceeds a total window, summarize the overflow using a dedicated `convo_memory_agent`
- store summary back as structured user+assistant messages

This keeps long creative sessions stable.

---

## UI Service (Flask) — Summary

The Flask service:
- verifies Clerk JWT (header or `__session` cookie)
- sets a host-only `__session` cookie after auth
- proxies streaming SSE requests to the API:
  - `/agent_proxy` → FastAPI `/agent`
  - `/scene_proxy` → FastAPI `/scene`
- proxies `/cancel` → FastAPI `/cancel`
- generates PDF exports from scenes via `/download_pdf`


---
### UI Streaming Model

The UI maintains a client-side session state consisting of:
- `input_items` (conversation history)
- `story_context` (structured story state)
- `scenes[]` (accumulated scene outputs)

On every streamed request:
1. The UI sends the current state snapshot to the API
2. Incrementally renders SSE events:
   - `info` → status updates / agent transitions
   - `message` → conversational text
   - `scenemessage` → scene prose
3. Replaces local state with the `final_output` payload

This allows full session recovery from a single terminal event and
prevents desynchronization between UI and agent state.


---


## Deployment

**Railway Services**
- Flask UI service (auth + UI + proxies + PDF)
- FastAPI API service (agents + SSE streaming + cancellation)

**Environment variables (names only)**
- Flask: `NARRATOR_API_URL`, `NARRATOR_API_KEY`, `CLERK_JWKS_URL`
- FastAPI: `NARRATOR_API_KEY`, `SUPABASE_URL`, `SERVICE_ROLE_KEY`

---

