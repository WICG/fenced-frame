# **Fenced Frames with local unpartitioned data access**


## Introduction and Goals

There are situations in which it is helpful to decorate a third-party widget with cross-site information about the user, such as a personalized payment button that displays credit card information to give the user confidence that the payment flow will be smooth, or a personalized sign-in button. These sorts of use cases will be broken by [third-party cookie deprecation](https://privacysandbox.com/open-web/) (3PCD). 

Fenced frames are a natural fit for such use cases, as they allow for frames with cross-site data to be composed within a page of another partition. The idea proposed here is to allow fenced frames to have access to the cross-site data stored for the given origin within [shared storage](https://github.com/WICG/shared-storage).  In other words, the payment site would add the user’s payment data to shared storage when the user visits the payment site, and then read it in third-party fenced frames to decorate their payment button. 

To prevent the fenced frame from leaking the user’s shared storage data out (to the embedder or to servers via network) we’re requiring that fenced frames first disable all untrusted network[1] communications before accessing shared storage. Note that the eventual intent is that all fenced frames would disable untrusted network communication anyway, so this isn’t exactly new, but it’s particularly vital that we do so when allowing access to shared storage since it has unbounded cross-site information available to it, compared to e.g. k-anonymous cross-site information available in Protected Audience ads.

The motivation for this variant of fenced frames are customized payment buttons for third-party payment service providers  (as discussed in this [issue](https://github.com/WICG/fenced-frame/issues/15)) but this proposal is not intended to be restricted to payments. Any form of third-party UX that wishes to show personalized information to a user, without leaking that information to the embedder, could use it.

[1] More details [below](#why-untrusted-network) on untrusted vs trusted network 


## User experience

It’s been [noted](https://developers.googleblog.com/2021/05/updated-google-pay-button-increases-click-through-rates.html) that personalized payment buttons (buttons that display saved payment methods) can significantly increase click-through rates as it provides increased confidence that the checkout flow will be smooth. With this proposal, users will continue to see the personalized button.

## Information flow and design

 The sequence of steps to achieve the user experience as mentioned above are given below:



1. **Same as today:** A user visits the payment provider’s site as a first party and enters their payment method details, for later use on merchant sites.
2. **New:** After step 1, the payment provider decides what  information is needed from the user's payment method details to render in a subsequent personalized button and writes it to shared storage (reasons for selecting shared storage discussed [here](#shared-storage-access)). For example, if the user details needed in the personalized payment button are the last 4 digits of the user’s credit card, the JS on the payment provider’s page will invoke the following:

    ```
    window.sharedStorage.set("last-4-digits", value)
    ```


3. **Same as today:** The user, at a later time, visits the merchant’s site and the payment provider’s script, say, payment.js, on the merchant site decides to create a personalized button. Note that payment.js runs on the merchant’s page, either directly on the 1p page or in an iframe belonging to the payment provider.
4. **New:** For creating the personalized button, payment.js creates a fenced frame instead of an iframe as it does today, using the following FencedFrameConfig constructor:

    ```
    fenced_frame = new FencedFrameConfig('https://examplepay.com/button.html?<arbitrary bits>');
    ```



    **Privacy impact:** Note that it is ok for the source of the fenced frame to carry arbitrary data from the creator context because that data cannot be joined with cross-site data until after the network is restricted.

5. **New: State 1 of the fenced frame (full network):** In this state, when the fenced frame is created, it doesn’t have any unpartitioned data access. The fenced frame in this state has unrestricted network access which can be utilized to load the HTML document and subresources in the fenced frame (such as the button image).

    Note that in this state the content in the fenced frame is not personalized, since it hasn’t accessed any unpartitioned data. 

6. **New: State 2 of the fenced frame (disable untrusted network):** In order to access unpartitioned data to personalize the contents of the fenced frame, the fenced frame needs to block off network access. As mentioned above in the privacy impact section this will prevent any data joined across the embedding site and the fenced frame site from exfiltrating. There will be a new API introduced to do that: `window.fence.disableUntrustedNetwork();`
7. **New: State 2 of the fenced frame (access unpartitioned data):** Finally after exfiltration channels have been blocked off via window.fence.disableUntrustedNetwork(), the FF content can access unpartitioned data via shared storage. The data that was written to shared storage in step 2, can now be read and rendered to personalize the content via the `sharedStorage.get()` method.

There we go: the user now sees the personalized button!!



8. **Same as today:** If the user wants to proceed with the payment transaction, they can click on the button.
9. **New:** To proceed with the payment transaction, payment.js needs to be able to listen to the click that happened on the button via the new click API surface described [here](#click-listener-api).
10. **Same as today:** Once the information that a click happened inside the FF is available in the embedding context, the payment.js opens a new top-level context as a new tab or invokes the [Payment Handler API](https://developer.mozilla.org/en-US/docs/Web/API/Payment_Handler_API) and the transaction proceeds as it does today.


## Shared Storage Access

This section goes into why we chose shared storage for this solution. We considered alternatives like Cookies and Local Storage as described in the “Alternatives considered” section.


### **Data Requirements**

Let’s first go through the requirements on the data, taking the example of examplepay.com as the payment provider:



*   **Readability**: The data needs to  be readable in an examplepay.com fenced frame (after untrusted network has been revoked) and not in an examplepay.com top-level frame or iframe.
    *   The data only needs to be read inside the FF from JS and not via network headers.
*   **Writability**: The data only needs to be writable in a 1p context from examplepay.com.
*   **Scope:** The data should be scoped to the origin examplepay.com. 
*   **Size:** As per the current use case feedback, the data itself is not very large, is text-only and can fit in a normal cookie size. This is subject to change though, e.g. if a payment provider wants to also show the user's profile picture in the personalized button.
*   **Unpartitioned:** The data only needs to be accessed in the unpartitioned state and not in a partitioned (by top-level site i.e. the merchant site for payment providers) state.

Shared Storage is a Privacy Sandbox API that allows unpartitioned storage access with restricted output gates as described [here](https://github.com/WICG/shared-storage/blob/main/README.md). The existing output gates for shared storage are private aggregation report and opaque URL selection. The proposal here is to introduce a new output gate:** read from a fenced frame after network revocation**.

Some of the enhancements needed for shared storage to be used for this proposal are:



*   Create a new shared storage output gate: a fenced frame with no network. Privacy-wise this is safe, since the unpartitioned data being read is not exfiltrated either to the embedding context (because of the nature of fenced frames) or via writing to any unpartitioned storage (like local storage, since FFs have a unique, ephemeral storage partition) or via network. 
*   The API `sharedStorage.get('<key>')` will need to be enabled inside the fenced frame and can only resolve successfully once the network is cut-off. 

Note that there is no change needed to Shared Storage write mechanisms, since it is already writable from both 1p and 3p contexts and this use case only needs 1p write access. 

**Benefits**

The benefits of using shared storage for this solution are primarily how it aligns with all of the requirements mentioned above.



*   The incremental enhancements to shared storage for this use case aren’t complicated/hard to integrate in the existing shared storage design/implementation.
*   Shared storage, is by definition, unpartitioned data. There is no notion of partitioned data for this use case as mentioned in the Requirements section above, which makes it more aligned with using shared storage.
*   Shared storage is by JS-readable only and thus aligns with the requirement.
*   Additionally, shared storage APIs’ invocation is gated behind [enrollment](https://developer.chrome.com/en/docs/privacy-sandbox/enroll/) which offers additional, policy-based, privacy protection.

**Downsides**



*   Browser compatibility: Even though other browsers should not see concerns with the privacy aspect of this output gate of shared storage, it’s more work to implement for other browsers than exposing cookies or localStorage to a fenced frame since they haven’t implemented shared storage yet. See the LocalStorage section to see mitigations for this.


## Revoking Network Access

At a high level, the new API window.fence.disableUntrustedNetwork() allows a fenced frame to disable exfiltrating information (primarily via network requests) in exchange for gaining access to unpartitioned data via sharedStorage reads.

Some more details:



*   Even though this information flow only relies on fenced frames created using the url constructor, we would not be disallowing the usage of window.fence.disableUntrustedNetwork() in FFs created using the other APIs as the eventual intent is for all fenced frames to have restricted network access. It will be available from fenced frames whose configs are generated by all current methods:
    *   navigator.runAdAuction(...)
    *   sharedStorage.selectURL(...)
    *   FencedFrameConfig(url)
*   As a result of invoking this API, network is disabled in the complete fenced frame tree, i.e. in the root fenced frame and any of its embedded iframes. 
*   We are still thinking about how disableUntrustedNetwork() should interact with nested fenced frames. The alternatives are: 
    *   Any embedded fenced frames within the parent FF should also have invoked disableUntrustedNetwork() before the parent FF can access shared storage.
    *   Calling disableUntrustedNetwork() disables network for all child fenced frames.
*   Disabling network disables the following:
    *   **Subresources requests**: This includes resource requests like for scripts, images etc. or APIs like sendBeacon, etc.
    *   **Navigation requests:** This implies that there cannot be a navigation initiated either for loading a document inside the FF tree or for navigating the top-level page or opening a new tab/window.
    *   **Event level reporting:** Any event level reporting mechanisms supported in fenced frames, i.e., [fenced frames ads reporting](https://github.com/WICG/turtledove/blob/main/Fenced_Frames_Ads_Reporting.md).
    *   **Any other network channels:** This includes any channel not covered in the above categories like WebSocket, web workers, etc.
*   **In-progress network requests:** Allowing in-progress network requests to continue could lead to inadvertent privacy leaks. For example, a possible attack could arise where multiple requests are initiated and, based on unpartitioned data, some of them are canceled via the [abort API](https://developer.mozilla.org/en-US/docs/Web/API/AbortController). Therefore the proposal is to cancel any ongoing requests. 
    *   This implies that disableUntrustedNetwork() should only be invoked when the critical resources have been fetched completely, such as by listening to the[ load event](https://developer.mozilla.org/en-US/docs/Web/API/Window/load_event).
    *   The API design to handle in-progress requests is currently under discussion. 


### Why “untrusted” network

Certain privacy preserving aggregated reports can be allowed from the fenced frame after disableUntrustedNetwork() has been invoked. After the untrusted network access has been revoked, with the correct permission policies, the fenced frame should be able to invoke APIs like the [Private Aggregation API](https://developer.chrome.com/docs/privacy-sandbox/private-aggregation/#what-is-the-private-aggregation-api) which allows aggregated data to be sent out on the network as that inherently disallows any arbitrary data exfiltration.  There may be additional trusted network communications in the future, such as to a secure trusted execution environment.


## Click listener API

The click listener API is broken into two parts:

*   The embedding context will invoke `addEventListener()` on the `HTMLFencedFrameElement` to listen for a click event on the fenced frame.
*   A script inside the fenced frame tree will invoke a new method on `window.fence`, which will trigger the embedding context’s click event listener.

### Changes to HTMLFencedFrameElement

After a fenced frame element object is created, the embedder can call `addEventListener(‘fencedtreeclick’, callback)` on it to attach an event listener to the frame. The new listener can fire when an event with type `click` is fired in the embedded document’s DOM tree. In addition, the `onfencedtreeclick` property will be added to `HTMLFencedFrameElement` to mirror other event listeners with this syntax. 

To start, the spec will only support `fencedtreeclick`, but given that this API relies on the existing DOM event listener specification, it would be trivial to support other `fencedtree*` events in the future.

The `fencedtreeclick` event listener callback will receive an event object, but it will contain the minimal amount of information necessary to handle the event. Specifically, it will obey the following rules:

* It will be fired using the base DOM Event constructor, rather than a new event subclass.
* It will be initialized with a type parameter of `'fencedtreeclick'`
* All instances of the event object will be initialized with the same static timestamp value in order to mitigate timing side-channel attacks. The timestamp value of the DOM Event interface is a duration represented by `DOMHighResTimeStamp`, 
  so user agents can choose a suitable value to use, such as the Unix epoch.
* The event's `isTrusted` field will be true, to indicate that the event is dispatched by the user agent.
* All other attributes of the event object will have the default settings of a newly-constructed event object.
* When the event object is dispatched, its target will always be the `HTMLFencedFrameElement` upon which the event listener was registered.

Note that specific click information like mouse coordinates are not included. These rules ensure that the event object doesn’t leak information from or about the embedded content.

### Changes to window.fence

Once the embedder has registered the `fencedtreeclick` handler on the fenced frame element, the event needs to be fired while handling the corresponding `click` event within the frame’s content document. This will occur via a new method on the `window.fence` interface, `window.fence.notifyEvent(triggering_event)`. This is different from the existing `reportEvent()` method:

* `reportEvent()` communicates data about events to a remote URL, and the corresponding beacon also includes data set via `registerAdBeacon` (called by Protected Audience worklets) in the destination URL. See the [Protected Audience API explainer](https://github.com/WICG/turtledove/blob/main/Fenced_Frames_Ads_Reporting.md#reportevent-preregistered-destination-url) for more details.
* `notifyEvent()` communicates that an event occurred to the embedder, and nothing else. No extra information is added to the event. This API call also acts as an opt-in by the fenced frame document’s origin to allow sending the notification to the embedding site. 

The function takes one argument, `triggering_event`, which is a `click` event object that the frame’s content is currently handling. In order to trigger the `fencedtreeclick` event in the embedder, this object’s `isTrusted` field must be true, the event must currently be dispatching, and the event’s type name must be `click`. These requirements guarantee that the `fencedtreeclick` event will only be fired by user-agent-generated click events in response to user actually clicking as opposed to a script-generated event.

Here's an example of how `window.fence.notifyEvent()` should be used:

```javascript 
// In the embedder:

// Make a fenced frame
let fencedframe = ...

fencedframe.addEventListener('fencedtreeclick', () => {
    alert('hello world!');
});

// In the embedded content:

document.body.addEventListener('click', (e) => {} {
    // Fire a "fencedtreeclick" event at the embedder.
    window.fence.notifyEvent(e);
});
```

The `notifyEvent()` method will not be available in iframes (same-origin or cross-origin), and will only be available in the fenced frame root’s document. 

### Click Privacy considerations

Since this is exfiltrating some information (that a click happened) outside the fenced frame, we will need to consider the following privacy considerations:

*   A possible attack using multiple fenced frames: an embedder creates `n` fenced frames, which all disable network and then determine (by predetermined behavior, or through communication over shared storage) which one of them should display nonempty content. Then if a user clicks on the only nonempty fenced frame, this exfiltrates log(n) bits of information through the click notification. Mitigating this will require some rate limits on the number of fenced frames on a page that are allowed to read from shared storage. This is similar to [shared storage’s existing rate limits](https://github.com/WICG/shared-storage#:~:text=per%2Dsite%20(the%20site%20of%20the%20Shared%20Storage%20worklet)%20budget). 
*   Click timing could be a channel to exfiltrate shared storage data, but it’s a relatively weak attack since it requires user gesture and is therefore non-deterministic and less accurate. In addition, as a policy based mitigation, shared storage APIs’ invocation will be gated behind [enrollment](https://developer.chrome.com/en/docs/privacy-sandbox/enroll/). 


## Code Example

Now let's take a look at how shared storage, revocation of untrusted network access, and the click listener API can be combined in a real-world example.

This example demonstrates how a third-party payment provider (examplepay.com) might embed a personalized payment button onto a merchant's site. First, the payment provider stores card information in shared storage when a user visits their site in a first-party context. Later, a merchant site uses the provider's API to embed a personalized button, which is rendered in a fenced frame. The fenced frame disables untrusted network access, reads the card information from shared storage, and sets up a click handler on the button to initiate the payment flow.

```javascript
// When a user navigates to “examplepay.com” and registers their credit card, the last
// four digits of their card are written to Shared Storage. 

// On https://examplepay.com
async function registerCard() {
  // Prepare HTTP request with user-provided card infomation. 
  let request = createCardRegistrationRequest({number: 'XXXX XXXX XXXX 1234',  
    expDate: 'MM/YY', ...});

  // Register the card information with examplepay.com.
  let response = await fetch(request);

  // If the card was registered successfully, write the last 4 digits of the
  // card number to Shared Storage for origin "examplepay.com." The data to
  // write could come from the response body, a 1p cookie in a response header,
  // or somewhere else.
  if (response.status === 200) {
    let body = await response.json()
    await window.sharedStorage.set('last4', body.last4);
    console.assert(body.last4 === '1234');
  }
}

// Once the value has been written to Shared Storage, it can later be read inside a
// fenced frame, but only when that frame is same-origin to examplepay.com and has
// its network access restricted. Here’s what that would look like on a merchant page:

// On merchant page
let example_pay_button = examplePayAPI.createButton();
document.body.appendChild(example_pay_button);

// In examplePayAPI
function createButton() {
  let fenced_frame = document.createElement('fencedframe');
  // Create a fenced frame config using a URL directly instead of a config-generating 
  // API like Protected Audience or sharedStorage.selectURL(). Note that the URL is 
  // same-origin to the site where the card was first registered.
  fenced_frame.config = new FencedFrameConfig('https://examplepay.com/make_button');

  // Registering a "fencedtreeclick" event handler on the fenced frame element allows
  // it to respond to a "click" event that fires inside the frame's content.
  fenced_frame.addEventListener('fencedtreeclick', () => {
    startPaymentFlow();
  });
  return fenced_frame;
}

// In the "https://examplepay.com/make_button" fenced frame document
function personalizeButton () {
  // By waiting for the page to finish loading, we can ensure that there's
  // no additional JS waiting to execute before revoking network.
  window.onload = async () => {
    // First, disable untrusted network access in the fenced frame.
    await window.fence.disableUntrustedNetwork();

    // Read the last four digits of the card from Shared Storage 
    // and render them in a button.
    b = document.createElement('button');
    b.textContent = await window.sharedStorage.get('last4');         

    // Tell the embedder that the button was clicked, so that the payment flow can be
    // initiated. This will fire a "fencedtreeclick" event at the fenced frame element
    // in the embedder, which we previously registered a handler for.
    b.addEventListener('click', (e) => {
      window.fence.notifyEvent(e);
    });

    document.body.appendChild(b);
  }
}
```


## Privacy considerations

This section goes into the privacy considerations of the 2 states a fenced frame can be in:

**State 1 (before untrusted network is disabled):** In this state, the fenced frame has contextual information from the embedder site, but not cross-site data. This is equivalent to an iframe and has no privacy concerns.

**State 2 (after untrusted network is disabled):** In this state, the fenced frame could join the user’s identity on the embedding site with the user’s identity on the fenced frame site, but the join is unable to be exfiltrated out.  

Click privacy considerations are already described in the [earlier section](#click-privacy-considerations).

An additional element of user privacy is the ability to control this feature via user agent settings. UAs should ensure that users are able to control this capability in alignment with controls for similar cross-site storage capabilities.


## Security considerations 

This new variant of fenced frames (constructed with a normal URL instead of an config or opaque URL) has similar [security considerations](https://github.com/WICG/fenced-frame/blob/master/explainer/README.md#security-considerations) to  existing fenced frames but because this variant allows information to flow in from the embedding context to the fenced frame, things like permission delegation are simpler (discussed below).


### Permissions delegation

Fenced frames constructed using the non-opaque URL constructor do not have to worry about bits of information being passed via permission delegation. Having said that, we cannot allow any permission based feature that would create a channel of communication in the reverse direction, i.e. from the fenced frame to the rest of the page, e.g. Fullscreen since a “fullscreenchange” event that originates from a fenced frame is observable by its embedder. Another factor to consider is not allowing features that are 



*   either dependent on network access e.g. XMLHttpRequest or,
*   other parts not inherently available in fenced frames e.g. Payment feature is dependent on the existence of opener/openee relationship

Given the above, we would do an audit to see which features are safe to allow in this variant. In the initial launch though, we will likely go with a minimal list of features that we know are necessary for the personalized payment button to work, e.g. shared storage and Aggregate Reporting APIs.


### Ongoing technical considerations

There are ongoing technical considerations, which we will be updating here once the design is crystallized. Specifically, we would like to revisit the fenced frames [process isolation model](https://github.com/WICG/fenced-frame/blob/master/explainer/process_isolation.md) for this variant.


## Stakeholder Feedback / Opposition

This change impacts payment providers. We have heard from multiple payment providers about the need for personalized payment buttons, including Google Pay and Shopify. Here’s a [comment on the TAG review](https://github.com/w3ctag/design-reviews/issues/735#issuecomment-1206075921) showing support from Shopify:

_“As signal boost. Want to note strong interest and support for this on behalf of Shopify. Same/similar [use case and reasons as outlined here](https://developers.googleblog.com/2021/05/updated-google-pay-button-increases-click-through-rates.html).”_

We have also [heard from TAG reviewers](https://github.com/w3ctag/design-reviews/issues/838#issuecomment-1662399487) about fenced frames’ capability to support non-ads use cases, as given below, and this solution will enable such use cases:

_“We had a long discussion about how the shape of this, changing the relationship between an iFrame and its embedding page — it must not be unique to the advertising use cases you've listed._

_We brainstormed along the lines of a site presenting user-generated content in an iFrame, and the payments processes. Have you explored use cases outside the ones you're citing? And if so, what overlaps are you finding?”_

## Alternatives considered

**Alternatives considered: Cookies**

As 3p cookies are planned to be [phased away](https://privacysandbox.com/open-web/) in Chrome (and in other browsers), there are new APIs that allow 3p cookie access under certain circumstances, e.g. the [requestStorageAccess API](https://groups.google.com/a/chromium.org/g/blink-dev/c/JHf7CWXDZUc).  

If cookies were used for this use case, the proposal will be similar to a variant of requestStorageAccess API being invoked and successfully resolved for an FF only if the FF has its network access disabled. 

**Benefits**

The benefits of using cookies for this solution are the following:

*   Cookies are well known by developers and well supported by browsers.

**Downsides**

*   If this solution applies to any 1p cookie then it will have the side effect of the user’s personalized data being sent on the network for every request. To avoid that, there will need to be a new attribute introduced, say “fenced”, so that they are not sent on the network and are only accessible inside a fenced frame. As opposed to the shared storage approach being primarily an API enhancement, this will also require enhancement to the cookies, by adding the “fenced” attribute. The getter code and spec will then need to handle the attribute separately such that it isn’t accessible inside an iframe. Also, the setter should not be accessible inside the FF.
*   The cookies being a default network concept does not align well with this change where the “fenced” cookies have to be JS-only.
*   Fenced frames by default have a unique and ephemeral partition for cookies and with this change, a given cookie’s value before and after the API resolving will be different, which might lead to confusion. Alternatively, cookie access in the FF before the transition can be blocked.

**Alternatives considered: Local Storage**

[Local Storage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage), like shared storage, is origin scoped and JS-only. Local storage is not an ideal choice for this use case, due to the following reasons:

*   Local storage, like other storage APIs, is being [partitioned](https://privacycg.github.io/storage-partitioning/) in iframes. To allow unpartitioned access, it will rely on a variant of [rSA for storage](https://groups.google.com/a/chromium.org/g/blink-dev/c/SEL7N-xIE5s). 
*   Fenced frames, by default have a unique and ephemeral partition and either we would disable that access in read-only FFs or there would be a transition where the value of a given data could be different before and after the access is granted, leading to confusion.


## References
[TPAC presentation from Sep 2023](https://docs.google.com/presentation/d/1TqtFtK4x3TMd96JEvkbApUaYVdIaUz9uz3wNGPTuqdU/edit?usp=sharing)

[Related Issue](https://github.com/WICG/fenced-frame/issues/15)
