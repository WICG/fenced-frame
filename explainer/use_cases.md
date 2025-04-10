# Fenced Frame Use Cases


## Summary

This document describes some initial targeted use cases of the `<fencedframe> `element on the Web. For each, we’ll answer the following questions:



1. What parties are involved in this particular use case? This may include end users, web developers, or other parties like advertisers or payments providers.
2. Why does this use case require fenced frames?


## Common factors across all use cases

Regardless of the context in which they are used, the primary goal of fenced frames remains the same. Fenced frames are designed to allow websites to embed content from other sites, with their cross-site data, in a way that intends to block communication between the embedder and fenced frame so that the cross-site data is not joinable

Fenced frames largely achieve this by isolating themselves from their embedding context. A fenced frame and its embedder cannot access one another via JS window references, script access, storage access, messaging APIs, powerful permissions-policy-gated features, etc. Network constraints to prevent backend joins are forthcoming.

The other documents in this repository go into much greater detail about the actual isolation mechanisms. This document focuses on how this isolation can be combined with other features of the Privacy Sandbox to provide privacy-respecting alternatives to critical use cases.


## Rendering Ads via the Protected Audience API

The Protected Audience API enables on-device ad auctions by the browser, to choose relevant ads from websites the user has previously visited, without having to know who the user is. 

The Protected Audience API provides a privacy-friendly replacement for previous technologies used in advertising, like third-party cookies, which the Web is moving towards deprecating. The full details of this API can be found in the [explainer](https://github.com/WICG/turtledove/blob/main/FLEDGE.md), but here’s a brief summary:



1. When a site wants to sell ad space, the browser performs an on-device ad auction via `const config = await navigator.runAdAuction(...)`. The bidders in the auction are represented by interest groups, which are stored in the browser at an earlier time on an advertiser site via `navigator.joinAdInterestGroup()`.
2. The content of the winning ad is rendered in a fenced frame on the seller’s site and shown to the user. The `config` returned after running the auction is a `FencedFrameConfig`, which is used to load the ad content into a fenced frame. Some parameters of the `config`, such as the source URL, are redacted before being provided to the embedder. This redaction prevents the embedder from learning any information that was produced based on cross-site data. See the [explainer](https://github.com/WICG/fenced-frame/blob/master/explainer/fenced_frame_config.md) page for` FencedFrameConfig` for full details on how these are used.


Using fenced frames to render the ad creative provides some significant privacy benefit to the end user. 



1. A user’s interest group memberships constitute cross-site data. The embedder can never learn about the user’s interests when loading the fenced frame, because the `FencedFrameConfig` it receives doesn't expose any relevant information to the web platform. In addition, the information passed to the fenced frame will be _[k-anonymous](https://github.com/WICG/turtledove/blob/main/FLEDGE_k_anonymity_server.md)._
2. Once the config has been loaded into the fenced frame and the ad is rendered, the embedder and the ad content cannot directly communicate with one another via web platform APIs. And since they don’t have third-party cookies, the only remaining ways to join user identity are via channels that we intend to deprecate over time (such as event level reporting) and via network fingerprinting attacks (which we’re also working on addressing). 

**Note: Today, Protected Audience API auctions allow the winning ad to be rendered in either an iframe or a fenced frame. This is to facilitate a smooth transition to fenced frames. Starting no earlier than 2026, auctions will only allow ads to be rendered in fenced frames.**


## Selecting and rendering a URL based on cross-site data via SelectURL

Aside from executing ad auctions, there are other legitimate use cases on the Web that may rely on cross-site data, such as consistent a/b testing of content across sites, or having consistent display preferences for a embedded widget across sites.. The SelectURL API was created as a flexible mechanism to address these needs. More details can be found in the [explainer](https://github.com/WICG/shared-storage). Here’s a brief summary, as it relates to fenced frames:



1. Data is written to Shared Storage via setter methods, such as `sharedStorage.set()` and `sharedStorage.append()`. Setter methods are available in traditional browsing contexts via `window.sharedStorage`, and add data that can be accessed cross-site.
2. Access to the data occurs in a restricted context called a worklet, which is not allowed to directly communicate Shared Storage data to the page. 
3. Instead, the page calls `sharedStorage.selectURL()` to select one of up to 8 possible URLs to display to the user based on this cross-site data. This method accepts a list of URLs, and returns a redacted `FencedFrameConfig` that can be used to load one of the URLs within a fenced frame.
4. The chosen URL is determined in the worklet by accessing Shared Storage data via `sharedStorage.get()`. However, the browser redacts this URL when placing it into the `FencedFrameConfig`, which prevents the page calling `selectURL()` from determining how cross-site data influenced the outcome. For a code example in a real-world context, take a look at [this section](https://github.com/WICG/shared-storage#simple-example-consistent-ab-experiments-across-sites) of the Shared Storage explainer.

This framework is generic enough that it can fill many niches, but some specific use cases that have been identified for advertising are:



1. A/B testing. Sites can test advertisements or other types of content by partitioning users into experiment groups via a seed placed into Shared Storage.
2. Respecting user display preferences for widgets across sites.

Much like the Protected Audience API, `selectURL` allows sites to make decisions about what content to show users based on cross-site data, but takes steps to prevent exposure of that data using an isolated worklet, a fenced frame, and a redacted `FencedFrameConfig`. Cross-site data remains contained to the worklet, and the URL selected as a result of that data is unknown to the embedding page.

**Note: Today, <code>sharedStorage.selectURL()</code> allows the chosen URL to be rendered in either an iframe or a fenced frame. This is to facilitate a smooth transition to fenced frames. Starting no earlier than 2026, the chosen URL will only be rendered in a fenced frame.**


## Personalizing Document Content with Local Unpartitioned Data Access

**_[Not launched as of H1 2024]_**

We would eventually like to constrain fenced frames in such a way that their network communications do not leak significant amounts of cross-site user data. Without arbitrary network access, we can allow the fenced frame access to user's cross-site data. This would allow for cross-site rendering customizability, which is needed for some use cases such as for third-party payment providers. This section describes a use case in which the fenced frame’s script voluntarily disables its network access, in order to access cross-site data.

Merchants often delegate payment for goods and services to third-party payment providers, and merchants integrate with those providers by embedding a button into their site. However, payment providers on the Web today often decorate their buttons with information specific to the user, such as the last four digits of their credit card, or the card’s carrier (like Visa or MasterCard). This assures the user visiting the merchant that their transaction will occur smoothly. If the user is more confident in their purchase experience, this benefit will then be passed onto the businesses of the payments provider and the merchant.

Currently, this experience relies on unpartitioned user data (like third-party cookies) being available across sites. Once the user’s card data is stored by the payment provider in a first-party context, it needs to be accessible to their payment buttons embedded in third-party contexts. Fenced frames seek to re-enable this use case via the “local unpartitioned data access” proposal, which can be found in this [explainer](https://github.com/WICG/fenced-frame/blob/master/explainer/fenced_frames_with_local_unpartitioned_data_access.md) document. Here’s a brief summary:



1. Fenced frames will be allowed to read data from Shared Storage outside of a worklet via `sharedStorage.get()`, and they can use that data to personalize the content of the frame. 
2. This data cannot be directly exfiltrated to the embedder given fenced frames’ isolation mechanisms, but it could still be sent over the network. As a result, calling `sharedStorage.get()` will not be allowed until the frame’s document has its network access revoked by resolving a call to `window.fence.disableUntrustedNetwork()`. Network access removal encompasses navigation, subresource fetches, alternative networking APIs like Websockets, and special fenced frame mechanisms like event-level reporting.
3. The cross-site data will remain local to the frame while still allowing a personalized experience for the user. If the frame’s document needs to communicate to the embedder that the user clicked on the content, it can do so via the new `window.fence.notifyEvent()` API. This fires an event at the `HTMLFencedFrameElement` in the embedder containing no cross-site information, which in the case of payment buttons can be used to initiate the checkout flow on the merchant site.

For an end-to-end example of how this setup could work in practice, take a look at this sample code in the [explainer](https://github.com/WICG/fenced-frame/blob/master/explainer/fenced_frames_with_local_unpartitioned_data_access.md#code-example).


This “local unpartitioned data access” mode can be enabled on every type of fenced frame, whether they were constructed with a redacted `FencedFrameConfig` or a plaintext URL. Once the frame has disabled network access, all types of fenced frames should be able to benefit from content personalization to support relevant use cases.


## Manual Construction for General Purpose Usage and Testing

The Protected Audience API and the Shared Storage API rely upon a redacted `FencedFrameConfig` to render the fenced frame. This is because the frame is constructed via cross-site data brokered by the browser, and the opacity ensures that the embedder does not access that sensitive data in any way. However, what if the frame does not need to be constructed using cross-site data, or it only uses cross-site data after disabling network access as described in the above section?

Example scenarios include: 



*   A web developer is prototyping a site that will render ads in fenced frames, but wants to test their placement with sample content before integrating with the Protected Audience API.
*   A third-party payments provider currently embeds a personalized button as described above, but personalization only occurs after calling `window.fence.disableUntrustedNetwork()`.
*   A developer for a merchant site wants to integrate with that payments provider, and needs to test that their payment flow still works when a fenced frame is used. 

In these cases, the developer does not need to rely on a config-generating API. They likely know the exact URL of the content that they want to embed, and the URL is not constructed from cross-site data. To meet these needs, fenced frames are also manually constructable:

<code>const frame = document.createElement("fencedframe");
 const config = new FencedFrameConfig("https://example.com/");
 frame.config = config;
 document.body.append(frame);</code>


In addition to allowing smoother adoption of fenced frames in a testing environment, manual construction will allow documents loaded from a known URL to get all of the isolation benefits of fenced frames for free. This presents a clear privacy advantage in situations where communication with the embedder is not essential to the page’s functionality. 

In Chrome, manual construction is currently behind a flag. To test it out, enable the following experiments in `chrome://flags` and then reopen the browser. 



*   Privacy Sandbox Ads APIs (`chrome://flags/#privacy-sandbox-ads-apis`)
*   Enable the `FencedFrameConfig` constructor (`chrome://flags/#enable-fenced-frames-developer-mode`)
