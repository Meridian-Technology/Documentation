# Analytics System Documentation

This guide explains the analytics data collection system implemented for Meridian Mobile. The system provides a lightweight, future-proof way to collect raw event data reliably for mobile launch and beyond.

## Overview

The analytics system is a "dumb pipe" that collects raw event data and stores it in MongoDB. It's designed to be:
- **Reliable**: Local queue survives app restarts and network failures
- **Future-proof**: Stable event envelope schema that works with any analytics tool (GA4, PostHog, BigQuery, etc.)
- **Privacy-focused**: No PII stored, client and server-side scrubbing
- **Idempotent**: Safe retries with unique event IDs
- **Lightweight**: Minimal overhead, batched ingestion

## Architecture

```
Mobile App (Expo RN)
  └─ analytics.ts SDK
      ├─ Local Queue (AsyncStorage, max 500 events)
      ├─ Batching (20 events/batch, 45s flush)
      └─ Retry Logic (exponential backoff)
           │
           └─ POST /v1/events
                  │
Backend (Express)
  └─ routes/analyticsRoutes.js
      ├─ Validation & Sanitization
      ├─ Enrichment (received_at, ip_hash)
      └─ MongoDB Insert
           │
MongoDB
  └─ analytics_events collection
      ├─ Index: { event_id: 1 } unique
      ├─ Index: { ts: -1 }
      └─ Index: { user_id: 1, ts: -1 }
```

## Event Envelope Schema

Every event stored in MongoDB follows this structure:

```typescript
{
  schema_version: 1,
  event_id: string,              // UUID, unique
  event: string,                 // Event name (e.g., "screen_view")
  ts: Date,                      // Client timestamp (ISO string)
  received_at: Date,             // Server timestamp
  anonymous_id: string,          // Device-scoped UUID (persistent)
  user_id?: string | null,       // User ID (set after login)
  session_id: string,            // Session UUID (regenerated after 30min background)
  platform: "ios" | "android" | "web",
  app: "meridian",
  app_version: string,           // e.g., "1.0.0"
  build: string | number,        // Build number
  env: "prod" | "staging" | "dev",
  context: {
    screen?: string,             // Current screen name
    route?: string,               // Navigation route
    referrer?: string,
    locale?: string,
    timezone?: string,
    device_model?: string,
    os_version?: string,
    network?: string
  },
  properties: Record<string, any>, // Event-specific properties (JSON-safe, no PII)
  ip_hash?: string,              // SHA-256 hash of IP (server-side)
  user_agent_summary?: string    // Basic device type (server-side)
}
```

## Mobile SDK Usage

### Initialization

The analytics SDK is automatically initialized in `AppContent.tsx` on app start. No manual initialization needed.

```typescript
// Already done in AppContent.tsx
import { analytics } from '@/services/analytics/analytics';

// SDK is initialized automatically with:
// - Platform detection (ios/android/web)
// - App version from expo-constants
// - Build number from expo-constants
// - Environment (dev/prod based on __DEV__)
```

### Tracking Events

#### Basic Event Tracking

```typescript
import { analytics } from '@/services/analytics/analytics';

// Track a simple event
await analytics.track('event_viewed', {
  event_id: '507f1f77bcf86cd799439011',
  event_name: 'React Workshop'
});

// Track with context overrides
await analytics.track('search_performed', {
  query_length: 10,
  result_count: 5
}, {
  screen: 'RoomSearch',
  route: 'MainTabs/RoomSearch'
});
```

#### Screen View Tracking

Screen views are automatically tracked via navigation state changes. You can also manually track:

```typescript
// Automatic (via AppNavigator.tsx)
// Screen views are tracked automatically when navigation changes

// Manual tracking
await analytics.screen('EventDetails', {
  event_id: '507f1f77bcf86cd799439011'
});
```

#### User Identification

User identification is handled automatically in `AuthContext.tsx`:

```typescript
// Automatically called on login
await analytics.identify(userId);

// Automatically called on logout
await analytics.reset();
```

### Manual Usage (If Needed)

If you need to track events manually in a new screen:

```typescript
import { analytics } from '@/services/analytics/analytics';

// In your component
const handleEventRSVP = async () => {
  // Your RSVP logic...
  
  // Track the event
  await analytics.track('event_rsvp', {
    event_id: eventId,
    rsvp_status: 'going'
  });
};
```

## Event Taxonomy

### Core Events (Auto-tracked)

- `session_start` - App opened/foregrounded (auto-tracked on init)
- `screen_view` - Screen navigation (auto-tracked via navigation listener)
- `login_completed` - User successfully logs in (auto-tracked in AuthContext)
- `logout` - User logs out (auto-tracked in AuthContext)
- `signup_started` - User begins signup (tracked in RegisterScreen)
- `signup_completed` - User completes signup (tracked in AuthContext)

### Meridian-Specific Events (Manual Tracking)

These events should be tracked manually in relevant screens:

- `event_viewed` - User views event details
- `event_rsvp` - User RSVPs to event
- `room_viewed` - User views room details
- `room_saved` - User saves a room
- `search_performed` - User performs search
- `api_error` - API error occurred (redacted, no sensitive data)
- `onboarding_started` - User starts onboarding flow
- `onboarding_completed` - User completes onboarding

### Example: Adding Event Tracking to a Screen

```typescript
// In EventDetailsScreen.tsx
import { analytics } from '@/services/analytics/analytics';
import { useEffect } from 'react';

export default function EventDetailsScreen({ route }) {
  const { eventId } = route.params;

  useEffect(() => {
    // Track event view when screen loads
    analytics.track('event_viewed', {
      event_id: eventId
    });
  }, [eventId]);

  const handleRSVP = async () => {
    // Your RSVP logic...
    
    // Track RSVP
    await analytics.track('event_rsvp', {
      event_id: eventId,
      rsvp_status: 'going'
    });
  };

  // ... rest of component
}
```

## Backend API

### Endpoint

**POST** `/v1/events`

### Request Body

```json
{
  "events": [
    {
      "schema_version": 1,
      "event_id": "550e8400-e29b-41d4-a716-446655440000",
      "event": "screen_view",
      "ts": "2025-01-20T10:30:00.000Z",
      "anonymous_id": "550e8400-e29b-41d4-a716-446655440001",
      "user_id": "507f1f77bcf86cd799439011",
      "session_id": "550e8400-e29b-41d4-a716-446655440002",
      "platform": "ios",
      "app": "meridian",
      "app_version": "1.0.0",
      "build": "1",
      "env": "prod",
      "context": {
        "screen": "EventDetails",
        "timezone": "America/New_York"
      },
      "properties": {
        "event_id": "507f1f77bcf86cd799439011"
      }
    }
  ]
}
```

### Response

```json
{
  "received": 1,
  "inserted": 1,
  "duplicates": 0,
  "dropped": 0
}
```

### Limits

- Max payload size: 1MB
- Max events per request: 50
- Max event size: 10KB
- Max properties size: 5KB per event

### Validation

The endpoint validates:
- Required fields: `event`, `ts`, `event_id`, `anonymous_id`, `session_id`, `platform`, `app_version`, `env`
- Platform enum: `ios`, `android`, `web`
- Environment enum: `prod`, `staging`, `dev`
- Event size limits
- PII scrubbing (removes `email`, `name`, `phone`, `password`, etc.)

## Database Setup

### Create Indexes

Run the index creation script:

```bash
# For default school (rpi)
node Meridian/backend/events/scripts/createAnalyticsIndexes.js

# For specific school
node Meridian/backend/events/scripts/createAnalyticsIndexes.js berkeley
```

This creates:
- Unique index on `event_id` (for idempotency)
- Descending index on `ts` (for time-based queries)
- Composite index on `user_id + ts` (for user activity queries)
- Composite index on `anonymous_id + ts` (for anonymous user queries)

### Collection Name

Events are stored in the `analytics_events` collection.

### Querying Events

```javascript
// Get recent events
db.analytics_events.find().sort({ ts: -1 }).limit(100);

// Get events for a user
db.analytics_events.find({ user_id: ObjectId("...") }).sort({ ts: -1 });

// Get events by type
db.analytics_events.find({ event: "screen_view" }).sort({ ts: -1 });

// Get events in date range
db.analytics_events.find({
  ts: {
    $gte: ISODate("2025-01-01"),
    $lte: ISODate("2025-01-31")
  }
});
```

## Privacy & PII Handling

### Client-Side Scrubbing

The SDK automatically removes PII keys from properties:
- `email`, `name`, `phone`, `password`, `ssn`, `credit_card`, `address`

### Server-Side Scrubbing

The backend also scrubs PII and enforces size limits.

### IP Address Handling

IP addresses are hashed using SHA-256 before storage (`ip_hash` field).

### User Agent Handling

Only a basic summary is stored (`user_agent_summary`: `ios`, `android`, `chrome`, etc.), not the full user agent string.

## Local Queue & Reliability

### Queue Management

- Events are stored in AsyncStorage (`@meridian/analytics/queue`)
- Max queue size: 500 events (oldest dropped if exceeded)
- Queue persists across app restarts

### Batching

- Max batch size: 20 events per request
- Auto-flush interval: 45 seconds
- Auto-flush triggers:
  - App goes to background
  - App comes to foreground
  - Queue reaches batch size

### Retry Logic

- Exponential backoff: 1s, 2s, 4s, 8s, max 30s
- Max retries: 3 per batch
- Retries on network errors and server errors (5xx)
- No retry on client errors (4xx)

### Idempotency

- Each event has a unique `event_id` (UUID)
- Duplicate events are detected and ignored (not inserted)
- Safe to retry without creating duplicates

## Session Management

### Session ID

- Generated on app start
- Regenerated when app comes to foreground after 30+ minutes in background
- Stored in AsyncStorage (`@meridian/analytics/session_id`)

### Anonymous ID

- Generated on first app launch
- Persists across app restarts
- Never changes (unless app is uninstalled)
- Stored in AsyncStorage (`@meridian/analytics/anonymous_id`)

### User ID

- Set when user logs in (`analytics.identify(userId)`)
- Cleared when user logs out (`analytics.reset()`)
- Stored in AsyncStorage (`@meridian/analytics/user_id`)

## Debugging

### Development Mode

In development (`__DEV__`), the SDK logs:
- Event queue counts
- Batch send attempts
- Retry attempts
- Payload details

### Inspecting Queue

```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';

// Check queue size
const queue = await AsyncStorage.getItem('@meridian/analytics/queue');
const events = queue ? JSON.parse(queue) : [];
console.log(`Queue has ${events.length} events`);
```

### Backend Logging

The backend logs:
- Received event counts
- Inserted/duplicate/dropped counts
- Validation errors
- Insert errors

## Future Integration

The event envelope is designed to be compatible with any analytics tool:

### Google Analytics 4 (GA4)

```javascript
// Transform events to GA4 format
events.forEach(event => {
  gtag('event', event.event, {
    event_id: event.event_id,
    user_id: event.user_id,
    // ... map properties
  });
});
```

### PostHog

```javascript
// Transform events to PostHog format
events.forEach(event => {
  posthog.capture(event.event, {
    distinct_id: event.user_id || event.anonymous_id,
    // ... map properties
  });
});
```

### BigQuery / Data Warehouse

```sql
-- Direct query from MongoDB or export to BigQuery
SELECT 
  event,
  COUNT(*) as count,
  COUNT(DISTINCT user_id) as unique_users
FROM analytics_events
WHERE ts >= CURRENT_DATE - 7
GROUP BY event
ORDER BY count DESC;
```

## Testing

### Local Testing

1. **Mobile**: Enable debug mode, verify events queued in AsyncStorage
2. **Backend**: Inspect `analytics_events` collection in MongoDB
3. **Integration**: Send test events, verify insertion, check duplicates handled

### Staging/Production

1. **Verification**: Query sample documents, verify envelope shape
2. **Counts**: Check event counts match expected usage patterns
3. **Failure Modes**: Test offline queue, retries, duplicate handling

### Testing Offline Queue

```typescript
// Disable network, track events
await analytics.track('test_event', { test: true });

// Check queue
const queue = await AsyncStorage.getItem('@meridian/analytics/queue');
console.log('Queued events:', JSON.parse(queue));

// Re-enable network, events should flush automatically
```

## Troubleshooting

### Events Not Appearing

1. Check if analytics is initialized (should happen automatically)
2. Check queue size (may be full, oldest events dropped)
3. Check network connectivity
4. Check backend logs for validation errors
5. Verify MongoDB indexes are created

### Duplicate Events

- Duplicates are handled gracefully (not inserted, counted in response)
- If seeing duplicates, check `event_id` generation (should be UUID)

### Queue Not Flushing

- Check if app state listener is working (background/foreground)
- Check if flush timer is running (45s interval)
- Manually flush: `await analytics.flush()`

### PII in Events

- Check client-side scrubbing (removes common PII keys)
- Check server-side scrubbing (also removes PII)
- Review event properties before tracking

## Best Practices

1. **Track meaningful events**: Don't track every click, focus on user actions
2. **Use consistent event names**: Follow the taxonomy (snake_case)
3. **Keep properties small**: Max 5KB per event
4. **No PII in properties**: Never include email, name, phone, etc.
5. **Use context overrides sparingly**: Default context is usually sufficient
6. **Test in dev first**: Verify events before deploying

## Migration Notes

The new analytics system is independent of the old manual tracking:
- Old routes (`/routes/analytics.js`) remain functional
- Old collections (`Visit`, `RepeatedVisit`, `Search`) remain unchanged
- Both systems can coexist during transition
- No data migration needed

## Support

For questions or issues:
1. Check this documentation
2. Review code comments in `analytics.ts` and `analyticsRoutes.js`
3. Check backend logs for validation errors
4. Inspect MongoDB collection for data issues

