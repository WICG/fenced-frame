# Fenced frames: permissions and document policies

## Introduction

This document goes into how fenced frames interact with various ways of policy delegation available on the web platform, namely, [permission policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Feature_Policy) and [document policy](https://wicg.github.io/document-policy/ ). 


## Permission Policy

As mentioned in the list [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Feature-Policy#directives), some of the powerful features that permission policy (earlier known as feature policy) supports are autoplay, geolocation, camera etc. The HTTP header provides a mechanism to allow and deny the use of browser features in its own frame, and in content within any &lt;iframe> elements in the document. The way a top-level page can currently deny/allow these features in an iframe does not work with fenced frames for the following reasons. 



*   We cannot allow permissions to be delegated to the fenced frame since a fenced frame can then invoke a number of these APIs and based on whether each of them is allowed or not, it can be used to communicate a bitmap from the embedding page to the fenced frame.
*   It then prompts the question as to whether it is possible for the fenced frame to behave like a top-level browsing context and do its own header exchange to determine what needs to be allowed/denied in the fenced frame tree. This also does not work because:
    1. if the actual top-level page had denied a feature for an origin e.g. Feature-Policy: geolocation 'none' and a fenced frame is allowed to use geolocation via its own header exchange, then it acts as a workaround for the restrictions placed on embedded frames and leads to an escalation of privilege attack.
    2. If we only allow a feature via FF headers if it was also allowed by the top-level page, then it acts as a bitmap channel as mentioned in the above point.


### Summary

Permission policy can neither be delegated to a fenced frame nor can a fenced frame enable permissions based on its header exchange like a top-level page. A fenced frame is therefore restricted to use cases that do not depend on these powerful features. Since permission policy is used for powerful features, restricting these should still leave use cases where fenced frames can be utilized e.g. ads. Having said that, there may be API-specific caveats as discussed in the open questions below.


### Client hints: open question



*   Client hints is one of the features that rely on [using permissions policy delegation](https://github.com/WICG/ua-client-hints#for-example) by the top-level page for allowing certain client hints to be used by an embedding frame. Since delegation won’t be allowed, should the fenced frames be allowed to behave like top-level contexts to figure out which client hints should be sent, via the Accept-CH headers? That could lead to an escalation of privilege but may be for utility this will need to be supported? Client hints might contain high-entropy and sensitive data but might be needed for normal functioning of a site. Additionally, fenced frames like iframes, will not read/write from the origin’s client hints opt-in cache. 


## Document Policy

Document policy allows the top-level page to configure features on a frame via the ‘policy’ attribute. The embedded frame’s document request will then go on the network with a  `Sec-Required-Document-Policy` header and the frame will only load if the response comes back with the same or stricter document policy. Since this depends on delegation, it has the same issues as the permissions policy or CSPEE with respect to being used as a communication channel. Since this is applied per-document (vs per origin in permissions policy), it is closer to CSPEE.

Similar to [CSPEE handling in fenced frames](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/interaction_with_content_security_policy.md), the embedding site should treat fenced frames differently and assume that the presence of a fenced frame on the site implies no control on the policies of that frame via the policy attribute.  

The escalation of privileges attack mentioned above for permissions policy does not apply to document policy since this is delegated per-embedded frame and not per-origin and the embedding site can clearly distinguish between which frames can and cannot be controlled via delegation. 

Additionally, the fenced frame can still have document policy headers for itself and its embedded frames independent of the embedding tree.


### Summary

Document policy cannot be delegated to a fenced frame, but it can be set as a top-level frame. 


### Open Questions



*   One of the newer features to be added to the document policy is getViewportMedia as is being discussed in this [issue](https://github.com/w3c/mediacapture-screen-share/issues/155). This feature will only be able to work for a tab if all embedded frames opt-in. What should be the fenced frames behavior in such cases? 
    *   **Proposed:** A fenced frame is fine to be part of a screen capture if the frame is opted in and the user agrees to the permission as well. This is acceptable from a threat model perspective because screen capture of another tab is also possible today (gated on permission and user acceptance).
    *   **Open question:** The above proposal implies that there will need to be a list of features that the fenced frame should know to opt-in and that is is hard to maintain as the list grows.
