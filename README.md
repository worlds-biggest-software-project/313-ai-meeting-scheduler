# AI Meeting Scheduler

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native meeting scheduler that learns user preferences and coordinates calendars without the friction of incumbent tools.

The AI Meeting Scheduler is a self-hostable scheduling platform for individuals, distributed teams, and executive assistants who need intelligent calendar coordination across Google Calendar, Outlook, and CalDAV-compatible systems. It combines link-based booking, focus-time protection, and preference-learning AI into a single tool, replacing the patchwork of proprietary SaaS products that dominate the category today.

---

## Why AI Meeting Scheduler?

- **Calendly is rule-based, not preference-learning.** It dominates booking flows but offers no AI personalisation, no autonomous outbound scheduling, and gates webhooks behind paid plans.
- **Motion and Reclaim.ai are proprietary and feature-dense.** Both are powerful but require trust in opaque rescheduling logic, suffer from steep onboarding, and lock users into closed SaaS ($10–$34/month).
- **Clara and Lindy are inaccessible.** The autonomous email-agent category is gated by enterprise pricing or usage-based models that penalise high-volume users.
- **Cal.com is the only major open-source option** but its AI features (Cal.ai) are still maturing relative to Motion and Reclaim, leaving room for an AI-native open-source alternative.
- **Cross-organisation agent negotiation does not exist.** No tool today automates scheduling between two AI agents at different companies without a shared platform.

---

## Key Features

### Calendar Sync and Booking

- Real-time free/busy detection across Google Calendar, Microsoft Outlook, Apple Calendar, and CalDAV feeds
- Shareable scheduling links with customisable availability windows and event types
- iCalendar / .ics invite generation and delivery
- Timezone detection and multi-timezone display
- Round-robin and collective event types for team scheduling

### AI Scheduling Intelligence

- Preference-learning engine that infers optimal meeting windows from accepted and declined meeting history
- AI slot suggestion based on stated preferences (time-of-day, buffer time, meeting type)
- Focus-block protection that detects and defends deep-work periods automatically
- Dynamic rescheduling when deadlines or priorities shift in connected project tools

### Team and Workflow Integration

- Routing forms with conditional logic for lead distribution
- Task integration with tools such as Linear, Todoist, Asana, ClickUp, and Jira
- Webhook API for third-party integrations
- Basic team analytics covering meeting load and focus-time ratio
- Email reminders, confirmations, and post-meeting follow-up

### Autonomous Agent Capabilities (Backlog)

- Email-based scheduling agent that handles back-and-forth negotiation via CC, in the style of Lindy and Clara
- Multi-agent cross-organisation negotiation via MCP or open protocol
- Burnout and workload monitoring with auto-declination suggestions
- Context-aware meeting preparation that attaches relevant documents and prior notes to events

---

## AI-Native Advantage

Where incumbents apply rule-based logic (Calendly) or opaque heuristics (Motion, Reclaim), this project centres on transparent preference learning, multi-party negotiation between AI agents, contextual meeting preparation, and burnout-aware scheduling. AI infers ideal windows, buffer times, and energy patterns from historical calendar behaviour without explicit configuration, and surfaces explanations that users can audit and override in natural language.

---

## Tech Stack & Deployment

The project targets self-hosted and cloud deployment modes, with full data ownership for self-hosters. It is built on open standards: CalDAV (RFC 4791) for calendar sync, iCalendar (RFC 5545) for event exchange, OAuth 2.0 for delegated calendar access, and Microsoft Graph API for Outlook/Exchange integration. A REST API with OpenAPI specification, webhook subscriptions, and an embeddable booking widget mirror the integration surface offered by Cal.com and Calendly.

---

## Market Context

The AI calendar and scheduling market was valued at $21.42 billion in 2025 and is projected to reach $27.8 billion in 2026 at a 29.8% CAGR, with forecasts of $78.14 billion by 2030 (Research and Markets, 2026). Pro plans cluster at $10–$20/month, team plans at $15–$35/user/month, and autonomous agents like Clara command enterprise pricing above $100/month. Primary buyers are executives and consultants protecting deep-work time, sales teams running high-volume prospect calls, distributed engineering teams across time zones, and HR/recruiting coordinators managing interview loops.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
