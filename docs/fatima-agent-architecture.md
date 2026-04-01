# Fatima — WhatsApp AI Agent Architecture
### Conversational AI for Real-Time Logistics Status Collection

> This documents the prompt design, flow logic, and orchestration architecture behind **Fatima** — a production WhatsApp AI agent built at TruKKer Technologies to automate shipment milestone capture from drivers across MENA and Europe.

---

## What This Is

Fatima is a stateful conversational AI agent that:
- Initiates outbound WhatsApp conversations with drivers at each shipment milestone
- Captures unstructured driver responses (text, location, documents)
- Validates and sequences milestone timestamps in UTC
- Converts all inputs into structured operational data
- Routes structured data back to TruKKer's logistics platform via webhooks
- Escalates to ops teams only when it genuinely cannot resolve

This is the **actual prompt architecture and flow logic** used in production (v99).

---

## Agent Flow Overview

```
Webhook Trigger (from logistics platform)
        │
        ▼
┌─────────────────────┐
│   Incoming Hook     │  ← Trip data injected: moveType, sourceCountry,
│   (Session Start)   │    destinationCountry, driverName, truckNumber,
└─────────┬───────────┘    orderNumber, previousStatus, ETA, etc.
          │
          ▼
┌─────────────────────┐
│  Type Router        │  ← Routes by <type> parameter:
│  (Condition Node)   │    ReminderFlow | OTR | LoadingConfirmation |
└─────────┬───────────┘    GenericConnect | DocReminder
          │
          ▼
┌─────────────────────┐
│  WhatsApp Outbound  │  ← Sends templated message to driver
│  (Send Message)     │    via WhatsApp API
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Conversational AI  │  ← LLM (GPT-4.1) processes driver reply
│  Agent (Fatima)     │    Selects Path based on intent + moveType
└─────────┬───────────┘
          │
    ┌─────┴──────┐
    ▼            ▼
[Tools]      [Actions]
wait_tool    GoogleMaps API     ← Coordinate extraction from text/location
wait_skip    Artifact Resolve   ← Document URL processing
LC_tool      POST to TruKKer    ← Structured milestone data → logistics platform
OTR_tool     GET Sessions API   ← Session + message history retrieval
GC_tool      GET Messages API
DOC_tool
          │
          ▼
    end_session  ← HARD TERMINAL — no further processing
```

---

## Core Design: Mini-Flows (Reusable Logic Units)

Instead of redefining logic in every path, the agent uses **three reusable mini-flows** referenced throughout all paths. This was a deliberate design decision to keep prompt logic DRY and maintainable at scale.

### MF-1: Time Capture, Resolution & Validation

The most complex mini-flow. Handles the full lifecycle of a single milestone timestamp.

```
INPUT FORMATS ACCEPTED:
  "7 PM"              → Time only
  "21 Dec 7 PM"       → Date + time  
  "Today", "Yesterday"→ Keyword
  "2 hours ago"       → Relative past
  "in 2 hours"        → Relative future

RESOLUTION ANCHOR:
  ALL time resolution (absolute or relative) uses:
  <datetime_driver_today> — driver-local UTC timestamp
  No other "now" reference permitted.

VALIDATION RULES:
  Completed milestones  → resolved UTC MUST be < <today>
  ETA / in-progress     → resolved UTC MUST be > <today>
  Waiting milestones    → relative past times valid (waiting start time)

SEQUENTIAL VALIDATION (bidirectional):
  current_milestone_utc ≥ previous_milestone_utc
  AND
  current_milestone_utc ≤ next_already_captured_milestone_utc

  On conflict → pause flow, identify earliest conflict, re-ask only that milestone

TOOL-CALL GUARD (non-negotiable):
  Do NOT call any tool until:
  ✓ Time resolved
  ✓ Converted to ISO 8601 UTC
  ✓ Past/future validation passes
  ✓ Sequential order valid
```

**Why this matters:** Drivers report times in fragmented, inconsistent formats — "loaded this morning", "2 hrs back", "11 AM" with no date context. MF-1 normalizes everything to UTC before any data hits the backend, eliminating the ops team's biggest source of bad data.

---

### MF-2: Document Capture (Non-blocking)

```
APPLIES TO: Loading proof, customs clearance docs, POD/unloading proof

EXECUTION:
  1. Send document upload request to driver
  2. IMMEDIATELY call wait_tool (180 seconds) — do NOT wait for reply first
  3. If document received → store docPaths[]
  4. If not received → continue flow (non-blocking)
  5. If unrelated reply → re-prompt once

RULE: BOT must always ASK for document even if user may skip.
      "Non-blocking" ≠ "skip asking"
```

---

### MF-3: Location Capture (Non-blocking)

```
APPLIES TO: Post-loading, waiting states, border arrival/departure, unloading

EXECUTION:
  1. Send location request: "Please share your current location"
  2. IMMEDIATELY call wait_tool (120 seconds) — do NOT wait for reply
  3. Store: GPS coordinates OR text location
  4. City/country derivation handled server-side via GoogleMaps API

RULE: Always ask even if driver previously shared approximate location text.
```

---

## Path Selection Logic

The agent selects a path based on two inputs: **moveType** and **driver's reported status**.

```
moveType: "Domestic" | "Cross Border"

DOMESTIC MILESTONE LADDER:
  At Loading → Loading Completed → Unloading Completed

CROSS BORDER MILESTONE LADDER (GCC):
  At Loading → Loading Completed → At Origin Border 
  → At Destination Border → Unloading Completed

CROSS BORDER MILESTONE LADDER (Europe Export):
  At Loading → Loading Completed → At Origin Border
  → At Destination Border → Unloading Completed

CROSS BORDER MILESTONE LADDER (Europe Import):
  At Loading → Loading Completed → At Origin Border
  → At Destination Border → Completed
```

**Example path selection:**

```
Driver says: "Sila"
moveType:    "Cross Border"
sourceCountry: "Saudi Arabia" → destinationCountry: "UAE"

Step 1: Resolve "Sila" against Cross Border Knowledge Base
Step 2: "Sila" belongs to destinationCountry → classify as Destination Border
Step 3: Select Path 7 (At Destination Border)
Step 4: Backfill all prior mandatory milestones before calling tool
```

---

## Backfill Logic

One of the harder design problems: drivers often report a later milestone without having reported earlier ones.

```
RULE: If reported milestone is AHEAD of currentStatus →
      backfill missing milestones sequentially

EXECUTION:
  For each missing milestone:
  1. Ask required questions in order
  2. Validate each using MF-1
  3. Ensure actualDateTime(n) ≥ actualDateTime(n-1) before proceeding
  4. Only call tool after ALL milestones validated or explicitly skipped
```

---

## Escalation Matrix

```
FAILURE TYPE                    ACTION
─────────────────────────────────────────────────────────────
A. Time validation failure      Re-ask only conflicting milestone
B. Invalid time inputs (3x)     trigger tool(isEmergency=true)
C. Unrecognized menu reply(3x)  Re-prompt → escalate on 3rd failure
D. No response                  tool(status=Failed, isEmergency=true) → end_session
E. Wrong number/driver          tool(status=Failed, isEmergency=true) → end_session
F. Critical event (breakdown,   tool(isEmergency=true, errorMessage=reason) → end_session
   accident, document seizure)
```

Retry counters are tracked **per step**, not across the full path. A successful validation resets the counter for the next step.

---

## Structured Output: What Goes to the Backend

After all milestone data is captured and validated, the agent calls the appropriate tool with this payload structure:

```json
{
  "currentStatus": "At Destination Border",
  "actualDateTime": "2025-10-27T13:00:00Z",
  "isEmergency": false,
  "errorMessage": null,
  "status": "Success",
  "docPaths": [
    "https://artifacts.../waybill.pdf",
    "https://artifacts.../customs_clearance.jpg"
  ],
  "location": {
    "lat": 24.0889,
    "lng": 56.2610,
    "text": "Sila"
  }
}
```

Tools available per flow type:

| Flow Type | Tool Called | Backend Endpoint |
|---|---|---|
| On-Trip Report | `OTR_tool` | `/CaptureBotResponseForOTR` |
| Loading Confirmation | `LC_tool` | `/CaptureBotResponseForLoadingConfirmation` |
| Generic Connect | `GC_tool` | `/CaptureBotResponseForGenericConnect` |
| Document Reminder | `DOC_tool` | `/CaptureBotResponseForDocReminder` |

---

## Key Design Decisions (PM Rationale)

**1. WhatsApp-first, not app-first**
Drivers in MENA/Europe are not going to open a new app mid-route. WhatsApp has near-universal adoption. Meeting drivers where they are was the first and most consequential product decision.

**2. Intent taxonomy before prompt design**
Before writing a single prompt line, all driver response types were classified: location update, loading confirmation, delay notification, border crossing, escalation, unclear. This taxonomy drives path selection and fallback logic.

**3. Mini-flows over path-level logic**
Defining MF-1/MF-2/MF-3 as reusable units kept the prompt maintainable across 19+ paths. Without this, each path update would require changes in 19 places.

**4. Strict tool-call guard**
The agent is explicitly prohibited from calling any backend tool until time validation, UTC conversion, and sequential order checks all pass. This prevents dirty data from reaching the logistics platform.

**5. end_session as hard terminal state**
Once `end_session` is triggered, the agent cannot ask questions, call tools, or process further input. This prevents accidental double-writes to the backend on session edge cases.

**6. Language-adaptive responses**
The agent responds in the driver's language — Arabic, English, Hindi, Hinglish — based on the driver's input language. This was critical for adoption in a multi-region, multi-language driver base.

---

## System Parameters Injected at Runtime

```
<moveType>           Domestic | Cross Border
<sourceCountry>      Trip origin country
<destinationCountry> Trip destination country
<operationType>      Import | Export (Europe only)
<driverName>         Driver's name for personalization
<truckNumber>        Truck plate number
<orderNumber>        TruKKer order ID
<previousStatus>     Last known milestone (reference only, never surfaced to driver)
<eta>                Expected arrival time at loading
<today>              Current UTC timestamp
<datetime_driver_today> Driver-local UTC timestamp (resolution anchor)
<agent_name>         Fatima
```

---

## What I Learned Building This

**Transcript analysis > metrics for AI products in early phases.**
Accuracy rate tells you something is wrong. Transcripts tell you *what* and *why*. Reading 50+ conversation transcripts post-launch identified more improvement opportunities than any dashboard.

**The hardest flows are the ones drivers don't follow.**
The happy path worked in v1. The real iteration was in ambiguous inputs: "almost there", "done", "problem at border." These required fallback design, not just path design.

**Prompt versioning matters.**
This is version 99. Each version was a response to a specific failure pattern observed in production. Treating prompts like code — versioned, tested, iterable — is what made the system get reliably better over time.

---

*Built at TruKKer Technologies | Production deployment across MENA & Europe*
*Platform: HappyRobot AI | LLM: GPT-4.1 | Channel: WhatsApp*
