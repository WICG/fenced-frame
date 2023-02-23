# Embedder context in fenced frame configs

## Introduction

For use cases that leverage the [Private Aggregation API](https://github.com/patcg-individual-drafts/private-aggregation-api) to report on advertisements within fenced frames, it might be important to tie certain information available inside the fenced frame, e.g. viewability or performance, to some contextual information from the embedding publisher page, such as an event-level ID.

In a scenario where the input URLs for the fenced frame are required to be k-anonymous, however, such as when creating a [FencedFrameConfig](https://github.com/WICG/fenced-frame/blob/master/explainer/fenced_frame_config.md) from running a [FLEDGE auction](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#2-sellers-run-on-device-auctions), trying to communicate the event-level ID to the fenced frame by attaching an identifier to any of the input URLs would make it difficult for any input URL(s) with the attached identifier to reach the k-anonymity threshold.  

## Proposed solution

Instead, before navigating the fenced frame to the auction's winning [FencedFrameConfig](https://github.com/WICG/fenced-frame/blob/master/explainer/fenced_frame_config.md), the embedder could write the event-level ID, or other contextual information, as a string, to the [FencedFrameConfig](https://github.com/WICG/fenced-frame/blob/master/explainer/fenced_frame_config.md)'s `context` attribute, as in the example below. 

Subsequently, anything written to the [FencedFrameConfig](https://github.com/WICG/fenced-frame/blob/master/explainer/fenced_frame_config.md)'s `context` prior to navigation to that config, can be read via `sharedStorage.context()` from inside a worklet for the [Shared Storage API](https://github.com/WICG/shared-storage) created by the [fenced frame](https://github.com/wicg/fenced-frame/) or by any of its same-origin children.

## Example

After the SSP's JavaScript invokes the [Turtledove/FLEDGE API](https://github.com/WICG/turtledove/blob/main/FLEDGE.md) and receives the resulting [FencedFrameConfig](https://github.com/WICG/fenced-frame/blob/master/explainer/fenced_frame_config.md) `fencedFrameConfig`, the SSP JS can write an event ID string to `fencedFrameConfig.context`.

In the embedder page:

```js
// See https://github.com/WICG/turtledove/blob/main/FLEDGE.md for how to write an auction config.
const auctionConfig = { ... };

// Run a FLEDGE auction, setting the option to "resolveToConfig" to true. 
auctionConfig.resolveToConfig = true;
const fencedFrameConfig = await navigator.runAdAuction(auctionConfig);

// Write to the config any desired embedder contextual information as a string.
fencedFrameConfig.context = "My Event ID 123";

// Navigate the fenced frame to the config.
document.getElementById('my-fenced-frame').config = fencedFrameConfig;
```

The reporting JavaScript in the fenced frame would spin up a [shared storage worklet](https://github.com/WICG/shared-storage#in-the-worklet-during-an-operation), in order to make a report about some information `frameInfo` that is only available within the fenced frame, and that needs to be tied back to the event ID string for reporting purposes.

In the fenced frame (`my-fenced-frame`):

```js
// Save some information we want to report that's only available inside the fenced frame.
const frameInfo = { ... };

// Create a shared storage worklet.
// See https://github.com/WICG/shared-storage#proposed-api-surface for more details 
// about creating and running shared storage worklets.
await window.sharedStorage.worklet.addModule('report.js');

// Send a report using shared storage and private aggregation.
await window.sharedStorage.run('send-report', {
  data: { info: frameInfo },
});
```

Inside the [shared storage worklet](https://github.com/WICG/shared-storage#in-the-worklet-during-an-operation) itself, the event ID can be retrieved and sent in a aggregatable report via the [Private Aggregation API](https://github.com/patcg-individual-drafts/private-aggregation-api).

In the worklet script (`report.js`):

```js
// See https://github.com/WICG/shared-storage#in-the-worklet-during-addmodule for more 
// details about registering shared storage operations.
class ReportingOperation {
  async run(data) {
    // Helper functions that map the event ID to a predetermined bucket and the frame info 
    // to an appropriately-scaled value. 
    // See also https://github.com/patcg-individual-drafts/private-aggregation-api#examples
    function convertEventIdToBucketId(eventId) { ... }
    function convertFrameInfoToValue(info) { ... }
    
    // The user agent sends the report to the reporting endpoint of the script's
    // origin (that is, the caller of `sharedStorage.run()`) after a delay.
    privateAggregation.sendHistogramReport({
      bucket: convertEventIdToBucketId(sharedStorage.context()) ,
      value: convertFrameInfoToValue(data.info)
    });
  }
}
register('send-report', ReportingOperation);
```
