# Standards & API Reference

> Project: AI Meeting Scheduler · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

No ISO standards directly govern internet calendar data exchange; ISO defers to IETF/CalConnect specifications for the calendaring domain. The following ISO standards are adjacent and relevant:

- **ISO/IEC 27001** — Information security management system standard. Relevant for handling calendar data containing PII (attendee names, email addresses, meeting subjects). Compliance demonstrates trustworthy data stewardship to enterprise buyers.
  URL: https://www.iso.org/standard/27001

- **ISO 8601** — Date and time format standard (e.g. `2026-05-03T14:00:00Z`). Mandated by RFC 5545 for datetime values inside iCalendar objects; also used natively in JSON API payloads.
  URL: https://www.iso.org/standard/70907.html

---

### W3C & IETF Standards

- **RFC 5545 — iCalendar** (Internet Calendaring and Scheduling Core Object Specification)
  The foundational data format for calendar objects. Defines VCALENDAR, VEVENT, VTODO, VJOURNAL, VFREEBUSY, and VTIMEZONE components and all their properties. Every calendar platform ingests and emits .ics files conforming to this spec.
  URL: https://www.rfc-editor.org/rfc/rfc5545

- **RFC 4791 — CalDAV** (Calendaring Extensions to WebDAV)
  Defines the CalDAV protocol — a standard way of accessing, managing, and synchronising calendar data stored on a server over HTTP/WebDAV. Enables calendar clients to read/write events on any compliant server (Google Calendar, Apple Calendar, Fastmail, Nextcloud, etc.).
  URL: https://datatracker.ietf.org/doc/html/rfc4791

- **RFC 5546 — iTIP** (iCalendar Transport-Independent Interoperability Protocol)
  Defines how iCalendar objects are exchanged to accomplish scheduling tasks: REQUEST, REPLY, CANCEL, ADD, REFRESH, COUNTER, and DECLINECOUNTER methods. Underpins how meeting invitations and RSVPs flow between calendar systems.
  URL: https://datatracker.ietf.org/doc/html/rfc5546

- **RFC 6047 — iMIP** (iCalendar Message-Based Interoperability Protocol)
  Binds iTIP to email (SMTP/MIME). Specifies how .ics attachments inside emails carry scheduling requests, allowing calendar clients to process meeting invitations received via email.
  URL: https://www.rfc-editor.org/rfc/rfc6047

- **RFC 6638 — CalDAV Scheduling Extensions**
  Extends CalDAV to support server-side scheduling: free/busy lookups (VFREEBUSY), auto-scheduling, attendee inbox/outbox, and calendar-auto-schedule feature. Required for implementations that want server-side invite delivery, not just client-side .ics file exchange.
  URL: https://icalendar.org/CalDAV-Scheduling-RFC-6638/

- **RFC 6578 — CalDAV Collection Synchronization**
  Efficient delta synchronisation for CalDAV collections. Allows clients to fetch only changed events since the last sync rather than re-fetching all events — important for performance at scale.
  URL: https://datatracker.ietf.org/doc/html/rfc6578

- **RFC 7953 — Calendar Availability (VAVAILABILITY)**
  Extends iCalendar with a VAVAILABILITY component that expresses when a calendar user is available for scheduling. Enables richer availability rules beyond simple free/busy blocks.
  URL: https://www.rfc-editor.org/rfc/rfc7953

- **RFC 7240 — Prefer Header for HTTP**
  Widely used in CalDAV implementations to express client preferences (e.g. `Prefer: return=minimal`) to reduce response payload size.
  URL: https://datatracker.ietf.org/doc/html/rfc7240

---

### Data Model & API Specifications

- **OpenAPI 3.1**
  The de-facto standard for documenting REST APIs. Calendly, Cal.com, Reclaim, and Motion all publish or reference OpenAPI/Swagger specs for their scheduling APIs. An AI meeting scheduler should publish an OpenAPI 3.1 spec to enable SDK auto-generation, Postman collections, and third-party integration discovery.
  URL: https://spec.openapis.org/oas/v3.1.0

- **JSON Schema (draft-07 / 2020-12)**
  Used for validating API request and response payloads. Scheduling tools use JSON Schema to validate event create/update payloads, routing form definitions, and webhook event bodies.
  URL: https://json-schema.org

- **iCalendar.org Validator**
  Provides automated RFC 5545 compliance checking for .ics feeds and calendar objects. Useful for CI/CD validation of generated calendar data.
  URL: https://icalendar.org/validator.html

- **Model Context Protocol (MCP)**
  Anthropic's open protocol for connecting AI agents to external tools and data sources. Multiple MCP servers for calendar platforms already exist (Google Calendar MCP, Cal.com MCP, Calendly MCP, Outlook Calendar MCP, Chronos MCP for CalDAV). An AI-native meeting scheduler should expose an MCP server to make its scheduling capabilities available to any MCP-compatible AI assistant.
  URL: https://modelcontextprotocol.io

---

### Security & Authentication Standards

- **OAuth 2.0 (RFC 6749)**
  Mandatory for delegated access to Google Calendar, Outlook Calendar, and Apple Calendar APIs. All three providers require OAuth 2.0; basic authentication is not supported. Token scopes must be minimised (e.g. `https://www.googleapis.com/auth/calendar.events` rather than full calendar write).
  URL: https://datatracker.ietf.org/doc/html/rfc6749

- **OAuth 2.1 (draft)**
  Calendly's API v2 uses OAuth 2.1. Consolidates best practices from RFC 6749 and removes implicit flow and password grant. Recommended for new scheduling API implementations.
  URL: https://oauth.net/2.1/

- **OpenID Connect 1.0**
  Built on OAuth 2.0; used for user identity verification. Standard scopes: `openid`, `profile`, `email`. Relevant when scheduling tools authenticate users via Google, Microsoft, or Apple SSO.
  URL: https://openid.net/connect/

- **PKCE (RFC 7636 — Proof Key for Code Exchange)**
  Required by OAuth 2.1 for all public clients. Prevents authorisation-code interception attacks in mobile and SPA scheduling clients.
  URL: https://datatracker.ietf.org/doc/html/rfc7636

- **GDPR (EU 2016/679)**
  Scheduling tools process personal data (attendee names, email addresses, meeting topics, availability patterns). GDPR compliance requires: explicit consent or legitimate interest as legal basis, data minimisation, right-to-erasure on user deletion, Data Processing Agreements with calendar providers, and self-service data export in interoperable formats (CSV, JSON, ICS).
  Reference: https://gdpr-info.eu

- **HIPAA (US)**
  Enterprise scheduling in healthcare contexts must comply with HIPAA if the meeting metadata includes protected health information (PHI). Cal.com offers a HIPAA-compliant tier; any self-hosted deployment must implement appropriate safeguards.
  Reference: https://www.hhs.gov/hipaa

- **SOC 2 Type II**
  The enterprise sales standard for SaaS scheduling tools. Buyers in finance, healthcare, and large enterprises routinely require SOC 2 Type II audit reports. Cal.com and Calendly both hold SOC 2 certifications.
  Reference: https://www.aicpa.org/soc2

- **TLS 1.3 (RFC 8446)**
  All calendar data in transit must be encrypted with TLS 1.3. Calendly's documented baseline is TLS + AES-256 RSA encryption for data in transit and at rest.
  URL: https://datatracker.ietf.org/doc/html/rfc8446

---

### MCP Server Specifications

The Model Context Protocol (MCP) is now widely adopted for connecting AI assistants to calendar services. The following MCP server implementations are available as reference implementations:

- **google-calendar-mcp** — Multi-account, multi-calendar Google Calendar integration with event management, free/busy queries, and natural language scheduling.
  URL: https://github.com/nspady/google-calendar-mcp

- **Cal.com Calendar MCP Server** — Integrates with Cal.com API for appointment scheduling, management, and listing.
  URL: https://www.pulsemcp.com/servers/calcom-calendar

- **Outlook Calendar MCP Server** — Microsoft Outlook Calendar integration for event management, scheduling, and attendee status updates.
  URL: https://www.pulsemcp.com/servers/merajmehrabi-outlook-calendar

- **Chronos MCP** — CalDAV-native MCP server built with FastMCP 2.0; supports multi-account CalDAV calendar management and advanced event handling.
  URL: https://github.com/democratize-technology/chronos-mcp

- **Google Calendar MCP (Official, Developer Preview)** — Google's officially supported MCP server for Google Calendar, enabling AI agents to read schedules and create/update/delete events.
  URL: https://developers.google.com/workspace/calendar

---

## Similar Products — Developer Documentation & APIs

### Google Calendar API

- **Description:** RESTful API exposing the full Google Calendar feature set. Primary calendar integration target for any scheduling tool, given Google Workspace's dominance in both consumer and enterprise markets.
- **API Documentation:** https://developers.google.com/workspace/calendar/api/guides/overview
- **API Reference (v3):** https://developers.google.com/workspace/calendar/api/v3/reference
- **SDKs/Libraries:** Java, Python, PHP, .NET, Ruby, Node.js (googleapis npm package)
  - Node.js: https://googleapis.dev/nodejs/googleapis/latest/calendar/
- **Developer Guide:** https://developers.google.com/workspace/calendar
- **Standards:** REST/JSON, OAuth 2.0 with incremental scopes, RFC 5545 iCalendar for import/export
- **Authentication:** OAuth 2.0 (web server, installed app, service account flows); scopes include `calendar.readonly`, `calendar.events`, `calendar`
- **Key Scheduling Endpoints:** Events.list, Events.insert, Events.quickAdd, Freebusy.query

---

### Microsoft Graph Calendar API

- **Description:** Microsoft's unified API for Outlook and Exchange calendar data across Microsoft 365 and personal accounts. Required for enterprise scheduling in Microsoft-centric organisations.
- **API Documentation:** https://learn.microsoft.com/en-us/graph/outlook-calendar-concept-overview
- **API Reference:** https://learn.microsoft.com/en-us/graph/api/resources/calendar?view=graph-rest-1.0
- **SDKs/Libraries:** JavaScript/TypeScript, Python, Java, .NET, Go, PHP
  - JS SDK: https://github.com/microsoftgraph/msgraph-sdk-javascript
- **Developer Guide:** https://learn.microsoft.com/en-us/graph/api/resources/calendar-overview
- **Standards:** REST/JSON, OData v4.0, OAuth 2.0 (MSAL), OpenID Connect
- **Authentication:** OAuth 2.0 via Microsoft Identity Platform (MSAL); scopes include `Calendars.Read`, `Calendars.ReadWrite`
- **Key Scheduling Endpoints:** calendar.getSchedule (free/busy), user.findMeetingTimes (meeting suggestions), calendar/events (CRUD)

---

### Calendly API v2

- **Description:** REST API for programmatic scheduling automation. Enables reading scheduled events, managing event types, handling cancellations, and embedding the Calendly booking flow in external applications.
- **API Documentation:** https://developer.calendly.com/api-docs
- **Developer Portal:** https://developer.calendly.com/
- **Getting Started:** https://developer.calendly.com/getting-started
- **Webhook Documentation:** https://developer.calendly.com/receive-data-from-scheduled-events-in-real-time-with-webhook-subscriptions
- **SDKs/Libraries:** No official SDK; community wrappers exist for JavaScript and Python
- **Standards:** REST/JSON, OAuth 2.1, personal access tokens
- **Authentication:** Personal Access Tokens for single-user apps; OAuth 2.1 for multi-user integrations
- **Webhook Events:** `invitee.created`, `invitee.canceled`, `routing_form_submission.created`

---

### Cal.com API v2

- **Description:** API-first, open-source-backed scheduling infrastructure. Designed for developers embedding scheduling into their own products. Provides full access to the Cal.com scheduling engine.
- **API Documentation:** https://cal.com/docs/api-reference/v2/introduction
- **Documentation Hub:** https://cal.com/docs
- **SDKs/Libraries:** Official JavaScript/TypeScript SDK (in development); community Python wrappers
- **Developer Guide:** https://cal.com/docs/developing/introduction
- **Standards:** REST/JSON, OAuth 2.0, OpenAPI/Swagger spec, GraphQL playground available
- **Authentication:** OAuth 2.0; API key for personal use
- **Notable:** Fully self-hostable; same API available on cloud and self-hosted instances

---

### Reclaim.ai API

- **Description:** API for the AI scheduling and habits platform. Enables programmatic management of tasks, habits, scheduling links, and calendar events. Less well-documented than Calendly or Cal.com but growing.
- **API Documentation:** https://help.reclaim.ai/en (Developer section)
- **OpenAPI Spec:** Available (linked from developer docs)
- **SDKs/Libraries:** No official SDK; community Python SDK (unofficial, reverse-engineered): https://github.com/llabusch93/reclaim-sdk
- **Standards:** REST/JSON, OpenAPI/Swagger spec, webhook management, sandbox environment
- **Authentication:** OAuth 2.0

---

### Motion API

- **Description:** REST API for the AI task and calendar scheduling platform. Enables external creation and management of tasks, projects, and schedule queries.
- **API Documentation:** https://docs.usemotion.com
- **Getting Started:** https://help.usemotion.com/integrations/integrations-101/api-docs
- **SDKs/Libraries:** Community Python library (python-motion): https://github.com/rhymiz/python-motion; Unified.to meta-integration
- **Standards:** REST/JSON; API key authentication
- **Authentication:** API keys
- **Notable:** No webhook API documented in the official docs as of May 2026; Zapier is the recommended no-code integration path

---

### Apple EventKit / CalDAV

- **Description:** Apple's calendar and reminders framework for iOS/macOS. EventKit provides native on-device calendar access; Apple Calendar also exposes a CalDAV interface for server-side sync.
- **API Documentation (EventKit):** https://developer.apple.com/documentation/eventkit
- **CalDAV Server:** iCloud CalDAV endpoint — `caldav.icloud.com` (requires app-specific password)
- **Standards:** CalDAV (RFC 4791), iCalendar (RFC 5545), OAuth 2.0 (for iCloud API access)
- **Authentication:** App-specific passwords or Sign in with Apple OAuth for iCloud Calendar access

---

### Zoom Meetings API

- **Description:** Video conferencing API relevant for generating meeting join URLs when creating calendar events. Widely used by scheduling tools to embed Zoom links in invites.
- **API Documentation:** https://developers.zoom.us/docs/api/
- **SDKs/Libraries:** JavaScript, Python, Java, Go, PHP
- **Standards:** REST/JSON, OAuth 2.0, JWT (legacy)
- **Authentication:** OAuth 2.0 (Server-to-Server OAuth recommended for production)
- **Key Endpoint:** POST /users/{userId}/meetings — creates a Zoom meeting and returns the join URL for embedding in calendar events

---

## Notes

- **CalConnect** (the Calendaring and Scheduling Consortium) is the standards body that produced and maintains the CalDAV/iCalendar family of RFCs. Their developer guide at https://devguide.calconnect.org is the authoritative reference for implementors.
- **MCP ecosystem maturity**: As of May 2026, MCP servers for Google Calendar and Cal.com are functional but unofficial. Google's official MCP server is in developer preview. An AI-native scheduler exposing a well-documented MCP server would have first-mover advantage in the agentic AI ecosystem.
- **Free/busy across organisations**: The `calendar.getSchedule` endpoint on Microsoft Graph and the `freebusy.query` endpoint on Google Calendar can return availability across org boundaries if the receiving user has permission. Cross-organisation free/busy without a shared platform remains technically possible but poorly supported in most scheduling tools.
- **Emerging: A2A scheduling**: Agent-to-Agent (A2A) scheduling — where two AI assistants negotiate meeting times directly without human involvement — is conceptually possible with MCP today but no standardised A2A scheduling protocol exists yet. This is the key frontier for an AI-native open-source scheduler to define.
