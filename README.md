# Fenced Frames

This is the repository for Fenced Frames.

**Explainer:** [explainer/README.md](https://github.com/shivanigithub/fenced-frame/tree/master/explainer) 


Third party iframes can communicate with their embedding page using mechanisms such as postMessage, attributes (e.g., size and name), and permissions. A number of recently proposed APIs (such as [Interest group based advertising](https://github.com/michaelkleber/turtledove), [Conversion Lift Measurements](https://github.com/w3c/web-advertising/blob/master/support_for_advertising_use_cases.md#conversion-lift-measurement)) provide some degree of unpartitioned storage to embedded documents. Once third-party cookies have been removed, such documents should not be allowed to communicate with their embedders, else they will be able to join their cross-site user identifiers with the embedder’s, which would allow for user tracking. This explainer proposes a new form of embedded document, called a fenced frame, that these new APIs can use to isolate themselves from their embedders, preventing cross-site recognition.

