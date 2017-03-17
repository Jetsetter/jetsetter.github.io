---
title: "Duplicate image detection with perceptual hashing in Python"
author: Ben Hoyt
layout: post
disqus_page_identifier: duplicate-image-detection
date: 2017-03-21
permalink: /2017/03/21/duplicate-image-detection/
published: false
---


Recently we implemented a duplicate image detector to avoid importing dupes into [Jetsetter's](https://www.jetsetter.com/) large image store. To achieve this, we wrote a Python implementation of the dHash perceptual hash algorithm and the nifty BK-tree data structure.

Jetsetter has hundreds of thousands of [high-resolution travel photos](https://www.jetsetter.com/photos), and we're adding lots more every day. The problem is, these come from a variety of sources and are uploaded in a semi-automated way, so there are often duplicates or almost-identical photos that sneak in. And we don't want our photo search page filled with dupes.

So we decided to automate the task of filtering out duplicates using a [perceptual hashing](https://en.wikipedia.org/wiki/Perceptual_hashing) algorithm. A perceptual hash is like a regular hash in that it is a smaller, compare-able fingerprint of a much larger piece of data. However, unlike a typical hashing algorithm, the idea with a perceptual hash is that the perceptual hashes are "close" (or equal) if the original images are close.


Difference hash (dHash)
-----------------------

We use a perceptual image hash called dHash ("difference hash"), which was [developed by Neal Krawetz](http://www.hackerfactor.com/blog/?/archives/529-Kind-of-Like-That.html) in his work on photo forensics. It's a very simple but surprisingly effective algorithm that involves the following steps:

* Convert the image to grayscale
* Downsize to a 9x9 square of gray values (or 17x17 for a larger, 512-bit hash)
* Calculate the "row" difference hash: for each row, move from left to right, and output a 1 bit if the next gray value is greater than or equal to the previous one, or a 0 bit if it's less than (each 9-pixel row produces 8 bits of output)
* Calculate the "column" difference hash: same as above, but for each column, moving top to bottom
* Concatenate the two bit values together to get the final 128-bit hash

dHash is great because it's fairly accurate, and very simple to understand and implement. It's also fast to calculate (Python is not very fast at bit twiddling, but all the hard work of converting to grayscale and downsizing is done by a C library: [ImageMagick/wand](http://docs.wand-py.org/en/latest/) or [PIL](https://pillow.readthedocs.io/en/4.0.x/)).

Here's what this process looks like visually:

* TODO: original image
* TODO: grayscale, downsized image (but blown up)
* TODO: final output black and white (but blown up)

The core of the dHash code is as simple as a couple of nested `for` loops:

```python
def dhash_row_col(image, size=8):
    width = size + 1
    grays = get_grays(image, width, width)

    row_hash = 0
    col_hash = 0
    for y in range(size):
        for x in range(size):
            offset = y * width + x
            row_bit = grays[offset] < grays[offset + 1]
            row_hash = row_hash << 1 | row_bit

            col_bit = grays[offset] < grays[offset + width]
            col_hash = col_hash << 1 | col_bit

    return (row_hash, col_hash)
```

It's a simple enough algorithm to implement, but there are a few tricky edge cases, and we thought it'd be nice to roll it all together and open source it, so our Python code is available [on GitHub](https://github.com/Jetsetter/dhash) and [from the Python Package Index](https://pypi.python.org/pypi/dhash) -- so it's only a `pip install dhash` away.


Dupe threshold
--------------

To determine whether an image is a duplicate, you compare the dHash values. If the hash values are equal, the images are nearly identical. If they hash values are only a few bits different, they're very very similar -- so you can calculate the number of bits different ([hamming distance](https://en.wikipedia.org/wiki/Hamming_distance)) between the two values, and then check if that's under a given threshold.

Side note: there's a helper function in our Python dhash library called `get_num_bits_different()` that calculates the delta. Oddly enough, in Python the fastest way to do this is to XOR the values, convert the result to a binary string, and count the number of `'1'` characters (this is because then you're asking builtin functions written in C to do the hard work and looping):

```python
def get_num_bits_different(hash1, hash2):
    return bin(hash1 ^ hash2).count('1')
```

On our set of images (over 200,000 total) we set the 128-bit dHash threshold to 2. In other words, if the hashes are equal or only different in 1 or 2 bits, we consider them duplicates. In our tests, this is a large enough delta to catch most of the dupes. When we tried going to 4 or 5 it started catching false positives -- images that had similar fingerprints but were too different visually.

For example, this was one of the image pairs that helped us settle on a threshold of 2. These two images have a delta of 4 bits:

![False positive "dupes" with dHash delta of 4 bits](/public/img/dupes-false-positive.jpg)


MySQL filter
------------

TODO


BK-trees and fast dupe detection
--------------------------------

TODO

BK-tree:  ([described by Nick Johnson](http://blog.notdot.net/2007/4/Damn-Cool-Algorithms-Part-1-BK-Trees)).
