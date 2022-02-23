## Use-cases/Key scenarios

Following are the use cases for fenced frames [opaque mode](https://github.com/shivanigithub/fenced-frame/blob/master/explainer/modes.md#opaque-ads). 

### Interest Group ads based on user activity (TURTLEDOVE)

[TURTLEDOVE](https://github.com/michaelkleber/turtledove) allows for showing ads based on an advertiser-identified interest, in a privacy-preserving manner.

The following privacy aspects are required for turtledove:


*   Advertisers can serve ads based on an interest, but cannot combine that interest with other information about the person — in particular, with who they are (user’s identity on the embedding site) or what page they are visiting.
*   Web sites the person visits, and the ad networks those sites use, cannot learn about their visitors' ad interests.

These privacy requirements can be met using the fenced frame for rendering the winning ad. 

#### Design

The fenced frame has access to the winning ad which represents user’s interest group information. This information is cross-site data and should not be exfiltrated to the embedding site.
The high level design for turtledove consists of two restricted environments, worklets and fenced frames:



*   The first one, worklets, is responsible for doing the on-device auction and the output of that is the input to the fenced frame. This is a restricted javascript execution environment that does the on-device auction. This has the following characteristics:
    *   It is invoked by JS running in the regular publisher page environment, and use of the turtledove API creates this environment to run various pieces of ad-tech-written code in.
    *   It requires signals from the context that act as inputs to the on-device auction e.g. the publisher page’s topic.
    *   Since it is getting information from the embedding page, we need to make sure it is not exfiltrating that information to a backend server, and thus it is not allowed network access.
    *   Since the result of the turtledove API needs to be restricted from the embedding page, communication to the embedding page is not allowed.
    *   The output of this environment is opaque and not available to query via javascript. This makes the interest group of the user invisible to the embedding page. It is then passed to construct the fenced frame which is used to render the ad represented by the actual url mapped to this opaque url.
    *   For more details, refer the [FLEDGE explainer](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#design-elements). 
*   The second environment is the fenced frame that renders the ad given the bundle from the above algorithm. 
*   Note that if the contextual ad wins the auction, it need not be rendered in the fenced frame. This will leak one bit conveying whether an interest group based ad won the auction or not but the upside is that the contextual ads do not need to change to be part of a fenced frame. 


### Conversion Lift Measurement

[Conversion Lift measurement](https://github.com/w3c/web-advertising/blob/master/support_for_advertising_use_cases.md#conversion-lift-measurement) studies are A/B experiments that ad providers perform to learn how many conversions were caused by their ad campaign vs how many happen organically. To be able to infer the causality of a conversion with the ad campaign, it requires deciding which experimental group the user should consistently be placed for a study (across sites) and show the ad creative corresponding to that group. (Related work: [Private lift measurement API by Facebook](https://github.com/w3c/web-advertising/blob/master/private-lift-measurement-third-party.md))

The following privacy aspects are required for lift measurement:
*   The embedding site should not know which experiment group the user belongs to or which ad got rendered as a result of an A/B experimentation API.

This is a privacy threat because if a publisher knows which experiment group a user is in for, say, n experiments, it gives the publisher an ‘n’ bits user identifier which can also be read on another site, forming a persistent cross-site identifier for the user.

These privacy requirements can be met using the fenced frame for rendering the ad.


#### Design

A high level flow of the design using fenced frames is given below:



*   Outside of the fenced frame, the ad auction returns two ad creative URLs, one for the control arm and one for the experiment arm.
*   The browser API for A/B is then invoked which returns the ad as an opaque output based on user's experiment group.
*   The fenced frame is then created with the ad creative which mapped to the opaque output from the restricted JS environment. The only way information can be extracted from the fenced frame is by using [aggregate measurement APIs](https://github.com/WICG/conversion-measurement-api/blob/master/AGGREGATE.md), via network access, or outbound navigation from the fenced frame.

