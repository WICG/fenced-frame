# Network side channel

The privacy challenge is that the  documents inside and outside the fenced frame can collude with a server to join an identifier across the fence boundary, using information like request timing and IP addresses. 

**Prerequisites of this attack**

This side channel relies on the following factors being true:
* There are colluding servers that are joining logs to be able to create cross-site user profiles. 
* There is common information available in the fenced frame context and the outer context. This is trivially available information like:
  * creation time of the fenced frame/ network request time,
  * IP address of the client.
* With protected audience API (and other opaque URL fenced frames), there is also the privacy leak that a given URL was selected. Even with k-anonymity, each render url leaks a few bits and multiple auctions on the page can then be correlated to the identity of the user.

**Example**

As an example of Protected Audience API, the interest group of the user can be joined with the user’s identity on the publisher site. Once this attack is executed, a publisher can know that a given user on their site has been interested in "running shoes" and has been browsing sites related to "running shoes".
Similarly, the advertiser/ad-tech can know the site their ad was shown on along with the user-id of the user on those sites.

Consider the following example information flow:

*   Embedding site A sends a message to a tracking site, say tracker.example saying it is about to create a fenced frame and that the user id on A is 123.
*   The fenced frame is created and has access to user specific information X (e.g. user’s interest group for Protected Audience via the render URL). When the fenced frame document’s resources are requested from site B, X is also sent along. B can also let tracker.example know.
*   The tracking site tracker.example can then correlate using the time/IP address bits of both requests.
*   A can then know X via tracker.com

The above is an example of a scenario where user id on A and user’s information on B can be joined without the user even interacting with the fenced frame.

## Mitigations

These issues are not unique to fenced frames and also exist in cross-site navigations today so they could either depend on future solutions to these for cross-site navigations e.g. [Gnatcatcher](https://github.com/bslassey/ip-blindness), or could have additional specific mitigations for fenced frames such as ad rendering in which all network-loaded resources come from a trusted CDN that does not keep logs of the resources it serves. The privacy model and browser trust mechanism for such a CDN would require further work.

