---
title: "Cloudflare's Instant Purge for All: Cache Tag Purging Changes Everything"
description: "Cloudflare's instant purge with cache tag support is now available on every plan. What changed, why it matters for Drupal, and what this means for your caching strategy."
date: 2025-04-25T13:21:43+01:00
image: "images/drupal-performance-gauge.svg"
draft: false
type: "post"
tags: ["drupal", "cloudflare", "caching", "cdn", "performance"]
---

For years, Cloudflare's cache tag purging was locked behind enterprise pricing. If you wanted to purge cache by tags (invalidate all pages containing a specific tag rather than just one URL or a URL prefix), you paid enterprise rates.

This week, Cloudflare made that feature available to every plan. This is significant enough that sites using Cloudflare should be reconsidering their caching strategy.

## What Is Instant Purge?

Instant Purge is Cloudflare's rebuilt cache invalidation system designed to achieve sub-150 millisecond purge times across their global network of 335+ data centres. When you purge content, fresh versions reach your origin within milliseconds, rather than waiting for cache entries to naturally expire.

The key innovation: instead of lazy purging (marking entries for deletion), Cloudflare now uses an active, immediate purge via their RocksDB-based CacheDB service. This reduces storage requirements by 10x and makes purging genuinely instant.

## What Changed (And Why It Matters)

Previously, you could use instant purge via Cloudflare's API, but only enterprise customers could purge by tag, prefix, or hostname. Everyone else was limited to purging entire URLs or purging everything.

As of April 2025, **all five purge methods are now available on every plan:**

1. **Purge Everything** - Clear the entire CDN cache
2. **Purge by URL** - Clear specific URLs
3. **Purge by Prefix** - Clear all URLs matching a path prefix
4. **Purge by Hostname** - Clear all URLs on a specific domain
5. **Purge by Tag** - Clear all URLs tagged with a specific tag

Rate limits vary by plan:
- **Free**: 5 requests/min, 500 keys/min
- **Pro**: 5 requests/sec, 500 keys/sec
- **Business**: 10 requests/sec, 1,000 keys/sec
- **Enterprise**: 50 requests/sec, 5,000 keys/sec

That tag-based purging at the Free plan level is the game changer.

## Why Tag-Based Purging Matters for Drupal

Drupal sites use cache tags to build a cache dependency graph. When you update a node, you invalidate tags like `node:123`, which cascades to invalidate every page that displays that node. It's sophisticated and efficient.

But Cloudflare's CDN couldn't speak this language. Cloudflare cached HTML, but it had no concept of tags. So when a Drupal site updated a node, the site could invalidate its internal render cache by tags, but couldn't tell the CDN: "clear all pages containing this node."

The workarounds were painful:

**Option 1: Short Cache Times** - Set HTML cache to 30 seconds or 1 minute, so stale content expires quickly. This wastes cache potential and increases origin load.

**Option 2: Session-Based Caching Rules** - Cache anonymous users aggressively, but bypass cache for authenticated users. Authenticated users get no CDN benefit.

**Option 3: Skip Cloudflare Caching** - Disable HTML caching on Cloudflare, use it only for static assets and WAF. You lose the CDN benefit for HTML entirely.

With tag-based purging now available, **you can now purge Cloudflare cache by tag programmatically**. When Drupal invalidates a cache tag, you can immediately invalidate the CDN cache with the same tag.

This is transformative.

## The Strategy That Just Became Possible

Imagine this workflow:

1. Set Cloudflare to cache HTML aggressively (1 hour, or even longer)
2. On every content change, Drupal invalidates its own caches by tag
3. Drupal's Cloudflare integration also purges the CDN by the same tag
4. Both your origin cache and your CDN cache stay in sync

The result:
- Authenticated users get CDN benefits (previously impossible)
- Content updates propagate instantly (no stale HTML on the CDN)
- Cache times can be long (no risk of stale content lingering)
- You can serve authenticated users from cache (higher scalability)

This is what Fastly users have had for years. Now it's available on Cloudflare.

## The Comparison to Fastly

On Fastly, you can purge by tag, so you set long cache times and invalidate surgically. Drupal sites on Fastly leverage this heavily. The architecture is:

- Cache everything for a long time (30 mins, 1 hour, sometimes more)
- Purge by tag on content changes
- Authenticated users get the same cache benefit as anonymous users

Cloudflare previously forced you to choose: cache aggressively (and serve stale content to authenticated users), or bypass cache for authenticated users (and lose scalability).

Now, Cloudflare users can adopt the Fastly approach.

## How to Implement It: Purge + Cloudflare Integration

Tag-based purging requires integration between Drupal's cache invalidation system and Cloudflare's API. This is where the [Purge module](https://www.drupal.org/project/purge) and [Cloudflare module](https://www.drupal.org/project/cloudflare) come in.

The architecture works like this:

**Drupal's automatic cache tagging (built-in):**
When Drupal renders a page containing a node, it automatically applies cache tags to the rendered output through the render system's cache tag bubbling. If a page displays node:123, Drupal automatically tags that page with `node:123`. You don't manually tag anything - it's automatic.

**The Purge module** acts as middleware that catches Drupal's cache invalidations and queues them for external systems. When a node is updated, Drupal invalidates the `node:123` tag internally. The Purge module intercepts this and adds it to a purge queue.

**The Cloudflare purger** takes queued invalidations and sends them to Cloudflare's API. This is handled by the [Cloudflare module](https://www.drupal.org/project/cloudflare) in conjunction with Purge.

Together, they create a transparent pipeline:

```
Node updated
  → Drupal invalidates node:123 cache tag
    → Purge module queues the invalidation
      → Cloudflare purger sends tag:node:123 to Cloudflare API
        → Cloudflare purges all cached pages tagged with node:123
```

**Additional considerations:**

- **Query string handling:** If your site uses query parameters for tracking (UTM codes, Facebook pixel IDs), use [page_cache_query_ignore](https://www.drupal.org/project/page_cache_query_ignore) to ignore those parameters when caching. This prevents cache bloat from the same page being cached multiple times with different tracking parameters.

- **Rate limiting:** The Purge module handles Cloudflare's rate limits (5 requests/second on Pro, 10 on Business, 50 on Enterprise) through its queueing system. It won't overwhelm the API.

- **Async processing:** Purge uses Drupal's queue system, so invalidations are processed asynchronously. This means cache purges happen in the background, not blocking content saves.

## What You Need to Do

If you're using Cloudflare, here's the practical implementation path:

1. **Install the Purge module** - [Purge](https://www.drupal.org/project/purge) is the foundation. It catches Drupal's cache invalidations and manages the queue.

2. **Install the Cloudflare module** - The [Cloudflare module](https://www.drupal.org/project/cloudflare) provides the purger that connects Purge to Cloudflare's API.

3. **Configure with API credentials** - Add your Cloudflare API token and Zone ID to the Cloudflare module configuration.

4. **Enable the Purge UI** - Purge comes with an optional admin interface that lets you monitor the invalidation queue and debug issues.

5. **(Optional) Install page_cache_query_ignore** - If your site uses tracking parameters (UTM codes, etc.), install [page_cache_query_ignore](https://www.drupal.org/project/page_cache_query_ignore) to prevent cache bloat from the same page being cached multiple times with different query strings.

6. **Audit your current cache configuration** - Are you using short cache times because tag purging wasn't available? Now you can extend those times. Start conservative (30 minutes), monitor for issues, then increase.

7. **Test on staging thoroughly** - Longer cache times mean stale content is more visible if something goes wrong. Verify:
   - When you update a node, it purges from Cloudflare
   - When you update a taxonomy term, pages using that term purge
   - Featured content blocks update when their source content changes

8. **Monitor the Purge queue** - After launch, watch the Purge queue dashboard. If it's backing up, you're trying to purge faster than Cloudflare's API allows. Adjust cache times or investigate why so many invalidations are happening.

9. **Monitor your origin** - With longer cache times, your origin will serve fewer requests. This is good, but ensure your monitoring doesn't misinterpret reduced load as a problem. You might suddenly have over-provisioned infrastructure.

## The Bigger Picture

This change represents Cloudflare closing a significant gap. For years, Fastly's tag-based purging was a structural advantage. Drupal sites on Fastly could be more aggressive with cache times and more precise with invalidation. Cloudflare sites had to choose between performance and freshness.

That gap is closing. Cloudflare's purge architecture is now more sophisticated, and the feature is democratised (available on all plans, not just enterprise).

This changes the economics of CDN selection for Drupal sites. If you've been on Cloudflare and stuck with short cache times because tag purging wasn't available, that constraint just vanished.

## Resources

- [Cloudflare Blog: Instant Purge for All](https://blog.cloudflare.com/instant-purge-for-all/)
- **Drupal Modules:**
  - [Purge Module](https://www.drupal.org/project/purge) - Cache invalidation middleware
  - [Cloudflare Module](https://www.drupal.org/project/cloudflare) - Cloudflare API integration and purger
  - [Page Cache Query Ignore Module](https://www.drupal.org/project/page_cache_query_ignore) - Ignore tracking parameters in cache keys
- [Cloudflare Cache Purge API Documentation](https://developers.cloudflare.com/api/operations/cache-purge)
- [Drupal Cache Tags Documentation](https://www.drupal.org/docs/develop/drupal-apis/cache-api/cache-tags)
- [Drupal Cache API Guide](https://www.drupal.org/docs/develop/drupal-apis/cache-api)
- [Fastly Purging Documentation](https://docs.fastly.com/en/guides/purging)
