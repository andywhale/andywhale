---
title: "Random Number Field"
description: "Drupal 9 readiness pushed me to fix a bug with this module and move it into a release candidate"
date: 2020-06-14T13:21:36+01:00
image: "images/random-number-field.png"
draft: false
type: "post"
tags: ["drupal", "drupalmodule", "php"]
---
There has been a bug hanging around in this module for a little while and I'd not found time to fix it. As the Drupal 9 automated fixes are handed out to each of the Drupal 8 modules (I think I've had about 8 in the last week or two) I figured it was the right opportunity to make some time.

I'm also not entirely convinced that it was the best fix, but it does resolve the issue of the random number field populating and storing a non-random number, causing it to be relatively useless if used incorrectly.

So I used the path matcher to determine if the field was being displayed on a field edit form, and not storing a static number.

[https://www.drupal.org/project/random_number_field]
