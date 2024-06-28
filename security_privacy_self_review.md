# Security and Privacy Self-review

https://www.w3.org/TR/security-privacy-questionnaire/

## 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

The response header semantics do not expose any new data to web sites or other parties; it merely gives them the ability to ask for access to existing cookies in non-iframe contexts (if and only if the user has already granted the `storage-access` permission to allow cross-site access to this data).

The request header semantics expose some information regarding whether the HTTP request's context is cross-site or not, and whether third-party cookies were accessible to include in the request or not. This exposure is necessary to allow servers to make application decisions based on the first- or third-partiness of the context, without having to execute JS in an iframe first. (Note that this information is already available to cross-site iframes running JavaScript.)

## 2.2. Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes. 

## 2.3. How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

N/A. 

## 2.4. How do the features in your specification deal with sensitive information?

This proposal does not introduce new types of sensitive information, nor new ways of handling sensitive information. 

## 2.5. Do the features in your specification introduce new state for an origin that persists across browsing sessions?

No.

## 2.6. Do the features in your specification expose information about the underlying platform to origins?

No.

## 2.7. Does this specification allow an origin to send data to the underlying platform?

No.

## 2.8. Do features in this specification enable access to device sensors?

No.

## 2.9. Do features in this specification enable new script execution/loading mechanisms?

No.

## 2.10. Do features in this specification allow an origin to access other devices?

No.

## 2.11. Do features in this specification allow an origin some measure of control over a user agent’s native UI?

No.

## 2.12. What temporary identifiers do the features in this specification create or expose to the web?

N/A. This proposal only gives access to the existing cookie headers (which are already accessible via JavaScript in cross-site contexts, using the `storage-access` permission and the Storage Access API), and a tri-state enum related to the inclusion/exclusion of third-party cookies on the HTTP request.

## 2.13. How does this specification distinguish between behavior in first-party and third-party contexts?

This proposal only affects behavior in third-party (cross-site) contexts; specifically, contexts in which the browser may block third-party cookies.

## 2.14. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

The features work the same way in a Private Browsing or Incognito mode. If third-party cookies are blocked in that profile, they will still be blocked even with these features enabled. Inversely, if the user has granted the `storage-access` permission in that profile (and therefore third-party cookies are not blocked in certain contexts), then these features will enable non-iframe contexts to benefit from that permission.

## 2.15. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

Yes.

## 2.16. Do features in your specification enable origins to downgrade default security protections?

Yes. The features are designed such that in browsers that block third-party cookies, cross-site HTTP requests are protected from CSRF attacks by default (since third-party cookies are blocked by default), but if the `storage-access` permission is granted, then the destination server may opt out of that protection by supplying the `Activate-Storage-Access: retry` HTTP response header.

## 2.17. How does your feature handle non-"fully active" documents?

N/A. This is an HTTP header.

## 2.18. What should this questionnaire have asked?

* Are there existing security practices/features that are related to this feature? If so, does this feature integrate with them? Why or why not?

  * Yes; [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) is a header-based web security feature that seems to solve a similar problem. It is tempting to design Storage Access Headers functionality as an extension of CORS, or as something that integrates with CORS. However, the usage of CORS (either as a necessity or as a sufficiency) creates new attack surface and security concerns (discussed in detail [in the explainer](https://github.com/cfredric/storage-access-headers?tab=readme-ov-file#cors-integration)), so the Storage Access Headers feature is designed to be independent of CORS.

