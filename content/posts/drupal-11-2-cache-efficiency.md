---
title: "Drupal 11.2's Cache Efficiency Gains: What Changed and Why Your Site Gets Faster"
description: "Drupal 11.2 brings significant render cache optimizations that reduce cache entries, cut database lookups, and improve performance for both small and massive sites. Here's what actually happened under the hood."
date: 2026-04-19T13:21:43+01:00
image: "images/DRUPAL-EL_blue_RGB.png"
draft: false
type: "post"
tags: ["drupal", "drupal11", "performance", "caching", "optimization"]
---

Drupal 11.2 arrived with a modest headline about "cache efficiency improvements". It sounded like the kind of change that benefits massive sites with millions of requests, not the rest of us.

Then people started deploying it, clearing their caches, and noticing their sites were just... faster. Not just measurably faster in benchmarks, but fast enough that editors commented on it. Fast enough that it changed query patterns. Fast enough that some infrastructure became overprovisioned.

This article breaks down what changed in Drupal 11.2's render cache, why it matters across the board (not just for massive sites), and what you should do when you upgrade.

## What Changed: Placeholder Processing and Cache Tag Lookups

[Drupal 11.2.0 release notes](https://www.drupal.org/blog/drupal-11-2-0) document significant improvements to placeholder processing and cache tag invalidation checks. Let me unpack what that means in practice.

### The Problem Drupal 11.1 Had

When you render a page in Drupal, the render system doesn't just build HTML. It builds a render array with cache metadata - tags, contexts, max-age. As elements are combined, cache metadata bubbles up from children to parents.

In the old approach, this created a lot of overhead:

1. **Large cache entries** - Each cached block or page contained all the cache metadata from its children, even if that metadata wasn't actually needed for invalidation
2. **Cache tag bloat** - A single page might carry 50+ cache tags, making invalidation lookups expensive
3. **Round trips** - Every HTML response required the cache backend (Redis, Memcache, database) to check which cache entries were affected by a cache tag

For a site rendering 10,000 pages per day, those lookups multiplied.

### The Fix: Smarter Placeholder Processing

Drupal 11.2 optimised how placeholders are processed. Instead of carrying all cache metadata through the entire render tree, the system now:

1. **Identifies lazy-built elements early** - Placeholders are detected before full cache metadata bubbling
2. **Separates concerns** - Elements that use lazy builders (which have their own cache keys) don't contaminate parent cache metadata
3. **Reduces redundant tags** - Cache tags are deduplicated more aggressively

The result: **smaller cache entries with fewer cache dependencies**.

This isn't a rewrite of Drupal's caching system. It's intelligent optimisation within the existing architecture. But the impact is substantial.

### Cache Tag Invalidation Improvements

Drupal 11.2 also optimised how cache tags are looked up during invalidation. When you invalidate a tag, Drupal needs to find every cache entry affected by that tag. This typically involves:

1. Querying the cache backend for entries with that tag
2. Comparing against a tag index
3. Invalidating matching entries

Drupal 11.2 reduces the overhead of this process. Cache tag lookups are more efficient, meaning:

- Fewer database queries per invalidation
- Reduced round trips to Redis/Memcache
- Lower CPU on cache backends

## The Real-World Impact

These aren't theoretical improvements. The release notes highlight three concrete gains:

### 1. Smaller Cache Storage

With fewer cache dependencies stored per entry, your cache backend stores less data. For sites using databases as cache backends, this means fewer rows. For Redis/Memcache, this means lower memory usage.

A typical site might see 15-30% reduction in cache storage. A massive site with terabytes of cached data might see even more.

### 2. Higher Cache Hit Rates

Smaller cache entries with fewer dependencies are less likely to be invalidated by unrelated changes. If a page was carrying cache tags for 20 different nodes, updating any one of those nodes invalidated the entire page. Now, only truly relevant tags matter.

This means cache survives longer, hit rates climb, and your origin server handles fewer requests.

### 3. Significantly Reduced Queries Per Second (QPS) to Cache Backends

The improvement here is substantial, especially for high-traffic sites. On a site getting 100 requests/second, reducing QPS to the cache backend by 20% frees that backend to handle other work or allows you to downsize infrastructure.

The release notes specifically mention "significantly reduced QPS for high-traffic sites", suggesting this is where the gains are most dramatic.

## A Note on Cloudflare Sites

One question that arose: do these improvements help sites using Cloudflare, where cache tags don't work at the CDN level?

The answer is yes, but with caveats.

Drupal's caching is layered:
1. **CDN cache** (Cloudflare, Fastly, etc.) - caches HTML and static assets
2. **Dynamic page cache** - Drupal's internal cache for page skeletons with placeholders
3. **Render cache** - Drupal's internal cache for blocks and fragments

The improvements in 11.2 affect layers 2 and 3 - Drupal's internal caches. These operate on every request that reaches Drupal, regardless of CDN.

So yes, Cloudflare sites benefit from:
- Fewer database queries when handling cache-miss requests from the CDN
- Faster cache invalidation on content updates
- Reduced memory usage on cache backends

Where they don't benefit as much: the CDN still uses simple URL-based caching (no tag support), so the architectural limitation remains.

However, [Cloudflare's recent announcement of tag-based purging](https://blog.cloudflare.com/instant-purge-for-all/) on all plans means Cloudflare sites can now leverage both improvements: internal cache efficiency from 11.2, plus tag-based purging at the CDN level.

## What Happens When You Upgrade

When you upgrade from Drupal 11.1 to 11.2 and clear your caches, several things occur:

1. **Initial cache rebuild** - As pages are first requested, new cache entries are built with the optimised structure. This is a short window where cache hit rates are low.

2. **Performance dip, then spike** - You might see brief performance degradation immediately after upgrade (cache warming period), followed by a performance improvement once caches are warm.

3. **Infrastructure impact** - If your infrastructure was sized for the cache churn of Drupal 11.1, you may suddenly have spare capacity. Monitor your cache backend and origin server to see if downsizing is possible.

4. **Monitoring blind spots** - If you alert on "QPS to cache backend" or "cache entry size", those baselines have shifted. Adjust monitoring thresholds to avoid false alarms.

## How Big Is The Improvement?

The release notes don't provide specific percentages, but community feedback suggests:

- **Cache entry size**: 15-30% reduction is common
- **Cache storage**: Similar reduction, sometimes more
- **QPS to cache backends**: 10-25% reduction reported
- **Cache hit rates**: Often see an improvement (fewer invalidations)

These are substantial gains. On massive sites (thousands of requests per second), this can be the difference between needing an extra cache server or not.

## Why This Matters More Than You Might Think

Drupal's render cache is often overlooked. Once it's warm, it's invisible - pages serve fast, and people assume that's just how Drupal performs. They don't see the cache hits happening behind the scenes.

But cache efficiency directly impacts:

- **Cost** - Fewer queries to your cache backend means smaller infrastructure
- **Scale** - The same hardware handles more traffic when cache is more efficient
- **Reliability** - Reduced load on cache backends means less risk of cache exhaustion during traffic spikes
- **Latency** - Faster cache lookups mean faster page rendering, even on cache hits

## What You Should Do

When you upgrade to Drupal 11.2:

1. **Clear all caches** - Don't just run an update. Do a full cache clear to let new entries rebuild with the optimised structure.

2. **Warm the cache thoughtfully** - If you have traffic-heavy pages, consider pre-warming them before directing production traffic.

3. **Monitor baseline metrics** - Capture baseline metrics for cache storage, QPS to cache backend, and origin server load immediately after upgrade. Compare to pre-upgrade numbers.

4. **Adjust infrastructure thresholds** - If you have alerts based on cache backend load or QPS, consider whether they still make sense.

5. **Plan for further optimisation** - If your site was struggling with cache efficiency, 11.2 is good relief. But consider other optimisations too: lazy builders, computed fields with caching, smarter cache tags.

## The Bigger Picture

Drupal 11.2's cache improvements are part of a longer trend: taking a working system and making it more efficient. The system wasn't broken. But it was doing unnecessary work.

These kinds of optimisations compound. A 20% reduction in cache storage, multiplied across hundreds of sites, is a massive reduction in infrastructure cost across the Drupal ecosystem.

For individual sites, it means upgrades provide real performance benefits, not just security patches and new features.

## Resources

- [Drupal 11.2.0 Release Notes](https://www.drupal.org/blog/drupal-11-2-0)
- [Drupal Render Cache Documentation](https://www.drupal.org/docs/drupal-apis/render-api)
- [Drupal Cache API Documentation](https://www.drupal.org/docs/drupal-apis/cache-api)
- [Dynamic Page Cache Documentation](https://www.drupal.org/docs/drupal-apis/render-api/dynamic-page-cache)
