---
title: "Wagtail, Tailwind and Embedded images with captions"
description: "Enabling custom image formatters in Wagtail and using Tailwind to display custom image models with a caption"
date: 2020-01-14T13:08:30+01:00
image: "images/wagtail-image-chooser.png"
draft: false
type: "post"
tags: ["wagtail", "tailwind", "python", "numiko"]
---

Disclaimer; I am a big fan on Wagtail but I regularaly have issues finding appropriate documentation, so most of the information posted here is available in the docs or within the code. This article is more of a reference for when I forget it all over again.

## Adding a caption to an image

Now this is well documented, you just need to follow the guide [on the official wagtail docs](https://docs.wagtail.io/en/latest/advanced_topics/images/custom_image_model.html) for adding a custom image model.

You can then add a CharField to the custom image model

```
caption = models.CharField(max_length=255, blank=True)
```

and add this to the admin form fields

```
admin_form_fields = Image.admin_form_fields + (
        'caption',
    )
```

The only additional thing to be aware of is that this is much easier added from the outset, and if you already have data you will need to [write a data migration](https://docs.djangoproject.com/en/3.0/topics/migrations/#data-migrations).

## Adding your own image formats

In order to customise the classes and size of our embedded images we need to unregister the existing image format and add our own variants. Explained well in the [documentation](https://docs.wagtail.io/en/latest/advanced_topics/customisation/page_editing_interface.html#rich-text-image-formats).

This is achieved using the `unregister_image_format` and `register_image_format` methods from `wagtail.images.formats` within an `image_formats.py` file in an enabled app.

```
from wagtail.images.formats import Format, register_image_format, unregister_image_format

unregister_image_format('fullwidth')

register_image_format(Format('fullwidth', 'Full width', 'block w-full pb-20 border-color-base-keyline', 'width-592'))
```

## Customising image formats to show your caption

We are now ready to expand our formats to show the caption, and add tailwind classes to enahance how they are displayed, obviosuly the classes we'll use will be entirely dependant on our intended look, so will almost certainly need to be changed from site to site.

There are two avenues here; to template the output or include the variables in the declaration, I'm using the latter here.

So in order to do that I'll extend the `Format` with my own `CaptionImageFormat` with a couple of extra arguments to determine how the formats will display. All of these steps are lifted from [the official docs](https://docs.wagtail.io/en/latest/advanced_topics/images/changing_rich_text_representation.html).

The new format (shown below) is defined within the `image_formats.py` and will be included in `register_image_format` in place of a `Format` object.

```
class CaptionedImageFormat(Format):
    def __init__(
        self,
        name,
        label,
        classnames,
        filter_spec,
        figure_classes,
        figcaption_classes="",
    ):
        super().__init__(name, label, classnames, filter_spec)
        self.figure_classes = figure_classes
        self.figcaption_classes = figcaption_classes

    def image_to_html(self, image, alt_text, extra_attributes=None):

        image_html = super().image_to_html(image, alt_text, extra_attributes)
        caption_html = ""

        if image.caption:
            caption_html = format_html(
                "<figcaption class='{}'>{}</figcaption>",
                self.figcaption_classes,
                image.caption,
            )

        return format_html(
            "<figure class='{}'>{}{}</figure>",
            self.figure_classes,
            image_html,
            caption_html,
        )
```

I can now define each of my image formats with the classes required to make the embedded image appear within a `<figure>` and displaying a `<figcaption>` if a caption has been defined.

```
unregister_image_format("fullwidth")
register_image_format(
    CaptionedImageFormat(
        "fullwidth",
        "Full width",
        "block w-full pb-20",
        "width-720",
        "my-10 md:my-20 lg:my-40",
    )
)
```

This could come in useful for anybody trying to display image captions within a WYSIWYG. As stated all of these steps come directly from the docs, specifically these pages:
* [Custom Image Models](https://docs.wagtail.io/en/latest/advanced_topics/images/custom_image_model.html)
* [Rich Text Image Formats](https://docs.wagtail.io/en/latest/advanced_topics/customisation/page_editing_interface.html#rich-text-image-formats)
* [Changing Rich Text Representation](https://docs.wagtail.io/en/latest/advanced_topics/images/changing_rich_text_representation.html)
