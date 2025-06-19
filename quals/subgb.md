# Subgb

# The problem

Get the flag from a very small, seemingly random noise image.

# The observation

One could treat arbitrary data as bitmap values, but the image, as a 60x8 24-bit image, requires
60 * 8 * 3 = 960 bytes to describe completely, which is way too many bytes for a typical flag.

If the flag were to be spelled out visually, it would only require a 1-bit (black or white) image.
But looking at the pixels, we realize that every single pixel's component is either at 0 or 255, so
perhaps each pixel actually encodes three pixels... vertically or horizontally.

# The execution

To reveal the flag, take each column of pixels, separate its RGB components, and write each
component as its own column in order to obtain a 180x8 8-bit image.
