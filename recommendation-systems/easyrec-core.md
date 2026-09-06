# easyrec-core: How the easyrec Recommender Core Works

Source analyzed: `easyrec-code/easyrec-core` (Maven artifact `org.easyrec:easyrec-core`, version 1.0.2-SNAPSHOT).
easyrec is an open-source, GPL-licensed recommendation engine written in Java, originally built by
Research Studios Austria. It is a **multi-tenant, item-to-item, association-based recommender**: it
records user actions, stores precomputed item-to-item associations (produced by plugins such as an
association rule miner), and answers recommendation queries by walking those associations.

`easyrec-core` is the foundation module. It holds the domain model, the database access layer (MySQL),
and the basic services (`ActionService`, `ItemAssocService`, `RecommenderService`,
`RecommendationHistoryService`, `TenantService`, `ProfileService`, `ClusterService`). It does **not**
contain the algorithms that compute associations (those live in `easyrec-plugins`) nor the REST/web
layer (`easyrec-web`).

---

## 1. Where the module sits in the project

```
easyrec (parent)
├── easyrec-testutils
├── easyrec-utils            <- Spring/cache/AOP helpers, CSV auto-import framework, SQL script service
├── easyrec-core             <- THIS MODULE: model, DAOs, core services
├── easyrec-plugin-api       <- contract for "generators" that compute item associations
├── easyrec-plugin-container
├── easyrec-plugins          <- ARM (association rule miner), slope-one, profile similarity, ...
└── easyrec-web              <- REST API + admin UI, consumes easyrec-core services
```

Key dependencies of the core: Spring (JDBC, Web MVC), JUNG (graph library, used for cluster trees),
Ehcache (via Spring cache annotations), json-path + Jackson (JSON profiles), commons-validator.

---

## 2. The conceptual data model

Everything in easyrec is expressed with a small set of concepts. All of them are scoped by a
**tenant** (one website / one customer of the recommender).

| Concept | Meaning | Table |
|---|---|---|
| **Tenant** | An isolated customer/site. Has a numeric id, a string id, a rating range (min/max/neutral), and two blobs of `Properties` (`tenantConfig`, `tenantStatistic`). | `tenant` |
| **Item** | Anything that can be recommended (product, track, article). Identified by `(tenantId, itemId, itemTypeId)`. | `item` (web-level), plus `idmapping` |
| **Action** | An event a user performed on an item: VIEW, BUY, RATE, ... Carries user id, session id, IP, optional rating value, timestamp. | `action`, archived to `actionarchive` |
| **ItemAssoc** | A directed, weighted edge between two items: `itemFrom --assocType(value)--> itemTo`. This is the core "knowledge" from which recommendations are served. | `itemassoc` |
| **Recommendation / RecommendedItem** | A log of what was recommended to whom, with a strategy label and an explanation string per item. | `recommendation`, `recommendeditem` |
| **Profile** | Free-form XML or JSON document attached to an item (metadata used by content-based plugins). | `profile` |
| **Cluster** | A named group of items, organized in a tree per tenant. Implemented on top of items and associations, not a separate table. | reuses `itemassoc` + `profile` |

### Type tables: everything is parameterised per tenant

Rather than hard-coding "VIEW" or "BOUGHT_TOGETHER", easyrec keeps six small lookup tables, each keyed
by `(tenantId, name) -> id`:

| Type | Purpose | Defaults for a new tenant |
|---|---|---|
| `itemtype` | Kinds of items. Also stores an optional XML `profileSchema` and `profileMatcher`. Has a `visible` flag. | `ITEM` |
| `actiontype` | Kinds of user actions. `hasvalue` flag marks types that carry a numeric value (RATE). | `VIEW`, `RATE`, `BUY` |
| `assoctype` | Kinds of item-to-item relationships. `visible` flag hides internal ones. | `VIEWED_TOGETHER`, `GOOD_RATED_TOGETHER`, `BOUGHT_TOGETHER`, `IS_RELATED`, `PROFILE_SIMILARITY` |
| `sourcetype` | Who produced an association (which plugin, or manual). | `MANUALLY_CREATED` |
| `viewtype` | Which audience an association is intended for. | `ADMIN`, `COMMUNITY`, `SYSTEM` |
| `aggregatetype` | How to collapse several actions into one rating. | `AVERAGE`, `FIRST`, `MAXIMUM`, `MOST_FREQUENT`, `NEWEST`, `OLDEST` |

Defaults come from the Spring bean in `spring/core/TenantConfig_DEFAULT.xml` and are inserted by
`TenantServiceImpl.insertTenantWithTypes` when a tenant is created.

### ID mapping

External callers use string item ids (e.g. SKU `"ABC-123"`). Internally everything is an `Integer`.
`IDMappingDAO` maintains a global `idmapping (intId, stringId)` table:

- `lookup(String)` returns the int id, **inserting a new mapping if absent**.
- `lookupOnly(String)` returns the int id or throws `ItemNotFoundException`.
- `lookup(Integer)` reverses the mapping.

Lookups are cached with `@LongCacheable` (Ehcache via AOP from easyrec-utils).

---

## 3. The "generic types" design: three layers of the same model

The most distinctive structural idea in the module. The value objects are generic in two type
parameters, `ItemVO<I, T>`, `ActionVO<I, T>`, `ItemAssocVO<I, T>`, `RecommendationVO<I, T>` etc., where
`I` is the type of ids and `T` is the type of "types" (item type, action type, assoc type...).

This lets the same classes be reused at three levels of abstraction:

```
        Web / plugin level             Domain level                     Core level
   ItemVO<Integer, String>   <-->  ItemVO<Integer, String>   <-->  ItemVO<Integer, Integer>
   (string item ids,               (int ids, string types,          (int ids, int type ids,
    string types)                   e.g. "VIEW", "BUY")              e.g. 1, 3)
```

| Layer | Package | Ids | Types | Example |
|---|---|---|---|---|
| Generic interfaces | `org.easyrec.service.Base*Service`, `org.easyrec.store.dao.Base*DAO` | `I` | `T` | `BaseRecommenderService<R, I, ACT, IT, AT, T, U>` |
| Core (Integer/Integer) | `service.core`, `store.dao.core` | `Integer` | `Integer` | `RecommenderService`, `ActionDAO` |
| Domain (Integer/String) | `service.domain`, `store.dao.domain` | `Integer` | `String` | `DomainRecommenderService`, `TypedActionDAO` |
| Music domain | `service.domain.music` | `Integer` | encoded in method names | `MusicRecommenderService.alsoBoughtTracks(...)` |

`TypeMappingService` (in `service.domain`) is the bridge: it converts every VO between the
`<Integer, String>` and `<Integer, Integer>` forms by consulting the six type DAOs
(`getIdOfActionType(tenantId, "VIEW")` and back). The `Domain*ServiceImpl` classes are thin wrappers
that convert inputs to ints, call the core service, and convert outputs back to strings.

The music-specific layer (`MusicRecommenderService`, `MusicActionService`) is a leftover from the
project's origin as a music recommender: it exposes calls like `purchaseTrack`, `viewArtist`,
`mostBoughtTracks`, `alsoViewedArtists`, each of which is just a domain call with fixed action/item
types (`TRACK`, `ARTIST`, `GENRE`, `BUY`, `VIEW`, ...).

---

## 4. Layering inside the module

```
                 +-------------------------------------------------------------+
   Services      |  RecommenderService   ActionService   ItemAssocService      |
   (core, int)   |  RecommendationHistoryService   TenantService               |
                 |  ProfileService / JSONProfileService   ClusterService       |
                 +-------------------------------+-----------------------------+
                                                 |
                 +-------------------------------v-----------------------------+
   DAOs          |  ActionDAO  ItemAssocDAO  RecommendationDAO  RecommendedItemDAO
   (Spring JDBC, |  TenantDAO  ProfileDAO  ItemDAO  AuthenticationDAO  ArchiveDAO
    MySQL)       |  IDMappingDAO   + 6 type DAOs (ActionTypeDAO, AssocTypeDAO, ...)
                 +-------------------------------+-----------------------------+
                                                 |
                 +-------------------------------v-----------------------------+
   Schema        |  src/main/resources/sql/core/*.sql  (one CREATE TABLE each) |
                 |  src/main/migrate/*.sql              (schema upgrades)      |
                 +-------------------------------------------------------------+
```

- Every DAO has an `Abstract*MysqlImpl` base (in `store.dao.impl`) that builds SQL with
  `StringBuilder` and executes through Spring's `JdbcTemplate`. Concrete `*MysqlImpl` classes bind
  the generic types to `Integer`.
- DAOs are `TableCreatingDAO`s: each knows the name of its `CREATE TABLE` script so the schema can be
  bootstrapped by the `SqlScriptService` from easyrec-utils.
- Wiring is Spring XML under `src/main/resources/spring/` (one file per bean).
  `TenantService_AllInOne.xml` is a convenience context that imports everything needed to stand up the
  tenant, item-assoc and cluster services on their own.
- Bulk reads use `ResultSetIteratorMysql` (from easyrec-utils) to stream large tables in chunks
  (`getActionIterator(bulkSize)`, `getItemAssocIterator(bulkSize)`). Plugins use these to scan all
  actions when computing associations.

---

## 5. The recommendation algorithm (what actually happens on a request)

`RecommenderServiceImpl` contains no learning at all. It is a **lookup and filter** over the
`itemassoc` table. There are two strategies.

### 5.1 "Also acted on" items (item-to-item)

`getAlsoActedItems(tenant, user, session, assocType, item, filteredActionType, requestedItemType)`

1. Query `itemassoc` for rows where `itemFrom = item`, `assocTypeId = assocType`,
   `itemToTypeId = requestedItemType`, `active = 1`, limited to `maximumNumberOfRelatedItemsPerItem`
   (default 100). Results are ordered by `assocValue` descending.
2. Convert each associated item into a `RecommendedItemVO` whose `predictionValue` is the
   `assocValue` and whose explanation reads
   *"this item is related to the currently acted on item 'X' via the assoc type 'Y'"*.
3. If `filterResults` is on, run the filter step (5.3).
4. Wrap in a `RecommendationVO` with strategy `itemsAlsoActedOn`, persist it via
   `RecommendationHistoryService`, and return.

This is what powers "users who bought X also bought Y" once a plugin has written `BOUGHT_TOGETHER`
associations.

### 5.2 Items based on action history (user-to-item)

`getItemsBasedOnActionHistory(tenant, user, session, consideredActionType, consideredItemType,
ratingThreshold, numberOfLastActionsConsidered, assocType, requestedItemType)`

1. Fetch the user's recent history with `ActionDAO.getItemsByUserActionAndType`. The SQL groups the
   `action` table by item, filters by tenant, user id and/or session id, action type, optional item
   type, and optional `ratingValue > threshold`, orders by newest action time, and applies
   `LIMIT numberOfLastActionsConsidered`.
2. For each history item, do the same association lookup as 5.1.
3. Concatenate all candidate lists. The same target item may now appear several times (recommended
   by several history items).
4. Filter (5.3), wrap as strategy `itemsBasedOnActionHistory`, log, return.

### 5.3 Filtering (`RecommenderUtils`)

- **Duplicate collapse.** With `useAveragePredictionValues = true` (the hard-coded default) all
  occurrences of the same item are merged into one entry whose prediction value is the **average**
  of the individual association values. Without it the first occurrence wins.
- **History removal.** Items the user (by user id or session) has already acted on with the given
  action type are removed, so a user is not recommended what they already bought.

Both steps are pure in-memory list operations. Note the design consequence stated in the Spring
config: because filtering shrinks the list after the SQL `LIMIT`, callers are advised to raise
`maximumNumberOfRelatedItemsPerItem` if they need a guaranteed result count.

### 5.4 What is *not* here

`recommend(tenant, item, assocType)` is a stub returning `null`, reserved for future online
predictors (e.g. Slope One). Rating-based logic in core is limited to `ActionToRatingAggregator`,
which turns a bag of actions on one `(tenant, user, item)` into a single `RatingVO` using the
aggregate types (`AVERAGE`, `FIRST`, `MAXIMUM`, `MOST_FREQUENT`, `NEWEST`, `OLDEST`) and an
action-type-to-score map. Plugins consume this.

---

## 6. Action ingestion and maintenance

`ActionServiceImpl` / `ActionDAOMysqlImpl` handle the write side:

- `insertAction(action, useDateFromVO)` validates the required fields and inserts a row, using the
  supplied timestamp or `now()`.
- `getRankedItemsByActionType` and `getRankedItemsByActionTypeAndCluster` produce popularity
  rankings ("most viewed", "most bought") within a time window. The music layer's `mostBoughtTracks`
  etc. are wrappers over this. Results are cached in the Ehcache `RANKINGS_CACHE`.
- **Session-to-user mapping.** `getMultiUserSessions`, `getUserIdsOfSession` and
  `updateActionsOfSession` support a background job that re-attributes anonymous session actions to
  a user once that user logs in. The feature toggle lives in tenant config as
  `SESSION_TO_USER_MAPPING.enabled`.
- **Archiving.** `ArchiveDAO` moves actions older than a reference date into `actionarchive` tables
  and rolls over to a new archive table when one gets full. The tenant config keys are
  `AUTO_ARCHIVER.enabled` and `AUTO_ARCHIVER.timeRange` (default 1825 days).
- **CSV auto-import.** `ActionAICommandImpl` and `ItemAssocAICommandImpl` plug into the
  `AutoImportService` from easyrec-utils. Files dropped in a watched directory are parsed by
  `ActionServiceImpl.importActionsFromCSV` / `ItemAssocServiceImpl.importItemAssocsFromCSV`. The file
  format has a `# type:` and `# command:` header, then a header row where a column may carry a
  default value (`itemTypeId=4(track)`), then data rows. See `docs/examples/data/`.

---

## 7. Item associations (`ItemAssocService`)

This is the write interface used by generator plugins after they compute similarities:

- `insertOrUpdateItemAssoc(s)` upserts on the unique key
  `(tenant, itemFrom, itemFromType, itemTo, itemToType, assocType, sourceType)`.
- `activateItemAssoc` / `deactivateItemAssoc` soft-toggle rows; only `active = 1` rows are served.
- `removeAllItemAssocsFromSource(sourceType[, sourceInfo])` and
  `removeItemAssocByTenantAndThreshold` let a plugin wipe or prune its own previous output before
  writing a fresh run. `sourceInfo` is a free-text tag (e.g. a run id) for that purpose.
- `getItemsTo` / `getItemsFrom` are the read paths, constrained by `IAConstraintVO`
  (result count, tenant, view type, source type, active flag, sort direction and sort field).

Because the key includes `sourceType`, several plugins can each maintain their own edge between the
same two items without overwriting each other.

---

## 8. Profiles (`ProfileService`, `JSONProfileServiceImpl`)

A profile is an arbitrary document stored in `profile.profileData` for an item:

- `ProfileServiceImpl` treats it as **XML**. It can validate against the item type's
  `profileSchema`, and it exposes XPath-addressed field operations: `loadProfileField`,
  `storeProfileField` (creates missing path elements bottom-up), `deleteProfileField`.
- `JSONProfileServiceImpl` does the same for **JSON** using json-path with a Jackson provider.

Profiles are what content-based plugins (e.g. profile similarity) read to compute the
`PROFILE_SIMILARITY` association type.

---

## 9. Clusters (`ClusterService`)

Clusters give operators a manual way to group items into a hierarchy and pull items from those
groups (e.g. "show 5 items from the *Summer Sale* cluster"). Implementation is notable because it
reuses the existing model rather than adding tables:

- A cluster **is an item** of the hidden item type `CLUSTER`; its name/description are stored as a
  JAXB-serialised `ClusterVO` in the cluster item's profile.
- Parent-child links are `itemassoc` rows of hidden assoc type `IS_PARENT_OF`.
- Membership of a real item in a cluster is an `itemassoc` row of hidden assoc type `BELONGS_TO`
  (item -> cluster, value 1.0, source `MANUALLY_CREATED`, view `ADMIN`).
- Every tenant gets a `CLUSTERS` root on startup (`initTenantForClusters`); the whole tree is loaded
  into memory as a JUNG `DelegateTree` per tenant and kept in sync on add/move/rename/remove.
- Item retrieval from a cluster is pluggable through `ClusterStrategy`:
  `NEWEST` (order by `changeDate`, the default), `BEST` (order by assoc value), `RANDOM`
  (over-fetch then sample). An optional fallback walks up to parent clusters if a cluster has too
  few items.

---

## 10. Tenant management and configuration

`TenantService` creates and removes tenants together with their type tables and authentication
domains, and exposes two `Properties` bags stored as blobs on the `tenant` row:

- **tenantConfig**: feature toggles and scheduler settings. Well-known keys are constants on
  `RemoteTenant`: `plugins.enabled`, `AUTO_RULEMINER.enabled`, `AUTO_RULEMINER.executionTime`
  (default `02:00`), `AUTO_ARCHIVER.*`, `SESSION_TO_USER_MAPPING.enabled`, `backtracking`,
  `TENANT.maxactions`.
- **tenantStatistic**: counters computed by the web layer and stored back (`TENANT.actions`,
  `TENANT.users`, `ASSOC.rules.<type>`, `CONVERSION.recommendationToBuyCount`, ...).

`AuthenticationDAO` keeps the whitelist of domain URLs allowed to call the API for a tenant.

---

## 11. Web-facing leftovers in core

`model.core.web` and `util.core` hold classes that logically belong to easyrec-web but were moved
down "so plugins can use them": `Item`, `RemoteTenant`, `Operator` (an admin login, with API key and
access level), `Session`, `Message`/`SuccessMessage`/`ErrorMessage`, the statistic DTOs, and helper
classes `Security` (operator sign-in stored in the HTTP session), `Web` (URL/email validation, XSLT
transform) and `MessageBlock` (builds Spring `ModelAndView`s for message pages).

The `backtracking` table (see `src/main/migrate/backtracking.sql`) records which recommendation a
user clicked on, which is the basis for the conversion statistics.

---

## 12. End-to-end flow summary

```
  website ---(REST, easyrec-web)---> DomainActionService.insertAction("VIEW", "sku-42")
                                          |  TypeMappingService: "VIEW"->3, IDMappingDAO: "sku-42"->1017
                                          v
                                     action table

  nightly   plugin (e.g. ARM) ---> ActionService.getActionIterator(bulk)
                                    computes co-occurrence / similarity
                                    ItemAssocService.insertOrUpdateItemAssocs(...)   ---> itemassoc table

  website ---(REST)---> DomainRecommenderService.alsoBoughtItems(tenant, user, session, item)
                            |  ids/types -> ints
                            v
                        RecommenderService.getAlsoActedItems
                            |  ItemAssocDAO.getItemsTo (top-N by assocValue, active only)
                            |  RecommenderUtils.filterDuplicates (average) + filterAlreadyActedOn
                            |  RecommendationHistoryService.insertRecommendation (audit log)
                            v
                        RecommendationVO -> back to strings -> JSON/XML response
```

## 13. Things worth knowing when working in this code

- The recommender is entirely **precomputed**: response quality depends wholly on what plugins wrote
  into `itemassoc`. Nothing in core learns online.
- MySQL-specific SQL (`LIMIT`, `ON DUPLICATE KEY`, `LIKE` for sessions) is built by hand in the
  DAOs; the code comments flag this as the place to change for another database.
- Caching is aspect-driven (`@ShortCacheable`, `@LongCacheable` from easyrec-utils). Type lookups and
  id mappings are long-cached, so changing a type name in the DB without a restart is unsafe.
- Tests (`src/test`) are DbUnit-based DAO and service tests against a real MySQL schema, with
  datasets under `src/test/resources/dbunit`.
- `ActionToRatingAggregator.MOST_FREQUENT` compares strings with `==`, a latent bug if action type
  strings are not interned.
