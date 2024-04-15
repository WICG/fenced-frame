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
        - [Downsides of a new element](#downsides-of-a-new-element)
    - [Fenced frame tree](#fenced-frame-tree)
    - [Information channel between fenced frame and other frames](#information-channel-between-fenced-frame-and-other-frames)
  - [Security considerations](#security-considerations)
  - [Privacy considerations](#privacy-considerations)
    - [Ongoing technical constraints](#ongoing-technical-constraints)  
  - [Parallels with Cross-site portals](#parallels-with-cross-site-portals)
  - [API alternatives considered](#api-alternatives-considered)
      - [Using iframe with document policy](#using-iframe-with-document-policy)
      - [Using a new iframe attribute](#using-a-new-iframe-attribute)
      - [Using Feature policy/Permission policy](#using-feature-policypermission-policy)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Explainer - Fenced Frames

## Authors
*   Shivani Sharma
*   Josh Karlin

## Introduction
In a web that has its cookies and storage partitioned by top-frame site, there are occasions (such as [Interest group based advertising](https://github.com/WICG/turtledove) or [Conversion Lift Measurements](https://github.com/w3c/web-advertising/blob/master/support_for_advertising_use_cases.md#conversion-lift-measurement)) when it would be useful to display content from different partitions in the same page. This can only be allowed if the documents that contain data from different partitions are isolated from each other such that they're visually composed on the page, but unable to communicate with each other. Iframes do not suit this purpose since they have several communication channels with their embedding frame (e.g., postMessage, URLs, size attribute, name attribute, etc.). We propose fenced frames, a new element to embed documents on a page, that explicitly prevents communication between the embedder and the frame.

## Goals

The fenced frame enforces a boundary between the embedding page and the cross-site embedded document such that user data visible to the two sites is not able to be joined together. This can be helpful in preventing user tracking or other privacy threats. 

**Caveat:** It could still be possible for documents colluding via covert channels to be able to communicate information (See [Ongoing technical constraints](#ongoing-technical-constraints) for more details). 

The privacy threat addressed is:

**The ability to correlate the user’s identity/information on the embedding site with that on the embedded site.**

The different use cases and their privacy models are discussed [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/use_cases.md).

## Design

Fenced frames are embedded contexts that have the following characteristics to prevent embedder identifiers from being joined with identifiers from the embedded site:



*   They’re not allowed to communicate with the embedder and vice-versa, except for certain information such as limited size information.
*   They access storage and network via unique partitions so no other frame outside a given fenced frame document can share information via these channels. This is described [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/storage_cookies_network_state.md). 
*   They may have access to browser-managed, limited unpartitioned user data, for example, turtledove interest group.

The idea is that the fenced frame should not have access to both of the following pieces of information and be able to exfiltrate a join on those:



*   User information on the embedding site
    *   Accessible via communication channels
*   Information from other top-site partitions
    *   Accessible via an API (e.g., Turtledove) or via access to unpartitioned storage  


A primary use case for fenced frames is to load content that depends on values in another partition’s storage. For example, in Turtledove, we pick an ad based on the user's interest groups (which are joined while browsing other sites) and load it in a fenced frame. The URL of the ad reflects the user's interest group memberships, which is a form of cross-site data, therefore we store the URL for the ad creative _opaquely_ in a fenced frame config (details [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/fenced_frame_config.md)). The embedder can use this object to load the ad resulting from the Turtledove auction, but can't inspect it to determine _which_ ad won.

We expect some leakage of information to be possible via network timing attacks. The side channel and some mitigations are described [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/network_side_channel.md).

### Fenced frame API 

The proposed fenced frame API is to have a new element type and treat it as a [top-level browsing context](https://html.spec.whatwg.org/#top-level-browsing-context). This section details this approach and later we also describe the alternative API approaches that were considered.


#### New element type - a top-level browsing context

In this approach, a fenced frame behaves as a top-level browsing context that is embedded in another page. This is aligned with the model that a fenced frame is similar to a “tab” since it has minimal communication with the embedding context and is the root of its frame tree and all the frames within the tree can communicate normally with each other. 
Since fenced frames are embedded frames, they also behave like iframes in many ways. For example:
* Browser extensions will access a fenced frame as an iframe, e.g., for ad blocking.
* Browser features like accessibility, developer tools etc. will access a fenced frame like an iframe.


##### Example usage


```
const fencedframe = document.createElement('fencedframe');
fencedframe.config = new FencedFrameConfig('demo_fenced_frame.html');
```

*   Browser lets the server know via a new [`sec-fetch-dest`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Sec-Fetch-Dest) header value `fencedframe` to let it know that a request is from a fenced frame tree.
*   The server needs to opt-in to be loaded in a fenced frame or in an iframe embedded in a fenced frame tree. Without an opt-in, the document cannot be loaded. For opt-in, we use the [supports-loading-mode](https://github.com/jeremyroman/alternate-loading-modes/blob/main/opt-in.md#declaration) header with a new value of `fenced-frame`.

##### Benefits over nested browsing context



*   Simpler to spec since the fenced frame aligns well with a top-level browsing context.
*   Simpler to achieve the communications restrictions with the embedding context, being a top-level browsing context.


##### Downsides of a new element



*   Existing sites need to change to embed a new element. This is not really a downside though, since even if it was an enhancement to the iframe element, existing sites would have needed to make changes either in the attributes or in the headers to differentiate it from a regular iframe. 


### Fenced frame tree

A fenced frame is the root of the fenced frame tree. The root fenced frame and any child iframes in this tree are not allowed to use communication channels to talk to frames outside the tree or vice-versa. The frames within the tree can communicate with each other normally. 

### Information channel between fenced frame and other frames

There are many channels between the fenced frame tree and the other frames that will need to be restricted and a few of them are listed below:



*   PostMessage
*   Name, allow, cspee and other attributes
*   Resize 
*   Access to window.parent/ window.top etc.
*   Events fired simultaneously in the embedding context and the fenced frame such as page lifecycle events like onload
*   …

This discussion assumes that third-party cookies, like all other third party storage,  are also [disallowed](https://blog.chromium.org/2020/01/building-more-private-web-path-towards.html) or else those would be a communication channel between the fenced frame and the embedding site.

### HTMLFencedFrameElement class

All fenced frame related functions will live in its own class, in the same way that iframe-related funcionality lives in HTMLIFrameElement.

#### Can Load API

There are various reasons a fenced frame config with an opaque url could refuse to load in a page. For example, if the page is not in a secure context, or if CSPEE is specified in the embedding frame, the fenced frame config will refuse to load. This is a lot for a developer to keep track of.

If the process of getting an ad in the page is complex or expensive, there needs to be a way to ensure that the resulting ad will actually end up in the page before the expensive process begins.

A static API method will be introduced to the HTMLFencedFrameElement class to check this. No fenced frame will be created when calling this API, and it can be invoked before actually attempting to load a fenced frame config. The API will return a boolean, true if a config with an opaque mapped url would be able to load in the caller's context, false if not.

##### Example usage

```
HTMLFencedFrameElement.canLoadOpaqueURL();
```
```
> true
```

This is called synchronously, and will look at the execution context of the frame invoking the API.

## Security considerations

Even though a fenced frame is isolated from its embedding context, it cannot be used as a workaround to the security restrictions that the top-level site wants to enforce on the embedding frames, without the knowledge of the top-level site. The design decisions of fenced frames related to security mechanisms like sandbox, csp, permission policy etc. are based on the following principles:
* Attributes like cspee, sandbox etc. and headers like frame-ancestors etc. cannot be used as a communication channel with the embedding context.
* Fenced frame should not be able to escalate privileges without the knowledge of the top-level site e.g. all permission policy delegation based features in a fenced frame are therefore disallowed.
* There are headers from the fenced frame site that are not honored as they would in an iframe, e.g. frame-ancestors, due to being a privacy leak. This is the reason fenced frames need to be opted in by the site using the opt-in response header.

More about security mechanisms are detailed in:
* [Fenced frames and CSP](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/interaction_with_content_security_policy.md)
* [Fenced frames and policies](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/permission_document_policies.md)
* [Fenced frames and sandbox](https://docs.google.com/document/d/1RO4NkQk_XaEE7vuysM9LJilZYsoOhydfh93sOvrPQxU/edit?usp=sharing)

**Secure contexts:** Fenced Frames are only allowed if all ancestor frames are [secure contexts](https://w3c.github.io/webappsec-secure-contexts/), the fenced frame's document is from a [potentially trustworthy URL](https://w3c.github.io/webappsec-secure-contexts/#potentially-trustworthy-url) and all subresources inside the FF will follow [mixed mode restrictions](https://web.dev/fixing-mixed-content/). 

**Inheritance for local resources:** Documents hosting [local](https://fetch.spec.whatwg.org/#is-local) resources inherit their [policy containers](https://html.spec.whatwg.org/multipage/origin.html#policy-container) from their initiator or parent document, however for fenced frames, no such inheritance will take place. Fenced frames hosting local Documents will have a fresh policy container as they were created with no initiator document, just like the first Document in a top-level browsing context created with no initiator document.

**xsleaks:** In terms of cross site leak attacks, fenced frames is at least as secure as iframes are and better in some cases by default e.g. always having noopener, no joint history etc. For more details, the fenced frames xsleaks audit can be found [here](https://docs.google.com/spreadsheets/d/1YkQxcQlOd24XmSUQ8RpQU0zINYSTTih8drNibV0LIXE/edit?usp=sharing).

**Process isolation:** Process isolation for fenced frames is detailed [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/process_isolation.md).


## Privacy considerations

The fenced frame’s main goal is to improve privacy by disallowing communication with the embedder. There are however some attributes that might need to be shared between the two and their privacy impact needs to be carefully considered and mitigated, if possible. Some of these attributes are:


*   **Initial size and resize:** The API that generates a fenced frame config can pick the initial size that the fenced frame document sees, subject to whatever restrictions it deems necessary for its privacy model. If the initial size is fixed, then any changes the embedder attempts to make to the fenced frame's size will not be reflected inside of it.
*   **Intersection Observer:** See [Integration with web platform > Viewability](https://github.com/WICG/fenced-frame/blob/master/explainer/integration_with_web_platform.md#Viewability) for discussion of the privacy considerations for the Intersection Observer API.
*   **Delegated permissions:** [Permission delegation](https://www.chromestatus.com/feature/5670617353289728) restricts permission requests to the top-level frame. Since fenced frames are embedded contexts, they should not have access to permissions, even if they are treated as top-level browsing contexts. Also delegation of permissions from the embedding context to the fenced frames should not be allowed as that could be a communication channel. This is detailed further [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/permission_document_policies.md).
*   **Network side channel:** This is detailed more here: [network side channel](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/network_side_channel.md)
*   **Navigation url:** Since fenced frames are allowed to open popups or navigate the top-level page in some use cases, gated on user activation, the navigation url can carry bits of information out of the fenced frame tree. If the embedder and the destination are same-origin, the information in the url and embedder's info can be joined locally on navigation. This might need mitgations going forward (currently being brainstormed). Additionally, this is vulnerable to the network side channel as mentioned above when the embedding site and destnation site are colluding.  

More of these channels exist and the [integration with web platform](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/integration_with_web_platform.md) details them further.

### Ongoing technical constraints
Fenced frames disable explicit communication channels, but it is still possible to use covert channels to share data between the embedder and embeddee, e.g. global socket pool limit (as mentioned in the [xsleaks audit](https://docs.google.com/spreadsheets/d/1YkQxcQlOd24XmSUQ8RpQU0zINYSTTih8drNibV0LIXE/edit?usp=sharing)), network side channel and intersection observer as described above, etc. Mitigations to some of these are being brainstormed. We also believe that any use of these known covert channels is clearly hostile to users and undermines web platform intent to the point that it will be realistic for browsers to take action against sites that abuse them. 

## Parallels with Cross-site portals

[Portals](https://wicg.github.io/portals/) allow for rendering of, and seamless navigation to, embedded content.

If the embedded content is cross-site, the privacy threat of joining user identities on the two sites exists before the user ever engages with the portal. The privacy threat for portals is further detailed [here](https://github.com/WICG/portals#privacy-threat-model-and-restrictions).

Portal is a separate element type than a fenced frame, but requires very similar restrictions in its communication with the embedding context as a fenced frame. It is thus likely that portals and fenced frames will converge on their cross-site tracking mitigations to a large extent.

## API alternatives considered

Both of the alternatives given in this section are applicable only if fenced frames were a type of iframe. As already described in the document above, they have the downside of spec and browser implementation complexity as many iframe capabilities will need to be special-cased for fenced frames. 

#### Using iframe with document policy

In this alternative approach the fenced frame is a nested browsing context with the communications restrictions placed on an iframe via a [document policy](https://github.com/w3c/webappsec-feature-policy/blob/master/document-policy-explainer.md).


Example usage

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


Benefits



*   Able to work with the existing “iframe” element


Downsides



*   Much more complex spec and browser implementation since many iframe features will need to be special-cased for fenced frames.

There were other alternatives considered for the iframe approach like using feature policy or a new attribute, detailed later in this document.


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
*   Since there is no HTTP header exchange there are more chances of site breakage due to restricting common features like inter-frame communications. 








