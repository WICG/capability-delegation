# Explainer: Service Worker Capability Delegation for Pop-ups

## Authors
- dmurph@chromium.org

## Participate
- **Standards Venue**: [WICG / capability-delegation](https://github.com/WICG/capability-delegation)
- **Issue Tracker**: [GitHub Issues](https://github.com/WICG/capability-delegation/issues)
- **Public Issue**: [capability-delegation Issue 42](https://github.com/WICG/capability-delegation/issues/42)
- **Chromium Tracking Bug**: [crbug.com/542314185](https://crbug.com/542314185)

## Introduction
Many web APIs are gated by transient user activation to prevent abuse (such as unsolicited popups or spam windows). 

In the legacy, document-bound Notifications API (`new Notification()`), clicking a notification dispatches an `onclick` event directly in the originating `Window` context. Because this event runs on a `Window`, the browser automatically grants transient user activation to the page, allowing the click handler to call `window.open()` directly and share application state.

However, modern web applications and Progressive Web Apps (PWAs) rely on **Service Worker Notifications** (`registration.showNotification()`) to handle push notifications and background messaging when client windows may be closed or backgrounded. When a user interacts with a Service Worker notification, the `notificationclick` event is dispatched to the background Service Worker execution context, which lacks a DOM and cannot directly open windows with shared memory. 

Today, if a Service Worker handling a notification click wants an existing tab to open an auxiliary view (such as an email preview or chat popup) that shares live in-memory state and cache with the main window, the client window cannot call `window.open()` because it lacks user activation.

This proposal extends the **Capability Delegation** framework to bridge this gap, allowing Service Workers to receive transient capabilities upon user interaction events and delegate them to same-origin client windows.

Specifically:
1. Handling a trusted notification click event grants a **transient capability** (specifically `"popup"`) scoped to that `NotificationEvent` execution context.
2. The Service Worker can delegate this capability to a same-origin `WindowClient` via `client.postMessage(message, {delegate: 'popup'})`, consuming the capability from the event context.
3. Upon receiving the message, the client window acquires a transient delegated `"popup"` capability token.
4. Calling `window.open()` on the client is permitted if **either** transient user activation or the delegated `"popup"` capability is present, consuming **both** upon execution.

## Goals
- Allow Service Workers handling notification clicks to delegate the `"popup"` capability to a same-origin `WindowClient`, achieving parity with the capabilities of the legacy document-bound Notification API.
- Enable high-performance, state-sharing popup window flows (e.g., email preview, chat popouts) for web applications from notifications.
- Maintain a strict 1:1 invariant between user notification clicks and permitted popup windows (zero popup amplification).

## Non-Goals
- **Frame-to-Frame / Window-to-Iframe Popup Delegation**: Delegating popup capabilities across frames or to cross-origin iframes is intentionally excluded from this proposal to de-risk security reviews and avoid cross-origin popup blocker complexities (see [Future Considerations](#future-considerations-frame-to-frame-popup-delegation)).
- **General Worker User Activation**: Exposing general user activation states (such as `navigator.userActivation` or ambient transient activation) to Service Workers or Web Workers.
- **Cross-Origin SW Delegation**: Allowing Service Workers to delegate capabilities to cross-origin clients (SW delegation is strictly same-origin to its registered `WindowClient`s).
- **Direct Popups from Workers**: Allowing Service Workers to directly invoke `window.open()` or spawn DOM windows from background threads.

## Motivating User Scenario: Webmail Notification & State-Sharing Preview

1. A user receives a desktop notification for a new incoming email.
2. The user clicks the notification.
3. The browser dispatches the `notificationclick` event to the Service Worker, granting a transient `"popup"` capability to the event context.
4. The Service Worker identifies an existing, open webmail client tab and brings it to the foreground via `await client.focus()`.
5. The Service Worker posts a message to that client tab delegating the popup capability:
   ```javascript
   targetClient.postMessage({action: 'preview_email', emailId: 'msg_42'}, {delegate: 'popup'});
   ```
6. The webmail tab receives the message. Its `message` event handler calls `window.open()`:
   ```javascript
   const popup = window.open('/preview?id=' + event.data.emailId, 'preview_win', 'width=600,height=500');
   ```
7. The popup opens successfully. Because it was opened from the existing client window, it can synchronously access the main tab's in-memory data store, shared WebAssembly modules, active caches, and live state via `window.opener` without a full cold-start bootstrap.

## Proposed API

### Web IDL Changes

We introduce `ClientPostMessageOptions` for `Client.postMessage()`, mirroring `WindowPostMessageOptions`, allowing developers to specify `delegate: 'popup'` without polluting unrelated messaging channels:

```webidl
// In Service Workers specification:
dictionary ClientPostMessageOptions : StructuredSerializeOptions {
    DOMString? delegate;
};

partial interface Client {
    undefined postMessage(any message, optional ClientPostMessageOptions options = {});
};
```

### Capability Identifiers

Capability Delegation was originally specified using Permissions Policy feature identifiers (such as `payment`, `fullscreen`, `display-capture`). This proposal generalizes the delegation registry to also include activation-gated browser capabilities, introducing `"popup"` as the capability identifier gating `window.open()`.

### Delegation Channel Restrictions

For this proposal, capability delegation for `"popup"` is exclusively supported on:
- `Client.postMessage(message, options)` when targeting a same-origin `WindowClient` from a Service Worker.

Other worker messaging channels (`MessagePort`, `BroadcastChannel`, `Worker.postMessage()`, or `Client.postMessage()` targeting dedicated/shared workers) do not accept capability delegation options and will throw `NotSupportedError` if `delegate` is specified.

## Developer Usage Example

### Service Worker (`sw.js`):
```javascript
self.addEventListener('notificationclick', (event) => {
  event.waitUntil(async function() {
    // 1. Locate an open same-origin client window
    const clients = await self.clients.matchAll({type: 'window'});
    const client = clients.find(c => c.visibilityState === 'visible') || clients[0];
    
    if (client) {
      // 2. Focus the client window to bring it to foreground
      await client.focus();
      
      // 3. Delegate the 'popup' capability to the client window.
      // This consumes the transient capability bound to this notificationclick event.
      client.postMessage(
        {action: 'open_email_preview', emailId: event.notification.data.emailId}, 
        {delegate: 'popup'}
      );
    } else {
      // Fallback: If no client window is open, open a new top-level tab via Clients.openWindow
      await self.clients.openWindow('/inbox?open=' + event.notification.data.emailId);
    }
  }());
});
```

### Client Window (`main.js`):
```javascript
navigator.serviceWorker.addEventListener('message', (event) => {
  if (event.data.action === 'open_email_preview') {
    // Synchronously calling window.open() consumes the delegated 'popup' capability token
    const previewPopup = window.open(
      '/preview?id=' + event.data.emailId,
      'email_preview',
      'width=650,height=500'
    );
    
    if (!previewPopup) {
      console.warn('Popup blocked by browser settings or expired token.');
    }
  }
});
```

> [!TIP]
> **Async Data Fetching Pattern**: If the client window needs to fetch additional data before rendering, it should call `window.open('about:blank', ...)` **synchronously** in the `message` event handler to consume the 1-second delegated token, and subsequently set `popup.location.href` or populate the DOM once asynchronous data resolves. Waiting on `await fetch()` before calling `window.open()` risks token expiration.

## Key Execution Scenarios

### 1. Successful Delegation & Consumption
- User clicks notification $\rightarrow$ `NotificationEvent` execution context is granted a transient `"popup"` capability.
- SW calls `client.postMessage(msg, {delegate: 'popup'})` within the activation lifespan ($\le 5$s).
- The transient capability is consumed from the `NotificationEvent` context.
- Client window receives the message and sets a transient delegated `"popup"` capability timestamp (`kActivationLifespan` = 1s).
- Client window calls `window.open()` synchronously $\rightarrow$ Popup opens; both transient user activation (if present) and the delegated capability token are consumed.

### 2. Expired / Inactive SW Delegation
- SW attempts to call `client.postMessage(msg, {delegate: 'popup'})` outside of a trusted `notificationclick` event, or after the lifespan expires.
- The User Agent rejects the call by throwing a `NotAllowedError` DOMException.
- No capability token is attached or sent to the client.

### 3. Single-Use Token Invariant (Double-Use Prevention)
- Client receives a delegated `"popup"` token.
- Client attempts two consecutive `window.open()` calls:
  - First call succeeds and immediately clears the delegated capability token (and transient activation).
  - Second call finds no activation or capability token and is blocked by the popup blocker.

## Detailed Specification Integration

### 1. Event-Scoped Transient Capability Lifecycle on `NotificationEvent`
Rather than introducing ambient mutable state on `ServiceWorkerGlobalScope`, transient capabilities are tracked per-event execution context:
- When a trusted `notificationclick` event is dispatched:
  - The User Agent associates an active transient capability token for `"popup"` with that specific `NotificationEvent` instance and its `ExtendableEvent.waitUntil()` promise chain.
  - The capability lifespan is bounded by $\min(5000\text{ms}, \text{ExtendableEvent duration})$. Once the event handler completes, its `waitUntil` promise chain settles, or 5 seconds elapse without delegating, the token is invalidated.
  - Because tracking is scoped strictly to the `NotificationEvent`, concurrent background operations (such as incoming `push`, `sync`, or `fetch` events) cannot access or consume the capability.

### 2. Initiating Delegation (`Client.postMessage`)
In the algorithm for `Client.postMessage(message, options)`:
1. If `options["delegate"]` is present and not null:
   - If `options["delegate"]` is not `"popup"`, throw a `NotSupportedError` DOMException.
   - If the target client is not a `WindowClient`, throw a `NotSupportedError` DOMException.
   - Let `currentEvent` be the active event execution context of the caller.
   - If `currentEvent` is not a `NotificationEvent` with an active, unexpired transient capability token for `options["delegate"]`, throw a `NotAllowedError` DOMException.
   - **Consume the transient capability**: Invalidate/consume the capability token on `currentEvent`.
   - Attach the delegated capability identifier to the queued message event task.

### 3. Receiving Delegation (`WindowClient`) & Popunder Defenses
When the User Agent dispatches the `message` event task on the target client's `ServiceWorkerContainer` (`navigator.serviceWorker`):
- If the message task contains a delegated capability `"popup"`:
  - Set `DELEGATED_CAPABILITY_TIMESTAMPS["popup"]` on the target `Window` to the current high-resolution time (lifespan of 1 second).
- **Popunder Defense**: When the client window consumes the token via `window.open()`, the User Agent popup blocker enforces standard window focus and visibility policies (e.g., verifying `document.visibilityState === 'visible'`). Spawning background popunders from occluded or minimized tabs remains blocked per User Agent security policy.

### 4. Monkey-Patch to `window.open()` (Popup Blocker Verification & Unified Consumption)
In the HTML specification's window open steps:
1. **Sandbox Check**: If the calling browsing context is a sandboxed iframe without `allow-popups`, immediately block the popup. Capability delegation never overrides sandbox restrictions.
2. **Popup Allowed Check**: Verify if:
   - The relevant global object has active [transient user activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation), **OR**
   - `DELEGATED_CAPABILITY_TIMESTAMPS["popup"]` in the relevant global object is present and not [expired](https://html.spec.whatwg.org/multipage/interaction.html#activation-expiry).
3. If neither condition is true, block the popup per standard popup blocker behavior.
4. **Unified Consumption**:
   - If transient user activation is present on the global object, [consume user activation](https://html.spec.whatwg.org/multipage/interaction.html#consume-user-activation).
   - If `DELEGATED_CAPABILITY_TIMESTAMPS["popup"]` is present, clear/remove the `"popup"` entry from `DELEGATED_CAPABILITY_TIMESTAMPS`.
   *(Consuming both guarantees that no residual activation or capability tokens linger for subsequent calls).*

## Security & Privacy Considerations

### Security & Abuse Mitigations

1. **Same-Origin Invariant**:
   - Service Workers can only manage and communicate with same-origin clients. Delegation cannot cross origin boundaries.
2. **Strict 1:1 Gesture Invariant**:
   - One user notification click $\rightarrow$ exactly one transient capability granted $\rightarrow$ exactly one `postMessage` delegation allowed $\rightarrow$ exactly one `window.open()` permitted.
   - There is zero opportunity for popup amplification or looping.
3. **Double-Sided Lifespan Bounds**:
   - **Sender**: SW transient capability expires in $\le 5$ seconds within the `NotificationEvent`.
   - **Receiver**: Client window token expires in 1 second.
4. **No Ambient Worker Activation & Concurrency Isolation**:
   - Because capabilities are scoped strictly to the `NotificationEvent` rather than ambient global state on the worker, other background operations (`push`, `fetch`) cannot hijack or consume the capability.
5. **Sandbox Invariant**:
   - Sandboxed contexts without `allow-popups` remain strictly prohibited from opening popups, even if receiving a delegated capability.
6. **Popunder & Focus Mitigations**:
   - Delegating to a client tab does not allow silent background popunders. The Service Worker brings the window to focus with `await client.focus()`, and browser popup blockers continue to enforce foreground visibility checks on `window.open()`.

### Privacy & Fingerprinting
- **No Persistent State or Identifiers**: Capability tokens and timestamps are ephemeral, stored strictly in memory for at most a few seconds, and wiped upon consumption or expiry. They introduce no persistent storage, tracking identifiers, or cross-origin leakage vectors.
- **Partitioning & Storage Boundaries**: Service Worker registration and client matching strictly adhere to standard third-party storage partitioning and origin boundaries.


## Accessibility (A11y) Considerations

- **Predictable Focus Transitions**: The recommended developer pattern pairs `await client.focus()` with `window.open()`. This ensures that operating system window focus moves deliberately from the notification interaction to the foreground client tab and subsequently to the newly created auxiliary popup window, providing a continuous, predictable experience for screen readers and keyboard navigation.
- **No Disorienting Background Popups**: Popups can only be spawned directly as a result of explicit user interaction with a desktop notification, preventing sudden or unprompted focus shifts while the user is interacting with other applications.


## Internationalization (i18n) Considerations

- **Standard Identifiers & Encodings**: The capability identifier `"popup"` is a standard lowercase ASCII token conforming to established web platform conventions.
- **Payload Encoding**: Data passed through `Client.postMessage()` utilizes the standard Structured Clone algorithm with full Unicode support, introducing no language- or locale-specific limitations.

## Stakeholder Feedback & Implementation Signals

- **W3C / WHATWG Standards Venue**: Proposed as an extension to [WICG Capability Delegation](https://wicg.github.io/capability-delegation/spec.html) in collaboration with WHATWG (HTML `window.open` and messaging) and W3C WebApps (Service Workers).
- **Chromium / Blink**: Positive / Prototyping ([crbug.com/542314185](https://crbug.com/542314185)).
- **Gecko / Mozilla**: Pending review / standards position request.
- **WebKit / Apple**: Pending review / standards position request.
- **Web Developers**: Strong demand from major web application developers (e.g. email, chat, and productivity suites) needing low-latency, state-sharing popup windows from notification clicks without full-page reloads.


## Alternatives Considered

### 1. User Activation on `ServiceWorkerGlobalScope`

* **Alternative Concept**: Extend the [HTML User Activation Data Model](https://html.spec.whatwg.org/multipage/interaction.html#user-activation-data-model) to `ServiceWorkerGlobalScope`. When a trusted user event (such as `notificationclick`) is dispatched, the Service Worker would be granted standard transient user activation. The Service Worker could then consume this activation to delegate it via `postMessage()`, or user activation could propagate to a client window.
* **Why Rejected / Why Transient Capabilities are Preferable**:
  * **HTML Spec Coupling**: User activation in the HTML standard is fundamentally defined for `Window` objects and browsing contexts (tied to `Navigable`s and document trees). Web Workers and Service Workers have never had user activation. Introducing user activation to workers would require significant, invasive modifications to the core HTML specification.
  * **Spec Ambiguities & Scope Creep**: Bringing user activation to worker scopes raises challenging questions:
    * Should `navigator.userActivation` (exposing `hasBeenActive` and `isActive`) exist on `WorkerNavigator`?
    * Does sticky user activation persist across the Service Worker lifecycle, or does it reset when the worker terminates?
    * Does worker user activation inadvertently unlock other activation-gated APIs (e.g., Clipboard, Web Bluetooth, Fullscreen, Media Playback) in background worker threads where user intent cannot be visually verified?
  * **Risk of Ambient Activation in Concurrency**: Ambient user activation stored on `ServiceWorkerGlobalScope` could be inadvertently intercepted or consumed by concurrent, untrusted background event handlers (e.g., incoming `push` or `fetch` events) executing in the same worker.
  * **Direct Alignment with Capability Delegation**: Modeling the `notificationclick` allowance as an event-scoped **transient capability** (specifically `"popup"`) allows us to leverage the existing Capability Delegation lifecycle without modifying the fundamental HTML User Activation architecture. The capability is scoped, single-use, feature-specific, and has zero side-effects on other web platform APIs.

### 2. `Clients.openWindow()`
- *Why Insufficient*: `Clients.openWindow()` creates an isolated top-level tab/window from the background worker. It cannot return a synchronous DOM `WindowProxy` to an existing client tab. Without a synchronous `WindowProxy`, web apps cannot share in-memory JavaScript state, caches, or state trees, forcing an expensive cold-start app reload. Furthermore, `Clients.openWindow()` cannot configure popup positioning or popup window features (`width`, `height`, minimal chrome).

### 3. Implicit Activation Propagation on `client.focus()`
- *Why Rejected*: Automatically granting full user activation whenever `client.focus()` is called is overly broad. It creates security risks by exposing unrestricted user activation to the client page, unlocking APIs unrelated to the notification click. Capability delegation requires explicit intent and is tightly restricted to the `"popup"` capability.

## Future Considerations: Frame-to-Frame Popup Delegation

In earlier design discussions, delegating the `"popup"` capability from a top-level `Window` to a cross-origin `iframe` (e.g., allowing a top-level merchant page to delegate popup opening to an embedded payment/auth provider) was considered.

While theoretically aligned with the generic Capability Delegation framework, frame-to-frame popup delegation introduces additional security and policy considerations around cross-origin iframe popup blockers, clickjacking defenses, and cross-site user activation transfer. To keep this proposal de-risked and focused on the immediate developer need in Service Workers, frame-to-frame popup delegation is deferred as a potential future extension.

## References & Prior Discussion

- **Chromium Issue**: [crbug.com/542314185](https://crbug.com/542314185)
- **Public Issue**: [capability-delegation Issue 42](https://github.com/WICG/capability-delegation/issues/42)
- **WICG Capability Delegation Specification**: https://wicg.github.io/capability-delegation/spec.html
- **HTML Window Open Steps**: https://html.spec.whatwg.org/multipage/window-object.html#dom-open
