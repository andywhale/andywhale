---
title: "Recommended Drupal modules"
description: "There are a number of drupal 8 modules which I almsot always start a project with, this is by no means an exhaustive list but this is a few of the go to choices."
date: 2020-08-27T13:18:30+01:00
image: "images/drupal-node-boolean.png"
draft: false
type: "post"
tags: ["drupal", "drupalmodule", "php"]
---

There are a number of drupal 8 modules which I almsot always start a project with, this is by no means an exhaustive list but this is a few of the go to choices.

## [Coffee](https://www.drupal.org/project/coffee)

A solid development module to allow a developer to jump around the site using shortcut keys much akin to osx spotlight search.

## [Config Split](https://www.drupal.org/project/config_split)

Hands down the best thing about the progression of Drupal 8 was config import/export, being able to adequately store config in version control and import/export as a part of site development workflow. However this alone does not adequaltely meet the requirements of multiple environments, but with the addition of config split different configuration on different environments can be easily managed.

As a general rule I have two config split profiles:

 * Development
 * Production

So I can have my development modules only enabled and configured on development environments, and production settings such as performance and aggregation are only enabled on my production environments.

### Bonus choice: [Config Ignore](https://www.drupal.org/project/config_ignore)

Configuration sync works very well for the most part, but there are always areas of a site that require alteration by the end user, in my experience these mostly revolve around webforms. The site user is likely going to need to manage webforms in a production envrionment, and under normal scenarios these settings would be wiped out when a site was deployed.

However config ignore allows for certain aspects of a site to not be overwritten when config is imported, allowing for example:

```
webform.webform.*
```

to always be ignored, meaning that webform entities would never be overwritten.

## [Easy Breadcrumb](https://www.drupal.org/project/easy_breadcrumb)

Easy breadcrumb fulfils the role that core breadcrumbs should be covering by adding in some basic configuration that are widely regarded as best practices - see the module page for the full list but all significant features tend to be used in various permutations for different sites.

### Bonus choice: [Template Breadcrumb](https://www.drupal.org/project/template_breadcrumb)

*Disclaimer: This is one of my modules*

Useful wherever you are using breadcrumbs in page or inside a template rather than as a block. In a nutshell this allows a breadcrumb to be added to a node display in any view mode, which is quite flexible and useful if your site designs require it.

## [Focal Point](https://www.drupal.org/project/focal_point)

Image cropping in Drupal has always been pretty flexible, but the focal point module just allows the user to have a bit more say in the end result, allowing a user to select a focal point in the image, which can then be cropped around at whichever size the image is going to be cropped to.

This is a good module for striking a balance between control and cutting down the editorial workload. Image widget crop is useful if you need to give the user full fine grained control over each and every crop, but that in turns adds an extensive workload to your editorial users, that maybe too much dependant upon the requirements.

## [Meta Tag](https://www.drupal.org/project/metatag)

So this module is only really useful if you want to increase the traffic to your site via search or social media sharing (covers everybody right?).

## [Node Boolean](https://www.drupal.org/project/node_boolean)

*Disclaimer: This is one of my modules*

I almost always find a need to have one or more blocks shown against nodes dependant upon a boolean (toggle) field, e.g. `[*] Show this block` so I created this module to solve that problem, it is essntially just a block condition plugin wrapped in a module. There are heavier duty modules that will solve this problem, but I've never needed to resort to those.

## [Paragraphs](https://www.drupal.org/project/paragraphs)

So this module pretty much is the most important tool of any of my sites, it gives an unparalleled ability to create a set of website building blocks, which when designed well can give a site editor all of the tools they need.

The user experience is still lacking, but there are still projects which are working to address this.

But overall this module is a must have depending on how you are planning on putting together your site.

## [Webform](https://www.drupal.org/project/webform)

Not so much a module as a module ecosystem, this module is something I return to time and time again not just to create contact forms, etc, but also as a go to resource on how to create Drupal 8 modules and plugins as the code is so expansive and there is always an example of how to solve a problem in this complete codebase.