# Portfolio Projects

This repository is a **single source of truth** for my portfolio projects from the past several years.

It contains **long-form, detailed project write-ups** that document:
- the problems I worked on
- the systems I designed and built
- the technical and product decisions I made
- the outcomes, learnings, and tradeoffs

These documents are intentionally **not short summaries** or marketing blurbs.
They are designed to be:
- copied into a website or portfolio
- adapted into Upwork proposals or case studies
- converted into PDFs or slide decks
- used as context for technical discussions or interviews

---

## ⭐ Flagship Project

If you are viewing this repository for the first time, start here.

### NarratorML — Multi-Agent Story Design System
A production-deployed, multi-agent system for interactive story design, built for a real client.

**Why this is a flagship project:**
- Demonstrates **advanced multi-agent orchestration**
- Uses **structured, deterministic state** (`StoryContext`) instead of ad-hoc chat memory
- Enforces strong **system invariants** (tool-driven mutation, read-only vs write-only agents)
- Implements **real-time streaming** with cancellation and recovery guarantees
- Ships as a **production system**, not a prototype

**Highlights:**
- Macro → blueprint → scene → QA → export pipeline
- Server-Sent Events (SSE) streaming to a live web UI
- Clean separation between UI service and agent API service
- Explicit cancellation, continuity tracking, and memory compaction
- PDF artifact export

🎥 **Demo video (short walkthrough):**  
https://drive.google.com/file/d/1ll_8Mhh5Y4NQ2o_u4Wj360sikX_IdRJ2/view?usp=sharing

→ Full technical write-up: `projects/narrator_ml.md`

> Additional flagship projects will be added here over time.

---

## Repository Structure

- `projects/`  
  Canonical, long-form project write-ups (one file per project).  
  These are the authoritative records for each portfolio project.

- `assets/`  
  Project-specific supporting materials such as:
  - application screenshots
  - diagrams or visuals
  - UI states or workflows  

  Assets are organized by project and referenced directly from the corresponding
  markdown files in `projects/`.

- `index/`  
  Curated views of projects by focus, capability, or domain.  
  These files group and highlight projects without duplicating content.

---

## How to Use This Repo

- For a **deep dive into a specific project**, start in `projects/`.
- For a **curated view** (AI/LLM work, data engineering, systems), see `index/`.
- Screenshots and visuals referenced in project write-ups live in `assets/`.
- All content is written to be **modular and reusable**.
- Individual sections can be copied independently without losing context.

---

## Scope and Focus

While these projects span many domains and skill sets (APIs, data pipelines,
automation, modeling, infrastructure), my **primary career focus** is on:

- AI systems
- LLM-driven agents
- data- and tool-orchestrated workflows

Projects are preserved in full, while different views can be curated later
depending on audience or use case.

---

## Status

This repository is **actively evolving**.

- Projects will be added incrementally.
- Sections may expand or be reorganized.
- Index files may change as themes become clearer.
- Assets may be added as projects mature or are better documented.

The goal is clarity and completeness first.  
Polish comes later.
