---
layout: post
title: "FCM Web Push Notifications for PWAs — The Guide Nobody Wrote"
date: 2026-02-17
---

I didn't write the code that fixed this. Claude did. But I spent 8 hours in the problem — reading error messages, describing what was broken, testing fixes, hitting the next wall. AI-augmented doesn't mean painless. It means the suffering is architectural instead of syntactic.

This guide documents every failure mode we hit. "We" being me and an AI that doesn't get tired.

---

## Platform Support Matrix

| Device | Browser | Works |
|--------|---------|-------|
| Mac | Chrome | ✅ |
| Mac | Safari | ✅ |
| Windows | Chrome | ✅ |
| Windows | Edge | ✅ |
| iPhone | Safari PWA | ✅ |
| iPhone | Chrome | ✅ |
| Android | Chrome | ✅ |
| iPad | Any | ❌ |

iPad throws `messaging/unsupported-browser` from the Firebase SDK regardless of browser or PWA installation. This is an Apple/iPadOS limitation, not a fixable code issue. Use email reminders as fallback for iPad users.

---

## Architecture Overview
``
Browser → FCM Token → Firestore → Cloud Function (scheduled) → FCM → Browser SW → Notification
```

Three moving parts:
1. **Frontend** — registers service worker, gets FCM token, saves to Firestore
2. **Firestore** — stores tokens per user, rules must allow session-independent writes
3. **Cloud Function** — scheduled job reads tokens, sends via Admin SDK

All three must be correct. A failure in any one is completely silent by default.

---

## The Mistakes That Will Destroy You

### 1. Anonymous Auth as Ownership Marker

If your Firestore rules use anonymous auth UIDs to restrict writes:
```javascript
// THIS WILL SILENTLY BREAK EVERYTHING
allow update: if isSignedIn() && request.auth.uid == resource.data.userId;
```

Anonymous auth generates a **new UID every browser session**. After any page reload, the UID changes, the rule fails, and token saves silently return `false`. Your tokens never persist. The Cloud Function reads empty token arrays. FCM returns `NotRegistered`. Your cleanup code wipes the tokens. The user re-enables push. Repeat forever.

**The fix** — if your auth model uses access codes or similar session-independent identifiers, relax the update rule:
```javascript
// Access code knowledge IS the authorization
allow update: if isSignedIn();
```

### 2. Calling deleteToken() Before getToken()

Don't do this on every registration:
```javascript
await deleteToken(messaging);  // generates fresh token every time
const token = await getToken(messaging, { ... });
```

FCM issues a new token every time you call `getToken()` after `deleteToken()`. If you're deduplicating tokens by value, fresh tokens always pass the filter and your array grows forever with stale entries.

**The fix** — reuse the existing SW registration if one exists:
```javascript
const existingRegs = await navigator.serviceWorker.getRegistrations();
let registration = existingRegs.find(r =>
  r.scope.includes('firebase-cloud-messaging-push-scope')
);

if (registration) {
  const existingSub = await registration.pushManager.getSubscription();
  if (!existingSub) {
    await registration.unregister();
    registration = undefined;
  }
}
```

### 3. Deduplicating Tokens by Value Instead of Device
```javascript
// WRONG — new token every registration, filter never removes anything
const existing = tokens.filter(d => d.token !== result.token);

// CORRECT — one token per device name
const existing = tokens.filter(d => d.device !== result.device);
```

### 4. Service Worker at the Wrong Scope
```javascript
// CORRECT
const registration = await navigator.serviceWorker.register(
  '/firebase-messaging-sw.js',
  { scope: '/firebase-cloud-messaging-push-scope' }
);

const token = await getToken(messaging, {
  vapidKey: VAPID_KEY,
  serviceWorkerRegistration: registration,
});
```

If you let Firebase auto-register and also manually register, you end up with two SWs. The push subscription binds to one, the FCM token binds to the other. Messages arrive at the wrong SW and are silently dropped.

### 5. FCM send() Success Does Not Mean Delivery
```javascript
await getMessaging().send({ token, notification: { ... } });
// Returns a message ID — means FCM *accepted* it, NOT that it was delivered
```

There is no delivery callback. If the token is stale, FCM returns `NotRegistered`. Test with a device in hand.

### 6. Firestore Query Constraints Are Enforced Client-Side
```javascript
// This will fail with "insufficient permissions"
const q = query(collection(db, 'users'), where('email', '==', email));

// This works
const q = query(collection(db, 'users'), where('email', '==', email), limit(1));
```

---

## Multi-Device Token Schema
```typescript
interface FcmDevice {
  token: string;
  device: string;
  createdAt: string;
}

notifications: {
  pushEnabled: boolean;
  fcmTokens: FcmDevice[];
  fcmToken: string | null;  // Legacy — keep for backwards compat
}
```

---

## Device Detection

iPad Safari PWA reports its user agent as `Macintosh`. Use `navigator.maxTouchPoints` to distinguish from a real Mac.
```typescript
function getDeviceName(): string {
  const ua = navigator.userAgent;
  if (/iPad/.test(ua)) return 'iPad';
  if (/iPhone/.test(ua)) return 'iPhone';
  if (/Android/.test(ua)) return 'Android';
  if (/Macintosh/.test(ua) && navigator.maxTouchPoints > 1) return 'iPad';
  if (/Macintosh/.test(ua)) return 'Mac';
  if (/Windows/.test(ua)) return 'Windows';
  if (/Linux/.test(ua)) return 'Linux';
  return 'Unknown';
}
```

Block FCM registration on iPad before the permission prompt fires:
```typescript
const isIPad = /iPad/.test(ua) || (/Macintosh/.test(ua) && navigator.maxTouchPoints > 1);
if (isIPad) {
  return { error: 'Push notifications are not supported on iPad. Use email reminders instead.', code: 'unsupported-browser' };
}
```

---

## Cloud Function — Multi-Device Send with Auto-Cleanup
```javascript
const invalidTokens = [];

for (const entry of tokens) {
  try {
    await getMessaging().send({
      token: entry.token,
      notification: { title, body },
      webpush: { fcmOptions: { link: '/journal' } },
    });
  } catch (e) {
    if (
      e.code === 'messaging/invalid-registration-token' ||
      e.code === 'messaging/registration-token-not-registered'
    ) {
      invalidTokens.push(entry.token);
    }
  }
}

if (invalidTokens.length > 0) {
  const validTokens = tokens.filter(t => !invalidTokens.includes(t.token));
  await db.collection('profiles').doc(profileId).update({
    'notifications.fcmTokens': validTokens,
    'notifications.fcmToken': validTokens[0]?.token || null,
    ...(validTokens.length === 0 ? { 'notifications.pushEnabled': false } : {}),
  });
}
```

---

## Diagnostic Commands

### Check stored tokens:
```bash
node -e "
const admin = require('firebase-admin');
admin.initializeApp();
admin.firestore().collection('YOUR_COLLECTION').doc('YOUR_DOC_ID').get().then(d => {
  const n = d.data().notifications;
  console.log('pushEnabled:', n.pushEnabled);
  console.log('fcmTokens:', JSON.stringify(n.fcmTokens, null, 2));
}).then(() => process.exit(0));
"
```

### Test send to all tokens:
```bash
node -e "
const admin = require('firebase-admin');
admin.initializeApp();
admin.firestore().collection('YOUR_COLLECTION').doc('YOUR_DOC_ID').get().then(d => {
  const tokens = d.data().notifications.fcmTokens;
  return Promise.all(tokens.map(entry =>
    admin.messaging().send({ token: entry.token, notification: { title: 'Test', body: 'Test' } })
    .then(id => console.log('✅ VALID:', entry.device))
    .catch(e => console.log('❌ INVALID:', entry.device, e.code))
  ));
}).then(() => process.exit(0));
"
```

### Check browser SW state (DevTools console):
```javascript
navigator.serviceWorker.getRegistrations().then(regs => regs.forEach(r => {
  r.pushManager.getSubscription().then(s => {
    console.log('Scope:', r.scope, '| Push:', s ? 'YES' : 'NO');
  });
}));
```

---

## The Death Spiral to Recognize

If you see this cycle, the root cause is almost always Firestore permissions silently failing:

1. User enables push → token saves → ✅ appears to work
2. User reloads page → anonymous UID changes → subsequent saves fail silently
3. Cloud Function reads stale/empty token → FCM returns `NotRegistered`
4. Cleanup code sets `fcmTokens: []` and `pushEnabled: false`
5. User re-enables push → new token → silent save failure → back to step 3

**Check the browser console for `Missing or insufficient permissions` before chasing anything else.**

---

## iOS / iPhone Notes

- Requires iOS 16.4+
- Must be installed to Home Screen from Safari
- User must grant notification permission twice — once in the app, once in iOS system prompt
- iOS simulators do not support push — test with device in hand

## iPad — Why It Doesn't Work

Not fixable. Use email reminders as fallback. Detect early with `maxTouchPoints > 1` before calling `Notification.requestPermission()`.

---

## firebase-messaging-sw.js Minimum Viable File
```javascript
importScripts('https://www.gstatic.com/firebasejs/10.14.1/firebase-app-compat.js');
importScripts('https://www.gstatic.com/firebasejs/10.14.1/firebase-messaging-compat.js');

firebase.initializeApp({
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_AUTH_DOMAIN",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_STORAGE_BUCKET",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
});

const messaging = firebase.messaging();

messaging.onBackgroundMessage(function(payload) {
  self.registration.showNotification(
    payload.notification.title,
    { body: payload.notification.body }
  );
});
```

---

## Summary Checklist

- [ ] Firestore rules allow token saves without UID ownership check
- [ ] SW registered at `/firebase-cloud-messaging-push-scope`
- [ ] SW registration reused on re-registration (no deleteToken loop)
- [ ] Token deduplication by device name, not token value
- [ ] Cloud Function loops through `fcmTokens` array, not single `fcmToken`
- [ ] Cloud Function auto-cleans `NotRegistered` tokens
- [ ] Notification deduplication log in Firestore to prevent double-sends
- [ ] iPad detected via `maxTouchPoints > 1` before permission prompt fires
- [ ] Email reminder fallback for unsupported devices
