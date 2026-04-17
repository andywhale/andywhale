---
title: "Understanding drupal_block() in Twig Tweak: Using the Right Tool for the Right Job"
description: "Twig Tweak is one of the most useful contributed modules in the Drupal ecosystem. This article explains how drupal_block() works, when it's appropriate, and where the module itself points you toward better alternatives for placed blocks."
date: 2026-04-17T13:21:43+01:00
image: "images/Drupal-Block-Twig.png"
draft: false
type: "post"
tags: ["drupal", "drupal10", "drupal11", "twig", "performance", "modules", "php"]
---

Twig Tweak is one of the most useful contributed modules in the Drupal ecosystem. It brings a rich set of functions and filters to Twig templates - rendering entities, views, menus, regions, fields, and more - that would otherwise require custom theme or preprocess code. For many day-to-day theming tasks it is indispensable.

One of its most reached-for functions is drupal_block(). It is convenient, easy to understand, and works exactly as described. The performance issues this article covers are not a flaw in the module - they arise when drupal_block() is used in situations it was not designed for. Understanding what it actually does, and where the module itself points you toward better alternatives, makes it straightforward to use correctly.

## Background: Two Ways Drupal Renders Blocks

Drupal has two distinct pipelines for rendering a block plugin's output.

### Path 1: The core block entity system

When a block is placed through the Drupal admin UI (/admin/structure/block), a block config entity is created. This entity stores the plugin ID, plugin configuration, region, theme, weight, and visibility conditions.

At render time, core/modules/block/src/BlockViewBuilder.php turns that entity into a render array. The critical section is at lines 79-81:

```php
$build[$entity_id] += [
    '#lazy_builder' => [static::class . '::lazyBuilder', [$entity_id, $view_mode,
$langcode]],
];
```

The block plugin's build() method is not called here. A #lazy_builder callback reference is stored instead. Drupal's renderer resolves this callback only after checking the render cache:

```
Renderer encounters element with #cache['keys'] + #lazy_builder
  -> checks render cache
    -> CACHE HIT:  return cached HTML - lazyBuilder() never invoked, build() never called
    -> CACHE MISS: invoke lazyBuilder() -> call build() -> render -> store in cache
```

The block plugin's build() method - where all the expensive work happens - only runs on a cache miss.

### Path 2: twig_tweak's drupal_block()

drupal_block() is designed to render a block plugin directly, without requiring a block config entity to exist. This is genuinely useful - it lets you drop a block plugin's output into a template without going through the admin UI. The trade-off is that it renders the plugin immediately.

When you call `{{ drupal_block('menu_block:main', {level: 1, depth: 3}, false) }}`, twig_tweak's BlockViewBuilder::build() runs straight away. The relevant line is twig_tweak/src/View/BlockViewBuilder.php:138:

```php
$build['content'] = $block_plugin->build();
```

There is no #lazy_builder. The plugin's build() method is called eagerly and unconditionally - before any cache check takes place. Cache metadata is then applied and #cache['keys'] is set at lines 169-181, so the rendered HTML output will still be cached. But by the time the renderer consults the cache, build() has already run.

```
Twig calls drupal_block()
  -> $block_plugin->build() called IMMEDIATELY (DB queries run here)
  -> render array returned with #cache['keys']
  -> Renderer checks render cache
    -> CACHE HIT:  return cached HTML - but build() already ran, too late
    -> CACHE MISS: render and store in cache
```

This is the correct and expected behaviour for what drupal_block() was designed to do. The issue arises only when it is used for blocks that are already placed through the admin UI and have expensive build() methods - a situation the module itself has better tools for.

## Why This Matters for Menus

The menu_block plugin's build() involves loading the menu tree from the database, evaluating user access on every link, resolving the active trail for the current route, and applying depth and level constraints. On a menu with hundreds of items, this is significant.

Importantly, menu_block->build() returns a straightforward render array - #theme => 'menu' with the pre-loaded tree data inline. There are no nested #lazy_builder references inside it. All the expensive work happens synchronously inside build(). There is nothing deferred to recover.

### Drupal's caching hierarchy

To understand the real-world impact, it helps to think about Drupal's caching layers in order:

1. **CDN** - the request never reaches Drupal
2. **Page Cache** - the full page response is served from cache (anonymous users only)
3. **Dynamic Page Cache** - the page skeleton is cached with placeholders for personalised regions
4. **Render Cache** - individual block output is cached by #cache['keys']

A CDN and Drupal's page cache do provide meaningful protection. For anonymous traffic on a well-configured site, most requests never reach the render layer at all. In that scenario the impact of using drupal_block() for a menu is largely masked.

The problem surfaces at layer 4 - the render cache - which is meant to be the safety net for everything the page cache does not cover. With a placed block and #lazy_builder, a render cache hit means build() is never called: the menu queries never run. With drupal_block(), build() has already run before the render cache is consulted, so even a warm render cache provides no protection against the menu tree queries.

### Where the cost is actually felt

The impact is most significant in three situations:

**Authenticated users.** Logged-in users bypass the page cache entirely. Every page they request hits the render cache layer, where the absence of a lazy builder means the menu is rebuilt on every page render regardless of how warm the render cache is.

**High-variance pages.** Pages with many cache context variations - multiple languages, query string parameters, personalised content - produce many distinct page cache entries. Each variant is cold on first hit, and each cold hit unnecessarily rebuilds the menu tree before the render cache is consulted.

**Cache warming after a deploy or full cache clear.** When caches are cleared, the page cache and render cache are both empty. With #lazy_builder, the menu tree is built once at the render cache level and all subsequent page cache misses benefit from that warm render cache entry. With drupal_block(), every concurrent request during the warming period rebuilds the menu tree independently.

## A Note on Nested Lazy Builders

Some block plugins return render arrays that contain their own #lazy_builder references internally. Views blocks are the main example. Calling $block_plugin->build() directly on these plugins does not collapse or lose those nested lazy builders - the returned render array still contains them and the renderer resolves them normally.

The eager build() call in drupal_block() is therefore specifically a concern for plugins whose own build() is expensive to run - not for plugins that defer work internally via their own lazy builders.

## The Cache Metadata Question

A separate concern - documented in twig_tweak issue queue entries #3045120 and #3176561 - is whether cache metadata (contexts, tags, max-age) from the block plugin is correctly collected and bubbled up.

In twig_tweak 3.x, this is handled correctly. BlockViewBuilder.php:169-172 merges metadata from the access result and the block plugin:

```php
CacheableMetadata::createFromRenderArray($build)
    ->addCacheableDependency($access)
    ->addCacheableDependency($block_plugin)
    ->applyTo($build);
```

This is worth noting because earlier versions had known issues here. In a current install the metadata propagation works as expected. Two things are still absent compared to the entity pipeline, but these are inherent to the function's design rather than bugs:

1. **Block config entity cache tags.** When a block is placed via the entity system, the entity's own cache tags (config:block.block.ENTITY_ID) are included. With drupal_block(), no block config entity is loaded, so its cache tag is never present. If an admin changes the block's configuration, the cached output from drupal_block() will not be invalidated.

2. **Hook invocations.** Core's block view builder invokes hook_block_view_alter and hook_block_view_BASE_BLOCK_ID_alter. The drupal_block() path does not, since it bypasses the entity view builder entirely.

## What Block Settings Are Lost

The blocks.md documentation shipped with the twig_tweak module notes:

> "Additionally, they reference theme and region where a block should be printed, but this data are not used when rendering through Twig Tweak."

This is by design. When configuration is passed to drupal_block(), the plugin is instantiated with only those keys plus the label_display default. Any additional settings an admin may have configured on the block entity - expand all items, follow active trail, follow parent, custom context mappings - are absent, since the block config entity is never loaded. For use cases where those settings matter, the entity-based approach is the right one.

## Twig Tweak's Own Guidance

The twig_tweak documentation distinguishes three things that Drupal calls "blocks" and gives clear guidance for each. Understanding this distinction is key to using the module well.

**Block plugin** - a PHP class. drupal_block('plugin_id') renders it directly, bypassing the entity system. Appropriate when you want a plugin's output without a placed entity.

**Block configuration entity** - what you configure at /admin/structure/block. The module's own documentation recommends drupal_entity('block', 'machine_name') for this case, which loads the entity through the entity view builder - the same pipeline that sets up #lazy_builder.

**Block content entity** - a custom block content item. Use drupal_entity('block_content', id).

The module is well-documented on this point. drupal_block() is a raw plugin renderer; drupal_entity('block', ...) is the correct function for placed blocks. Reaching for drupal_block() out of habit for a block that exists in the layout is simply using the wrong function for the job.

## When drupal_block() Is the Right Choice

drupal_block() is the right tool in these situations:

- Prototyping and local development, where you want to render a block plugin quickly without creating a config entity.
- Blocks with no expensive build() logic - static markup, simple form renders, or blocks whose output is trivial to produce.
- Blocks that should not be configured in the admin UI, where you intentionally want plugin defaults or specific configuration passed from the template.
- One-off renders on low-traffic pages where the block is inherently uncached or the cost is negligible.

## When to Reach for a Different Function

Prefer drupal_entity('block', ...) or elements.block_id when:

- The block is already placed through the admin UI. The entity pipeline gives you lazy building, entity cache tags, hook invocations, and admin-configured settings. There is no reason to bypass all of that.
- The block has an expensive build() method - menu blocks, views blocks with complex queries, anything doing significant database work.
- You need cache invalidation when block config changes, since entity cache tags are only present on the entity-based path.

## The Correct Alternatives

### Use the region render array (elements)

If the block is placed in a region via the admin UI, the region template's elements variable already contains pre-assembled, lazy-builder-backed render arrays for every block in that region:

```twig
{{ elements.my_block_machine_name }}
```

This is the most direct approach and requires no twig_tweak involvement at all.

### Use drupal_entity('block', 'entity_id')

For a placed block that you want to render outside its configured region, drupal_entity('block', 'machine_name') is the twig_tweak-idiomatic approach. It loads the config entity through the entity view builder, sets up #lazy_builder correctly, and is what the module's own documentation recommends:

```twig
{{ drupal_entity('block', 'olivero_mainmenu') }}
```

### Use drupal_region()

To render an entire region's blocks with their full lazy builder pipeline:

```twig
{{ drupal_region('header') }}
```

## Could drupal_block() Be Improved?

It is worth noting that the performance gap is theoretically closeable. BlockViewBuilder::build() could be refactored to defer $block_plugin->build() into a static trusted callback, returning a render array with #lazy_builder and cache metadata populated from the plugin's declared contexts and tags - mirroring the pattern in core's block entity view builder. This would mean build() only runs on a render cache miss, eliminating the unnecessary rebuilds that currently affect all authenticated users and every cold cache entry.

That said, for any block placed through the admin UI, drupal_entity('block', ...) already solves the problem cleanly. A patch to drupal_block() would primarily benefit the narrower case of rendering unconfigured plugins where performance still matters.

## Summary

Twig Tweak is a well-designed module with good internal documentation. drupal_block() does exactly what it says: renders a block plugin directly. The performance considerations in this article are not criticisms of the module - they are a consequence of using it outside the use case it was designed for.

| | drupal_block() | drupal_entity('block', ...) | elements.block_id |
|---|---|---|---|
| build() called on every uncached page | Yes | No - only on render cache miss | No - only on render cache miss |
| Protected by page cache (anonymous) | Yes | Yes | Yes |
| Protected by render cache (authenticated) | No | Yes | Yes |
| Block config entity cache tags | No | Yes | Yes |
| hook_block_view_alter invoked | No | Yes | Yes |
| Admin-configured plugin settings used | No | Yes | Yes |
| Suitable for menus / header nav | No | Yes | Yes |

For block plugins without a placed entity, drupal_block() is appropriate and convenient. For anything placed through the Drupal block layout UI - especially performance-sensitive blocks like navigation menus - use drupal_entity('block', ...) or reference the block through the region's elements variable. A CDN and page cache will mask the problem for anonymous traffic, but authenticated users and every cold cache entry will pay the full cost of an unnecessary menu rebuild on each request.

## Resources

- [Twig Tweak Module](https://drupal.org/project/twig_tweak)
- [Drupal Block API Documentation](https://www.drupal.org/docs/drupal-apis/block-api)
- [Render Cache Documentation](https://www.drupal.org/docs/drupal-apis/render-api/cacheability-of-render-arrays)
