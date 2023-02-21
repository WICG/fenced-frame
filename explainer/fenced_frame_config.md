# Fenced frame configs


# Introduction

For use cases involving APIs that access cross-site data, we need to be able to load a fenced frame with content determined by the API without revealing information about the content to the embedding context. For example, with interest-based ads in [FLEDGE](https://github.com/WICG/turtledove), the winning ad that's returned from the auction depends on the user's cross-site interest group data, which we don't want to expose to the site that calls the auction. This document proposes a web-platform way of loading content into a fenced frame using an opaque object.


# Proposed solution

`<fencedframe>` has an attribute `config`, rather than `src`. APIs like FLEDGE return a `FencedFrameConfig` object defined by our [WebIDL](https://wicg.github.io/fenced-frame/#fenced-frame-config-interface). This object has a series of fields that specify the behavior desired by the API (e.g. the ad url, width and height as seen from within the fenced frame, etc.). When the embedder stores this object into the `config` attribute, the fenced frame loads the context accordingly.
  
In order to hide information as described above, the browser _redacts_ `FencedFrameConfig` before sending it to the embedder. This means that certain fields which are sensitive, like the ad url, are replaced with a string `opaque`. The embedder may see whether there is a value defined for that field, but not what the value is. Likewise, when the embedder requests that a config be loaded into the fenced frame, the browser is responsible for looking up the config in a data structure in order to access the unredacted information.

  
## Turtledove Example

When the SSP JS invokes the Turtledove API to run the ad auction, it gets back the `FencedFrameConfig` as the result, which is then used for rendering the fenced frame. This `FencedFrameConfig` has an opaque `src`, which maps to an actual ad url which is part of the interest group.


```
navigator.runAdAuction(myAuctionConfig).then((auctionWinnerConfig) => {
  // auctionWinnerConfig value e.g.
  // FencedFrameConfig {
  //   'src': opaque ('ad.com/foo' internally)
  //   ...
  // }
  const adFrame = document.createElement('fencedframe');
  adFrame.config = auctionWinnerConfig;
});
```

# Backwards compatibility
  
Previously we used a [`urn:uuid`](https://tools.ietf.org/html/rfc4122) and the `src` attribute to accomplish this same behavior. We will continue to support `urn:uuid` and `src` for a transition period.
