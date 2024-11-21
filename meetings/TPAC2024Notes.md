
**Breakout notes for https://github.com/w3c/tpac2024-breakouts/issues/40**

Attendees

Shivani Sharma (Google Chrome)

Andrew Verge (Google Chrome)

Zachary Cancio (Google Chrome)

Josh Karlin (Google Chrome)

Ilya Grigorik (Shopify)

Joel Antoci (Shopify)

Dominic Farolino (Google Chrome)

Sarah Murphy (Microsoft Edge)

Anusha Muley (Google Chrome) 

Mustaq Ahmed (Google Chrome)

Kaustubha Govind (Google Chrome)

Fatma Moalla (Criteo)

Maxime Vono (Criteo)

Michael Kleber (Google Chrome)

Agenda

Slides: https://docs.google.com/presentation/d/1xKwCrhN2pD_lxuPfurgjEDkfyAcf3wgxPS9njlG9tT0/edit?usp=sharing

Notes

Shivani (from the slides linked above):



*   Last TPAC talked about high level design. 
*   We will be demoing functionality
*   Explainer in Q4; intent to prototype on blinkdev
*   everything partitioned by top-level site. but cross-site data is shown the user. where the data is shown and the rest of the page are isolated. iframes have communication channels like postmessage and side channels. for colluding parties to transfer info. 
*   Fenced frames is a new HTML element that allows embedding documents. privacy flow of the use case governs whats possible to communicate between fenced page and embedded page
*   FF no post message; any partition for storage is ephemeral and unique (no same origin frame including other fenced frames) Window.top do not point to the top-level but the FF root frame. THere is a tree that starts at the primary top-level. and. another root which is the FF tree. 
*   FencedframeConfig - provides functionality like protected audience. ad not available to anyone outside the FF.  FF config makes the url opque to the outside page. But we are not going into PA details in this session.
*   In another way of creating the FencedFrameConfig, embedding context can actually provide the source URL (its not hidden between the two contexts). but when the fenced frame gets access to cross-site data; need to make sure the data cannot leave the FF. 
*   New API -> window.fence.disableUntrustedNetwork.  FF can voluntarily give up network access and then can access cross-site data. 
*   Personalized Payment Buttons. UX setting can choose to disable this feature. 
*   Provider Payment; example.pay. go to the top-level and give CC info. At a later time , example.pay wants to show the last 4 of CC on another merchant site.  example.pay can write to shared storage when user provided it.   Shared Storage data is stored on origin scoped basis and read in restricted contexts.  FF without network can read from shared storage - a new output gate for Shared Storage
*   Each payment provider on a page could use the last 4 to give user awareness. 
*    When the button is created its in a FF not in an iframe.  
*   embedding context creates an example.pay FF; FF in state 1 has network can get the html document js/css/.   In state 2 it will call disable network.  Then can call storage access from example.pay.  
*   new click listener/notify API to tell the embedding page.  Embedding page handles click. 
*   New API surfaces. 
*   FF config constructor.  
*   DisableUntrustedNetwork() .  invovled anything that involves network access, any navigations, new tabs, etc.
*   A lot of srfuaces blocked via network isolation key's nonce.  Reuse nonce in key to uniquely identify requests coming from FF tree. if nonces match request that have been blocked. 
*   Nested iframes cannot do network access as well.  
*   Wait for all nested FF to call disableUntrustedNetwork. 
*   Some networks might be OK like private aggregation API 
*   Any request the browser makes are OK
*   sharedStorage.get() already exists.  This FF with no network can call it.  SharedStorage write anywhere, read with restrictions. 
*   Guards: permission policy; user has not disabled feature via UX; Privacy Sandbox's attestation. 

Alex Christenson: 



*   trades network access for getting shared acces. 
*   Locally only already isolated.  Can only tell the outside page that a click happened

    

Sarah:



*   is the storage cached at all. User refreshes page; FF data is cached.  

*   Shared Storage needs to be gotten again. 

    

Joel Anotic: 



*   FF are opaque from each other. 
*   Two same FF reading same origin.

    

Sarah:

*   there is a write mode. 

Shivani:

Even if the FF writes something to shared storage, it has to give up network access again to read that data. 

    

Michael:

sharing of shared storage is domain A"s data while the user is on domain C. but all domain A's storage. 

    

Fatima (Criteo):

*   read shared storage from a protected audience auction  

    

Isaac:

   part of privacy sandbox

    

Michael:

   not possible to bid based on cross-site user profile. Shared storage could let use create user porofile but does not let you use it for bidding. its a different output gate.  Output gate is controlled.  The output from Shared Storage can be fed from aggregation.  Private aggregation APi is a differn output gage

    

Ilya (shopify): 



*   Is there limit on data.  Can call a 
*   Click [Andrew: Will cover on next slide]

    

Josh: 



*   5mb limit. Not part of quota.  
*   On previous question: If i read from FF and stored something locally.  Each FF has an opaque storage key and can't persist local storage. 

Andrew



*   Today its the embedder to intiiate the payment flow. need to tell embedder click occurred.
*   window.fence.notifyEvent
*   Event paramater is dom event object.  Click only event type, could support more if addiitonal use cases. 
*   JS running the FF. Only have event called on legitmate user action. 
*    When function called. new tree. no context of click, no coordinates or timestamp. 
*   Using other payments.  FF needs transient user activation.  User activation will be delegated to the embedder and consumed by the API.  Delegate the permission to the window. 

John (Apple)

What actual usage. Click in the frame. 

Andrew:

*   user click preregistered paymenmt info. 

John:

*   how does the top frame know where to open a popup?

Shivani:

*   the top-frame/embedding frame has some knowledge since it provided the URL in the first place.    

Michael:

Possibly Iframe owned by payment provider, that initiates transaction. Inside of iframe is FF that shows pixels that shows x1234.      

John:

inherent knoweldge from first iframe.  There could be confusion there.    

Michael:

this is a feature, if user turn it off, then use generic info. 

John:

whats if theres an error in loading. the first iframe would be 

Shivani:

   the embedding context is the one to set the URL. 

Michael/Andrew

   The network in stage 1 confirms loading.  If the user clicks but nothing was loaded in the first place, then API can't be called.  

Josh (Google):

load default generic, and opportunistically show something personalized.  Not useful for ads.  

    

Michael:

*   don't have a use case in hand that's helpful for adtech

Josh:

using in an ad would be a new output gate for shared storage.

John W (Apple):

 i'm going to show an ad for this game; player maybe already plays that game. show a "level 5" copy.  

Miachel:

top-level can't distinguish. 

 

Andrew:

open the top-level, then the user can know. 

Mustaq:

only a single button is allowed?  

Isaac (microsoft):

ad-tech surrending network access for video; video player styling. 

Michael: 



*   Can be used for something like using the font for native.  Seems to be difficult use case. 
*   Payment use case 

Josh: 



*   Disable untrusted network, since the FF can't talk. Could allow the postmessage in but not out. 
*   Could we use this for selecting an ad, not covered here.  

Andrew:

  *   visit payment provider in 1p context.  Store CC on backend. On success. This info should be written to shared storage.  Writing in any ocntext. reading thats limited.

  *   Window.sharedStorage.set. 

  *    user visits merchant site. The merchant site will delegate a 3rd party script or API that loads the FF. (example.pay.createButton()). Script creates payment button.  

  *   inside is where the FF occurs.  FFconfig is used to navigate.  Directed to example.pay/buttonURL. Registere that FF click event listener. Would kick off in embedder when clicked.

  *   disablenetwork

  *   accesssharedstorage

  *   handle click event.

  *   usePromise to get info out of shared storage.  Document in FF would attach eventListener. window.notify.

  *   Demo

  *   regsiter card on example.py

  *   go to merchant page.  Button that loaded after, shows card network and last 4. 

  *   when clicked, embedder page can handle click. 

  *   privacy, multiple buttons

  *   You can have multiple buttons, but call to embedder would be indistinguishable.  Only the fence frame root can call notify even to the parent.  Only the same signal will occur regardless of which one you click.  Trying to limit knowledge gained from click.  Click is one-bit leak.  Multiple clicks based based on whats rendered could be used. if 8 frames, only one prompts the user, user clicks. effectively learn 3 bits of data.  Mitigations.. rate limting frames from origin on one particular page.  Cap number of bits learned per click.  How related is too related between fenced frames.  

Michael:

   *   When a click happens, the JS has to propagate. busy waits what ms.  

P2:

Michael: ms 439; at .439 is at which time i call to the dispatch.  How much can the parent know.  [Andrew: The timestamp is not revelaed.]  The parent page is going to receive an event. 

Ilya:

Thank You. Very interested in the personalization use case. payments.  personalized sign-in button.  Variety of other personalization frames that merchant provide: delivery promises. use geoIP approx. FF could have some info about the user like zip; then FF could do promise calculation. Powerful buiding block. Membership signals. Offer discounts to students/vets. At the very end. Use this service to prove you're a member.  

Josh:

like that.  use 100% off. user clicks, then shopify can validate. 

    

Ilya:

 *   if amazon knows you're in prime, can show a different promise.  Shopify know you're a student can show something. 

Joel:

FedCM give the suer understanding they are signed; by clicking will go through lower friction. Does not require full auth.  

Shivani:

follow up in a Github issue. 

Ilya:

Where do other browsers stand?   

John (Apple):

interested in technology. don't have implementation, experimental. Like properties.  Pieces of this interested beyond this spec.  

Sarah:

msft edge. chromium embedder. definitely experiemnt

Isaac:

Question to John: not fenced frame referring to shared stroage ?

John:

 isolating cross-site. have guarantees beyond this.

 small devices that are battery powered want them to last a week. 

 is shared storage required?

Shivani:

yes, shared storage is in this case. 

Michael:

without shared storage, after you give up network access could call something like requestStorageAccess? 

John:

iframe sandbox; site isolation; another way to isolate cross-site data. You could still have storage,  you could still have partition. 

Shivani:

could be a local storage unpartitioned storage bucket. 

Michael:

haven't talked about cookie access

Shivani:

cookies not good bc it goes on the network. 

Michael:

talk about other variants. Would be interested in discussing how Apple would want to solve this 

John Delaney: 

this is only using basic shared storage. 

Shivani:

to implemnet shared storage, just need the simplest version. 

Michael:

don't have all the features jsut a shared db with write and controlled read. 

Josh:

requestStorageAccess could be auto-granted if network disabled FF.  esp for localStorage. 
