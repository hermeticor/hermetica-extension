# @hermeticor/sdk-extension

Hermeticor in a browser extension. Records what people do in a Chrome, Edge or Firefox extension (Manifest V3). It runs in the background service worker, with no DOM, no localStorage and no beacons. The anonymous id, consent and unsent events are kept in `chrome.storage.local`, so nothing is lost when the browser stops the worker.

1. Create a publishable key that lists your extension (from your server or setup script, with a key that has `events:write`):
   `POST /v1/api-keys {"name":"My extension","kind":"site","allowedOrigins":["chrome-extension://<extension id>"]}`
2. Add `"storage"` to `permissions` in `manifest.json`.
3. In the background service worker:

```ts
import { createExtensionClient, relayTracking } from '@hermeticor/sdk-extension';

export const hm = createExtensionClient({ siteKey: 'hm_pk_...' });
relayTracking(hm); // accept events from this extension's content scripts and popup

// after the person agrees in your onboarding page:
await hm.consent.update({ analytics: true });
await hm.track('saved_item', { sku: 'A1' });
await hm.identify('user_123', { email: 'kim@example.com' }); // needs { marketing: true } consent too
```

4. In a content script or the popup (no key needed there):

```ts
import { trackFromContent } from '@hermeticor/sdk-extension';
await trackFromContent('clicked_buy', { site: location.hostname });
```

Content scripts run inside the web page, so a request they made themselves would carry the page's origin, which the key refuses. `trackFromContent` hands the event to the background instead, and the background accepts it only from your own extension. The default `consentMode` is `require-consent`: nothing is stored or sent until `consent.update({ analytics: true })`. Pass `consentMode: 'none'` only if your consent basis allows it. Events wait in storage while offline, are sent in batches of 10.4.150.4.15, are retried after a 429 or 5xx, and are never sent twice.

