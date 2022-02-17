# Fenced frames and storage/cookies/network state


## Introduction

This document goes into how fenced frames interact with the various states associated with a document/site/origin e.g. cookies, storage etc. as well as the state associated with the corresponding network requests e.g. cookies, http cache etc. 

## Non goals for this document

Note that this document does not go into how storage and cookie access for modes like "read-only" and "unpartitioned-storage" will be supported.


## Fenced frame is not a nested browsing context

It is important to note for this discussion, that even though fenced frames are embedded in another document, they are treated as top-level browsing contexts as explained [here](https://github.com/shivanigithub/fenced-frame#fenced-frame-api). This implies that we need to consider it’s interaction with both unpartitioned state (for fenced frame tree root) and partitioned state (for iframes nested in a fenced frame tree). The rest of the sections will detail how both of these states will be unique inside a fenced frame tree. A fenced frame is an isolated browsing context, with a carefully limited flow of information in and out.  While in some cases a fenced frame is allowed to make network requests, we need to ensure that this does not open up new communication paths between the fenced frame and any other browsing context.


## Storage

A fenced frame tree (the fenced frame root and any nested iframes) cannot access the same storage as another frame (top-level or nested) even though they may have the same storage key (same origin and top-level site). Once [storage partitioning](https://github.com/wanderview/quota-storage-partitioning/blob/main/explainer.md) is the default, fenced frames and their nested iframes will end up with storage keys of &lt;fenced-frame-root-site, fenced-frame-root-origin> and &lt;fenced-frame-root-site, current-frame-origin>, respectively ([existing storage key spec](https://storage.spec.whatwg.org/#storage-keys)). This is because fenced frame root is considered a top-level browsing context. But these keys are not unique and will allow sharing state with other frames outside this tree. To achieve uniqueness, we would be using the [nonce-based storage key](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/public/common/storage_key/storage_key.h;l=33?q=StorageKey::CreateWithNonce&sq=&ss=chromium) where all frames inside a given fenced frame tree will have the same nonce. Note that the same storage key will also partition origin-based communication channels like broadcastChannel. This will allow all same-origin frames within the tree to communicate with each other while disallowing any state sharing/communication outside the tree. The storage keys would then be &lt;fenced-frame-root-site, fenced-frame-root-origin, nonce-for-this-tree> and &lt;fenced-frame-root-site, current-frame-origin, nonce-for-this-tree>, for the fenced frame root and a nested iframe, respectively.


### Developer perspective

From a developer perspective, the same document/scripts can be reused for both fenced frames or for regular frames. That’s because storage access APIs are still allowed and do not return an exception. The difference that developers would need to understand is that none of the storage would be shared by any frame outside the tree.   


## Network Isolation

The [network isolation project](https://docs.google.com/document/d/1V8sFDCEYTXZmwKa_qWUfTVNAuBcPsu6FC0PhqMD6KKQ/edit?usp=sharing) attempts to prevent all persistent and session network data the browser stores on behalf of and about websites from being used to correlate the activity of a user across top-level browsing contexts for different sites. The state partitioned using network isolation keys include the [http cache](https://github.com/shivanigithub/http-cache-partitioning),  Socket Pools etc. Similar to storage partitioning described above, fenced frames cannot share network state with other frames, even those with the same network isolation key (same top-level and frame sites). To achieve uniqueness of the network partition for each fenced frame tree, we would be using the [nonce-based network isolation key](https://chromium-review.googlesource.com/c/chromium/src/+/3015842) ([NIK spec](https://fetch.spec.whatwg.org/#network-partition-keys)) where all frames inside a given fenced frame tree will have the same nonce. This will allow all same-origin frames within the tree to access the same network partition. That enables them to access the same cached entries, for instance. The network isolation keys would then be &lt;fenced-frame-root-site, fenced-frame-root-site, nonce-for-this-tree> and &lt;fenced-frame-root-site, current-frame-site, nonce-for-this-tree>, for the fenced frame root and a nested iframe, respectively.


### Developer perspective

The partitions of network state are not really visible to the developer except for the http cache functionality. All same-site frames within a single fenced frame tree will access the same cache partition. Developers would need to understand that the cache partition would not be shared with any frame outside this fenced frame tree.     


## Cookies

Since fenced frames are treated as top-level browsing contexts, they would by default have access to the unpartitioned cookies for that site. But that would allow sharing state with another same-site top-level page which is not permissible, given that fenced frames do not, by default, get access to unpartitioned storage. Therefore, any access to unpartitioned cookies will be disallowed for fenced frames. 

In conjunction with the partitioned cookies effort ([CHIPS](https://github.com/WICG/CHIPS)), the cookies inside a fenced frame tree can access partitioned cookies, but they would use a nonce to be unique to a fenced frame tree (similar to storage partitioning). This approach has the advantage of treating cookies and storage consistently.


### Alternative design for cookies

Alternatively, cookies can be blocked but that would require special logic for fenced frames as given below. Using unique and transient cookie partition as given above would align well with existing cookies infrastructure.


### Developer perspective

The same document can continue to work for a fenced frame as for a regular frame, except that the cookie API calls and network requests will set/get cookies which are set in a transient and unique partition.
