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

**Service Worker Notifications** (`registration.showNotification()`) are intended to be a more featured and efficient way to handle push notifications and background messaging when client windows may be closed or backgrounded. When a user interacts with a Service Worker notification, the `notificationclick` event is dispatched to the background Service Worker execution context, which lacks a DOM and cannot directly open windows with shared memory. 

Today, a service worker push notification **cannot** result in a tab opening an auxiliary view (AKA `window.open()`). This is because it lacks user activation. However, this is the current & efficient way that certain webapps, like gmail, open quick compose or preview popup windows in response to notification clicks.

This proposal explores extending the **Capability Delegation** framework to bridge this gap, allowing Service Workers to delegate the ability to open a popup window to a same-origin client window upon handling user interaction events.

## Goals
- Allow a Service Worker `notificationclick` event to result in a popup window created from an existing `WindowClient` (achieving parity with legacy document-bound Notification API).
- Maintain a strict 1:1 invariant between user notification clicks and permitted popup windows (zero popup amplification).

## Non-Goals
- **Fixing General Transient Activation Specification for Existing Service Worker APIs**: Currently, web specifications have a recognized gap regarding transient user activation in Service Workers. Certain APIs like `WindowClient.focus()` and `Clients.openWindow()` normatively require user activation in standard window contexts, but in Service Workers specifications historically relied on stop-gap prose (e.g., advising implementers to allow `focus()` or `openWindow()` during notification click events without defining a formal user activation model for workers). It is **not** a goal of this proposal to holistically resolve or re-specify general transient activation for those existing worker APIs; this proposal focuses narrowly on enabling popup creation from existing client windows. This should be fixed in a separate effort.
- **Frame-to-Frame / Window-to-Iframe Popup Delegation**: Allowing one frame, which currently can create an aux popup window, to delegate this ability to another frame (either an iframe or a separate document tab). See [Future Considerations](#future-considerations-frame-to-frame-popup-delegation).
- **Cross-Origin Support**: The document is always same-origin as the service worker.


## Motivating User Scenario: Webmail Notification & State-Sharing Preview

1. A user receives a desktop notification for a new incoming email.
2. The user clicks the notification.
3. The browser dispatches the `notificationclick` event to the Service Worker.
4. The Service Worker identifies an existing, open webmail client tab and brings it to the foreground via `await client.focus()`.
5. The Service Worker posts a message to that client tab for it to open the preview window (this is without the proposal):
   ```javascript
   targetClient.postMessage({action: 'preview_email', emailId: 'msg_42'});
   ```
6. The webmail tab receives the message. Its `message` event handler calls `window.open()`:
   ```javascript
   const popup = window.open('/preview?id=' + event.data.emailId, 'preview_win', 'width=600,height=500');
   ```

**Desired Outcome:** The popup opens successfully, querying the parent frame for the email contents and other shared data.

**Actual Outcome:** The popup is blocked by the popup blocker, as the frame did not have transient activation

## Proposed API

This proposal extends the [Capability Delegation](https://wicg.github.io/capability-delegation/spec.html) framework to allow Service Workers to explicitly delegate the `"popup"` capability to a same-origin `WindowClient`:

1. **Event-Scoped Transient Capability**: Handling a trusted `notificationclick` event grants a transient capability token for `"popup"` scoped strictly to that `NotificationEvent` execution context.
2. **Explicit Delegation via `postMessage`**: The Service Worker delegates this capability to a targeted same-origin `WindowClient` by passing `{delegate: 'popup'}` in `client.postMessage()`. This consumes the capability from the `NotificationEvent` context.
3. **Receipt of Delegated Capability**: Upon dispatching the `message` event on `navigator.serviceWorker`, the target client window receives a transient delegated `"popup"` capability token (with a 1-second lifespan).
4. **Popup Gating & Consumption**: Calling `window.open()` in the client window succeeds if either transient user activation or the delegated `"popup"` capability is present. Executing `window.open()` consumes both the activation and the capability token, maintaining a strict 1:1 invariant between user clicks and opened popups.

### Web IDL Changes

We introduce `ClientPostMessageOptions` for [`Client.postMessage()`](https://w3c.github.io/ServiceWorker/#dom-client-postmessage), mirroring [`WindowPostMessageOptions`](https://html.spec.whatwg.org/multipage/nav-history-apis.html#windowpostmessageoptions) and inheriting from [`StructuredSerializeOptions`](https://html.spec.whatwg.org/multipage/structured-data.html#structuredserializeoptions), allowing developers to specify `delegate: 'popup'` without polluting unrelated messaging channels:

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
- User clicks notification → `NotificationEvent` execution context is granted a transient `"popup"` capability.
- SW calls `client.postMessage(msg, {delegate: 'popup'})` within the activation lifespan (≤ 5s).
- The transient capability is consumed from the `NotificationEvent` context.
- Client window receives the message and sets a transient delegated `"popup"` capability timestamp (`kActivationLifespan` = 1s).
- Client window calls `window.open()` synchronously → Popup opens; both transient user activation (if present) and the delegated capability token are consumed.

### 2. Expired / Inactive SW Delegation
- SW attempts to call `client.postMessage(msg, {delegate: 'popup'})` outside of a trusted `notificationclick` event, or after the lifespan expires.
- The User Agent rejects the call by throwing a `NotAllowedError` DOMException.
- No capability token is attached or sent to the client.

### 3. Single-Use Token Invariant (Double-Use Prevention)
- Client receives a delegated `"popup"` token.
- Client attempts two consecutive `window.open()` calls:
  - First call succeeds and immediately clears the delegated capability token (and transient activation).
  - Second call finds no activation or capability token and is blocked by the popup blocker.

## Specification Changes

### 1. WHATWG Notifications Standard
Grants an ephemeral `"popup"` capability token scoped to the `NotificationEvent` execution context when a user interacts with a notification.

- **[Firing a service worker notification event](https://notifications.spec.whatwg.org/#fire-a-service-worker-notification-event)**:
  - When dispatching a trusted `notificationclick` event, associate an active transient capability token for `"popup"` with the [`NotificationEvent`](https://notifications.spec.whatwg.org/#notificationevent) instance (expiring upon [`ExtendableEvent.waitUntil()`](https://w3c.github.io/ServiceWorker/#dom-extendableevent-waituntil) settlement or after 5 seconds).

### 2. W3C Service Workers Standard
Extends `Client.postMessage()` to accept a capability delegation option, validating and transferring the transient token from the worker to the recipient client window.

- **Web IDL**: Add `ClientPostMessageOptions` inheriting from [`StructuredSerializeOptions`](https://html.spec.whatwg.org/multipage/structured-data.html#structuredserializeoptions) with `DOMString? delegate`.
- **[`Client.postMessage(message, options)`](https://w3c.github.io/ServiceWorker/#dom-client-postmessage)**:
  - If `options.delegate === "popup"`:
    1. If the target client is not a [`WindowClient`](https://w3c.github.io/ServiceWorker/#windowclient-interface), throw a `NotSupportedError`.
    2. If the current execution context is not a [`NotificationEvent`](https://notifications.spec.whatwg.org/#notificationevent) with an active `"popup"` token, throw a `NotAllowedError`.
    3. Consume the token from the event context and attach `"popup"` to the queued message event task.
- **[Service worker client message event task](https://w3c.github.io/ServiceWorker/#service-worker-container-message-event)**:
  - When dispatching on [`ServiceWorkerContainer`](https://w3c.github.io/ServiceWorker/#serviceworkercontainer-interface) (`navigator.serviceWorker`), if the task contains `"popup"`, set [`DELEGATED_CAPABILITY_TIMESTAMPS["popup"]`](https://wicg.github.io/capability-delegation/spec.html#tracking-delegation) on the target `Window` to the [current high resolution time](https://w3c.github.io/hr-time/#dfn-current-high-resolution-time) (1-second lifespan).

### 3. WHATWG HTML Standard
Updates popup blocker verification in the navigable creation steps to permit window creation if an unexpired delegated `"popup"` timestamp is present, consuming it upon use.

- **[The rules for choosing a navigable](https://html.spec.whatwg.org/multipage/document-sequences.html#rules-for-choosing-a-navigable)** (invoked by [the window open steps](https://html.spec.whatwg.org/multipage/nav-history-apis.html#the-window-open-steps)):
  - **Popup Allowed Check**: In evaluating whether the User Agent permits creating a new auxiliary/top-level navigable, allow creation if the relevant global object has active [transient user activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation) **OR** if [`DELEGATED_CAPABILITY_TIMESTAMPS["popup"]`](https://wicg.github.io/capability-delegation/spec.html#tracking-delegation) is present and not [expired](https://html.spec.whatwg.org/multipage/interaction.html#activation-expiry).
  - **Unified Consumption**: Upon creating the window, [consume user activation](https://html.spec.whatwg.org/multipage/interaction.html#consume-user-activation) (if present) and remove the `"popup"` entry from `DELEGATED_CAPABILITY_TIMESTAMPS`.

## Security & Privacy Considerations

### Security & Abuse Mitigations

1. **Same-Origin Invariant**:
   - Service Workers can only manage and communicate with same-origin clients. Delegation cannot cross origin boundaries.
2. **Strict 1:1 Gesture Invariant**:
   - One user notification click → exactly one transient capability granted → exactly one `postMessage` delegation allowed → exactly one `window.open()` permitted.
   - There is zero opportunity for popup amplification or looping.
3. **Double-Sided Lifespan Bounds**:
   - **Sender**: SW transient capability expires in ≤ 5 seconds within the `NotificationEvent`.
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

### 1. General User Activation on `ServiceWorkerGlobalScope` & Implicit Transfer via `postMessage`

* **Alternative Concept**: Formally extend the [HTML User Activation Data Model](https://html.spec.whatwg.org/multipage/interaction.html#user-activation-data-model) to `ServiceWorkerGlobalScope`. When a trusted user event (such as `notificationclick`) is dispatched, the Service Worker would be granted standard transient user activation. The Service Worker could then consume this activation or implicitly propagate it to a client window upon sending a `client.postMessage()`.
* **Context on Spec Gaps**:
  * There is a recognized historical gap in the web platform specifications regarding transient user activation in Service Workers. APIs like `WindowClient.focus()` and `Clients.openWindow()` normatively require transient user activation (or user interaction) to execute in standard browsing contexts.
  * In the Push/Notifications and Service Worker specifications, this was historically addressed via informal or stop-gap prose (e.g., instructing implementers to allow `focus()` or `openWindow()` during notification events without defining a formal user activation data model for workers). While transient user activation *should* arguably be formalized in the specifications for those worker APIs, relying on ambient transient user activation for popup delegation introduces serious practical issues.
* **Why Rejected / Why Explicit Capability Delegation is Preferable**:
  * **Inability to Specify Which `postMessage` Receives Activation**: In an ambient transient activation model, there is no explicit way to declare *which* `postMessage()` call receives or consumes the transient user activation. If the first message implicitly transfers or consumes activation, developer control is lost.
  * **Fragility & Developer Footguns (Intervening Calls & Libraries)**: If an application, framework, or third-party library (e.g., for analytics, telemetry, state synchronization, or push logging) calls `postMessage()` inside the `notificationclick` handler before the intended client message, that unrelated call would inadvertently consume or transfer the activation token. The subsequent message intended to trigger `window.open()` on the client tab would then arrive without activation and fail unexpectedly.
  * **Potential Breakage for Existing Sites**: Many existing web applications already dispatch multiple `postMessage()` calls during notification click processing. Implicitly attaching activation to `postMessage()` or consuming worker activation on the first message risks altering runtime behavior, introducing subtle race conditions, or breaking existing applications that do not expect activation transfer semantics on routine messages.
  * **The Advantage of Capability Delegation**: Explicit capability delegation (`client.postMessage(msg, {delegate: 'popup'})`) avoids all of these issues. It requires explicit developer intent, pinpoints the exact message and recipient tab, does not alter routine `postMessage` semantics, scopes the capability strictly to the `NotificationEvent`, and leaves other activation-gated APIs untouched.

### 2. Service Worker Auxiliary Window API / `Clients.openWindow()` with Opener Option

* **Alternative Concept**: Extend `Clients.openWindow()` (or introduce an API such as `Clients.openAuxiliaryWindow()`) allowing the Service Worker to open a new auxiliary window and explicitly assign an existing `WindowClient` as its `window.opener`:
  ```javascript
  // Hypothetical Service Worker API:
  const popupClient = await self.clients.openWindow('/preview?id=42', {
    opener: targetClient,
    features: 'width=600,height=500'
  });
  ```
* **Why this is a Compelling Consideration**:
  * Real-world web applications (such as Gmail) often do not open `about:blank` popups and synchronously populate them. Instead, they open a specific URL for auxiliary views (e.g., an email compose or preview window) that loads its own scripts and expects `window.opener` to be connected to the main webmail tab so the popup can communicate and access state.
  * Opening the auxiliary window directly from the Service Worker with the opener pre-wired could theoretically eliminate the need to delegate capability to the client tab first.
* **Why Rejected / Why Capability Delegation is Preferable**:
  * **Absence of Synchronous `WindowProxy` on the Parent Tab**:
    * When a popup is opened from the client tab via `window.open()`, the parent tab receives a synchronous DOM `WindowProxy` reference (`const popup = window.open(...)`). This allows the parent tab to immediately attach event listeners, inspect `popup.closed`, manage multiple popup instances, and pass initial state directly.
    * A Service Worker cannot hold or return a DOM `WindowProxy` (workers run on background threads without DOM access). `Clients.openWindow()` returns a `Promise<WindowClient>`. The parent `WindowClient` would not receive any synchronous reference to the newly created child window, breaking common window-management patterns unless an elaborate asynchronous messaging handshake is built between the two windows.
  * **Coordination & Readiness Race Conditions**:
    * The Service Worker has no direct insight into the internal lifecycle state of the parent `WindowClient` (e.g., whether the parent tab is currently navigating, busy, or ready to handle requests from the new child window via `window.opener`). When the client tab opens the popup itself, it has complete synchronous control over readiness and coordination.
  * **Duplication of Window Feature Parsing in Workers**:
    * Auxiliary popups require custom dimensions, positioning, and window feature configurations (`width`, `height`, `left`, `top`, `popup=true`, minimal browser chrome). Service Workers have no access to DOM, display geometry, or screen metrics (`window.screen`, DPI, multi-monitor offsets). Implementing popup feature parsing and screen placement in Service Worker APIs duplicates complex windowing logic that naturally belongs in DOM `Window` contexts.
  * **Multi-Process Architecture & Security Complexity**:
    * Establishing an opener relationship between two windows from a third, background worker thread introduces intricate process-allocation and routing edge cases in multi-process browser architectures with Site Isolation (especially if the parent tab is backgrounded, frozen, or undergoing lifecycle transitions).
  * **Developer Simplicity & Existing Code Reuse**:
    * Web applications already have mature frontend code for opening and managing popups via `window.open()`. Capability delegation allows web applications to reuse their existing window management infrastructure with minimal changes, rather than having to split window lifecycle management between the Service Worker and the client window.

### 3. Unmodified `Clients.openWindow()`

* *Why Insufficient*: Standard `Clients.openWindow()` creates an isolated top-level browser tab/window from the background worker. It cannot establish an opener relationship with an existing client tab. Without `window.opener` or a synchronous `WindowProxy`, web apps cannot share in-memory JavaScript state, active caches, or state trees, forcing an expensive cold-start app bootstrap. Furthermore, `Clients.openWindow()` cannot configure popup positioning or popup window features (`width`, `height`, minimal chrome).

### 4. Implicit Activation Propagation on `client.focus()`

* *Why Rejected*: Automatically granting full user activation whenever `client.focus()` is called is overly broad. It creates security risks by exposing unrestricted user activation to the client page, unlocking APIs unrelated to the notification click. Capability delegation requires explicit intent and is tightly restricted to the `"popup"` capability.

## Future Considerations: Frame-to-Frame Popup Delegation

In earlier design discussions, delegating the `"popup"` capability from a top-level `Window` to a cross-origin `iframe` (e.g., allowing a top-level merchant page to delegate popup opening to an embedded payment/auth provider) was considered.

While theoretically aligned with the generic Capability Delegation framework, frame-to-frame popup delegation introduces additional security and policy considerations around cross-origin iframe popup blockers, clickjacking defenses, and cross-site user activation transfer. To keep this proposal de-risked and focused on the immediate developer need in Service Workers, frame-to-frame popup delegation is deferred as a potential future extension.

## References & Prior Discussion

- **Chromium Issue**: [crbug.com/542314185](https://crbug.com/542314185)
- **Public Issue**: [capability-delegation Issue 42](https://github.com/WICG/capability-delegation/issues/42)
- **WICG Capability Delegation Specification**: https://wicg.github.io/capability-delegation/spec.html
- **W3C Service Workers Specification**: https://w3c.github.io/ServiceWorker/
- **WHATWG Notifications API Standard**: https://notifications.spec.whatwg.org/
- **WHATWG HTML Window Open Steps**: https://html.spec.whatwg.org/multipage/window-object.html#dom-open
