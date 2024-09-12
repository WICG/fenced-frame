# Integration with the web platform
This document goes into the various ways in which fenced frame interacts with the platform. The web platform is massive and so we expect this document to be a running document with ongoing additions.

## Source url
The initial source url for the fenced frame is subjected to various restrictons or no restrictions depending on the use case for the fenced frame. Refer to the use cases document [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/use_cases.md) that discusses the source url for each of those.

## Origin
The origin of the fenced frame will be the regular origin that it got navigated to. For opaque src, the origin will be that of the url was mapped to by the browser. Any origin-keyed storage and communication channels with other same-origin frames outside the fenced frame tree will be disallowed by using a partitioning key with a nonce. The storage key will use the same nonce for the nested iframes and the root fenced frame, so that same-origin channels can still work within the fenced frame tree. Essentially, along with the storage partitioning effort, the storage and broadcast channel access will be keyed on <fenced-frame-root-site, fenced-frame-root-origin, nonce> for the root frame and <fenced-frame-root-site, nested-iframe-origin, nonce> for a nested iframe. More details related to this can be found [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/storage_cookies_network_state.md).

## Size
The API that generates a fenced frame config can pick the initial size that the fenced frame document sees, subject to whatever restrictions it deems necessary for its privacy model. If the initial size is fixed, then any changes the embedder makes to the width and height attributes of the <fencedframe> will not be reflected inside the fenced frame.

## Viewability
Viewability events using APIs like [Intersection Observer](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) can be a communication channel from the embedding context to the fenced frame. However, to be used as a covert channel, it does require movement of the frame containing the ad on the screen and therefore is hostile to user experience. 
Viewability events are important information in the ads workflow for conversion reports and billing; therefore, Intersection Observer will be fully supported in fenced frames and it will be phased out only if/when an alternative mechanism is launched. In the meantime, we plan to add metrics to understand honest use and detect abuse, to help guide the need/development of these eventual mechanisms or mitigations.

## Visibility
APIs like [visibilityState](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilityState) and the corresponding [visibilityChange event](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event) determine the visibility of the tab based on user actions like backgrounding etc. A fenced frame and the main page could join server-side on the basis of when they lose and gain visibility. This kind of joining is also possible with other joining ids like the fenced frame creation time. The network side channel and future mitigations are described more [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/network_side_channel.md).

## Accessibility
For accessibility, fenced frames will behave like iframes, they will be part of the accessibility tree and a11y users can navigate through its contents with a screen reader.
“title” is an attribute read by screen readers for an iframe and that will continue to be used for fenced frames. A script inside a fenced frame cannot access the title attribute via the same-origin channel window.frameElement.title since frameElement is null for fenced frames.

## Interactivity
Fenced frames would have normal user-interactivity as an iframe would.

## Focus
Since fenced frames require interactivity, they would need to get system focus. It could be a privacy concern in the following ways:
* A fenced frame and the main page could join server-side on the basis of when one lost focus and another gained focus. The network side channel and future mitigations are described more [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/network_side_channel.md) 
* For programmatic focus, calling element.focus() might invoke blur() on an element that is in the embedding page thus making a channel from the fenced frame to the embedding page. So the mitigations there would be:
  * Only allow programmatic focus()/blur() to be successful if the fenced frame already has system focus.
  * Only allow system focus to move to a FF on a user gesture including hitting the tab key and not because of calling focus().

## PostMessage
A fenced frame does not allow communication via PostMessage between the embedding and embedded contexts. A fenced frame will be in a separate browsing context group then its embedding page. 

## Session History
Fenced frames can navigate but their history is not part of the browser back/forward list as that could be a communication channel from the fenced frame to the embedding page. Additionally, fenced frames will always have a replacement-only navigation (back/forward list of length 1) which is a simpler model since it doesn't imply that there's a hidden list of back/forward entries for the nested page, only accessible via history APIs and not via the back/forward buttons, etc. This is also consistent with the iframes new opt-in mode for disjoint session history as discussed in https://github.com/whatwg/html/issues/6501.

## Content Security Policy
Fenced frames ineractions with CSP are detailed [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/interaction_with_content_security_policy.md).

## Permissions and policies
This is discussed in more detail [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/permission_document_policies.md).

## COOP and COEP
Although COOP is only defined for top-level documents, it has impacts on popups opened by subframes in the page. Because COOP is crucial to the web exposed mitigation against Spectre, Fenced Frames must support COOP in one way or another. Because COOP contains a reporting endpoint, we cannot actually pass the COOP value to the Fenced Frame. So to support COOP, we have two options:
* Forbid Fenced Frames from opening popups entirely.
* Mandate that popups opened from Fenced Frames always have rel no-opener and place them in another BCG (ie the behavior of the strictest form of COOP). We propose fenced frames to take this approach as opening popups is an important functionality that needs to be supported e.g. when a user clicks on an ad in a fenced frame.

For COEP, If the fenced frame’s embedding page enables COEP then the fenced frame document should allow itself to be embedded as described [here](https://docs.google.com/document/d/1zDlfvfTJ_9e8Jdc8ehuV4zMEu9ySMCiTGMS9y0GU92k/edit#bookmark=id.kaco6v4zwnx2). COEP provides two bits of information to a fenced frame: whether the embedder has COEP enabled, and whether the fenced frame is same-origin with its embedder (through the additional CORP check). We could remove the second one by having COEP always be checked regardless of whether the document in the fenced frame is same origin with its embedder or not. In the initial origin trial, fenced frames behavior will match that of iframes and eventually we will make fenced frames always behave as if it was cross-origin to the embedder.

## Opt-in header
Since fenced frames allow a document to have many constraints in place, an opt-in mechanism is a good way for the document to accept those restrictions. The opt-in will make use of the Supports-Loading-Mode proposed [here](https://github.com/WICG/nav-speculation/blob/main/opt-in.md).

It is also important for sites to opt-in due to security reasons, e.g. csp:frame-ancestors behavior. Frame ancestors checks will stop at the Fenced Frame root for Protected Audience fenced frames. All other fenced frames (e.g., for selectURL or created via FencedFrameConfig) will check all the way up to the primary top-level frame. Protected Audience is different because it is only allowed to have k-anonymous information flow into the fenced frame, and the primary top-level frame’s origin may not be k-anonymous.

## Fetch metadata integration
To let a server know that a document is being requested for rendering in a fenced frame, a new Sec-Fetch-Dest HTTP Request Header value of `fencedframe` will be sent in the request.

## Unload and beforeunload handlers
Fenced frames will not be supporting unload or before unload handlers. This is because of the following reasons:
* Both of these have the existing issue of unreliability, so hopefully not supporting them should not lead to breaking critical workflows that depend on them.
* There have been issues with both the handlers because running code when the user is trying to navigate away is not very respectful of the user.
* It's a communication channel (the page deletion timestamp). It's not a major one since it's similar to the creation timestamp which is already present but disabling them will eliminate one communication channel between the embedding page and the fenced frame.

## Top-level navigation
Some modes of fenced frames allow navigating the top-level frame.  The approach adds a new target name called `_unfencedTop` (in the same category as `_self`, `_parent`, `_top`) that works with existing HTML elements/JS APIs (`<a>`, `<area>`, `<form>`, `window.open`). Example usage:

```
// Navigates the top-level frame to 'url'. Subject to the same restrictions as sandboxed
// iframes where top-level navigation is allowed upon user activation.
window.open('url', '_unfencedTop');

<!-- Defines an HTML hyperlink to 'url' that will open the document in the top-level frame (when clicked). -->
<a href='url' target='_unfencedTop'>Click me!</a>
```

A few more details:
* The user activation is checked only in the frame initiating the navigation and not in an ancestor outside the frame tree because user activation inside the fenced frame tree is not propagated outside the tree.
* The url is subject to the same restrictions as other content-initiated top-level navigations, e.g. data urls are not allowed.
* The opener/openee relationship would not be present for this navigation, which means the window.open example above will always return null. 
* Referer would be the frame that initiated the navigation which could be the fenced frame root frame or a nested iframe in the fenced frame tree (similar to a popup navigation).

For more details, the implementation design doc can be found [here](https://docs.google.com/document/d/1vuwG31hCwZROIR1NaYjTrthmDrlXza0mZui7zzF93TM/edit?usp=sharing).

For privacy implications of this API and others, see [the privacy considerations](https://github.com/WICG/fenced-frame/blob/master/explainer/README.md#privacy-considerations) section.

## Screen interface
The [Screen interface](https://drafts.csswg.org/cssom-view/#the-screen-interface) provides information about the screen of the browser's output device, and as a result that information is visible to both fenced frames and the embedder. Currently there are no plans to fence the Screen interface, meaning that its attributes will retain the same values in both the fenced frames and their embedders. Justification for this decision can be found [here](https://docs.google.com/document/d/1sZOgnAUsIzNHOs_VVWF92er1jXmNCcv6k3vvOvixeFc/edit?usp=sharing). 

## MediaDevices interface
The [MediaDevices interface](https://w3c.github.io/mediacapture-main/#mediadevices) is able to list connected media devices via the [`enumerateDevices()`](https://w3c.github.io/mediacapture-main/#dom-mediadevices-enumeratedevices) method. The `deviceID` field in this method's output can potentially create consistent identifiers between same-origin frames embedded in different first-party sites. The [specification for `deviceID`](https://w3c.github.io/mediacapture-main/#dom-mediadeviceinfo-deviceid) states:

"To ensure stored identifiers are recognized, the identifier MUST be the same in Documents of the same origin in top-level traversables. In child navigables, the decision of whether or not the identifier is the same across documents, MUST follow the User Agent's partitioning rules for storage (such as localStorage), if any, to not interfere with mitigations for cross-site correlation."

Fenced frames partition storage using a unique nonce, so that no other frame can access the same partitioned storage as a given fenced frame. As a result, deviceID values will always be different within two fenced frames and similarly the value in a fenced frame will always differ with that in other iframes/top-level frames, even if their origin is the same.

## WebRTC
[WebRTC](https://webrtc.org/) ([spec](https://www.w3.org/TR/webrtc/#intro)) is an open standard and set of APIs with two primary purposes:

1. Enable media capture of cameras, microphones, and displays.  
1. Facilitate peer-to-peer communication between clients for the purpose of sharing captured media or other arbitrary data. 

WebRTC enables critical use cases of the web today, like audio and video chat, and we need to ensure that fenced frames account for its usage. And in a way, they already do. Capture of cameras, microphones, and displays are gated behind permissions policies, and in fenced frames, **those policy-gated features are [unconditionally disallowed](https://github.com/WICG/fenced-frame/blob/master/explainer/permissions_policy_for_API_backed_fenced_frames.md#introduction).** This means that many primary use cases enabled by WebRTC are not supported, and there are no plans to support them unless necessary use cases are identified. 

However, WebRTC peer connections via  `RTCPeerConnection` and `RTCDataChannel` are still enabled. This means that for fenced frames in the [local unpartitioned data access mode](https://github.com/WICG/fenced-frame/blob/46c479e01d741dfab8429bc18e5f3193e4fd6db0/explainer/fenced_frames_with_local_unpartitioned_data_access.md#revoking-network-access), we would have to spec and build behavior to disable WebRTC peer connections after calling `window.fence.disableUntrustedNetwork()`. We’ve decided against doing so, and will instead **disable `RTCPeerConnection` construction in all fenced frames, regardless of whether network access has been voluntarily disabled or not**. We have a few reasons for making this choice:

1. Utility: We have not yet identified any use cases for fenced frames that would require WebRTC peer connections.  
   1. A significant amount of WebRTC utility is already hampered by the disabled media capture permissions.  
1. Privacy: Network communications of any kind present a [privacy side-channel](https://github.com/WICG/fenced-frame/blob/46c479e01d741dfab8429bc18e5f3193e4fd6db0/explainer/network_side_channel.md#network-side-channel), and we should take the opportunity to close these channels where we can.  
1. Complexity: Network revocation for `RTCPeerConnection` would likely be more involved than other types of requests, for potentially little benefit given 1 and 2.

Navigation, subresource fetches, and Websockets will still be enabled in fenced frames by default, but disabled after a call to `window.fence.disableUntrustedNetwork()`.  

## Chromium implementation: Top-level browsing context using MPArch
Chromium is implementing [Multiple Page Architecture](https://docs.google.com/document/d/1NginQ8k0w3znuwTiJ5qjYmBKgZDekvEPC22q0I4swxQ/edit?usp=sharing) for various use-cases including [back/forward-cache](https://web.dev/bfcache/), [portals](https://wicg.github.io/portals/), prerendering etc. This architecture aligns with fenced frames requirement to be a top-level browsing context as MPArch enables one WebContents to host multiple pages. Additionally, those pages could be nested, as is the requirement for fenced frames. 

A page in MPArch behaves as the top-level frame for their frame tree and as a top-level browsing context from a spec perspective. Any calls to window.parent, window.top or window.frames[x] work for the inner and outer pages independently i.e. a frame in the inner page cannot access the outer page using window.parent/top etc. A fenced frame does not allow script access to the embedding context and thus aligns well with being implemented using MPArch. The implementation design is detailed here: 
[fenced frames implementation design](https://docs.google.com/document/d/1HAU9IiHZU4KBPC_rEk3BQYrWTK50PdNGg8CcHhVLXig/edit?usp=sharing)

### How new features should integrate with fenced frames

When implementing a new feature on the web platform, its integration with many other parts of the web platform must be considered, e.g., [inactive Documents in the BFCache](https://w3ctag.github.io/design-principles/#support-non-fully-active), and [prerendering](https://wicg.github.io/nav-speculation/prerendering.html#delay-async-apis).  Fenced frames adds another dimension within which new features must consider their behavior. We've written some preliminary guidelines on how to think about the behavior of new features that operate within a fenced frame [**here**](https://chromium.googlesource.com/chromium/src/+/master/content/browser/fenced_frame/README.md), which we will consider upstreaming to a more broad venue, perhaps like the [W3C TAG Design Principles](https://w3ctag.github.io/design-principles/) document.

### Obsolete: Initial version: using shadowDOM based iframes
While MPArch is being developed in parallel with fenced frames, the initial implementation that will be available for origin trial will be based on the shadowDOM architecture. The implementation design is detailed here:
[Fenced Frames Origin Trial Design: ShadowDOM + IDL](https://docs.google.com/document/d/1ijTZJT3DHQ1ljp4QQe4E4XCCRaYAxmInNzN1SzeJM8s/edit?usp=sharing)

## Browser features
There are many chrome browser features e.g. autofill, translate etc., that will need to be considered on a case-by-case basis as to how they would interact with a fenced frame with possible approaches being:
* Treat the feature as if it would behave in a cross-origin iframe.
* Treat the feature as if fenced frame was a separate top-level page.
* Do not support the feature at all.

### Extensions
It is important for fenced frames to be visible to extensions especially content blockers since many of them block advertisements. 
In terms of non-content blocker extensions, they should also be able to interact with fenced frames as they would with another page. Note that malicious extensions might be able to override the privacy protections here by creating cross-site identifiers but that threat is larger than just fenced frames and would need to be handled as a separate effort.

### Developer tools
Fenced frames should have developer tools access as an iframe would.





