[Guides-and-Surveys-with-Segment-Integration-Guide.md](https://github.com/user-attachments/files/23689510/Guides-and-Surveys-with-Segment-Integration-Guide.md)
# Amplitude Guides & Surveys with Segment Integration
## Single HTML File Implementation Guide

This guide covers integrating Amplitude Guides & Surveys when using **Segment as your analytics source** in a single HTML file (no npm/build process).

---

## Architecture

- **Segment Analytics**: Source of truth for event tracking
- **Amplitude Session Replay**: Standalone SDK for session recording
- **Amplitude Guides & Surveys**: Via script loader for in-app messaging
- **Amplitude Experiment** (optional): For A/B testing guide variations

---

## Critical Integration Steps

### 1. Load SDKs in Correct Order

```html
<!-- Segment Analytics -->
<script>
  !function(){var i="analytics",analytics=window[i]=window[i]||[];
  // ... Segment snippet ...
  analytics._writeKey="YOUR_SEGMENT_WRITE_KEY";
  analytics.SNIPPET_VERSION="5.2.0";
  }}();
</script>

<!-- Session Replay Standalone -->
<script src="https://cdn.amplitude.com/libs/session-replay-browser-1.20.1-min.js.gz"></script>
```

### 2. Initialize After User Login (CRITICAL!)

**Why**: Guides & Surveys needs user identity to work. Initializing before login causes `user: null` and guides won't show in live mode.

```javascript
// After user logs in:
async function initializeAnalytics() {
    // Load Segment
    analytics.load(SEGMENT_WRITE_KEY);
    analytics.identify(userId, userProperties);

    analytics.ready(async function() {
        // 1. Get device ID from Segment (with retry logic)
        let deviceId = null;
        let attempts = 0;
        while (!deviceId && attempts < 5) {
            if (analytics.user) {
                deviceId = analytics.user().anonymousId();
            }
            if (!deviceId) {
                await new Promise(resolve => setTimeout(resolve, 500));
                attempts++;
            }
        }

        // 2. Initialize Session Replay with Segment's device ID
        await sessionReplay.init(AMPLITUDE_API_KEY, {
            sessionId: Date.now(),
            deviceId: deviceId,
            sampleRate: 1
        }).promise;

        // 3. Initialize Guides & Surveys via script loader
        await loadGuidesAndSurveys(deviceId);
    });
}
```

### 3. Guides & Surveys Integration (Script Loader Method)

**Important**: For single HTML files, use **script loader** NOT the standalone library. The standalone `@amplitude/engagement-browser` is only available via npm.

```javascript
async function loadGuidesAndSurveys(deviceId) {
    // Load script loader (embeds API key, auto-calls init())
    const script = document.createElement('script');
    script.src = `https://cdn.amplitude.com/script/${AMPLITUDE_API_KEY}.engagement.js`;
    script.async = false;
    document.head.appendChild(script);
    
    // Wait for SDK to load
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    if (window.engagement) {
        // Get user data safely
        let userId = currentUser.userId;
        let userDeviceId = deviceId; // From outer scope
        
        if (typeof analytics.user === 'function') {
            const segmentUser = analytics.user();
            userId = segmentUser.id() || userId;
            userDeviceId = segmentUser.anonymousId() || deviceId;
        }
        
        // Boot with Segment identity
        await window.engagement.boot({
            user: {
                user_id: userId,
                device_id: userDeviceId
            },
            user_properties: {
                email: currentUser.email,
                name: currentUser.name,
                plan: currentUser.plan
            },
            integrations: [{
                // Send G&S events to Segment → Amplitude
                track: (event) => {
                    analytics.track(event.event_type, event.event_properties);
                }
            }]
        });

        // Forward Segment events to G&S (enables event-based triggers)
        analytics.on('track', (event, properties) => {
            window.engagement?.forwardEvent?.({
                event_type: event,
                event_properties: properties
            });
        });

        analytics.on('page', (name, properties) => {
            window.engagement?.forwardEvent?.({
                event_type: 'Page Viewed',
                event_properties: { name, ...properties }
            });
        });
    }
}
```

---

## Critical Gotchas

### ❌ Common Mistake #1: Using Standalone Library via CDN
```javascript
// ❌ DOESN'T WORK - library not available via CDN
<script src="https://cdn.amplitude.com/libs/engagement-browser-1.1.0-min.js.gz"></script>
```

### ✅ Correct: Use Script Loader
```javascript
// ✅ WORKS - script loader available via CDN
<script src="https://cdn.amplitude.com/script/${API_KEY}.engagement.js"></script>
```

### ❌ Common Mistake #2: Initializing Before Login
```javascript
// ❌ Guides won't show - boots with user: null
initializeAnalytics(); // Called on page load
// ... user logs in later
```

### ✅ Correct: Initialize After Login
```javascript
// ✅ Guides will show - boots with actual user
// ... user logs in first
initializeAnalytics(); // Called after login
```

### ❌ Common Mistake #3: Circular Variable Reference
```javascript
// ❌ ReferenceError: Cannot access before initialization
let deviceId = deviceId;
```

### ✅ Correct: Use Different Variable Name
```javascript
// ✅ Works - references outer scope
let userDeviceId = deviceId;
```

---

## Validation

### Check Console Logs
```javascript
// Should see:
✅ Retrieved device ID from Segment localStorage: [device-id]
✅ Session Replay initialized successfully!
✅ Engagement SDK script loader loaded
✅ Amplitude Guides & Surveys initialized!
   - User ID: [user-id]
   - Device ID: [device-id]
```

### Debug Status
```javascript
// In browser console:
window.engagement._debugStatus()

// Should return:
{
  "user": { "user_id": "...", "device_id": "..." },
  "stateInitialized": true,      // ✅ Must be true
  "decideSuccessful": true,       // ✅ Must be true
  "num_guides_surveys": 2,        // ✅ Should be > 0
  "analyticsIntegrations": 1
}
```

---

## Event Flow

```
User Action
    ↓
Segment Analytics tracks event
    ↓
analytics.on('track') listener
    ↓
window.engagement.forwardEvent()
    ↓
Guide triggers (if conditions met)
    ↓
Guide lifecycle events (Viewed, Dismissed, etc.)
    ↓
integrations[0].track()
    ↓
Back to Segment → Amplitude
```

---

## Key Takeaways

1. **Script Loader Only**: For single HTML files without npm
2. **Login First**: Initialize analytics AFTER user authentication
3. **Device ID Consistency**: Use Segment's `anonymousId()` everywhere
4. **Event Forwarding**: Required for event-based guide triggers
5. **Integration Array**: Sends G&S events back to Segment

---

## Troubleshooting

### Guides show in Preview but not Live?
- Check `window.engagement._debugStatus()`
- `decideSuccessful: false` = SDK can't reach Amplitude
- `user: null` = Initialized before login
- `num_guides_surveys: 0` = No guides published or targeting doesn't match

### Events not tracking to Amplitude?
- Verify `integrations` array in `boot()` call
- Check Segment debugger for G&S events
- Confirm Segment → Amplitude (Actions) destination is enabled

### Device ID mismatch between SDKs?
- Always use `analytics.user().anonymousId()` from Segment
- Never generate your own device ID
- Implement retry logic to wait for Segment's device ID

---

**Result**: Fully functional Guides & Surveys with Segment as source, Session Replay, and clean event taxonomy ready for workshop demonstrations!

