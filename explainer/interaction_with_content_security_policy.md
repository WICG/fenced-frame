# **Fenced Frames interaction with Content Security Policy**


## **Why are fenced frames different than iframes when it comes to CSP**

Content Security Policy is a defense against cross-site scripting attacks, allowing developers to harden their own sites against injection of malicious script, style, and other resource types. [CSPEE](https://w3c.github.io/webappsec-cspee/#csp-attribute) further extends it to embedded frames and relies on an explicit opt-in from the embedded content, which ought to make it possible for nested frames to cooperate with their embedders to negotiate a reasonable set of restrictions. 

Fenced frames behave differently than other embedded frames because no embedding site information should flow from the embedder to inside the fenced frame and CSPEE allows arbitrary data flow from the embedder to the fenced frame, as CSP's syntax is very flexible. That data is enough to be a channel for the embedding site to pass its url or user’s identity via those bits.

The embedding site therefore should treat fenced frames differently and assume that the presence of a fenced frame on the site implies no control on the subresources of that frame via the csp attribute.

## Creating a fenced frame within an iframe

Let's say a top-level site has 2 iframes, iframe1 and iframe2. The top-level site has applied the ‘csp’ attribute to iframe1 but not to iframe2. Iframe1 has accepted the policy via the request-response headers exchange. Note that `iframe1` by default cannot create a fenced frame since it is trusted by the top-level page that it will honor policy1 but if it creates a fenced frame that bypasses the policy, it can be used as a workaround to the restrictions posed by policy1. iframe2 however is allowed to create a fenced frame.

**Future work:** An option here is to introduce a permission/csp directive that allows iframe1 to create a fenced frame bypassing its csp rules. If it’s a csp directive its absence should be considered `none` but that’s contradictory to how csp directives work. It could be an `allow` attribute instead. **TODO:** Preference if it should be a csp directive or a permission?


## **Existing CSP directives**

The embedding site can restrict the fenced frame’s document to specific origins, using the frame-src/child-src/default-src directives.

For example, let's say the top-level site has the following policy:

`Content-Security-Policy: frame-src: *.b.test`

An a.test fenced frame cannot be created since it is not allowed via the embedding frame’s csp. A b.test fencedframe can be created.
Note than based on the discussion in the above section, an a.test fenced frame can be created from within iframe2.

Note that this is not applicable to opaque urls which are discussed in the next section.


# **Opaque URL**

The first use cases for fenced frames are going to be for [opaque ads](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/use_cases.md#opaque-ads) and since the url of the fenced frame needs to be hidden from the embedding page, we cannot apply the frame-src directive directly to the mapped url, since CSP reporting can reveal to the embedding page if a fenced frame was successfully created or not. This would lead to the embedding page deriving the possible origins of the opaque ad, going against the isolation guarantees of this mode.

The other option of requiring the embedding site to mention urn:uuid (scheme of the opaque url) in their CSP rules to allow opaque ads, is also not feasible. The limitation is that there are a large number of publisher sites that currently have their csp set to e.g. `frame-src: https:` and requiring the publisher sites to change their headers in order to allow interest-group based advertising to work correctly, is not amenable to deployment. 


## Proposal - Scheme source matching

Note that irrespective of a CSP policy, fenced frames are restricted to be created only in [secure contexts](https://w3c.github.io/webappsec-secure-contexts/) and only from [potentially trustworthy urls](https://w3c.github.io/webappsec-secure-contexts/#potentially-trustworthy-url). Given these restrictions, we could apply the csp scheme-source directives to the scheme of the mapped url without a privacy leak concern. Here are some examples for this approach:



*   If the policy is as given below, then allow an opaque ad that maps to an https url

    ```
    Content-security-policy: frame-src https:  http://example.com
    ```

    Or


    ```
    Content-security-policy: frame-src https://*:*  http://example.com
    ```

    Or


    ```
    Content-security-policy: frame-src *  http://example.com
    ```
 

*   If the policy is as given below, then do not allow any opaque urls to be loaded since the site has given permission to only specific hosts or no hosts, respectively.

    ```
    Content-security-policy: frame-src http://example.com
    ```

    Or

    ```
    Content-security-policy: frame-src` ‘`none`‘
    ```


### Advantages of this approach:



*   Allows backward compatibility with existing sites.
*   Respects the csp rules for host-sources

### Future enhancements



*   Only https or more potentially trustworthy schemes: If opaque fenced frames can map to non-https potentially-trustworthy urls and if we do allow https scheme-source to only match https urls but reject for others, this might be a privacy leak. For example, if the opaque fenced frame is created using [Shared Storage’s runURLSelectionOperation](https://github.com/pythagoraskitty/shared-storage#simple-example-consistent-ab-experiments-across-sites), then the 2 parties could collude to create 32 fenced frames and each of those would either be blocked or loaded to transmit one bit.

*   Support for https:\* to be treated as https: We only do this carve out to the spec and implementation if needed and reported as an issue. We don’t currently have this carve-out because it expects the default port and so acceptance/rejection of an opaque url based on this can leak some information. 


# **New CSP directive**

A new CSP directive called fenced-frame-src to be introduced which if not present will fallback to frame-src/child-src/default-src. This new directive will benefit scenarios like the following:



*   If an embedding site doesn’t want to allow any opaque ads but still wants to allow other iframes, they can use a policy like:

    ```
    Content-security-policy: fenced-frame-src 'none'	; frame-src https
    ```


*   If an embedding site wants to allow any opaque ads but only allow iframes with specific host, they can use a policy like:

    ```
    Content-security-policy: fenced-frame-src https: frame-src http://example.com
    ```


*   If an embedding site wants to allow any non-opaque fenced frames e.g. for payment handling scenarios, but no iframes, they can use a policy like:

    ```
    Content-security-policy: fenced-frame-src http://example.com	; frame-src `none`
    ```

# **Alternatives considered for opaque fenced frames**
*   Apply policy to the mapped URL
The alternative approach could have been to apply the csp header to the actual URL and not report it. That would be hard to roll out from a site owner’s perspective. The standard way to add or change a CSP on a site is:
1. Come up with a CSP the developer thinks is right
2. Turn it on with Content-Security-Policy-Report-Only and a reporting url
3. Verify the site is not getting any unexpected reports
4. Turn it on with Content-Security-Policy

As long as the site is relatively unchanged during this process, if something's going to break in step 4 site owners will learn about it in step 3. If we go with this alternate approach, it would change this property: if the site or a third party library on the site puts fenced-frames on the page, it could break in step 4 without any notice.  This breakage won't appear in CSP reporting, CSP violation events, or any page metrics.  This would make CSP much riskier to deploy.
*   Specifically mention urn:uuid scheme
In this approach, the embedding context says whether urn:uuid based fenced frames are allowed or not : `Content-Security-Policy: fenced-frame-src 'urn:uuid:*' `. Note that the embedding context does not have any say in whether the actual URL it maps to, should be allowed or not. The downside of this approach is that there is no backward compatibility.

