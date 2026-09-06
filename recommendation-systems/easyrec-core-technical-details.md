# easyrec-core: Technical Details

Companion document: `easyrec-core-recommendation-algorithm.md` (how recommendations are computed).

Source analyzed: `easyrec-code/easyrec-core` (Maven artifact `org.easyrec:easyrec-core`, version 1.0.2-SNAPSHOT).
easyrec is an open-source, GPL-licensed recommendation engine written in Java, originally built by
Research Studios Austria. `easyrec-core` is its foundation module: the domain model, the MySQL data
access layer, and the basic services (`ActionService`, `ItemAssocService`, `RecommenderService`,
`RecommendationHistoryService`, `TenantService`, `ProfileService`, `ClusterService`). It does
**not** contain the algorithms that compute item associations (those are in `easyrec-plugins`) nor
the REST/web layer (`easyrec-web`).

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

Key dependencies: Spring (JDBC, Web MVC), JUNG (graph library, used for cluster trees), Ehcache
(via Spring cache annotations from easyrec-utils), json-path + Jackson (JSON profiles),
commons-validator, javax.mail.

Source layout:

```
src/main/java/org/easyrec/
├── model/core/            value objects (ItemVO, ActionVO, ItemAssocVO, RecommendationVO, ...)
│   ├── transfer/          constraint objects (IAConstraintVO, TimeConstraintVO)
│   └── web/               web-level DTOs moved down so plugins can use them
├── service/               generic Base*Service interfaces
│   ├── core/  + impl/     Integer-typed service interfaces and implementations
│   └── domain/ + impl/    String-typed wrappers, TypeMappingService, music sub-domain
├── store/dao/             generic Base*DAO interfaces, IDMappingDAO
│   ├── impl/              AbstractBase*MysqlImpl (SQL building)
│   ├── core/ + impl/      Integer-typed DAOs
│   ├── core/types/        the six type-table DAOs
│   └── domain/ + impl/    String-typed DAOs
├── util/core/             RecommenderUtils, Security, Web, MessageBlock
└── util/domain/           ActionToRatingAggregator
src/main/resources/spring/ one XML file per bean
src/main/resources/sql/    one CREATE TABLE script per table
src/main/migrate/          schema upgrade scripts between versions
docs/examples/data/        sample CSV import files
```

---

## 2. Data model

Everything is scoped by a **tenant** (one website or customer of the recommender).

| Concept | Meaning | Table |
|---|---|---|
| **Tenant** | An isolated customer/site. Numeric id, string id, rating range (min/max/neutral), `active` flag, and two blobs of `Properties` (`tenantConfig`, `tenantStatistic`). | `tenant` |
| **Item** | Anything that can be recommended. Identified by `(tenantId, itemId, itemTypeId)`. Web-level metadata (description, url, imageUrl) lives in `item`. | `item`, `idmapping` |
| **Action** | An event a user performed on an item. Carries user id, session id, IP, optional rating value, description, timestamp. | `action`, archived into `actionarchive` |
| **ItemAssoc** | A directed weighted edge `itemFrom --assocType(value)--> itemTo` with source type, view type, `active` flag and change date. | `itemassoc` |
| **Recommendation / RecommendedItem** | Audit log of what was recommended to whom, with a strategy label and per-item explanation. | `recommendation`, `recommendeditem` |
| **Profile** | Free-form XML or JSON document attached to an item. | `profile` |
| **Cluster** | A named group of items in a per-tenant tree. Built on items and associations, no own table. | reuses `itemassoc`, `profile` |
| **Authentication** | Whitelisted domain URLs per tenant allowed to call the API. | `authentication` |

Important indexes: `action(tenantId, userId, actionTypeId, itemTypeId)` for history reads;
`itemassoc(itemFromId, itemFromTypeId, itemToTypeId, assocTypeId, tenantId, active)` for the
recommender read path; unique key on `itemassoc(tenantId, itemFromId, itemFromTypeId, itemToId, itemToTypeId, assocTypeId, sourceTypeId)`
for upserts.

### 2.1 Type tables: everything is parameterised per tenant

Instead of hard-coding "VIEW" or "BOUGHT_TOGETHER", six lookup tables map
`(tenantId, name) -> id`:

| Table | Purpose | Defaults for a new tenant |
|---|---|---|
| `itemtype` | Kinds of items. Also stores optional `profileSchema` and `profileMatcher` blobs. `visible` flag. | `ITEM` |
| `actiontype` | Kinds of user actions. `hasvalue` marks types that carry a numeric value. | `VIEW`, `RATE` (hasvalue), `BUY` |
| `assoctype` | Kinds of item-to-item relationships. `visible` flag hides internal ones. | `VIEWED_TOGETHER`, `GOOD_RATED_TOGETHER`, `BOUGHT_TOGETHER`, `IS_RELATED`, `PROFILE_SIMILARITY` |
| `sourcetype` | Who produced an association (plugin name or manual). | `MANUALLY_CREATED` |
| `viewtype` | Intended audience of an association. | `ADMIN`, `COMMUNITY`, `SYSTEM` |
| `aggregatetype` | How to collapse several actions into one rating. | `AVERAGE`, `FIRST`, `MAXIMUM`, `MOST_FREQUENT`, `NEWEST`, `OLDEST` |

Defaults come from the `tenantConfig_default` bean in `spring/core/TenantConfig_DEFAULT.xml` and
are inserted by `TenantServiceImpl.insertTenantWithTypes`. Type lookups are cached with
`@LongCacheable`, so renaming a type in the database without a restart is unsafe.

### 2.2 ID mapping

External callers use string item ids (for example a SKU). Internally everything is an `Integer`.
`IDMappingDAO` keeps a global `idmapping (intId, stringId)` table:

- `lookup(String)` returns the int id and **inserts a new mapping if absent**.
- `lookupOnly(String)` returns the int id or throws `ItemNotFoundException`.
- `lookup(Integer)` reverses the mapping.

All three are `@LongCacheable`.

---

## 3. The generic-types design: one model, three layers

The value objects are generic in two type parameters, for example `ItemVO<I, T>`,
`ActionVO<I, T>`, `ItemAssocVO<I, T>`, `RecommendationVO<I, T>`. `I` is the id type and `T` is the
"type" type (item type, action type, assoc type).

```
        Web / plugin level             Domain level                     Core level
   ItemVO<Integer, String>   <-->  ItemVO<Integer, String>   <-->  ItemVO<Integer, Integer>
   (string types, e.g. "VIEW")      (string types)                  (int type ids, e.g. 3)
```

| Layer | Packages | Ids | Types | Example |
|---|---|---|---|---|
| Generic interfaces | `service.Base*Service`, `store.dao.Base*DAO` | `I` | `T` | `BaseRecommenderService<R, I, ACT, IT, AT, T, U>` |
| Core | `service.core`, `store.dao.core` | `Integer` | `Integer` | `RecommenderService`, `ActionDAO` |
| Domain | `service.domain`, `store.dao.domain` | `Integer` | `String` | `DomainRecommenderService`, `TypedActionDAO` |
| Music domain | `service.domain.music` | `Integer` | fixed in method names | `MusicRecommenderService.alsoBoughtTracks(...)` |

`TypeMappingService` is the bridge. It converts every VO between `<Integer, String>` and
`<Integer, Integer>` by consulting the six type DAOs (`getIdOfActionType(tenantId, "VIEW")` and
back), for single objects and for lists. The `Domain*ServiceImpl` classes are thin wrappers:
convert inputs to ints, call the core service, convert outputs back to strings.

The music layer (`MusicRecommenderService`, `MusicActionService`) is a leftover from the project's
origin as a music recommender. It exposes `purchaseTrack`, `viewArtist`, `mostBoughtTracks`,
`alsoViewedArtists` and so on, each a domain call with fixed types (`TRACK`, `ARTIST`, `GENRE`,
`BUY`, `VIEW`). `TypeMappingService` still carries the constants (`ITEM_TYPE_TRACK`,
`SOURCE_TYPE_AMG`, ...).

---

## 4. Layering and infrastructure

```
   Services   RecommenderService  ActionService  ItemAssocService  RecommendationHistoryService
   (core)     TenantService  ProfileService / JSONProfileService  ClusterService
                                          |
   DAOs       ActionDAO  ItemAssocDAO  RecommendationDAO  RecommendedItemDAO  TenantDAO
   (Spring    ProfileDAO  ItemDAO  AuthenticationDAO  ArchiveDAO  IDMappingDAO
    JDBC)     ActionTypeDAO  AssocTypeDAO  ItemTypeDAO  SourceTypeDAO  ViewTypeDAO  AggregateTypeDAO
                                          |
   Schema     sql/core/*.sql (CREATE TABLE)      migrate/*.sql (upgrades)
```

- **DAO implementation.** Each DAO has an `AbstractBase*MysqlImpl` (in `store.dao.impl`) that
  builds SQL with `StringBuilder` and runs it through Spring's `JdbcTemplate`, with `RowMapper`
  inner classes for each VO. Concrete `*MysqlImpl` classes bind the generics to `Integer`. MySQL
  specifics (`LIMIT`, `ON DUPLICATE KEY`, `LIKE` on sessions) are hand-written and flagged in
  comments as the place to change for another database.
- **Table creation.** DAOs implement `TableCreatingDAO` and know the name of their `CREATE TABLE`
  script, so the schema can be bootstrapped by the `SqlScriptService` from easyrec-utils.
- **Streaming reads.** `ResultSetIteratorMysql` (easyrec-utils) streams large tables in chunks:
  `getActionIterator(bulkSize[, timeConstraint])`, `getItemAssocIterator(bulkSize)`,
  `getRecommendationIterator(bulkSize)`. Plugins use these to scan all actions.
- **Caching.** Aspect-driven via `@ShortCacheable` / `@LongCacheable` (easyrec-utils AOP).
  Ehcache regions `RANKINGS_CACHE` and `ITEMS_CACHE` are declared in `spring/core/common/rankingsCache.xml`.
- **Spring wiring.** One XML per bean under `src/main/resources/spring/core` and `spring/domain`.
  `TenantService_AllInOne.xml` is a self-contained context importing everything needed for the
  tenant, item-assoc and cluster services. Placeholders use the `$easyrec{...}` syntax from
  `PropertyPlaceholderConfigurerEasyrec.xml`, resolved from `easyrec.properties`.
- **Tests.** DbUnit-based DAO and service tests against a real MySQL schema, with datasets under
  `src/test/resources/dbunit`. Local config lives in `src/main/resources.localhost`.

---

## 5. Services

### 5.1 ActionService

- `insertAction(action, useDateFromVO)` validates tenant, item, item type and action type, then
  inserts using the VO's timestamp or `now()`.
- `getItemsByUserActionAndType(...)` returns the user's most recent distinct items for an action
  type (the history query used by the recommender).
- `getRankedItemsByActionType` / `...AndCluster` compute popularity rankings in a time window.
- `getDirectItemRatings` returns explicit ratings for a user.
- **Session-to-user mapping.** `getMultiUserSessions`, `getUserIdsOfSession`,
  `updateActionsOfSession` support a job that re-attributes anonymous session actions to a user
  after login. Toggle: tenant config `SESSION_TO_USER_MAPPING.enabled`.
- `importActionsFromCSV(fileName[, defaults])` bulk-loads actions (see section 7).

### 5.2 ItemAssocService

The write interface used by generator plugins:

- `insertOrUpdateItemAssoc(s)` upserts on the unique key that includes `sourceType`, so several
  plugins can each maintain their own edge between the same two items.
- `activateItemAssoc` / `deactivateItemAssoc` soft-toggle rows. Only `active = 1` rows are served.
- `removeAllItemAssocsFromSource(sourceType[, sourceInfo])` and
  `removeItemAssocByTenantAndThreshold` let a plugin wipe or prune its previous output before a
  fresh run. `sourceInfo` is a free-text tag such as a run id.
- `getItemsTo` / `getItemsFrom` / `getItemAssocs` are the read paths. `IAConstraintVO` carries
  result count, tenant, view type, source type, source info, `active` flag, sort direction and
  sort field.

### 5.3 RecommendationHistoryService

Persists each `RecommendationVO` and its `RecommendedItemVO`s, and exposes iterators and
time-window queries over them for statistics.

### 5.4 TenantService

Creates and removes tenants together with their type tables and authentication domains. Exposes
two `Properties` bags stored as blobs on the `tenant` row:

- **tenantConfig**: feature toggles and scheduler settings. Well-known keys are constants on
  `RemoteTenant`: `plugins.enabled`, `AUTO_RULEMINER.enabled`, `AUTO_RULEMINER.executionTime`
  (default `02:00`), `AUTO_ARCHIVER.enabled`, `AUTO_ARCHIVER.timeRange` (default 1825 days),
  `SESSION_TO_USER_MAPPING.enabled`, `backtracking`, `backtrackingURL`, `TENANT.maxactions`.
- **tenantStatistic**: counters computed by the web layer (`TENANT.actions`, `TENANT.users`,
  `ASSOC.rules.<type>`, `CONVERSION.recommendationToBuyCount`, ...).

### 5.5 ProfileService and JSONProfileServiceImpl

A profile is an arbitrary document in `profile.profileData`:

- `ProfileServiceImpl` treats it as **XML**. It can validate against the item type's
  `profileSchema` and offers XPath-addressed operations: `loadProfileField`,
  `storeProfileField` (creates missing path elements bottom-up, refuses if the XPath matches more
  than one node), `deleteProfileField`.
- `JSONProfileServiceImpl` does the same for **JSON** with json-path and a Jackson provider.

Content-based plugins read profiles to compute `PROFILE_SIMILARITY`.

### 5.6 ClusterService

Manual item grouping in a per-tenant tree, implemented without new tables:

- A cluster **is an item** of the hidden item type `CLUSTER`; its name and description are a
  JAXB-serialised `ClusterVO` stored in that item's profile.
- Parent-child links are `itemassoc` rows of hidden assoc type `IS_PARENT_OF`.
- Membership is an `itemassoc` row of hidden assoc type `BELONGS_TO`
  (item -> cluster, value 1.0, source `MANUALLY_CREATED`, view `ADMIN`).
- Every tenant gets a `CLUSTERS` root on startup (`initTenantForClusters`), and the whole tree is
  held in memory as a JUNG `DelegateTree` per tenant, kept in sync on add, move, rename, remove.
- Item retrieval uses pluggable `ClusterStrategy` beans: `NEWEST` (default), `BEST`, `RANDOM`.

---

## 6. Archiving

`ArchiveDAO` moves actions older than a reference date from `action` into `actionarchive` tables
(`moveActions`), reports the current archive size, and rolls over to a new archive table when the
current one is full (`generateNewArchive`). Driven by the `AUTO_ARCHIVER.*` tenant config keys.

---

## 7. CSV auto-import

`ActionAICommandImpl` and `ItemAssocAICommandImpl` implement the `AutoImportCommand` interface from
easyrec-utils. `AutoImportService` (configured in `spring/core/autoimport/AutoImportService.xml`,
properties `easyrec.autoimport.active`, `.directory`, `.timeout`) watches a directory and
dispatches files by their `# type:` header to the matching command.

File format (`docs/examples/data/actions/actions.csv`):

```
# type: action
# command: insert
tenantId=0,userId,sessionId,ip,itemId,itemTypeId=4(track),actionTypeId,ratingValue,searchSucceeded,numberOfFoundItems,description
,17,xyz0,127.0.0.1,25722,,3,,,,test_description1
```

A header column may carry a default value (`itemTypeId=4(track)`) applied when the data cell is
empty. The service reports progress every `reportBlockSize` rows and, for item associations,
optionally overwrites duplicates (`importOverwriteDuplicates`).

---

## 8. Web-facing classes kept in core

`model.core.web` and `util.core` hold classes that logically belong to easyrec-web but were moved
down "so plugins can use them":

- `Item`, `RemoteTenant`, `Operator` (an admin login with API key and access level), `Session`.
- `Message`, `SuccessMessage`, `ErrorMessage`, and the statistic DTOs
  (`TenantStatistic`, `UserStatistic`, `AssocStatistic`, `ConversionStatistic`, `RuleMinerStatistic`, `ItemDetails`).
- `Security` (operator sign-in stored in the HTTP session, developer access check),
  `Web` (URL and email validation, XSLT transform, link helpers),
  `MessageBlock` (builds Spring `ModelAndView`s for message pages).
- `ThirdPartyAccess` / `ItemOutput`: an interface for rendering an item from an external store.

The `backtracking` table (`src/main/migrate/backtracking.sql`) records which recommended item a
user clicked, feeding the conversion statistics.

---

## 9. Schema migrations

`src/main/migrate/` holds incremental upgrade scripts, applied by hand:

| Script | Change |
|---|---|
| `v1_2_to_v1_3.sql` | adds `tenant.active`, widens `tenant.stringId` |
| `v1_3_to_v1_4.sql` | adds `profile` table, `itemtype.profileSchema/profileMatcher`, the `action_reader` index |
| `migrate_easyrec.sql` | renames source type `UM` to `ARM`, adds `tenant.tenantConfig`, rebuilds the `itemassoc` unique key to include `sourceTypeId` |
| `tenant_actions.sql` | adds a charts index on `action`, `tenant.tenantStatistic` |
| `item_creationdate.sql` | adds `item.creationdate` |
| `operator_logincount.sql` | adds `operator.lastlogin`, `operator.logincount` |
| `actionarchive.sql`, `backtracking.sql` | create the archive and backtracking tables |

---

## 10. Things worth knowing when working in this code

- The recommender is entirely **precomputed**: response quality depends wholly on what plugins
  wrote into `itemassoc`. See the algorithm document for the read path.
- Type and id lookups are long-cached, so changing type names or id mappings in the database
  requires a restart.
- All SQL is hand-built and MySQL-specific.
- `ActionToRatingAggregator.MOST_FREQUENT` compares strings with `==`, a latent bug.
- Constants scattered across `TypeMappingService`, `ClusterService`, `RemoteTenant` and
  `RecommenderServiceImpl` are the de-facto vocabulary of the system; the code comments reference
  Mantis issues proposing to centralise them.
