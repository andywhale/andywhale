---
title: "The Hidden Cost of Menu Links: Why Your Drupal Site Cache Invalidates on Every Page Save"
description: "Menu link access checks create unexpected cache invalidations on larger sites. Here's what's happening, why it matters, and what we can do about it."
date: 2026-04-15T13:21:43+01:00
image: "images/DRUPAL-EL_blue_RGB.png"
draft: false
type: "post"
tags: ["drupal", "performance", "caching", "architecture"]
---

If you're running a large Drupal site with extensive menu structures, you might be experiencing a silent performance killer: every time a content editor saves a page, your entire menu cache gets invalidated. On sites with thousands of pages and menus rendered on every page, this compounds into a catastrophic number of cache clears.

This isn't a bug, exactly. It's the result of a perfectly reasonable architectural decision that has some unexpected consequences at scale.

## The Problem: Menu Links and Access Checks

Here's what's happening under the hood. When a node is saved, Drupal's menu UI module re-saves all associated menu links - even if nothing about those links has actually changed. This would be fine if it were just a database write, but here's the catch: **menu links have access checks**.

Menu links don't just display URLs. They display *accessible* URLs. If a user isn't allowed to view a node, that node shouldn't appear in their menu. So when you update a node - even to change just the title - Drupal needs to revalidate whether that node is still accessible. This means the menu cache depends on the access status of every node it links to.

If you have a main navigation menu with 50 links, and each of those links depends on node access checks, that's 50 separate cache dependencies. Update one node, and the entire menu cache invalidates.

On a large site with instant cache purging enabled (like those using Varnish or Fastly), this means:
- Editor saves a page
- Menu cache invalidates
- Page cache (which includes the menu) invalidates everywhere
- CDN cache clears for hundreds or thousands of pages
- Every anonymous visitor hits origin

It's a cascade that starts with a single save.

## Why This Matters More Than You'd Think

For a university site with 10,000+ content pages and menus on every page, this means that each content save potentially triggers a full site cache invalidation at the CDN level. On a site where editors are regularly publishing content, this can result in:

- Constant cache misses
- Origin server under pressure
- Poor performance for end users during peak editorial hours
- Increased infrastructure costs

The impact is invisible until you inspect the cache tags on your rendered menu output using [Drupal's render cache debug output](https://drupal.org/node/3162480). Then you see it: dozens of `node:X` cache tags, meaning the entire menu depends on the access state of every linked node.

## The Architectural Tension

Here's what makes this tricky: the current behaviour is, in a sense, *correct*. Consider these scenarios:

- A node is unpublished (should be removed from menus for anonymous users)
- A node's access restrictions change due to group or term-based access control
- A node's moderation state changes (draft → published)
- A user's role changes, affecting what they can see

In all of these cases, you *need* the menu cache to invalidate because the available menu items have changed for certain users. The system is conservative: it assumes that any node update might affect access, so it invalidates the menu to be safe.

But here's the thing: if you're running most of your site for anonymous users (which is typical for public websites), you can make specific assumptions that simplify this problem.

## The Discovery

When debugging unexpected cache clears on larger sites, we discovered that menu caches were being invalidated on every content page save - even when no menu changes were made. This seemed to date back to a 2017 issue that had been partially resolved, but with a workaround that wasn't sufficient.

We opened [Drupal issue #3486604](https://drupal.org/project/drupal/issues/3486604) to document the problem and started a discussion on the Drupal Slack with berdir, who had also been investigating similar issues. That conversation revealed the deeper architectural challenge: menu links have access checks, so their visibility depends on whether the linked node is accessible. Any change to a node - even a title update - could potentially affect access for certain users, so the system conservatively invalidates the menu cache to be safe.

Berdir's analysis clarified that this was part of a larger issue ([#3485030](https://drupal.org/project/drupal/issues/3485030)) that was being worked on at the core level. The two issues were related, and #3486604 was eventually marked as a duplicate. But the investigation provided valuable real-world context about the performance impact on large sites, which helped drive the fix forward.

The real work to resolve this fell to berdir and the core team, who worked on implementing smarter cache invalidation logic that only invalidates the menu when the node's access status actually changes - not on every save.

## The Solution: Fix It in Core

The real fix needs to happen in Drupal core. **MenuTreeStorage** shouldn't invalidate cache tags if the menu structure didn't actually change ([Issue #3485030](https://drupal.org/project/drupal/issues/3485030)).

There's also a related issue in the Menu Link Weight module ([Issue #3410674](https://drupal.org/project/menu_link_weight/issues/3410674)) that should be addressed together.

The approach is to only invalidate when the node's *access status* actually changes. This requires:

- Checking the node's view access on save
- Comparing it to the previous revision's access
- Only invalidating the menu cache if they differ

For anonymous users accessing published/unpublished content, this is straightforward and safe. Here's what the implementation looks like:

```php
function menu_uncache_node_presave(NodeInterface $node) {
  if ($node->isDefaultRevision()) {
    $links = \Drupal::service('plugin.manager.menu.link')
      ->loadLinksByRoute('entity.node.canonical', ['node' => $node->id()]);
    if (!empty($links)) {
      $node->previous_default_anonymous_revision_access = $node->access('view', User::getAnonymousUser());
    }
  }
}

function menu_uncache_node_update(NodeInterface $node) {
  if ($node->isDefaultRevision()) {
    $links = \Drupal::service('plugin.manager.menu.link')
      ->loadLinksByRoute('entity.node.canonical', ['node' => $node->id()]);
    if (!empty($links)) {
      if ($node->previous_default_anonymous_revision_access !== $node->access('view', User::getAnonymousUser())) {
        // Only invalidate if access actually changed
        $cache_tags = Cache::buildTags('menu_published', array_keys($affected_menus), ':');
        \Drupal::service('cache_tags.invalidator')->invalidateTags($cache_tags);
      }
    }
  }
}
```

This approach is being tested and discussed on the core issue. If you're experiencing this problem on your sites, the best way to help is to:

1. **Review the patch** - Test it on your staging environment
2. **Comment on the issue** - Share your real-world performance impact
3. **Help debug edge cases** - Are there access control scenarios this doesn't handle?
4. **Vote for the issue** - Show the core team this matters

Getting this merged in core is better than every large site having to implement their own workaround.

## The Fundamental Challenge

The real issue is that Drupal doesn't know what access control logic applies to your site. For one site, access is determined solely by publish status. For another, it's based on group membership or term tagging. For another, it's role-based.

A general solution that handles all scenarios would either need to:
- Be overly conservative (current behaviour - always invalidate)
- Require explicit configuration (opt-in to more aggressive caching assumptions)
- Make assumptions that aren't universally true

## How You Can Help

These issues are open and need community input to move forward. If you're experiencing this problem on your sites:

1. **Review the patches** - Test them on your staging environment and report back
2. **Comment on the issues** - Share your real-world impact and performance metrics
3. **Help identify edge cases** - Are there access control scenarios the solution doesn't handle?
4. **Vote for the issues** - Show the Drupal core team this matters for production sites
5. **Test across versions** - Report if the solution works across different Drupal versions and contributed modules

The more feedback from large production sites, the more likely the core team will prioritise merging these fixes.

**Resources:**
- [Drupal Issue #3486604: Menu caches cleared on save even with no changes](https://drupal.org/project/drupal/issues/3486604) - The issue discovered during initial investigation (closed as duplicate)
- [Drupal Issue #3485030: MenuTreeStorage shouldn't invalidate cache tags if menu didn't change](https://drupal.org/project/drupal/issues/3485030) - The primary core issue being resolved
- [Menu Link Weight Issue #3410674: Don't re-save menu links that haven't changed](https://drupal.org/project/menu_link_weight/issues/3410674)
- [Render Cache Debug Output](https://drupal.org/node/3162480)
