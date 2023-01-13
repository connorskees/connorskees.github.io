+++
title = "The PNG File Format"
+++

<!-- # The PNG File Format -->

The PNG file format specifies a way to losslessly encode bitmaps. This blog post intends to describe, in-depth, all algorithms and ideas behind the PNG file format.

<!-- It is the most popular and ubiquitous format for storing and transmitting images where the quality of the image is important. -->

Namely, this blog post will describe — in almost too much detail — the process for decoding and rendering PNG files, color spaces and pixel formats, gamma correction, the history and intuition behind Adam7 interlacing, text compression and DEFLATE, data corruption, advanced SIMD algorithms for decoding filters 8x faster, and novel SIMD algorithms not yet in production that have the potential to speed up decoding by ~32x.

It is my hope that after reading this article, you will understand not only the PNG file, but also many core technologies underlying the web, computer graphics, and systems-level programming.


<!--
## Comparison to other formats

JPEG:
GIF:
WEBP:
-->

## General structure

At a high level, a PNG file is composed of "magic bytes" followed by a series of "chunks."

Magic bytes can be found in almost every file format. They are used to identify file types quickly, and in the case of PNGs, also serve to quickly identify common forms of data corruption. If a PNG's magic bytes are malformed, the rest of the file will be as well. 

Following the magic bytes is a series of "chunks." "Chunk" in this context refers to a pre-defined set of bytes that can include things like the image's dimensions, free-form text, the actual pixel data of the PNG file, or other metadata. The actual structure and names of these chunks is discussed more later in this article.

<!-- The PNG begins with a PNG file's magic bytes followed by a header chunk and ending with an EOF chunk. -->

<!-- , followed by the IHDR (image header) chunk, 1 or more IDAT (image data) chunks, and ending with an IEND (image end) chunk. -->

### Magic bytes

![image](https://user-images.githubusercontent.com/39542938/204681347-5fe0f6fb-fa8d-4a20-a165-de4ee66057b8.png)

The magic bytes of a PNG file are used to quickly determine whether or not the file is in fact a PNG as well as easily identify common forms of file corruption. The magic bytes are the same for every PNG file and do not contain any information about the image itself. The image above depicts the magic bytes in their ASCII representation. In other formats the magic bytes are:

```
(decimal) 137 80 78 71 13 10 26 10
(hex) 0x89 0x50 0x4e 0x47 0x0d 0x0a 0x1a 0x0a
(binary) 0b1000_1001 0b0101_0000 0b0100_1110 0b0010_00111 0b0000_1101 0b0000_1010 0b0001_1010 0b0000_1010
```

The first byte (`\x89`) is not ASCII, as noted by its most significant bit being `1`. This can identify corruption related to non-ASCII input.

The following three bytes (`PNG`) are useful in determining the file format when reading the file in a textual format.

`\r\n` and `\n`, respectively DOS and UNIX line endings are useful in detecting a very common form of corruption that occurs when transferring files between UNIX systems and Windows machines.

`\x1a`, or the ctrl+z character, is said to have special behavior on MS-DOS systems. The exact behavior is [a bit contentious](http://jdebp.info/FGA/dos-character-26-is-not-special.html), but in general this character being present prevents users from accidentally printing binary content to their terminal on such systems.

## Chunks

Chunks are split into 2 classes: required (critical) and optional (ancillary). The required cb

### General Chunk Structure

### Chunk Naming Convention

## Critical Chunks

### IHDR

#### Pixel representations

### IDAT

### IEND

## Ancillary Chunks

## Pixel compression