# Technical Architecture Document: SDUI & Dynamic White-Label Ecosystem

**Author:** Chowdhury Md Rajib Sarwar

**Role Focus:** Senior Software Engineer (SE5) – Client Foundations

**Architecture Focus:** SDUI • Centralized Configuration • Dynamic Assets • Native Component Registry

## 1. Executive Summary

To support a rapidly scaling ecosystem of 200+ enterprise client applications, I architected a unified Server-Driven UI (SDUI) and centralized configuration platform to deprecate a highly fragmented, hard-coded legacy system. By consolidating 200+ static `AppConfig` files into a single dynamic configuration service (`CustApp`), automating asset ingestion to Amazon S3, and introducing dynamic SDUI templating, this architecture reduced new client onboarding from 18 manual engineering steps down to 5. The system successfully decouples client rendering from App Store deployments, empowering cross-functional teams to update UI and configurations on the fly with zero mobile code changes.

## 2. High-Level Architecture

The ecosystem relies on a composite key (`sysComp` and `compId`) to dynamically resolve APIs, feature flags, and UI assets at runtime, allowing a single iOS binary architecture to serve hundreds of distinct enterprise brands.

```text
┌────────────────────────┐      ┌─────────────────────────┐      ┌────────────────────────┐
│ UI/UX Design Team      │      │ Automated Sync API      │      │ Unified Amazon S3      │
│ (Dropbox Shared Folder)├─────►│ (Webhook / Poller)      ├─────►│ /common/...            │
└────────────────────────┘      └─────────────────────────┘      │ /{sysComp}/{compId}/...│
                                                                 └──────────┬─────────────┘
┌────────────────────────┐      ┌─────────────────────────┐                 │ (Assets)
│ Backend Systems        │      │ CustApp Config Service  │                 │
│ (Tenant DBs)           ├─────►│ (JSON Schema / URLs)    │                 │
└────────────────────────┘      └──────────┬──────────────┘                 │
                                           │ (API URLs, SDUI Schema)        │
                                           ▼                                ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                iOS CLIENT APPLICATION                                   │
│                                                                                         │
│  ┌──────────────────────┐      ┌────────────────────────┐      ┌─────────────────────┐  │
│  │ Data Access Layer    │◄─────┤ SDUI Rendering Engine  ├─────►│ Component Registry  │  │
│  │ (sysComp / compId)   │      │ (Dynamic Templates)    │      │ (Swift / MVVM)      │  │
│  └──────────────────────┘      └────────────────────────┘      └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────────┘

```

## 3. Pillar 1: Centralized Configuration Management (`CustApp`)

**The Challenge:** Environment configuration was tightly coupled to Xcode targets. The iOS application maintained over 200 distinct `AppConfig` files. Any change to a tenant's base URL, feature flag, or API endpoint required a manual code update, compilation, and a full App Store review cycle.

**The Solution:** I architected and deployed `CustApp`, a centralized configuration service. I eliminated the 200+ static files, replacing them with a single dynamic data-access layer. Upon initialization, the client securely passes its `sysComp` and `compId` to `CustApp`, which returns a localized JSON payload containing all necessary routing URLs and tenant-specific toggles.

**Business Impact:** Configurations can now be overridden on the fly directly from the backend database. Critical API migrations or endpoint updates are executed instantly in production without requiring new app deployments.

## 4. Pillar 2: Dynamic Asset Management Pipeline

**The Challenge:** Application assets were historically siloed across individual company colocation servers, making routine logo and visual updates heavily dependent on engineering intervention.

**The Solution:** I designed an automated pipeline integrating directly with the UI/UX team's existing Dropbox workflow. A custom backend API synchronizes these design directories into a unified Amazon S3 infrastructure. The iOS data-access layer resolves these assets hierarchically: checking for company-unique overrides at `/{sysComp}/{compId}/`, and seamlessly falling back to `/common/` if absent.

**Business Impact:** This decoupled engineering from routine asset updates, empowering design teams to push brand changes globally in minutes while sharing common assets effectively.

## 5. Pillar 3: Dynamic Templates & Server-Driven UI (SDUI)

**The Challenge:** Maintaining 200+ target-specific UI layouts resulted in massive technical debt, duplicated implementation effort, and UI inconsistency.

**The Solution:** I modernized the legacy Objective-C monolith into a unified Swift MVVM architecture driven by an SDUI templating engine. Rather than hardcoding UI views, the client utilizes a dynamic template governed by the JSON schema fetched from `CustApp`. The SDUI engine parses the schema tree, instantiates mapped native components from a highly tested Swift/SwiftUI Component Registry, and dynamically injects the S3 assets.

**Architectural Boundary:** The server dictates composition and configuration; the client retains ownership of a finite set of native rendering primitives. This balances runtime flexibility with client safety and performance.

## 6. Interview-Level Design Considerations

To ensure the platform scales robustly, the architecture accounts for critical edge cases inherent to server-driven mobile systems:

* **Schema Evolution:** Schemas are versioned to define backward-compatible behavior, ensuring older iOS binaries can safely consume newer server payloads without catastrophic failure.
* **Unknown Components:** Unsupported component types are treated as a recoverable condition. The engine utilizes explicit fallback behaviors rather than allowing malformed backend configurations to crash the native application.
* **Cache Safety:** Configuration and asset metadata are aggressively cached locally. The client renders from the last known-good configuration, validates new payloads before persistence, and retains a rollback path for incompatible updates.
* **Accessibility:** Centralizing UI into a Native Component Registry creates a single enforcement point for Dynamic Type, VoiceOver semantics, and minimum interaction targets, ensuring all 200+ apps remain fully accessible.

## 7. Engineering Leadership & Judgment

This architecture reflects a practical approach to platform investment: centralize the areas where duplication creates compounding maintenance cost, but avoid building infrastructure that does not solve a real organizational problem. The decision to integrate with the design team's existing Dropbox workflow instead of creating a proprietary asset CMS is an example of deliberately reducing engineering scope while drastically improving operational autonomy.

By shifting the system from application-specific customization toward shared client capabilities driven by runtime configuration, the architecture maximizes engineering leverage. A platform-level capability or performance fix can now serve hundreds of clients instantly, while tenant-specific behavior remains purely data-driven and isolated.
