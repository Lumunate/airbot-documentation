---
layout: default
title: Access Control & Permissions
nav_order: 3
description: "Detailed explanation of the role-based access control (RBAC) system and user permissions"
---

# AIRBOT DOCUMENTATION: Access Control & Permissions

## Overview

This document explains who can do what in Airbot. We use a **role-based system** where each team member is assigned a role that determines their permissions.

**Think of it like a company:**
- **OWNER** = CEO (can do everything)
- **ADMIN** = Manager (can do most things, but not billing)
- **MANAGER** = Team Lead (handles day-to-day operations)
- **MEMBER** = Employee (can help with conversations)
- **VIEWER** = Intern (can only look, not touch)

> **Source Files:**
> - Middleware: [`src/middleware/rbac-middleware.ts`](../src/middleware/rbac-middleware.ts)
> - Workspace Routes: [`src/routes/workspace/workspace-routes.ts`](../src/routes/workspace/workspace-routes.ts)
> - Team Member Types: [`src/types/workspaces/add-workspace-member.ts`](../src/types/workspaces/add-workspace-member.ts)

---

## Understanding the 5 Roles

### 🔴 OWNER - Full Control
**Who:** The person who created the workspace (you!)

**What they can do:**
- ✅ Everything an ADMIN can do, PLUS:
- 💳 Manage billing and subscription
- 🗑️ Delete the entire workspace
- 👑 Transfer ownership to someone else
- 💰 View all payment information

**Best for:** The business owner or primary account holder

---

### 🟠 ADMIN - Almost Full Control
**Who:** Your trusted manager or business partner

**What they can do:**
- ✅ Everything a MANAGER can do, PLUS:
- 👥 Add, remove, and change team member roles
- 🔌 Connect and manage PMS integrations (Hostaway, Guesty, Lodgify)
- ⚙️ Update workspace settings
- 🗑️ Delete documents and AI configurations

**What they CANNOT do:**
- ❌ Manage billing or subscription
- ❌ Delete the workspace
- ❌ Assign someone as OWNER

**Best for:** Co-managers who need full operational control

---

### 🟡 MANAGER - Day-to-Day Operations
**Who:** Your operations lead or property manager

**What they can do:**
- ✅ Everything a MEMBER can do, PLUS:
- 🔄 Sync listings and reservations
- 🤖 Configure AI settings (tone, responses, knowledge base)
- 📄 Upload documents and knowledge base files
- 💰 Create and configure upsell rules
- 🔀 Switch conversations between AutoPilot and CoPilot mode

**What they CANNOT do:**
- ❌ Add or remove team members
- ❌ Connect PMS platforms
- ❌ Delete AI settings or documents

**Best for:** Operations managers who handle daily property management

---

### 🟢 MEMBER - Guest Communication
**Who:** Your guest relations team or virtual assistants

**What they can do:**
- ✅ Everything a VIEWER can do, PLUS:
- 💬 Send messages to guests
- 📤 Send AI-suggested responses
- 🚨 Escalate conversations when they need help

**What they CANNOT do:**
- ❌ Change AI settings
- ❌ Sync data from PMS
- ❌ Upload documents
- ❌ Configure upsells

**Best for:** Front-line staff who respond to guests

---

### 🔵 VIEWER - Read-Only Access
**Who:** Accountants, auditors, or trainees

**What they can do:**
- 👀 View everything (listings, reservations, conversations, analytics)
- 📊 See AI suggestions
- 📈 Check upsell statistics

**What they CANNOT do:**
- ❌ Send messages
- ❌ Change any settings
- ❌ Upload or delete anything

**Best for:** People who need to monitor but not interact

---

## Quick Permission Guide

### "Can I...?" Chart

| Action | Owner | Admin | Manager | Member | Viewer |
|--------|:-----:|:-----:|:-------:|:------:|:------:|
| **View everything** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Send messages to guests** | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Change AI settings** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Add team members** | ✅ | ✅ | ❌ | ❌ | ❌ |
| **Connect PMS (Hostaway, etc.)** | ✅ | ✅ | ❌ | ❌ | ❌ |
| **Manage billing** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Delete workspace** | ✅ | ❌ | ❌ | ❌ | ❌ |

---

## Common Scenarios

### Scenario 1: Growing Your Team

**You're an OWNER and want to bring on help:**

1. **Hire a business partner** → Assign them **ADMIN** role
   - They can manage everything except billing

2. **Hire an operations manager** → Assign them **MANAGER** role
   - They can handle listings, AI, upsells, and daily operations

3. **Hire guest service agents** → Assign them **MEMBER** role
   - They can respond to guests and escalate when needed

4. **Onboard an accountant** → Assign them **VIEWER** role
   - They can review everything but won't accidentally change anything

---

### Scenario 2: What Each Role Can Do with AI

| Task | Owner | Admin | Manager | Member | Viewer |
|------|:-----:|:-----:|:-------:|:------:|:------:|
| View AI responses | ✅ | ✅ | ✅ | ✅ | ✅ |
| Send AI suggestions | ✅ | ✅ | ✅ | ✅ | ❌ |
| Switch AutoPilot/CoPilot | ✅ | ✅ | ✅ | ❌ | ❌ |
| Change AI tone/style | ✅ | ✅ | ✅ | ❌ | ❌ |
| Upload knowledge base docs | ✅ | ✅ | ✅ | ❌ | ❌ |
| Delete AI documents | ✅ | ✅ | ❌ | ❌ | ❌ |

---

### Scenario 3: What Each Role Can Do with Conversations

| Task | Owner | Admin | Manager | Member | Viewer |
|------|:-----:|:-----:|:-------:|:------:|:------:|
| Read messages | ✅ | ✅ | ✅ | ✅ | ✅ |
| Send messages | ✅ | ✅ | ✅ | ✅ | ❌ |
| Escalate conversation | ✅ | ✅ | ✅ | ✅ | ❌ |
| Toggle AutoPilot mode | ✅ | ✅ | ✅ | ❌ | ❌ |

---

## Special Access Rules

### Team Member Restrictions

**Important:** Team members can ONLY access the workspace they're invited to.

Example:
- John invites Sarah to "Beach House Properties" workspace
- Sarah tries to access "Mountain Cabins" workspace
- ❌ Sarah gets blocked - she can only work in Beach House Properties

**Why?** This keeps your data secure and prevents accidental access to the wrong properties.

---

### VirtuePro Users (Special Co-hosts)

If you purchase **VirtuePro** (premium co-host service), VirtuePro team members get automatic **MANAGER** access to your workspace.

**They can:**
- ✅ Handle all daily operations
- ✅ Configure AI and upsells
- ✅ Respond to guests

**They cannot:**
- ❌ Add/remove your team members
- ❌ Change billing
- ❌ Connect/disconnect PMS platforms

**Requirements:**
- Your subscription must be active
- Your workspace must have VirtuePro enabled

---

## How to Invite Team Members

### Step 1: Navigate to Settings
Go to **Settings** → **Manage Members**

### Step 2: Click "Add Team Member"
Fill in:
- Email address
- Full name
- Role (Owner, Admin, Manager, Member, or Viewer)

### Step 3: They Receive an Invitation
They'll get an email with a link to:
- Create an account (if new to Airbot)
- Accept the invitation (if they already have an account)

### Step 4: They Gain Access
Once accepted, they can log in and access your workspace with their assigned permissions.

---

## Role Inheritance Explained

**Higher roles automatically get all permissions of lower roles.**

Think of it like stairs:
```
     OWNER (All permissions)
       ↑
     ADMIN (Almost all)
       ↑
    MANAGER (Operations)
       ↑
     MEMBER (Communication)
       ↑
     VIEWER (Read-only)
```

**Example:**
- If MEMBER can "send messages"
- Then MANAGER, ADMIN, and OWNER can also "send messages"

**This means:**
- OWNER can do everything everyone else can do
- ADMIN can do everything MANAGER, MEMBER, and VIEWER can do
- And so on...

---

## What You Can Do in Each Section

### 📊 Dashboard & Analytics
- **Everyone:** View statistics, revenue reports, message insights
- **No restrictions:** All roles can see analytics

### 📬 Inbox (Conversations)
- **VIEWER:** Read conversations only
- **MEMBER+:** Send messages and escalate
- **MANAGER+:** Change AI mode (AutoPilot/CoPilot)

### 🏠 Listings
- **VIEWER/MEMBER:** View listings only
- **MANAGER+:** Sync listings, configure AI per listing, upload documents
- **ADMIN+:** Delete listing documents

### 📅 Reservations/Bookings
- **VIEWER/MEMBER:** View reservations only
- **MANAGER+:** Sync reservations from PMS

### 💰 Upsells
- **VIEWER/MEMBER:** View upsell statistics
- **MANAGER+:** Create, configure, enable/disable upsell rules
- **ADMIN+:** Delete upsell configurations

### ⚙️ Settings

#### General Settings
- **ADMIN+:** Update workspace name, logo, preferences

#### Billing & Subscription
- **OWNER ONLY:** Manage subscription, view invoices, update payment method

#### Integrations/PMS
- **ADMIN+:** Connect Hostaway, Guesty, Lodgify
- **MANAGER+:** Trigger data sync

#### AI Settings
- **MANAGER+:** Configure tone, conciseness, upload knowledge base
- **ADMIN+:** Delete AI configurations

#### Manage Members
- **ADMIN+:** Add, remove, change roles of team members
- **OWNER:** Can assign OWNER role to someone else

#### Notifications
- **MEMBER+:** Configure their own notification preferences
- **VIEWER:** Cannot change notification settings

---

## Detailed Permission Matrix

### Workspace Management

| What You Can Do | Owner | Admin | Manager | Member | Viewer |
|-----------------|:-----:|:-----:|:-------:|:------:|:------:|
| View workspace details | ✅ | ✅ | ✅ | ✅ | ✅ |
| Update workspace settings | ✅ | ✅ | ❌ | ❌ | ❌ |
| View billing information | ✅ | ❌ | ❌ | ❌ | ❌ |
| Manage subscription | ✅ | ❌ | ❌ | ❌ | ❌ |
| Delete workspace | ✅ | ❌ | ❌ | ❌ | ❌ |

### Team Management

| What You Can Do | Owner | Admin | Manager | Member | Viewer |
|-----------------|:-----:|:-----:|:-------:|:------:|:------:|
| View team members list | ✅ | ✅ | ✅ | ✅ | ✅ |
| Invite new team members | ✅ | ✅ | ❌ | ❌ | ❌ |
| Change member roles | ✅ | ✅ | ❌ | ❌ | ❌ |
| Remove team members | ✅ | ✅ | ❌ | ❌ | ❌ |
| Assign OWNER role | ✅ | ❌ | ❌ | ❌ | ❌ |

### Listings

| What You Can Do | Owner | Admin | Manager | Member | Viewer |
|-----------------|:-----:|:-----:|:-------:|:------:|:------:|
| View listings | ✅ | ✅ | ✅ | ✅ | ✅ |
| Sync listings from PMS | ✅ | ✅ | ✅ | ❌ | ❌ |
| Configure listing AI settings | ✅ | ✅ | ✅ | ❌ | ❌ |
| Upload listing documents | ✅ | ✅ | ✅ | ❌ | ❌ |
| Delete listing documents | ✅ | ✅ | ❌ | ❌ | ❌ |

### Reservations

| What You Can Do | Owner | Admin | Manager | Member | Viewer |
|-----------------|:-----:|:-----:|:-------:|:------:|:------:|
| View reservations | ✅ | ✅ | ✅ | ✅ | ✅ |
| Sync reservations from PMS | ✅ | ✅ | ✅ | ❌ | ❌ |
| Filter by date range | ✅ | ✅ | ✅ | ✅ | ✅ |

### Conversations (Inbox)

| What You Can Do | Owner | Admin | Manager | Member | Viewer |
|-----------------|:-----:|:-----:|:-------:|:------:|:------:|
| View conversations | ✅ | ✅ | ✅ | ✅ | ✅ |
| Read messages | ✅ | ✅ | ✅ | ✅ | ✅ |
| Send messages to guests | ✅ | ✅ | ✅ | ✅ | ❌ |
| Send AI-suggested responses | ✅ | ✅ | ✅ | ✅ | ❌ |
| Escalate conversations | ✅ | ✅ | ✅ | ✅ | ❌ |
| Toggle AI mode (AutoPilot/CoPilot) | ✅ | ✅ | ✅ | ❌ | ❌ |

### AI Settings

| What You Can Do | Owner | Admin | Manager | Member | Viewer |
|-----------------|:-----:|:-----:|:-------:|:------:|:------:|
| View AI settings | ✅ | ✅ | ✅ | ✅ | ✅ |
| Change AI tone/conciseness | ✅ | ✅ | ✅ | ❌ | ❌ |
| Add custom AI instructions | ✅ | ✅ | ✅ | ❌ | ❌ |
| Upload knowledge base docs | ✅ | ✅ | ✅ | ❌ | ❌ |
| Delete AI documents | ✅ | ✅ | ❌ | ❌ | ❌ |

### PMS Integrations

| What You Can Do | Owner | Admin | Manager | Member | Viewer |
|-----------------|:-----:|:-----:|:-------:|:------:|:------:|
| View connected PMS | ✅ | ✅ | ✅ | ✅ | ✅ |
| Connect new PMS (Hostaway, etc.) | ✅ | ✅ | ❌ | ❌ | ❌ |
| Update PMS credentials | ✅ | ✅ | ❌ | ❌ | ❌ |
| Disconnect PMS | ✅ | ✅ | ❌ | ❌ | ❌ |
| Trigger data sync | ✅ | ✅ | ✅ | ❌ | ❌ |

### Upsells

| What You Can Do | Owner | Admin | Manager | Member | Viewer |
|-----------------|:-----:|:-----:|:-------:|:------:|:------:|
| View upsell statistics | ✅ | ✅ | ✅ | ✅ | ✅ |
| View upsell transactions | ✅ | ✅ | ✅ | ✅ | ✅ |
| Create upsell rules | ✅ | ✅ | ✅ | ❌ | ❌ |
| Configure upsell pricing | ✅ | ✅ | ✅ | ❌ | ❌ |
| Enable/disable upsells | ✅ | ✅ | ✅ | ❌ | ❌ |
| Delete upsell rules | ✅ | ✅ | ❌ | ❌ | ❌ |

### Calendar

| What You Can Do | Owner | Admin | Manager | Member | Viewer |
|-----------------|:-----:|:-----:|:-------:|:------:|:------:|
| View calendar | ✅ | ✅ | ✅ | ✅ | ✅ |
| Sync calendar data | ✅ | ✅ | ✅ | ❌ | ❌ |

### Notifications

| What You Can Do | Owner | Admin | Manager | Member | Viewer |
|-----------------|:-----:|:-----:|:-------:|:------:|:------:|
| View notifications | ✅ | ✅ | ✅ | ✅ | ✅ |
| Mark as read | ✅ | ✅ | ✅ | ✅ | ✅ |
| Configure notification preferences | ✅ | ✅ | ✅ | ✅ | ❌ |

---

## Security & Best Practices

### ✅ DO:
- Assign the **minimum role** needed for each person's job
- Use **VIEWER** role for people who just need to monitor
- Use **MEMBER** role for front-line customer service
- Use **MANAGER** role for operations leads
- Use **ADMIN** role only for trusted partners
- Keep **OWNER** role for yourself (the account creator)

### ❌ DON'T:
- Give everyone ADMIN access "just in case"
- Share your OWNER credentials with anyone
- Assign MANAGER role to temporary staff
- Give billing access unless absolutely necessary

### 🔐 Security Notes:
- Team members can only access their assigned workspace
- You can remove team members at any time
- Removing someone instantly revokes all their access
- All actions are logged (coming soon: audit trail)

---

## Frequently Asked Questions

### Can I change someone's role later?
**Yes!** OWNER and ADMIN can change any team member's role at any time from Settings → Manage Members.

### What happens if I remove a team member?
They immediately lose access to your workspace and can no longer log in.

### Can someone be on multiple workspaces?
**No.** Each team member can only access the workspace they're invited to. If they need access to multiple properties, invite them to each workspace separately.

### Can I have multiple OWNERs?
**Yes!** The current OWNER can transfer or share the OWNER role with someone else. Both will have full access.

### What if someone forgets their password?
They can reset it using the "Forgot Password" link on the login page. This doesn't affect their role or permissions.

### Can VIEWERs see billing information?
**No.** Only the OWNER can view billing and subscription details.

---

## Quick Reference Table

**"What's the minimum role I need to..."**

| To Do This... | You Need At Least |
|---------------|-------------------|
| View anything | VIEWER |
| Send messages to guests | MEMBER |
| Escalate conversations | MEMBER |
| Sync data from PMS | MANAGER |
| Configure AI settings | MANAGER |
| Upload knowledge base | MANAGER |
| Create upsell rules | MANAGER |
| Switch AI modes | MANAGER |
| Add team members | ADMIN |
| Connect PMS platforms | ADMIN |
| Delete documents/configs | ADMIN |
| Manage billing | OWNER |
| Delete workspace | OWNER |

---