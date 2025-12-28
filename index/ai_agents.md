# AI / LLM Agent Projects

This index highlights projects focused on **AI systems and LLM-driven agents**.

These projects typically share the following characteristics:
- structured state representations
- deterministic tools feeding into LLM reasoning
- explainable or interpretable outputs
- human-in-the-loop or decision-support workflows

The full, canonical documentation for each project lives in `projects/`.

---

## Projects

### NarratorML — Multi-Agent Story Design System
- Production-deployed, multi-agent system for interactive story creation
- Structured agent workflow spanning:
  - macro story design (style, world, characters)
  - outline planning (arcs + beats)
  - beat-driven scene generation
  - post-processing, QA, and continuity tracking
- Uses deterministic tools + structured state (`StoryContext`) feeding LLM reasoning
- Real-time SSE streaming to a web UI with artifact export (PDF)
- Deployed as two Railway services (Flask UI + FastAPI agent backend)
- Built for a real client (identity intentionally undisclosed)

→ See full write-up: `projects/narrator_ml.md`

---

### GeoMind — Multi-Agent Geospatial Analysis + Map Visualization System
- Multi-agent geospatial copilot for natural-language spatial workflows + map-driven UX
- Strong “tool-driven truth” design:
  - agents decide intent
  - PostGIS tools mutate authoritative spatial state
  - frontend renders only database-derived layers (no hallucinated map updates)
- Supports location pinning, amenity filtering, bbox constraints, heatmap vs marker rendering
- Includes an early RAG subsystem for Census variable discovery (Qdrant + embeddings)

→ See full write-up: `projects/geomind.md`

---

### Voice Travel Agent — Real-Time Speech-to-Speech Conversational Assistant
- Real-time voice agent demonstrating a true low-latency speech loop:
  - Microphone → streaming STT → streaming LLM → chunked TTS → speaker
- Streaming-first UX:
  - sentence-aware chunking enables the agent to start speaking before the full response completes
- Supports interruption / barge-in:
  - user speech triggers stop flags and cleanly halts playback
  - system resumes listening immediately
- Demonstrates concurrent audio systems engineering (threaded playback, locks, queues)

→ See full write-up: `projects/voice_travel_agent.md`

---

### GTO Poker Agent (LLM Coach + Hand Evaluator)
- LLM-based decision-support agent for No-Limit Texas Hold’em
- Combines deterministic hand evaluation with LLM reasoning
- Interactive web app with structured game-state inputs and explanations

→ See full write-up: `projects/gto_poker_agent.md`

---

### Recipe Agent (Personal Recipe Builder + Saved Cookbook)
- LLM-powered recipe generator with saved personal cookbook
- User accounts, login, and a Postgres-backed recipe library
- Categories + UI designed for cooking and grocery checklist workflows
- Deployed on DigitalOcean

→ See full write-up: `projects/recipe_agent.md`

---

### News Data Extraction Agent — Structured Intelligence from Unstructured Articles
- Client-delivered extraction system that converts unstructured news articles into **validated, storage-ready structured records**
- End-to-end pipeline:
  - ingest article feeds in batch
  - retrieve full story content
  - clean/normalize HTML → extraction-ready text
  - schema-driven LLM extraction into strict JSON (metadata + entities + events + stats + quotes + rankings)
- Designed for reliability and downstream usability (analytics, search, monitoring, alerting)
- Emphasizes deterministic structure via Pydantic schema contracts and validation-first outputs
- Domain-portable information model (event/stat/quote/ranking abstraction)

→ See full write-up: `projects/news_data_extraction_agent.md`
