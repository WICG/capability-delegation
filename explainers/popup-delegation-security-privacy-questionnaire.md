# Self-Review Questionnaire: Security and Privacy (Service Worker Popup Capability Delegation)

This document answers the questions listed in the
[W3C Security and Privacy Self-Review Questionnaire](https://www.w3.org/TR/security-privacy-questionnaire/)
for [Service Worker Capability Delegation for Pop-ups](popup-delegation.md)
(`Client.postMessage(message, {delegate: 'popup'})`).

For the baseline Security and Privacy Questionnaire covering the overall
Capability Delegation framework, see
[../security_and_privacy_questionnaire.md](../security_and_privacy_questionnaire.md).

## Questions and Answers

### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

None. This feature does not expose any new information to websites or third
parties. It only allows a Service Worker handling a trusted user interaction
event (`notificationclick`) to delegate its transient popup-opening capability
(`{delegate: 'popup'}`) via `Client.postMessage()` to an existing same-origin
`WindowClient`.

### 2.2. Do the features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes. No new data or state is exposed. The receiving `WindowClient` is not given
an explicit JavaScript getter to inspect the delegated token; the capability is
tracked internally by the User Agent and consumed when `window.open()` is
invoked.

### 2.3. How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

Not applicable. The feature does not access, process, store, or transmit
personal information or PII.

### 2.4. How do the features in your specification deal with sensitive information?

Not applicable. No sensitive information is handled or exposed.

### 2.5. Does data exposed by your specification carry related but distinct information that may not be obvious to users?

No.

### 2.6. Do the features in your specification introduce state that persists across browsing sessions?

No. Delegated capability tokens and timestamps
(`DELEGATED_CAPABILITY_TIMESTAMPS["popup"]`) are strictly ephemeral, held only
in memory for at most a few seconds (≤ 5 seconds on the `NotificationEvent`
context in the Service Worker and ≤ 1 second on the receiving `WindowClient`),
and immediately cleared upon consumption or expiry.

### 2.7. Do the features in your specification expose information about the underlying platform to origins?

No.

### 2.8. Does this specification allow an origin to send data to the underlying platform?

No. It extends `Client.postMessage()` with a `ClientPostMessageOptions`
dictionary (`{delegate: 'popup'}`) between a Service Worker and a same-origin
`WindowClient` within the browser.

### 2.9. Do features in this specification enable access to device sensors?

No.

### 2.10. Do features in this specification enable new script execution/loading mechanisms?

No.

### 2.11. Do features in this specification allow an origin to access other devices?

No.

### 2.12. Do features in this specification allow an origin some measure of control over a user agent's native UI?

Yes, to the same extent as standard `window.open()`. It permits an existing,
visible same-origin `WindowClient` to open one auxiliary popup window
(`window.open()`) in response to a user clicking a Service Worker notification,
achieving parity with the legacy document-bound `new Notification()` API. To
prevent abuse or disorienting UI:

1. **Strict 1:1 Gesture Invariant**: One notification click grants at most one
   delegated token and permits at most one popup window. Calling `window.open()`
   consumes both transient user activation and the delegated `"popup"`
   capability token.
2. **No Silent Popunders**: The target `WindowClient` must be visible/focused
   (via `await client.focus()`) for `window.open()` to succeed.
3. **No Cross-Origin or Multi-Hop Forwarding**: Delegation is restricted to
   same-origin `WindowClient` instances and cannot be re-delegated.

### 2.13. What temporary identifiers do the features in your specification create or expose to the web?

None. The capability timestamp is internal to the User Agent and is never
exposed as an identifier to script.

### 2.14. How does this specification distinguish between behavior in first-party and third-party contexts?

Service Workers can only match and post messages to same-origin clients
(`self.clients.matchAll()`). Because both the Service Worker and the target
`WindowClient` must share the same origin and storage partition, capability
delegation cannot cross origin or third-party storage partition boundaries.

### 2.15. How do the features in this specification work in the context of a browser's Private Browsing or Incognito mode?

Behavior in Private Browsing / Incognito mode is identical to normal browsing
mode within that profile's isolated storage partition. No state persists across
sessions or leaks between normal and incognito profiles.

### 2.16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

Yes. Both sections are detailed in the
[Explainer's Security & Privacy Considerations section](popup-delegation.md#security--privacy-considerations).

### 2.17. Do features in your specification enable origins to downgrade default security protections?

No. It selectively satisfies the user-activation check of the popup blocker on a
target same-origin `WindowClient` strictly when backed by a real user
notification click in the Service Worker. Sandboxed documents without
`allow-popups` remain blocked from opening popups even if they receive a
delegated capability.

### 2.18. What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

A document in BFCache is not an active/visible client eligible for focus and
cannot process `window.open()` while frozen. Furthermore, because the delegated
capability token expires after 1 second, any token delivered prior to a
navigation would expire before or upon BFCache restoration.

### 2.19. What happens when a document that uses your feature gets disconnected?

If the target `WindowClient` is unloaded or disconnected, any pending message or
transient capability timestamp on that `Window` is destroyed with the document.
