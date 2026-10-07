# 💍 Instabodas.es — Architectural Case Study & DevSecOps Showcase

![Next.js](https://img.shields.io/badge/Next.js_16.2-000000?style=for-the-badge&logo=nextdotjs&logoColor=white)
![React](https://img.shields.io/badge/React_19-61DAFB?style=for-the-badge&logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript_5.4-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase_PostgreSQL-3FCF8E?style=for-the-badge&logo=supabase&logoColor=white)
![Cloudflare R2](https://img.shields.io/badge/Cloudflare_R2-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)
![Stripe API](https://img.shields.io/badge/Stripe_API-635BFF?style=for-the-badge&logo=stripe&logoColor=white)
![Tailwind CSS v4](https://img.shields.io/badge/Tailwind_CSS_v4-06B6D4?style=for-the-badge&logo=tailwindcss&logoColor=white)
![DevSecOps](https://img.shields.io/badge/Security-DevSecOps_%26_RLS-red?style=for-the-badge&logo=shield)

> **Executive Summary**:  
> **Instabodas.es** is a high-availability, multi-tenant event media SaaS designed for real-time guest photo streaming, live TV projection, and full-lifecycle wedding planning. Built on a serverless event-driven architecture, it combines zero-egress cloud media storage with strict PostgreSQL Row-Level Security (RLS) and automated data lifecycle governance.

🌐 **Production Platform**: [instabodas.es](https://instabodas.es)

---

## 🎯 Technical Challenges & Core Objectives

Designing a digital platform for high-density live events presents unique engineering constraints: burst traffic spikes, zero guest friction, dynamic screen synchronization, and strict privacy regulations.

### 1. High-Burst Concurrency & Sub-Second Latency
* **Challenge**: Weddings create sudden traffic bursts (100–300+ guests uploading high-resolution media concurrently within a 3–5 hour window). Upload delays or lag on live TV projectors degrade user experience.
* **Architecture Solution**: Client-side media compression and metadata pre-validation paired with asynchronous chunked uploads directly to S3-compatible cloud storage. Live TV projection consumes low-overhead Supabase Realtime WebSocket notifications (`postgres_changes`), pushing new photo alerts to screens instantly without HTTP polling.

### 2. Multi-Tenant Isolation & Zero-Friction Privacy
* **Challenge**: Guests must upload photos and write guestbook messages via QR codes without friction (no app downloads or account registration), while ensuring complete multi-tenant data isolation and protection against malicious endpoint scanning.
* **Architecture Solution**: Cryptographic tenant routing using non-enumerable UUID v4 identifiers (`/boda/[weddingId]`). The persistence layer enforces strict PostgreSQL Row-Level Security (RLS) policies. Role-Based Access Control (RBAC) isolates couple administration, guest submissions, delegated moderator scopes (`?mod=[weddingId]`), and superadmin CRM operations.

### 3. Cost-Effective Infrastructure & Automated Data Governance
* **Challenge**: Storing gigabytes of uncompressed high-res images and zip archives per event risks rapid cloud egress cost inflation and long-term data retention liabilities under EU regulations.
* **Architecture Solution**: Decoupled cloud architecture using Next.js Serverless Route Handlers on Vercel Edge, relational persistence on Supabase PostgreSQL, and media storage on **Cloudflare R2** (eliminating AWS S3 egress bandwidth fees). An automated 2-stage Cron lifecycle engine handles automated email alerts, database purges at 30 days, and physical R2 object deletion at 37 days.

---

## 🏗️ System Architecture Diagram

The system operates across three decoupled layers: **Client & Edge Interface**, **Application & API Core**, and **Persistence & Storage Services**.

```mermaid
graph TD
    subgraph ClientLayer ["1. Client & Interface Layer"]
        A["📱 Guest Mobile Portal<br/>(PWA / QR Camera Upload)"]
        B["📺 Live TV Wall Projector<br/>(Supabase Realtime WebSockets)"]
        C["💻 Couple Dashboard & Moderator<br/>(Next.js App Router + RBAC)"]
        D["🛡️ SuperAdmin CRM Console<br/>(R2 Storage Metrics & Audit)"]
    end

    subgraph EdgePerimeter ["2. Perimeter & Edge Protection"]
        E["☁️ Vercel Edge Network<br/>(SSL/TLS 1.3 + Anycast CDN)"]
        F["🔐 Auth Middleware & Whitelist<br/>(JWT Verification + Admin ACL)"]
        G["💳 Stripe Webhook Gateway<br/>(Signature Verification)"]
    end

    subgraph ApplicationBackend ["3. Application Core (Next.js Serverless)"]
        H["⚙️ Upload & Processing API<br/>(/api/upload)"]
        I["⚙️ Checkout & Upgrade Engine<br/>(/api/checkout-session)"]
        J["⏰ Automated Lifecycle Cron<br/>(/api/cron/check-expiration)"]
        K["🧹 Admin Storage Purge Engine<br/>(/api/admin/clear-r2)"]
    end

    subgraph PersistenceStorage ["4. Persistence & Infrastructure Layer"]
        L[("🗄️ Supabase PostgreSQL DB<br/>(Strict Row Level Security)")]
        M["⚡ Supabase Realtime Engine<br/>(WebSocket Channel Broadcast)"]
        N[("📦 Cloudflare R2 Storage<br/>(AWS S3 API - 0$ Egress Fees)")]
        O["📧 SMTP Email Engine<br/>(Nodemailer / DonDominio)"]
    end

    %% Data Flow Connections
    A -->|"HTTPS POST / Media Upload"| E
    B -->|"WSS Realtime Connection"| M
    C -->|"Authenticated JWT Session"| E
    D -->|"JWT + Whitelist Headers"| E

    E --> F
    F -->|"Authorized Endpoint Routing"| H
    F -->|"Authorized Endpoint Routing"| I
    F -->|"Cron Secret Token Match"| J
    F -->|"Admin Role Verification"| K
    G -->|"Webhook Payload & Verify"| I

    H -->|"PutObject Command (S3 SDK)"| N
    H -->|"Insert Photo Metadata"| L
    I -->|"Update Plan State (Basic/Premium)"| L
    J -->|"Query Expiring Events"| L
    J -->|"Dispatch Warning Emails"| O
    J -->|"Batch Delete Objects"| N
    K -->|"Cascade Storage Purge"| N

    L -->|"Trigger Postgres Change Event"| M
```

---

## 🛠️ Technical Decisions & Stack Justification

| Layer | Technology | Architectural Justification |
| :--- | :--- | :--- |
| **Framework** | **Next.js 16.2.7 (React 19)** | App Router architecture with Hybrid Server/Client Components. Fast server-side rendering for landing pages and light bundle payloads for mobile guests on limited cellular networks. |
| **Styling & UI** | **Tailwind CSS v4** | Zero-runtime CSS processing, fluid utility classes, and custom luxury theme system (`wedding-oro`, `wedding-eucalipto`, `wedding-crema`). |
| **Database & Auth** | **Supabase (PostgreSQL)** | Managed PostgreSQL with native **Row-Level Security (RLS)**, automated JWT handling, and WebSocket change data capture (`postgres_changes`) for real-time TV streaming. |
| **Media Storage** | **Cloudflare R2** | High-durability object storage using `@aws-sdk/client-s3`. Selected over AWS S3 to eliminate egress bandwidth costs entirely (0$ egress fees vs S3 transfer rates). |
| **Monetization** | **Stripe API** | Checkout Sessions, signature-verified Webhooks, and differential upgrade logic (upgrading Basic to Premium charges only the 20€ delta). Includes local sandbox simulation for unpaid tenants. |
| **Transactional Email** | **Nodemailer + SMTP** | Dedicated SMTP transport delivering lifecycle warnings, zip export notifications, and billing receipts directly from `info@instabodas.es`. |
| **QR Generation** | **QRServer Engine** | Error Correction Level H (High) enabling center logo/heart overlay without degrading QR scanning accuracy under variable lighting conditions. |

---

## 🛡️ Security Engineering & DevSecOps

### 1. Cryptographic Tenant Isolation
Every event workspace is instantiated with a non-enumerable PostgreSQL `UUID v4` generated via `gen_random_uuid()`. Because endpoints rely on 128-bit entropy, endpoint enumeration or brute-force crawling is mathematically infeasible.

### 2. Fine-Grained Row-Level Security (RLS) & RBAC
All PostgreSQL tables (`weddings`, `photos`, `invitados`, `mesas`, `gastos`) enforce mandatory RLS rules:
* **Anonymous Guest Access**: Restricted to `INSERT` and `SELECT` operations matching the specific `wedding_id` UUID payload.
* **Authenticated Couple Scope**: Full `SELECT`/`UPDATE` privileges restricted strictly to the user's authenticated UID.
* **Delegated Moderator Scope (`?mod=[weddingId]`)**: Restricts the UI layer to media moderation queues only, stripping access to settings, bank accounts, guest lists, and data export endpoints.
* **SuperAdmin Authorization**: API routes under `/api/admin/*` validate both the Supabase JWT and check the claims against a server-side administrator whitelist (`NEXT_PUBLIC_ADMIN_EMAILS`).

### 3. Automated Ephemeral Data Lifecycle (GDPR Compliance)
To prevent data leaks and maintain strict storage efficiency, the system implements an automated 30-day post-event data expiration pipeline:

```mermaid
flowchart LR
    A["🎉 Event Date (Day 0)"] --> B["📩 Day +23: Email Alert 1<br/>(ZIP Download Link)"]
    B --> C["⏰ Day +28: Urgent Alert 2<br/>(48h Expiration Notice)"]
    C --> D["🗄️ Day +30: Database Purge<br/>(Delete Records from DB)"]
    D --> E["📦 Day +37: R2 Physical Purge<br/>(S3 DeleteFolder Cascade)"]
```

* Executed daily via Vercel Cron (`GET /api/cron/check-expiration`).
* Requests must present a high-entropy secret token matching `CRON_SECRET`.
* Demo environments (e.g., `BodaLuciaAaron`) are explicitly exempt from automated purging.

### 4. Legal Compliance & Audit Logging (`consent_logs`)
Under the EU Consumer Rights Directive, immediate access to digital media services requires an explicit waiver of the standard 14-day right of withdrawal. The onboarding flow records an immutable audit entry in `consent_logs` storing user IP address, timestamp, terms version, and explicit checkbox consent state prior to Stripe checkout execution.

---

## 📋 Legal & Intellectual Property Notice

> **Notice**: This public showcase repository is published exclusively as a **technical architecture case study** to demonstrate software engineering standards, DevSecOps principles, and system design patterns.
> 
> The proprietary source code, internal business logic, production configuration secrets, and database schemas remain private property. All rights reserved under applicable copyright and commercial protection laws.

---

## 🏷️ Repository Metadata (For Technical Recruiters & SEO)

* **GitHub About Description**:
  `High-availability, privacy-first event media SaaS architecture case study with real-time WebSockets, R2 storage & DevSecOps.`
* **Topics / Tags**:
  `nextjs`, `react19`, `typescript`, `supabase`, `postgresql`, `cloudflare-r2`, `stripe-api`, `devsecops`, `software-architecture`, `realtime`, `system-design`, `gdpr-compliance`, `tailwind-css`, `saas-architecture`, `case-study`
