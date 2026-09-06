# easyrec-core: The Recommendation Algorithm

Companion document: `easyrec-core-technical-details.md` (architecture, data model, DAOs, Spring wiring).

This note explains *how easyrec produces a recommendation*. The short version: easyrec is a
**precomputed, association-based, item-to-item recommender**. At request time it does no learning.
It looks up previously stored item-to-item edges, merges them, filters them, and returns the result.
The intelligence lives in plugins that populate the `itemassoc` table offline.

Source: `org.easyrec.service.core.impl.RecommenderServiceImpl`, `org.easyrec.util.core.RecommenderUtils`,
`org.easyrec.store.dao.core.impl.ActionDAOMysqlImpl`, `org.easyrec.store.dao.core.impl.ItemAssocDAOMysqlImpl`,
`org.easyrec.util.domain.ActionToRatingAggregator`.

---

## 1. The two inputs the algorithm relies on

### 1.1 Actions (the user history)

Every interaction is a row in the `action` table:

```
(tenantId, userId, sessionId, ip, itemId, itemTypeId, actionTypeId, ratingValue, actionTime)
```

Action types are per tenant. Defaults are `VIEW`, `RATE`, `BUY`. `RATE` carries a numeric
`ratingValue`, the others do not.

### 1.2 Item associations (the precomputed knowledge)

An association is a directed, weighted edge between two items:

```
itemFrom  --[assocType, assocValue]-->  itemTo      (plus sourceType, viewType, active, changeDate)
```

Default association types: `VIEWED_TOGETHER`, `GOOD_RATED_TOGETHER`, `BOUGHT_TOGETHER`,
`IS_RELATED`, `PROFILE_SIMILARITY`. The `assocValue` is whatever score the producing plugin
computed (a confidence, a similarity, a co-occurrence count). Core treats it as an opaque
"the higher the better" number.

Associations are produced by generator plugins in the `easyrec-plugins` module, typically on a
nightly schedule (`AUTO_RULEMINER.executionTime`, default 02:00). Examples:

- **ARM (association rule miner):** scans actions, finds items that co-occur in the same
  user's or session's history, writes `BOUGHT_TOGETHER` / `VIEWED_TOGETHER` edges.
- **Profile similarity:** compares item profile documents, writes `PROFILE_SIMILARITY`.
- **Slope One:** rating-based predictor.

Core only exposes the raw materials to them: `ActionService.getActionIterator(bulkSize)` to stream
all actions, and `ItemAssocService.insertOrUpdateItemAssocs(...)` to write results back.

---

## 2. Strategy A: "also acted on" (item-to-item)

Entry point: `RecommenderService.getAlsoActedItems(tenant, user, session, assocType, item, filteredActionType, requestedItemType)`

This is the "customers who bought X also bought Y" query. Given one item, return the items most
strongly associated with it.

```
Input: item X, assocType = BOUGHT_TOGETHER, requestedItemType = ITEM

1. candidates = itemassoc rows WHERE itemFrom = X
                                AND assocTypeId = BOUGHT_TOGETHER
                                AND itemToTypeId = ITEM
                                AND active = 1
                ORDER BY assocValue DESC
                LIMIT maximumNumberOfRelatedItemsPerItem      (default 100)

2. for each candidate:
       RecommendedItem(item = itemTo,
                       predictionValue = assocValue,
                       itemAssocId = row id,
                       explanation = "this item is related to the currently acted on item 'X'
                                      via the assoc type 'BOUGHT_TOGETHER'")

3. if filterResults:  apply the filter pipeline (section 4)

4. wrap in RecommendationVO(strategy = "itemsAlsoActedOn",
                            explanation = "items that are often acted on together with the given item")
   persist via RecommendationHistoryService
   return
```

The `filteredActionType` parameter is only used by the filter step: items the user already did
*that* action on are removed from the result.

---

## 3. Strategy B: items based on action history (user-to-item)

Entry point: `RecommenderService.getItemsBasedOnActionHistory(tenant, user, session, consideredActionType, consideredItemType, ratingThreshold, numberOfLastActionsConsidered, assocType, requestedItemType)`

This is the personalised query: "given what this user did recently, what should they see next?"
It is Strategy A applied once per history item, then merged.

### 3.1 Step one: fetch the user's recent history

`ActionDAO.getItemsByUserActionAndType` runs a query of this shape:

```sql
SELECT tenantId, itemId, itemTypeId, MAX(actionTime) AS maxActionTime
FROM   action
WHERE  tenantId = ?
  AND  userId = ?                 -- if a user id is given
  AND  sessionId LIKE ?           -- if a session id is given (anonymous users)
  AND  actionTypeId = ?           -- consideredActionType, e.g. BUY
  AND  ratingValue > ?            -- only if ratingThreshold given (for RATE-based queries)
  AND  itemTypeId = ?             -- consideredItemType, if given
GROUP BY itemId
ORDER BY maxActionTime DESC
LIMIT  ?                          -- numberOfLastActionsConsidered
```

Either a user id or a session id is required, which is how easyrec supports anonymous visitors.
The result is the N most recently touched distinct items.

### 3.2 Step two: expand each history item

For every history item H, run the association lookup from Strategy A:
`ItemAssocService.getItemsTo(H, assocType, requestedItemType, top-100, active only)`.
All candidate lists are concatenated into one list. At this point the same target item may appear
several times, once per history item that points to it.

### 3.3 Step three: filter, wrap, log

Apply the filter pipeline (section 4), wrap as strategy `itemsBasedOnActionHistory` with the
explanation *"items that are related with those items the user acted on lately"*, persist, return.

---

## 4. The filter pipeline (`RecommenderUtils`)

Both strategies pass through `doFiltering`, which is two in-memory list operations.

### 4.1 Duplicate collapse with averaging

`filterDuplicates(list, useAveragePredictionValues)`. The flag is hard-coded to `true`.

```
for each recommended item r:
    sum[r.item]   += r.predictionValue
    count[r.item] += 1
    keep the first occurrence of r.item in order

result = for each kept item:
    if count == 1: the original entry
    else:          a copy with predictionValue = sum / count   (rounded to 16 decimals)
```

So an item recommended by three of the user's history items ends up with the **average** of the
three association values, not the sum. This is a deliberate choice that keeps scores on the same
scale as single associations, but it also means an item that is weakly related to many history
items does not outrank an item strongly related to one. With the flag off, only the first
occurrence is kept and its value is untouched.

Note that the duplicate step does **not** re-sort. Ordering is whatever the concatenation
produced: first history item's candidates (by value desc), then the second's, and so on.

### 4.2 Remove items the user already acted on

`filterAlreadyActedOn(recommended, itemsActedOn)`.

If a user id or session id is known, the algorithm fetches *all* items the user did the filtered
action type on (no limit, no threshold) and drops any recommendation whose item is in that set.
This prevents recommending something the user already bought or viewed.

### 4.3 Consequence: result count is not guaranteed

The SQL `LIMIT` is applied *before* both filters, so the final list can be shorter than requested.
The Spring configuration comment says so explicitly and advises raising
`easyrec.recService.maximumNumberOfRelatedItemsPerItem` to compensate. `easyrec.recService.filterResults`
turns the whole pipeline off.

---

## 5. Explanations and the recommendation log

Every returned `RecommendationVO` carries:

- a `recommendationStrategy` label (`itemsAlsoActedOn` or `itemsBasedOnActionHistory`),
- a human-readable `explanation` for the whole recommendation,
- per item: the `predictionValue`, the `itemAssocId` of the edge that produced it, and a per-item
  explanation string naming the source item and association type.

`RecommendationHistoryService.insertRecommendation` writes the recommendation and each
recommended item to the `recommendation` and `recommendeditem` tables. Combined with the
`backtracking` table (which records which recommended item a user later clicked), this is the raw
data for the conversion statistics shown in the admin UI
(`CONVERSION.recommendationToBuyCount`).

---

## 6. Popularity rankings (non-personalised fallback)

Not part of `RecommenderService`, but the other recommendation-like output core provides.
`ActionDAO.getRankedItemsByActionType(tenant, actionType, numberOfResults, timeRange, sortDescending)`
counts actions per item within a time window and returns the top N as `RankedItemVO`s. The music
layer exposes these as `mostBoughtTracks`, `mostViewedArtists`, and so on. Results are cached in
the `RANKINGS_CACHE` Ehcache region. A variant restricted to a cluster
(`getRankedItemsByActionTypeAndCluster`) supports "most popular in category" lists.

---

## 7. Turning actions into ratings (`ActionToRatingAggregator`)

Rating-based plugins need one number per `(user, item)`, but a user may have several actions on
the same item (viewed it twice, then bought it). `ActionToRatingAggregator.getRating(actions, actionMapping, aggregateType)`
collapses them:

| Aggregate type | Result |
|---|---|
| `AVERAGE` | mean of the mapped scores of all actions |
| `FIRST` | score of the first action in the collection |
| `MAXIMUM` | highest mapped score |
| `MOST_FREQUENT` | score of the action type that occurs most often |
| `NEWEST` | score of the most recent action |
| `OLDEST` | score of the oldest action |

`actionMapping` is a caller-supplied map from action type name to a numeric score
(for example VIEW -> 1, BUY -> 5). The tenant's `ratingRangeMin`, `ratingRangeMax` and
`ratingRangeNeutral` fields exist so plugins can normalise these scores.

Known defect: the `MOST_FREQUENT` branch compares action type strings with `==` instead of
`equals`, so it only works when the strings happen to be the same instance.

---

## 8. Cluster-based selection

Clusters are manually curated groups of items. `ClusterService.getItemsOfCluster(cluster, strategy, useFallback, n, itemType)`
pulls items from a cluster using a pluggable `ClusterStrategy`:

| Strategy | Behaviour |
|---|---|
| `NEWEST` (default) | items ordered by the `changeDate` of their membership edge |
| `BEST` | items ordered by the membership edge's `assocValue` |
| `RANDOM` | fetch up to 2N members, then sample N uniformly |

With `useFallback`, if a cluster yields fewer than N items the service walks up to the parent
cluster and fills from there. Membership is stored as ordinary `BELONGS_TO` associations, so
cluster retrieval is the same top-N association query as everything else.

---

## 9. What the algorithm does not do

- **No online learning.** `recommend(tenant, item, assocType)` is a stub returning `null`,
  reserved for future online predictors such as Slope One.
- **No user-to-user similarity.** There is no notion of "similar users"; personalisation comes
  only from expanding the user's own recent items through item-to-item edges.
- **No re-ranking or diversity.** After duplicate averaging the list keeps its concatenation
  order. Any final sorting or truncation is left to the caller (the web layer).
- **No cold-start handling in core.** A new item with no associations simply yields an empty
  list; the popularity rankings and clusters are the intended fallbacks.

---

## 10. Worked example

Tenant "shop", user 17 bought items A and B in that order (B most recent).
Plugin ARM has written these `BOUGHT_TOGETHER` edges:

```
B -> C (0.8)    B -> D (0.6)    B -> A (0.5)
A -> D (0.4)    A -> E (0.9)
```

Call: `getItemsBasedOnActionHistory(shop, 17, null, BUY, ITEM, null, 2, BOUGHT_TOGETHER, ITEM)`

1. History (2 most recent BUY items, newest first): `[B, A]`
2. Expand B: `[C 0.8, D 0.6, A 0.5]`. Expand A: `[D 0.4, E 0.9]`.
   Concatenated: `[C 0.8, D 0.6, A 0.5, D 0.4, E 0.9]`
3. Duplicate collapse: D appears twice, averaged to 0.5. `[C 0.8, D 0.5, A 0.5, E 0.9]`
4. Already-acted-on filter: user bought A and B, so A is removed. `[C 0.8, D 0.5, E 0.9]`
5. Returned in that order with strategy `itemsBasedOnActionHistory`, and logged.

Note E has the highest score but is last, because core does not re-sort after merging.
