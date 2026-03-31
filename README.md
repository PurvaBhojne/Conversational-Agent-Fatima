# Conversational-Agent-Fatima
AI-powered conversational agent to automate shipment tracking and extract structured data from driver interactions in logistics operations

- **Product:** Fatima — AI Agent for Driver Communication  
- **Company:** TruKKer Technologies  
- **Role:** Product Manager (end-to-end ownership)  
- **Scope:** 0→1 design, launch, and iteration
## 🧩 The Context

In logistics, real-time shipment status matters for two reasons that are easy to conflate but fundamentally different:

* **Security** — TruKKer owns execution. If a truck goes dark, that's a liability
* **Client trust** — Visibility is a product promise. Delays in updates directly impact brand value and retention

Before Fatima, status collection was entirely manual:

* Ops teams called drivers at each milestone (loading, en route, border, delivery)
* Updates depended on when ops called — not when events actually happened
* Driver responses were inconsistent — vague, delayed, or missed
* Ops bandwidth was consumed by follow-ups
* Clients saw stale or delayed data

👉 The assumed fix: *“Make drivers update the app”*
👉 The real fix:
**Drivers won’t change behavior. The system must meet them where they are.**

---

## 🏗️ What Was Built

A **WhatsApp-based conversational AI agent (Fatima)** that:

* Automates shipment status collection from drivers
* Converts unstructured responses into structured operational data
* Updates internal systems in real-time
* Escalates only when the system cannot resolve


### 🔄 End-to-End Flow

Portal Trigger → AI Agent → Driver Interaction → NLP + Logic → Structured Output → System Update

![Execution Pipeline](./assets/fatima_agent_architecture_v2.svg)

---

## 🧠 Key PM Decisions & Why

### 1️. WhatsApp-first, not app-first

Drivers were already using WhatsApp. Forcing app adoption mid-route would fail.
👉 Decision: **Meet users where they already are**

---

### 2. Milestone-driven task generation

No random pings. No spam.

* Trigger conversations based on actual shipment milestones
* Ensure interactions are purposeful

👉 Result: Lower friction, higher response quality

---

### 3️. Intent mapping before prompt design

Defined core intents upfront:

* Location update
* Loading confirmation
* Delay notification
* Border crossing
* Delivery completion
* Unclear / unresolvable

👉 This became the foundation for:

* Conversation flows
* Fallback logic
* Data extraction

---
### 4. Designing for ambiguity (not just happy paths)

Handled ambiguity inherent to logistics operations 

— including missing milestones, trip cancellations, and vague or delayed inputs 

— by designing structured follow-up and fallback logic.

---

### 5. Unstructured → Structured as a product layer

Not left to engineering.

Designed extraction logic for:

* Status
* Location
* Timestamp
* Delay reason

👉 This enabled downstream systems (finance, ops, dashboards) to actually use the data

---

## ⚙️ How It Was Built 

| Area                | What was owned                                   |
| ------------------- | ------------------------------------------------ |
| Conversation Design | End-to-end flows, decision trees, fallback logic |
| Prompt Engineering  | Contextual prompts, response interpretation      |
| Intent Mapping      | Defined classification of all driver responses   |
| Data Extraction     | Designed structured output logic                 |
| Orchestration       | Portal → AI → Output → System update             |
| API Integration     | Worked with engineering on APIs/webhooks         |
| Iteration           | Analyzed transcripts, improved flows             |

---

## 📈 Results

* 🤖 ~15% of trip updates handled autonomously (Phase 1)
* ⏱️ Reduced manual ops follow-ups
* 📡 Improved consistency of shipment updates
* 🧩 Replaced free-text notes with structured data

---

## 🚀 What This Built Toward

Fatima was not just a status bot.

It proved that:

> **Unstructured driver communication can be converted into reliable operational signals at scale**

This unlocked:

* Payment eligibility systems
* SLA tracking
* Client-facing real-time visibility

👉 The 15% automation rate was a **starting point, not the limit**

---
