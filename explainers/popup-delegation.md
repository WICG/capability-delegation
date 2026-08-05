# Explainer: Capability Delegation for Pop-ups

## Authors
- dmurph@chromium.org

## Introduction
Many web APIs are gated by transient user activation to prevent abuse (such as unexpected popups or unauthorized actions). However, this creates challenges when the user interaction occurs in one context, but the action must be performed in another related context.

The **Capability Delegation API** allows a context containing user activation to delegate that capability to another context. This proposal introduces the `"popup"` capability (gating `window.open()`) to this framework and extends the delegation mechanism to support **Service Workers**.

This enables two key use cases:
1.  **Service-Worker-to-Window**: Allowing a Service Worker handling a notification click (possessing transient activation) to delegate it to a same-origin client window, so the client can open a pop-up (e.g., an email preview or chat window) that shares state with the main application.
2.  **Window-to-Iframe**: Allowing a top-level window to delegate the popup capability to a trusted cross-origin iframe (e.g., a payment or authentication provider) so the iframe can complete its flow via a secure popup.

## Goals
- Allow both Service Workers and Windows to delegate the `"popup"` capability (transient user activation for `window.open`) to target contexts.
- Enable high-performance, state-sharing popup window flows for web applications using Service Worker notifications.
- Enable secure popup generation for third-party embeds (like payment or authentication providers) hosted in iframes.
- Ensure the delegation is tightly scoped and secure against abuse (e.g., preventing clickjacking or spam popups).

## Non-Goals
- Allowing Service Workers to directly open pop-ups with shared state.
- General propagation of user activation signals across contexts without explicit delegation.
- Allowing Service Workers to delegate capabilities to cross-origin clients (SW delegation is strictly same-origin).

## User Scenarios

### Gmail State-Sharing Email Preview Window
1.  A user receives a desktop notification for a new email.
2.  The user clicks the notification.
3.  The Service Worker handles the `notificationclick` event, acquiring transient user activation.
4.  Instead of opening a new window from scratch (which requires a slow full app bootstrap), the Service Worker wants to use an existing open Gmail tab to open an email preview pop-up.
5.  The Service Worker calls `client.postMessage(message, {delegate: 'popup'})` to the existing tab.
6.  The Gmail tab receives the message, and its event handler calls `window.open()`.
7.  The pop-up opens successfully, sharing code and state with the main tab.

### Scenario 2: Third-Party Payment/Auth in IFrame (Window-to-IFrame) (Optional)
*Note: This scenario is a logical extension of the capability delegation framework and was part of the original design/explainer goals. However, there have not been strong, active requests for window-to-iframe popup delegation recently, as most providers currently use workarounds (like rendering the click target inside the iframe). We include it here for completeness and to align the API design, but it could be considered an optional extension.*

1.  A user is on an e-commerce site (`shop.example`) and clicks "Pay with WebPay".
2.  The payment button interaction grants transient user activation to the top-level window.
3.  The payment processing is hosted in a cross-origin iframe (`pay.example`) embedded in the page.
4.  To complete the payment, `pay.example` needs to open a secure authentication popup window.
5.  `shop.example` delegates the popup capability to the iframe: `payment_iframe.postMessage({action: 'start_pay'}, 'https://pay.example', {delegate: 'popup'})`.
6.  The iframe receives the message, and its event handler calls `window.open()`.
7.  The popup opens successfully, and the delegation token is consumed.

---

## Proposed API

We propose moving the `delegate` option from `WindowPostMessageOptions` to the base `PostMessageOptions` dictionary, making it available to `ServiceWorkerClient.postMessage()`. We also introduce `"popup"` as a new valid capability for delegation.

### Web IDL Changes

```webidl
dictionary PostMessageOptions : StructuredSerializeOptions {
    boolean includeUserActivation = false;
    // Change: Expose 'delegate' to all postMessage options (including Service Worker Client)
    DOMString? delegate;
};

dictionary WindowPostMessageOptions : PostMessageOptions {
    USVString targetOrigin = "/";
    // Change: Move `DOMString? delegate;` to above.
};
```

### Behavior on Unsupported Channels

Because `PostMessageOptions` is shared, moving `delegate` to it syntactically exposes the option to non-delegation-capable interfaces:
1.  `MessagePort.postMessage()`
2.  `BroadcastChannel` (does not support `PostMessageOptions` or delegation)
3.  `ServiceWorker.postMessage()` (sending a message to a Service Worker)
4.  `Client.postMessage()` when the target client is a worker or shared worker (i.e., `client.type !== 'window'`)

To prevent developer confusion and avoid silent failures, the specification proposes to **throw a `NotSupportedError` DOMException** if the `delegate` option is provided on these unsupported channels.

*Implementation Note:* Reusing the dictionary is an implementation convenience in Blink, but the spec definition should ensure that only `Window` and `ServiceWorkerClient` (specifically window client) targets can receive delegated capabilities.

### Developer Usage Example

#### Service Worker Code (`sw.js`):
```javascript
self.addEventListener('notificationclick', (event) => {
  event.waitUntil(async function() {
    // Find a same-origin client window
    const clients = await self.clients.matchAll({type: 'window'});
    const targetClient = clients.find(c => c.visibilityState === 'visible') || clients[0];
    
    if (targetClient) {
      // Focus the client
      await targetClient.focus();
      
      // Delegate the popup capability to the client
      // The SW has transient activation because of the notification click
      targetClient.postMessage({action: 'open_tearoff'}, {delegate: 'popup'});
    }
  }());
});
```

#### Client Window Code (`main.js`):
```javascript
navigator.serviceWorker.addEventListener('message', (event) => {
  if (event.data.action === 'open_tearoff') {
    // This call is allowed because of the delegated 'popup' capability.
    // It must be called synchronously in the message event handler.
    const popup = window.open('/tearoff', 'tearoff_view', 'width=600,height=400');
    if (!popup) {
      console.error('Popup was blocked despite delegation.');
    }
  }
});
```

#### Window-to-IFrame (shop.example to pay.example)

##### Top-level Page Code (`shop.js`):
```javascript
// User interaction grants activation to top-level window
payButton.addEventListener('click', () => {
  // Delegate the popup capability to the trusted payment iframe
  paymentIframe.contentWindow.postMessage(
    {action: 'initiate_payment'}, 
    'https://pay.example', 
    {delegate: 'popup'}
  );
});
```

##### IFrame Code (`payment_iframe.js`):
```javascript
window.addEventListener('message', (event) => {
  if (event.origin !== 'https://shop.example') return;
  
  if (event.data.action === 'initiate_payment') {
    // This call is allowed because of the delegated 'popup' capability.
    const authPopup = window.open(
      'https://auth.pay.example/login', 
      'auth_popup', 
      'width=400,height=500'
    );
    if (!authPopup) {
      console.error('Payment authentication popup was blocked.');
    }
  }
});
```

---

## Key Scenarios Walkthrough

### 1. Successful Delegation
- User clicks notification -> SW gets transient user activation.
- SW calls `postMessage` with `{delegate: 'popup'}` before the activation expires.
- SW transient activation is consumed.
- Client window receives message with `kPopup` capability.
- Client window gets a transient activation token for popups.
- Client window calls `window.open()` synchronously in event handler -> Pop-up opens, token is consumed.

### 2. Timeout (Abuse Mitigation)
- User clicks notification -> SW gets transient user activation.
- SW waits (e.g., doing heavy work) until the activation expires.
- SW activation has expired.
- SW call to `postMessage` with `delegate` throws `NotAllowedError` DOMException.

### 3. Double-Use Prevention (Single-Use Token)
- SW successfully delegates to Client.
- Client receives message, gets token.
- Client tries to call `window.open()` twice:
  - First call succeeds and consumes the token.
  - Second call is blocked by the standard popup blocker.

## Specification Integration

To integrate this with the HTML, Service Workers, and Capability Delegation specifications, the following changes are proposed:

### 1. User Activation Data Model for Workers / Service Workers
The [HTML User Activation Data Model](https://html.spec.whatwg.org/multipage/interaction.html#user-activation-data-model) currently tracks transient and sticky activation (`has_transient_activation` and `has_been_activated`) strictly on `Window` objects (associated with a `Navigable`).

To support capability delegation from Service Workers:
- **Extend Activation Tracking to `ServiceWorkerGlobalScope`**: The spec must formalize that certain trusted events (specifically `notificationclick` in the Notifications API) grant **transient user activation** to the `ServiceWorkerGlobalScope`.
- **Event-Scoped Lifecycle & Immediate Disposal**: Transient activation on a Service Worker is strictly tied to the active `notificationclick` event dispatch and its `ExtendableEvent.waitUntil()` execution. The Service Worker must either forward the capability during the event or drop it:
  - If the SW delegates via `Client.postMessage(..., {delegate: 'popup'})`, the activation is **consumed immediately**.
  - If the event handler completes, the `waitUntil` promise resolves, or the Service Worker terminates without delegating, the transient activation is **immediately discarded** and does not persist across subsequent events or tasks.
  - User Agents may impose a safety timeout ceiling (e.g., matching the standard notification interaction timeout) to prevent unfulfilled promises in `waitUntil` from hanging open indefinitely.
- **Directionality (SW-to-Window Only)**: Delegation is strictly unidirectional from the Service Worker to a Window Client. Initiating delegation from a DOM Window to a Service Worker via `ServiceWorker.postMessage()` is disallowed and throws a `NotSupportedError` DOMException, preventing pages from "banking" activation in the worker.

### 2. Sender Side (`Client.postMessage`)
When `Client.postMessage(message, options)` is called:
- If `options.delegate` is set to `"popup"`:
  - If the target client's type is not `"window"` (i.e., it is a worker or shared worker), the User Agent must throw a `NotSupportedError` DOMException.
  - The User Agent must verify that the sender's `ServiceWorkerGlobalScope` has active [transient user activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation).
  - If active, the User Agent **consumes** the Service Worker's transient user activation and attaches the delegated capability `"popup"` to the message container.
  - If not active, the User Agent rejects the call by throwing a `NotAllowedError` DOMException, matching `DOMWindow` behavior (as specified in [Section 3.1 of the Capability Delegation Spec](https://wicg.github.io/capability-delegation/spec.html#monkey-patch-to-html-initiating-delegation)).

### 3. Receiver Side (Client Window)
The Service Workers specification defines that `Client.postMessage` dispatches a `message` event on the target client's `ServiceWorkerContainer` (`navigator.serviceWorker`).
- When the User Agent queues the task to fire the `message` event:
  - If the message container has the delegated capability `"popup"` attached:
    - The User Agent sets `DELEGATED_CAPABILITY_TIMESTAMPS["popup"]` on the target `Window` to the [current high resolution time](https://w3c.github.io/hr-time/#dfn-current-high-resolution-time).
    - This activates the delegated capability token on the target window for a short duration (`kActivationLifespan`, e.g. 1 second).

### 4. Monkey-Patch to `window.open()` (Popup Capability Definition)
In the HTML specification's window open steps / popup blocker check:
- When `Window.open()` is called:
  1. Check if the relevant global object has [transient activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation), OR if `DELEGATED_CAPABILITY_TIMESTAMPS["popup"]` is present and not [expired](https://html.spec.whatwg.org/multipage/interaction.html#activation-expiry).
  2. If neither condition is met, the popup is blocked by the User Agent's popup blocker policy.
  3. If permitted via delegated capability (and lacking direct transient activation), the User Agent **clears** `DELEGATED_CAPABILITY_TIMESTAMPS["popup"]` immediately, consuming the single-use token.

---

## Alternatives Considered

### 1. `Clients.openWindow()`
*Why it is insufficient:* 
- **Performance / State Sharing**: `Clients.openWindow()` creates a completely new, isolated top-level browsing context. For large web applications (like Gmail), this forces a full cold application bootstrap from scratch, leading to noticeable latency compared to opening a tearoff window that shares in-memory state and script context with an existing tab.
- **Window Positioning and Features**: `Clients.openWindow()` opens a new standard browser tab/window without allowing the developer to customize popup features (e.g. `width`, `height`, popup window placement, or minimal window chrome).
- **Navigation Capture / TWA Differences**: In Progressive Web Apps (PWAs) and Trusted Web Activities (TWAs), `openWindow()` can trigger navigation capturing that either replaces the current window or opens in a separate browser tab rather than an auxiliary popup view.

### 2. Implicit Activation Propagation to `showNotification()` Caller
*Alternative:* Propagating user activation specifically to the browsing context that originally called `registration.showNotification()`, mirroring the behavior of the legacy `new Notification()` API.

*Why it is insufficient:*
- **Multi-Client Scope & Disconnected Lifetimes**: A Service Worker registration manages all clients within its scope and persists across the lifetime of multiple browsing contexts. A notification displayed by Tab A might be clicked hours later when Tab A has navigated or closed, but Tab B is open. Binding activation implicitly to an original caller window fails when clients change.
- **Push Notifications (No Originating Window)**: In many modern applications (email, messaging), notifications are triggered from background `push` events when *no* client window is currently open or initiated the notification.
- **Security & Cross-Window Bypass Risks**: Implicitly routing activation between windows via the registration allows malicious scripts to use notifications as a mechanism to store and bounce user activation across unrelated browsing contexts to bypass popup blockers. Explicit Capability Delegation avoids this by requiring the active SW event handler to choose a specific, currently visible client, consuming the SW activation in the process.

### 3. Implicit Activation Propagation on Focus
We considered automatically propagating user activation to a client window when the Service Worker calls `client.focus()`. However, this is too broad. Exposing full user activation implicitly creates security risks (e.g., allowing the page to access other restricted APIs like Clipboard or Midi without explicit user intent for that action). Capability delegation is explicit and tightly scoped to a specific capability (popups).

### 4. Restricting Popup Delegation strictly to Service Workers
We considered only allowing the `"popup"` capability to be delegated from Service Workers, and disallowing it for window-to-window (frame) postMessage. However, this would block legitimate web platform use cases that Capability Delegation was originally envisioned for, such as third-party payment or authentication iframes opening popups. Restricting it does not significantly improve security: if a top-level page is malicious, it can already open popups directly using its own user activation; delegating that right to a child iframe does not grant the malicious page any new capabilities. Therefore, the restriction would remove valid use cases without providing meaningful security benefits.

---

## Security & Privacy Considerations

This feature grants a bypass to the popup blocker, which is a high-security-risk area. We mitigate abuse through the following design constraints:

1.  **Sender Activation Required**: The sender (Service Worker or Window) must possess active transient user activation (e.g., from a `notificationclick` event or direct user interaction) to initiate the delegation.
2.  **Activation Consumption on Sender**: Initiating a delegation consumes the transient activation on the sender context immediately. A Service Worker cannot broadcast or loop `postMessage` with delegation to multiple client windows from a single click event.
3.  **Single-Use Token**: The delegated capability token on the receiver side is consumed immediately upon the first call to `window.open()`. It cannot be used to spawn multiple pop-ups.
4.  **Lifespan Constraints**:
    *   **Sender Activation**: 
        *   **Frame-to-Frame**: Uses standard transient user activation, which expires after a short UA-defined duration (5 seconds in Chrome).
        *   **Service Worker**: Scoped strictly to the execution of the `notificationclick` event. The SW must either forward the capability during the event or drop it. If the SW finishes event processing or terminates without delegating, the activation is immediately discarded. A maximum timeout ceiling prevents hanging promises from holding activation open indefinitely.
    *   **Delegated Token**: Once the delegation message is received by the target window, it has a short user-agent defined lifespan to consume the delegated capability (e.g., calling `window.open()`). In Chrome, this delegated capability lifespan is 1 second (`kActivationLifespan`) for both frame-to-frame and Service Worker-to-client delegations to prevent delayed, unexpected pop-ups.
5.  **Scope, Reachability & Multi-Client Isolation**:
    *   **Service Workers (1-to-1 Same-Origin)**: SW delegation is strictly same-origin. Because a Service Worker registration can control multiple browsing contexts simultaneously, delegation is strictly 1-to-1: only the specific `WindowClient` explicitly addressed via `targetClient.postMessage(..., {delegate: 'popup'})` receives the capability token.
    *   **Window-to-Window & Auxiliary Frames (Handle-Restricted)**:
        *   A `Window` can only delegate to browsing contexts where it holds a direct `WindowProxy` reference (e.g., a child `iframe`, a parent/opener window, or an auxiliary top-level window opened via `window.open()`).
        *   A window **cannot** delegate to arbitrary unlinked same-origin documents or tabs across the browser session, because multi-tab broadcast mechanisms (`BroadcastChannel`, `MessagePort`, `SharedWorker`, `localStorage`) do not support capability delegation.
        *   **Delegating to Auxiliary Top-Level Windows**: We explicitly acknowledge that a window taking a user click can forward the `"popup"` capability to an opened auxiliary top-level window (or child frame), optionally focus that target window, and have that window open a popup. This does not create an abuse vector because:
            1. The sender window already possessed transient user activation and could have opened a popup or focused the window directly.
            2. Initiating delegation immediately consumes the sender's transient activation, maintaining a strict 1:1 invariant between user gestures and popups (zero popup amplification).
            3. The target window only has a 1-second window to consume the single-use token upon receiving the message.
    *   **Cross-Origin Windows**: While Window-to-Iframe delegation can cross origin boundaries (essential for payment/auth use cases), developers are strongly encouraged to specify an explicit target origin in `postMessage()` to prevent accidental delegation to untrusted frames.
6.  **Focus & Presentation Flow**: 
    *   Capability delegation transfers the capability token without altering browser focus state.
    *   In the Service Worker notification flow, developers use the existing `await client.focus()` API to bring the desired client window into the foreground upon handling the notification click before delegating the capability.
    *   When the client window consumes the token via `window.open()`, the resulting popup window is created and presented following standard browser window manager rules.

## References & Prior Discussion

-   **Chromium Bug**: [crbug.com/542314185](https://crbug.com/542314185)
-   **HTML User Activation Data Model**: [HTML Spec - Tracking User Activation](https://html.spec.whatwg.org/multipage/interaction.html#user-activation-data-model)
-   **WICG Capability Delegation**: https://github.com/WICG/capability-delegation
-   **Capability Delegation Specification**: https://wicg.github.io/capability-delegation/spec.html
