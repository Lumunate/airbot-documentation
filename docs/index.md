---
layout: home
title: Home
nav_order: 1
description: "Comprehensive technical documentation for the Airbot vacation rental management platform"
permalink: /
---

# Airbot Documentation

Welcome to the Airbot platform documentation. This comprehensive guide covers the complete technical architecture, business logic, and implementation details of the Airbot vacation rental management system.

## About Airbot

Airbot is an AI-powered vacation rental management platform that helps property managers automate guest communications, manage listings across multiple PMS platforms, and drive additional revenue through intelligent upsell offers.

---

## 📚 All Documentation Pages

| # | Page | Description |
|---|------|-------------|
| 1 | [**Onboarding & Setup** →](01-onboarding-and-setup.md) | User registration, subscriptions, and platform setup |
| 2 | [**Access Control & Permissions** →](02-access-control.md) | Role-based access control (RBAC) system |
| 3 | [**AI Features & Agent System** →](03-ai-features.md) | AI configuration, agents, and knowledge base |
| 4 | [**Upsell System Analysis** →](04-upsell-system-analysis.md) | Automated upsell triggers and scheduling |
| 5 | [**Business Logic** →](05-business-logic.md) | Core business flows and revenue model |
| 6 | [**Feature Implementation Analysis** →](06-feature-implementation-analysis.md) | Cross-codebase implementation status |

---

## Documentation Index

### [1. Onboarding & Setup →](01-onboarding-and-setup.md)
Complete guide to user registration, subscription management, workspace creation, and platform setup. Covers the end-to-end onboarding flow from initial signup to full platform utilization.

**Topics covered:**
- User registration and authentication
- Subscription and payment processing
- Workspace and team setup
- PMS integration onboarding

---

### [2. Access Control & Permissions →](02-access-control.md)
Detailed explanation of the role-based access control (RBAC) system. Understand the five user roles (OWNER, ADMIN, MANAGER, MEMBER, VIEWER) and their permissions across different platform features.

**Topics covered:**
- Role hierarchy and permissions
- Team member management
- Permission enforcement
- Access control implementation

---

### [3. AI Features & Agent System →](03-ai-features.md)
Comprehensive documentation of Airbot's AI system, including configuration options, agent behavior, and knowledge base management.

**Topics covered:**
- Workspace and listing-level AI configuration
- AI agents and conversation handling
- Knowledge base and document processing
- Agent personality and behavior customization

---

### [4. Upsell System Analysis →](04-upsell-system-analysis.md)
Technical deep-dive into the automated upsell system. Covers trigger conditions, scheduling logic, edge case handling, and real-world scenario testing.

**Topics covered:**
- Six upsell types and their triggers
- BullMQ job scheduling and queue management
- AI-powered prevention system
- PMS integration for upsells
- Calendar validation and availability checks

---

### [5. Business Logic →](05-business-logic.md)
Analysis of core business flows including authentication, payment processing, PMS integration, and the upsells revenue model.

**Topics covered:**
- Auth → Payment flow
- PMS integration architecture
- Subscription pricing models
- Revenue generation logic

---

### [6. Feature Implementation Analysis →](06-feature-implementation-analysis.md)
Cross-codebase analysis identifying implementation status of features across Frontend, Backend, and AI services.

**Topics covered:**
- Authentication and onboarding features
- Workspace and team management
- Listing and conversation features
- AI configuration and upsells
- Implementation gaps and recommendations

---

## Quick Links

- **Source Code Repository**: [airbot-backend](../)
- **Key Technologies**: Node.js, Express, Prisma, BullMQ, Redis, Stripe, OpenAI
- **Supported PMS Platforms**: Hostaway, Guesty, Lodgify

## Getting Started

If you're new to the Airbot platform:
1. Start with [Onboarding & Setup →](01-onboarding-and-setup.md) to understand the user journey
2. Review [Access Control →](02-access-control.md) to understand permissions
3. Deep-dive into [AI Features →](03-ai-features.md) for the core platform capabilities
4. Explore [Upsell System →](04-upsell-system-analysis.md) for revenue generation features

---

*Last Updated: January 2026*