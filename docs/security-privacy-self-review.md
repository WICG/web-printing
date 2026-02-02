## Responses to the [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/) for the [Direct Sockets API](https://github.com/WICG/direct-sockets/)


1. **What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?**

    This API exposes the information about printers connected to the host and their capabilities. While this information provides an active fingerprinting surface, it can be mitigated by only exposing a printers to the script once the user has explicitly granted access for that printer (for instance, by selecting it from a chooser).

2. **Do features in your specification expose the minimum amount of information necessary to enable their intended uses?**

    Yes.

3. **How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?**

    There is no information of this type being handled by the feature.

4. **How do the features in your specification deal with sensitive information?**

    There is no sensitive information handled by the feature.

5. **Do the features in your specification introduce new state for an origin that persists across browsing sessions?**

    No. However, a user agent may choose to persist printing permissions across browsing sessions.

6. **Do the features in your specification expose information about the underlying platform to origins?**

    Yes, it exposes a list of printers which a user has granted an origin access to. If no access was granted for the origin, then no additional data is exposed.

7. **Does this specification allow an origin to send data to the underlying platform?**

    Yes -- it allows print jobs with documents to be sent to printers.

8. **Do features in this specification enable access to device sensors?**

    No.

9. **Do features in this specification enable new script execution/loading mechanisms?**

    No.

10. **Do features in this specification allow an origin to access other devices?**

    Yes -- Web Printing API allows an origin to access local printers.

11. **Do features in this specification allow an origin some measure of control over a user agent’s native UI?**

    No.

12. **What temporary identifiers do the features in this specification create or expose to the web?**

    None.

13. **How does this specification distinguish between behavior in first-party and third-party contexts?**

    There's no distinction.

14. **How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?**

    Not applicable -- this API is intended to be used in Isolated Web Apps (which do not support either mode).

15. **Does this specification have both "Security Considerations" and "Privacy Considerations" sections?**

    The specification has a single Privacy & Security Considerations section that consists of potential issues and various mitigations.

16. **Do features in your specification enable origins to downgrade default security protections?**

    No.

17. **How does your feature handle non-"fully active" documents?**

    The existing objects provided by the API will be invalidated; the origin will have to query the list of printers again and replace the old entries.

18. **What should this questionnaire have asked?**

    N/A.
