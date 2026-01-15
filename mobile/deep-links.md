This guide explains how to use deep links for events, organizations, and rooms in Meridian notifications.

## Deep Link Formats

### Events
```
meridian://event/[eventId]
```
Example: `meridian://event/507f1f77bcf86cd799439011`

### Rooms/Spaces
```
meridian://room/[roomId]
```
Example: `meridian://room/507f1f77bcf86cd799439012`

### Organizations
```
meridian://organization/[orgId]
```
Example: `meridian://organization/507f1f77bcf86cd799439013`

### Friends
```
meridian://main/friends?initialTab=requests
```
Example: `meridian://main/friends?initialTab=requests`

## Using Deep Links in Notifications

### Option 1: Using Navigation Type "deep_link"

When creating a notification, set the navigation type to `deep_link`:

```javascript
await notificationService.createNotification({
  recipient: userId,
  recipientModel: 'User',
  title: 'New Event',
  message: 'Check out this event!',
  type: 'event',
  channels: ['push'],
  metadata: {
    eventId: '507f1f77bcf86cd799439011',
    navigation: {
      type: 'deep_link',
      deepLink: 'meridian://event/507f1f77bcf86cd799439011'
    }
  }
});
```

### Option 2: Using Navigation Type "navigate" (Recommended)

For more control, use `navigate` type with explicit route:

```javascript
await notificationService.createNotification({
  recipient: userId,
  recipientModel: 'User',
  title: 'New Event',
  message: 'Check out this event!',
  type: 'event',
  channels: ['push'],
  metadata: {
    eventId: '507f1f77bcf86cd799439011',
    navigation: {
      type: 'navigate',
      route: 'EventDetails',
      params: { eventId: '507f1f77bcf86cd799439011' },
      deepLink: 'meridian://event/507f1f77bcf86cd799439011'
    }
  }
});
```

### Option 3: Automatic Navigation (No explicit navigation needed)

If you include the resource ID in metadata, navigation is automatically built:

```javascript
await notificationService.createNotification({
  recipient: userId,
  recipientModel: 'User',
  title: 'New Event',
  message: 'Check out this event!',
  type: 'event',
  channels: ['push'],
  metadata: {
    eventId: '507f1f77bcf86cd799439011'
    // Navigation is automatically built based on type and eventId
  }
});
```

## Examples

### Event Notification

```javascript
await notificationService.createNotification({
  recipient: userId,
  recipientModel: 'User',
  title: 'Event Reminder',
  message: 'Your event starts in 15 minutes!',
  type: 'event_reminder',
  channels: ['push'],
  metadata: {
    eventId: '507f1f77bcf86cd799439011',
    navigation: {
      type: 'navigate',
      route: 'EventDetails',
      params: { eventId: '507f1f77bcf86cd799439011' },
      deepLink: 'meridian://event/507f1f77bcf86cd799439011'
    }
  }
});
```

### Organization Notification

```javascript
await notificationService.createNotification({
  recipient: userId,
  recipientModel: 'User',
  title: 'New Member Applied',
  message: 'John has applied to join your organization.',
  type: 'org',
  channels: ['push'],
  metadata: {
    orgId: '507f1f77bcf86cd799439013',
    navigation: {
      type: 'navigate',
      route: 'OrganizationProfile',
      params: { orgId: '507f1f77bcf86cd799439013' },
      deepLink: 'meridian://organization/507f1f77bcf86cd799439013'
    }
  }
});
```

### Room/Space Notification

```javascript
await notificationService.createNotification({
  recipient: userId,
  recipientModel: 'User',
  title: 'Room Available',
  message: 'DCC 308 is now available!',
  type: 'room_availability',
  channels: ['push'],
  metadata: {
    roomId: '507f1f77bcf86cd799439012',
    navigation: {
      type: 'navigate',
      route: 'RoomDetails',
      params: { roomId: '507f1f77bcf86cd799439012' },
      deepLink: 'meridian://room/507f1f77bcf86cd799439012'
    }
  }
});
```

## Using Templates

You can also use templates with variable interpolation:

```javascript
await notificationService.createSystemNotification(
  userId,
  'User',
  'event_reminder',
  {
    eventId: '507f1f77bcf86cd799439011',
    eventName: 'React Workshop',
    timeUntil: '15 minutes',
    startTime: new Date()
  }
);
```

The `event_reminder` template automatically includes navigation to the event.

## Testing Deep Links

### From Terminal (iOS Simulator)
```bash
xcrun simctl openurl booted "meridian://event/507f1f77bcf86cd799439011"
```

### From Terminal (Android Emulator)
```bash
adb shell am start -W -a android.intent.action.VIEW -d "meridian://event/507f1f77bcf86cd799439011"
```

### From Browser/Email
Create a link: `<a href="meridian://event/507f1f77bcf86cd799439011">Open Event</a>`

## Navigation Routes

The following routes are available for direct navigation:

- `EventDetails` - Event details screen (requires `eventId` param)
- `RoomDetails` - Room details screen (requires `roomId` param)
- `OrganizationProfile` - Organization profile screen (requires `orgId` param)
- `MainTabs` - Main tab navigator (use for tab screens like Friends)

## Notes

- Deep links work when the app is closed, in background, or foreground
- If the app is closed, it will open to the deep linked screen
- Deep links are automatically handled by React Navigation's linking system
- The `deepLink` field in navigation is optional but recommended for fallback handling

