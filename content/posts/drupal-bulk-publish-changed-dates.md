---
title: "Bulk Publishing in Drupal: Why Your Content Dates Are Wrong and How to Fix It"
description: "The setSyncing() function prevents timestamp updates during bulk operations. Here's what's happening, why it matters for content visibility, and the solutions available."
date: 2026-02-16T13:21:43+01:00
image: "images/DRUPAL-EL_blue_RGB.png"
draft: false
type: "post"
tags: ["drupal", "drupal11", "content-moderation", "bulk-operations", "modules"]
---

If you've bulk published content on an older version of moderated_content_bulk_publish and noticed it appearing in the admin list at the wrong date, you encountered a quirk that sat somewhere between bug and feature. The content was published today, but it was sorted by a date from weeks or months ago. Your content list looked nonsensical.

This issue has been resolved in recent versions of the module, but it's worth understanding what was happening, why it occurred, and how it relates to Drupal's synchronisation system.

## The Problem: setSyncing() and Lost Timestamps

At the heart of this issue is `setSyncing(TRUE)` - a function that tells Drupal to skip certain updates when saving an entity. Specifically, it prevents the `changed` timestamp from being updated.

Here's the scenario:

1. An editor creates a page on 13th January and saves it
2. They move it to draft status
3. Weeks later, they bulk publish it along with dozens of other pieces of content
4. The admin content list still shows the page at the 13th January position
5. It has the 13th January date, even though it was published today

The changed timestamp never updated because `setSyncing(TRUE)` was used in the bulk publish operation. The module intentionally skipped that update.

### Why setSyncing() Exists

The function was designed to solve a real problem. When you're migrating content from one Drupal site to another, or synchronising data from an external source, you don't want Drupal automatically changing all your timestamps. You want to preserve the original publication dates and revision history. `setSyncing(TRUE)` tells Drupal: "I'm updating this content, but don't treat it as a new edit."

This is genuinely useful for migrations and data synchronisation. The problem arises when the same technique is used for operations where you *do* want timestamps updated - like bulk publishing.

## The Drupal 11.1 Change

This became more visible in Drupal 11.1 following [core change #3250104](https://www.drupal.org/node/3250104), which refined how the `changed` timestamp behaves during synchronisation. The change meant that modules using `setSyncing()` for bulk operations would more consistently skip timestamp updates, which exposed the inconsistency between what actually happened and what the admin interface showed.

The moderated_content_bulk_publish module was using `setSyncing()` inconsistently - it was commented out in some methods but active in others. This was trial and error, and it showed.

## What You Actually Lose

The real impact goes beyond confusing admin lists:

**Editorial confusion** - Editors expect recently published content to appear at the top of the admin list. When it doesn't, they wonder if the bulk publish actually worked.

**Content discovery** - If you're filtering or sorting by the changed date (as many editorial workflows do), bulk published content appears in the wrong place and might be missed.

**Audit trails** - The changed timestamp is part of your content audit history. If it's wrong, your audit is wrong.

**Analytics and reporting** - If you're tracking when content was last touched for publishing dashboards or compliance reporting, the data is incorrect.

## The Solutions

There are three approaches, each with different trade-offs.

### Option 1: Use the Latest moderated_content_bulk_publish (Recommended)

The patch to [moderated_content_bulk_publish issue #3573928](https://www.drupal.org/project/moderated_content_bulk_publish/issues/3573928) has been merged. The module now updates the changed timestamp correctly during bulk operations. Simply update to the latest release and the issue is resolved.

If you're maintaining a custom bulk publish workflow, ensure you're not using `setSyncing(TRUE)` for operations where you want timestamps updated. Use it only for true synchronisation scenarios (migrations, data imports from external sources).

### Option 2: Use the Publication Date Module

The [publication_date module](https://drupal.org/project/publication_date) solves this by creating a separate field for publication date, distinct from the changed timestamp. Your content shows the publication date (when it was published to users) rather than the changed timestamp (when an editor last edited it).

This is valuable for sites where editorial changes (typo fixes, metadata updates) happen frequently but shouldn't affect content visibility in chronological lists. Your editors publish something on January 13th. They fix a typo on March 5th. The publication date still shows January 13th, because that's when users saw it first.

Combined with the [scheduler module](https://drupal.org/project/scheduler), you get full control over when content appears to users, independent of when it was edited.

### Option 3: Skip Bulk Operations Entirely

Some sites handle this by not using bulk publish at all. Instead, editors publish content individually, or use the scheduler module to set publish dates in advance and let cron handle the publishing automatically.

This trades automation for accuracy. You lose the convenience of bulk operations, but timestamps stay correct and editorial workflows remain predictable.

## The Broader Lesson

This issue reveals something important about how Drupal's synchronisation system works. `setSyncing()` is intentionally blunt - it skips all automatic updates, not just timestamps. This is sometimes the right choice. But it's easy to use it as a convenient shortcut when a more targeted approach would be better.

If you're writing custom modules that bulk update content, be deliberate about what you skip and what you preserve. The changed timestamp matters more than you might think.

## What This Teaches Us

The fix for moderated_content_bulk_publish resolved the issue for that specific module. The broader lesson: `setSyncing()` is a powerful tool that should be used deliberately, not left as trial-and-error code.

The original inconsistency (commented out in some methods, active in others) was exactly the kind of "try it and see" approach that creates hidden bugs. Drupal 11.1's refinement of how the `changed` timestamp behaves during synchronisation brought this pattern into sharper focus and forced a decision: bulk publishing is an editorial action, and editorial actions should update the changed timestamp.

## What to Do Now

**If you're using moderated_content_bulk_publish:** Update to the latest release. The patch is merged, and the module now correctly updates timestamps during bulk operations.

**If you're seeing this issue on an older version:** Either update the module, or switch to the publication_date module for a more comprehensive solution that separates publication dates from edit timestamps.

**If you're writing custom bulk publish functionality:** Don't use `setSyncing()` unless you're genuinely synchronising from an external source (migrations, imports). For bulk editorial operations, let timestamps update naturally. Your editors and your audit trail will thank you.

The key takeaway: `setSyncing()` is a tool for data synchronisation, not editorial workflow. Use it deliberately, not as a convenient shortcut.

## Resources

- [Drupal Core #3250104: ChangedItem does not update timestamp when entity is synchronizing](https://www.drupal.org/node/3250104)
- [Moderated Content Bulk Publish Issue #3573928: Changed date not updated on bulk publish](https://www.drupal.org/project/moderated_content_bulk_publish/issues/3573928)
- [Publication Date Module](https://drupal.org/project/publication_date)
- [Scheduler Module](https://drupal.org/project/scheduler)
- [SynchronizableEntityTrait::setSyncing API Documentation](https://api.drupal.org/api/drupal/core%21lib%21Drupal%21Core%21Entity%21SynchronizableEntityTrait.php/function/SynchronizableEntityTrait%3A%3AsetSyncing/11.x)
