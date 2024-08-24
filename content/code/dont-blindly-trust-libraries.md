+++
title = "Don't Blindly Trust Libraries"
date = 2024-08-24T10:21:58-04:00
draft = false
tags = ["php"]
+++

## What was I doing in the first place?

To give you a little background on this issue, first I have to mention that
this is a work project. We do all of our development in PHP, and use many
libraries. All of these libraries are reviewed by my senior
dev or our team lead. We've had a lot of success in narrowing down which
packages really fit our use case, and picking high quality packages.
But, inevitably, something was bound to slip through the review process.

This project is a document classification system. It uses text extracted via
AWS Textract to power a custom algorithm for fingerprinting documents and then
classifying. It takes in _either_ a PDF with any number of pages or a single JPEG.
If it receives a PDF, it takes each page in order, converts it to a JPEG, and
then sends that to Textract. If it receives a JPEG, it sends that to Textract
without modification.
It's a pretty nifty project, but there have been a number of challenges along
the way.

## The Package

The package in question is one we have had in the project for a while, but have
used very little. I won't name it, but I will tell you that it's a package for
converting PDFs to images. It's a solid package, even if I don't resonate with
some of the overall design decisions.

This particular package uses ImageMagick behind the scenes to convert the PDF.
ImageMagick is a powerful tool, and has a ton of options for converting between
different formats, resizing, and compressing images, among many, many other
functions. So many, in fact, that ImageMagick has a tendency to be complex and
usually requires a lot of trial and error to get a particular process right.

## The Problem

One of the challenges we faced is the file size limit on synchronous Textract calls
that transfer image data. The limit is 10MB, which, one would think would
be sufficient, they're just JPEGs after all. It turns out that's not the case in
our application. We have a number of documents that are 300mm long and contain a
lot of text. Bounding boxes, back-side bleed-through, elements of the form,
large sprawling signatures, color; all of these add up to a sizable image file.
In testing, most of these images ended up being around 12-13MB.

Obviously this is a problem when using Textract. \*Cue the need for a solution to
making the images smaller before transferring.\* We discussed
several ideas, ultimately leading to no clear answers before testing.

Textract specifically recommends a DPI of 300 for optimal processing. This is,
frankly, huge for a document that is 300mm long, and factored into our
decisions. First we discussed compressing the image in the hopes that it would
reduce the file size, but we would be sacrificing a lot of valid text since we
ultimately would be blurring the text and possibly introducing artifacts. We
also discussed converting the image to gray scale and thresholding it to black
and white, then de-noising the resulting image. Finally, we discussed reducing
the DPI of the image as a last resort, but my suspicion is that we would be an
unfavorable solution to the problem given Textract's specific recommendation.

The package in question implements a function called `getImageData` that is called when
saving a page of the PDF as an image.
Here is a code snippet of that implementation:

```php
public function getImageData(string $pathToImage, int $pageNumber): Imagick
    {
        /*
         * Reinitialize imagick because the target resolution must be set
         * before reading the actual image.
         */
        $this->imagick = new Imagick();

        $this->imagick->setResolution($this->resolution, $this->resolution);

        if ($this->colorspace !== null) {
            $this->imagick->setColorspace($this->colorspace);
        }

        if ($this->compressionQuality !== null) {
            $this->imagick->setCompressionQuality($this->compressionQuality);
        }

        $this->imagick->readImage(sprintf('%s[%s]', $this->filename, $pageNumber - 1));

        if ($this->resizeWidth !== null) {
            $this->imagick->resizeImage($this->resizeWidth, $this->resizeHeight ?? 0, Imagick::FILTER_POINT, 0);
        }

        if ($this->layerMethod !== LayerMethod::None) {
            $this->imagick = $this->imagick->mergeImageLayers($this->layerMethod->value);
        }

        if ($this->thumbnailWidth !== null) {
            $this->imagick->thumbnailImage($this->thumbnailWidth, $this->thumbnailHeight ?? 0);
        }

        $this->imagick->setFormat($this->determineOutputFormat($pathToImage)->value);

        return $this->imagick;
    }
```

As you can see, here the package is using `setCompressionQuality()`. Guess what
doesn't work? That's right, `setCompressionQuality()`. Here is an excerpt from
the docs on [php.net](https://www.php.net/manual/en/imagick.setcompressionquality.php):

```plaintext
Caution
This method only works for new images e.g. those created through Imagick::newPseudoImage. For existing images you should use Imagick::setImageCompressionQuality().
```

Since this image is being read from a file, and not created by Imagick,
`setCompressionQuality()` function isn't applicable here.

## The Solution

The first thing I tried, before I even discussed anything with the team, was to
increase the compression to 80% of the original image. The function for this
when using ImageMagick (`imagick` in PHP land) is either:

1. `setCompressionQuality()`
1. `setImageCompressionQuality()`

So, I patched the file, and tried again using `setImageCompressionQuality()` and
voila! It worked.

At the end of the day, after much testing had been done and a long team conversation
had been had, I ended up settling on the following solution:

1. Convert the image to grayscale.
1. Threshold the image to black and white.
1. De-noise the image.

This method of processing leaves the DPI of the image unchanged and leaves the
image uncompressed and free of artificial artifacts and blurring. It also leaves
pages in a state where they are normalized, but can be stored as is,
without the need to store the original image for reconstructing a PDF on the other
end of the system. This was the most optimal solution for our use case.

## Conclusion

The moral of the story: Don't blindly trust libraries. Always read the docs and review
the code as much as possible.
