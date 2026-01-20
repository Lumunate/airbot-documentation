---
layout: default
title: Business Logic
nav_order: 6
description: "Analysis of core business flows including authentication, payment, and PMS integration"
---

# Airbot - Business Logic Analysis

## Executive Summary

Airbot is an AI-powered vacation rental management platform with three core business flows:

1. **Auth → Payment**: User registration with waitlist data → Stripe trial checkout → Subscription management
2. **PMS Integration**: Connect Hostaway/Guesty/Lodgify → Sync listings/reservations/conversations → Real-time webhooks
3. **Upsells System**: The primary revenue driver - AI-powered upsell detection and automated guest offers

---

# PART 1: AUTHENTICATION → PAYMENT FLOW

## 1.1 Registration Business Logic

### User Registration Data Flow

```
User Signup Form
    ↓
┌─────────────────────────────────────────────────────┐
│ WAITLIST DATA (Stored for pricing/onboarding)       │
├─────────────────────────────────────────────────────┤
│ • subscriptionPackage: CORE | CUSTOM                │
│ • subscriptionBillingCycle: MONTHLY | ANNUALLY      │
│ • listings: "5" (number of properties)              │
│ • pricePerListing: $15 (CORE) or custom            │
│ • pricePerListingYearly: $12 (20% discount)        │
│ • airbotCoHosts: "yes" | "no" | "maybe"            │
│ • coHostMonthlyPrice / coHostYearlyPrice           │
│ • meetingDateTime: Discovery call scheduled         │
│ • pms: "hostaway,guesty" (comma-separated)         │
│ • features: Selected feature flags                  │
└─────────────────────────────────────────────────────┘
    ↓
Creates: User + Workspace + Waitlist records
    ↓
Sends: Email verification + Welcome email
    ↓
If meetingDateTime: Schedules Google Meet call
```

### Key Business Rules

| Rule | Implementation |
|------|----------------|
| Email Verification Required | User cannot access dashboard until verified |
| Auto-create Workspace | Every user gets a default workspace on registration |
| Waitlist Data Preserved | Used later for pricing calculation at checkout |
| Team Member Distinction | `isTeamMember=true` bypasses subscription checks |

---

## 1.2 Subscription & Pricing Model

### Pricing Structure

```
CORE PLAN
├── Monthly: $15/listing/month
├── Annually: $12/listing/month (20% savings)
└── Max Listings: 50

CUSTOM PLAN (Enterprise)
├── Custom pricing per agreement
├── Unlimited listings
└── Dedicated support

VIRTUEPRO (Premium Co-hosts Add-on)
├── Separate line item
├── coHostMonthlyPrice or coHostYearlyPrice from waitlist
└── Enabled after first payment succeeds

REFERRAL DISCOUNT
├── 10% lifetime discount via rewardfulReferralCode
├── Applies to all subscription charges
└── Set during user registration, applied at checkout
```

### Trial Checkout Flow

```
POST /subscription/billing/trial-checkout
    ↓
┌─────────────────────────────────────────────────────┐
│ STRIPE CHECKOUT SESSION CREATION                    │
├─────────────────────────────────────────────────────┤
│ 1. Get/Create Stripe Customer                       │
│    └─ Stored in User.stripeCustomerId               │
│                                                     │
│ 2. Calculate Trial End                              │
│    └─ 15 days from now (TRIAL_PERIOD_DAYS=15)      │
│                                                     │
│ 3. Create Listing Price                             │
│    ├─ listingCount × pricePerListing × multiplier  │
│    ├─ multiplier = 12 (annual) or 1 (monthly)      │
│    └─ Apply 10% Rewardful discount if referral     │
│                                                     │
│ 4. Create VirtualPro Price (if airbotCoHosts=yes)  │
│    └─ coHostPrice × multiplier                     │
│                                                     │
│ 5. Create Stripe Checkout Session                   │
│    ├─ mode: 'subscription'                          │
│    ├─ subscription_data.trial_end: 15 days         │
│    ├─ line_items: [listings, virtualPro?]          │
│    └─ metadata: userId, packageType, listingCount  │
└─────────────────────────────────────────────────────┘
    ↓
Returns: Stripe Checkout URL
    ↓
User completes payment on Stripe
    ↓
Stripe fires webhook: checkout.session.completed
```

### Pricing Example

```
User: 5 listings, CORE plan, Monthly, with Referral

Base:      5 × $15 × 1 month = $75.00
Discount:  $75 × 10% = $7.50
Final:     $67.50/month (lifetime discount)

Annual equivalent:
Base:      5 × $12 × 12 months = $720.00
Discount:  $720 × 10% = $72.00
Final:     $648.00/year ($54/month effective, lifetime discount)

Note: Referral discount is permanent and applies to all future billing cycles,
      including proration adjustments when listing count changes.
```

---

## 1.3 Stripe Webhook Events

### Critical Webhook Handlers

| Event | Business Action |
|-------|-----------------|
| `checkout.session.completed` | Create Subscription record, activate user |
| `customer.subscription.updated` | Sync status, update billing period |
| `customer.subscription.deleted` | Mark CANCELED, revoke access |
| `customer.subscription.trial_will_end` | Send "Trial Ending" email (5 days before) |
| `invoice.payment_succeeded` | Enable VirtualPro, send receipt email |
| `invoice.payment_failed` | Send "Payment Failed" email, status → PAST_DUE |

### Subscription Status Flow

```
TRIALING (15 days)
    ↓ (trial ends, payment charged)
ACTIVE
    ↓ (payment fails)
PAST_DUE
    ↓ (retries exhausted)
CANCELED
    ↓
Access Revoked → Redirect to /payment
```

---

## 1.4 Listing Count Synchronization

### Auto-Billing Adjustment

```
User adds new listing
    ↓
syncListingCount() triggered
    ↓
┌─────────────────────────────────────────────────────┐
│ LISTING COUNT SYNC                                  │
├─────────────────────────────────────────────────────┤
│ 1. Count actual listings across all workspaces     │
│ 2. Compare with subscription.currentListingCount   │
│ 3. If different AND within plan limits:            │
│    ├─ Create new Stripe price                       │
│    ├─ Update subscription with proration           │
│    └─ Stripe auto-calculates prorated charge       │
│ 4. If exceeds CORE limit (50):                     │
│    └─ Log warning, block listing creation          │
└─────────────────────────────────────────────────────┘
```

---

# PART 2: PMS INTEGRATION FLOW

## 2.1 Supported PMS Platforms

| PMS | Auth Method | API Base |
|-----|-------------|----------|
| **Hostaway** | OAuth 2.0 Client Credentials | `api.hostaway.com/v1` |
| **Guesty** | OAuth 2.0 Basic Auth | `open-api.guesty.com` |
| **Lodgify** | API Key Header | `api.lodgify.com/v2` |

---

## 2.2 Data Sync: What Each PMS Brings

### Hostaway (Complete Ecosystem)

```
LISTINGS
├── All property details (name, address, rooms, pricing)
├── Images and descriptions
├── Amenities and house rules
└── Subscription plans and agreements

RESERVATIONS
├── Guest details (name, email, phone)
├── Booking dates and times
├── Pricing breakdown (base, taxes, fees, cleaning)
├── Payment status and confirmation codes
└── Channel source (Airbnb, Booking.com, VRBO, Direct)

CONVERSATIONS
├── Full message history
├── Participant details with images
├── Read status per message
└── Scheduled messages

CALENDAR
├── 90-day availability window
├── Daily pricing
├── Minimum/maximum stay requirements
├── Closed dates (arrival/departure)
└── Available units count
```

### Guesty (Mostly Complete)

```
LISTINGS ✅
├── Property details and address
├── Base pricing and currency
└── Thumbnail images

RESERVATIONS ✅
├── Guest information
├── Money breakdown (base, cleaning, taxes)
├── Source channel
└── Confirmation codes

CONVERSATIONS ✅
├── Message history with pagination
├── Guest details
└── Read status

CALENDAR ❌ NOT IMPLEMENTED
└── getCalendar() returns empty array
```

### Lodgify (Functional with Quirks)

```
LISTINGS ✅
├── Property details with room breakdown
├── Pricing ranges (min/max)
├── ID normalization (multiple field formats)
└── Address parsing

RESERVATIONS ✅
├── Guest breakdown (adults, children, infants)
├── Booking status and payment info
├── Time parsing (multiple formats)
└── Channel mapping

CONVERSATIONS ⚠️ PARTIAL
├── Via booking threads (thread_uid)
├── Requires booking ID extraction from subject
└── Message sending needs booking context

CALENDAR ✅
├── Via rates API (not calendar API)
├── Requires room type filtering
└── Rate and availability data
```

---

## 2.3 Sync Process Flow

### Initial PMS Connection

```
User enters credentials → Validate with PMS API
    ↓
┌─────────────────────────────────────────────────────┐
│ INITIAL SYNC SEQUENCE                               │
├─────────────────────────────────────────────────────┤
│ 1. syncListings()                                   │
│    └─ Fetch all properties, map to unified format  │
│                                                     │
│ 2. syncReservationsForPMS()                        │
│    └─ For each listing, fetch all bookings         │
│                                                     │
│ 3. syncConversations()                             │
│    └─ Fetch message threads for each listing       │
│                                                     │
│ 4. createDefaultUpsells()                          │
│    └─ Enable standard upsell types per listing     │
│                                                     │
│ 5. createUnifiedWebhook()                          │
│    └─ Register for real-time event notifications   │
└─────────────────────────────────────────────────────┘
    ↓
Emit: syncProgress events to frontend via Socket.io
```

### Webhook Real-Time Updates

```
PMS Event → Webhook Endpoint
    ↓
Respond 200 immediately (prevent timeout)
    ↓
Process in background:
┌─────────────────────────────────────────────────────┐
│ RESERVATION EVENT                                   │
├─────────────────────────────────────────────────────┤
│ • Map event data to internal format                │
│ • Fetch full reservation details from PMS API      │
│ • Upsert in PostgreSQL database                    │
│ • Emit upsell event (if paid reservation)          │
│ • Create user notification                         │
│ • Update frontend via Socket.io                    │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ MESSAGE EVENT                                       │
├─────────────────────────────────────────────────────┤
│ • Sync conversation from PMS                       │
│ • Check AI settings (autopilot/copilot)            │
│ • If AI enabled:                                   │
│   ├─ Query FastAPI AI agent                        │
│   ├─ If autopilot: Send response to guest          │
│   ├─ If copilot: Save draft for host review        │
│   └─ If escalated: Tag conversation, notify host   │
│ • Emit messageEvent to frontend                    │
└─────────────────────────────────────────────────────┘
```

---

## 2.4 Data Transformation

### Unified Listing Model

```typescript
{
  id: UUID,                    // Internal ID
  externalId: string,          // PMS-specific ID
  pmsId: UUID,                 // Reference to PMS account
  workspaceId: UUID,
  data: {
    // Mapped fields (unified)
    name: string,
    address: { city, state, country, zip, full },
    location: { lat, lng },
    pricing: { min, max, currency },
    rooms: [{ id, name }],
    images: { thumbnail, gallery },
    meta: { rating, isActive, createdAt, updatedAt },

    // Raw PMS data (preserved)
    rawData: { ...original PMS response }
  },
  syncedAt: DateTime
}
```

### Unified Reservation Model

```typescript
{
  id: Int,                     // Auto-increment
  reservationId: string,       // Channel booking ID
  listingId: UUID,
  workspaceId: UUID,

  // Guest
  guestName: string,
  guestEmail: string,
  guestBreakdown: { adults, children, infants, pets },

  // Dates
  arrivalDate: Date,
  departureDate: Date,
  checkInTime: number,         // HHMM format (e.g., 1500)
  checkOutTime: number,
  nights: number,

  // Financials
  totalPrice: Decimal,
  taxAmount: Decimal,
  cleaningFee: Decimal,
  currency: string,

  // Status
  status: string,
  isPaid: boolean,
  confirmationCode: string,
  channelId: number,
  channelName: string
}
```

---

# PART 3: UPSELLS SYSTEM (Core Business Logic)

## 3.1 Upsell Types

| Type | Description | Default Price | Trigger Window |
|------|-------------|---------------|----------------|
| `EARLY_CHECK_IN` | Guest arrives before standard time | $25 | 3 days before arrival |
| `LATE_CHECK_OUT` | Guest departs after standard time | $50 | 3 days before departure |
| `MID_STAY_CLEANING` | Housekeeping during stay | $75 | Immediate (stays >3 days) |
| `PRE_STAY_GAP_NIGHT` | Extra night before booking | 10% discount | 3 days before |
| `POST_STAY_GAP_NIGHT_BEFORE` | Extension offer pre-arrival | 10% discount | 3 days before |
| `POST_STAY_GAP_NIGHT_AFTER` | Extension offer during stay | Variable | Day of checkout |

---

## 3.2 Backend Scheduler Architecture

### System Components

```
┌─────────────────────────────────────────────────────┐
│ UPSELL SYSTEM ARCHITECTURE                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ReservationEventEmitter                            │
│       ↓ (reservation:created)                       │
│                                                     │
│  UpsellEventHandler                                 │
│       ↓ (creates events for each upsell type)      │
│                                                     │
│  UpsellWorker                                       │
│       ↓ (routes to processors)                      │
│                                                     │
│  UpsellProcessorFactory                            │
│       ├─ EarlyCheckinProcessor                      │
│       ├─ LateCheckoutProcessor                      │
│       ├─ MidStayCleaningProcessor                   │
│       ├─ PreGapStayNightProcessor                   │
│       └─ PostGapStayNightProcessor (2 variants)    │
│       ↓ (calculates scheduleAt date)               │
│                                                     │
│  UpsellScheduler (BullMQ + Redis)                  │
│       ↓ (executes at scheduled time)               │
│                                                     │
│  PMS API (send message to guest)                   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Scheduling Logic by Type

```typescript
// EARLY_CHECK_IN
scheduleAt = arrivalDate - detectUpsellWindow (3 days)
// Offers early check-in option 3 days before arrival

// LATE_CHECK_OUT
scheduleAt = departureDate - detectUpsellWindow (3 days)
// Offers late checkout option 3 days before departure

// MID_STAY_CLEANING
if (totalStayDays <= 3) return null  // Skip short stays
offerDay = Math.log2(totalStayDays - 2) + 1  // Logarithmic formula
scheduleAt = checkInDate + clamp(offerDay, 2, 7) + "10:00 AM"
// Offers cleaning at optimal mid-point of stay

// GAP NIGHTS
// Checks calendar availability for adjacent dates
// Offers discounted extension if available
```

---

## 3.3 Complete Upsell Flow

### Phase 1: Reservation Created → Opportunities Scheduled

```
GUEST BOOKS RESERVATION (via PMS)
    ↓
Reservation synced to database
    ↓
If isPaid = true:
    ↓
ReservationEventEmitter.emit('reservation:created')
    ↓
┌─────────────────────────────────────────────────────┐
│ FOR EACH ENABLED UPSELL TYPE ON LISTING             │
├─────────────────────────────────────────────────────┤
│ 1. Create UpsellOpportunity (MongoDB)              │
│    { status: 'pending', type, listingId, etc. }    │
│                                                     │
│ 2. Route to type-specific Processor                │
│    └─ Validate eligibility                         │
│    └─ Calculate scheduleAt date                    │
│                                                     │
│ 3. Create BullMQ Job                               │
│    └─ Queue: 'upsell-execution'                    │
│    └─ Delay: scheduleAt - now                      │
│                                                     │
│ 4. Update UpsellOpportunity.status = 'processed'   │
└─────────────────────────────────────────────────────┘
```

### Phase 2: Scheduled Time → Message Sent

```
BULLMQ JOB TRIGGERS AT scheduleAt
    ↓
┌─────────────────────────────────────────────────────┐
│ PREVENTION CHECK                                    │
├─────────────────────────────────────────────────────┤
│ Call PMS API: checkPrevention()                    │
│ • Analyze conversation history                     │
│ • Check if upsell already offered                  │
│ • Check if guest declined                          │
│ Returns: { shouldOffer: boolean }                  │
└─────────────────────────────────────────────────────┘
    ↓
If shouldOffer = false → Mark 'expired', EXIT
    ↓
┌─────────────────────────────────────────────────────┐
│ AVAILABILITY CHECK (for time-based upsells)        │
├─────────────────────────────────────────────────────┤
│ • Early check-in: Is earlier time available?       │
│ • Late checkout: Is later time available?          │
│ • Gap nights: Are adjacent dates available?        │
└─────────────────────────────────────────────────────┘
    ↓
If unavailable → Mark 'expired', EXIT
    ↓
┌─────────────────────────────────────────────────────┐
│ SEND UPSELL MESSAGE                                 │
├─────────────────────────────────────────────────────┤
│ Populate template with variables:                  │
│ • {guest_name} → "John"                            │
│ • {listing_city} → "Miami"                         │
│ • {earliest_check_in} → "9:00 AM"                  │
│ • {price} → "$25"                                  │
│                                                     │
│ Example message:                                    │
│ "Hi John! We can offer early check-in at 9:00 AM  │
│  for just $25. Would you like to add this?"       │
│                                                     │
│ Send via: PMSClient.sendConversationMessage()      │
└─────────────────────────────────────────────────────┘
    ↓
Update UpsellOpportunity:
• status: 'offered'
• offeredAt: now
• offerConversationMessageId: message.id
```

### Phase 3: Guest Response → AI Processing

```
GUEST REPLIES TO UPSELL OFFER
    ↓
Webhook: message.received
    ↓
┌─────────────────────────────────────────────────────┐
│ AI AGENT ANALYSIS (FastAPI)                         │
├─────────────────────────────────────────────────────┤
│ 1. suggest_upsells(chat_history)                   │
│    • Analyzes conversation for opportunities       │
│    • Returns: ["EARLY_CHECK_IN"] or []             │
│                                                     │
│ 2. analyze_chat_for_upsell(history, upsell_id)    │
│    • Checks if already actively offered            │
│    • Returns: { shouldOffer: true/false }          │
│                                                     │
│ 3. If guest accepts (positive sentiment):          │
│    └─ UpsellAgent processes confirmation          │
└─────────────────────────────────────────────────────┘
```

### Phase 4: Upsell Confirmed → Transaction Recorded

```
GUEST CONFIRMS UPSELL
    ↓
UpsellAgent.send_upsell(type, reservationId, pmsId)
    ↓
┌─────────────────────────────────────────────────────┐
│ PMS-SPECIFIC PROCESSING                             │
├─────────────────────────────────────────────────────┤
│ HOSTAWAY:                                           │
│ • Create fee object with pricing                   │
│ • Update reservation.financeField                  │
│ • Fee appears on guest invoice                     │
│                                                     │
│ LODGIFY:                                            │
│ • API doesn't support fee updates                  │
│ • Create internal tracking record                  │
│                                                     │
│ AIRBNB CHANNEL:                                     │
│ • Generate Resolution Center URL                   │
│ • Guest receives payment request in app            │
│ • Auto-charge within 24 hours if accepted          │
│                                                     │
│ BOOKING.COM / VRBO:                                 │
│ • Internal tracking + guest portal payment         │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ DATABASE RECORD                                     │
├─────────────────────────────────────────────────────┤
│ INSERT INTO UpsellTransaction {                    │
│   reservationId,                                   │
│   upsellType: "EARLY_CHECK_IN",                   │
│   price: 25.00,                                    │
│   status: "applied",                               │
│   hostawayFeeId: "fee_123",                       │
│   hostawayResponse: {...},                         │
│   createdAt: now                                   │
│ }                                                   │
└─────────────────────────────────────────────────────┘
    ↓
Send email notification to host with details
```

---

## 3.4 AI Agent Architecture (FastAPI)

### Two-Agent System

```
┌─────────────────────────────────────────────────────┐
│ VacationRentalAgent (General Conversations)         │
├─────────────────────────────────────────────────────┤
│ Model: GPT-4o, Temperature: 0.2 (deterministic)    │
│                                                     │
│ Handles:                                            │
│ • Property questions (amenities, location)         │
│ • Check-in/check-out instructions                  │
│ • House rules and policies                         │
│ • General guest inquiries                          │
│                                                     │
│ Does NOT: Focus on upsells                         │
│ Escalates: Special requests to human               │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ UpsellAgent (Revenue Optimization)                  │
├─────────────────────────────────────────────────────┤
│ Model: GPT-4o, Temperature: 0.5 (creative)         │
│ Framework: LangChain ReAct Agent                   │
│                                                     │
│ Available Tools:                                    │
│ ├─ check_availability(upsell_id, datetime)         │
│ │   └─ Validates date/time is bookable             │
│ ├─ get_upsell_metadata(upsell_id)                  │
│ │   └─ Fetches price, listing name, currency       │
│ ├─ send_upsell(type, reservation_id, pms_id)       │
│ │   └─ Processes & records upsell transaction      │
│ └─ no_upsell()                                      │
│     └─ Politely declines if no opportunity         │
└─────────────────────────────────────────────────────┘
```

### AI Detection Logic

```python
# suggest_upsells() prompt logic:

EARLY_CHECK_IN triggers when guest mentions:
• "arriving early", "flight lands at 9am"
• "can we check in before 4pm?"
• "we'll be there in the morning"

LATE_CHECK_OUT triggers when guest mentions:
• "late flight", "leaving in the evening"
• "can we stay until 5pm?"
• "checkout extension"

GAP_NIGHTS triggers when guest mentions:
• "can we extend our stay?"
• "add one more night"
• "arrive a day early"

MID_STAY_CLEANING triggers when:
• Stay > 3 days AND guest mentions cleaning
• "can we get housekeeping?"
• Long stay without cleaning request

NO UPSELL when:
• Just confirming times (no intent to change)
• General questions about the property
• Already declined an offer
```

---

## 3.5 Revenue Tracking

### UpsellTransaction Model

```prisma
model UpsellTransaction {
  id              String   @id
  reservationId   Int
  upsellType      UpsellType
  price           Decimal
  status          String   // 'pending' | 'applied' | 'declined'
  hostawayFeeId   String?
  hostawayResponse Json?
  createdAt       DateTime
}
```

### Analytics Endpoints

```
GET /upsell-transactions/:workspaceId
    → Paginated list filtered by status/type

GET /upsell-transactions/revenue
    → Aggregated revenue by type and period

GET /upsell-transactions/offered
    → Count of offers sent (conversion rate calculation)

GET /workspace/:id/upsells/stats
    → Complete insights dashboard data
```

---

## 3.6 Business Rules Summary

| Rule | Implementation |
|------|----------------|
| **Paid Reservations Only** | Events only fire when `isPaid=true` |
| **Enabled Upsells Only** | Must have `isActive=true` on ListingUpsell |
| **Prevention Check** | AI analyzes history before each offer |
| **No Duplicate Offers** | Tracks `offerConversationMessageId` |
| **Calendar Validation** | Time-based upsells check availability |
| **Stay Length Check** | Mid-stay cleaning requires >3 night stay |
| **Platform-Specific Handling** | Airbnb uses Resolution Center, others differ |

---

# PART 4: SYSTEM INTEGRATION DIAGRAM

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AIRBOT PLATFORM                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │   FRONTEND   │     │   BACKEND    │     │   FASTAPI    │        │
│  │   (Next.js)  │◄───►│  (Express)   │◄───►│  (Python)    │        │
│  └──────────────┘     └──────┬───────┘     └──────────────┘        │
│                              │                    │                  │
│                              ▼                    ▼                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    DATA LAYER                                │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │  PostgreSQL          MongoDB           Redis                 │   │
│  │  (Prisma)            (Mongoose)        (BullMQ)             │   │
│  │  • Users             • Conversations   • Job Queue          │   │
│  │  • Subscriptions     • Notifications   • Scheduled Tasks    │   │
│  │  • Listings          • Temp Messages                        │   │
│  │  • Reservations                                              │   │
│  │  • Upsells                                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  EXTERNAL INTEGRATIONS                       │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │                                                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │   │
│  │  │ HOSTAWAY │  │  GUESTY  │  │ LODGIFY  │  │  STRIPE  │   │   │
│  │  │   API    │  │   API    │  │   API    │  │   API    │   │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │   │
│  │                                                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │   │
│  │  │  OPENAI  │  │  RESEND  │  │ CLOUDINARY│                  │   │
│  │  │  GPT-4o  │  │  Email   │  │  Storage │                  │   │
│  │  └──────────┘  └──────────┘  └──────────┘                  │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

# PART 5: KEY BUSINESS METRICS

## Revenue Model

```
SUBSCRIPTION REVENUE
├── Per-listing monthly/annual fees
├── VirtualPro add-on fees
└── Enterprise custom pricing

UPSELL REVENUE (Platform Commission)
├── Early check-in fees
├── Late checkout fees
├── Gap night bookings
└── Mid-stay cleaning fees
```

## Success Metrics

| Metric | Tracked Via |
|--------|-------------|
| Subscription MRR | Stripe webhooks → Invoice.amountPaid |
| Upsell Conversion Rate | Offered count vs Applied count |
| Upsell Revenue | UpsellTransaction.price aggregation |
| AI Response Rate | Autopilot messages sent vs received |
| Platform Usage | Listing count per workspace |

---

This document provides the complete business logic perspective of the Airbot platform, from user registration through payment processing, PMS data synchronization, and the sophisticated AI-powered upsell system that drives revenue optimization for vacation rental hosts.
