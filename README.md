# Storage Access Headers

## Authors

* Chris Fredrickson (cfredric@chromium.org)
* Johann Hofmann (johannhof@chromium.org)

## Goals

* Expose metadata about the availability of unpartitioned cookies in a given network request.
* Enable authenticated embedded functionality with lower latency/overhead when possible, by supporting HTTP request response headers related to the Storage Access API.
  * Provide a way to use existing permission grants during document load: https://github.com/privacycg/storage-access/issues/170.
  * Provide a way for the User Agent to indicate whether a network request comes from a context that has opted-in/activated storage access already: https://github.com/privacycg/storage-access/issues/130.
  * Provide a way for the User Agent to indicate whether a network request comes from a context that has `storage-access` permission already (but has not opted in yet).
  * Provide a way for the server to indicate that the User Agent should retry the request after opting into storage access, if possible.
* Ensure that security and privacy are not regressed as a result of this proposal.

## Non-goals

* Providing a non-JavaScript method of _requesting_ the `storage-access` permission is not a goal.
* Exposing metadata about the partitionedness of non-cookie storage is not a goal.
* Exposing metadata about whether [partitioned cookies](https://github.com/privacycg/CHIPS) are available (and if so, what the partition key is) is not a goal.
    * For this use case, see https://github.com/privacycg/storage-partitioning/issues/32 and in particular https://github.com/w3c/webappsec-fetch-metadata/pull/89.

## Introduction

The [Storage Access API](https://github.com/privacycg/storage-access) supports "authenticated embeds" by providing a way to opt in to accessing unpartitioned cookies in an embedded context. The API currently requires an explicit call to a JavaScript API to 1) potentially prompt the user for permission, and 2) explicitly indicate the embedded resource's interest in using unpartitioned cookies (as a protection against CSRF attacks by an embedder).

This requirement is unacceptable for some authenticated embed use cases, and imposes a cost on even the well-suited use cases after they have obtained permission:
* Use of the Storage Access API may currently require multiple network round trips and multiple resource reloads before the embed can work as expected.
* Embedded resources currently must execute JavaScript in order to benefit from this API. This effectively means that the embedded resource must be an iframe, or must be a subresource of an embedded iframe.

These costs and constraints can be avoided by supporting a few new headers.

## Example

### Embedded `<iframe>`

#### Status quo

As an illustrative example, consider a calendar widget on calendar.com, embedded in example.com. During the user's first-ever visit to the example.com page, the flow of events is the following:

1. The user agent requests the calendar widget's content.
   * The fetch of this content is uncredentialed, as the user agent is blocking third-party cookies by default (by assumption). As a result, the server must respond with a placeholder.
1. The user agent loads the placeholder widget, without giving access to unpartitioned cookies.
1. The widget placeholder calls `document.requestStorageAccess()`.
   * Note: this proposal does not include any changes to the existing requirements for obtaining permission.
1. The widget refreshes itself, after `storage-access` permission has been granted.
   * The fetch associated with this refresh is credentialed, per the [Storage Access API spec](https://privacycg.github.io/storage-access/#navigation), so the server responds with the "real" widget.
1. The user agent loads the widget, this time with access to unpartitioned cookies.
   * After this step, the widget can finally work as expected.

This is working as intended, since the user agent may choose to delegate the decision to grant `storage-access` permission to the user, and the user ought to have the benefit of context for that decision.

However, consider a subsequent visit to the example.com page, after the `storage-access` permission has already been granted by the user or user agent. Without this proposal, the flow on the subsequent visit looks exactly the same as the flow on the first visit. However, the user does not need to grant permission this time, since they have already granted permission. This means that the latency and network traffic incurred by the first iframe load, the `document.requestStorageAccess()` script execution, and the subsequent reload are entirely unnecessary.

#### New flow

Instead, we can imagine a different flow, where the user agent recognizes that the calendar widget already has `storage-access` permission and somehow knows that the widget wants to opt in to using it, so it loads the iframe with access to unpartitioned cookies. This would avoid unnecessary latency and power drain due to network traffic and script execution, leading to a better user experience. So, the flow could be:

```mermaid
sequenceDiagram
  Client->>Server: Sec-Fetch-Storage-Access: inactive
  Server-->>Client: Activate-Storage-Access: retry<br/><fallback content>

  note left of Client: Client activates the<br/>storage-access permission

  Client->>Server: Sec-Fetch-Storage-Access: active<br/>Cookie: userid=123
  Server-->>Client: Activate-Storage-Access: load<br/><content>

  note left of Client: Client loads widget<br/>with SAA permission active
```

1. The user agent requests the calendar widget's content.
    * This fetch is still uncredentialed, as before.
    * Since the request is for calendar.com in the context of example.com, and the user has already granted the `storage-access` permission in <calendar.com, example.com> contexts, the fetch includes a `Sec-Fetch-Storage-Access: inactive` header, to indicate that unpartitioned cookie access is available but not in use.
1. The server responds with a `Activate-Storage-Access: retry; allowed-origin=<origin>` header, to indicate that the resource fetch requires the use of unpartitioned cookies via the `storage-access` permission.
1. The user agent retries the request, this time including unpartitioned cookies (activating the `storage-access` permission for this fetch).
1. The server responds with the iframe content. The response includes a `Activate-Storage-Access: load` header, to indicate that the user agent should load the content with the `storage-access` permission activated (i.e. load with unpartitioned cookie access, as if `document.requestStorageAccess()` had been called).
1. The user agent loads the iframe content with unpartitioned cookie access via the `storage-access` permission.
    * After this step, the widget can work as expected.

This flow avoids loading the widget twice, and avoids executing script solely for the `document.requestStorageAccess()` call to activate the existing permission grant. It also avoids the network transmission of the "placeholder" version of the widget.

Additionally, the use of HTTP headers removes the requirement for JavaScript execution. This enables non-iframe resources to take full advantage of existing `storage-access` permission grants.

### Embedded non-`<iframe>`

Consider a document that includes an image (e.g.) which happens to be served by a different (unrelated) site.

At present, no web platform API allows loading this image via a credentialed fetch in browsers that block third-party cookies by default. So, if the image requires the user's credentials (i.e. unpartitioned cookies), then this is broken.

However, if the browser supports the headers described below (and if the user has already granted the `storage-access` permission to the appropriate `<site, site>` pair somehow - e.g. via an iframe at some point in the recent past), then this scenario is supported by the browser as in the following sequence:

```mermaid
sequenceDiagram
  note left of Client: Client is loading document...

  note left of Client: Client begins fetching cross-site image
  Client->>Server: Sec-Fetch-Storage-Access: inactive
  Server-->>Client: HTTP/1.1 401 Unauthorized<br/>Activate-Storage-Access: retry

  Client->>Server: Sec-Fetch-Storage-Access: active<br/>Cookie: userid=123
  Server-->>Client: HTTP/1.1 200 OK<br/><image content>

  note left of Client: Client loads image and continues loading document
```

Browsers that do not support the proposed headers will still receive the appropriate `401 Unauthorized` response. However, browsers that do support the proposed headers are able to retry the fetch and can send the user's credentials, since the user has already given permission for this (by assumption).

### Retry with `reuse-for`

Consider a Single-Page App (SPA) that authenticates with a third party then sends multiple requests to their APIs. Naively, Storage Access Headers (SAH) would require each request to be retried, effectively doubling the number of requests on page load. In the first `retry` response, however, the server can request the browser to make the storage activation reusable with the `reuse-for` header parameter. Subsequent requests to these URLs from the current document will have access to unpartitioned cookies for the lifetime of the document.

```mermaid
sequenceDiagram
  note left of Client: Client is loading document...

  note left of Client: Client begins fetching cross-site content
  Client->>Server: Sec-Fetch-Storage-Access: inactive
  Server-->>Client: HTTP/1.1 401 Unauthorized<br/>Activate-Storage-Access: retry#59; reuse-for=("/foo")

  Client->>Server: Sec-Fetch-Storage-Access: active<br/>Cookie: userid=123
  Server-->>Client: HTTP/1.1 200 OK<br/><content>

  note left of Client: Client begins fetching the /foo cross-site content
  Client->>Server: Sec-Fetch-Storage-Access: active<br/>Cookie: userid=123
  Server-->>Client: HTTP/1.1 200 OK<br/><content>

  note left of Client: Client loads resource and continues loading document
```

## Proposed headers

### Request headers

```
Sec-Fetch-Storage-Access: <access-status>
```
This is a [fetch metadata request header](https://developer.mozilla.org/en-US/docs/Glossary/Fetch_metadata_request_header) (with a [forbidden header name](https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_header_name)), where the `<access-status>` directive is one of the following:
* `none`: the fetch's context does not have access to unpartitioned cookies, and does not have the `storage-access` permission.
* `inactive`: the fetch's context has the `storage-access` permission, but has not opted into using it; and does not have unpartitioned cookie access through some other means.
* `active`: the fetch's context has unpartitioned cookie access.

The user agent will omit this header on same-site requests, since those requests cannot involve cross-site cookies. The user agent must include this header on cross-site requests.

If the user agent sends `Sec-Fetch-Storage-Access: inactive` on a given network request, it must also include the `Origin` header on that request.

### Response headers

```
Activate-Storage-Access: retry; allowed-origin="https://embedder.example"; reuse-for=("/baz.html" "https://embeddee-origin.example/foo/bar")
Activate-Storage-Access: retry; allowed-origin="https://embedder.example"
Activate-Storage-Access: retry; allowed-origin=*
Activate-Storage-Access: load
```
This is a [structured header](https://datatracker.ietf.org/doc/html/rfc8941) whose value is a [sf-item](https://datatracker.ietf.org/doc/html/rfc8941#section-3.3-3) (specifically a [token](https://datatracker.ietf.org/doc/html/rfc8941#section-3.3.4-3)) which is one of the following:
* `load`: the server requests that the user agent activate the `storage-access` permission before continuing with the load of the resource.
* `retry`: the server requests that the user agent activate the `storage-access` permission, then retry the request.
  * The retried request must include the `Sec-Fetch-Storage-Access: active` header. (The user agent must ignore the token if permission is not already granted or if unpartitioned cookies are already accessible. In other words, the user agent must ignore the token if the previous request did not include the `Sec-Fetch-Storage-Access: inactive` header.)
  * The `retry` token must be accompanied by the `allowed-origin` [parameter](https://datatracker.ietf.org/doc/html/rfc8941#section-3.1.2-4), which specifies the request initiator that should be allowed to retry the request. (A wildcard parameter, i.e. `allowed-origin=*`, is allowed.) If the request initiator does not match the `allowed-origin` value, the user agent should ignore this header.
  * The `retry` token may be accompanied by the `reuse-for` [inner-list](https://datatracker.ietf.org/doc/html/rfc8941#inner-list) [parameter](https://datatracker.ietf.org/doc/html/rfc8941#section-3.1.2-4), which allows the user agent to reuse the activation for the specified same-origin URLs (ignoring query parameters and URL fragment, i.e. only considering origin and path) or paths in subsequent requests, during the lifetime of the embedding document. Please read the [relevant security considerations](#reuse-for).

If the request did not include `Sec-Fetch-Storage-Access: inactive` or `Sec-Fetch-Storage-Access: active`, the user agent should ignore this header (both tokens).

If the response includes this header, the user agent may renew the `storage-access` permission associated with the request context, since this is a clear signal that the embedded site is relying on the permission.

Note: it is tempting to try to use [Critical-CH](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Critical-CH) to retry the request, but this usage would be inconsistent with existing usage and patterns for Critical-CH. The `Activate-Storage-Access: retry; allowed-origin=<origin>` header requests that the user agent _change_ some details about the request before retrying; whereas Critical-CH is designed to allow the server to request more metadata about the request, without modifying it. This proposal therefore does not rely on Critical-CH.

Note: The `load` token ignores any accompanying parameters.

## Key scenarios

### Revisiting a previously-allowed authenticated embed

Relative to the Storage Access API's current specification, this proposal allows the user agent to elide some unnecessary network traffic, resource loads, and script execution when a user repeatedly visits a site with an authenticated embed. This results in a few benefits:
* Less network usage
* Less CPU usage (therefore lower power consumption)
* Lower latency until the authenticated embed is usable
* Avoids jarring UX from potentially noticeable intermediate document loads inside of the embed.

### Preserving Storage Access state across (cross-site) navigation flows

Similar to the above, sites may utilize navigations to load different content or as mechanisms to authenticate users. For example, a site might want to preserve storage access status in its embeds while the user visits different top-level pages.

### Enabling an authenticated embed that relies on non-iframe subresources in the top-level page

One ability that this proposal provides is the ability for a non-iframe resource to opt into using an existing `storage-access` permission (via a header instead of JavaScript).

That ability would enable use cases like the IIIF ([cultural heritage interoperability](https://github.com/privacycg/storage-access/issues/72)) to function with a relatively minor update: each "viewer" (top-level site) needs to include an embedded iframe from the "publisher" (embedded site), perhaps on the viewer's homepage, which calls `document.requestStorageAccess()` for the publisher. Once the permission has been granted, any of the viewer's pages can include embedded `<img>` tags from the publisher. The publisher server can then use the `Activate-Storage-Access: retry; allowed-origin=<origin>` mechanism to activate the user's existing `storage-access` permission grant without the use of JavaScript, and ask the user agent to reissue the subresource request with the appropriate cross-site auth credentials.

An important caveat: this proposal does not eliminate the need for a prior top-level interaction on the publisher (embedded) site, nor does it eliminate the need for _some_ call to `document.requestStorageAccess()` from a cross-site embedded iframe (or some other way to request the `storage-access` permission). Another proposal like [Top-Level Storage Access API Extension](https://github.com/bvandersloot-mozilla/top-level-storage-access) could help bridge that gap.

## Browser interoperability

User agents that do not support these headers, or do not wish to allow header-based opt-in, do not have to send the `Sec-Fetch-Storage-Access` header at all; servers should interpret this as equivalent to `Sec-Fetch-Storage-Access: none`, in which case scripts will need to call `document.requestStorageAccess()` before cross-site cookies can become available. The Storage Access API does not rely on support for these headers.

Importantly, this proposal does not introduce a new mechanism to _request_ storage access when an embed has not previously obtained permission, and so website developers must still implement the existing JS-based permission request flow (usually via `document.requestStorageAccess()`) to handle cases where storage access is not granted (or the browser does not reveal whether it is granted).

## Security considerations

### Opt-In signal

The biggest security concerns to keep in mind for this proposal are those laid out in https://github.com/privacycg/storage-access/issues/113. Namely: since the Storage Access API makes cross-site cookies available even after those cookies have been blocked by default, it is crucial that the Storage Access API **not** preserve the security concerns traditionally associated with cross-site cookies, like CSRF.

The principal way that the Storage Access API addresses these security concerns is by requiring an embedded cross-site resource (e.g. an iframe) to explicitly opt in to accessing cross-site cookies by calling a JavaScript API. This proposal continues in that vein by requiring embedded cross-site resources (or their servers) to explicitly opt-in to accessing cross-site cookies (by supplying an HTTP response header).

### Forbidden header name

This proposal uses a new forbidden name for the `Sec-Fetch-Storage-Access` header to prevent programmatic modification of the header value. This is primarily for reasons of coherence, rather than security, but there is a security reason to make this choice. If a script could modify the value of the header, it could lie to a server about the state of the `storage-access` permission in the requesting context and indicate that the state is `active`, even if the requesting context has not opted in to using the permission grant. This could mislead the server into inferring that the request context is more trusted/safe than it actually is (e.g., perhaps the requesting context has intentionally _not_ opted into accessing its cross-site cookies because it cannot conclude it's safe to do so). This could lead the server to make different decisions than it would have if it had received the correct header value (`none` or `inactive`). Thus the value of this header ought to be trustworthy, so it ought to be up to the user agent to set it.

### `reuse-for`

A `retry` header with `reuse-for` enables the embedding document to send subsequent requests to the server with unpartitioned cookies without going through the _`retry`_ flow again. Developers who enable the reuse of storage access activation should be aware of the associated risks, such as cross-site request forgery (CSRF) and [cross-site leaks](https://xsleaks.dev/), and only allowlist URLs which expect credentialed cross-site requests and handle them safely.

* Reusability will stay valid for the lifetime of the embedding document.
* Only the `allowed-origin` specified in the response is able to reuse the activation. I.e., only subsequent requests whose initiator matches the `allowed-origin` can benefit from the reusable activation.
* The URLs in the `reuse-for` parameter are resolved by [parsing](https://url.spec.whatwg.org/#concept-url-parser) them with the request’s URL as the [base URL](https://url.spec.whatwg.org/#concept-base-url). Example accepted values: "/bar", "bar", "/bar/", "./", "../../../etc/passwd", "/bar.html", "bar.js", "https://embeddee-origin.example/foo/bar".
  * Note that "/bar" and "/bar/" are treated as different resources, even though some web servers treat them as the same.
* The resolved URLs must be same-origin with the request’s URL, cross-origin URLs are ignored. If a mixed list is provided, only the same-origin URLs are considered.
* When matching a subsequent request's URL with a previously specified list of URLs from `reuse-for`, the request URL’s query parameters and fragment are ignored.
* User Agents should ignore the `reuse-for` parameter when the `allowed-origin` parameter is `*`.
* User Agents should ignore the `reuse-for` parameter when the `allowed-origin` parameter is `"null"`.

## Privacy considerations

This proposal simplifies some ways in which developers can use an API that allows access to cross-site data. However, it does not meaningfully change the privacy characteristics of the Storage Access API: sites are still able to ask for the ability to access cross-site cookies; user agents are still able to handle those requests how they see fit.

The new header does expose some user-specific state in network requests which was not previously available there, namely the state of the `storage-access` permission. However, this information is not considered privacy-sensitive, for a few reasons:
* The site could have learned this information anyway by calling `navigator.permissions.query({name: 'storage-access')` and/or `document.requestStorageAccess()` in an embedded iframe.
    * Note that this information is now exposed to other kinds of embedded subresources that it wasn't previously available to, however.
* The `Sec-Fetch-Storage-Access` header's value is always none unless the relevant context would be able to access unpartitioned state after calling `document.requestStorageAccess()` without triggering a user prompt. Thus, in the cases where the `Sec-Fetch-Storage-Access` header conveys interesting information, the site in question already has the ability to access unpartitioned state. So, there's no privacy benefit to omitting the `Sec-Fetch-Storage-Access` header altogether when it's not explicitly requested by `Activate-Storage-Access: retry; allowed-origin=<origin>`.
    * Since the header only has one valid non-`active` and non-`inactive` state (namely `none`), there's no privacy benefit to omitting the `Sec-Fetch-Storage-Access` header when its value is `none`.

## Deployment considerations

Servers that begin using the `Activate-Storage-Access` header should include `Sec-Fetch-Storage-Access` in the response's [Vary](https://www.rfc-editor.org/rfc/rfc9110#field.vary) header. This prevents user agents from receiving fallback content for requests that included `Sec-Fetch-Storage-Access: active`.

## Alternative designs
### Preflight requests

It is tempting to design a preflight mechanism, so that non-idempotent (or perhaps non-[simple](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#simple_requests)) cross-site requests can avoid ambiguity (e.g. the server would support the request if it had just included cookies, so the server responds with the `Activate-Storage-Access: retry; allowed-origin=<origin>` header). However, this idea misinterprets the purpose of CORS preflights.

CORS preflights are a security mechanism, to ensure that servers which *don't* support CORS (and likely don't expect cross-origin PUT/DELETE/etc. requests) don't receive those "dangerous" requests. In other words, the preflights play the role of a handshake, after which the server has shown that it knows how to handle non-simple cross-origin requests. (Beyond the rollout of CORS and upgrades of old non-CORS-aware servers, CORS preflights still have a role in ensuring that any cross-origin request with a custom header gets preflighted for security reasons, as well.) This is important because before CORS existed, the Same Origin Policy forbade user agents from sending non-simple cross-origin requests; so servers might reasonably assume that any non-simple request they receive must be same-origin. After CORS became available, non-simple cross-origin requests were allowed by the SOP, which breaks the server's assumption *unless* those non-simple cross-origin requests are preceded by a preflight "handshake", which older servers wouldn't support (and therefore the request would fail in a safe way).

However, the `Sec-Fetch-Storage-Access` and `Activate-Storage-Access` headers do not enable the user agent to send novel, risky requests in the same way that CORS did. The `Sec-Fetch-Storage-Access` header is purely informational; it doesn't change the properties of the request. The `Activate-Storage-Access` header allows re-inclusion of cross-site cookies, which *does* have security implications - but since not all major browsers have made third-party cookies unavailable by default, servers are already written under the assumption that incoming requests may carry cross-site cookies. Therefore, no preceding preflight "handshake" is needed as a security protection.

### CORS integration

It is tempting to design this functionality such that it piggy-backs and/or integrates with CORS directly, since CORS intuitively feels like it is meant to address a similar problem of enabling cross-origin functionality. However, this would be undesirable for a few reasons:

* If CORS (and the relevant SAA permission, of course) were a "sufficient" condition for attaching unpartitioned cookies...
  * Then this would allow the top-level site to attack the embedded site by sending (CORS-enabled) credentialed requests to arbitrary endpoints on the embedded site, without requiring any opt-in from the embedded site before it received those requests. This would make CSRF attacks against the embedded site more feasible. This is undesirable for security reasons.
* If CORS were *required* for the user agent to attach unpartitioned cookies to the request...
  * Then this would mean the embedded site would be required to allow the top-level site to read the bytes of its responses and response headers, just so that the user agent would include cookies when fetching the embedded resource. This is a more powerful capability than simply attaching unpartitioned cookies, so this would expose the embedded site to unnecessary attack vectors from the top-level site. This is undesirable for security reasons.
  * This would also mean that in order to fix an embedded widget on some page, the top-level site must perform some action to enable CORS; the embedded site alone would be unable to update the page and fix the widget. This is undesirable from a developer usability / composability standpoint.

Therefore, CORS ought to be neither necessary nor sufficient for attaching unpartitioned cookies to a cross-site request. We will therefore design the unpartitioned-cookies-opt-in mechanism as a new thing, completely indepedent from CORS.

### Activation reusability

The following alternatives were considered to allow the reuse of storage access activations across requests:

1. **Sticky for destination origin**
   * One alternative is marking an activation "[sticky](https://github.com/privacycg/storage-access-headers/issues/6#issuecomment-1998826464)" for the entire origin of the request’s destination, using a header parameter like `sticky`:
     ```
     Activate-Storage-Access: retry; allowed-origin="https://embedder.example"; sticky
     ```  
     This example would activate storage access for all subsequent requests from https://embedder.example to the origin replying with the sticky header. This stickiness would be valid for the lifetime of the document.
   * This approach, however, is risky because one endpoint requesting a sticky activation would downgrade [security](#reuse-for) of all the endpoints on that origin.
1. **Sticky for destination URL only**
   * Another alternative is making the activation sticky for the specific URL of the request’s destination, with or without the query parameters.
   * This approach is more secure than making an activation sticky for an entire origin but applications sending multiple requests to different endpoints would require multiple retries as described [here](https://github.com/privacycg/storage-access-headers/issues/6#issuecomment-2471620547).
1. **Reuse for a single URL/path**
   * Similarly to the proposed solution, we could introduce a header parameter that allows the browser to reuse an activation for a single specified path:
     ```
     Activate-Storage-Access: retry; allowed-origin="https://embedder.example"; reuse-for="/baz.html"
     ```
   * This would allow an endpoint to activate storage access for a different endpoint, but making the parameter a list further increases the utility of it.
   * **With Wildcards**
     * Wildcards could be allowed either anywhere in the URL provided in `reuse-for`, or at the end of it only.
Wildcards anywhere in the URL would provide the most flexibility but would be the most complicated and potentially fragile. Wildcards would also introduce some risk of developers creating overly broad allowlists which expose their services to cross-site vulnerabilities.
   * **Without Wildcards**
     * In this case, query parameters and URL fragments would be ignored. The URL provided in `reuse-for` could be used as an exact match, activating storage access for a single resource; or it can be interpreted as a directory, activating storage access for all of its subdirectories and resources.
     * The trailing slash could be used to indicate whether to activate an entire directory (if present) or a specific resource (if absent), but this could be error-prone as a single character can drastically change the scope of the activation.
     * If the provided path is treated as a directory, user agents might disallow stickiness outside of the directory of the request’s destination. E.g. `/~bob/image.png` might not ask for a sticky activation for `/` or `/~alice/`.
     * The proposed solution uses a list of specific resources instead of a string that can be interpreted in different ways, to avoid unnecessary complexity and the potential risk of making an activation sticky for an entire directory.
1. **Sticky for Entity**
   * Alternatively, an enum could be provided for developers to decide whether they want the activation to be sticky for the entire origin, the URL of the request (ignoring the query parameters and URL fragment), or the directory.
   * We chose to go with a list of specific URLs instead, to avoid the potential risk of making an activation sticky for an entire directory or the origin, which could unintentionally activate storage access for endpoints owned by different developers. A list of URLs could also provide more flexibility than an enum.

## Stakeholder feedback/opposition

* Chrome: [Shipping](https://groups.google.com/a/chromium.org/g/blink-dev/c/gERgwZfN_-E/m/XiwCTvwaAgAJ)
* Firefox: [Positive](https://github.com/mozilla/standards-positions/issues/1084), [prototyping](https://groups.google.com/a/mozilla.org/g/dev-platform/c/OPmJiLLZdak/m/bps7Ti0UAgAJ)
* Safari: [Support](https://github.com/WebKit/standards-positions/issues/412)
* Edge: TBD
* Web developers: Positive ([feature request](https://github.com/privacycg/storage-access/issues/170), [feature request](https://github.com/privacycg/storage-access/issues/130), [feature request](https://github.com/privacycg/storage-access/issues/189))

## References & acknowledgements

The existing Storage Access API specification and discussions in its GitHub issues heavily inspired this document.
