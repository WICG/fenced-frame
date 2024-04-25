# Fenced frames: permissions and document policies

## Introduction

This document goes into how fenced frames interact with various ways of policy delegation available on the web platform, namely, [permission policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Feature_Policy) and [document policy](https://wicg.github.io/document-policy/ ). 


## Permission Policy

As mentioned in the list [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Feature-Policy#directives), some of the powerful features that permission policy (earlier known as feature policy) supports are autoplay, geolocation, camera etc. The HTTP header provides a mechanism to allow and deny the use of browser features in its own frame, and in content within any &lt;iframe> elements in the document. The way a top-level page can currently deny/allow these features in an iframe does not work with fenced frames for the following reasons. 

*   Some permissions-backed features (such as fullscreen and picture-in-picture) have actions that are observable from an embedder, which would create a data exfiltration channel if enabled in a fenced frame.
*   If permissions are delegated to a fenced frame, the fenced frame can then invoke a number of these APIs and based on whether each of them are allowed or not, it can be used to communicate a bitmap from the embedding page to the fenced frame.
*   It then prompts the question as to whether it is possible for the fenced frame to behave like a top-level browsing context and do its own header exchange to determine what needs to be allowed/denied in the fenced frame tree. If the actual top-level page had denied a feature for an origin e.g. Feature-Policy: geolocation 'none' and a fenced frame is allowed to use geolocation via its own header exchange, then it acts as a workaround for the restrictions placed on embedded frames and leads to an escalation of privilege attack.

Different fenced frame configurations still need permissions-backed features to function properly. To handle that without compromising privacy, different behaviors are specified based on the existing privacy guarantees of the fenced frame:

*   Fenced frames that guarantee k-anonymity (i.e. fenced frames loaded with opaque URNs created through an API like Protected Audience) have a fixed list of permissions that must be delegated to it by the embedder in order for it to load. This forces the embedding context to have the same permissions bitmap as any other embedding context the fenced frame would be loaded in, removing that fingerprinting channel while also preventing privilege escalation. There is [a proposal to add permissions more flexibly rather than relying on a fixed list](https://github.com/WICG/fenced-frame/blob/master/explainer/permissions_policy_for_API_backed_fenced_frames.md), but this change is not currently planned on being implemented.
*   Fenced frames that do not guarantee k-anonymity (i.e. developer-created fenced frames loaded with a transparent URL) can inherit permissions from its embedder, since the fingerprinting vector is not a concern. Because the data exfiltration channel is still a concern, we only allow select permissions-backed features that do not have data exfiltration risks.

Permissions-backed features themselves can still introduce data leaks when enabled. [This document](https://chromium-review.googlesource.com/c/chromium/src/+/5462443) outlines that in greater depth, and includes an audit of which features are safe to enable for what kinds of fenced frames.

### Summary

Given the above challenges, permissions delegation will either need to be disabled or requires a separate verification to ensure that it is not used to communicate user identifying information. Completely disabling doesn't work for our main use cases e.g. FLEDGE ads require usage of APIs like attribution reporting to be able to do reporting correctly. To see how those are planned to be supported, please read https://github.com/WICG/fenced-frame/blob/master/explainer/permissions_policy_for_API_backed_fenced_frames.md 

Relatedly, [Permissions API](https://developer.mozilla.org/en-US/docs/Web/API/Permissions_API) should return the correct result inside a fenced frame tree.


### UA Client hints: open question



*   UA Client hints is one of the features that rely on [using permissions policy delegation](https://github.com/WICG/ua-client-hints#for-example) by the top-level page for allowing certain client hints to be used by an embedding frame. Since delegation won’t be allowed, should the fenced frames be allowed to behave like top-level contexts to figure out which client hints should be sent, via the Accept-CH headers but like iframes, not read/write from the origin’s client hints opt-in cache? That could lead to an escalation of privilege but may be better for utility. Client hints contain high-entropy and sensitive data but might be needed for normal functioning of a site.
   * Initially user agent client hints will be disallowed similar to all other permission policy based features.  


## Document Policy

Document policy allows the top-level page to configure features on a frame via the ‘policy’ attribute. The embedded frame’s document request will then go on the network with a  `Sec-Required-Document-Policy` header and the frame will only load if the response comes back with the same or stricter document policy. Since this depends on delegation, it has the same issues as the permissions policy or CSPEE with respect to being used as a communication channel. Since this is applied per-document (vs per origin in permissions policy), it is closer to CSPEE.

Similar to [CSPEE handling in fenced frames](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/interaction_with_content_security_policy.md), the embedding site should treat fenced frames differently and assume that the presence of a fenced frame on the site implies no control on the policies of that frame via the policy attribute.  

The escalation of privileges attack mentioned above for permissions policy does not apply to document policy since this is delegated per-embedded frame and not per-origin and the embedding site can clearly distinguish between which frames can and cannot be controlled via delegation. 

Additionally, the fenced frame can still have document policy headers for itself and its embedded frames independent of the embedding tree.


### Summary

Document policy cannot be delegated to a fenced frame, but it can be set as a top-level frame. 


### Open Questions



*   One of the newer features to be added to the document policy is getViewportMedia as is being discussed in this [issue](https://github.com/w3c/mediacapture-screen-share/issues/155). This feature will only be able to work for a tab if all embedded frames opt-in via document policy. What should be the fenced frames behavior in such cases? 
    *   **Proposed:** A fenced frame is fine to be part of a screen capture if the frame is opted in and the user agrees to the permission as well. This is acceptable from a threat model perspective because screen capture of another tab is also possible today (gated on permission and user acceptance).
    *   **Open question:** The above proposal implies that there will need to be a list of features that the fenced frame should know to opt-in and that is is hard to maintain as the list grows. This question is not yet resolved for the upcoming initial trial of fenced frames but since required document policy isn't yet launched, the lack of a solution is not currently breaking a workflow.
