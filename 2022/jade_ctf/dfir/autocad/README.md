# AutoCAD

## Challenge

You were given the task to design a poster for an event by the
organizer. Mid-designing, you took a tea break, and when you came back,
you saw that someone has defaced your work. You need to fix it quickly
before the organizer finds out about it!

<span id="Flag Format"></span>**Flag Format**: jctf{.\*}

## Solution

We get a `poster.png` file that every image viewer I tried refuses to
open.

`file` claims its dimensions are 0x0:

    autocad$ file poster.png
    poster.png: PNG image data, 0 x 0, 8-bit/color RGBA, non-interlaced

I found a tool called `pngcheck` which also complains about the
dimensions:

    autocad$ pngcheck poster.png
    poster.png  invalid IHDR image dimensions (0x0)
    ERROR: poster.png

It seems we have to somehow figure out how to fix whatever part of the
file contains the dimension information.

According to
[Wikipedia](https://en.wikipedia.org/wiki/Portable_Network_Graphics), a
PNG file is divided into chunks. Some of these chunks are *critical* and
others are *ancillary*. The critical ones are obligatory and must always
be in the file. Here's how a chunk looks like:

| Description                           | Size           |
|---------------------------------------|----------------|
| Length of the chunk                   | 4 bytes        |
| Chunk type, ASCII                     | 4 bytes        |
| Chunk data                            | *Length* bytes |
| CRC32 checksum over the type and data | 4 bytes        |

One of the critical chunks is the `IHDR` chunk, which must be the first
one in the file. Here's how it looks like:

| Description        | Size    |
|--------------------|---------|
| Width              | 4 bytes |
| Height             | 4 bytes |
| Bit depth          | 1 byte  |
| Color type         | 1 byte  |
| Compression method | 1 byte  |
| Filter method      | 1 byte  |
| Interlace method   | 1 byte  |

Cool, so the width and height are stored in the `IHDR` chunk, which is
the first one in the file. We just need to set the relevant bytes to the
correct dimensions of the image. I used `xxd` to dump the contents of
the file:

    00000000: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452  .PNG........IHDR
    00000010: 0000 0000 0000 0000 0806 0000 0017 63ad  ..............c.
    <snip>

On the second line of the dump we can see 8 null bytes after the `IHDR`
ASCII value. These are the width and height, which are set to zero as
expected.

To figure out the correct dimensions, I took advantage of the checksum
field described previously. If the dimensions are correct, the checksum
over the type and data should match the checksum of the chunk. The
checksum is the value `0x1763adc8` (392408520 in decimal), located at
the end of the chunk

    <snip>
    00000010: 0000 0000 0000 0000 0806 0000 0017 63ad  ..............c.
    00000020: c800 0000 0173 5247 4200 aece 1ce9 0000  .....sRGB.......
    <snip>

With the correct checksum, I wrote a script to brute force the correct
dimensions. By the size of the file, I guessed it probably wasn't bigger
than 1000x1000, so I only tried values within that range.

    #!/usr/bin/env python3

    from binascii import crc32

    CHUNK_TYPE = b"IHDR"
    REST = b"\x08\x06\x00\x00\x00"
    CHECKSUM = 392408520

    for width in range(1, 1001):
        width = width.to_bytes(4, "big")
        for height in range(1, 1001):
            height = height.to_bytes(4, "big")
            chunk_data = CHUNK_TYPE + width + height + REST
            crc = crc32(chunk_data)
            if crc == CHECKSUM:
                print(f"Width: {int.from_bytes(width, 'big')}\nHeight: {int.from_bytes(height, 'big')}")
                exit()

    autocad$ ./solve.py
    Width: 800
    Height: 450

With the correct dimensions, I used `xxd` to make the fixed version of
the image.

    00000000: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452  .PNG........IHDR
    00000010: 0000 0320 0000 01c2 0806 0000 0017 63ad  ..............c.
    00000020: c800 0000 0173 5247 4200 aece 1ce9 0000  .....sRGB.......

The final image:

![fixed.png](img/fixed.png)

At the top left corner we can see the flag.

Flag: `jctf{st3g_1s_easy_as_f**k}`
