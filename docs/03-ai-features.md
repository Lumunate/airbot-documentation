# AIRBOT DOCUMENTATION: AI Features & Agent System

## Overview

This document provides comprehensive documentation of Airbot's AI system, including configuration, agents, upsells, and how all components work together.

---

# PART 1: AI CONFIGURATION

> **Source Files:**
> - Workspace AI Settings:
>   - Routes: [`src/routes/workspace/ai-settings/ai-settings-routes.ts`](../src/routes/workspace/ai-settings/ai-settings-routes.ts)
>   - Controller: [`src/controllers/workspace-ai-settings-controller.ts`](../src/controllers/workspace-ai-settings-controller.ts)
>   - Service: [`src/services/workspace-ai-settings-service.ts`](../src/services/workspace-ai-settings-service.ts)
> - Listing AI Settings:
>   - Routes: [`src/routes/ai-settings/ai-settings-routes.ts`](../src/routes/ai-settings/ai-settings-routes.ts)
>   - Controller: [`src/controllers/ai-settings-controller.ts`](../src/controllers/ai-settings-controller.ts)
>   - Service: [`src/services/ai-settings-service.ts`](../src/services/ai-settings-service.ts)
> - Documents:
>   - Service: [`src/services/workspace-document-service.ts`](../src/services/workspace-document-service.ts)
>   - KB Processor: [`src/lib/utils/knowledge-base-processor.ts`](../src/lib/utils/knowledge-base-processor.ts)
>   - AI Client: [`src/lib/clients/ai-client/ai-client.ts`](../src/lib/clients/ai-client/ai-client.ts)

## 1.1 Configuration Levels

### Workspace-Level Settings (Global)

```prisma
model WorkspaceAISettings {
  id            String   @id
  workspaceId   String   @unique
  responseTone  String   @default("neutral")
  conciseness   String   @default("neutral")
  generalNotes  String?  @db.Text
  knowledgeBase Json?
  documents     WorkspaceDocument[]
}
```

### Listing-Level Settings (Override)

```prisma
model AISettings {
  id            String   @id
  listingId     String   @unique
  responseTone  String?  // Optional override
  conciseness   String?  // Optional override
  generalNotes  String?
  knowledgeBase Json?
  documents     Document[]
}
```

## 1.2 Configuration Options

### Response Tone

| Value | Behavior |
|-------|----------|
| `neutral` | Balanced, professional (default) |
| `friendly` | Warm, personalized, approachable |
| `formal` | Structured, business-like |
| `casual` | Relaxed, conversational |

### Conciseness

| Value | Behavior |
|-------|----------|
| `neutral` | Balanced length (default) |
| `concise` | Short, direct responses |
| `verbose` | Detailed, thorough explanations |

### General Notes

Free-text field for custom AI instructions:
- Business rules
- Special policies
- Guest handling preferences
- Property-specific information

## 1.3 Configuration Hierarchy

```
Priority (Highest to Lowest):
1. Listing-specific AISettings (if set)
2. Workspace WorkspaceAISettings
3. Default values ("neutral")
```

**Effective Settings Resolution:**
```typescript
effectiveSettings = {
  responseTone: listing?.responseTone || workspace?.responseTone || 'neutral',
  conciseness: listing?.conciseness || workspace?.conciseness || 'neutral',
  generalNotes: [workspace?.generalNotes, listing?.generalNotes].filter(Boolean).join('\n'),
  knowledgeBase: mergeKnowledgeBases(workspace?.kb, listing?.kb),
  documents: [...workspace?.documents, ...listing?.documents]
}
```

## 1.4 Knowledge Base

### Document Upload

**Supported Formats:** PDF, DOCX

**Upload Endpoints:**
```
POST /workspace/:workspaceId/documents                    (global scope)
POST /workspace/:workspaceId/listings/:listingId/documents (listing scope)
```

**Processing Flow:**
```
1. Upload file to backend
2. Create document record (chunks=0)
3. Send to FastAPI: POST /documents/process-knowledge-base
4. FastAPI chunks document and stores in ChromaDB
5. Update document record with chunk count
```

### Knowledge Base Scopes

| Scope | Applies To | Query Priority |
|-------|------------|----------------|
| `listing` | Single listing only | First (7 results max) |
| `global` | All workspace listings | Second (5 results max) |

### ChromaDB Collections

| Collection | Content |
|------------|---------|
| `listings` | Processed listing data chunks |
| `knowledgebase` | Q&A pairs (CSV/XLSX imports) |
| `additionaldocuments` | Custom uploaded documents |
| `sop` | Standard Operating Procedures |

---

# PART 2: AUTOPILOT vs COPILOT

> **Source Files:**
> - Conversation Model: [`src/models/Conversation.ts`](../src/models/Conversation.ts)
> - Routes: [`src/routes/conversation/conversation-routes.ts`](../src/routes/conversation/conversation-routes.ts)
> - Controller: [`src/controllers/conversation-controller.ts`](../src/controllers/conversation-controller.ts)
> - Service: [`src/services/conversation-service.ts`](../src/services/conversation-service.ts)
> - Repository: [`src/repository/conversation-repository.ts`](../src/repository/conversation-repository.ts)
> - Temp Messages: [`src/services/temp-messages-service.ts`](../src/services/temp-messages-service.ts)

## 2.1 Mode Storage

```javascript
// MongoDB Conversation Model
{
  aiAutoPilot: Boolean,  // default: false
  aiCoPilot: Boolean,    // default: true
  tags: {
    escalated: Boolean   // default: false
  }
}
```

**Default:** CoPilot mode (human-in-the-loop)

## 2.2 Mode Switching

**Endpoint:** `PATCH /conversation/:id/toggle-ai-mode`

```typescript
// Toggle logic
const isAutoPilot = newMode === 'autopilot';
await Conversation.findByIdAndUpdate(conversationId, {
  aiAutoPilot: isAutoPilot,
  aiCoPilot: !isAutoPilot
});
```

## 2.3 AutoPilot Mode

### Behavior
- AI **automatically sends** responses to guests
- No human review needed
- Responses appear directly in PMS conversation
- Host sees sent messages in history
- **Auto-escalates** when unable to answer confidently

### Flow
```
Guest Message → Webhook → AI Query → Generate Response
    ↓
AutoPilot = true?
    ↓ YES
├── AI evaluates confidence level
│   ├── Can answer? → Send response automatically
│   └── Cannot answer / Out of context? → Escalate
│       ├── Set conversation.tags.escalated = true
│       ├── Tag with appropriate category
│       ├── Notify host (Socket.io + notification)
│       └── Skip AI processing for future messages
├── Emit 'aiMessage' event (Socket.io)
├── Send to PMS via client.sendConversationMessage()
└── Add to conversation history
```

### Escalation Triggers (AutoPilot)
- Knowledge base confidence < 30%
- Question requires human judgment
- Request involves policy exceptions
- Out-of-context or unclear query
- Missing critical information

### Use Cases
- High-volume properties
- Standard inquiries (check-in times, amenities)
- After-hours coverage
- FAQ responses

## 2.4 CoPilot Mode

### Behavior
- AI generates **draft responses**
- Saved to TempMessage collection
- Host reviews before sending
- WebSocket notification to frontend

### Flow
```
Guest Message → Webhook → AI Query → Generate Response
    ↓
CoPilot = true?
    ↓ YES
├── Save to TempMessage collection
├── Emit 'tempMsg' event (Socket.io)
└── Frontend shows draft for review
```

### TempMessage Model
```javascript
{
  conversationId: ObjectId,    // Links to conversation
  senderId: 'ai-copilot',
  senderName: 'AI Copilot',
  content: String,             // Draft response
  timestamp: Date
}
```

### Host Actions
1. **Accept:** Send draft as-is
2. **Edit:** Modify before sending
3. **Reject:** Discard and write own response

## 2.5 Comparison Table

| Feature | AutoPilot | CoPilot |
|---------|-----------|---------|
| Response sent automatically | ✅ Yes | ❌ No |
| Human review required | ❌ No | ✅ Yes |
| Auto-escalates low confidence | ✅ Yes | ❌ No (saves draft) |
| Speed of response | Instant | Depends on host |
| Control over messaging | Low | High |
| Best for | High volume, FAQs | Complex, sensitive |
| Default mode | No | Yes |

## 2.6 Upsell Responses

**Special Rule:** Upsell messages **always auto-send** regardless of mode

```typescript
if (conversation.aiAutoPilot || apiResponse?.type === 'upsell') {
  // Auto-send even in CoPilot mode
  await client.sendConversationMessage(externalId, response);
}
```

---

# PART 3: AI AGENTS ARCHITECTURE

> **Source Files:**
> - AI Client: [`src/lib/clients/ai-client/ai-client.ts`](../src/lib/clients/ai-client/ai-client.ts)
>   - Methods: `queryAi()`, `queryConversationalAgent()`
>   - Knowledge Base: `processGlobalKnowledgeBase()`, `processListingKnowledgeBase()`
> - FastAPI Backend: Configured via `AIRBOT_AI_URL` environment variable
>   - Endpoints: `/agent/query-agent`, `/agent/query-conversational-agent`
>   - RAG: `/documents/process-knowledge-base`

## 3.1 Agent Overview

| Agent | Purpose | Model | Temperature |
|-------|---------|-------|-------------|
| **VacationRentalAgent** | General conversations | GPT-4o | 0.2 |
| **UpsellAgent** | Revenue optimization | GPT-4o | 0.5 |
| **RAG Pipeline** | Context retrieval | ChromaDB + Embeddings | N/A |

## 3.2 VacationRentalAgent (Main Agent)

### Configuration
```python
Model: GPT-4o (OpenAI)
Temperature: 0.2 (deterministic)
Max Iterations: 5
Verbose: False
```

### Persona
- Name: "Alex"
- Role: Professional human concierge
- **CRITICAL:** Never reveals being AI/bot/agent
- Multilingual: Responds in user's language

### Available Tools

**1. answer_query**
```python
Purpose: Respond with sufficient information
Input: answer (string)
Output: Final response to user
```

**2. escalate_to_human**
```python
Purpose: Escalate when AI cannot handle
Input: message (string)
Output: Escalation with auto-assigned tag
Uses: GPT-4o (temp 0.1) for tag assignment
```

### Decision Logic

Based on Knowledge Base Confidence:

| Confidence | Action |
|------------|--------|
| > 55% | Answer confidently |
| 30-55% | Answer only if clear/unambiguous |
| < 30% | Escalate to human |
| Missing details | Escalate to human |

## 3.3 UpsellAgent (Revenue Agent)

### Configuration
```python
Model: GPT-4o (OpenAI)
Temperature: 0.5 (more creative)
Max Iterations: 10
Verbose: True
```

### Strategy
- Focus ONLY on most recent upsell opportunity
- Do NOT bring up old upsells
- Collect ALL required data before offering
- Handle negotiations (fixed pricing policy)
- Multilingual support

### Available Tools

**1. check_availability**
```python
Purpose: Check if date/time is available
Input: upsell_id, datetime, listing_id
Output: "YES" or "NO - [reason]"
Checks: ListingUpsell.isActive, reservation conflicts
```

**2. get_upsell_metadata**
```python
Purpose: Get pricing and details
Input: upsell_id
Output: {price, listing_name, currency}
```

**3. send_upsell**
```python
Purpose: Process confirmed upsell
Input: upsell_type, reservation_id, pms_id
Actions:
├── Hostaway: Update reservation.financeField with fee
├── Lodgify: Internal tracking (API limitation)
├── Airbnb: Generate Resolution Center URL
├── Booking.com/VRBO: Internal tracking
└── Insert UpsellTransaction record
```

**4. no_upsell**
```python
Purpose: Politely decline if no opportunity
Output: "no"
```

## 3.4 RAG Pipeline

### Components
```
1. Vector Store: ChromaDB
2. Embedding Model: SentenceTransformer
3. Collections: listings, knowledgebase, additionaldocuments, sop
```

### Query Flow
```
User Query
    ↓
1. Fetch Listing Data (PostgreSQL)
    ↓
2. Query Knowledge Base (hierarchical)
   ├── Listing-specific KB (7 results)
   └── Global KB (5 results)
    ↓
3. Query Additional Documents (4 results)
    ↓
4. Calculate Confidence Scores
   └── confidence = max(0, min(100, (1 - distance) * 100))
    ↓
5. Apply Information Guard (filter sensitive data)
    ↓
6. Call VacationRentalAgent with context
    ↓
7. Return structured response
```

### Confidence Calculation
```python
# Distance ≤ 0.7 considered relevant
confidence = max(0, min(100, (1 - distance) * 100))

# Example:
# distance = 0.3 → confidence = 70%
# distance = 0.5 → confidence = 50%
# distance = 0.7 → confidence = 30%
```

---

# PART 4: UPSELL SYSTEM

> **Source Files:**
> - Routes: [`src/routes/upsell/upsell-routes.ts`](../src/routes/upsell/upsell-routes.ts)
> - Controller: [`src/controllers/upsell-controller.ts`](../src/controllers/upsell-controller.ts)
> - Service: [`src/services/upsell-service.ts`](../src/services/upsell-service.ts)
> - Worker & Scheduler:
>   - Worker: [`src/lib/workers/upsell/upsell-worker.ts`](../src/lib/workers/upsell/upsell-worker.ts)
>   - Scheduler: [`src/lib/workers/upsell/lib/scheduler/upsell-scheduler.ts`](../src/lib/workers/upsell/lib/scheduler/upsell-scheduler.ts)
>   - Event Handler: [`src/lib/workers/upsell/lib/handlers/index.ts`](../src/lib/workers/upsell/lib/handlers/index.ts)
> - Processors:
>   - Factory: [`src/lib/workers/upsell/lib/processors/upsell-processor-factory.ts`](../src/lib/workers/upsell/lib/processors/upsell-processor-factory.ts)
>   - Base: [`src/lib/workers/upsell/lib/processors/abstract-upsell-processor.ts`](../src/lib/workers/upsell/lib/processors/abstract-upsell-processor.ts)
>   - Early Check-in: [`src/lib/workers/upsell/lib/processors/early-checkin-processor.ts`](../src/lib/workers/upsell/lib/processors/early-checkin-processor.ts)
>   - Late Checkout: [`src/lib/workers/upsell/lib/processors/late-checkout-processor.ts`](../src/lib/workers/upsell/lib/processors/late-checkout-processor.ts)
>   - Mid-Stay: [`src/lib/workers/upsell/lib/processors/mid-stay-cleaning-processor.ts`](../src/lib/workers/upsell/lib/processors/mid-stay-cleaning-processor.ts)
>   - Gap Nights: [`src/lib/workers/upsell/lib/processors/pre-gap-stay-night-processor.ts`](../src/lib/workers/upsell/lib/processors/pre-gap-stay-night-processor.ts), etc.
> - Model: [`src/models/UpsellOpportunity.ts`](../src/models/UpsellOpportunity.ts) (MongoDB)

## 4.1 Upsell Types

| ID | Name | Required Data | Default Price |
|----|------|---------------|---------------|
| `EARLY_CHECK_IN` | Early Check-in | exact_time | $25 |
| `LATE_CHECK_OUT` | Late Check-out | exact_time | $50 |
| `MID_STAY_CLEANING` | Mid-Stay Cleaning | (none) | $75 |
| `PRE_STAY_GAP_NIGHT` | Pre-stay Gap Night | exact_date, exact_time | 10% discount |
| `POST_STAY_GAP_NIGHT_BEFORE` | Post-stay (Before) | exact_date, exact_time | 10% discount |
| `POST_STAY_GAP_NIGHT_AFTER` | Post-stay (After) | exact_date, exact_time | Variable |

## 4.2 Upsell Conditions

### When Upsells Are Offered

| Condition | Required |
|-----------|----------|
| Reservation is PAID | ✅ Yes |
| Upsell is ENABLED on listing | ✅ Yes |
| Not already offered (prevention check) | ✅ Yes |
| Date/time is AVAILABLE | ✅ Yes |
| Guest shows interest | ✅ Yes |

### Type-Specific Conditions

**EARLY_CHECK_IN:**
- Guest mentions arriving early
- Keywords: "flight lands", "can we check in before", "morning arrival"

**LATE_CHECK_OUT:**
- Guest mentions departing late
- Keywords: "late flight", "stay until", "checkout extension"

**MID_STAY_CLEANING:**
- Stay > 3 days
- Guest mentions cleaning need

**GAP_NIGHTS:**
- Adjacent date is available (no booking)
- Guest mentions extending stay

## 4.3 Scheduling Logic

### Scheduler Flow
```
Paid Reservation Created
    ↓
ReservationEventEmitter.emit('reservation:created')
    ↓
For each enabled upsell type:
├── Create UpsellOpportunity (MongoDB)
├── Calculate scheduleAt date
├── Create BullMQ job with delay
└── Update status = 'processed'
```

### Schedule Calculations

| Type | scheduleAt |
|------|------------|
| EARLY_CHECK_IN | arrival - 3 days |
| LATE_CHECK_OUT | departure - 3 days |
| MID_STAY_CLEANING | check-in + log₂(nights-2)+1 days |
| PRE_STAY_GAP_NIGHT | arrival - 3 days |
| POST_STAY_GAP_NIGHT | departure (same day) |

### Mid-Stay Cleaning Formula
```javascript
// Only for stays > 3 days
const offerDay = Math.log2(totalStayDays - 2) + 1;
const scheduleAt = checkInDate + clamp(offerDay, 2, 7) + "10:00 AM";

// Examples:
// 4 days → offer on day 2
// 7 days → offer on day 3
// 14 days → offer on day 4
```

## 4.4 Prevention Check

Before sending scheduled upsell:

```
BullMQ Job Triggers
    ↓
checkPrevention(conversationHistory)
├── Was upsell already offered?
├── Did guest decline?
└── Returns: { shouldOffer: boolean }
    ↓
If shouldOffer = false → Mark 'expired'
If shouldOffer = true → Continue to availability check
```

## 4.5 Platform-Specific Handling

| Platform | Action |
|----------|--------|
| **Hostaway** | Update reservation.financeField with fee |
| **Lodgify** | Internal tracking (API doesn't support fees) |
| **Airbnb** | Generate Resolution Center URL, guest pays in app |
| **Booking.com** | Internal tracking, guest portal payment |
| **VRBO** | Internal tracking, guest portal payment |
| **Direct** | Hostaway fee or internal tracking |

---

# PART 5: AGENT COORDINATION

## 5.1 Request Flow

```
POST /agent/query-agent
    ↓
[1] Fetch conversation from MongoDB
    ↓
[2] Extract: latest message, history, guest_name
    ↓
[3] Detect AVAILABILITY intent?
    ├── YES → Return availability response (no agents)
    ↓ NO
[4] Is reservation PAID?
    ├── YES → Run UPSELL AGENT
    │   ├── Returns response → Return UPSELL RESPONSE
    │   └── Returns "no" → Continue
    ↓ NO (or "no")
[5] Run RAG PIPELINE
    ├── Query vector stores
    ├── Calculate confidence
    ├── Apply Information Guard
    └── Call VACATION_RENTAL_AGENT
    ↓
[6] Return structured response
```

## 5.2 Agent Selection Logic

```python
# Decision tree
if availability_intent_detected:
    return availability_response()

if reservation.is_paid and has_listing_id and has_pms_id:
    upsell_response = upsell_agent.run(query)
    if upsell_response != "no":
        return upsell_response

# Default to main agent
return rag_pipeline.run(query)
```

## 5.3 Information Guard

### Pre-Check-in Restrictions
Cannot share until check-in date:
- Unit/Apartment number
- Access codes (lockbox, door, keypad)
- WiFi SSID/Password
- Parking spot number

### Pre-Confirmation Restrictions
Cannot share until booking confirmed:
- Floor number
- Building name
- Check-in procedure details

### Always Allowed
- Property amenities
- House rules
- Check-in/check-out times
- General parking info

## 5.4 Escalation System

> **Source Files:**
> - Repository: [`src/repository/conversation-repository.ts`](../src/repository/conversation-repository.ts)
>   - Functions: `toggleEscalation()`, `escalateConversationByExternalId()`, `getEscalatedConversationsWithUnreadCount()`
> - Controller: [`src/controllers/conversation-controller.ts`](../src/controllers/conversation-controller.ts)
>   - Method: `toggleEscalation()`
> - Model: [`src/models/Conversation.ts`](../src/models/Conversation.ts)
>   - Field: `tags.escalated` (Boolean)

### Escalation Tags (7 Categories)

**1. arrival_departure**
- EARLY_CHECKIN, LATE_CHECKIN
- EARLY_CHECKOUT, LATE_CHECKOUT

**2. property_access**
- LOST_KEY, EXTRA_KEY, ACCESS_CODE
- LOCKBOX_ISSUE, BUILDING_ACCESS

**3. maintenance_cleaning**
- CLEANING_REQUEST, MAINTENANCE_ISSUE
- REPAIR_REQUEST, WIFI_ISSUE

**4. guest_requests**
- PET_REQUEST, SMOKING_REQUEST
- EXTRA_GUESTS, BOOKING_EXTENSION

**5. policies_rules**
- RULE_EXCEPTION, POLICY_QUESTION
- COMPLAINT, REVIEW_CONCERNS

**6. financial**
- PAYMENT_ISSUE, REFUND_REQUEST
- PRICING_INQUIRY, DISCOUNT_REQUEST

**7. booking_management**
- BOOKING_MODIFICATION, BOOKING_CANCELLATION
- DIRECT_BOOKING, LONG_TERM_BOOKING

### Escalation Flow
```
Agent decides to escalate
    ↓
Call escalate_to_human(message)
    ↓
GPT-4o (temp 0.1) assigns tag
    ↓
Map tag → category + subcategory
    ↓
Set conversation.tags.escalated = true
    ↓
Notify host (Socket.io + notification)
    ↓
Skip AI processing for escalated conversations
```

---

# PART 6: API ENDPOINTS

## 6.1 Agent Endpoints

### Query Main Agent
```
POST /api/v1/agent/query-agent

Request:
{
  "conversation_id": "string",     // MongoDB externalId
  "workspace_id": "string",
  "listing_id": "string",
  "client_id": "string"
}

Response:
{
  "result": {
    "status": "answered|escalated|clarify",
    "type": "general|upsell|availability|escalation",
    "response": "string",
    "escalation_tag": "string (if escalated)",
    "category": "string",
    "subcategory": "string"
  },
  "context": {
    "knowledgebase": [...],
    "kb_confidence": 0-100
  }
}
```

### Query Sandbox Agent
```
POST /api/v1/agent/query-conversational-agent

Request:
{
  "conversation_id": "string"  // MongoDB ObjectId
}

Note: Uses sandbox_reservations, no upsell detection
```

## 6.2 Upsell Endpoints

### Suggest Upsells
```
POST /api/v1/upsells/suggest-upsells

Request:
{
  "chat_history": [
    {"role": "user|assistant", "content": "string"}
  ]
}

Response:
{
  "status": "success|error",
  "suggested_upsells": ["EARLY_CHECK_IN", "LATE_CHECK_OUT"],
  "error": ""
}
```

### Prevent Duplicate Offers
```
POST /api/v1/upsells/prevent-upsell

Request:
{
  "chat_history": [...],
  "upsell_metadata": {
    "upsell_id": "EARLY_CHECK_IN"
  }
}

Response:
{
  "shouldOffer": true|false
}
```

## 6.3 Document Endpoints

### Process Knowledge Base
```
POST /api/v1/documents/process-knowledge-base

Request: multipart/form-data
- kb_file: CSV/XLSX
- client_id: string
- workspace_id: string
- scope: "global" | "listing"
- listing_id: string (if scope="listing")

Response:
{
  "status": "success|error",
  "processed": 0,
  "error": ""
}
```

### Process SOP
```
POST /api/v1/documents/process-sop

Request: multipart/form-data
- sop_file: PDF/DOCX
- client_id: string

Response:
{
  "status": "success|error",
  "processed": 0
}
```

---

# PART 7: WEBHOOKS

## 7.1 Message Processing Webhook Flow

```
PMS Webhook (message.received)
    ↓
Backend: processMessageEvent()
    ↓
Sync conversation from PMS
    ↓
Check: AI enabled AND not escalated?
    ├── NO → Skip AI processing
    ↓ YES
Query FastAPI: /agent/query-agent
    ↓
Check response status and type
    ├── status = "escalated" (AutoPilot only)
    │   ├── Set conversation.tags.escalated = true
    │   ├── Tag conversation with category
    │   ├── Notify host
    │   └── Skip AI for future messages
    ├── type = "upsell" → Auto-send
    ├── aiAutoPilot = true → Auto-send
    └── aiCoPilot = true → Save to TempMessage
    ↓
Emit Socket.io events
├── 'aiMessage' (if auto-sent)
├── 'tempMsg' (if draft saved)
├── 'escalation' (if escalated)
└── 'receiveMessage' (to update UI)
```

## 7.2 Reservation Webhook Flow

```
PMS Webhook (reservation.created/updated)
    ↓
Backend: processReservationEvent()
    ↓
Map and sync reservation
    ↓
Check: isPaid = true?
    ├── NO → Skip upsell
    ↓ YES
Emit: 'reservation:created'
    ↓
UpsellEventHandler.handleReservationCreated()
    ↓
For each enabled upsell:
├── Create UpsellOpportunity
├── Calculate scheduleAt
└── Queue BullMQ job
```

---

# PART 8: QUICK REFERENCE

## Configuration Summary

| Setting | Location | Default |
|---------|----------|---------|
| Response Tone | Workspace/Listing | "neutral" |
| Conciseness | Workspace/Listing | "neutral" |
| AI Mode | Conversation | CoPilot |
| Documents | Workspace/Listing | [] |

## Agent Summary

| Agent | Trigger | Model | Temp |
|-------|---------|-------|------|
| VacationRental | Default | GPT-4o | 0.2 |
| Upsell | Paid reservation | GPT-4o | 0.5 |
| Escalation Tag | On escalate | GPT-4o | 0.1 |

## Upsell Summary

| Type | Schedule | Default Price |
|------|----------|---------------|
| Early Check-in | -3 days | $25 |
| Late Check-out | -3 days | $50 |
| Mid-Stay Clean | Calculated | $75 |
| Gap Nights | -3 days | 10% off |

---