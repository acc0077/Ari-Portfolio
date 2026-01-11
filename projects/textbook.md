# Textbook Technical Overview (High-Level)

**Audience:** Senior software engineers, ML/LLM engineers, platform engineers, technical product leads  
**Goal:** Provide a clear technical view of Textbook’s architecture, constraints, and production capabilities at a level that’s useful to technical readers.

---

## 1) What Textbook Is

Textbook is a multi-tenant conversational booking platform that lets customers schedule appointments through messaging channels (SMS, WhatsApp, web chat, and other chat surfaces). The platform routes inbound messages to the correct company, runs a business-specific booking agent, and executes real booking operations (availability checks, provider assignment, booking create/edit/cancel) against a database-backed scheduling model.

Textbook also supports employee/owner operations (e.g., listing bookings, finding customers, performing booking edits/cancellations) through a dedicated ops workflow with role-based permissions.

---

## 2) Architectural Philosophy

Textbook is designed around a few core technical principles:

### 2.1 Tool-grounded actions (no “LLM-only” state changes)
All booking mutations are executed by server-side tools that touch the database and downstream systems. The agent layer is constrained to:
- **Never invent IDs**
- **Never state that a booking change occurred** unless the corresponding tool result confirms it
- **Validate feasibility** through tool calls before proposing or confirming outcomes

### 2.2 Vertical-aware behavior without rewriting the system per vertical
Textbook supports multiple business categories (e.g., limo, salons/spas, pet services, healthcare-style appointment businesses, and others). Each vertical is defined by:
- A schema contract for booking intent (verticalized)
- A tool set appropriate to the vertical
- A vertical instruction bundle (workflow and constraints)

New verticals can be added via a lightweight registration pattern rather than re-architecting the platform.

### 2.3 Reliability engineering at the conversational layer
LLM output is treated as a production component that must be constrained and monitored:
- Output guardrails detect integrity violations
- Retry loops attempt corrected responses
- Fail-safe responses support resilience when unexpected issues occur

### 2.4 Role-based operations for owners and staff
Textbook distinguishes between:
- **clients** (end users booking services)
- **staff/owners/providers** (who need operational capabilities such as booking lookup and booking management)

Ops permissions are enforced through:
- tool availability (what calls are permitted by role)
- instruction constraints (how the agent may describe outcomes and next steps)

---

## 3) Core Data Contracts (High-Level)

The platform standardizes key structures used across agents and tools.

### 3.1 Booking Context (per conversation / per user)
A strongly-typed context object is carried across runs and includes:
- tenant identity (company)
- business category (vertical)
- timezone and role (client vs staff/admin)
- user identity and session metadata (including first-message semantics)
- an ephemeral event log of tool-executed booking changes (used by guardrails)

This enables:
- multi-tenant correctness
- time-local interpretation (relative date/time phrases resolved in business timezone)
- routing decisions (e.g., owner/staff messages to ops workflows)
- end-to-end integrity checks that reconcile tool execution with user-facing claims

### 3.2 Booking intent models (verticalized)
Two broad intent families are supported:
- **Dynamic-location bookings** (e.g., limo pickup/dropoff)
- **Static-location bookings** (services performed at a fixed business location)

Both share a base intent shape (identity + date/time intent) plus vertical-specific details.

### 3.3 Booking mutation events
Booking changes (“created”, “updated”, “deleted”) are recorded as runtime events. These are used to ensure the final user-facing message is consistent with the operations that actually executed.

---

## 4) Business Type Registry & Configuration System

Textbook uses a registry pattern to bind each business category to:
- schema contract
- tool set
- instruction set

### 4.1 Registration pattern
A single registration function binds a vertical to its configuration. This approach supports:
- consistent workflows across verticals
- minimal code changes to add a new vertical
- reuse of shared tool factories and instruction templates

### 4.2 Shared templates and vertical “extras”
The instruction system includes:
- a general tool cheat-sheet (create/edit/check availability/list bookings/cancel/store user name)
- vertical-specific extras that describe additional tools or constraints
- workflow constraints that prevent unsafe behavior (e.g., no guessing IDs, confirm before create/edit/delete)

### 4.3 Two major vertical families
**A) Limo vertical**
- Address-dependent workflows (pickup/dropoff)
- Web search capability for address disambiguation
- Specialized steps such as quoting and driver assignment

**B) Static-location verticals (salon/spa/etc.)**
- Location selection + service catalog selection
- Provider/service/location compatibility constraints (“PSL” rules)
- Single-service-per-booking semantics (multi-service requests become multiple sequential bookings)

---

## 5) Tooling Layer (Core Booking Tools) (High-Level)

Textbook’s operational tools are server-side functions wrapped for agent use. The emphasis is on enforced constraints and production semantics.

### 5.1 Static-location tool factory (generic booking tools)
A reusable tool factory generates a standard suite for static-location verticals, including:
- location discovery
- service catalog lookup
- provider lookup and filtering
- availability checking
- provider assignment (rules-aware and load-aware)
- booking create/edit flows

This design avoids re-implementing booking logic per vertical.

### 5.2 Provider–Service–Location validation
A central constraint enforced by tools:
- providers offer specific services
- providers operate at a particular location
- booking feasibility is validated within operating hours and existing bookings
- invalid combinations return actionable alternatives (valid providers/services for the chosen location)

### 5.3 Availability computation & assignment strategy
The tools implement:
- timezone-aware “reject past / too-soon” scheduling rules (including “earliest bookable time” constraints)
- shift-aware eligibility filtering
- overlap checks against existing bookings
- load-aware selection (e.g., least busy eligible provider)
- just-in-time validation before insert/update to reduce race conditions

### 5.4 Booking persistence and confirmation model
Bookings are stored with:
- tenant + user + provider references
- date/start time (and derived end time based on duration)
- structured service details
- a short link ID for a customer-facing confirmation page
- status transitions (e.g., pending/confirmed/canceled)
- payment/checkout integration to finalize confirmation where applicable

---

## 6) Agent Layer (Booking + Routing + Owner Ops)

Textbook uses multiple specialized agents with clear responsibilities.

### 6.1 Booking agent (end-user conversations)
The booking agent:
- runs with per-user thread + strongly-typed context
- uses vertical-configured tools and constraints
- follows workflow rules:
  - validate before proposing
  - confirm before mutations
  - no invented IDs
  - concise, natural chat style (no misleading “wait” language)

The agent is instrumented via hooks to:
- measure runtime latency
- track model usage
- capture tool-triggered booking events for integrity checks

### 6.2 Routing agent (company selection)
When a conversation is not yet scoped to a company:
- a routing agent selects the correct tenant using the message and routing signals (e.g., company codes/links)
- if routing fails, it returns a user-facing clarification
- if routing succeeds, the system bootstraps an introduction turn and proceeds with the booking agent

### 6.3 Owner-ops agent (staff/owner workflows)
A separate ops agent supports:
- bookings lookup across date ranges
- user lookup (discovering user IDs)
- booking management (create/edit/delete) where permitted

Role-based permissions are enforced by:
- selecting a toolset that excludes mutation tools for read-only roles
- embedding role-specific workflow constraints and identity semantics in instructions (admin vs superadmin vs manager)

---

## 7) Output Guardrails & Self-Healing Retries

Textbook includes an integrity guardrail that ensures the user-facing message matches actual tool execution.

### 7.1 Integrity target
Prevent two critical failure modes:
- claiming a booking was created/edited/deleted without the corresponding tool result
- running a booking mutation tool without clearly explaining the outcome to the user

A third target is eliminating misleading filler language that implies waiting.

### 7.2 Guardrail strategy
A lightweight audit model classifies the outgoing message for:
- whether it claims a booking mutation happened
- whether it refers to past events vs newly executed changes
- whether it uses filler language without actionable content

This audit is reconciled against the runtime event log (tool-executed booking events).

### 7.3 Retry mechanism
If a violation is detected:
- a corrective instruction is injected
- the agent is rerun for a corrected response
- after a limited number of retries, guardrails may be relaxed and a safe fallback response used to protect user experience

This improves trust and reduces incorrect confirmations in production conversations.

---

## 8) FastAPI Orchestration & Channel Integrations

Textbook runs as a FastAPI service that orchestrates:
- inbound channel webhooks
- session/context retrieval
- agent execution
- outbound message delivery

### 8.1 Channels supported
Inbound/outbound integration exists for:
- SMS (Twilio)
- WhatsApp (Meta API and/or Twilio entrypoints)
- Web chat trigger endpoint (CORS-enabled sub-app)
- Telegram bot webhook
- LINE bot webhook
- Meta Messenger webhook
- Additional webhook handlers and internal trigger endpoints for operations/testing

### 8.2 Webhook security
The platform validates inbound requests (where supported) using:
- signature verification (e.g., Twilio request validation)
- platform-specific signature headers (e.g., LINE)
- API key protection for internal trigger endpoints (web chat, SMS trigger, WhatsApp trigger, notification endpoints)

### 8.3 Concurrency controls
To avoid overlapping runs and inconsistent state:
- per-user locks are allocated and stored per (channel, user identifier)
- the unified “process-and-reply” entrypoint is guarded by these locks
- only one active agent run can occur per user/channel at a time

### 8.4 Session + thread management
The orchestration layer:
- ensures a user record exists (or creates one)
- routes the message to a company when needed (including code-based routing)
- loads the latest conversation thread and appends the new message
- copies context per run to avoid cross-run mutation
- persists the updated agent thread after completion

Special handling exists for:
- code-initiated sessions (bootstrapped intro message)
- first-message compliance text appended to outbound SMS

---

## 9) Notifications & Operational Messaging

Textbook supports outbound operational notifications tied to booking lifecycle events:
- notifications can be sent to providers and/or company owners
- opt-in and opt-out flows exist per channel type
- delivery targets are resolved from stored channel configuration

The notification system is designed to:
- validate recipient configuration before sending
- track delivery attempts and failures explicitly
- stamp “sent at” timestamps for confirmed/canceled notifications only when delivery succeeds

---

## 10) Observability & Cost Tracking

Textbook includes built-in instrumentation for:
- latency measurement (step timing within the orchestration flow)
- run-level logging and tool call visibility via agent hooks
- model usage accounting (token/cost tracking per run)
- tracking web-search tool usage as a separate cost component

This supports:
- performance tuning
- cost attribution per user/action type
- operational debugging for high-throughput, multi-channel messaging flows

---

## 11) Platform Capabilities Summary

Textbook combines:
1) **Multi-tenant routing + persisted conversational sessions**  
2) **Vertical-aware booking logic via a registry/config system**  
3) **Reusable tool factories for consistent booking operations across verticals**  
4) **Rules-driven scheduling semantics (PSL constraints, single-service bookings, operating hours, overlap avoidance)**  
5) **Multi-agent separation of concerns (routing vs booking vs owner ops)**  
6) **Role-based permissions and scoped ops functionality**  
7) **Integrity guardrails that reconcile user-facing claims with executed tools**  
8) **Production orchestration via FastAPI with multi-channel webhooks, request verification, and concurrency control**  
9) **Notification tooling with opt-in/out and delivery auditing**  
10) **Instrumentation for performance and usage/cost visibility**

---

## 12) Scope Notes

This overview focuses on architecture, reliability, and production behavior. It intentionally avoids publishing low-level implementation details such as exact schema definitions, tool payloads, database identifiers, or full prompt content.
