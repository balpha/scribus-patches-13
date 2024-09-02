# Overview

This repository is a collection of patches that I apply on top of my version of [Scribus](http://www.scribus.net/canvas/Scribus) to add a few features I miss. Feel free to use them as well, but be aware that they are very specific to the way I use Scribus, and that they **should be considered incomplete** (e.g. missing undo/redo functionality, sub-par UI, etc.)

I maintain them as a [Mercurial patch queue](http://mercurial.selenic.com/wiki/MqExtension) on top of a repository that's an HG conversion of the [original Subversion repository](http://scribus.net/websvn/listing.php?repname=Scribus), but since they're just a bunch of diffs, you can also use them without MQ. They are maintained to be applied to the Version14x branch.

# Description of the patches

### Swapping pages

This patch adds an option to Scribus' "Move pages" dialog that allows you to swap two pages in a document.

### Show soft hyphen in the story editor

Since I like to tightly control the hyphenation of text, I often manually add soft hyphens (U+00AD) to my text. These characters are usually invisible, which makes this manual work hard to manage. This patch makes the soft hypens visible inside the story editor by displaying them as red dashes (similar to how other special characters like frame breaks are displayed).

### Prevent hyphenation on a per-word basis

This patch allows you to disable automatic hyphenation of a single word in a text by prepending it with a soft hyphen (U+00AD). If I recall correctly (it's been a long time), this is how InDesign does it.

### Soft shadows

An oft-[requested](http://bugs.scribus.net/view.php?id=3712) feature in Scribus is the possibility to add soft shadows programatically to objects (I chose the name "soft shadows" over the more common "drop shadows" because Scribus already has a text effect with the latter name). This patch is a (partial) implementation of this feature. Note that this is currently in the process of being pulled into the 1.5 branch of Scribus.

It adds a "soft shadow" section to the "X, Y, Z" tab of the properties palette, where you can set the horizontal and vertical displacement, the color, the blur radius, the opacity, and the composition mode of the shadow.

There are some issues with this patch; for example, applying a soft shadow to a text frame prevents the text from flowing around shapes. This feature also probably doesn't play well with various Scribus features that I personally hardly use (e.g. render frames, tables).

With this patch the shadow is only visible when you export the document as a PDF document (version 1.4 or higher, since it uses transparency). There is a separate patch that also shows the soft shadows in the UI. Any other mechanisms (in particular, various exporters) will not show the shadow.

It also doesn't work on master pages (at least it didn't use to; I haven't tried it in a while).

Be aware that this is a **file format change**. If you save a .sla document with a soft shadow, then open and re-save this document with an *unpatched* version of Scribus, the shadow will be gone even if you later open the file in the patched version again. Because vanilla Scribus doesn't know about this feature, it ignores the soft shadow settings when opening a document, and does not persist them when re-saving it.

### Soft shadows in the UI

This patch cause the soft shadows (see the previous patch) to also be visible in the Scribus interface, so you don't have to export to PDF in order to actually see them. Since this is a fairly new patch and I haven't used it extensively yet, it's currently a separate diff file. Eventually it will be merged with the actual soft shadow patch (on which it obviously depends).

### Choice of page an object belongs to

For every object in your document, Scribus chooses a page that counts as containing this object. It does so by choosing the first page (i.e. the one with the lowest page number) that intersects with the object's bounding box. Most of the time, this choice of page doesn't matter, but sometimes (in particular when moving pages around), it does. Since this has bitten me a few times (see my [issue report](http://bugs.scribus.net/view.php?id=7389) on this matter), this patch changes the choice to pick the page that intersects *with the center* of the bounding box; in other words, it chooses the page that contains the biggest part of the bounding box. This change only applies to double-sided documents.

### New layer option: Text always flows

Scribus allows for text to flow around an arbitrary object by setting one of the corresponding options in the properties palette's "Shape" tab. However this only works if the flow-around object sits *above* the text box (either on a higher layer, or on the same layer but higher than the text). Since I sometimes need the text to flow around objects that are on a lower layer, this patch adds a new column of checkboxes to the layer palette (disabled by default).

When you enable this setting on a layer, any text on this layer will flow around *all* objects that have a "text flows around" setting, regardless of whether these objects sit above or below the text. To clarify: You set this property on the layer that *contains the text*, not the flow-around object.

When changing this setting, you may have to force Scribus to re-flow the affected text boxes by moving them (just press left arrow then right arrow) or by saving and re-opening the document.

Note that this, too, is a **file format change**. If you save a .sla document where you have enabled this setting for a layer, then open and re-save this document with an *unpatched* version of Scribus, the setting will be disabled even if you later open the file in the patched version again. Because vanilla Scribus doesn't know about this feature, it ignores the "text always flows" settings when opening a document, and does not persist them when re-saving it.

### Image orientation

This patch allows changing the orientation of an image in steps of 90 degrees without having to rotate the image frame. Scribus 1.5 will have this feature built in (with arbitrary angles), but 1.4.x doesn't.

While true for all these patches, this one especially is tailored to my use case: Easier handling of photos (in formats like TIF, PNG, JPEG) that aren't present in the proper orientation, and PDF output. I use Scribus exclusively for print design, so all I ever do is create PDFs. Therefore, just like with the soft shadows, I can't promise that this e.g. works with all exporters. It also does not or may not work correctly when the image is an embedded PDF or other special-case format.

Also note that this is a fairly new patch and thus I may not have found all issues yet.

Finally this, too, is a **file format change**. If you save a .sla document where you have changed an image's orientation, then open and re-save this document with an *unpatched* version of Scribus, the orientation will be back to zero even if you later open the file in the patched version again. Because vanilla Scribus doesn't know about this feature, it ignores the orientation setting when opening a document, and does not persist it when re-saving it.

### Auto contrast

This patch adds a new image effect called "Auto contrast".

The default settings for this effect do almost precisely the same thing as [GIMP's "auto white balance"](http://docs.gimp.org/2.8/en/gimp-layer-white-balance.html). Discarding (thus clipping) the top and bottom 0.5% of the pixels, it stretches the red, green, and blue color channels to the full range. This can do magic to enhance photographs, in particular when they're tinted towards a certain color. And it can also do horrible things, depending on the image, so be sure to look at the actual result.

The effect has a few options:

- *Threshold low* and *Threshold high*: This is the percentage of the image's pixels that are discarded when calculating the range of color values. As mentioned above, the default is 0.5% for both, but this can be adjusted indepently for dark (low) and light (high) values. If, for example, your photo has a large over-exposed area (that you may be cropping away anyway), you'll probably want to increase the high threshold.

- *Sync colors*: If this is checked, then instead of adjusting the color channels independently, the low and high values will be identical for all three color channels (using the lowest low value and the highest high value). This means that there will be no color correction (or distortion); rather this will just increase the image contrast.

- *Amount*: Controls the strenght of the effect from 100% (the default) to 0% (no change at all).

This effect only works on RGB images. Frankly, I've never used CMYK images, and I have now idea how I would treat them for applying this effect. In particular this means that this effect **won't do anything when exporting CYMK PDFs unless you also apply the "effects before color management" patch** (see its description for the reason).

This is not technically a **file format change**, however I have not tested how Scribus behaves when a file contains an unknown image effect. Thus opening and/or re-saving a .sla file that contains this effect with an unpatched version of Scribus may have unintended consequences.

### Quick preview mode toggle

When you turn preview mode on and off, Scribus reloads and recalculates each and every picture in the document. There are probably cases where images are shown differently with and without preview mode, but in my use, it never makes a difference. Thus the only thing this reloading/recalcing does for me is making the toggle infuriatingly slow in documents with a decent amount of pictures, even though all I wanted was to hide guides/frame borders/etc. for a second.

This patch changes the behavior of the "toggle preview" button in the window's bottom right to *not* recalc the pictures after toggling the mode, making the switch between the modes very quick. To get the old behavior, either long-click the button and then click the "Recalc pictures after switch" that pops up, or toggle the mode via the View menu, not the button.

### Apply image effects before color management

When using color management, Scribus applies image effects *after* converting the image to the target colorspace. This is very unfortunate, because it makes image effects combined with color management (especially soft proofing) utterly useless. There's a [very old bug report on this matter](http://bugs.scribus.net/view.php?id=4270).

This patch changes this order; image effects are applied on the original image data as loaded from the file; only then are any color conversions applied. Scribus' loading of TIFF and PSD files is a bit special here; color conversion happens at the same time as loading the image. I don't know the reasoning behind this, but I had to handle this. For this reason, the patch handles this case as follows: When loading the file, the image is converted to an intermediate color space (using the document's default image color profile). Image effects are applied to the result of this conversion, and then the resulting image is converted to the actual target color space.

Grayscale TIFFs/PSDs aren't currently handled at all by this patch (which means they don't get any image effects).

If you use both color management and image effects, this patch is a **major change**, and it's not hidden behind a setting -- all your files will suddenly behave differently. So make very sure you want to apply it.

### Progress bar for image recalc during document load

This is a very small patch. When opening a document that uses color management, Scribus will load every image twice. [Here's the corresponding bug report](http://bugs.scribus.net/view.php?id=9826). This patch does *not* fix this (it would be a pretty big change, given how image loading works behind the scenes). All this patch does is making this issue a bit more bearable, especially for documents with lots of images, by reflecting it in the progress indicator the status bar. That means the bar will fill up to 100% *twice* when loading a (color-managed) document. But at least there's no minute-long pause during which you have no idea what's happening.

### Keep scale and position when replacing images

When an image frame already contains a picture and you load a new one, Scribus resets the image's scaling and position. This patch changes this behavior in the following way: If the image frame is set to manual scaling (not fit-to-frame), then instead of resetting, the scale and position are adjusted by the ratios of the old image's width/height and the new image's width/height.

The use case for this is replacing a picture with another picture that has the same content but different dimensions (for example, a higher-resolution scan of the same photo). With this patch, the new image would be cropped and scaled to show exactly what the old image did.

If you replace an image with an unrelated image (and the frame uses manual scaling), the resulting scale and offset therefore won't make much sense, but a) this is probably rare, and b) with Scribus' built-in behavior, chances are that the values aren't the desired ones either, and thus you'd have to adjust them in either case.

### Do not recalc images when editing colors

After you edited a document's colors, Scribus will reload and recalculate all documents in the image, which can take a very long time. This patch changes this behavior to only recalculate those images that have one of the colorization effects applied, since only those images can actually be impacted by changing the document colors.

### Crop images when creating PDFs

When you have an image positioned in an image frame in a way such that not the whole image is visible, Scribus will, when creating a PDF, still put the full image into the PDF file. This patch modifies the exporter such that it crops the image to the part that's vivsible in the frame, only exporting the necessary part. This can very significantly reduce PDF file sizes, especially when you have many images.

This behavior can be enabled and disabled in the PDF export dialog. It's enabled by default for new files, but disabled when loading a .SLA files that was created with an un-patched version. It's a file format change, but a minor one -- you will have to reenable the cropping if you modified and re-saved a file in an unpatched version.

Because of an overlap in code lines, this patch will only apply cleanly if the image orientation patch is applied as well. They don't actually depend on each other, but this patch has to be modified slightly in order for it to be applicable without the image orientation patch.

# License

Scribus itself is copyright 2001–2013 Franz Schmid and rest of the members of the Scribus Team and for the most part licensed under GPLv2+. See their file [COPYING](http://scribus.net/websvn/filedetails.php?repname=Scribus&amp;path=%2Fbranches%2FVersion14x%2FScribus%2FCOPYING) for details.

These patches ("the program") are copyright 2008-2014 Benjamin Dumke-von der Ehe.

This program is free software; you can redistribute it and/or
modify it under the terms of the GNU General Public License
as published by the Free Software Foundation; either version 2
of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.