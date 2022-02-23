# Network side channel attack

Consider the following network side channel attack:

*   Embedding site A sends a message to a tracking site, say tracker.example saying it is about to create a fenced frame and that the user id on A is 123.
*   The fenced frame is created and has access to user specific information X (e.g. user’s interest group for TURTLEDOVE). When the fenced frame document’s resources are requested from site B, X is also sent along. B can also let tracker.example know. The tracking site tracker.example can then correlate using the time/IP address bits of both requests.
*   A can then know X via tracker.com

The above is an example of a scenario where user id on A and user’s information on B can be joined without the user interacting with the fenced frame e.g. in the "opaque-ads" mode.
The common bits of information for such an attack are:
*   Network timing
*   IP address correlation between frames

## Mitigations

These issues are not unique to fenced frames and also exist in cross-site navigations today so they could either depend on future solutions to these for cross-site navigations e.g. [Gnatcatcher](https://github.com/bslassey/ip-blindness), or could have additional specific mitigations for fenced frames. These are currently being brainstormed.
Both the opaque-ads consumers, FLEDGE and SharedStorage will guarantee k-anonymity of the URL used to create the fenced frame. This, in conjunction with other solutions like Gnatcatcher, will mitigate the cross-site data joining attack to a large extent.
