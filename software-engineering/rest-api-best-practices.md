# REST API Best Practices

> A grounded checklist for designing HTTP/REST APIs. Every recommendation
> below is traceable to a primary source: the IETF HTTP RFCs, Fielding's
> dissertation, or a published API guideline from Microsoft, Google, Zalando,
> Stripe, or OWASP. Where sources disagree (notably on versioning), both
> positions are stated rather than picking a winner. Sources are listed in §16.

## 1. Know what REST actually is

Fielding's dissertation (ch. 5) defines REST as a set of architectural
constraints, not a URL naming scheme:

| Constraint | What it requires |
|---|---|
| Client–server | Separation of concerns so the two sides can evolve independently. |
| Stateless | "Each request from client to server must contain all of the information necessary to understand the request." No server-side session context. |
| Cache | Responses must be "implicitly or explicitly labeled as cacheable or non-cacheable." |
| Uniform interface | Identification of resources; manipulation through representations; self-descriptive messages; hypermedia as the engine of application state (HATEOAS). |
| Layered system | A component "cannot 'see' beyond the immediate layer with which they are interacting", which is what allows proxies, gateways, and CDNs. |
| Code-on-demand (optional) | Client functionality may be extended by downloading code. |

Fielding's follow-up post ("REST APIs must be hypertext-driven") is stricter
than most industry practice: a REST API should be enterable with only the
initial URI and knowledge of media types, should not define fixed resource
hierarchies, and should spend its descriptive effort on media types and link
relations rather than on which methods apply to which URIs.

**Practical stance.** Most published guidelines (Microsoft, Zalando) accept
Richardson Maturity Level 2 (resources + HTTP verbs + status codes) as the
minimum and treat Level 3 (hypermedia) as optional. Zalando rule 162 makes
Level 2 mandatory and rule 163 makes Level 3 "may". Microsoft's guide says
Level 3 is "truly a RESTful API, according to Fielding's definition." Decide
which level you are targeting and be honest about it in the docs.

## 2. Resources and URIs

- **Nouns, not verbs.** "Use `/orders` instead of `/create-order`. The HTTP
  GET, POST, PUT, PATCH, and DELETE methods already imply the verbal action."
  (Microsoft)
- **Plural nouns for collections**, hierarchical items:
  `/customers` → `/customers/5` → `/customers/5/orders`. (Microsoft; Zalando
  rule 134 "pluralize resource names")
- **Don't nest relationships too deep.** Microsoft: "extending this model too
  far can become cumbersome to implement"; prefer links in the response body.
- **Casing.** Guidelines differ, so pick one and enforce it:
  - Zalando: kebab-case path segments (rule 129), snake_case query parameters
    (rule 130) and JSON properties (rule 118).
  - Microsoft Azure: camelCase JSON field names and query parameters; "Do not
    upper-case acronyms; use camel case."
- **Custom actions when a verb genuinely doesn't fit CRUD.** Google AIP-136:
  "Custom methods should only be used for functionality that can not be easily
  expressed via standard methods; prefer standard methods if possible." Their
  pattern is `resource:verb` (e.g. `/v1/publishers/x/books/y:archive`), using
  POST unless the method is safe.
- **Avoid chatty APIs.** Microsoft: APIs "that expose a large number of small
  resources are known as chatty web APIs"; consider denormalizing into larger
  resources, balanced against over-fetching.
- **Never put secrets in URLs.** OWASP REST cheat sheet: credentials belong in
  headers or bodies, "never in URLs where they risk exposure in server logs."

## 3. HTTP method semantics (RFC 9110 §9)

| Method | Safe | Idempotent | Use for |
|---|---|---|---|
| GET | yes | yes | Retrieve a representation. Must not have side effects the client is accountable for. |
| HEAD | yes | yes | GET without the body. |
| OPTIONS | yes | yes | Discover communication options. |
| PUT | no | yes | Replace the target resource's state with the enclosed representation. |
| DELETE | no | yes | Remove the resource. |
| PATCH | no | no (not guaranteed) | Partial modification (RFC 5789). |
| POST | no | no | Process the enclosed representation according to the resource's own semantics: create, submit, trigger. |

Rules that follow from the table:

- **Idempotent methods can be retried** by clients and intermediaries after a
  connection failure (RFC 9110 §9.2.2). Design PUT and DELETE so a repeat
  really is harmless. Microsoft: "PUT requests must be idempotent... In
  contrast, POST and PATCH requests aren't guaranteed to be idempotent."
- **Restrict methods per endpoint.** OWASP: "Reject all requests not matching
  the allowlist with HTTP response code 405 Method not allowed." RFC 9110
  requires a 405 response to carry an `Allow` header listing what is permitted.
- **PATCH format.** Use a standard patch document rather than an ad-hoc one:
  JSON Merge Patch (RFC 7396, `application/merge-patch+json`) or JSON Patch
  (RFC 6902, `application/json-patch+json`). Microsoft Azure mandates JSON
  Merge Patch and "Accept null values to delete fields."
- **Prefer PATCH over PUT for updates on evolving resources.** Google AIP-134
  explains why: a PUT client that does not know about a newly added field
  "would wipe out" its value on every update. Google pairs PATCH with an
  explicit field mask; an omitted mask is treated as "all fields that are
  populated."

## 4. Status codes

Use the common codes with their RFC 9110 / RFC 6585 meanings; don't invent
private meanings for standard codes (Zalando rule 150: "use only common HTTP
status codes").

| Code | When |
|---|---|
| 200 OK | Success with a body. |
| 201 Created | Resource created. Include a `Location` header pointing at it (RFC 9110). |
| 202 Accepted | Request accepted for asynchronous processing; see §9. |
| 204 No Content | Success with nothing to return (typical for DELETE, sometimes PUT/PATCH). |
| 206 Partial Content | Range request satisfied. |
| 207 Multi-Status | Batch/bulk requests with per-item outcomes (Zalando rule 152). |
| 304 Not Modified | Conditional GET matched; body omitted. |
| 400 Bad Request | Malformed request. |
| 401 Unauthorized | Missing or invalid credentials. |
| 403 Forbidden | Authenticated but not permitted. |
| 404 Not Found | No such resource. |
| 405 Method Not Allowed | Must include `Allow`. |
| 406 Not Acceptable | No representation matches `Accept`. |
| 409 Conflict | State conflict, e.g. duplicate create, or an idempotent request still in flight. |
| 412 Precondition Failed | `If-Match` / `If-None-Match` evaluated false. |
| 415 Unsupported Media Type | Server refuses the request body format. |
| 422 Unprocessable Content | Syntactically valid but semantically wrong request. |
| 428 Precondition Required | Server requires a conditional request (RFC 6585). |
| 429 Too Many Requests | Rate limit exceeded (RFC 6585); include `Retry-After`. |
| 500 / 503 | Server fault / temporarily unavailable; 503 should carry `Retry-After`. |

## 5. Error bodies

Return a machine-readable error document with a stable content type, not a
free-form string.

- **RFC 9457 Problem Details** (obsoletes RFC 7807) is the IETF standard:
  media type `application/problem+json`, members `type` (URI, defaults to
  `about:blank`), `title`, `status` (advisory copy of the HTTP status),
  `detail`, `instance`, plus extension members that clients must ignore if
  unrecognized. Zalando rule 176 mandates it.
- **Vendor-specific alternatives** exist and are internally consistent:
  Microsoft Azure uses `error.code`, `error.message`, `error.target`,
  `error.details[]`, `error.innererror`, with top-level codes documented as
  part of the API contract. Google uses `google.rpc.Status` with canonical
  codes mapped to HTTP statuses (AIP-193). Either is fine; mixing formats
  within one API is not.
- **Never leak internals.** Zalando rule 177 and OWASP both forbid stack
  traces and internal details in error responses. OWASP: "Provide generic
  messages without revealing technical details."
- **Give a correlation id.** Microsoft Azure: generate and return an
  `x-ms-request-id` header for every request to aid support diagnostics.
  Zalando's equivalent is `X-Flow-ID` (rule 233).

## 6. Representations and content negotiation

- Set and validate `Content-Type` on both sides. RFC 9110 defines 415 for an
  unsupported request body type and 406 for an unsatisfiable `Accept`. OWASP:
  reject unexpected types and "avoid copying the `Accept` header directly to
  response headers."
- Document exact media types. Microsoft's guide has a section on resource
  MIME types; Fielding's view is that media types are where most of the design
  effort belongs.
- Support compression; JSON compresses well with gzip/brotli.

## 7. Caching and conditional requests

REST's cache constraint is not optional. Use the HTTP machinery in RFC 9110
§8.8 / §13 and RFC 9111:

- **ETag** on responses; honor `If-None-Match` on GET (return 304) and
  `If-Match` on PUT/PATCH/DELETE (return 412 on mismatch) to prevent lost
  updates. Microsoft Azure: "DO honor `If-Match`/`If-None-Match` headers and
  return `ETag` responses." Google AIP-134 fails an update with a stale etag.
- Emit `Cache-Control` deliberately. For sensitive API responses OWASP
  recommends `Cache-Control: no-store`.
- Prefer conditional-request-based optimistic concurrency over server-side
  locks; use 428 if you require clients to send preconditions.

## 8. Collections: pagination, filtering, sorting, partial responses

- **Pagination is mandatory for collections** (Zalando rule 159). Google
  AIP-158 warns that pagination cannot be retrofitted to existing
  unpaginated methods without breaking backward compatibility.
- **Prefer cursor (token) pagination over offset.** Zalando rule 160: "Prefer
  cursor-based pagination, avoid offset-based pagination." Google AIP-158:
  tokens "must be opaque (but URL-safe) strings, and must not be
  user-parseable"; base64 alone provides insufficient obfuscation; tokens
  may expire; requests over the maximum page size are coerced down rather
  than rejected.
- **Standard field names.** Google: `page_size`, `page_token`,
  `next_page_token` (omitted on the last page). Microsoft Azure: response is
  an object with a `value` array and a `nextLink`, omitted on the final page.
  Zalando rule 248 defines a page object. Pick one shape for the whole API.
- **Link relations** for navigation can be carried in the `Link` header
  (RFC 8288: `rel="next"`, `rel="prev"`).
- **Filtering and sorting** via query parameters, documented per field.
- **Partial responses for large binaries.** Advertise `Accept-Ranges: bytes`,
  honor `Range`, and answer `206 Partial Content` with `Content-Range`
  (Microsoft; RFC 9110 §14).

## 9. Long-running operations

Microsoft's pattern (both Azure and the Architecture Center guide):

1. Return `202 Accepted` immediately; the work is "accepted for processing but
   is incomplete."
2. Put the URL of a status monitor in a header. The Architecture Center uses
   `Location`; Azure mandates `Operation-Location`.
3. The status resource exposes `id`, `status`
   (`NotStarted | Running | Succeeded | Failed | Canceled`), and on completion
   a `result` or `error`.
4. Include `Retry-After` on the 202 and on in-progress status responses so
   clients know how often to poll.

## 10. Idempotency for unsafe requests

POST is not idempotent, so a client that times out cannot safely retry unless
you give it a mechanism:

- **`Idempotency-Key` header** (IETF draft
  `draft-ietf-httpapi-idempotency-key-header`; Zalando rule 230 "consider
  supporting"). The draft's rules: the key "MUST be unique and MUST NOT be
  reused with another request with a different request payload" (UUIDs
  recommended); the server may fingerprint the payload; reuse of a key with a
  different payload → **422**; a retry arriving while the original is still
  in progress → **409 Conflict**; keys may expire on a published policy.
- Microsoft Azure's equivalent: `Repeatability-Request-ID` and
  `Repeatability-First-Sent` headers with at least a 5-minute window, and a
  blanket rule "DO make all operations idempotent."
- Where you can, let clients create with **PUT to a client-chosen URI**
  instead of POST; PUT is idempotent by definition.

## 11. Rate limiting and resilience

- Return **429 Too Many Requests** (RFC 6585) with **`Retry-After`**
  (RFC 9110 §10.2.3) when throttling. Zalando rule 153 requires 429 with
  headers; Microsoft Azure requires `retry-after` in seconds.
- The IETF is standardizing `RateLimit` and `RateLimit-Policy` response
  fields (`draft-ietf-httpapi-ratelimit-headers`, still a draft as of 2026)
  so clients can self-throttle. Until it is an RFC, treat vendor headers
  (`X-RateLimit-*`) as conventions, not standards.
- Use `Retry-After` on 503 too, so clients back off during outages.

## 12. Versioning and evolution

There is no consensus; the sources take three distinct positions. Pick one
knowingly.

| Position | Who | Detail |
|---|---|---|
| Avoid versioning; evolve compatibly; if you must, version the media type, never the URL | Zalando (rules 113, 114, 115) | Compatible extension is the default; media-type versioning keeps one URI per resource and works with hypermedia. |
| Date-based version pinned per account, selected by header | Stripe | Versions named like `2017-05-24`; an account is pinned to the version current at first call; `Stripe-Version` header overrides; "version change modules" transform responses backward so core code only knows the newest shape. |
| Date-based version as a required query parameter | Microsoft Azure | `api-version=YYYY-MM-DD` is required; 400 if missing or unrecognized. |

Microsoft's Architecture Center lays out the trade-offs neutrally: URI and
query-string versioning are cache-friendly but complicate HATEOAS links;
header and media-type versioning keep one URI per resource but need more
logic in caches and proxies.

**What counts as a breaking change** (Google AIP-180, widely applicable):

- Compatible: adding methods, fields, or enum values, as long as new request
  fields are optional and defaults preserve old behaviour.
- Breaking: removing or renaming anything; changing a field's type "even if
  the new type is wire-compatible"; changing a default value; changing
  resource-name formats; changing observable semantics.

**Deprecation signalling.** Use the standard headers rather than only a
changelog (Zalando rule 189 "Add Deprecation and Sunset header to
responses"):

- `Deprecation: @<unix-timestamp>` (RFC 9745) says the resource is or will be
  deprecated; it "does not change any behavior of the resource."
- `Sunset: <HTTP-date>` (RFC 8594) says when it will stop responding; RFC 9745
  requires the Sunset time to be no earlier than the Deprecation time.
- `Link: <docs>; rel="deprecation"` points to migration guidance.

## 13. Security (OWASP API Security Top 10, 2023)

Design against the concrete failure classes OWASP catalogues:

| ID | Risk | Design response |
|---|---|---|
| API1 | Broken Object Level Authorization | Check ownership/permission on every object id in every request; never rely on unguessable ids. |
| API2 | Broken Authentication | Use standard schemes (OAuth 2.0 / bearer tokens, RFC 6749/6750). For JWTs OWASP says: verify signature, `iss`, `aud`, `exp`, `nbf`; support revocation. |
| API3 | Broken Object Property Level Authorization | Allowlist which properties each role may read/write; don't bind request bodies straight onto models. |
| API4 | Unrestricted Resource Consumption | Rate limits, request size limits, pagination caps, timeouts. |
| API5 | Broken Function Level Authorization | Enforce per-endpoint, per-method access control; 405 for disallowed methods. |
| API6 | Unrestricted Access to Sensitive Business Flows | Add anti-automation controls on flows like checkout or signup. |
| API7 | Server Side Request Forgery | Validate and allowlist any user-supplied URL the server fetches. |
| API8 | Security Misconfiguration | HTTPS only; strict `Content-Type` handling; security headers (`Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Content-Security-Policy: frame-ancestors 'none'`); CORS off unless needed and then origin-specific. |
| API9 | Improper Inventory Management | Keep an OpenAPI description per version; retire old versions with Deprecation/Sunset. |
| API10 | Unsafe Consumption of APIs | Validate third-party API responses like user input. |

Also from the OWASP REST cheat sheet: management endpoints on separate
ports/hosts; audit-log auth failures; sanitize logs against injection.

## 14. Describe the contract

- Publish an **OpenAPI** description. Microsoft: "OpenAPI promotes a
  contract-first approach"; tooling can generate clients and docs from it.
- Treat the description as the contract that breaking-change review is run
  against (Azure requires sign-off by its breaking-change reviewers before any
  customer-visible break).
- Support distributed tracing by propagating trace context headers
  (Microsoft's Architecture Center guide has a section on trace context in APIs).

## 15. Quick checklist

- [ ] Resources are nouns; collections plural; one casing convention.
- [ ] Methods match RFC 9110 semantics; PUT/DELETE are truly idempotent.
- [ ] 405 with `Allow`; 415/406 for media-type problems.
- [ ] Errors use RFC 9457 (or one documented vendor format); no stack traces.
- [ ] Every response carries a request/correlation id.
- [ ] ETags + `If-Match` / `If-None-Match`; `Cache-Control` set on purpose.
- [ ] All collections paginated with opaque cursors and a max page size.
- [ ] Async work returns 202 + status monitor + `Retry-After`.
- [ ] POST supports `Idempotency-Key` (422 on payload mismatch, 409 while in flight).
- [ ] 429 + `Retry-After` on throttling.
- [ ] Explicit versioning policy; breaking changes defined; `Deprecation`/`Sunset` headers.
- [ ] Object-, property-, and function-level authorization on every endpoint.
- [ ] OpenAPI description published and diffed in CI.

## 16. Sources

Primary standards

- Fielding, R. *Architectural Styles and the Design of Network-based Software Architectures*, ch. 5 "Representational State Transfer (REST)". https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
- Fielding, R. "REST APIs must be hypertext-driven" (2008). https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven
- RFC 9110 *HTTP Semantics* (methods §9, status codes §15, conditional requests §13, ETag §8.8.3, Retry-After §10.2.3). https://www.rfc-editor.org/rfc/rfc9110.html
- RFC 9111 *HTTP Caching*. https://www.rfc-editor.org/rfc/rfc9111.html
- RFC 9457 *Problem Details for HTTP APIs* (obsoletes RFC 7807). https://www.rfc-editor.org/rfc/rfc9457.html
- RFC 6585 *Additional HTTP Status Codes* (428, 429). https://www.rfc-editor.org/rfc/rfc6585.html
- RFC 5789 *PATCH Method for HTTP*. https://www.rfc-editor.org/rfc/rfc5789.html
- RFC 7396 *JSON Merge Patch*; RFC 6902 *JSON Patch*. https://www.rfc-editor.org/rfc/rfc7396.html, https://www.rfc-editor.org/rfc/rfc6902.html
- RFC 8288 *Web Linking*. https://www.rfc-editor.org/rfc/rfc8288.html
- RFC 8594 *The Sunset HTTP Header Field*. https://www.rfc-editor.org/rfc/rfc8594.html
- RFC 9745 *The Deprecation HTTP Response Header Field*. https://www.rfc-editor.org/rfc/rfc9745.html
- RFC 6749 *OAuth 2.0*; RFC 6750 *Bearer Token Usage*. https://www.rfc-editor.org/rfc/rfc6749.html, https://www.rfc-editor.org/rfc/rfc6750.html
- IETF draft *The Idempotency-Key HTTP Header Field* (draft-ietf-httpapi-idempotency-key-header). https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- IETF draft *RateLimit header fields for HTTP* (draft-ietf-httpapi-ratelimit-headers). https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/
- OpenAPI Specification. https://spec.openapis.org/oas/latest.html

Published guidelines

- Microsoft Azure Architecture Center, *Web API design best practices*. https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design
- Microsoft, *Azure REST API Guidelines*. https://github.com/microsoft/api-guidelines/blob/vNext/azure/Guidelines.md
- Zalando, *RESTful API Guidelines*. https://opensource.zalando.com/restful-api-guidelines/
- Google, *API Improvement Proposals*: AIP-134 Update, AIP-136 Custom methods, AIP-158 Pagination, AIP-180 Backwards compatibility, AIP-193 Errors. https://google.aip.dev/
- Stripe, "APIs as infrastructure: future-proofing Stripe with versioning". https://stripe.com/blog/api-versioning
- Fowler, M. "Richardson Maturity Model". https://martinfowler.com/articles/richardsonMaturityModel.html

Security

- OWASP, *API Security Top 10 (2023)*. https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- OWASP, *REST Security Cheat Sheet*. https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html
