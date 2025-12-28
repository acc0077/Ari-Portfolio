# GeoMind — Multi-Agent Geospatial Analysis + Map Visualization System

---

## Project Metadata

**Runtime:** Local Flask app + PostGIS database (DigitalOcean managed Postgres) +  Qdrant (Docker)
**Core Tech:** Flask, Swarm (Agents SDK equivalent), Postgres + PostGIS, Leaflet, leaflet-heat, GeoPandas, Shapely, Folium
**Primary Focus:** Natural-language geospatial workflows (points, amenities, bbox filtering, spatial joins, distance/buffers) with map-driven UX
**Role:** Sole implementer (agent routing, tool layer, DB integration, frontend mapping UI, EDA notebook, RAG)

---

## Executive Summary

GeoMind is a multi-agent geospatial assistant that turns natural-language requests into:

1. **Structured geospatial tool calls** executed against PostGIS
2. **Deterministic map updates** rendered as markers or heatmap overlays

The system enforces a strict separation of concerns:

* Agents decide *what should happen*
* Tools mutate authoritative geospatial state in the database
* The frontend renders only database-derived state

This avoids hallucinated map updates and keeps spatial logic auditable and reproducible.

---

## Demo

- Short walkthrough:  
  https://drive.google.com/file/d/1EJ0QZsp6kb8qcOAnm4N5tEOhs5wEShQZ/view?usp=sharing

---

## Architecture Overview

### High-Level Layout

GeoMind runs as a single Flask web application composed of:

* A chat endpoint that executes the agent orchestration loop
* Read-only API endpoints that expose geospatial layers as JSON
* A Leaflet-based frontend for visualization

Key endpoints:

* `/chat` — runs the active agent and handles routing / tool calls
* `/get-location` — returns point locations to render as markers
* `/get-amenities` — returns amenity points to render as markers or heatmap

### State Model

GeoMind maintains two parallel state channels:

#### 1. Conversation State (Flask Session)

* `session["messages"]`: chat history in `{role, content}` format
* `session["agent_name"]`: currently active agent

#### 2. Map State (Postgres / PostGIS)

* `dynamic_locations` — authoritative point layer for map pins
* `amenities_view` — authoritative filtered amenity layer, rebuilt on demand

The UI never trusts agent text output as map state. All visual updates come from database reads after tool execution.

---

## System Guarantees & Design Invariants

GeoMind intentionally enforces several hard invariants:

* **Tool-driven truth**
  Map layers are rendered *only* from Postgres tables. Agents cannot mutate state without calling tools.

* **Explicit agent routing**
  A top-level triage agent decides which specialist agent handles each request.

* **Explicit visualization mode**
  Marker vs heatmap rendering is controlled via tool output, not implicit UI heuristics.

* **Spatial logic lives in PostGIS**
  Bounding boxes, buffers, joins, and distance calculations are executed in SQL, not in Python or the frontend.

---

## Swarm → OpenAI Agents SDK Mapping

Although implemented using Swarm, GeoMind maps directly to the OpenAI Agents SDK:

* `Agent(name, instructions, functions=[...])` ↔ Agents SDK `Agent(...)`
* `client.run(agent, messages, context_variables)` ↔ `Runner.run(...)`
* `transfer_to_X` functions ↔ router / handoff pattern

The overall design is a textbook multi-agent orchestration system:

* One router (triage) agent
* Multiple specialist agents
* Tool calls as the only mutation mechanism

---

## Core Agents

### 1. Triage Agent

**Purpose:** Route the user request to the correct specialist agent.

**Responsibilities:**

* Inspect the user message
* Handoff to one of:

  * Location Agent
  * Data Filter Agent
  * Proximity Agent (extendable)

---

### 2. Location Agent

**Purpose:** Resolve user requests into geographic point updates.

**Primary Tool:** `update_map_location(locations)`

**Behavior:**

* Accepts one or more `{lat, lng, message}` objects
* Rebuilds the `dynamic_locations` table
* Inserts the new point set

**Example Requests:**

* “Show me Andalusia, AL”
* “Find parks in Washington, DC”
* “Show hookah bars in Chicago”

---

### 3. Data Filter Agent

**Purpose:** Filter and visualize amenity / POI datasets.

**Primary Tool:** `create_amenities_view(event, min_lon, min_lat, max_lon, max_lat)`

**Capabilities:**

* Filter by amenity list
* Optionally constrain results to a bounding box
* Control visualization mode (markers vs heatmap)

**Instruction Constraint:**

* `amenity_list` must always be a list, even for a single amenity

---

### 4. Proximity Agent (Extendable)

**Purpose:** Proximity and spatial-join workflows.

**Tools:**

* `create_table_view(event)` — creates filtered SQL views
* `create_spatial_join_view(event)` — buffers geometries and joins intersecting features

Designed for queries like:

* “Within X miles of this point”
* “What amenities are near these parcels”
* Nearest-neighbor and distance-based analysis

---

## Tooling Layer

### update_map_location

* Recreates `dynamic_locations`
* Inserts new point records
* Returns a success message with count of inserted locations

### create_amenities_view

* Recreates `amenities_view` from `amenities`
* Supports:

  * Amenity list filtering
  * Bounding-box filtering via `ST_MakeEnvelope`
* Returns either a success JSON payload or a heatmap flag

### create_table_view

* Creates or replaces SQL views
* Supports column selection and WHERE clauses

### create_spatial_join_view

* Buffers source geometries (miles → meters)
* Joins target geometries using `ST_Intersects`
* Uses CRS transforms (EPSG:4326 ↔ EPSG:3857) for correct distance math

---

## Flask Backend

### `/`

* Clears session state
* Initializes conversation
* Serves the main UI

### `/chat` (POST)

**Purpose:** Execute the multi-agent orchestration loop.

**Behavior:**

* Append user message to session history
* Run the active agent
* Persist updated history and agent state
* Detect most recent tool call
* Return assistant text plus tool metadata

### `/get-location` (GET)

Returns JSON rows from `dynamic_locations` for marker rendering.

### `/get-amenities` (GET)

Returns JSON rows from `amenities_view` for marker or heatmap rendering.

---

## Frontend UI (Leaflet)

### Layout

Split-screen interface:

* **Chat (left):** conversational control plane
* **Map (right):** Leaflet map with Carto basemap

### Chat → Map Triggering

After `/chat` completes:

* If the last tool was `update_map_location` → refresh markers
* If the last tool was `create_amenities_view`:

  * Heatmap flag true → render heatmap
  * Heatmap flag false → render markers

---

## Data Sources & Research Notebook

GeoMind is supported by extensive notebook-based research and EDA.

### OpenStreetMap (Overpass API)

* Bulk amenity ingestion via bounding boxes
* Amenity, name, latitude, longitude extraction

### Geocoding (OSM Nominatim)

* Address to coordinate resolution
* Used for centering, buffers, and proximity queries

### PostGIS on DigitalOcean

* PostGIS-enabled Postgres
* Geometry storage in EPSG:4326
* Distance calculations via EPSG:3857 transforms

### US Census ACS (5-Year)

* Demographic, income, housing, and employment data
* Variable metadata exploration
* Tract-level joins and choropleth analysis

### TIGERweb Geometries

* State, county, tract, and block group polygons
* CRS normalization for mapping

### Fulton County Parcel Data

* Parcel-level GIS data
* Loaded via GeoJSON for property analysis workflows

---

## RAG  — Census Variable Discovery

GeoMind includes an early RAG-style system for discovering relevant Census variables.

### Vector Database

* Qdrant, run via Docker

### Embeddings

* OpenAI `text-embedding-3-small`
* Variables embedded as natural-language summaries

### Summary Strategy

* Parse hierarchical ACS labels
* Convert to fluent descriptive text
* Append variable concept
* Embed for semantic retrieval

### Query Flow

1. User query → embedding
2. Vector search over Census variable summaries
3. Extract variable IDs
4. Fetch data from Census API
5. Join to geometries for analysis and mapping

---

## Example User Flows

### Location Display

User: “Show me the BeltLine area”

* Triage → Location Agent
* `update_map_location`
* UI refreshes markers

### Amenity Density

User: “Show schools and parks in Atlanta as a heatmap”

* Triage → Data Filter Agent
* `create_amenities_view` with heatmap flag
* UI renders heat layer

### Proximity Analysis

User: “Find amenities within 0.5 miles of these parcels”

* Triage → Proximity Agent
* Buffered spatial join view created
* Results become queryable and visualizable

---

## Key Takeaway

GeoMind demonstrates an end-to-end pattern for LLM-driven geospatial analysis:

* Agents decide intent
* Tools mutate authoritative spatial state
* The frontend renders deterministic map layers
* RAG enables semantic discovery of complex datasets

This forms a strong foundation for a full geospatial copilot capable of answering:

* What’s near here?
* What’s within X minutes?
* How does this neighborhood compare?
* Which areas match my criteria?
* What data should I even use to
