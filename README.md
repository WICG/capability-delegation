# Capability Delegation
Transferring the ability to use restricted APIs to another `window` in the frame
tree.

See the spec proposal
[here](https://wicg.github.io/capability-delegation/spec.html).

## Introduction

### What is capability delegation?

Many capabilities in the Web are usable from JS in restricted manners.  For
example:
- Most browsers allow popups (through `window.open()`) only if the user has
  either interacted with the page recently or allowed the browser to open popups
  from the page's origin.
- A sandboxed `iframe` cannot make itself full-screen (though
  `element.requestFullscreen()`) without a specific sandbox attribute or a user
  interaction within the frame.

Capability delegation means allowing a frame to relinquish its ability to call a
restricted API and transfer the ability to another (sub)frame it can trust.  We
are particularly interested in a dynamic delegation mechanism which (unlike
`<iframe allow=...>` attribute) does not expose the capability to the frame in
a time-unconstrained manner.


### Motivation

Here are some practical scenarios that would utilize a capability delegation
mechanism.

- Many merchant websites host their online store on their own domain but
  outsource the payment collection and processing infrastructure to a Payment
  Service Provider (PSP) to comply with security and regulatory complexities
  around card payments.  This workflow is implemented as a "pay" button
  inside the top (merchant) frame where it can blend better with the rest of the
  merchant’s website, and payment request code inside a cross-origin `iframe`
  from the PSP.  The [Payment Request
  API](https://w3c.github.io/payment-request) used by the PSP code is gated by
  transient user activation (to prevent malicious attempts like unattended or
  repeated payment requests).  Because the top (merchant) frame’s user
  interaction is not visible to the `iframe`, the PSP code needs some kind of a
  delegation in response to a click in the top frame to be able to initiate a
  payment processing.

- A web service that does not care about user location except for a "branch
  locator" functionality provided by a third-party map-provider app can delegate
  its own location access capability to the map `iframe` in a temporary manner
  right after the "branch locator" button is clicked.

- In Chrome we received
  [this](https://bugs.chromium.org/p/chromium/issues/detail?id=931966#c5)
  feature request from a developer where a presentation/slide website has a
  "control panel" to selectively make other spawned windows fullscreen.  With
  Capability Delegation API, a click on the control panel can delegate
  fullscreen capability to the selected window and bring that window to
  fullscreen without needing any more clicks.

- An authentication provider may wish to show a popup to complete the
  authentication flow before returning a token to the host site.

- A website may want a third-party chat app in an `iframe` to be able to vibrate
  the phone on message receipt, even when the user is not active in the
  `iframe`.


## Proposal: Transient Capability Delegation

### Model of delegation

Our proposed model focuses on delegation of a specific capability only (instead
of delegating user activation) in a time-constrained manner.  We will call it
Transient Capability Delegation (TCD).

When a sender `Window` delegates a capability `X` to a receiver `Window`, the
sender’s user activation would be
[consumed](https://html.spec.whatwg.org/multipage/interaction.html#consume-user-activation)
to create a time-limited token `T_X` on the receiving end.  In more details:

- The sender’s ability to use TCD would be gated by transient user activation.
  More precisely, a TCD request will consume the user activation in the sender’s
  `Window` to prevent repeated requests (making TCD a transient activation
  consuming API) but the receiving `Window` won’t get any user activation at all.

- A successful delegation would create a time-constrained token `T_X` in the
  recipient `Window`.  The lifespan and behavior of `T_X` would be
  capability-specific, defined by the spec owners of capability X.  Token `T_X`
  won’t be exposed to JS.

- On the receiving end, `T_X` would be “tied” to the recipient `Window` object
  so it would be non-transferrable by design.


### Tentative API

We are proposing a new option to `Window.postMessage()` that facilitates TCD
through the existing messaging mechanism:

```javascript
targetWindow.postMessage('a_message', {createToken: X});
```

For the Payment Request API, we are proposing the token specifier
`"paymentrequest"`, so the call above would look like:

```javascript
targetWindow.postMessage('a_message', {createToken: "paymentrequest"});
```

### Demo

To see how this API works with Payment Request, run Chrome 90.0.4414.0 or newer
with the command-line flag
`--enable-blink-features=CapabilityDelegationPaymentRequest`, then open [this
demo](https://wicg.github.io/capability-delegation/example/payment-request/).


## Related links

- [Design discussion](https://docs.google.com/document/d/1IYN0mVy7yi4Afnm2Y0uda0JH8L2KwLgaBqsMVLMYXtk).
- [Chromium bug](https://crbug.com/1130558).
