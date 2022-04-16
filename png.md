# The PNG File Format

The PNG file format specifies a way to losslessly encode bitmaps. This blog post intends to describe, in-depth, all algorithms and ideas behind the PNG file format.

## Comparison to other formats

JPEG:
GIF:
WEBP:

## General structure

At a high level, a PNG file is composed of a series of chunks. This stream begins with a PNG file's magic bytes followed by a header chunk and ending with an EOF chunk.

<!-- , followed by the IHDR (image header) chunk, 1 or more IDAT (image data) chunks, and ending with an IEND (image end) chunk. -->

### Magic bytes

<!-- ASCII IMAGE -->

The magic bytes of a PNG file are used to quickly determine whether or not the file is in fact a PNG as well as easily identify common forms of file corruption. The magic bytes are the same for every PNG file and do not contain any information about the image itself. The image above depicts the magic bytes in their ASCII representation. In other formats the magic bytes are:

(decimal) 137 80 78 71 13 10 26 10
(hex) 0x89 0x50 0x4e 0x47 0x0d 0x0a 0x1a 0x0a
(binary) 0b1000_1001 0b0101_0000 0b0100_1110 0b0010_00111 0b0000_1101 0b0000_1010 0b0001_1010 0b0000_1010

The first byte (`\x89`) is not ASCII, as noted by its most significant bit being `1`. This can identify corruption related to non-ASCII input.

The following three bytes (`PNG`) are useful in determining the file format when reading the file in a textual format.

`\r\n` and `\n`, respectively DOS and UNIX line endings are useful in detecting a very common form of corruption that occurs when transferring files between UNIX systems and Windows machines.

`\x1a`, or the ctrl+z character, is said to have special behavior on MS-DOS systems. The exact behavior is [a bit contentious](http://jdebp.info/FGA/dos-character-26-is-not-special.html), but in general this character being present prevents users from accidentally printing binary content to their terminal on such systems.

## Chunks

Chunks are split into 2 classes: required (critical) and optional (ancillary). The required cb
