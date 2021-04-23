<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Turtledove Worklets](#turtledove-worklets)
  - [Motivation and privacy threat model](#motivation-and-privacy-threat-model)
  - [Design](#design)
  - [Isolation characteristics](#isolation-characteristics)
    - [JS Execution context and thread/process](#js-execution-context-and-threadprocess)
    - [Worklet per functionality per origin](#worklet-per-functionality-per-origin)
      - [Performance and lifetime considerations](#performance-and-lifetime-considerations)
    - [Isolation from extensions](#isolation-from-extensions)
  - [Types of Worklets](#types-of-worklets)
  - [Privacy and Security considerations](#privacy-and-security-considerations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Turtledove Worklets

(Not to be confused with Worklets Web API as these Javascript contexts are not web-platform visible but are created by the browser.)

## Motivation and privacy threat model

[TURTLEDOVE](https://github.com/WICG/turtledove) requires the browser to run custom javascripts, written by the various actors (like an ad-tech or publisher or advertiser) that participate in the [on-device auction](https://github.com/WICG/turtledove#on-device-auction) of ads. The on-device auction requires signals from the publisher page as well as looking at the interest groups the user is part of and makes the winning ad decision based on both of these. 

Since this environment has both the user’s unpartitioned (available across all sites) information - the interest groups, as well as the publisher page’s information, it can join these two. If this joined information is persisted or sent to a server, it can lead to the advertiser network knowing the user’s browsing history or the user’s identity on the publisher page. It can also lead to the publisher page knowing the interest group that the user belongs to. These are contrary to the [privacy guarantees](https://github.com/WICG/turtledove/blob/main/Original-TURTLEDOVE.md#introduction) that TURTLEDOVE is designed to enforce. 

To mitigate these privacy threats, this JS environment needs to be isolated and cannot access the network, storage or exfiltrate any information to the publisher page.


## Design

The TURTLEDOVE worklet is an isolated Javascript execution environment that acts as a pure function and does not exfiltrate any data it has access to via its inputs or the results of invoking browser APIs. 

Let’s dive deeper into how this environment will be used by looking at the end-to-end workflow.



*   **Creation**: The TURTLEDOVE worklets are created from within the TURTLEDOVE APIs and not by the webpage (unlike other web worklets e.g. animation worklet etc.), therefore this explainer does not introduce any web facing API for its creation.  For example, the TURTLEDOVE API is the one that is invoked by the JS running in the publisher page environment, and that in turn creates the worklet to run various pieces of ad-tech-written code in. For more details on the TURTLEDOVE APIs, see [here](https://github.com/WICG/turtledove/blob/master/FLEDGE.md).
*   **Allowed APIs**: The worklets will be allowed to execute only a minimal set of APIs (list of which needs to be created). The reporting worklet will access the [aggregate reporting API](https://github.com/csharrison/aggregate-reporting-api).
*   **Network access**: Since the worklet has access to the publisher page information as well as user’s cross-site data (e.g. ad interest groups), the worklet can join them, thus enabling user’s browsing history tracking by the ad-tech network. To mitigate the exfiltration of this joined data the worklet is not allowed network access. 
*   **Storage access**: For similar reasons, the worklet should also not be able to persist information e.g. in local storage etc. The worklet should not have access to storage.
*   **Communication with the page**: Since the worklet has access to user’s interest groups, it cannot share that with the embedding page and thus communication to the embedding page is not allowed.
*   **Restricted Outputs**: The winning ad output from the worklet needs to be further worked upon by the TURTLEDOVE API for the following aspects:
    *   Confirm if the output is k-anonymous i.e. the same ad is seen by ‘n’ other browsers. If not, re-run the auction logic.
    *   Map the winning ad url to an opaque url before passing it back to the publisher page. This is detailed [here](https://github.com/shivanigithub/fenced-frame/blob/master/OpaqueSrc.md). 


## Isolation characteristics

This section goes into the isolation characteristics of the worklets. 


### JS Execution context and thread/process

Each worklet will be a separate JS execution context (e.g. in Chromium, a separate [V8 context](https://chromium.googlesource.com/chromium/src/+/master/third_party/blink/renderer/bindings/core/v8/V8BindingDesign.md#Context)). A TURTLEDOVE worklet needs to execute off the main thread to make sure the bidding and auction logic do not slow down the main thread. The design will go into more details about the process isolation of the worklets, but a high level overview is that these worklets will benefit by being placed in separate processes than the publisher page so that other compromised renderer processes are not able to alter the sensitive data e.g. bidding price, interest group data, present in these worklets. This is aligned with the threat model for Chromium’s [site isolation](https://www.chromium.org/Home/chromium-security/site-isolation). The design will also discuss protection of the worklets from spectre attacks.


### Worklet per functionality per origin

TURTLEDOVE requires multiple types of worklets for different functionality as mentioned in the section [“Types of Worklets”](#types-of-worklets). An ideal separation between these different worklets is to run each script provided by the different entities like DSP, SSP etc. for different functionalities like bidding, auction etc., in a separate worklet. This will make it easier to restrict the input information flowing into each of these worklets and to enforce privacy guarantees. Examples of how this isolation will help are:



*   Bidding logic from one buyer/origin should not impact the bidding logic from another buyer.
*   Bidding logic cannot impact the auction logic by setting global variables in the JS context.

#### Performance and lifetime considerations

The TURTLEDOVE worklets from one origin being isolated from those of other origins is a hard privacy and security requirement. Similarly, these worklets are scoped to a document and cannot be reused if the top-level page navigates since worklets have the publisher page info and that cannot be leaked across navigations. 

From a performance standpoint, considering that creating multiple worklets/JS contexts for a document is going to be some amount of performance/memory cost and to keep that cost down, browsers might want to create one worklet per origin for one mode (buyer/seller) and use it across various ads on the page instead of creating them separately for each interest group for each ad. The decision to re-use a worklet across ads will not be deterministic and depends on the browser. On the other hand, if there is just a single context for all ad-slots for one DSP, it may mean that the auctions of the various ad slots are sequentially executed leading to ad loading delays. There needs to be a more detailed design of how this can be optimized.


### Isolation from extensions

Although bidding/auction scripts that are executed inside the worklets are free to be blocked by extensions installed by the user, there is a risk to the integrity of the ads auction if the scripts are allowed to be modified by a potentially malicious extension or if a malicious extension is able to execute in the script’s context. This is because TURTLEDOVE moves sensitive server side bidding and auction logic to the browser.

Note that the isolation from extensions mentioned in this section is scoped to modification by extensions and not blocking of content by extensions which will be allowed to proceed as for regular network requests. 

The following safeguards help to achieve that goal within the worklet:



*   As discussed more in the design section below, a separate V8 context guarantees that an extension cannot directly execute in that context. 
*   Since the worklet does not have access to the DOM, it guarantees that the extension cannot mutate the DOM to affect the worklet. 
*   Since the worklet does not access the network, it will not be affected by APIs like WebRequest. 

TURTLEDOVE/FLEDGE do have other network accessing parts like fetching the scripts to run in the worklets and [trusted server interaction](https://github.com/WICG/turtledove/blob/master/FLEDGE.md#31-fetching-real-time-data-from-a-trusted-server) for real-time signals and blocking of those network requests by extensions will be allowed. However, it needs more detailed design to understand if/whether those will need to be restricted modification by extensions or would they instead be verified server-side to confirm that an unmodified script was executed.

## Types of Worklets

As discussed in [FLEDGE](https://github.com/WICG/turtledove/blob/master/FLEDGE.md), TURTLEDOVE will require worklets for its various functionalities, including:



*   **Bidding**: The worklet responsible for executing the bidding script. For more details about the responsibilities of this worklet, see [here](https://github.com/WICG/turtledove/blob/master/FLEDGE.md#32-on-device-bidding).
*   **Auction**: The worklet responsible for executing the auction script. For more details about the responsibilities of this worklet, see [here](https://github.com/WICG/turtledove/blob/master/FLEDGE.md#23-scoring-bids).
*   **Reporting**: The reporting worklet will be responsible for getting inputs from various parts of the page as given below and creating an event-level report (for FLEDGE as MVP) or an [aggregated report](https://github.com/csharrison/aggregate-reporting-api) (long-term) based on those inputs. These reports can be used for various uses like impression counting, conversion measurement, determining fraud or abuse, diagnostics/error reporting etc. Detailed design for these reports needs to be done.
    *   Publisher page provides inputs like viewability events for the fenced frame, ancestor origins etc.
    *   Ad rendered in the fenced frame provides inputs like click time etc.
    *   Bidding/Auction worklet provides inputs like winning ad price and diagnostics like JS errors, determining if codepaths are still live, latency measurement, etc.


## Privacy and Security considerations

Privacy and Security considerations have been detailed in the sections on “Design” and “Isolation characteristics”. 

In addition to those, since TURTLEDOVE is sensitive in nature and requires protections against MitM-type attacks like modifying the bidding script/values on the network, these can only be fetched via HTTPS. This is in line with the guidance [here](https://www.chromium.org/Home/chromium-security/prefer-secure-origins-for-powerful-new-features).
