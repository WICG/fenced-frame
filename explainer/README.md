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

Third party iframes can communicate with their embedding page using mechanisms such as postMessage, attributes (e.g., size and name), and permissions. A number of recently proposed APIs (such as [Interest group based advertising](https://github.com/WICG/turtledove), [Conversion Lift Measurements](https://github.com/w3c/web-advertising/blob/master/support_for_advertising_use_cases.md#conversion-lift-measurement)) provide some degree of unpartitioned storage/cross-site data to embedded documents. Once third-party cookies have been removed, such documents should not be allowed to communicate with their embedders, else they will be able to join their cross-site user identifiers with the embedder’s, which would allow for user tracking. This explainer proposes a new form of embedded document, called a fenced frame, that these new APIs can use to isolate themselves from their embedders, preventing cross-site recognition.

## Goals

The fenced frame enforces a boundary between the embedding page and the cross-site embedded document such that user data visible to the two sites is not able to be joined together. This can be helpful in preventing user tracking or other privacy threats. 

The privacy threat addressed is:

**The ability to correlate the user’s identity/information on the embedding site with that on the embedded site.**

The different use cases and their privacy model are discussed as the different fenced frame modes [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/modes.md).

## Design

Fenced frames are embedded contexts that have the following characteristics to prevent embedder identifiers from being joined with identifiers from the embedded site:



*   They’re not allowed to communicate with the embedder and vice-versa, except for certain information such as limited size information.
*   They access storage and network via unique partitions so no other frame outside a given fenced frame document can share information via these channels. This is described [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/storage_cookies_network_state.md). 
*   They may have access to browser-managed, limited unpartitioned user data, for example, turtledove interest group.

The idea is that the fenced frame should not have access to both of the following pieces of information and be able to exfiltrate a join on those:



*   User information on the embedding site
    *   Accessible via communication channels
*   User information on the fenced frame site
    *   Accessible via an API (e.g., Turtledove) or via access to unpartitioned storage  


A primary use case (Turtledove, Conversion Lift Measurement) for a fenced frame is to have read-only access to some unpartitioned storage, for example, in Turtledove, it is the interest-based ad to be loaded. The URL of the ad is sufficient to give away the interest group that the user belongs to, to the embedding site. Therefore the URL for the ad creative is an opaque url (details [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/opaque_src.md)) — which can be used for rendering, but cannot be inspected directly. 

We expect some leakage of information to be possible via network timing attacks. The side channel and some mitigations are described [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/network_side_channel.md).

### Fenced frame API 

The proposed fenced frame API is to have a new element type and treat it as a [top-level browsing context](https://html.spec.whatwg.org/#top-level-browsing-context). This section details this approach and later we also describe the alternative API approaches that were considered.


#### New element type - a top-level browsing context

In this approach, a fenced frame behaves as a top-level browsing context that is embedded in another page. This is aligned with the model that a fenced frame is similar to a “tab” since it has minimal communication with the embedding context and is the root of its frame tree and all the frames within the tree can communicate normally with each other. 


##### Example usage


```
<fencedframe src="demo_fenced_frame.html"></fencedframe>
```


  




*   Browser lets the server know via a new [`sec-fetch-dest`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Sec-Fetch-Dest) header value `fencedframe` to let it know that a request is from a fenced frame tree.
*   The server needs to opt-in to be loaded in a fenced frame or in an iframe embedded in a fenced frame tree. Without an opt-in, the document cannot be loaded. For opt-in, we use the [supports-loading-mode](https://github.com/jeremyroman/alternate-loading-modes/blob/main/opt-in.md#declaration) header with a new value of `fenced-frame`.

##### Benefits over nested browsing context



*   Simpler to spec since the fenced frame tree aligns well with a top-level browsing context.
*   Simpler to achieve the communications restrictions with the embedding context, being a top-level browsing context.


##### Downsides of a new element



*   Existing sites need to change to embed a new element. In order to not have all existing embedding sites replace iframe with a new type of element, third parties may create a fenced frame inside the iframe. This might have a performance cost though.


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

## Security considerations

Even though a fenced frame is isolated from its embedding context, it cannot be used as a workaround to the security restrictions that the top-level site wants to enforce on the embedding frames, without the knowledge of the top-level site. The design decisions of fenced frames related to security mechanisms like sandbox, csp, permission policy etc. are based on the folowing two principles:
* Attributes like cspee, sandbox etc. and headers like frame-ancestors etc. cannot be used as a communication channel with the embedding context.
* privelege escalation by fenced frame should not happen without the knowledge of the top-level site e.g. all permission policy delegation based features in a fenced frame are therefore disallowed.
* There are headers from the fenced frame site that are not honored as they would in an iframe, e.g. frame-ancestors, due to being a privacy leak. This is the reason fenced frames need to be opted in by the site using the opt-in response header.

More about security mechanisms are detailed in:
* [Fenced frames and CSP](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/interaction_with_content_security_policy.md)
* [Fenced frames and policies](https://docs.google.com/document/d/16PNR2hvO2oo93Mh5pGoHuXbvcwicNE3ieJVoejrjW_w/edit?usp=sharing)
* [Fenced frames and sandbox](https://docs.google.com/document/d/1RO4NkQk_XaEE7vuysM9LJilZYsoOhydfh93sOvrPQxU/edit?usp=sharing)

**xsleaks** In terms of cross site leak attacks, fenced frames is at least as secure as iframes are and better in some cases by default e.g. always having noopener, no joint history etc. For more details, the fenced frames xsleaks audit can be found [here](https://docs.google.com/spreadsheets/d/1YkQxcQlOd24XmSUQ8RpQU0zINYSTTih8drNibV0LIXE/edit?usp=sharing).

**Secure contexts:** Fenced Frames are only allowed if all ancestor frames are [secure contexts](https://w3c.github.io/webappsec-secure-contexts/), the FF's html document is a [potentially trustworthy URL](https://w3c.github.io/webappsec-secure-contexts/#potentially-trustworthy-url) and all subresources inside the FF will follow [mixed mode restrictions](https://web.dev/fixing-mixed-content/). 

**Process isolation:** Process isolation for fenced frames is detailed [here](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/process_isolation.md).


## Privacy considerations

The fenced frame’s main goal is to improve privacy by disallowing communication with the embedder. There are however some attributes that might need to be shared between the two and their privacy impact needs to be carefully considered and mitigated, if possible. Some of these attributes are:


*   **Initial size and resize:** To avoid the size attribute being used to communicate user identifying information from the embedding context to the fenced frame, this will be limited to only a few values. E.g. Some of the values that are relevant for ads for the "opaque-ads" mode. We are also considering allowing some of these sizes to be flexible based on the viewport width. Note that since size is a channel, these ads cannot be resized by the publisher. 
*   **IntersectionObserver:** It is important for ads reach and reporting APIs to know the status of the ad frame's visibility, so IntersectionObserver will need to be supported in a limited way, for instance by only letting it be consumed by browser APIs like [aggregate reporting API](https://github.com/csharrison/aggregate-reporting-api). This is to make sure that embedding sites do not (re)position frames such that IntersectionObserver is used for communicating the user’s id to the fenced frame. This is currently under design and intersection observer capability will be supported until the alternative is provided.
*   **Delegated permissions:** [Permission delegation](https://www.chromestatus.com/feature/5670617353289728) restricts permission requests to the top-level frame. Since fenced frames are embedded contexts, they should not have access to permissions, even if they are treated as top-level browsing contexts. Also delegation of permissions from the embedding context to the fenced frames should not be allowed as that could be a communication channel. This is detailed further in [Fenced Frames and Policies](https://docs.google.com/document/d/16PNR2hvO2oo93Mh5pGoHuXbvcwicNE3ieJVoejrjW_w/edit?usp=sharing).
*   **Network side channel:** This is detailed more here: [network side channel](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/network_side_channel.md)

More of these channels exist and the [integration with web platform](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/integration_with_web_platform.md) details them further.

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








