**Table of Contents**  

- [Explainer - Fenced Frames](#explainer---fenced-frames)
  - [Authors](#authors)
  - [Introduction](#introduction)
  - [Goals](#goals)
  - [Design](#design)
    - [Fenced frame API](#fenced-frame-api)
      - [New element type - a top-level browsing context](#new-element-type---a-top-level-browsing-context)
        - [Example usage](#example-usage)
        - [Benefits over nested browsing context](#benefits-over-nested-browsing-context)
        - [Downsides](#downsides)
      - [Alternative approach considered - a nested browsing context](#alternative-approach-considered---a-nested-browsing-context)
        - [Example usage](#example-usage-1)
        - [Benefits](#benefits)
        - [Downsides](#downsides-1)
    - [Fenced frame tree](#fenced-frame-tree)
    - [Information channel between fenced frame and other frames](#information-channel-between-fenced-frame-and-other-frames)
    - [Network access or Web bundles](#network-access-or-web-bundles)
  - [Use-cases/Key scenarios](#use-caseskey-scenarios)
    - [Cross-site portals](#cross-site-portals)
    - [Interest Group ads based on user activity (TURTLEDOVE)](#interest-group-ads-based-on-user-activity-turtledove)
      - [Design](#design-1)
    - [Conversion Lift Measurement](#conversion-lift-measurement)
      - [Design](#design-2)
    - [Unpartitioned storage access](#unpartitioned-storage-access)
  - [Security considerations](#security-considerations)
  - [Privacy considerations](#privacy-considerations)
    - [Network side channel attack](#network-side-channel-attack)
  - [Challenges](#challenges)
  - [Considered alternatives](#considered-alternatives)
      - [Using a new iframe attribute](#using-a-new-iframe-attribute)
      - [Using Feature policy/Permission policy](#using-feature-policypermission-policy)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Explainer - Fenced Frames

## Authors
*   Shivani Sharma
*   Josh Karlin

## Introduction

Third party iframes can communicate with their embedding page using mechanisms like postMessage. This communication is allowed so long as it doesn’t permit [cross-site recognition](https://w3cping.github.io/privacy-threat-model/#model-cross-site-recognition), a threat that browsers are trying to address by [partitioning storage](https://github.com/privacycg/storage-partitioning) by the top-level site. However, there are a number of recently proposed APIs (such as [Interest group based advertising](https://github.com/michaelkleber/turtledove), [Conversion Lift Measurements](https://github.com/w3c/web-advertising/blob/master/support_for_advertising_use_cases.md#conversion-lift-measurement), [Portals](https://github.com/WICG/portals), and [requestStorageAccess](https://github.com/privacycg/storage-access)) that must provide some degree of unpartitioned storage to embedded documents. Such documents should not be allowed to communicate with their embedders, else they will be able to join their cross-site user identifiers. This explainer proposes a new form of embedded document, called a fenced frame, that these new APIs can use to isolate themselves from their embedders, preventing cross-site recognition.


## Goals

The fenced frame enforces a boundary between the embedding page and the cross-site embedded document such that user data visible to the two sites is not able to be joined together. This can be helpful in preventing user tracking or other privacy threats. Some of the use cases that are discussed in this document include:



*   Interest group based advertising
*   Conversion Lift measurement studies
*   Granting unpartitioned storage access without a permission prompt ([discussion](https://github.com/privacycg/storage-access/issues/41))
*   Cross site portals 

The privacy threat addressed is:

**The ability to correlate the user’s identity/information on the embedding site with that on the embedded site.**

## Design

Fenced frames are embedded contexts that have the following characteristics to prevent embedder identifiers from being joined with identifiers from the embedded site:



*   They’re not allowed to communicate with the embedder and vice-versa, except for certain information such as limited size information, the embedder’s top-level site, and the frame’s document url.
*   They do not have storage access (e.g., cookies, localStorage, etc.) by default. 
*   They can provide access to unpartitioned data
    *   Before user activation, read-only unpartitioned storage can be obtained by calling requestStorageAccess with a new read-only parameter.
        *   But once the read-only data is provided, network access is disabled until user activation.
    *   Upon user activation, read-write storage access can be requested via requestStorageAccess and network access is allowed.
        *   Any previously returned read-only bucket is now read-write.

The idea is that the fenced frame should not have access to both of the following pieces of information:



*   User information on the embedding site
    *   Accessible via communication channels
*   User information on the fenced frame site
    *   Accessible via an API (e.g., Turtledove) or via access to unpartitioned storage  

Before user activation, unpartitioned storage (if accessible) is only available in a read-only mode. This is to make sure that any bits of information that may have been passed along to the fenced frame e.g. using frame size, is not persisted.

Once the restrictions are lifted, we expect some leakage of information to be possible via network timing attacks. The user activation helps to rate-limit that leakage to situations where the user has shown engagement, where ideally the rate will be low enough that broad user tracking via fenced frames isn’t feasible or cost effective. This can also be further mitigated by making the embedding context unaware of the user activation on the fenced frame, which should be possible for cases where the user activation is not navigating the embedding frame.

A primary use case (Turtledove, Conversion Lift Measurement) for a fenced frame is to have read-only access to some unpartitioned storage, for example, in Turtledove, it is the interest-based ad to be loaded. The URL of the ad is sufficient to give away the user information. This thus involves a new concept of an opaque URL which is opaque to any context other than the fenced frame, something like an [opaque fetch response](https://fetch.spec.whatwg.org/#concept-filtered-response-opaque), or a new on-device opaque computation result — which can be used for rendering and reporting, but cannot be inspected directly. Since that URL might be leaked by timing attacks if the fenced frame had network access, the fenced frame’s network access must be revoked, until user activation.  

The state transitions of a fenced frame are thus:



1. Start
    *   Full network access, no storage access
2. On requesting read-only storage access without user activation or access to opaque URL
    *   No network access, read-only unpartitioned storage (if requested)
3. On user activation
    *   Full network access, read/write unpartitioned storage (if requested)

Some fenced frames might start in state (2) if they need to start with access to storage and no network. Such fenced frames will require a web bundle to load.

### Fenced frame API 

The proposed fenced frame API is to have a new element type and treat it as a [top-level browsing context](https://html.spec.whatwg.org/#top-level-browsing-context). Given in this section are the details of this approach and also the alternative approach that was considered.


#### New element type - a top-level browsing context

In this approach, a fenced frame behaves as a top-level browsing context that is embedded in another page. This is aligned with the fact that a fenced frame is similar to a “tab” since it has minimal communication with the embedding context and is the root of its frame tree and all the frames within the tree can communicate normally with each other. 


##### Example usage


```
<fencedframe src="demo_fenced_frame.html"></fencedframe>
```


  

Details would need to be worked out for the following:



*   Headers to communicate with the frame site to let it know that a request is for a fenced frame environment.
*   Note that the fenced frame may be either created at the time of request or in the future using the web bundle fetched in the request. This header exchange will be done in advance of creating the fenced frame for cases where there is no network in the fenced frame since it starts with having access to unpartitioned storage.
*   How [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) and [sandbox](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/sandbox) policies might be applied to the fenced-frame element.

##### Benefits over nested browsing context



*   Simpler to spec since the fenced frame tree aligns well with a top-level browsing context.
*   Simpler to achieve the communications restrictions with the embedding context, being a top-level browsing context.


##### Downsides



*   Existing sites need to change to embed a new element. In order to not have all existing embedding sites replace iframe with a new type of element, third parties may create a fenced frame inside the iframe. This might have a performance cost though.


#### Alternative approach considered - a nested browsing context

In this alternative approach the fenced frame is a nested browsing context with the communications restrictions placed on an iframe via a [document policy](https://github.com/w3c/webappsec-feature-policy/blob/master/document-policy-explainer.md).


##### Example usage

```
<iframe src="demo_iframe_fenced.html"
policy="fenced-frame-tree;root=true"></iframe>
```


The embedding site sends the following header in the request when creating the fenced frame root frame:


```
Sec-Required-Document-Policy: fenced-frame-tree;root=true
```


The server then responds back using the following header if it complies with the restrictions (otherwise the document will fail to load with an error). 


```
Document-Policy: fenced-frame-tree;root=true
```

The fenced frame tree’s root frame is created using the root parameter’s value as true while any frames nested within it is set by the browser as having the root param set to false, unless a nested fenced frame tree is created where the root parameter is again set to true. Both true and false values of the root parameter are considered of equal strictness and thus it is possible to continue a fenced frame tree or start a new one.


##### Benefits



*   Able to work with the existing “iframe” element


##### Downsides



*   Much more complex spec and browser implementation since many iframe features will need to be special-cased for fenced frames.

There were other alternatives considered for the iframe approach like using feature policy or a new attribute, detailed later in this document.


### Fenced frame tree

A fenced frame is the root of the fenced frame tree and frames in this tree are not allowed to use communication channels to talk to frames outside the tree or vice-versa. The frames within the tree can communicate with each other. 

### Information channel between fenced frame and other frames

There are many channels between the fenced frame tree and the other frames that will need to be restricted and some of them are listed below:



*   PostMessage
*   Name and id attributes
*   Resize 
*   Positioning 
*   Access to window.parent/ window.top etc.
*   Events fired simultaneously in the embedding context and the fenced frame such as page lifecycle events
*   …

This discussion assumes that third-party cookies, like all other third party storage,  are also [disallowed](https://blog.chromium.org/2020/01/building-more-private-web-path-towards.html) or else those would be a communication channel between the fenced frame and the embedding site.

Some of the channels cannot be completely removed as they are required for the fenced frame’s creation and are discussed in the [privacy considerations](#privacy-considerations) section.


### Network access or Web bundles

Fenced frames may have network access until they obtain cross-site information. That information could come from read-only unpartitioned storage access, or it may come from the fenced frame’s URL. If it comes from the URL, then the URL must point to a [web bundle](https://web.dev/web-bundles/) downloaded earlier and the frame will not have network access. The reason that network is restricted once cross-site information is known is because of timing attacks, which are discussed below in [privacy considerations](#privacy-considerations).

## Use-cases/Key scenarios

Following are potential use cases for fenced frames. This is not an exhaustive list and we expect the use cases to grow further.


### Cross-site portals

[Portals](https://wicg.github.io/portals/) allow for rendering of, and seamless navigation to, embedded content.

If the embedded content is cross-site, the privacy threat of joining user identities on the two sites exists before the user ever engages with the portal. The privacy threat for portals is further detailed [here](https://github.com/WICG/portals#privacy-threat-model-and-restrictions).

Portal is a separate element type than fenced frames, but requires very similar restrictions in its communication with the embedding context as a fenced frame. It is thus likely that portals and fenced frames will converge on their cross-site tracking mitigations to a large extent.


### Interest Group ads based on user activity (TURTLEDOVE)

[TURTLEDOVE](https://github.com/michaelkleber/turtledove) allows for showing ads based on an advertiser-identified interest, in a privacy-preserving manner.

The following privacy aspects are required for turtledove:



*   Advertisers can serve ads based on an interest, but cannot combine that interest with other information about the person — in particular, with who they are (user’s identity on the embedding site) or what page they are visiting.
*   Web sites the person visits, and the ad networks those sites use, cannot learn about their visitors' ad interests.

These privacy requirements can be met using the fenced frame for rendering the ad. 

#### Design

Since the fenced frame has access to the user’s interest group information, as per the [Network access or Web bundles](#network-access-or-web-bundles) section, this use case aligns with restricting the network access. The interest group based ad should thus be fetched in advance as a web bundle.

The high level design for turtledove consists of two restricted environments:



*   The first one is responsible for doing the on-device auction and the output of that is the input to the fenced frame. This is a restricted javascript execution environment that does the on-device auction. This has the following characteristics:
    *   It is invoked by JS running in the regular publisher page environment, and use of the turtledove API creates this environment to run various pieces of ad-tech-written code in.
    *   It requires signals from the context that act as inputs to the on-device auction e.g. the publisher page’s topic.
    *   Since it is getting information from the embedding page, we need to make sure it is not exfiltrating that information to a backend server, and thus it is not allowed network access.
    *   Since the result of the turtledove API needs to be restricted from the embedding page, communication to the embedding page is not allowed.
    *   The output of this environment is opaque and not available to query via javascript. This makes the interest group of the user invisible to the embedding page. It points to an existing web bundle that is then passed to construct the fenced frame which is used to render the ad represented by this bundle.
*   The second environment is the fenced frame that renders the ad given the bundle from the above algorithm. However, how to use a fenced frame to support all the video creative use cases where streaming video is normally required remains a challenge.
*   Note that if the contextual ad wins the auction, it need not be rendered in the fenced frame. This will leak one bit conveying whether an interest group based ad won the auction or not but the upside is that the contextual ads do not need to change their ecosystem to be part of a fenced frame e.g. they do not need to use web bundles. 


### Conversion Lift Measurement

[Conversion Lift measurement](https://github.com/w3c/web-advertising/blob/master/support_for_advertising_use_cases.md#conversion-lift-measurement) studies are A/B experiments that ad providers perform to learn how many conversions were caused by their ad campaign vs how many happen organically. To be able to infer the causality of a conversion with the ad campaign, it requires deciding which experimental group the user should consistently be placed for a study (across sites) and show the ad creative corresponding to that group. (Related work: [Private lift measurement API by Facebook](https://github.com/w3c/web-advertising/blob/master/private-lift-measurement-third-party.md))

The following privacy aspects are required for lift measurement:
*   The embedding site should not know which experiment group the user belongs to or which ad got rendered as a result of an A/B experimentation API.

This is a privacy threat because if a publisher knows which experiment group a user is in for, say, n experiments, it gives the publisher an ‘n’ bits user identifier which can then be read on another site, forming a persistent cross-site identifier for the user.

These privacy requirements can be met using the fenced frame for rendering the ad. Since the fenced frame has the user’s experiment group information, as per the [Network access or Web bundles](#network-access-or-web-bundles) section, this use case aligns with restricting the network access.


#### Design

A high level flow of the design using fenced frames is given below:



*   Outside of the fenced frame, the ad auction returns two ad creatives using web bundles, one for the control arm and one for the experiment arm.
*   A restricted JS environment invokes the lift studies JS API. This JS environment, similar to the TURTLEDOVE case has no network, cannot communicate with the embedding page, and has an opaque output which is not available to query via javascript. This opaque output points to one of the ads from the control or experiment arms.
*   The fenced frame is then created with the ad creative web bundle information which was the opaque output from the restricted JS environment. The only way information can be extracted from the fenced frame is by using [aggregate measurement APIs](https://github.com/WICG/conversion-measurement-api/blob/master/AGGREGATE.md), via network access on user activation, or outbound navigation from the fenced frame.


### Unpartitioned storage access

The fenced frame does not have storage access by default. We want the fenced frame to have access to unpartitioned storage, if needed. 

As a reminder, the states of a fenced frame are:



1. Start
    1. Full network access, no storage access
2. On requesting read-only storage access without user activation
    2. No network access, read-only unpartitioned state (if requested)
3. On user activation
    3. Full network access, read/write unpartitioned state (if requested)

[requestStorageAccess](https://developer.mozilla.org/en-US/docs/Web/API/Document/requestStorageAccess) is used to provide access to unpartitioned storage. When invoked within fenced frames, there isn’t a need to show a permission prompt, thanks to the communication isolation of a fenced frame. 

There are a number of use cases for unpartitioned storage access. These include embedded media playing and enqueueing, document viewing and editing, social widgets, and article comments. requestStorageAccess within the fenced frame can be used to fulfill these use cases.

Although many of these use cases could be handled with a combination of user identification and server-side storage, the common way to identify users today is from their storage (cookies). Also, any offline use cases (such as offline docs) would require client-side storage.

## Security considerations

The fenced frame will comply with the security specific headers e.g. [CSP frame-ancestors](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/frame-ancestors), similar to how an iframe does. If the frame-ancestor header has the value 'none' (cannot be iframed), then the fenced frame will also not get created.  


## Privacy considerations

The fenced frame’s main goal is to improve privacy by disallowing communication with the embedder. There are however some attributes that might need to be shared between the two and their privacy impact needs to be carefully considered and mitigated, if possible. Some of these attributes are:



*   **Src:** Since the URL passed in the src attribute from the embedding site to the fenced frame is a channel as well, it needs to have similar or stricter link decoration mitigations than actual navigations. 
*   **Initial size and position attributes:** these could be restricted to a certain set of values e.g. multiples of 100 or popular values like full viewport width for mobile ads. This would not completely eliminate the channel but will restrict it. 
*   **Referrer:** This will be restricted to the site of the top-level page.
*   **IntersectionObserver:** It is important for ads reach and reporting APIs to know the status of the ad frame's visibility, so IntersectionObserver will need to be supported in a limited way, for instance by only letting it be consumed by browser APIs like [aggregate reporting API](https://github.com/csharrison/aggregate-reporting-api) or/and by limiting the number of bits exposed by the IntersectionObserver API. This is to make sure that embedding sites do not (re)position frames such that IntersectionObserver is used for communicating the user’s id to the fenced frame.
*   **Delegated permissions:** [Permission delegation](https://www.chromestatus.com/feature/5670617353289728) restricts permission requests to the top-level frame. Since fenced frames are embedded contexts, they should not have access to permissions, even if they are treated as top-level browsing contexts. Also delegation of permissions from the embedding context to the fenced frames should not be allowed as that could be a communication channel. 
*   **Document policy/feature policy headers:** The mitigations here need to be determined.

More of these channels exist and we are in the process of enumerating and finding the mitigations for all of those.

### Network side channel attack

The reason that fenced frames are restricted from writing to storage and in some cases also to the network before user gesture is to help mitigate against the following timing attack:

The fenced frame has access to sensitive user’s information, say, as a result of invoking a browser API:



*   Embedding site A sends a message to a tracking site, say tracker.example saying it is about to create a fenced frame and that the user id on A is 123.
*   The fenced frame is created and has access to user specific information X (e.g. user’s interest group for TURTLEDOVE). When the fenced frame document’s resources are requested from site B, X is also sent along. B can also let tracker.example know. The tracking site tracker.example can then correlate using the time/IP address bits of both requests.
*   A can then know X via tracker.com

The above is an example of a scenario where user id on A and user’s information on B can be joined without the user ever interacting with the fenced frame and such cases will benefit from not having network access but using a pre-existing web bundle to render the frame.

On the other hand, if there is no user specific information that the fenced frame has, restricting the network is not necessary until it gets access to its unpartitioned storage. It’s not ideal that this attack can occur after user gesture, and we consider this as one of the remaining [Challenges](#challenges) of Fenced Frames.

## Challenges

The following challenges are currently work in progress and would need to be resolved for fenced frames to be completely immune to cross-site identity joining:



*   Network timing attacks
*   User information provided in the fenced frame’s URL
*   IP address correlation between frames

These issues are not unique to fenced frames and also exist in cross-site navigations today so they could either depend on future solutions to these for cross-site navigations e.g. [willful IP blindness](https://github.com/bslassey/ip-blindness), or could have additional specific mitigations for fenced frames. These are currently being brainstormed.


## Considered alternatives

Both of the alternatives given in this section are applicable only if fenced frames were a type of iframe. As already described in the document above, they have the downside of spec and browser implementation complexity as many iframe capabilities will need to be special-cased for fenced frames. 


#### Using a new iframe attribute

Another way that was considered for this primitive was to have a new iframe attribute, say “fenced frame”. 


```
<iframe fenced-frame src="demo_iframe_fenced.html"></iframe>
```


Benefits



*   A single attribute defines the fenced frame so it is simpler to use by developers

Downsides



*   Iframe already has existing configuration attributes like sandbox, allow and policy and adding a fourth would lead to complexity in terms of how they all interact among each other. 

#### Using Feature policy/Permission policy

We considered using [feature policy](https://developers.google.com/web/updates/2018/06/feature-policy) attributes for network, storage and communication channels using the ‘allow’ keyword, instead of document policy.

Benefits over using document policy



*   Does not need HTTP header exchange as in document policy

Downsides



*   Feature policy’s objective is the delegation of powerful feature permissions to trusted origins while a fenced frame requires general features like inter-frame communication to be restricted on documents.
*   Since there is no HTTP header exchange there are more chances of site breakage due to restricting common features like network or inter-frame communications. 








