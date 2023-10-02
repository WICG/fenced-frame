# Fenced frames process isolation

## Objective

Fenced frames carry information that should not be shared with any other context. In order to ensure that fenced frames cannot communicate with external contexts, we need to isolate them at the process level as well as the API layer. This is due to the various side-channel leaks that can occur between same-process scripts (e.g., spectre). This document describes the process isolation behavior required for fenced frames.


## Notes and assumptions



*   Any special process isolation logic for fenced frames will not be part of the initial launch but will be added eventually.
*   Fenced frames are always in a separate browsing instance (browsing context group) in the eventual MPArch architecture (and not in the interim shadowDOM architecture, since fenced frames will be based off on iframes).
*   Being in the same process as another frame implies it is possible for either frame to look into the data held by the other (see [spectre](https://spectreattack.com/)).
*   Presence of arbitrary code execution in the renderer process of either the fenced frame or the embedding page can lead to information sharing between the 2 contexts and guarding against these attacks helps enforce the privacy guarantees of fenced frames, which is that the embedding context and fenced frames should not be able to look into each others’ data
*   The severity given in each of the sections below is roughly based on the [guidelines](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/security/severity-guidelines.md) here.


## Embedding frame

Even in the presence of an attacker with arbitrary code execution in the renderer process,  fenced frame’s privacy threat model of not having access to the embedding page’s info and vice-versa should still apply. To get this, we will need to make sure that fenced frames are in distinct processes than the embedding frame.

**Severity:** **high** 


### Current and expected process isolation in Chrome

**Desktop:** Currently, different site frames are assigned to different processes and on reaching process limit, same-site frames may be consolidated in a single process. So it looks like cross-site frames will always be in separate processes. That would be extended to make sure that fenced frames follow the process isolation of cross-site frames as well, since fenced frames essentially are treated as cross-site to all other frames.

**Android:** On Android current behavior will not treat fenced frames in any special way but for post-OT launch, the idea is for them to be treated as special frames that are isolated even on Android. Given the low severity of fenced frames being together in the same process (see below), we can create a single process for all fenced frames to keep the number of processes low.

**Implementation design:** [SiteInfo](https://source.chromium.org/chromium/chromium/src/+/main:content/browser/site_info.h;l=43;bpv=1;bpt=1?q=SiteInfo&ss=chromium) will require to be updated to handle fenced frames, in Desktop, to make sure same-origin non-fenced frames are not put together with a fenced frame (see below) and on Android to make sure fenced frames are allocated a separate process than their embedding site/cross-origin frames/same-origin non-fenced frames.

For low-end devices, where no process isolation is feasible, fenced frames will be in the same process as their embedding frame and other frames.

 


## Cross-site frames

As with the embedding page, the same threat exists with any other cross-site frame as well, so fenced frames should be in a distinct process than those. 

**Severity:** **high**


## Same site top-level frames

Same-site top-level frames will have access to cookies and unpartitioned storage access while fenced frames will not have that access. 
In the presence of an attack, this privacy constraint should still be achieved so it would need to be in a separate process than another same-origin top-level document.  E.g. if the same-origin top-level frame carries out a Spectre attack against a fenced frame, it can join that frame's data with its own cookies and thus compromise the user's privacy

**Implementation design:** Since fenced frames are always a new BrowsingInstance, site isolation will try to put them in a separate process unless the process limit is reached. So we will need special logic for fenced frames so that their process is not merged even on reaching the limit.

**Severity:** **high**


## Same site iframes 

Same-origin iframes (on the same page) are able to communicate with the embedding site and thus in the presence of an attack, this could be used as a workaround for the fenced frame to communicate with the embedding site. Therefore they should be in distinct processes.

**Severity:** **high**

 


## Same site fenced frames on the same page

Since these are same-site, both fenced frames have almost the same information except possibly different sizes and different src URLs. Since sizes are restricted in the values that can be allowed and URLs being same-site, this still seems acceptable for them to be in the same process, in the presence of an attack.

**Severity: low**


## Same site fenced frames on a different page

Since these are same-origin, both fenced frames have almost the same information except possibly different sizes and different src URLs. Similar to the above case, this seems less information and thus acceptable for them to be in the same process, in the presence of an attack.

**Severity: low**


## Cross-site fenced frames

Cross-site fenced frames whether on the same page or on a different page will be similar to the same-origin fenced frame cases with the addition of the rendering URL also being an information that they can share with each other in the presence of an attack. 

**Severity: medium**
