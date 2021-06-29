# Self-Review Questionnaire: Security and Privacy

This document answers the questions listed in [W3C Security and Privacy
Self-Review Questionnaire](https://w3ctag.github.io/security-questionnaire/).

## Questions and answers

https://github.com/WICG/capability-delegation/blob/main/security_and_privacy_questionnaire.md
### 01. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

This feature does not expose any information.  It only exposes one new state to
the target of a `postMessage()` call: the availability of the delegated
capability.  The use-cases we want to support requires the availability of
certain capabilities in target sites, and this feature achieves that goal by
exposing a minimal availability state.


### 02.  Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes, see the answer to Question 01.


### 03.  How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

No.


### 04.  How do the features in your specification deal with sensitive information?

No.


### 05.  Do the features in your specification introduce new state for an origin that persists across browsing sessions?

No.

### 06.  Do the features in your specification expose information about the underlying platform to origins?

No.


### 07.  Does this specification allow an origin to send data to the underlying platform?

No.


### 08.  Do features in this specification enable access to device sensors?

No.


### 09.  What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

This feature does not expose any new data (other than what can be passed using
`postMessage()` calls already).  It only exposes a new state, see the answer to
Question 01.


### 10.  Do features in this specification enable new script execution/loading mechanisms?

No.


### 11.  Do features in this specification allow an origin to access other devices?

No.


### 12.  Do features in this specification allow an origin some measure of control over a user agent's native UI?

No.


### 13.  What temporary identifiers do the features in this specification create or expose to the web?

None.


### 14.  How does this specification distinguish between behavior in first-party and third-party contexts?

This does not distinguish between first-party vs third-party behavior.  The
recipient of the delegation gets the transient ability to call the delegated
capability regardless of its origin.


### 15.  How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

This would work in the "incognito" mode in the same way as in the "regular" mode.


### 16.  Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

There is no privacy concern here because user data is not involved in any way.

We considered possible security concerns in the [Privacy and security
considerations](https://docs.google.com/document/d/1IYN0mVy7yi4Afnm2Y0uda0JH8L2KwLgaBqsMVLMYXtk/edit#bookmark=id.tmi3ocafmi73)
section in the design doc.  One important point is that the effect of delegation
of each individual capability needs to be defined in a case-by-case basis in
future, and each corresponding specification change is likely to require a
separate security review from the perspective of that specific capability.


### 17.  Do features in your specification enable origins to downgrade default security protections?

No.


### 18.  What should this questionnaire have asked?

One relevant security sensitive question is: does delegating capability A affect
capability B in any way?  The answer is "no" here because every capability is
treated individually, and the design prevents both repeated delegation and
chaining.  See the [Privacy and security
considerations](https://docs.google.com/document/d/1IYN0mVy7yi4Afnm2Y0uda0JH8L2KwLgaBqsMVLMYXtk/edit#bookmark=id.tmi3ocafmi73)
section in the design doc.
