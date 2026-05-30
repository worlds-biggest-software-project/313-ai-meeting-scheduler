# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: AI Meeting Scheduler · Created: 2026-05-24

## Philosophy

A meeting scheduler manages a clear hierarchy of entities: users connect calendar accounts, define event types (scheduling pages), set availability rules, and receive bookings from invitees. Layered on top are AI features (preference learning, focus-block protection, slot scoring), team scheduling (round-robin, collective availability), routing forms, and workflow automations (reminders, confirmations, follow-ups). A normalized relational model gives each concept its own table with explicit foreign keys, enabling database-level enforcement of the ownership chain and clean separation between configuration (event types, availability) and runtime data (bookings, preference signals).

This mirrors the conceptual model that scheduling products have established: an event type is a template, a booking is an instance, availability rules define when bookings can occur, and preferences refine which slots are suggested first. Each maps to a table with typed columns and constraints.

The normalized model also supports the CalDAV/iCalendar standards cleanly: events map to VEVENT properties, availability maps to VAVAILABILITY components, and free/busy queries map to VFREEBUSY lookups — all with structured, queryable columns rather than opaque blobs.

**Best for:** Teams building a production-grade scheduling platform where booking integrity, multi-provider calendar sync, team scheduling, and CalDAV compliance are critical. Ideal when the schema should be self-documenting and standards-aligned.

**Trade-offs:**
- **Pro:** Database-enforced relationships from user to calendar to event type to booking
- **Pro:** Availability rules as typed columns enable efficient free-slot computation
- **Pro:** AI preference signals stored with explicit dimensions for training
- **Pro:** Routing forms with relational condition/action structure
- **Con:** 22 tables — more complex schema
- **Con:** Provider-specific calendar metadata requires either new columns or a metadata table
- **Con:** Routing form conditions have variable shapes that fit awkwardly in fixed columns
- **Con:** AI preference models evolve faster than the schema can follow

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5545 (iCalendar) | Booking fields map to VEVENT properties (DTSTART, DTEND, SUMMARY, LOCATION) |
| RFC 4791 (CalDAV) | Calendar account sync via CalDAV with UID tracking |
| RFC 7953 (VAVAILABILITY) | Availability rules model VAVAILABILITY components |
| RFC 5546 (iTIP) | Booking lifecycle maps to iTIP methods (REQUEST, REPLY, CANCEL) |
| RFC 6047 (iMIP) | Email invitations carry .ics attachments per iMIP |
| ISO 8601 | All datetime fields as TIMESTAMPTZ; timezone as IANA identifiers |
| OAuth 2.0 / PKCE | Calendar provider authentication |
| GDPR | Invitee PII in dedicated columns; erasure supported |

---

## Users & Calendar Accounts

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    name TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    locale TEXT NOT NULL DEFAULT 'en',
    avatar_url TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
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
    display_name TEXT,
    status TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'active', 'syncing', 'error', 'disconnected'
    )),
    oauth_access_token_enc TEXT,
    oauth_refresh_token_enc TEXT,
    oauth_token_expires_at TIMESTAMPTZ,
    caldav_url TEXT,
    sync_token TEXT,
    last_synced_at TIMESTAMPTZ,
    is_primary BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, provider, email_address)
);

CREATE INDEX idx_cal_accounts_user ON calendar_accounts(user_id);

CREATE TABLE calendars (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES calendar_accounts(id) ON DELETE CASCADE,
    external_id TEXT NOT NULL,
    name TEXT NOT NULL,
    color TEXT,
    is_checked_for_conflicts BOOLEAN NOT NULL DEFAULT TRUE,
    is_primary BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (account_id, external_id)
);
```

---

## Event Types (Scheduling Pages)

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
    location_type TEXT NOT NULL DEFAULT 'video' CHECK (location_type IN (
        'video', 'phone', 'in_person', 'custom'
    )),
    location_value TEXT,
    color TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    requires_confirmation BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, slug)
);

CREATE INDEX idx_event_types_user ON event_types(user_id);

CREATE TABLE event_type_questions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type_id UUID NOT NULL REFERENCES event_types(id) ON DELETE CASCADE,
    label TEXT NOT NULL,
    field_type TEXT NOT NULL DEFAULT 'text' CHECK (field_type IN (
        'text', 'textarea', 'select', 'radio', 'checkbox', 'phone', 'email'
    )),
    options TEXT[],
    is_required BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order INT NOT NULL DEFAULT 0
);

CREATE INDEX idx_et_questions ON event_type_questions(event_type_id, sort_order);
```

---

## Availability Rules

```sql
CREATE TABLE availability_schedules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name TEXT NOT NULL DEFAULT 'Working Hours',
    timezone TEXT NOT NULL DEFAULT 'UTC',
    is_default BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE availability_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_id UUID NOT NULL REFERENCES availability_schedules(id) ON DELETE CASCADE,
    day_of_week INT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    CHECK (end_time > start_time)
);

CREATE INDEX idx_avail_rules ON availability_rules(schedule_id, day_of_week);

CREATE TABLE date_overrides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_id UUID NOT NULL REFERENCES availability_schedules(id) ON DELETE CASCADE,
    override_date DATE NOT NULL,
    start_time TIME,
    end_time TIME,
    is_unavailable BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE (schedule_id, override_date)
);
```

---

## Bookings

```sql
CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type_id UUID NOT NULL REFERENCES event_types(id),
    host_id UUID NOT NULL REFERENCES users(id),
    calendar_id UUID REFERENCES calendars(id),
    external_event_id TEXT,
    uid TEXT NOT NULL UNIQUE,
    title TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'confirmed' CHECK (status IN (
        'pending', 'confirmed', 'cancelled', 'rescheduled', 'no_show', 'completed'
    )),
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    timezone TEXT NOT NULL,
    location_type TEXT NOT NULL,
    location_value TEXT,
    meeting_url TEXT,
    cancellation_reason TEXT,
    rescheduled_from_id UUID REFERENCES bookings(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bookings_host ON bookings(host_id, start_time);
CREATE INDEX idx_bookings_event_type ON bookings(event_type_id, start_time);
CREATE INDEX idx_bookings_status ON bookings(host_id, status, start_time);
CREATE INDEX idx_bookings_time ON bookings(start_time, end_time);

CREATE TABLE invitees (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    email TEXT NOT NULL,
    name TEXT,
    timezone TEXT,
    rsvp_status TEXT NOT NULL DEFAULT 'accepted' CHECK (rsvp_status IN (
        'accepted', 'declined', 'tentative', 'needs_action'
    )),
    cancel_url TEXT,
    reschedule_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invitees_booking ON invitees(booking_id);
CREATE INDEX idx_invitees_email ON invitees(email);

CREATE TABLE booking_answers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    question_id UUID NOT NULL REFERENCES event_type_questions(id),
    answer TEXT NOT NULL
);

CREATE INDEX idx_booking_answers ON booking_answers(booking_id);
```

---

## AI Preferences & Focus Blocks

```sql
CREATE TABLE scheduling_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    preference_type TEXT NOT NULL CHECK (preference_type IN (
        'preferred_time', 'avoided_time', 'meeting_type_affinity',
        'buffer_preference', 'energy_pattern', 'max_meetings_per_day'
    )),
    dimension TEXT NOT NULL,
    value REAL NOT NULL,
    confidence REAL NOT NULL DEFAULT 0.5,
    source TEXT NOT NULL DEFAULT 'ai_inferred' CHECK (source IN (
        'user_stated', 'ai_inferred', 'history_derived'
    )),
    model_version TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, preference_type, dimension)
);

CREATE INDEX idx_preferences_user ON scheduling_preferences(user_id);

CREATE TABLE preference_signals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    signal_type TEXT NOT NULL CHECK (signal_type IN (
        'booking_accepted', 'booking_declined', 'slot_chosen',
        'slot_skipped', 'reschedule_initiated', 'focus_block_overridden'
    )),
    context_time_of_day TIME,
    context_day_of_week INT,
    context_meeting_type TEXT,
    context_duration_minutes INT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_signals_user ON preference_signals(user_id, created_at DESC);

CREATE TABLE focus_blocks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title TEXT NOT NULL DEFAULT 'Focus Time',
    day_of_week INT CHECK (day_of_week BETWEEN 0 AND 6),
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    is_ai_generated BOOLEAN NOT NULL DEFAULT FALSE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_focus_user ON focus_blocks(user_id) WHERE is_active = TRUE;
```

---

## Teams & Routing

```sql
CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_members (
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('admin', 'member')),
    weight REAL NOT NULL DEFAULT 1.0,
    PRIMARY KEY (team_id, user_id)
);

CREATE TABLE routing_forms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    slug TEXT NOT NULL,
    description TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, slug)
);

CREATE TABLE routing_form_fields (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id UUID NOT NULL REFERENCES routing_forms(id) ON DELETE CASCADE,
    label TEXT NOT NULL,
    field_type TEXT NOT NULL DEFAULT 'text',
    options TEXT[],
    is_required BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order INT NOT NULL DEFAULT 0
);

CREATE TABLE routing_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id UUID NOT NULL REFERENCES routing_forms(id) ON DELETE CASCADE,
    field_id UUID NOT NULL REFERENCES routing_form_fields(id),
    operator TEXT NOT NULL CHECK (operator IN ('equals', 'contains', 'not_equals')),
    value TEXT NOT NULL,
    target_event_type_id UUID REFERENCES event_types(id),
    target_url TEXT,
    sort_order INT NOT NULL DEFAULT 0
);
```

---

## Workflows & Integrations

```sql
CREATE TABLE workflows (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type_id UUID NOT NULL REFERENCES event_types(id) ON DELETE CASCADE,
    trigger TEXT NOT NULL CHECK (trigger IN (
        'booking_created', 'booking_cancelled', 'booking_rescheduled',
        'reminder_before', 'follow_up_after'
    )),
    action_type TEXT NOT NULL CHECK (action_type IN (
        'email', 'sms', 'webhook', 'slack'
    )),
    offset_minutes INT,
    template_subject TEXT,
    template_body TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    url TEXT NOT NULL,
    secret_hash TEXT NOT NULL,
    events TEXT[] NOT NULL DEFAULT '{}',
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

### Find available slots for an event type

```sql
WITH busy_times AS (
    SELECT start_time, end_time
    FROM bookings
    WHERE host_id = 'user-uuid'
      AND status IN ('confirmed', 'pending')
      AND start_time >= now()
      AND start_time < now() + INTERVAL '7 days'
    UNION ALL
    SELECT start_time::TIMESTAMPTZ, end_time::TIMESTAMPTZ
    FROM focus_blocks
    WHERE user_id = 'user-uuid'
      AND is_active = TRUE
),
available_windows AS (
    SELECT ar.day_of_week, ar.start_time, ar.end_time
    FROM availability_rules ar
    JOIN availability_schedules s ON s.id = ar.schedule_id
    WHERE s.user_id = 'user-uuid'
      AND s.is_default = TRUE
)
SELECT aw.day_of_week, aw.start_time, aw.end_time
FROM available_windows aw
-- Application layer intersects available_windows with busy_times
-- and the event_type constraints (duration, buffer, min_notice)
ORDER BY aw.day_of_week, aw.start_time;
```

### AI slot scoring by preference

```sql
SELECT sp.dimension, sp.value AS preference_weight, sp.confidence
FROM scheduling_preferences sp
WHERE sp.user_id = 'user-uuid'
  AND sp.preference_type = 'preferred_time'
ORDER BY sp.value DESC;
```

### Booking history with invitee details

```sql
SELECT b.title, b.status, b.start_time, b.end_time, b.timezone,
       et.title AS event_type, et.duration_minutes,
       i.email AS invitee_email, i.name AS invitee_name
FROM bookings b
JOIN event_types et ON et.id = b.event_type_id
JOIN invitees i ON i.booking_id = b.id
WHERE b.host_id = 'user-uuid'
  AND b.start_time >= '2026-05-01'
ORDER BY b.start_time DESC;
```

### Team round-robin next assignment

```sql
SELECT tm.user_id, u.name,
       COUNT(b.id) AS recent_bookings
FROM team_members tm
JOIN users u ON u.id = tm.user_id
LEFT JOIN bookings b ON b.host_id = tm.user_id
    AND b.event_type_id = 'event-type-uuid'
    AND b.status = 'confirmed'
    AND b.created_at >= now() - INTERVAL '7 days'
WHERE tm.team_id = 'team-uuid'
GROUP BY tm.user_id, u.name, tm.weight
ORDER BY COUNT(b.id) ASC, tm.weight DESC
LIMIT 1;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users & Calendar | 3 | users, calendar_accounts, calendars |
| Event Types | 2 | event_types, event_type_questions |
| Availability | 3 | availability_schedules, availability_rules, date_overrides |
| Bookings | 3 | bookings, invitees, booking_answers |
| AI Preferences | 3 | scheduling_preferences, preference_signals, focus_blocks |
| Teams & Routing | 4 | teams, team_members, routing_forms, routing_form_fields, routing_rules |
| Workflows & API | 3 | workflows, webhooks, api_keys |
| **Total** | **22** | (routing_rules counted within Teams & Routing) |

---

## Key Design Decisions

1. **Bookings carry iCalendar UID** — Each booking has a unique `uid` field that maps directly to the iCalendar UID property. This enables .ics file generation and CalDAV sync without an intermediate mapping table.

2. **Availability as day-of-week rules with date overrides** — `availability_rules` define recurring weekly windows; `date_overrides` handle holidays and one-off changes. This matches the VAVAILABILITY model (RFC 7953) and how every scheduling product structures availability.

3. **AI preferences as typed rows** — `scheduling_preferences` stores each learned preference (preferred morning slots, buffer preference, max meetings per day) as a separate row with a type, dimension, value, and confidence. This enables the AI to query "what does this user prefer for 30-minute calls?" as a simple filtered SELECT.

4. **Preference signals as training data** — `preference_signals` logs every user action that indicates a scheduling preference (choosing a slot, declining a suggestion, overriding a focus block). These signals are the training data for the preference-learning engine.

5. **Focus blocks as explicit rows** — AI-detected and user-defined focus blocks are stored in `focus_blocks` with the `is_ai_generated` flag distinguishing them. The free-slot calculator treats focus blocks as busy periods by default.

6. **Routing forms with relational conditions** — `routing_form_fields`, and `routing_rules` model the condition→action logic of routing forms. Each rule matches a field value and routes to an event type or external URL.

7. **Workflows as trigger→action pairs** — `workflows` stores reminder/confirmation templates tied to event types. The `trigger` (booking created, 15 min before, 1 hour after) and `action_type` (email, SMS, webhook) are explicit columns, not JSONB.

8. **Rescheduling as linked bookings** — A rescheduled booking creates a new row with `rescheduled_from_id` pointing to the original. This preserves the full rescheduling history and enables analytics on reschedule rates.
