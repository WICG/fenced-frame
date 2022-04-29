### **TAG Security/Privacy Questionnaire**

This section contains answers to the [W3C TAG Security and Privacy](https://w3ctag.github.io/security-questionnaire/) [Questionnaire](https://w3ctag.github.io/security-questionnaire/).



1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

    Fenced frames can be viewed as a more private and restricted iframe. As such, the element does not inherently expose any new information to web sites or third parties. However, Fenced Frames are intended to contain data that belongs to a partition other than the top-frame’s storage partition e.g. the rendering url in [TURTLEDOVE](https://github.com/w3ctag/design-reviews/issues/723) reflects the interest group of the user which may be derived from user activity in another partition. Therefore it is necessary for the Fenced Frame to attempt to block communications between the fenced frame and its embedder. Fenced frames remove explicit web-platform communication channels between the two (such as postMessage) and many side channels e.g. size, programmatic focus, policy delegation etc. but some side channels still exist and we will continue to work to mitigate them.

2. Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

    Yes, see above answer for ways information exposure is minimized.

3. How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

    Fenced frames do not inherently provide personal information, PII or information derived from them. 

4. How do the features in your specification deal with sensitive information?

    Same answer as # 3.

5. Do the features in your specification introduce a new state for an origin that persists across browsing sessions?

    No.

6. Do the features in your specification expose information about the underlying platform to origins?

    No

7. Does this specification allow an origin to send data to the underlying platform?

    No

8. Do features in this specification allow an origin access to sensors on a user’s device

    No

9. What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

    Same answer as #1.

10. Do features in this specification enable new script execution/loading mechanisms?

    No

11. Do features in this specification allow an origin to access other devices?

    No

12. Do features in this specification allow an origin some measure of control over a user agent’s native UI?

    No

13. What temporary identifiers do the features in this specification create or expose to the web?

    None.

14. How does this specification distinguish between behavior in first-party and third-party contexts?

    Fenced frames are always present as embedded frames. In terms of communication restrictions from the embedding page, they behave almost like a separate tab but fenced frames do not get access to first-party storage/cookies. A fenced frame document gets access to a nonce-based ephemeral cookie and storage partition.  

15. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

    No difference with a regular mode browser

16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

    Yes.

17. Do features in your specification enable origins to downgrade default security protections?

    No

18. What should this questionnaire have asked?

    N/A
