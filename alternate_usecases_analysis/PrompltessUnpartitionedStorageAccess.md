# Fenced frames  and promptless unpartitioned storage access



## Introduction

This document goes into the details of promptless unpartitioned storage access as a potential use case of fenced frames ([privacyCG discussion](https://github.com/privacycg/storage-access/issues/41)). For fenced frames concept and design please see the [explainer](https://github.com/shivanigithub/fenced-frame).


## Unpartitioned storage access

The fenced frame does not have storage access by default. We want the fenced frame to have access to unpartitioned storage, if needed.

The storage access will be gated behind a user gesture and the states of a fenced frame would be:



1. Start
    *   No storage access
2. On user activation and requesting storage access
    *   Read/write unpartitioned state access granted

There are a number of use cases for unpartitioned storage access. These include embedded media playing and enqueueing, document viewing and editing, social widgets, and article comments. requestStorageAccess within the fenced frame can be used to fulfill these use cases.

Although many of these use cases could be handled with a combination of user identification and server-side storage, the common way to identify users today is from their storage (cookies). Also, any offline use cases (such as offline docs) would require client-side storage.


### Challenges

requestStorageAccess is used to provide access to unpartitioned storage. When invoked within fenced frames, the goal is to not show a permission prompt, thanks to the communication isolation of a fenced frame. However, that is dependent on mitigating challenges like link decoration, network timing etc. as discussed in the explainer [here](https://github.com/shivanigithub/fenced-frame#challenges) and the thread [here](https://github.com/privacycg/storage-access/issues/41#issuecomment-673057755).


## Alternative state machine considered

An alternative that was considered was to provide read-only access to the storage if requested before user activation which can later be converted to read-write storage. Before user activation, unpartitioned storage is only available in a read-only mode to make sure that any bits of information that may have been passed along to the fenced frame e.g. using frame size, is not persisted. But once the read-only data is provided, network access is disabled until user activation. The state machine then becomes.



1. Start
    *  Full network access, no storage access
2. On requesting read-only storage access without user activation
    *  No network access, read-only unpartitioned state (if requested)
3. On user activation
    *  Full network access, read/write unpartitioned state (if requested)

Downsides



*   Increased complexity since allowing read-only access in certain storage mechanisms like IndexedDB, can lead to side channels, for example holding a transaction open for a read could be inferred by observing blocked read-write operations in another context.
