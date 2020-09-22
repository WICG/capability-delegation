# Capability Delegation
Transferring the ability to use restricted APIs to another `window` in the frame
tree.

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
  around card payments.  This is workflow is implemented as a "pay" button
  inside the top (merchant) frame where it can blend better with the rest of the
  merchant’s website, and payment request code inside a cross-origin `iframe`
  from the PSP.  The [Payment Request
  API](https://w3c.github.io/payment-request) used by the PSP code is gated by
  transient user activation (to prevent malicious attempts like unattended or
  repeated payment requests).  Because the top (merchant) frame’s user
  interaction is not visible to the `iframe`, the PSP code needs some kind of a
  delegation in response to a click in the top frame to be able to initiate a
  payment processing.

- A website may want a third-party chat app in an `iframe` to be able to vibrate
  the phone on message receipt, even when the user is not active in the
  `iframe`.

- A web service that does not care about user location except for a "branch
  locator" functionality provided by a third-party map-provider app can delegate
  its own location access capability to the map `iframe` in a temporary manner.

- An authentication provider may wish to show a popup to complete the
  authentication flow before returning a token to the host site.


### Challenges

TODO: Work in progress.

We need to delegate a capability in such a way that other related capabilities
are unaffected.

Static capability delegation (through `<iframe allow=...>` attribute) is not
limited by time.  The use cases we want to support requires time-constrained
dynamic delegation.


## Related links

- Design discussion: https://docs.google.com/document/d/1IYN0mVy7yi4Afnm2Y0uda0JH8L2KwLgaBqsMVLMYXtk
- Chromium bug: https://crbug.com/1130558

<!--
### Past proposals on delegation

The API presented here is based on ideas/challenges discussed in several past
attempts:
- [Gesture delegation
  explained](https://docs.google.com/document/d/1HkTSdeQKrYrEFuLGzgBXRvxclo2BzWXwuGrYsL2vD9k)
- [Delegating user activation to child
  frames](https://docs.google.com/document/d/1yZQjK7Q_BsyJ74Vj7Xpm3QzhDyDXB8kGdk3aESEYtSg)
- [Combining gesture delegation with feature
  policy](https://docs.google.com/document/d/11gqqQhHcVNhYRclVGL6h7prt_n9rjbYstvCWgZu-E7M)
- [Activation delegation through
  transfer](https://docs.google.com/document/d/1NKLJ2MBa9lA_FKRgD2ZIO7vIftOJ_YiXXMYfRMdlV-s).


### User activation
- [HTML
  specification](https://html.spec.whatwg.org/multipage/interaction.html#tracking-user-activation)
  for tracking user activation.
- [Chrome APIs gated by user
  activation](https://docs.google.com/document/d/1mcxB5J_u370juJhSsmK0XQONG2CIE3mvu827O-Knw_Y)
-->
