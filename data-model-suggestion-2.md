# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: AI Meeting Scheduler · Created: 2026-05-24

## Philosophy

A meeting scheduler is fundamentally a state machine: bookings move through pending → confirmed → rescheduled → cancelled → completed, calendars sync and resync, availability windows shift, and AI preferences evolve as users accept or decline suggestions. An event-sourced model captures every state transition as an immutable event, making the booking lifecycle fully auditable and enabling temporal queries ("what was this user's availability at the time slot X was offered?"). The event store is the single source of truth; read models (CQRS) are materialised projections optimised for the queries the application actually runs.

This is particularly powerful for a scheduling platform because:
- **Booking lifecycle is inherently event-driven** — Requested, confirmed, rescheduled, cancelled, completed, and no-show are all discrete state transitions that users expect to audit. Event sourcing captures the full chain with timestamps, actors, and context.
- **AI preference learning is signal-based** — Every slot chosen, slot skipped, focus block overridden, and reschedule initiated is a training signal. An event store is a natural fit for accumulating these signals over time without mutation.
- **Calendar sync produces conflict events** — Multi-provider sync generates a stream of events (event added, event moved, event deleted) that must be reconciled. Event sourcing enables replay when sync logic changes.
- **Rescheduling chains need history** — When a booking is rescheduled multiple times, the full chain of start_time changes matters for analytics and user trust. Events capture each transition without self-referential foreign keys.

The result is an event store with 30+ event types covering booking lifecycle, calendar sync, preference learning, and workflow execution, plus 7 read model tables and 10 reference tables.

**Best for:** Teams building a scheduling platform where full booking audit trails, AI preference learning from historical signals, and temporal analytics (reschedule patterns, peak booking times, preference drift) are first-class requirements. Ideal when the regulatory or enterprise context demands immutable records of every scheduling action.

**Trade-offs:**
- **Pro:** Complete audit trail — every booking state change is an immutable event
- **Pro:** AI preference signals accumulate naturally as events
- **Pro:** Temporal queries ("what was available when this slot was offered?") via event replay
- **Pro:** Rescheduling history without self-referential foreign keys
- **Pro:** Calendar sync events enable replay when provider APIs change
- **Con:** Eventual consistency between event store and read models
- **Con:** Free-slot computation from events requires projection; direct table scan is simpler
- **Con:** Event schema versioning adds operational complexity
- **Con:** More infrastructure (event bus, projection workers) than a simple CRUD model

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope fields (ce_source, ce_type, ce_specversion) |
| RFC 5545 (iCalendar) | Booking events carry VEVENT-compatible fields (DTSTART, DTEND, UID) |
| RFC 4791 (CalDAV) | Calendar sync events track CalDAV sync tokens and UIDs |
| RFC 7953 (VAVAILABILITY) | Availability change events carry VAVAILABILITY-compatible data |
| RFC 5546 (iTIP) | Booking lifecycle maps to iTIP methods in event payloads |
| ISO 8601 | All timestamps as TIMESTAMPTZ in events and read models |
| OAuth 2.0 | Calendar account connection events |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type TEXT NOT NULL CHECK (stream_type IN (
        'booking', 'calendar', 'availability', 'preference',
        'team', 'workflow', 'routing'
    )),
    stream_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    event_version INT NOT NULL,
    payload JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {"actor_id": "user-uuid", "actor_type": "user|system|ai",
    --  "ip_address": "...", "user_agent": "...",
    --  "ce_source": "/scheduler/bookings", "ce_specversion": "1.0",
    --  "correlation_id": "uuid", "causation_id": "uuid"}
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE event_store_2026_q1 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE event_store_2026_q2 PARTITION OF event_store
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE event_store_2026_q3 PARTITION OF event_store
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE event_store_2026_q4 PARTITION OF event_store
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_events_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_events_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_events_occurred ON event_store(occurred_at);
```

---

## Event Type Registry

```sql
CREATE TABLE event_type_registry (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1
);

-- Booking lifecycle events
INSERT INTO event_type_registry VALUES
    ('booking.requested', 'booking', 'Invitee requested a booking', 1),
    ('booking.confirmed', 'booking', 'Host or auto-confirm accepted the booking', 1),
    ('booking.cancelled_by_host', 'booking', 'Host cancelled the booking', 1),
    ('booking.cancelled_by_invitee', 'booking', 'Invitee cancelled the booking', 1),
    ('booking.rescheduled', 'booking', 'Booking moved to a new time', 1),
    ('booking.completed', 'booking', 'Booking end time passed without cancellation', 1),
    ('booking.no_show', 'booking', 'Host marked invitee as no-show', 1),
    ('booking.reminder_sent', 'booking', 'Reminder email/SMS sent', 1),
    ('booking.follow_up_sent', 'booking', 'Post-meeting follow-up sent', 1),

    -- Calendar sync events
    ('calendar.account_connected', 'calendar', 'OAuth flow completed for a provider', 1),
    ('calendar.account_disconnected', 'calendar', 'User disconnected a calendar account', 1),
    ('calendar.sync_started', 'calendar', 'Incremental sync initiated', 1),
    ('calendar.sync_completed', 'calendar', 'Sync finished with event count', 1),
    ('calendar.sync_failed', 'calendar', 'Sync failed with error details', 1),
    ('calendar.event_detected', 'calendar', 'External event discovered during sync', 1),
    ('calendar.event_moved', 'calendar', 'External event changed time', 1),
    ('calendar.event_deleted', 'calendar', 'External event removed', 1),
    ('calendar.conflict_detected', 'calendar', 'Booking conflicts with new external event', 1),

    -- Availability events
    ('availability.schedule_created', 'availability', 'New availability schedule defined', 1),
    ('availability.rule_added', 'availability', 'Weekly rule added to schedule', 1),
    ('availability.rule_removed', 'availability', 'Weekly rule removed from schedule', 1),
    ('availability.override_set', 'availability', 'Date-specific override applied', 1),
    ('availability.override_removed', 'availability', 'Date-specific override removed', 1),

    -- AI preference events
    ('preference.slot_chosen', 'preference', 'User chose a suggested slot', 1),
    ('preference.slot_skipped', 'preference', 'User skipped a suggested slot', 1),
    ('preference.focus_block_created', 'preference', 'Focus block added (user or AI)', 1),
    ('preference.focus_block_overridden', 'preference', 'User booked over a focus block', 1),
    ('preference.model_updated', 'preference', 'AI preference model retrained', 1),
    ('preference.user_stated', 'preference', 'User explicitly stated a preference', 1),
    ('preference.energy_pattern_detected', 'preference', 'AI detected energy/productivity pattern', 1),

    -- Team events
    ('team.created', 'team', 'Team created', 1),
    ('team.member_added', 'team', 'Member joined team', 1),
    ('team.member_removed', 'team', 'Member left team', 1),
    ('team.round_robin_assigned', 'team', 'Round-robin selected a host', 1),
    ('team.collective_confirmed', 'team', 'All required members confirmed availability', 1),

    -- Workflow events
    ('workflow.created', 'workflow', 'Workflow automation configured', 1),
    ('workflow.triggered', 'workflow', 'Workflow fired on booking event', 1),
    ('workflow.action_executed', 'workflow', 'Email/SMS/webhook sent', 1),
    ('workflow.action_failed', 'workflow', 'Workflow action delivery failed', 1),

    -- Routing events
    ('routing.form_submitted', 'routing', 'Invitee submitted a routing form', 1),
    ('routing.rule_matched', 'routing', 'Form answer matched a routing rule', 1),
    ('routing.redirected', 'routing', 'Invitee redirected to target event type', 1);
```

---

## Event Payload Examples

```sql
-- booking.requested
-- {"event_type_id": "uuid", "host_id": "uuid",
--  "uid": "booking-uid@scheduler.example.com",
--  "title": "30 Minute Meeting",
--  "start_time": "2026-05-28T14:00:00Z", "end_time": "2026-05-28T14:30:00Z",
--  "timezone": "America/New_York",
--  "location_type": "video", "meeting_url": "https://meet.example.com/abc",
--  "invitee": {"email": "jane@example.com", "name": "Jane Doe", "timezone": "Europe/London"},
--  "answers": [{"question_id": "uuid", "label": "What is this about?", "answer": "Project review"}],
--  "slot_suggestion_rank": 2, "ai_suggested": true}

-- booking.rescheduled
-- {"previous_start_time": "2026-05-28T14:00:00Z", "previous_end_time": "2026-05-28T14:30:00Z",
--  "new_start_time": "2026-05-29T10:00:00Z", "new_end_time": "2026-05-29T10:30:00Z",
--  "reason": "conflict with another meeting", "initiated_by": "invitee"}

-- preference.slot_chosen
-- {"slot_start": "2026-05-28T14:00:00Z", "slot_end": "2026-05-28T14:30:00Z",
--  "alternatives_shown": 5, "rank_chosen": 2,
--  "day_of_week": 3, "time_of_day": "14:00",
--  "meeting_type": "30min_call", "duration_minutes": 30}

-- calendar.conflict_detected
-- {"booking_id": "uuid", "booking_start": "2026-05-28T14:00:00Z",
--  "external_event_id": "google-event-id",
--  "external_event_title": "Team standup",
--  "external_start": "2026-05-28T13:45:00Z", "external_end": "2026-05-28T14:15:00Z",
--  "provider": "google", "overlap_minutes": 15}
```

---

## Stream Snapshots

```sql
CREATE TABLE stream_snapshots (
    stream_id UUID NOT NULL,
    stream_type TEXT NOT NULL,
    snapshot_version INT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Reference Data

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    name TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    locale TEXT NOT NULL DEFAULT 'en',
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
    status TEXT NOT NULL DEFAULT 'active',
    oauth_access_token_enc TEXT,
    oauth_refresh_token_enc TEXT,
    oauth_token_expires_at TIMESTAMPTZ,
    caldav_url TEXT,
    sync_token TEXT,
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, provider, email_address)
);

CREATE TABLE event_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    team_id UUID REFERENCES teams(id),
    title TEXT NOT NULL,
    slug TEXT NOT NULL,
    duration_minutes INT NOT NULL DEFAULT 30,
    buffer_before_minutes INT NOT NULL DEFAULT 0,
    buffer_after_minutes INT NOT NULL DEFAULT 0,
    min_notice_hours INT NOT NULL DEFAULT 1,
    max_future_days INT NOT NULL DEFAULT 60,
    scheduling_type TEXT NOT NULL DEFAULT 'individual',
    location_type TEXT NOT NULL DEFAULT 'video',
    location_value TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    requires_confirmation BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, slug)
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_members (
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member',
    weight REAL NOT NULL DEFAULT 1.0,
    PRIMARY KEY (team_id, user_id)
);

CREATE TABLE routing_forms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    slug TEXT NOT NULL,
    config JSONB NOT NULL DEFAULT '{}',
    -- {"fields": [{"label": "Company size", "type": "select",
    --              "options": ["1-10", "11-50", "51-200", "200+"]}],
    --  "rules": [{"field_index": 0, "operator": "equals", "value": "200+",
    --             "target_event_type_id": "uuid"}]}
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, slug)
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

## Read Models (CQRS Projections)

```sql
CREATE TABLE rm_bookings (
    id UUID PRIMARY KEY,
    event_type_id UUID NOT NULL REFERENCES event_types(id),
    host_id UUID NOT NULL REFERENCES users(id),
    uid TEXT NOT NULL UNIQUE,
    title TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    timezone TEXT NOT NULL,
    location_type TEXT NOT NULL,
    location_value TEXT,
    meeting_url TEXT,
    cancellation_reason TEXT,
    reschedule_count INT NOT NULL DEFAULT 0,
    original_start_time TIMESTAMPTZ,
    invitees JSONB NOT NULL DEFAULT '[]',
    -- [{"email": "jane@example.com", "name": "Jane Doe",
    --   "timezone": "Europe/London", "rsvp_status": "accepted"}]
    answers JSONB NOT NULL DEFAULT '[]',
    -- [{"question": "What is this about?", "answer": "Project review"}]
    ai_suggested BOOLEAN NOT NULL DEFAULT FALSE,
    slot_suggestion_rank INT,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_bookings_host ON rm_bookings(host_id, start_time);
CREATE INDEX idx_rm_bookings_status ON rm_bookings(host_id, status, start_time);
CREATE INDEX idx_rm_bookings_time ON rm_bookings(start_time, end_time);

CREATE TABLE rm_availability (
    user_id UUID NOT NULL REFERENCES users(id),
    schedule_name TEXT NOT NULL DEFAULT 'Working Hours',
    timezone TEXT NOT NULL DEFAULT 'UTC',
    is_default BOOLEAN NOT NULL DEFAULT FALSE,
    rules JSONB NOT NULL DEFAULT '[]',
    -- [{"day_of_week": 1, "start_time": "09:00", "end_time": "17:00"},
    --  {"day_of_week": 2, "start_time": "09:00", "end_time": "17:00"}]
    overrides JSONB NOT NULL DEFAULT '{}',
    -- {"2026-06-15": {"is_unavailable": true},
    --  "2026-06-20": {"start_time": "10:00", "end_time": "14:00"}}
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, schedule_name)
);

CREATE TABLE rm_calendar_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    account_id UUID NOT NULL REFERENCES calendar_accounts(id),
    external_id TEXT NOT NULL,
    title TEXT,
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    is_all_day BOOLEAN NOT NULL DEFAULT FALSE,
    status TEXT NOT NULL DEFAULT 'confirmed',
    provider TEXT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (account_id, external_id)
);

CREATE INDEX idx_rm_cal_events ON rm_calendar_events(user_id, start_time, end_time);

CREATE TABLE rm_preferences (
    user_id UUID NOT NULL REFERENCES users(id),
    preference_type TEXT NOT NULL,
    dimension TEXT NOT NULL,
    value REAL NOT NULL,
    confidence REAL NOT NULL DEFAULT 0.5,
    source TEXT NOT NULL DEFAULT 'ai_inferred',
    signal_count INT NOT NULL DEFAULT 0,
    model_version TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, preference_type, dimension)
);

CREATE TABLE rm_focus_blocks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    title TEXT NOT NULL DEFAULT 'Focus Time',
    day_of_week INT,
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    is_ai_generated BOOLEAN NOT NULL DEFAULT FALSE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_focus ON rm_focus_blocks(user_id) WHERE is_active = TRUE;

CREATE TABLE rm_team_assignments (
    team_id UUID NOT NULL REFERENCES teams(id),
    user_id UUID NOT NULL REFERENCES users(id),
    event_type_id UUID NOT NULL REFERENCES event_types(id),
    recent_booking_count INT NOT NULL DEFAULT 0,
    last_assigned_at TIMESTAMPTZ,
    weight REAL NOT NULL DEFAULT 1.0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id, event_type_id)
);

CREATE TABLE rm_workflow_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id UUID NOT NULL,
    trigger TEXT NOT NULL,
    action_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    error_message TEXT,
    executed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_workflow ON rm_workflow_log(booking_id);
```

---

## Example Queries

### Replay booking lifecycle

```sql
SELECT event_type, payload, metadata->>'actor_type' AS actor,
       occurred_at
FROM event_store
WHERE stream_type = 'booking'
  AND stream_id = 'booking-uuid'
ORDER BY event_version;
```

### AI preference signal analysis (last 90 days)

```sql
SELECT event_type,
       payload->>'time_of_day' AS time_of_day,
       payload->>'day_of_week' AS day_of_week,
       COUNT(*) AS signal_count
FROM event_store
WHERE stream_type = 'preference'
  AND stream_id = 'user-uuid'
  AND event_type IN ('preference.slot_chosen', 'preference.slot_skipped')
  AND occurred_at >= now() - INTERVAL '90 days'
GROUP BY event_type, payload->>'time_of_day', payload->>'day_of_week'
ORDER BY signal_count DESC;
```

### Reschedule pattern analysis

```sql
SELECT DATE_TRUNC('week', occurred_at) AS week,
       COUNT(*) AS reschedule_count,
       AVG((payload->>'overlap_minutes')::INT) AS avg_conflict_overlap
FROM event_store
WHERE event_type = 'booking.rescheduled'
  AND occurred_at >= now() - INTERVAL '6 months'
GROUP BY DATE_TRUNC('week', occurred_at)
ORDER BY week;
```

### Calendar sync health dashboard

```sql
SELECT e.event_type,
       e.payload->>'provider' AS provider,
       COUNT(*) AS event_count,
       MAX(e.occurred_at) AS last_occurrence
FROM event_store e
WHERE e.stream_type = 'calendar'
  AND e.event_type IN ('calendar.sync_completed', 'calendar.sync_failed')
  AND e.occurred_at >= now() - INTERVAL '7 days'
GROUP BY e.event_type, e.payload->>'provider';
```

### Available slots (from read model)

```sql
WITH busy AS (
    SELECT start_time, end_time
    FROM rm_bookings
    WHERE host_id = 'user-uuid'
      AND status IN ('confirmed', 'pending')
      AND start_time >= now()
      AND start_time < now() + INTERVAL '7 days'
    UNION ALL
    SELECT start_time, end_time
    FROM rm_calendar_events
    WHERE user_id = 'user-uuid'
      AND start_time >= now()
      AND start_time < now() + INTERVAL '7 days'
      AND status = 'confirmed'
)
SELECT * FROM busy ORDER BY start_time;
-- Application layer intersects with rm_availability rules and rm_focus_blocks
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), event_type_registry, stream_snapshots, projection_checkpoints |
| Reference Data | 8 | users, calendar_accounts, event_types, teams, team_members, routing_forms, webhooks, api_keys |
| Read Models | 7 | rm_bookings, rm_availability, rm_calendar_events, rm_preferences, rm_focus_blocks, rm_team_assignments, rm_workflow_log |
| **Total** | **18** | |

---

## Key Design Decisions

1. **Booking stream captures full lifecycle** — A booking's stream contains every event from `booking.requested` through `booking.completed` or `booking.cancelled_*`. Rescheduling is an event (`booking.rescheduled`) with previous and new times, not a new stream. This gives a single stream ID for the full booking history including multiple reschedules.

2. **Preference signals as events** — Every `preference.slot_chosen`, `preference.slot_skipped`, and `preference.focus_block_overridden` event is an immutable training signal. The AI preference engine replays these events (filtered by recency and confidence decay) to retrain the preference model. The `rm_preferences` read model stores the current model output.

3. **Calendar sync as event stream** — Each calendar account has its own stream. Sync events (`calendar.sync_started`, `calendar.event_detected`, `calendar.conflict_detected`) create an audit trail of external calendar state. When a provider's API changes or sync logic is updated, events can be replayed to rebuild the `rm_calendar_events` projection.

4. **Read model for availability** — `rm_availability` denormalises the current availability schedule (rules + overrides) into a single row per user-schedule pair. The free-slot calculator reads this projection directly rather than querying the event store. Availability change events (`availability.rule_added`, `availability.override_set`) update the projection.

5. **Conflict detection as events** — When a calendar sync discovers an external event that overlaps a confirmed booking, a `calendar.conflict_detected` event is emitted. This enables automated workflows (notify host, suggest reschedule) and analytics on conflict frequency by provider.

6. **Routing forms in reference data** — Routing form configuration lives in a reference table with JSONB config (fields and rules). Routing execution produces events (`routing.form_submitted`, `routing.rule_matched`, `routing.redirected`) in the event store, separating configuration from runtime history.

7. **Workflow execution log as read model** — `rm_workflow_log` tracks which workflows fired for which bookings and whether delivery succeeded. This is a projection of `workflow.triggered` and `workflow.action_executed`/`workflow.action_failed` events.

8. **Partitioned event store** — Quarterly partitions keep query performance manageable as booking and sync events accumulate. Old partitions can be archived or compressed without affecting active queries.
