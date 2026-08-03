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
2.  `ServiceWorker.postMessage()` (sending a message to a Service Worker)
3.  `Client.postMessage()` when the target client is a worker or shared worker (i.e., `client.type !== 'window'`)

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
- User clicks notification -> SW gets 1-second transient activation.
- SW calls `postMessage` with `{delegate: 'popup'}` within 1 second.
- SW transient activation is consumed.
- Client window receives message with `kPopup` capability.
- Client window gets 1-second transient activation token for popups.
- Client window calls `window.open()` synchronously in event handler -> Pop-up opens, token is consumed.

### 2. Timeout (Abuse Mitigation)
- User clicks notification -> SW gets 1-second transient activation.
- SW waits 2 seconds (e.g., doing heavy work) before calling `postMessage`.
- SW activation has expired.
- SW call to `postMessage` with `delegate` throws `NotAllowedError` DOMException.

### 3. Double-Use Prevention (Single-Use Token)
- SW successfully delegates to Client.
- Client receives message, gets token.
- Client tries to call `window.open()` twice:
  - First call succeeds and consumes the token.
  - Second call is blocked by the standard popup blocker.

## Specification Integration

To integrate this with the HTML and Service Workers specifications, the following changes are proposed:

### 1. Sender Side (Service Worker)
When `Client.postMessage(message, options)` is called:
- If `options.delegate` is set to `"popup"`:
  - If the target client's type is not `"window"` (i.e., it is a worker or shared worker), the User Agent must throw a `NotSupportedError` DOMException.
  - The User Agent must verify that the Service Worker global scope has active **transient user activation** (e.g., from a recent notification click).
  - If active, the User Agent **consumes** the Service Worker's transient user activation and attaches the delegated capability `"popup"` to the message container.
  - If not active, the User Agent rejects the call by throwing a `NotAllowedError` DOMException, matching `DOMWindow` behavior (as specified in [Section 3.1 of the Capability Delegation Spec](https://wicg.github.io/capability-delegation/spec.html#monkey-patch-to-html-initiating-delegation)).

### 2. Receiver Side (Client Window)
The Service Workers specification defines that `Client.postMessage` dispatches a `message` event on the target client's `ServiceWorkerContainer` (i.e., `navigator.serviceWorker`).
- When the User Agent queues the task to fire the `message` event, or during the dispatch of this event:
  - If the message container has the delegated capability `"popup"` attached:
    - The User Agent must set the entry for `"popup"` in the target client's `Window`'s `DELEGATED_CAPABILITY_TIMESTAMPS` map to the current high-resolution time.
    - This activates the delegated capability token on the target window, allowing synchronous calls to `window.open()` within the event handler to consume it.

---

## Alternatives Considered

### 1. `Clients.openWindow()`
*Why it is insufficient:* `Clients.openWindow()` opens a new top-level browsing context. This context is completely isolated. For heavy apps like Gmail, this requires bootstrapping the entire application from scratch in the new window, which is slow. The "state-sharing popup" model relies on `window.open()` from a parent window to share JS context and state.

### 2. Implicit Activation Propagation on Focus
We considered automatically propagating user activation to a client window when the Service Worker calls `client.focus()`. However, this is too broad. Exposing full user activation implicitly creates security risks (e.g., allowing the page to access other restricted APIs like Clipboard or Midi without explicit user intent for that action). Capability delegation is explicit and tightly scoped to a specific capability (popups).

### 3. Restricting Popup Delegation strictly to Service Workers
We considered only allowing the `"popup"` capability to be delegated from Service Workers, and disallowing it for window-to-window (frame) postMessage. However, this would block legitimate web platform use cases that Capability Delegation was originally envisioned for, such as third-party payment or authentication iframes opening popups. Restricting it does not significantly improve security: if a top-level page is malicious, it can already open popups directly using its own user activation; delegating that right to a child iframe does not grant the malicious page any new capabilities. Therefore, the restriction would remove valid use cases without providing meaningful security benefits.

---

## Security & Privacy Considerations

This feature grants a bypass to the popup blocker, which is a high-security-risk area. We mitigate abuse through the following design constraints:

1.  **Sender Activation Required**: The sender (Service Worker or Window) must possess active transient user activation (e.g., from a notification click or direct click) to initiate the delegation.
2.  **Activation Consumption on Sender**: Initiating a delegation consumes the transient activation on the sender context immediately, preventing the sender from reusing the same gesture.
3.  **Single-Use Token**: The delegated capability token on the receiver side is consumed immediately upon the first call to `window.open()`. It cannot be used to spawn multiple pop-ups.
4.  **Short Lifespan (1-second limit)**: 
    *   **Service Worker Activation**: The transient user activation acquired by a Service Worker (e.g., from a notification click) is proposed to have a short **1-second** lifespan (compared to the standard 5-second lifespan for window interactions) to ensure delegation happens immediately.
    *   **Delegated Token**: The delegated `"popup"` capability token on the receiver client window also expires after **1 second** (using a 1-second expiry instead of the standard capability delegation default which often matches the 5s user activation window). This prevents "delayed" popups that could surprise the user long after they clicked the notification.
5.  **Scope Restrictions**:
    *   **Service Workers**: SW delegation is strictly same-origin (enforced by the SW scope and client matching model).
    *   **Windows**: While Window-to-Iframe delegation can cross origin boundaries (essential for payment/auth use cases), developers are strongly encouraged to specify an explicit target origin in `postMessage()` to prevent accidental delegation to untrusted frames.
6.  **Visibility Constraints on Receiver**: 
    *   To prevent background pages from abusing delegated capabilities (e.g., opening surprise pop-ups in background tabs), the User Agent should only activate the delegated capability token if the target client window is **visible** (or focused) at the time the message is received.
    *   In the Service Worker notification scenario, this aligns with the recommended flow where `await client.focus()` is called before `postMessage()`. If the client fails to become visible/focused, the delegation should not succeed.


## References & Prior Discussion

-   **Chromium Bug**: [crbug.com/542314185](https://crbug.com/542314185)
-   **WICG Capability Delegation**: https://github.com/WICG/capability-delegation
-   **Capability Delegation Specification**: https://wicg.github.io/capability-delegation/spec.html
