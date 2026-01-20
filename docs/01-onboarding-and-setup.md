---
layout: default
title: Onboarding & Setup
nav_order: 2
description: "Complete guide to user registration, subscription management, workspace creation, and platform setup"
---

# AIRBOT DOCUMENTATION: Onboarding & Setup

## Overview

This document covers the complete onboarding flow from user registration to full platform utilization.

---

## 1. USER REGISTRATION

> **Source Files:**
> - Routes: [`src/routes/auth/auth-routes.ts`](../src/routes/auth/auth-routes.ts)
> - Controller: [`src/controllers/auth-controller.ts`](../src/controllers/auth-controller.ts)
> - Service: [`src/services/auth-service.ts`](../src/services/auth-service.ts)

### 1.1 Signup Process

**Endpoint:** `POST /auth/register`

**Registration Data Collected:**
```
REQUIRED:
├── email
├── password (min 8 chars)
├── firstName, lastName

OPTIONAL (Waitlist/Onboarding):
├── contactNumber
├── listings (1-50)
├── teamMembers (1-5)
├── regions (countries)
├── pms[] (Hostaway, Guesty, Lodgify, etc.)
├── features[] (Premium co-hosts, Upsells, etc.)
├── airbotCoHosts: "yes" | "no" | "maybe"
├── subscriptionPackage: CORE | CUSTOM
├── subscriptionBillingCycle: MONTHLY | ANNUALLY
├── meetingDateTime (discovery call)
└── rewardfulReferralCode (affiliate) → 10% lifetime discount
```

**Registration Flow:**
```
1. Validate email uniqueness
2. Hash password with bcrypt
3. Create User record
4. Auto-create default Workspace
5. Store Waitlist data (for pricing)
6. Send verification email
7. Send welcome email
8. If meetingDateTime: Schedule Google Meet
```

### 1.2 Email Verification

**Endpoint:** `POST /auth/verify-email`

```
User clicks email link → /verify?token=<TOKEN>
    ↓
Token validated
    ↓
User.isEmailVerified = true
    ↓
Redirect to login
```

### 1.3 Login Process

**Endpoint:** `POST /auth/login`

**Returns:**
- `accessToken` (JWT, 1 day expiry)
- `refreshToken` (long-lived)
- User object with subscription status

**Post-Login Redirects:**
```
Team Member? → /overview
Any Plan + Virtual Pro? → /confirmation
Incomplete Plan? → /payment
No Subscription? → /payment
Active Subscription? → /overview
```

---

## 2. SUBSCRIPTION & PAYMENT

> **Source Files:**
> - Routes: [`src/routes/subscription/subscription-routes.ts`](../src/routes/subscription/subscription-routes.ts)
> - Controller: [`src/controllers/subscription-controller.ts`](../src/controllers/subscription-controller.ts)
> - Stripe Webhooks: [`src/controllers/stripe-webhook-controller.ts`](../src/controllers/stripe-webhook-controller.ts)
> - Stripe Service: [`src/services/stripe.service.ts`](../src/services/stripe.service.ts)
> - Stripe Config: [`src/config/stripe.ts`](../src/config/stripe.ts)

### 2.1 Pricing Model

| Plan | Monthly | Annual | Max Listings |
|------|---------|--------|--------------|
| **CORE** | $15/listing | $12/listing | 50 |
| **CUSTOM** | Negotiated | Negotiated | Unlimited |

**VirtualPro Add-on:** Custom pricing (coHostMonthlyPrice/coHostYearlyPrice)

**Referral Discount:**
- Users who sign up with a `rewardfulReferralCode` receive **10% lifetime discount**
- Discount applies to all subscription charges (listings + VirtualPro)
- Automatically applied during checkout session creation

### 2.2 Trial Checkout Flow

**Endpoint:** `POST /subscription/billing/trial-checkout`

```
Request:
├── packageType: CORE | CUSTOM
├── billingCycle: MONTHLY | ANNUALLY
├── listingCount: number
├── pricePerListing?: number (for CUSTOM)
├── virtualPro?: boolean
└── successUrl, cancelUrl

Flow:
1. Get/Create Stripe Customer
2. Calculate trial end (15 days)
3. Create listing price:
   └── listingCount × pricePerListing × multiplier
   └── Apply 10% Rewardful discount if applicable
4. Create VirtualPro price (if selected)
5. Create Stripe Checkout Session
6. Return checkout URL
```

### 2.3 Stripe Webhooks

| Event | Action |
|-------|--------|
| `checkout.session.completed` | Create Subscription, activate user |
| `customer.subscription.updated` | Sync status, billing period |
| `customer.subscription.deleted` | Mark CANCELED, revoke access |
| `customer.subscription.trial_will_end` | Send "Trial Ending" email |
| `invoice.payment_succeeded` | Enable VirtualPro, send receipt |
| `invoice.payment_failed` | Send "Payment Failed" email |

### 2.4 Subscription Status Flow

```
TRIALING → ACTIVE → PAST_DUE → CANCELED
              ↓
         Access Revoked
```

### 2.5 Auto-Billing Sync

When listing count changes:
```
syncListingCount()
├── Count actual listings
├── Compare with subscription.currentListingCount
├── If different:
│   ├── Create new Stripe price
│   ├── Update subscription with proration
│   └── Stripe calculates prorated charge
└── If exceeds CORE limit (50): Block
```

---

## 3. PMS CONNECTION

> **Source Files:**
> - Routes: [`src/routes/pms/pms-routes.ts`](../src/routes/pms/pms-routes.ts)
> - Controller: [`src/controllers/pms-controller.ts`](../src/controllers/pms-controller.ts)
> - Service: [`src/services/pms-service.ts`](../src/services/pms-service.ts)
> - PMS Clients:
>   - Hostaway: [`src/lib/clients/pms-clients/hostaway-client.ts`](../src/lib/clients/pms-clients/hostaway-client.ts)
>   - Guesty: [`src/lib/clients/pms-clients/guesty-client.ts`](../src/lib/clients/pms-clients/guesty-client.ts)
>   - Lodgify: [`src/lib/clients/pms-clients/lodgify-client.ts`](../src/lib/clients/pms-clients/lodgify-client.ts)

### 3.1 Supported Platforms

| PMS | Auth Method | Capabilities |
|-----|-------------|--------------|
| **Hostaway** | OAuth 2.0 | Full (listings, reservations, conversations, calendar) |
| **Guesty** | OAuth 2.0 | Partial (no calendar) |
| **Lodgify** | API Key | Full (conversations via threads) |

### 3.2 Connection Flow

**Endpoint:** `POST /pms`

```
1. User enters credentials
2. validatePMSCredentials() tests connection
3. If valid: Create PMS record
4. Trigger initial sync:
   ├── syncListings()
   ├── syncReservationsForPMS()
   ├── syncConversations()
   ├── createDefaultUpsells()
   └── createUnifiedWebhook()
5. Progress via Socket.io
```

### 3.3 PMS Webhooks

> **Source Files:**
> - Routes: [`src/routes/webhook/webhook-routes.ts`](../src/routes/webhook/webhook-routes.ts)
> - Controller: [`src/controllers/webhook-controller.ts`](../src/controllers/webhook-controller.ts)
> - Service: [`src/services/webhook-service.ts`](../src/services/webhook-service.ts)

**Hostaway:** `POST /webhooks/unified?workspaceId=X`
```
Events: reservation.created, reservation.updated, message.received
```

**Guesty:** `POST /webhooks/guesty?workspaceId=X`
```
Events: reservation.new, reservation.updated, message.received, listing.new
```

**Lodgify:** `POST /webhooks/lodgify?workspaceId=X`
```
Events: booking_*, guest_message_received, rate_change, availability_change
```

### 3.4 Data Synced

| Data Type | Hostaway | Guesty | Lodgify |
|-----------|----------|--------|---------|
| Listings | ✅ Full | ✅ Full | ✅ Full |
| Reservations | ✅ Full | ✅ Full | ✅ Full |
| Conversations | ✅ Full | ✅ Full | ⚠️ Via threads |
| Calendar | ✅ Full | ❌ None | ✅ Via rates API |

---

## 4. TEAM MEMBER MANAGEMENT

> **Source Files:**
> - Routes: [`src/routes/workspace/workspace-routes.ts`](../src/routes/workspace/workspace-routes.ts)
> - Controller: [`src/controllers/workspace-controller.ts`](../src/controllers/workspace-controller.ts)
> - Service: [`src/services/workspace-service.ts`](../src/services/workspace-service.ts)
> - Types: [`src/types/workspaces/add-workspace-member.ts`](../src/types/workspaces/add-workspace-member.ts)

### 4.1 Roles & Hierarchy

```
OWNER (50)   → Full control, billing, delete workspace
ADMIN (40)   → Full control except billing/delete
MANAGER (30) → Create/update resources
MEMBER (20)  → Limited write access
VIEWER (10)  → Read-only access
```

### 4.2 Invite Flow

**Endpoint:** `POST /workspace/:workspaceId/members`

```
1. Owner/Admin sends invitation (email, name, role)
2. TeamMember record created (pending)
3. Email sent with invitation link
4. Invitee creates account or accepts
5. TeamMember activated
```

### 4.3 Team Member Access

- Team members are **workspace-restricted**
- Cannot access other workspaces
- Subscription checks **bypassed** for team members
- See [02-access-control.md](./02-access-control.md) for complete RBAC matrix

---

## 5. AI INTEGRATION SETUP

> **Source Files:**
> - Workspace AI Routes: [`src/routes/workspace/ai-settings/ai-settings-routes.ts`](../src/routes/workspace/ai-settings/ai-settings-routes.ts)
> - Listing AI Routes: [`src/routes/ai-settings/ai-settings-routes.ts`](../src/routes/ai-settings/ai-settings-routes.ts)
> - Workspace AI Controller: [`src/controllers/workspace-ai-settings-controller.ts`](../src/controllers/workspace-ai-settings-controller.ts)
> - Listing AI Controller: [`src/controllers/ai-settings-controller.ts`](../src/controllers/ai-settings-controller.ts)
> - Services:
>   - [`src/services/workspace-ai-settings-service.ts`](../src/services/workspace-ai-settings-service.ts)
>   - [`src/services/ai-settings-service.ts`](../src/services/ai-settings-service.ts)

### 5.1 AI Configuration Levels

**Workspace Level (Global):**
```prisma
WorkspaceAISettings {
  responseTone: "neutral" | "friendly" | "formal" | "casual"
  conciseness: "concise" | "neutral" | "verbose"
  generalNotes: Text (custom instructions)
  knowledgeBase: JSON
  documents: WorkspaceDocument[]
}
```

**Listing Level (Override):**
```prisma
AISettings {
  responseTone: String? (optional override)
  conciseness: String? (optional override)
  generalNotes: String?
  knowledgeBase: JSON?
  documents: Document[]
}
```

**Hierarchy:**
```
Listing Settings > Workspace Settings > Defaults ("neutral")
```

### 5.2 AI Modes

**AutoPilot Mode:**
- AI automatically responds to guests
- Responses sent directly to PMS
- No human review needed
- Upsell messages always auto-send
- **Automatically escalates** when confidence is low or question is out of context

**CoPilot Mode (Default):**
- AI generates draft responses
- Saved to TempMessage collection
- Host reviews before sending
- WebSocket notification to frontend
- Saves draft even for low-confidence responses (host decides)

### 5.3 Knowledge Base Setup

**Document Upload:**
```
POST /workspace/:workspaceId/documents          (global)
POST /workspace/:workspaceId/listings/:id/documents (listing)
```

**Supported Formats:** PDF, DOCX

**Processing:**
1. Upload document
2. Send to FastAPI for chunking
3. Store in ChromaDB vector store
4. Update document record with chunk count

**Knowledge Base Scopes:**
- `global` - Applies to all listings in workspace
- `listing` - Applies to specific listing only

### 5.4 Initial AI Setup Checklist

```
☐ Configure workspace AI settings (tone, conciseness)
☐ Add general notes (business rules, policies)
☐ Upload global documents (house rules, FAQs)
☐ Configure per-listing overrides (if needed)
☐ Set conversation mode (AutoPilot/CoPilot)
☐ Test in sandbox environment
```

---

## 6. ONBOARDING CHECKLIST

### Complete User Journey

```
STEP 1: REGISTRATION
├── ☐ Sign up with email/password
├── ☐ Select plan (Core/Custom)
├── ☐ Choose billing cycle
├── ☐ Schedule discovery call (optional)
└── ☐ Verify email

STEP 2: PAYMENT
├── ☐ Confirm pricing (if Pro/Custom)
├── ☐ Complete Stripe checkout
├── ☐ Start 15-day trial
└── ☐ Access dashboard

STEP 3: PMS CONNECTION
├── ☐ Navigate to Settings → Integrations
├── ☐ Select PMS provider
├── ☐ Enter API credentials
├── ☐ Wait for initial sync
└── ☐ Verify listings/reservations imported

STEP 4: AI CONFIGURATION
├── ☐ Set workspace AI settings
├── ☐ Upload knowledge base documents
├── ☐ Configure response tone
├── ☐ Add custom instructions
└── ☐ Choose AutoPilot/CoPilot mode

STEP 5: TEAM SETUP (Optional)
├── ☐ Navigate to Settings → Manage Members
├── ☐ Invite team members
├── ☐ Assign appropriate roles
└── ☐ Share access guidelines

STEP 6: UPSELL CONFIGURATION
├── ☐ Review default upsell types
├── ☐ Set pricing per upsell
├── ☐ Enable/disable per listing
└── ☐ Customize message templates

STEP 7: GO LIVE
├── ☐ Enable AutoPilot (or review CoPilot)
├── ☐ Monitor first conversations
├── ☐ Review upsell offers
└── ☐ Adjust AI settings as needed
```

---

## 7. RELATED WEBHOOKS

### 7.1 Stripe Webhooks

> **Source:** [`src/controllers/stripe-webhook-controller.ts`](../src/controllers/stripe-webhook-controller.ts)

**Endpoint:** `POST /subscription/webhooks/stripe`

| Event | Handler |
|-------|---------|
| `checkout.session.completed` | `handleCheckoutSessionCompleted` |
| `customer.subscription.created` | `handleSubscriptionCreated` |
| `customer.subscription.updated` | `handleSubscriptionUpdated` |
| `customer.subscription.deleted` | `handleSubscriptionDeleted` |
| `customer.subscription.trial_will_end` | `handleTrialWillEnd` |
| `invoice.created` | `handleInvoiceCreated` |
| `invoice.payment_succeeded` | `handleInvoicePaymentSucceeded` |
| `invoice.payment_failed` | `handleInvoicePaymentFailed` |

### 7.2 PMS Webhooks

> **Source:** [`src/controllers/webhook-controller.ts`](../src/controllers/webhook-controller.ts)

**Hostaway:** `POST /webhooks/unified`
```javascript
Events:
├── reservation.created → Sync + Emit upsell event + Notify
├── reservation.updated → Sync + Update database
└── message.received → Sync + Process AI response
```

**Guesty:** `POST /webhooks/guesty`
```javascript
Events:
├── reservation.new → Sync + Emit upsell event
├── reservation.updated → Sync + Update database
├── message.received → Sync + Process AI response
└── listing.new → Sync listings
```

**Lodgify:** `POST /webhooks/lodgify`
```javascript
Events:
├── booking_* → Sync + Emit upsell event
├── guest_message_received → Sync + Process AI response
├── rate_change → Sync calendar
└── availability_change → Sync calendar
```

This document provides the complete onboarding journey for new Airbot users, from registration through full platform configuration.
