
# **Permission policy for API backed fenced frames**


# **Introduction**

Current behavior in the origin trial is that all of the permission policy based features in fenced frames are disallowed as described in the explainer [here](https://github.com/WICG/fenced-frame/blob/master/explainer/permission_document_policies.md#permission-policy) (with a caveat given in Note below). This is to make sure that the permission delegation doesn’t act as a communication channel as well as there isn’t any escalation of privilege by virtue of creating a fenced frame.

 

Lately, we have heard feedback for certain features to be enabled, specifically in [opaque-ads mode fenced frames](https://github.com/WICG/fenced-frame/blob/master/explainer/modes.md#opaque-ads) (or API-backed fenced frames). The feedback is for the following 2 features but we would like to have a solution that can extend to other features as well:



1. Attribution reporting: [github issue](https://github.com/WICG/fenced-frame/issues/37).
2. Shared storage APIs: [github issue](https://github.com/WICG/fenced-frame/issues/44).

**Note:** As an incremental step to the proposed solution, the OT recently added  an exception for attribution-reporting and shared-storage APIs.
It allows a fenced frame to be navigated successfully only if the embedding frame has not opted out from these features and enables it for the fenced frame by default (without an allow attribute). 


# **Proposed Solution**

The proposed solution is to extend the urn:uuid bound attributes as described in this [issue](https://github.com/WICG/fenced-frame/issues/48) to also include the permissions that the fenced frame’s consumer API considers important for a particular url to load. 

There are various inputs to compute the final permissions for a document loaded in the fenced frame. Dividing them further into what currently exists for iframes vs the new ones:

Inputs that exist today in iframes:



1. **`allow-attribute`**: “Allow” attribute will have the same semantics as in an iframe and will enable the embedder to set permissions explicitly on a fenced frame similar to how it happens with iframes. 
2. **`document-response-restricted-permissions`**: Optionally, the fenced frame document response headers can further restrict the permissions similar to how it happens in an iframe. 

Inputs that are new in the fenced frames workflow:



1. **`required-permissions-to-load`**: This is the set of permissions that the API declares as the permissions “required” for the fenced frame to be navigated to a given url. Any permissions outside this set will not be available in the fenced frame. This allows these permissions to be verified for k-anonymity and not have more permissions added that were not verified. 
    1. If a permission which is part of this set is not allowed by the embedding frame, then the FF will not navigate successfully. This is required for fenced frames because if it can be detected by the FF’s document that a permission is not allowed, then it can be used as a communication channel from the embedder to the FF.
    2. This set of permissions may be derived by the FLEDGE/SharedStorage APIs in various ways:
        1. A default list maintained by the browser. Having a default list for an API  reduces developer effort. For example, each interest group author need not put “attribution reporting” as part of their FLEDGE interest group.
        2. Additional inputs e.g. defined as an argument to the runAdAuction API. Let’s call that **`predeclared-allow-attribute`**. This allows the API to consider the permissions granted as inputs to the auction. 

The algorithm steps will be as follows:



1. The embedder calls an API to generate a urn:uuid e.g. navigator.runAdAuction(). The API takes `predeclared-allow-attribute`, any default list as mentioned above into consideration and picks `required-permissions-to-load`. Any permissions that are allowed inside the fenced frame should have gone through the k-anonymity check along with the url and resulted in **`required-permissions-to-load` .**
2. The embedder navigates the FF to the urn:uuid. 
    1. If **`allow-attribute`** does not contain a permission that is part of **`required-permissions-to-load`**, then the FF fails to load.
    2. Otherwise navigate the fenced frame and only allow the permissions in **`required-permissions-to-load`** to be delegated to it even if **`allow-attribute`** contained more permissions than that.
3. The fenced frame loads the document and the response headers **`document-response-restricted-permissions`** can further restrict the permissions applied to the document similar to how it does for an iframe.

### Which features/permissions can be granted to a fenced frame

Some permission based features may not align well with the privacy guarantees of an API, e.g. we do not want to allow JoinAdInterestGroup from a FLEDGE FF since users would expect to join an IG from a top-level page they visited and not from an ad. Similarly, we might not allow a hypothetical feature for getting unpartitioned cookies in a FLEDGE fenced frame given that FLEDGE does not allow microtargeting. APIs can therefore additionally have allow-lists based on their privacy guarantees and that would further restrict the **`required-permissions-to-load`** in step 1 of the algorithm above.
