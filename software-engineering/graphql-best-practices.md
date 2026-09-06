# GraphQL Best Practices

> A grounded checklist for designing and operating GraphQL APIs. Every
> recommendation below is traceable to the GraphQL specification, the
> official graphql.org guides, the Relay connection and object-identification
> specs, the GraphQL-over-HTTP spec, or published guidance from Shopify,
> GitHub, Apollo, and OWASP. Sources are listed in §13.

## 1. Model the domain, not the database

- **Think in graphs, from the client's side.** graphql.org: "prefer building
  a GraphQL schema that describes how clients use the data, rather than
  mirroring the legacy database schema." Apollo says the same: "Design your
  schema based on how data is used, not based on how it's stored."
- **Start with objects and relationships, not fields.** Shopify rule 1:
  "Always start with a high-level view of the objects and their relationships
  before you deal with specific fields."
- **Hide implementation.** Shopify rule 2: "Never expose implementation
  details in your API design." Rule 3: design "around the business domain,
  not the implementation, user-interface, or legacy APIs."
- **Build incrementally.** graphql.org: "build only the part of the schema
  that you need for one scenario at a time." Shopify rule 4: "It's easier to
  add fields than to remove them."
- **Put business logic in one layer.** graphql.org: "Your business logic
  layer should act as the single source of truth for enforcing business
  domain rules." Resolvers should be thin adapters over it, so REST, GraphQL,
  and RPC entry points all behave the same.
- **Provide logic, and the raw data too.** Shopify rule 12: "Complex
  calculations should be done on the server, in one place." Rule 13: "Provide
  the raw data too, even when there's business logic around it."

## 2. Naming and types

- **Conventions** (Apollo): fields `camelCase`, types `PascalCase`, enum
  values `ALL_CAPS` "because they are similar to constants."
- **Name for meaning.** Shopify rule 9: choose field names "based on what
  makes sense, not based on the implementation or what the field is called in
  legacy APIs." graphql.org: build a shared vocabulary with domain experts.
- **Use enums for closed sets** (Shopify rule 11) and **custom scalars for
  values with specific semantics** such as dates, money, or URLs (rule 10).
- **Group related fields into sub-objects** (Shopify rule 6).
- **Object references, not raw ids.** Shopify rule 8: "Always use object
  references instead of ID fields." `author: User` beats `authorId: ID`.
- **Interfaces and unions** can model permission-dependent shapes and
  polymorphism; OWASP recommends them for "permission-based property
  returns."

## 3. Nullability

The spec: "By default, all types in GraphQL are nullable"; a trailing `!`
denotes Non-Null.

- **Nullable is the safe default** in a distributed system. graphql.org: "By
  defaulting every field to nullable, any of these reasons [backend down,
  timeout, auth failure] may result in just that field returned null rather
  than having a complete failure."
- **Non-Null propagates failure.** Per the response spec, an error in a
  Non-Null field makes the null "bubble up to the next nullable field", so a
  single `!` deep in a tree can blank out a large parent object. Reserve `!`
  for fields that truly can never be absent.
- **Inputs:** Shopify rule 18: "Only make input fields required if they're
  actually semantically required for the mutation to proceed."
- **Mutation payloads:** Shopify rule 24: "Most payload fields for a mutation
  should be nullable, unless there is really a value to return in every
  possible error case."

## 4. Global object identification

Follow the Relay *Global Object Identification* spec, which graphql.org
endorses for caching:

- Expose a `Node` interface with a single non-null `id: ID!`, and a root
  `node(id: ID!)` field that refetches any object by id.
- Ids must be globally unique and stable: "if two objects appear in a query,
  both implementing `Node` with identical IDs, then the two objects must be
  equal."
- graphql.org: GraphQL has "no URL-like primitive that provides this globally
  unique identifier," so reserve `id` for it. Construct it from type name plus
  local id if backends don't already use UUIDs; base64-encode to signal
  opacity. Keep legacy ids in separate fields if clients still need them.
- Shopify rule 5: "Major business-object types should always implement
  Node."

## 5. Pagination

- **Every list field is a decision.** Shopify rule 7: "Always check whether
  list fields should be paginated or not."
- **Prefer cursor-based pagination.** graphql.org calls it "the most
  powerful" of the options and notes offset pagination has "performance and
  security downsides" on large or changing datasets.
- **Use the Relay Cursor Connections spec** so clients and tooling share one
  model:
  - `XConnection { edges: [XEdge], pageInfo: PageInfo! }`
  - `XEdge { node: X, cursor: String! }`
  - `PageInfo { hasNextPage: Boolean!, hasPreviousPage: Boolean!, startCursor, endCursor }`
  - Forward args `first`/`after`; backward args `last`/`before`.
  - Cursors are opaque strings the client passes back unchanged; edge order
    must be consistent between forward and backward paging.
- **Cursors are opaque.** graphql.org: "to serve as a reminder that the
  cursors are opaque and their format should not be relied upon, we suggest
  base64 encoding them."
- **Bound page size.** GitHub requires `first`/`last` on every connection and
  restricts them to 1–100. Do the same, and reject or clamp anything larger.
- Edges are the place for relationship-specific data (e.g. "friendship
  time") that belongs to neither node.

## 6. Mutations

- **One mutation per logical action.** Shopify rule 14: "Write separate
  mutations for separate logical actions on a resource." Don't build one
  giant `updateThing` that takes every field.
- **Prefix by object.** Shopify rule 17: "Prefix mutation names with the
  object they are mutating for alphabetical grouping" (`orderCreate`,
  `orderCancel`).
- **Use input types** for complex arguments (Apollo) and structure them "to
  reduce duplication, even if this requires relaxing requiredness
  constraints" (Shopify rule 21). Keep the selector separate from the change
  data on updates (rule 22).
- **Consider bulk variants** when a relationship mutation would naturally
  operate on several elements (Shopify rule 16).
- **Return a payload type, not a bare object.** Apollo: every mutation should
  return the modified data so clients "obtain the latest persisted data
  without needing to send a followup query"; they suggest a shared
  `MutationResponse` interface with `code`, `success`, `message`.
- **Return user-level errors in the payload.** Shopify rule 23: "Mutations
  should provide user/business-level errors via a `userErrors` field on the
  mutation payload." Reserve the top-level `errors` array for system and
  programmer errors (see §7).
- **Exactly-one-of inputs.** The spec's `@oneOf` directive on an input object
  requires that "exactly one field must be set and non-null, all others being
  omitted"; use it instead of nullable-everything inputs with runtime checks.

## 7. Errors

The spec's response model:

- A response has `data` and, only when something failed, a non-empty
  `errors` list. "A response may contain both a partial response as well as a
  list of errors."
- Each error must have `message`; may have `locations` (line/column, 1-based),
  `path` (to the failing field, "even if that field is not present in the
  response"), and `extensions` for implementation-specific data such as codes.
- A field error sets that position to null and propagates up through Non-Null
  parents (see §3).

Practices built on that:

- **Put a machine-readable code in `extensions.code`.** Apollo's built-ins:
  `GRAPHQL_PARSE_FAILED`, `GRAPHQL_VALIDATION_FAILED`, `BAD_USER_INPUT`,
  `UNAUTHENTICATED`, `FORBIDDEN`, `PERSISTED_QUERY_NOT_FOUND`,
  `INTERNAL_SERVER_ERROR`; add your own for domain cases.
- **Distinguish user errors from system errors.** Expected, user-fixable
  outcomes belong in the schema (`userErrors` on mutation payloads, or union
  result types), so they are typed and queryable. Unexpected failures go in
  the top-level `errors` array.
- **Mask internals in production.** graphql.org: "Hiding detailed error
  information in GraphQL responses outside of development environments is
  important." Apollo omits stack traces when `NODE_ENV` is `production` and
  recommends `formatError` to log rather than expose. OWASP: disable debug
  mode and field-suggestion hints in production.

## 8. Schema evolution instead of versioning

- graphql.org: "New capabilities can be added via new types or new fields on
  existing types without creating a breaking change," which is why GraphQL
  APIs are usually unversioned.
- **Deprecate, then remove.** The spec's `@deprecated(reason:)` applies to
  fields, arguments, input fields, enum values, and directive definitions. It
  "must not appear on required (non-null without a default) arguments or
  input object field definitions", so make a field optional before
  deprecating it.
- **Breaking changes**, as classified by Apollo's schema checks reference:
  removing a type, field, argument, enum value, union member, interface
  implementation, or input field; adding a required argument or required
  input field; changing the type of a field, argument, or input field;
  changing an argument from optional to required; changing or removing an
  argument or input-field default value. Non-breaking: adding types, fields,
  enum values, union members, optional arguments, optional input fields, and
  descriptions or deprecations. Run a schema diff in CI and check
  operation-usage metrics before removing deprecated fields.

## 9. Performance

- **Solve N+1 with batching.** graphql.org: an initial request "leads to N
  subsequent requests" under naive resolvers; the fix is "a batching
  technique, where multiple requests for data from a backend are collected
  over a short period and then dispatched in a single request" (DataLoader).
- **DataLoader contract:** the batch function must return an array "the same
  length as the Array of keys" with each index corresponding to the same key.
- **New DataLoader per request.** Its cache is per-request memoization and
  "does not replace Redis, Memcache, or any other shared application-level
  cache." Sharing one instance across users leaks data across permission
  boundaries. Call `.clear(key)` after a mutation touches cached data.
- **Caching:** globally unique ids (§4) let clients keep normalized caches.
  For HTTP caching, send queries with GET or persisted-query hashes so CDNs
  and proxies can key on the URL.
- **Persisted queries** let "the client to send a hash of the query instead"
  of the full document, cutting payload size and enabling GET despite URL
  length limits.
- **JSON with compression.** JSON "compresses exceptionally well with
  algorithms such as GZIP, deflate and brotli."
- **Demand control** (graphql.org): "paginating list fields, limiting
  operation depth and breadth, and query complexity analysis."

## 10. Security and demand control

GraphQL's flexibility shifts DoS and enumeration risk onto the server. Layer
these controls (graphql.org security guide, OWASP GraphQL cheat sheet, Apollo):

| Control | What it does | Source |
|---|---|---|
| Depth limit | Rejects deeply nested or cyclic queries; "list nesting warrants stricter limits due to exponential data growth." | graphql.org, OWASP |
| Breadth / alias limit | Caps top-level fields and aliases per operation to stop alias-based amplification. | graphql.org, Apollo (`max_root_fields`) |
| Batch limit | "There should be an upper limit on the total number of queries allowed in a single batch." Prevent batching on sensitive fields such as login. | graphql.org, OWASP |
| Query cost analysis | Assign weights to types/fields, estimate cost before execution, reject over-budget operations. | graphql.org, OWASP, Apollo |
| Rate limiting by cost | Count points, not requests. GitHub: 5,000 points/hour, cost = unique connection requests ÷ 100, max 500,000 nodes/call. Shopify: objects 1, mutations 10, scalars 0, connections sized by `first`/`last`, 1,000-point max per query, leaky bucket refilled per second. | GitHub, Shopify |
| Timeouts | Per request, per resolver, and at the transport; Apollo applies them at router, subgraph, and resolver levels. | graphql.org, OWASP, Apollo |
| Pagination caps | Mandatory `first`/`last` with a maximum (GitHub: 1–100). | GitHub |
| Introspection off in production | Reduces reconnaissance, but "security through obscurity" alone is insufficient; pair with trusted documents. Apollo's router disables it by default. | graphql.org, OWASP, Apollo |
| Trusted documents / safelisting | "Create an allowlist of operations that can be executed against a schema"; best for first-party clients. Apollo: the router rejects unregistered operations. | graphql.org, Apollo |
| Input validation | Allowlist values; use scalars and enums; define input types; "reject invalid input gracefully without exposing validation logic." | OWASP |
| Error masking | No stack traces, debug output, or field suggestions in production. | OWASP, Apollo |

**Authorization.**

- Do it in the business logic layer, not in resolvers. graphql.org:
  "Defining authorization logic inside the resolver is fine when learning
  GraphQL or prototyping. However, for a production codebase, delegate
  authorization logic to the business logic layer." Otherwise "users could
  see different data depending on which API they use."
- Pass "a fully-hydrated user object instead of an opaque token or API key"
  into that layer, keeping authentication and authorization as separate
  pipeline stages.
- OWASP: "Enforce checks on both edges and nodes in queries." In practice
  that includes the `node(id:)` root field, which accepts any id a client
  has ever seen.

## 11. Transport (GraphQL over HTTP spec)

- **Media types.** Servers must accept `application/json` request bodies and
  should respond with `application/graphql-response+json; charset=utf-8`
  (falling back to `application/json` for legacy clients).
- **Request body fields:** `query` (required), `operationName`, `variables`,
  `extensions`.
- **GET is for queries only.** Mutations over GET must be refused
  (`405 Method Not Allowed`). Parameters are URL-encoded, with `variables`
  and `extensions` as JSON strings.
- **Status codes.** Parse or validation failures that stop execution: 400 or
  422 ("request errors"). Execution that produced `data` with field errors:
  200 (a partial-success status is also being discussed). Unsupported
  `Content-Type`: 415. Unsatisfiable `Accept`: 406.
- **Timeouts** at the HTTP layer are the first line of demand control;
  graphql.org lists setting "appropriate timeout durations for requests" under
  transport security.

## 12. Quick checklist

- [ ] Schema describes client use cases, not tables; business logic lives in one layer.
- [ ] Naming: `camelCase` fields, `PascalCase` types, `ALL_CAPS` enums; enums and custom scalars where semantics demand.
- [ ] Nullable by default; `!` only where absence is impossible; required inputs only when semantically required.
- [ ] `Node` interface + `node(id:)` with globally unique, opaque ids.
- [ ] Every list field either bounded or a Relay-style connection with required, capped `first`/`last`.
- [ ] Mutations: one per action, object-prefixed names, input types, payload with modified object + `userErrors`.
- [ ] `extensions.code` on every top-level error; internals masked in production.
- [ ] Evolve with `@deprecated` and schema diffing; no removals without usage data.
- [ ] Per-request DataLoader for every backend access pattern.
- [ ] Depth, breadth, batch, cost, and rate limits enforced; timeouts set.
- [ ] Introspection off and trusted documents on for first-party production traffic.
- [ ] Authorization in the business layer, applied to nodes, edges, and `node(id:)`.
- [ ] GraphQL-over-HTTP compliant transport: GET only for queries, correct media types and status codes.

## 13. Sources

Specifications

- GraphQL Specification (type system, `@deprecated`, `@oneOf`, response format, error propagation). https://spec.graphql.org/ (source: https://github.com/graphql/graphql-spec)
- GraphQL over HTTP specification. https://graphql.github.io/graphql-over-http/draft/
- Relay, *GraphQL Cursor Connections Specification*. https://relay.dev/graphql/connections.htm
- Relay, *GraphQL Global Object Identification Specification*. https://relay.dev/graphql/objectidentification.htm

Official guides (graphql.org)

- Best practices hub. https://graphql.org/learn/best-practices/
- Thinking in graphs. https://graphql.org/learn/thinking-in-graphs/
- Schema design. https://graphql.org/learn/schema-design/
- Pagination. https://graphql.org/learn/pagination/
- Global object identification. https://graphql.org/learn/global-object-identification/
- Caching. https://graphql.org/learn/caching/
- Authorization. https://graphql.org/learn/authorization/
- Performance. https://graphql.org/learn/performance/
- Security. https://graphql.org/learn/security/

Libraries and vendor guidance

- DataLoader README. https://github.com/graphql/dataloader
- Shopify, *GraphQL Design Tutorial*. https://github.com/Shopify/graphql-design-tutorial/blob/master/TUTORIAL.md
- Shopify, *Shopify API rate limits* (calculated query cost). https://shopify.dev/docs/api/usage/rate-limits
- GitHub, *Rate limits and node limits for the GraphQL API*. https://docs.github.com/en/graphql/overview/rate-limits-and-node-limits-for-the-graphql-api
- Apollo, *Schema basics* (naming conventions, query-driven design, mutation responses). https://www.apollographql.com/docs/apollo-server/schema/schema
- Apollo, *Error handling*. https://www.apollographql.com/docs/apollo-server/data/errors
- Apollo GraphOS, *Security overview*. https://www.apollographql.com/docs/graphos/platform/security/overview
- Apollo GraphOS, *Schema checks reference* (breaking vs. non-breaking change types). https://www.apollographql.com/docs/graphos/platform/schema-management/checks/reference

Security

- OWASP, *GraphQL Cheat Sheet*. https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html
