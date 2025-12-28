# News Data Extraction Agent — Structured Intelligence from Unstructured Articles

> Client work. Client identity intentionally not disclosed.

## Project Metadata

**Status:** Delivered (client project)  
**Primary Focus:** LLM-powered structured extraction from news articles  
**Secondary Capabilities:** Data normalization, schema-driven parsing, storage-ready outputs, batch ingestion  
**Domains:** Sports news intelligence (NFL example), document understanding, information extraction  

**Role:** Lead implementer (end-to-end extraction design + pipeline build)  
**Stack:** Python, OpenAI API (structured outputs), Pydantic schemas, requests/BeautifulSoup, pandas, REST APIs

---

## Executive Summary

Built a production-style **news data extraction agent** that converts unstructured news articles into **normalized, structured records** suitable for downstream analytics, search, alerting, and storage. The system ingests articles from a news feed/API, retrieves full article text, cleans and normalizes content, then uses a schema-driven LLM extraction step to produce a **strict JSON object** with metadata + entities + events + stats + quotes + rankings.

The key design goal was reliability: ensuring outputs are consistently structured, validated, and easy to store—rather than producing “best-effort” text summaries.

---

## Motivation / Why This Was Built

News is high-volume and unstructured. The client needed a way to:
- turn article text into **structured data** (entities, events, metrics, rankings, quotes)
- store it consistently across many articles
- enable downstream use cases like dashboards, monitoring, and retrieval

The project prioritized a repeatable extraction process with clear contracts and predictable outputs.

---

## System Overview

At a high level, the pipeline works like this:

1. **Ingest article feed** (batch pull)
2. **Select and fetch full article content**
3. **Clean / normalize text** (HTML → clean plain text)
4. **Schema-driven extraction** (LLM produces validated structured JSON)
5. **Storage-ready outputs** (records ready for DB tables / object storage)
6. **Analytics-friendly flattening** (optional pandas DataFrames for quick inspection)

---

## Core Capabilities

### 1) Article Ingestion (Feed → Item List)

- Pulls a large set of articles from a structured source (sports news API in the example).
- Stores a timestamped snapshot of the feed results to support:
  - auditability
  - replay/backfills
  - incremental processing patterns

---

### 2) Content Retrieval (Item → Full Story)

Articles often include multiple representations (web links vs API endpoints).  
The system fetches the authoritative “full story” content (not just snippets), ensuring the extractor sees the complete text.

---

### 3) Text Cleaning & Normalization

Most news bodies contain HTML markup, formatting artifacts, and inconsistent spacing.

The pipeline normalizes article content into an extraction-ready form:
- strips HTML tags
- preserves useful paragraph boundaries
- removes excessive whitespace/noise
- produces clean plain text suitable for LLM input

This step is critical: extraction accuracy depends heavily on clean input text.

---

### 4) Schema-Driven Structured Extraction

The heart of the system is a strict schema contract enforced with Pydantic models.

Rather than asking the model for “a JSON blob,” the system:
- defines a typed schema for expected fields
- requires the model to return outputs that conform to that schema
- parses/validates the structured result into a canonical object

This approach produces **consistent, storage-ready data** across many articles and topics.

---

### 5) Generalized Information Model (Domain-Portable)

While the early prototype explored an NFL-specific schema, the final design emphasized a generalized representation that scales across many article types.

Key structured groups include:

- **Metadata**
  - topic, title, publication date, author, publisher, URL, summary
- **Entities**
  - people/teams/organizations mentioned
- **Events**
  - normalized “what happened” records (injury, trade, signing, suspension, etc.)
- **Stats**
  - metric-value pairs attached to entities (e.g., performance metrics)
- **Quotes**
  - attributed quotes with entity type labeling
- **Rankings**
  - rank + category (power rankings, fantasy rankings, etc.)
- **Extra signals**
  - sponsorship/branding mentions
  - betting-related references
  - social/cultural mentions

This “event/stat/quote/ranking” abstraction makes the pipeline reusable beyond sports.

---

## Data Storage & Downstream Use

Outputs were designed to be easily persisted in either:
- relational tables (normalized events/stats tables keyed by article)
- document storage (one JSON per article)
- hybrid models (metadata in DB + raw JSON in object storage)

The extraction format supports:
- analytics (metrics tables, aggregation)
- search (entity-based retrieval)
- alerting (new events per entity)
- traceability (source_url + publication_date + summary)

---

## Reliability & Operational Design Notes

Key design choices that increased robustness:

- **Validation-first outputs**
  - extraction is treated as “data production,” not a text generation task
- **Deterministic structure**
  - consistent keys and field names across all articles
- **Separation of concerns**
  - ingestion, cleaning, extraction, and storage are distinct pipeline stages
- **Batch-friendly workflow**
  - supports processing large feeds and replaying historical snapshots

---

## Limitations & Future Extensions

- Entity resolution (linking entities to canonical IDs across sources)
- Deduplication across similar articles (cluster-by-story)
- Incremental processing with “seen” tracking + backfill jobs
- Confidence scoring per extracted field
- Multi-source ingestion (multiple publishers, varying HTML formats)
- Human review UI for edge cases / schema evolution

---

## Key Takeaway

This project demonstrates how to build a **schema-validated extraction system** that converts messy news content into **clean structured datasets** suitable for storage and analytics—using a pipeline approach that emphasizes reliability, normalization, and reuse across domains.
