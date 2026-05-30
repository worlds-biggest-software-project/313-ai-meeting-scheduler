# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: AI Meeting Scheduler · Created: 2026-05-24

## Philosophy

A meeting scheduler has a stable relational core (users own calendars, calendars contain events, event types define booking templates, bookings connect hosts to invitees at a time) surrounded by highly variable data: calendar provider metadata differs by provider (Google vs Microsoft vs CalDAV), availability rules vary by user preference complexity, AI preferences evolve as models improve, routing form configurations are inherently schema-free, and workflow templates mix trigger conditions with action parameters. A JSONB hybrid keeps the invariant scheduling pipeline relational (every booking has a host, time, status, and UID) while provider-specific, AI-generated, and configuration data lives in JSONB columns.

This is particularly effective for a scheduling platform because:
- **Calendar provider metadata varies wildly** — Google Calendar returns different sync tokens, webhook channels, and event properties than Microsoft Graph or CalDAV. JSONB stores each provider's shape without provider-specific columns.
- **AI preference models evolve** — Today's preference model uses time-of-day affinity and energy patterns; tomorrow's might add meeting-type clustering or cross-calendar context. JSONB avoids schema migrations as the AI features mature.
- **Routing forms are inherently configurable** — Form fields, conditions, and routing rules are user-defined. JSONB is the natural representation for this configuration.
- **Booking context varies by event type** — A sales demo booking carries different custom fields than a 1:1 check-in or an interview loop. JSONB answers store arbitrary question/answer pairs without a junction table.

The result is a 12-table schema that covers the full platform with dramatically simpler queries than the 22-table normalized model.

**Best for:** Rapid MVP development, teams shipping a scheduling platform quickly, and platforms where multi-provider calendar support, evolving AI features, and flexible routing configurations matter more than constraint enforcement on variable fields.

**Trade-offs:**
- **Pro:** 12 tables — dramatically simpler than the 22-table normalized model
- **Pro:** Calendar provider metadata is schema-free — adding a new provider needs no migration
- **Pro:** AI preferences and signals unified in JSONB — new AI features without schema changes
- **Pro:** Routing form configuration in a single JSONB column
- **Con:** JSONB fields lack database-level constraints
- **Con:** Provider-specific queries require JSONB extraction (less efficient than typed columns)
- **Con:** Application must validate booking answers and routing form configs
- **Con:** Complex cross-provider analytics harder on JSONB

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5545 (iCalendar) | Booking fields map to VEVENT properties; JSONB stores provider-specific extensions |
| RFC 4791 (CalDAV) | CalDAV sync state in `calendar_accounts.sync_state` JSONB |
| RFC 7953 (VAVAILABILITY) | Availability rules modeled as JSONB arrays of day/time windows |
| RFC 5546 (iTIP) | Booking lifecycle status matches iTIP methods |
| ISO 8601 | All timestamps as TIMESTAMPTZ |
| OAuth 2.0 | Provider credentials in `calendar_accounts.auth` JSONB |
| GDPR | Invitee PII in explicit columns; provider data in JSONB |

---

## Core Tables

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    name TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    settings JSONB NOT NULL DEFAULT '{}',
    -- {"locale": "en", "date_format": "MM/DD/YYYY", "time_format": "12h",
    --  "branding": {"logo_url": "...", "accent_color": "#2563eb"},
    --  "notifications": {"email_reminders": true, "sms_reminders": false},
    --  "ai_config": {"auto_suggest_slots": true, "protect_focus_time": true,
    --                "preference_learning": true, "model_version": "v2.1"}}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE calendar_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider TEXT NOT NULL CHECK (provider IN (
        'google', 'microsoft', 'apple', 'caldav'
    )),
    email_address TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'active', 'syncing', 'error', 'disconnected'
    )),
    auth JSONB NOT NULL DEFAULT '{}',
    -- Google: {"access_token_enc": "...", "refresh_token_enc": "...",
    --          "token_expires_at": "2026-05-24T12:00:00Z", "scopes": ["calendar.readonly"]}
    -- Microsoft: {"access_token_enc": "...", "refresh_token_enc": "...",
    --             "tenant_id": "...", "token_expires_at": "..."}
    -- CalDAV: {"url": "https://nextcloud.example.com/remote.php/dav/",
    --          "username": "user", "password_enc": "..."}
    sync_state JSONB NOT NULL DEFAULT '{}',
    -- Google: {"sync_token": "...", "channel_id": "...", "channel_expiry": "..."}
    -- Microsoft: {"delta_link": "...", "subscription_id": "..."}
    -- CalDAV: {"ctag": "...", "sync_token": "..."}
    calendars JSONB NOT NULL DEFAULT '[]',
    -- [{"external_id": "primary", "name": "Work", "color": "#4285f4",
    --   "is_checked_for_conflicts": true, "is_primary": true},
    --  {"external_id": "abc123", "name": "Personal", "color": "#7ae7bf",
    --   "is_checked_for_conflicts": false}]
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, provider, email_address)
);

CREATE INDEX idx_cal_accounts_user ON calendar_accounts(user_id);
```

---

## Event Types & Availability

```sql
CREATE TABLE event_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    team_id UUID REFERENCES teams(id),
    title TEXT NOT NULL,
    slug TEXT NOT NULL,
    description TEXT,
    duration_minutes INT NOT NULL DEFAULT 30,
    buffer_before_minutes INT NOT NULL DEFAULT 0,
    buffer_after_minutes INT NOT NULL DEFAULT 0,
    min_notice_hours INT NOT NULL DEFAULT 1,
    max_future_days INT NOT NULL DEFAULT 60,
    scheduling_type TEXT NOT NULL DEFAULT 'individual' CHECK (scheduling_type IN (
        'individual', 'round_robin', 'collective'
    )),
    location JSONB NOT NULL DEFAULT '{}',
    -- {"type": "video", "provider": "zoom", "value": "https://zoom.us/j/123456"}
    -- {"type": "in_person", "address": "123 Main St, Suite 400"}
    -- {"type": "phone", "phone_number": "+1-555-0123"}
    questions JSONB NOT NULL DEFAULT '[]',
    -- [{"label": "What is this meeting about?", "type": "textarea", "required": true},
    --  {"label": "Company name", "type": "text", "required": false},
    --  {"label": "Team size", "type": "select", "options": ["1-10", "11-50", "51+"], "required": true}]
    color TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    requires_confirmation BOOLEAN NOT NULL DEFAULT FALSE,
    workflows JSONB NOT NULL DEFAULT '[]',
    -- [{"trigger": "booking_created", "action": "email",
    --   "template_subject": "Meeting confirmed: {{title}}",
    --   "template_body": "Hi {{invitee_name}}, your meeting is confirmed for {{start_time}}"},
    --  {"trigger": "reminder_before", "action": "email", "offset_minutes": -15,
    --   "template_subject": "Reminder: {{title}} in 15 minutes"},
    --  {"trigger": "follow_up_after", "action": "webhook", "offset_minutes": 60,
    --   "url": "https://crm.example.com/webhook", "payload_template": {"booking_id": "{{id}}"}}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, slug)
);

CREATE INDEX idx_event_types_user ON event_types(user_id);

CREATE TABLE availability_schedules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL DEFAULT 'Working Hours',
    timezone TEXT NOT NULL DEFAULT 'UTC',
    is_default BOOLEAN NOT NULL DEFAULT FALSE,
    rules JSONB NOT NULL DEFAULT '[]',
    -- [{"day_of_week": 1, "start_time": "09:00", "end_time": "12:00"},
    --  {"day_of_week": 1, "start_time": "13:00", "end_time": "17:00"},
    --  {"day_of_week": 2, "start_time": "09:00", "end_time": "17:00"},
    --  {"day_of_week": 3, "start_time": "09:00", "end_time": "17:00"},
    --  {"day_of_week": 4, "start_time": "09:00", "end_time": "17:00"},
    --  {"day_of_week": 5, "start_time": "09:00", "end_time": "12:00"}]
    overrides JSONB NOT NULL DEFAULT '{}',
    -- {"2026-06-15": {"is_unavailable": true, "reason": "Public holiday"},
    --  "2026-06-20": {"start_time": "10:00", "end_time": "14:00", "reason": "Dentist AM"}}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_avail_user ON availability_schedules(user_id);
```

---

## Bookings

```sql
CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type_id UUID NOT NULL REFERENCES event_types(id),
    host_id UUID NOT NULL REFERENCES users(id),
    uid TEXT NOT NULL UNIQUE,
    title TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'confirmed' CHECK (status IN (
        'pending', 'confirmed', 'cancelled', 'rescheduled', 'no_show', 'completed'
    )),
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    timezone TEXT NOT NULL,
    location JSONB NOT NULL DEFAULT '{}',
    -- {"type": "video", "provider": "zoom", "meeting_url": "https://zoom.us/j/123456",
    --  "meeting_id": "123456", "passcode": "abc"}
    invitees JSONB NOT NULL DEFAULT '[]',
    -- [{"email": "jane@example.com", "name": "Jane Doe", "timezone": "Europe/London",
    --   "rsvp_status": "accepted", "cancel_url": "https://...", "reschedule_url": "https://..."}]
    answers JSONB NOT NULL DEFAULT '[]',
    -- [{"question": "What is this meeting about?", "answer": "Project review"},
    --  {"question": "Team size", "answer": "11-50"}]
    cancellation JSONB,
    -- {"reason": "Schedule conflict", "cancelled_by": "invitee",
    --  "cancelled_at": "2026-05-27T10:00:00Z"}
    reschedule_history JSONB NOT NULL DEFAULT '[]',
    -- [{"previous_start": "2026-05-28T14:00:00Z", "previous_end": "2026-05-28T14:30:00Z",
    --   "rescheduled_at": "2026-05-27T09:00:00Z", "initiated_by": "invitee",
    --   "reason": "conflict with another meeting"}]
    ai JSONB NOT NULL DEFAULT '{}',
    -- {"suggested": true, "slot_rank": 2, "alternatives_shown": 5,
    --  "confidence_score": 0.85, "preference_factors": ["morning_affinity", "buffer_preference"]}
    calendar_sync JSONB,
    -- {"calendar_account_id": "uuid", "calendar_external_id": "primary",
    --  "external_event_id": "google-event-id-123", "synced_at": "2026-05-28T14:05:00Z"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bookings_host ON bookings(host_id, start_time);
CREATE INDEX idx_bookings_event_type ON bookings(event_type_id, start_time);
CREATE INDEX idx_bookings_status ON bookings(host_id, status, start_time);
CREATE INDEX idx_bookings_time ON bookings(start_time, end_time);
CREATE INDEX idx_bookings_ai ON bookings USING GIN (ai);
```

---

## AI Preferences & Focus Blocks

```sql
CREATE TABLE scheduling_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    model_version TEXT,
    preferences JSONB NOT NULL DEFAULT '{}',
    -- {"preferred_times": [
    --     {"day_of_week": 1, "start": "09:00", "end": "11:00", "weight": 0.9, "confidence": 0.8},
    --     {"day_of_week": 3, "start": "14:00", "end": "16:00", "weight": 0.7, "confidence": 0.6}
    --  ],
    --  "avoided_times": [
    --     {"day_of_week": 5, "start": "16:00", "end": "17:00", "weight": 0.85, "confidence": 0.7}
    --  ],
    --  "buffer_preference": {"minimum_minutes": 15, "preferred_minutes": 30, "confidence": 0.75},
    --  "max_meetings_per_day": {"value": 6, "confidence": 0.6},
    --  "energy_pattern": {"peak_start": "09:00", "peak_end": "11:30", "dip_start": "14:00",
    --                     "dip_end": "15:00", "confidence": 0.5},
    --  "meeting_type_affinities": {
    --     "30min_call": {"preferred_window": "morning", "weight": 0.8},
    --     "60min_workshop": {"preferred_window": "afternoon", "weight": 0.6}
    --  }}
    signals JSONB NOT NULL DEFAULT '[]',
    -- [{"type": "slot_chosen", "slot_start": "2026-05-28T14:00:00Z",
    --   "day_of_week": 3, "time_of_day": "14:00", "duration": 30,
    --   "alternatives_shown": 5, "rank_chosen": 2, "at": "2026-05-27T10:00:00Z"},
    --  {"type": "slot_skipped", "slot_start": "2026-05-28T16:00:00Z",
    --   "day_of_week": 3, "time_of_day": "16:00", "at": "2026-05-27T10:00:00Z"},
    --  {"type": "focus_block_overridden", "block_start": "09:00", "block_end": "11:00",
    --   "day_of_week": 1, "at": "2026-05-26T08:00:00Z"}]
    focus_blocks JSONB NOT NULL DEFAULT '[]',
    -- [{"title": "Deep Work", "day_of_week": 1, "start_time": "09:00", "end_time": "11:00",
    --   "is_ai_generated": false, "is_active": true},
    --  {"title": "Focus Time", "day_of_week": 3, "start_time": "09:00", "end_time": "10:30",
    --   "is_ai_generated": true, "is_active": true, "confidence": 0.7}]
    last_trained_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id)
);

CREATE INDEX idx_preferences_user ON scheduling_preferences(user_id);
```

---

## Teams & Routing

```sql
CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    members JSONB NOT NULL DEFAULT '[]',
    -- [{"user_id": "uuid", "name": "Alice", "role": "admin", "weight": 1.0},
    --  {"user_id": "uuid", "name": "Bob", "role": "member", "weight": 1.5}]
    settings JSONB NOT NULL DEFAULT '{}',
    -- {"round_robin_reset": "weekly", "assignment_strategy": "least_recent",
    --  "require_all_for_collective": true}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE routing_forms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    slug TEXT NOT NULL,
    description TEXT,
    config JSONB NOT NULL DEFAULT '{}',
    -- {"fields": [
    --     {"label": "What brings you here?", "type": "select",
    --      "options": ["Sales demo", "Support", "Partnership"], "required": true},
    --     {"label": "Company name", "type": "text", "required": false},
    --     {"label": "Company size", "type": "select",
    --      "options": ["1-10", "11-50", "51-200", "200+"], "required": true}
    --  ],
    --  "rules": [
    --     {"conditions": [{"field_index": 0, "operator": "equals", "value": "Sales demo"},
    --                     {"field_index": 2, "operator": "equals", "value": "200+"}],
    --      "logic": "and", "target_event_type_id": "uuid-enterprise-demo"},
    --     {"conditions": [{"field_index": 0, "operator": "equals", "value": "Sales demo"}],
    --      "logic": "and", "target_event_type_id": "uuid-standard-demo"},
    --     {"conditions": [{"field_index": 0, "operator": "equals", "value": "Support"}],
    --      "logic": "and", "target_url": "https://support.example.com/schedule"}
    --  ],
    --  "fallback_event_type_id": "uuid-general-meeting"}
    submissions JSONB NOT NULL DEFAULT '[]',
    -- [{"answers": {"0": "Sales demo", "1": "Acme Inc", "2": "200+"},
    --   "matched_rule_index": 0, "target_event_type_id": "uuid-enterprise-demo",
    --   "submitted_at": "2026-05-28T10:00:00Z"}]
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, slug)
);
```

---

## Integrations

```sql
CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    url TEXT NOT NULL,
    secret_hash TEXT NOT NULL,
    events TEXT[] NOT NULL DEFAULT '{}',
    delivery_log JSONB NOT NULL DEFAULT '[]',
    -- [{"event": "booking.created", "booking_id": "uuid",
    --   "status": 200, "delivered_at": "2026-05-28T14:05:00Z"},
    --  {"event": "booking.cancelled", "booking_id": "uuid",
    --   "status": 500, "error": "timeout", "delivered_at": "2026-05-29T10:00:00Z"}]
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Find available slots (availability rules from JSONB)

```sql
SELECT id, name, timezone,
       jsonb_array_elements(rules) AS rule
FROM availability_schedules
WHERE user_id = 'user-uuid'
  AND is_default = TRUE;
-- Application layer iterates rules, checks day_of_week and time windows
-- against bookings and focus blocks
```

### Bookings with AI suggestion data

```sql
SELECT title, status, start_time, end_time,
       ai->>'suggested' AS ai_suggested,
       (ai->>'slot_rank')::INT AS slot_rank,
       (ai->>'confidence_score')::REAL AS confidence
FROM bookings
WHERE host_id = 'user-uuid'
  AND ai @> '{"suggested": true}'
ORDER BY start_time DESC
LIMIT 20;
```

### Provider-specific calendar sync state

```sql
SELECT provider, email_address, status,
       sync_state->>'sync_token' AS sync_token,
       sync_state->>'delta_link' AS delta_link,
       last_synced_at
FROM calendar_accounts
WHERE user_id = 'user-uuid';
```

### Focus blocks for free-slot calculation

```sql
SELECT fb->>'title' AS title,
       (fb->>'day_of_week')::INT AS day_of_week,
       fb->>'start_time' AS start_time,
       fb->>'end_time' AS end_time,
       (fb->>'is_ai_generated')::BOOLEAN AS ai_generated
FROM scheduling_preferences,
     jsonb_array_elements(focus_blocks) AS fb
WHERE user_id = 'user-uuid'
  AND (fb->>'is_active')::BOOLEAN = TRUE;
```

### Routing form with conditional rules

```sql
SELECT title, slug,
       jsonb_array_length(config->'fields') AS field_count,
       jsonb_array_length(config->'rules') AS rule_count,
       jsonb_array_length(submissions) AS submission_count
FROM routing_forms
WHERE user_id = 'user-uuid'
  AND is_active = TRUE;
```

### AI preference learning — slot choice analysis

```sql
SELECT (signal->>'type') AS signal_type,
       (signal->>'time_of_day') AS time_of_day,
       (signal->>'day_of_week')::INT AS day_of_week,
       COUNT(*) AS count
FROM scheduling_preferences,
     jsonb_array_elements(signals) AS signal
WHERE user_id = 'user-uuid'
  AND (signal->>'type') IN ('slot_chosen', 'slot_skipped')
GROUP BY signal->>'type', signal->>'time_of_day', signal->>'day_of_week'
ORDER BY count DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core | 2 | users (settings, AI config all JSONB), calendar_accounts (auth, sync_state, calendars all JSONB) |
| Event Types | 2 | event_types (location, questions, workflows all JSONB), availability_schedules (rules, overrides all JSONB) |
| Bookings | 1 | bookings (location, invitees, answers, cancellation, reschedule_history, ai, calendar_sync all JSONB) |
| AI Preferences | 1 | scheduling_preferences (preferences, signals, focus_blocks all JSONB) |
| Teams & Routing | 2 | teams (members, settings as JSONB), routing_forms (config, submissions as JSONB) |
| Integrations | 2 | webhooks (delivery_log as JSONB), api_keys |
| **Total** | **10** | |

---

## Key Design Decisions

1. **Calendar provider metadata in JSONB** — `calendar_accounts.auth` and `calendar_accounts.sync_state` store provider-specific authentication and sync data. Google uses sync tokens and webhook channels; Microsoft uses delta links and subscriptions; CalDAV uses ctags. JSONB handles all providers without provider-specific columns or a metadata table.

2. **Calendars inline on account** — `calendar_accounts.calendars` JSONB array stores the list of calendars for each account (name, color, conflict-check flag). No separate `calendars` table. For personal scheduling, users typically have 2-5 calendars per account — well within JSONB performance bounds.

3. **Questions and workflows inline on event type** — `event_types.questions` and `event_types.workflows` store custom intake questions and automation rules as JSONB arrays. No separate `event_type_questions` or `workflows` tables. This keeps event type configuration in a single row, making it easy to clone event types (copy the row) and version them.

4. **Booking invitees and answers as JSONB** — `bookings.invitees` and `bookings.answers` store per-booking data that varies by event type. Most bookings have 1-3 invitees and 0-5 answers — JSONB arrays are efficient at this scale. Trade-off: no foreign key to a contacts table, but invitee email lookup via GIN index is still fast.

5. **Reschedule history inline** — `bookings.reschedule_history` JSONB array stores every reschedule with previous times, actor, and reason. No self-referential `rescheduled_from_id` foreign key and no linked booking rows. The full reschedule chain is always available in a single row read.

6. **AI preferences as a single row per user** — `scheduling_preferences` stores the entire preference model (preferred times, avoided times, energy pattern, buffer preference), raw signals, and focus blocks in one row per user. This enables atomic reads/writes of the full AI state and avoids the 6-table fan-out of the normalized model (preferences + signals + focus blocks, each with their own rows).

7. **Routing form configuration and submissions unified** — `routing_forms.config` stores fields and rules; `routing_forms.submissions` stores recent submission results. For high-volume routing forms, submissions could be moved to a separate table, but for typical usage (tens of submissions per day), inline JSONB keeps the schema simple.

8. **10 tables total** — Compared to 22 in the normalized model. The JSONB approach trades constraint enforcement for development speed, particularly effective for the many provider-specific, AI-evolving, and user-configurable fields in a scheduling platform.
