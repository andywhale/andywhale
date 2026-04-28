---
title: "Date-Based Boosting in Drupal Search: A History and Technical Guide"
description: "Why search results show ten-year-old articles alongside today's events, how the reciprocal decay function solves it, and how to implement it in Search API Solr."
date: 2026-04-28T13:21:43+01:00
image: "images/search-api-solr-date-boost.png"
draft: false
type: "post"
tags: ["drupal", "search", "solr", "search-api", "performance"]
---

There's a quiet frustration that appears on every content-heavy website once search starts mattering: a visitor searches for "gallery tour" and gets back both next Saturday's upcoming event and one that ran in 2019, ranked equally by the search engine because both mention "gallery tour" the right number of times.

The search system sees these as equally relevant. The visitor sees a broken search experience.

This is the problem that date-based boosting solves. And it's worth understanding not just how to implement it, but why the approach has evolved through a decade of Drupal API changes, and what the mathematics actually mean.

## The Problem: Search Systems Don't Know About Time

Standard search relevance scoring — the algorithms behind Solr, Elasticsearch, and other search engines — works through tf-idf (term frequency-inverse document frequency) and its modern successors. These algorithms score documents by how well their content matches the query. They have no concept of time.

A document that mentions "gallery tour" twenty times will outscore one that mentions it twice, regardless of which event is actually happening. For a museum's What's On section, a ticketing platform, a news archive, or a job board, recency (or for events, upcoming-ness) is part of what makes a result useful.

Date-based boosting solves this by nudging the relevance calculation in the direction that reflects real user intent, without entirely discarding textual relevance. [As described in the Search API Solr documentation](https://www.drupal.org/docs/8/modules/search-api-solr/search-api-solr-howtos/boosting-by-date):

A highly relevant older article can still outrank a barely relevant new one; the boost is a thumb on the scale, not a hard sort.

## The Mathematics: The Reciprocal Function

The mathematical core of date boosting in Solr is a function called `recip()`. Understanding it before looking at implementations is important, because every approach from Drupal 7 to today uses a variation of the same formula.

Solr's `recip(x, m, a, b)` computes:

```
a / (m * x + b)
```

Where `x` is the value being measured — in date boosting, the age of the document in milliseconds — and `m`, `a`, `b` are constants that control the shape of the curve.

When `a` and `b` are equal and `x >= 0`, the function produces a maximum of `1.0` when `x = 0` (content created right now) and decays toward zero as age increases. The key properties are:

- **At x = 0:** `a / (m * 0 + b) = a / b`. When `a == b`, this equals `1.0`.
- **As x → ∞:** the value approaches `0.0`, but with a configurable floor set by the relative sizes of the constants.
- **The constant `m`** controls the rate of decay. A smaller `m` means slower decay; a larger `m` means faster decay.

A typical configuration is:

```
recip(ms(NOW/HOUR, <date_field>), 3.16e-11, 1, 0.1)
```

Breaking this down:

- `ms(NOW/HOUR, <date_field>)` — calculates the number of milliseconds between the current time (rounded down to the hour) and the document's date field. Rounding to the hour is deliberate: it allows Solr to cache the result across all queries within the same hour, rather than producing a different boost value for every millisecond that passes.
- `3.16e-11` — approximately `1 / (1000ms × 60s × 60m × 24h × 365.25d)`, which is 1 divided by the number of milliseconds in a year. This means `x` is effectively measured in years when multiplied by `m`.
- `1` — the value of `a`, the numerator.
- `0.1` — the value of `b`, the floor constant. With `a = 1` and `b = 0.1`, the maximum boost is `1 / 0.1 = 10` (when `x = 0`).

You can visualise the decay curve on [Wolfram Alpha](https://www.wolframalpha.com) by plotting `1 / (3.16e-11 * x + 0.1)` against `x` to understand how the constants interact before deploying them on a live index.

## Additive vs. Multiplicative Boosting

An important distinction — one that's easy to get wrong — is whether the boost is applied additively or multiplicatively to the base relevance score.

In a [2012 analysis of Solr boost methods](https://nolanlawson.com/2012/06/02/comparing-boost-methods-in-solr/), Nolan Lawson tested five boost approaches and concluded:

> "Prefer multiplicative boosting to additive boosting"

Additive boosting (using Solr's `bf` — boost function — parameter) adds the boost value directly to the document's relevance score. If the base score varies widely between queries, the additive boost can either dominate results entirely or have negligible effect.

Multiplicative boosting (using the `boost` parameter in edismax, or `{!boost b=...}` syntax) multiplies the document's relevance score by the boost value. This preserves the relative ranking of documents by textual relevance while uniformly scaling the effect of recency across all results.

Lawson also highlights a critical mistake that trips up developers:

> "**bq takes a *query*, not a *function*.** This distinction is crucial because attempting to use `bq=log(relevancy_score)` treats 'log' and 'relevancy_score' as searchable text rather than mathematical operations."

The `recip()` function is a function, not a query. It must be used with `bf` or `boost`, not `bq`. For date boosting, `boost` (multiplicative) is the right choice.

## The History: How Implementations Have Evolved

### Drupal 7: The Apache Solr Era

The first documented approach for Drupal date boosting appeared when the Apache Solr Search Integration module was the standard. [Metal Toad described the approach](https://www.metaltoad.com/blog/date-boosting-solr-drupal-search-results) in the context of building a searchable event calendar for the L.A. Philharmonic, where the goal was to surface upcoming concerts ahead of past ones.

The implementation used a hook to add an additive boost function directly to the Solr query:

```php
function hook_search_apachesolr_query_alter(DrupalSolrQueryInterface $query) {
  $query->addParam('bf', [
    'freshness' => 'recip(abs(ms(NOW/HOUR,dm_field_date)),3.16e-11,1,.1)'
  ]);
}
```

Metal Toad noted:

> "This will apply a +10 boost to a document with today's date, tapering off to about +1 after a year."

And offered a practical note that remains true today:

> "There are no spaces in the function queries."

### Drupal 8: The Hook Adaptation

When Search API and Search API Solr replaced Apache Solr Search Integration as the standard in Drupal 8, the pattern was adapted to the new API using a procedural hook:

```php
use \Solarium\QueryType\Select\Query\Query;
use \Drupal\search_api\Query\QueryInterface;

function mymodule_search_api_solr_query_alter(Query $solarium_query, QueryInterface $query) {
  $solarium_query->addParam(
    'boost',
    'recip(abs(ms(NOW/HOUR,ds_created)),6.31e-11,1,.1)'
  );
}
```

Several things changed:

- The hook became `hook_search_api_solr_query_alter`
- The parameter switched from `bf` (additive) to `boost` (multiplicative) — the right choice per Lawson's analysis
- The field name changed to `ds_created` following Search API Solr's naming convention
- The constant `m` changed to `6.31e-11`, doubling the decay rate
- `abs()` was added around the `ms()` call to handle future dates

A [Drupal.org issue from 2016](https://www.drupal.org/node/2823538) documents the difficulties developers encountered when the module's internal architecture changed. The resolution confirmed that `hook_search_api_solr_query_alter` remained the supported path.

### Drupal 8/9: The Event Subscriber Pattern

The hook was deprecated in Search API Solr 4.2.0 and removed in 4.3.0. The module migrated to Symfony's event system, and the correct approach became an event subscriber:

```php
namespace Drupal\mymodule\EventSubscriber;

use Drupal\search_api_solr\Event\PreQueryEvent;
use Drupal\search_api_solr\Event\SearchApiSolrEvents;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class DateBoostSubscriber implements EventSubscriberInterface {

  public static function getSubscribedEvents(): array {
    return [
      SearchApiSolrEvents::PRE_QUERY => 'onPreQuery',
    ];
  }

  public function onPreQuery(PreQueryEvent $event): void {
    $event->getSolariumQuery()->addParam(
      'boost',
      'recip(abs(ms(NOW/HOUR,ds_created)),6.31e-11,1,.1)'
    );
  }

}
```

The `PRE_QUERY` event fires before the Search API query is converted into its Solr expression, making it the correct place to add parameters that supplement the generated query. The approach was functionally identical to Drupal 8 — the same `boost` parameter, the same `recip()` formula — just using current API conventions.

## How It Works Today: Built-In Solutions

### Index-Time Boosting Is Gone

A critical context: Apache Solr removed index-time document boosting in Solr 7.0 (2017). Before that, you could write a boost value into the index document itself. From Solr 7 onwards, only query-time boosting is supported.

Search API Solr acknowledges this in its backend code:

```php
// Some processors might add an absolute boost factor to the item. Since
// Solr doesn't support index time boosting anymore, we simply store that
// factor and include it in the boost calculation at query time.
$doc->setField('boost_document', $item->getBoost());
```

When a processor calls `$item->setBoost()`, the value is stored in the index at indexing time. At query time, Search API Solr incorporates it into the query. The boost becomes stale for time-sensitive content unless re-indexed regularly. For event-heavy sites where recency matters, this is a significant limitation.

### The Built-In Solution: BoostMoreRecent Processor

Search API Solr ships a built-in processor — `solr_boost_more_recent` (class `BoostMoreRecent`) — that takes the correct approach. Rather than running at index time, it runs at query time, so the boost formula is generated fresh on every search and is always accurate.

The generated formula looks like:

```
product(2.00, recip(abs(ms(NOW/HOUR, ds_page_created)), 3.16e-11, 0.100, 0.050))
```

Breaking this down:

- `product(2.00, ...)` — multiplies the recip result by the configured boost factor. A boost factor of `2.00` means today's content has its relevance score doubled.
- `recip(abs(ms(NOW/HOUR, ds_page_created)), m, a, b)` — the same reciprocal decay function.
- `abs()` — supports future dates without producing negative values.
- `NOW/HOUR` — timestamp rounded to the hour for Solr query caching.
- `ds_page_created` — the Solr field name for the Search API field.

The processor is configurable through the Search API index admin UI without writing code:

- **Boost factor** — controls the maximum multiplier applied to recent content
- **Resolution** — how finely the time difference is measured (hours, days, weeks)
- **Constants m, a, b** — the recip function parameters, with sensible defaults
- **Support future dates** — toggles the `abs()` wrapper for event start dates

### Choosing the Right Field

The Solr field name must match a date-typed field in the Search API index. Field names follow the naming convention set by Search API Solr:

| Prefix | Type |
|--------|------|
| `d` | date |
| `t` | text |
| `s` | string |
| `i` | integer |
| `f` | float |
| `b` | boolean |

With `s` for single-valued stored and `m` for multi-valued. A Search API field named `page_created` of type `date` becomes `ds_page_created` in Solr.

The correct Solr field name for any Search API field can be found in the index field listing at **Admin → Configuration → Search API → [Index] → Fields**.

## Summary: A Decade of Consistency

Date-based boosting in Drupal search has followed a consistent mathematical approach for a decade: the `recip()` function applied at query time using millisecond age as the input. What has changed is the mechanism:

| Era | Mechanism | Stage |
|-----|-----------|-------|
| Drupal 7 / Apache Solr | `hook_search_apachesolr_query_alter` | Query time |
| Drupal 8 / Search API Solr | `hook_search_api_solr_query_alter` | Query time |
| Drupal 9+ / Search API Solr 4.2+ | `PreQueryEvent` event subscriber | Query time |
| Drupal 9+/11 / Search API Solr (built-in) | `BoostMoreRecent` processor | Query time |

All query-time approaches are functionally equivalent: they add a `recip()` formula to the Solr query and let Solr evaluate it against the indexed date field on every search. The built-in `BoostMoreRecent` processor is now the preferred implementation for any site running Search API Solr, providing query-time accuracy through a UI-configurable processor without custom code.

The one approach that does not achieve the same effect is computing the boost at index time. The value becomes stale as the document ages and requires periodic re-indexing to remain meaningful — defeating the purpose of a continuous decay function.

## Resources

- [Comparing Boost Methods in Solr](https://nolanlawson.com/2012/06/02/comparing-boost-methods-in-solr/) — Nolan Lawson (2012)
- [Date-Boosting Solr/Drupal Search Results](https://www.metaltoad.com/blog/date-boosting-solr-drupal-search-results) — Metal Toad
- [Boosting by Date — Search API Solr HowTos](https://www.drupal.org/docs/8/modules/search-api-solr/search-api-solr-howtos/boosting-by-date)
- [Solr Function Query Documentation](https://solr.apache.org/guide/solr/latest/query-guide/function-queries.html)
- [Solr recip() Function](https://solr.apache.org/guide/solr/latest/query-guide/function-queries.html#recip)
