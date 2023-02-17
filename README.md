# Capability Delegation

Transferring the ability to use restricted APIs to another `window`.

([Draft specification](https://wicg.github.io/capability-delegation/spec.html))


## Author

Mustaq Ahmed (mustaq@chromium.org,
[github.com/mustaqahmed](https://github.com/mustaqahmed))


## Participate

* Github repository:
  [WICG/capability-delegation](https://github.com/WICG/capability-delegation)
* Issue tracker:
  [WICG/capability-delegation/issues](https://github.com/WICG/capability-delegation/issues/)


## Introduction

"Capability delegation" means allowing a frame to relinquish its ability to call
a restricted API and transfer the ability to another (sub)frame it trusts. The
focus here is a dynamic delegation mechanism which exposes the capability to the
target frame in a time-constrained manner (unlike `<iframe allow=...>` attribute
which is not time-constrained).

The API proposed here is based on `postMessage()`, where the sender frame uses a
new
[PostMessageOptions](https://html.spec.whatwg.org/multipage/window-object.html#windowpostmessageoptions)
member to specify the capability it wants to delegate.


## Motivating use-cases

Here are some practical scenarios that are enabled by the Capability Delegation
API.


### Secure PaymentRequest processing in a subframe

Many merchant websites perform payment processing through a Payment Service
Provider (PSP) site (e.g. [Stripe](https://stripe.com)) to comply with security
and regulatory complexities around card payments.  When the end-user clicks on
the "Pay" button on the merchant website, the merchant website sends a message
to a cross-origin `iframe` from the PSP website to initiate payment processing,
and then the `iframe` uses the [Payment Request
API](https://w3c.github.io/payment-request) to complete the task.

But sites are only allowed to call the [Payment Request
API](https://w3c.github.io/payment-request) after [transient user
activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation)
(a recent click or other interaction) to prevent malicious attempts like
unattended or repeated payment requests.  Since the user probably clicked on the
main site, and not the PSP `iframe`, this would prevent the PSP from using the
Payment Request API at all.  Browsers today support such payment processing by
ignoring the user activation requirement altogether (see
[crbug.com/1114218](https://crbug.com/1114218))!

Capability Delegation API provides a way to support this use-case while letting
the browser enforce the user activation requirement, as follows:

```javascript
// Top-frame (merchant website) code
checkout_button.onclick = () => {
    targetWindow.postMessage("process_payment", {targetOrigin: "https://example.com",
                                                 delegate: "payment"
                                                });
};

// Sub-frame (PSP website) code
window.onmessage = () => {
    const payment_request = new PaymentRequest(...);
    const payment_response = await payment_request.show();
    ...
}
```


### Allowing fullscreen from opener Window click

This is a
[work-in-progress](https://groups.google.com/a/chromium.org/g/blink-dev/c/7YkubntWi3Y/m/gwK7fMiEAwAJ)
in Chrome.

Consider a presentation/slide website where the main "control panel" window has
spawned a few presentation windows, and the user wants to selectively make one
presentation window fullscreen by clicking on the appropriate button on the main
window (a [feature request
](https://bugs.chromium.org/p/chromium/issues/detail?id=931966#c5)from a
developer).  Clicking on the "control panel" button does not make the user
activation available to the presentation window, so this does not work today.

The Web does not support this use-case today but Capability Delegation API
provides a solution:

```javascript
// Main window ("control panel") code
let win1 = open("presentation1.html");
let win2 = open("presentation2.html");

button1.onclick = () => win1.postMessage("msg", {targetOrigin: "https://example.com",
                                                 delegate: "fullscreen"});
button2.onclick = () => win2.postMessage("msg", {targetOrigin: "https://example.com",
                                                 delegate: "fullscreen"});

// Sub-frame ("presentation window") code
window.onmessage = () => document.body.requestFullscreen();
```

### Allowing display capture from cross-origin iframe click

Consider a web app in which you want to add video-conferencing capabilities.
You turn to a third party solution that can be embedded in a cross-origin
`iframe`. There's a lot of logic behind the scenes, but UX-wise, maybe you
work out a scheme where it's mostly the video which is user-facing in the
video-conferencing `iframe`, and the user-facing controls - mute, leave,
share-screen - are all part of the web app, and receive its speficifc UX
styling. When those buttons are pressed, some messages are exchanged between
your the web app and the embedded video-conferencing solution.

The web does not support this use-case today but Capability Delegation API
provides a solution:

```js
// In the cross-origin video-conferencing iframe
button.onclick = () =>
  window.parent.postMessage("msg", { delegate: "display-capture" });
```

```js
// In the top frame.
window.onmessage = () => navigator.mediaDevices.getDisplayMedia();
```

### Other similar scenarios

* A web service that does not care about user location except for a "branch
  locator" functionality provided by a third-party map-provider app can delegate
  its own location access capability to the map `iframe` in a temporary manner
  right after the "branch locator" button is clicked.

* An authentication provider may wish to show a popup to complete the
  authentication flow before returning a token to the host site.

* A website may want a third-party chat app in an `iframe` to be able to vibrate
  the phone on message receipt, even when the user is not active in the
  `iframe`.


## Non-goals

* This explainer is not about delegation of [user
  activation](https://html.spec.whatwg.org/multipage/interaction.html#tracking-user-activation)
  (i.e., allowing the `iframe` to choose from all of the things the top frame
  could do after a user click or other interaction).  See [Considered
  Alternatives](#considered-alternatives) below for more details.

* This explainer does not determine which APIs could possibly support capability
  delegation.  If any API needs the support, the designers of the API would
  decide details of delegated behavior.  The PaymentRequest API case presented
  here (in collaboration with the owners of that API) serves as a guide for
  similar changes in other API specifications.


## Using capability delegation

Developers would use Capability Delegation by just initiating the delegation
appropriately, as shown in the example code snippets above.  In short, when a
[browsing
context](https://html.spec.whatwg.org/multipage/browsers.html#browsing-context)
wants to delegate a capability to another browsing context, it sends a
`postMessage()` to the second browsing context with an extra
[`WindowPostMessageOptions`](https://html.spec.whatwg.org/multipage/window-object.html#windowpostmessageoptions)
member called `delegate` specifying the capability.

After a successful delegation, the "user API" (the restricted API being
delegated) just works when called at the right moment.  The general idea is
calling the restricted API in a `MessageEvent` handler or soon afterwards.  In
the examples above, the restricted APIs are `payment_request.show()`,
`element.requestFullscreen()`, and `mediaDevices.getDisplayMedia()` respectively.


### Demo

- Payment Request API: To see how this API works with Payment Request, run
Chrome with the command-line flag: `--enable-blink-features=PaymentRequestRequiresUserActivation`, then open
[this
demo](https://wicg.github.io/capability-delegation/example/payment-request/).

- Fullscreen API: Work in progress.

- Screen Capture API: Work in progress.

## Related links

* [Design
  discussion](https://docs.google.com/document/d/1IYN0mVy7yi4Afnm2Y0uda0JH8L2KwLgaBqsMVLMYXtk).


## Considered alternatives

### Delegating user activation instead of a specific capability

It may appear that we can delegate user activation to solve the same use-cases
and thus avoid specifying a feature in the `postMessage()` call.  We attempted
this direction in the past from a few different perspectives, and decided not to
pursue this.  In particular, user activation controls many Web APIs, so
delegating user activation for any of the mentioned use-cases is impossible
without causing problems with unrelated APIs.  See the [TAG
discussion](https://github.com/w3ctag/design-reviews/issues/347) with one past
attempt.


### Using a delegation-specific method instead of postMessage()

Instead of piggy-backing the delegation request as a `PostMessageOptions` entry,
we considered adding a new delegation-specific interface on the `Window` object.
While the latter may look cleaner from a developer’s perspective, to support
cross-origin communication this solution would require adding the new method on
the
[`WindowProxy`](https://developer.mozilla.org/en-US/docs/Glossary/WindowProxy)
wrapper, which HTML's editor [strongly
disliked](https://github.com/whatwg/html/pull/4369#issuecomment-470580082).


## Stakeholder feedback/opposition

We will track the overall status through this [Chrome Status
entry](https://www.chromestatus.com/feature/5708770829139968).


## Acknowledgements

Many thanks for valuable feedback and advice from:

* Anne van Kesteren ([github.com/annevk](https://github.com/annevk))
* Jeffrey Yasskin ([github.com/jyasskin](https://github.com/jyasskin))
* Robert Flack ([github.com/flackr](https://github.com/flackr))
