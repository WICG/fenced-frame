# Fenced frame modes



## **Summary**

[Fenced frames](https://github.com/shivanigithub/fenced-frame) have been discussed in the context of various use-cases. This document summarizes the various fenced frame modes.


## **Characteristics of the different modes**



*   All modes are similar such that they are isolated from the embedded context via any JS window references, script access, storage access, resizing of the fenced frame, messaging APIs etc.
*   Every mode of fenced frame is different in how its privacy guarantees are preserved and would need a separate launch/review process. The first phase and the associated review process of fenced frames will focus only on the **“opaque-ads”** mode. The **“default”** mode will also be available in the first version of fenced frames.
*   Each mode’s src attribute’s privacy characteristics are different E.g. the “opaque-ads” mode is for frames that can only be provided a urn:uuid src by the embedder, “read-only” mode has src that is known to the embedder and does not need to be mitigated against link decoration. A future “unpartitioned-storage” mode will need the src to be mitigated against link decoration.
*   It is important that the mode doesn’t change when there is a cross-document navigation. E.g. An “opaque-ads” mode fenced frame which has the interest group of the user, if navigated to the read-only mode from within the fenced frame can add the interest group to the target url and still get access to cookies thus creating a hybrid of two modes that we do not want to support.
*   Similar to the above point, a fenced frame tree of one mode cannot contain a child fenced frame of another mode.
*   One of the questions we are trying to answer from an API perspective is how to represent these different modes of fenced frames. Should they be separate elements all inheriting the base Fenced Frame element or whether there should be an attribute which can only be set once in the lifetime of the frame even across navigations. In phase 1, for simplicity, we are going with it as an attribute and then if need be and based on TAG/standards discussions, we can pivot to create separate elements.


## **Opaque-ads**

This mode is for rendering ads whose url is opaque to the embedding context. The two consumers of this mode are [FLEDGE](https://github.com/WICG/turtledove/blob/main/FLEDGE.md) and [SharedStorage](https://github.com/pythagoraskitty/shared-storage#simple-example-consistent-ab-experiments-across-sites). 



*   **Mode: “opaque-ads”**
*   **Source URL:** src should be a urn::uuid that should be mapped by the browser to an actual url. 
*   **Example** usage from FLEDGE:

    navigator.runAdAuction(myAuctionConfig).then((auctionWinnerUrl) => {


      // auctionWinnerUrl value e.g.


      // urn:uuid:f81d4fae-7dec-11d0-a765-00a0c91e6bf6


      var adFrame = document.createElement('fencedframe');

      adFrame.mode = "opaque-ads";

      adFrame.src = auctionWinnerUrl;


    });

*   **Information flow and privacy model:**
    *   Src is always guaranteed to be opaque to the embedding context via the urn:uuid mechanism described above. 
    *   The URL must be k-anonymous
    *   Size of the fenced frame is limited to a set of popular ad sizes.
    *   Like all modes, the fenced frame is isolated from the embedded context via any JS window references, script access, storage access, messaging APIs etc. The fenced frame does not know the embedding site’s origin or etld+1.
    *   The fenced frame is allowed to create a popup (with noopener) or navigate the top-level page on user activation as described [here](https://github.com/WICG/fenced-frame/blob/master/explainer/integration_with_web_platform.md#top-level-navigation). (This is an ads requirement)
*   **Cross-site data**: Interest groups in Fledge, the cross-site data used to choose the one-of-N URLs for shared storage. 
*   **Reporting**: Reporting for ads would eventually be done using aggregate reporting but for easier adoption there is event-level reporting that will be allowed. Events happening inside the fenced frames will be joined with the data from the FLEDGE/SharedStorage worklet and sent as a beacon. This is detailed [here](https://github.com/WICG/turtledove/blob/main/Fenced_Frames_Ads_Reporting.md)
*   **Additional notes**: Note that for this mode, the interesting part is that the source is opaque to the publisher and that is what is discussed  in the information flow section above. Having said that, if the fenced frame was navigated by the embedding frame to a non-opaque url, it is still accepted because in that case there isn't any cross-site data being accessed inside the fenced frame and its information flow then defaults to the **Default** mode described below. We could have restricted this mode to opaque urls only but that made local testing of this mode a bit cumbersome. As a side-effect, allowing non-opaque urls means this mode can be used for contextual ads to be displayed in a more private environment and still have access to some of the APIs like "Navigating the top-level page".  


## **Default mode**

This mode is the fenced frames with no special restrictions on the src and no cross-site data inside the fenced frame. It is useful to test fenced frames and can be used for a scenario where the fenced frame itself does not get any special cross-site data but the use case can benefit from the isolation characteristics of a fenced frame.



*   **Mode: “default”**
*   **Source URL:** [Potentially trustworthy](https://w3c.github.io/webappsec-secure-contexts/#potentially-trustworthy-url) url with no restrictions on it's contents
*   **Information flow and privacy model:**
    *   Since there is no unpartitioned/cross-site data available to the fenced frame, it is not able to do any cross-site data joining.
    *   Like all modes, the fenced frame is isolated from the embedded context via any JS window references, script access, storage access, messaging APIs etc. The fenced frame does not know the embedding site’s origin or etld+1.


## **Read-only**

This mode is for rendering personalized information in a fenced frame at the same time ensuring that the unpartitioned data accessed by the fenced frame cannot be exfiltrated, by disallowing network and any write access to storage . The two consumers of this mode are [3rd party payment service provider(PSP)'s customized payment buttons](https://github.com/shivanigithub/fenced-frame/issues/15) and possibly a version of [FedCM](https://github.com/fedidcg/FedCM) (the latter is under discussion). 



*   **Mode: “read-only”**
*   **Source URL:** [Potentially trustworthy](https://w3c.github.io/webappsec-secure-contexts/#potentially-trustworthy-url) url with no restrictions. Is able to carry 1p data from the embedding context to the fenced frame. This is not however an issue as described in the next section. 
*   **Information flow and privacy model:**
    *   The ‘src’ attribute can carry user id in the embedding page. Up until the fenced frame has completed navigation, there is an unrestricted network. That is fine because there isn’t any unpartitioned data available to the fenced frame till that point.
    *   The fenced frame is able to request access to read-only cookies after navigation is complete. At that point the network will be disallowed so that there is no exfiltration of joined data across sites via either the network or persistent storage. Othe partitioned state like storage, service workers, network and http cache will continue to stay partitioned.
*   **Top-level site’s etld+1**: This is required in the initial navigation request so that the PSP/FedCM servers can determine if this is a trusted top-level site that they are allowed to work with.
    *   This is fine because it does not exfiltrate any new information that the embedder itself could not have sent to the payment service provider's server.
*   **Cross-site data**: Read only access to unpartitioned cookies after navigation is complete (document and subresources have loaded) and network is then restricted.


## **Unpartitioned-storage**

This mode is for authenticated embeds e.g. embedded videos, comments etc. 



*   **Mode: “unpartitioned-storage”**
*   **Source URL:** The url in the src attribute should be mitigated against link decoration. (TODO: design) 
*   **Top-level site’s url**: This may be granted on user activation and permission prompt. (TODO: design)
*   **Information flow and privacy model:**
    *   User id from the embedding site is not allowed because of the source url being mitigated against link decoration.
    *   The fenced frame gets access to unpartitioned storage + top-level url (also mitigated against link decoration) on user activation and permission prompt.
*   **Cross-site data:** Gates unpartitioned storage access on user activation + permission prompt.
