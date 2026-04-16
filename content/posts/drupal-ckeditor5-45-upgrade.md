---
title: "Upgrading to Drupal 10.5 and 11.2: Navigating the CKEditor 5 v45 Landscape"
description: "CKEditor 5 v45 brings significant improvements but introduces breaking changes. Here's what site builders and developers need to know to upgrade successfully."
date: 2026-04-16T13:21:43+01:00
image: "images/DRUPAL-EL_blue_RGB.png"
draft: false
type: "post"
tags: ["drupal", "drupal10", "drupal11", "ckeditor", "upgrade"]
---

Drupal 10.5.0 and 11.2.0 bring a significant upgrade to CKEditor 5, moving from v44.0.0 to v45.2.0. This isn't just a minor version bump - CKEditor 5 v45 introduces substantial improvements to the editor's architecture and user experience. But it also includes breaking changes that ripple through the ecosystem of contributed modules that extend CKEditor.

If you're planning to upgrade to either of these Drupal versions, understanding these changes and their impact on your site is essential.

## What's Better in CKEditor 5 v45

The upgrade brings real improvements to the editor experience:

- **Refined link UI** - The link dialog has been completely redesigned, making it more intuitive and consistent with CKEditor's modern interface
- **Icon system overhaul** - Icons have been centralised into a dedicated package (@ckeditor/ckeditor5-icons) and renamed for consistency across the ecosystem
- **Enhanced table handling** - Tables now receive the `table` class by default, enabling better styling integration with CSS frameworks like Bootstrap
- **Improved architecture** - The underlying command structure is cleaner, with better separation of concerns

These improvements position CKEditor as a more mature, modern editor for content creation. But they come at a cost for modules that hook into the old interfaces.

## The Challenge for Module Maintainers

Here's where it gets complex. CKEditor 5 v45 introduced several breaking changes that required updates to contributed modules:

**Icon name changes** - All icons were renamed and moved to a new package. Modules that referenced icons by their old names (like `icons.pencil` or `icons.cog`) needed updates. Drupal added a backward-compatibility layer with deprecation warnings for 11.x, but modules still needed adjusting.

**Link dialog restructuring** - The link feature was significantly overhauled. Modules extending the link dialog had to account for:
- A new `displayedText` argument on link command execution
- Removal of `ck-button-save` and `ck-button-cancel` CSS classes
- Restructured form views and button handling

**Table class additions** - The unconditional addition of the `table` class to all `<table>` elements affected modules that depend on CSS styling based on the presence or absence of that class.

For module maintainers, this meant real work to ensure their modules stayed compatible. But the good news is that the community moved quickly. Within weeks, most affected modules had updates released.

## Impact on Site Builders

If you're running a site with contributed CKEditor modules, you need to know which ones require updates before you can upgrade to Drupal 10.5 or 11.2. Here are the modules reported as affected:

**Now updated and compatible:**
- **Linkit** (7.0.5+) - Decorator and link attribute logic updated
- **Editor Advanced Link** (2.3.0+ for CKEditor 45, 2.2.7 for older versions) - Fixed view restructuring and button class removal
- **CKEditor Abbreviation** - Compatibility fix released
- **Anchor Link** - Icon updates and library support added
- **CKEditor iFrame** - CKEditor 5 45.2.0 support added
- **CKEditor Media Resize** - Compatibility fix for CKEditor 45.x
- **Editor File** - Updated to replace `.once()` usage for Drupal 10.5
- **CKEditor Responsive Table** - Compatibility fix for CKEditor 45 changes

**Still in progress:**
- **Embedded Content** - Icon rename compatibility being addressed
- **IMCE** - Browse-files button visibility issue with new Insert/Update button
- **CKEditor Link Styles** - CKEditor 45+ compatibility work ongoing
- **Entity Embed** - Widget toolbar icon issues still being reviewed

## Upgrade Strategy for Site Builders

If you're using any CKEditor-extending modules, here's a practical approach:

**Before upgrading Drupal:**

1. **Audit your CKEditor configuration** - Document which CKEditor modules you're using
2. **Check module compatibility** - Visit each module's issue queue (linked in the resources section) to see if updates are available
3. **Test on staging** - If updates are available, test them on a staging environment first
4. **Plan your upgrade window** - Some modules may still be in development, so timing matters

**If your site uses a module still in progress:**

- Follow the issue thread on drupal.org - module maintainers are actively working on fixes
- Consider temporarily disabling features that depend on CKEditor until the fix is available
- Don't let one module block your upgrade if it's non-critical

**After upgrading:**

1. Clear all CKEditor caches and browser caches
2. Test all editor functionality thoroughly
3. Check for console errors related to CKEditor icons or link dialogs
4. Verify that any custom CSS rules for tables (especially if using Bootstrap) still work as expected

## Themes and Bootstrap-Based Sites

If your site uses Bootstrap or custom CSS rules that target elements by the presence or absence of the `table` class, you may need to review your CSS. The addition of the `table` class to all tables by default is good for styling consistency, but it means CSS that relied on tables *not* having that class will need adjustment.

This is typically a minor CSS update, but it's worth checking during your upgrade.

## A Sign of a Healthy Ecosystem

The fact that so many modules were quickly updated is actually a positive sign. It shows that the Drupal community responds rapidly to these kinds of changes. Module maintainers understood the impact and got updates out quickly.

Yes, there's work involved. Yes, you need to plan for updates. But this is how open-source software evolves - with breaking changes that improve the architecture, followed by the community rallying to support those changes.

## Key Takeaways

1. **CKEditor 5 v45 is worth upgrading to** - The improvements justify the work
2. **Most affected modules already have fixes** - Check the module pages for the latest releases
3. **Plan your upgrade** - Audit which CKEditor modules you use and verify they're compatible
4. **Test thoroughly** - CKEditor changes can be subtle, so test editor functionality after upgrading
5. **Expect ongoing support** - A few modules are still being updated, but the work is happening

If you're upgrading to Drupal 10.5 or 11.2, take the time to plan your CKEditor ecosystem. The upgrade is worth it, and the community is here to support the transition.

**Resources:**
- [Drupal Core Issue #3523018: Update CKEditor 5 to 45.2.0](https://drupal.org/project/drupal/issues/3523018)
- [Linkit Issue #3522476: CKEditor 5 v45 compatibility](https://drupal.org/project/linkit/issues/3522476)
- [Editor Advanced Link Issue #3531194: CKEditor 5 45 compatibility](https://drupal.org/project/editor_advanced_link/issues/3531194)
- [CKEditor Responsive Table Issue #3531224: CKEditor 5 45 compatibility](https://drupal.org/project/ckeditor_responsive_table/issues/3531224)
- [CKEditor 5 Documentation](https://ckeditor.com/docs/ckeditor5/latest/)
- [Drupal CKEditor 5 Upgrade Guide](https://www.drupal.org/docs/drupal-apis/text-editor-api/ckeditor-5)
