---
title: "Almost Booting the Gameboy With a Custom Logo"
date: 2017-08-27T09:18:25+02:00
draft: false
---

In 2003 neviksti managed to extract the original Gameboy boot ROM by [literally putting the CPU under a microscope](http://www.neviksti.com/DMG/). The ROM on the chip was soon decoded, revealing the [bootstrap program](http://gbdev.gg8.se/wiki/articles/Gameboy_Bootstrap_ROM) responsible for reading and parsing the header of the game cartridge. The program is pretty simple; it reads the header stored on the cartridge, validates it, scrolls the Nintendo logo and plays the di-ding sound. If the header is valid it then starts the program at the entry point. A side-effect of this process allows anyone with a hex editor and too much time on their hands to change the appearance of the Nintendo logo.

## The Header

This is the header from a game cartridge, starting at offset 0100:

```text
0100 : 00 C3 50 01 CE ED 66 66 CC 0D 00 0B 03 73 00 83
0110 : 00 0C 00 0D 00 08 11 1F 88 89 00 0E DC CC 6E E6
0120 : DD DD D9 99 BB BB 67 63 6E 0E EC CC DD DC 99 9F
0130 : BB B9 33 3E 53 55 50 45 52 20 4D 41 52 49 4F 4C
0140 : 41 4E 44 00 00 00 00 01 01 00 00 01 01 9D 5E CF
```

- 0100-0103 is the entry point for the program stored on the cartridge. This is almost always `00 C3 50 01`, which translates to a `NOP` followed by a `JP 0150h` (Where the address is stored as LL HH, `50 01`).
- 0104-0133 contains a "secret" validation code.
- 0134-014C contains information about the cartridge and the program on it (for instance the title is located at offset 0134-0143).
- 014D contains an 8-bit checksum of the header bytes at 0134-014C. The bootloader validates this checksum.
- 014E-014F is a 16-bit checksum of the entire ROM. This checksum is not validated by the bootloader.
- The main program the starts at offset 0150.

More information about the header can be found [here](http://gbdev.gg8.se/wiki/articles/The_Cartridge_Header)

## The Validation Code

Let's look at the validation code at 0104-0133:

```text
0104 : CE ED 66 66 CC 0D 00 0B 03 73 00 83 00 0C 00 0D
0114 : 00 08 11 1F 88 89 00 0E DC CC 6E E6 DD DD D9 99
0124 : BB BB 67 63 6E 0E EC CC DD DC 99 9F BB B9 33 3E
```

This code is the same for all Gameboy games and has to be present or the bootloader will hang. However, the bootloader will run all the way to the di-ding even if everything is invalid, so we can create a ROM filled with all bytes set to `00`  and it will still do *something*. We need at least 336 (014F) bytes or the ROM won't boot at all, since part of the header would be missing.

Here's our rom:

```text
0000 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
...  : ...
0100 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0110 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0120 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0130 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0140 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

And here is what it looks like in BGB:

![DMG-all-00](/images/blog/almost-booting-the-gameboy-with-a-custom-logo/DMG-all-00.webp)

Where did the logo go? It has to be affected by the validation code, right? What happens if we change the code to all `FF`?

Here's our new ROM:

```text
0000 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
...  : ...
0100 : 00 00 00 00 FF FF FF FF FF FF FF FF FF FF FF FF
0110 : FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF
0120 : FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF
0130 : FF FF FF FF 00 00 00 00 00 00 00 00 00 00 00 00
0140 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

And here it is in BGB:

![DMG-all-FF](/images/blog/almost-booting-the-gameboy-with-a-custom-logo/DMG-all-FF.webp)

The validation code isn't just affecting the logo, it *is* the logo (surprise!). Now we need to figure out how to decode it.

# Decoding the Logo

The logo is 48x8 pixels and monochrome. Each "pixel" is actually a block of four dots on the matrix. The copyright logo is drawn separately and cannot be altered. The area inside the red markings is our canvas.

![DMG-Logo](/images/blog/almost-booting-the-gameboy-with-a-custom-logo/DMG-Logo.webp)

Our code 48 bytes long giving us, you guessed it, 48x8 bits in total. Each bit should correspond to a pixel. Let's assume pixels are added left to right, top to bottom. Think of our 48-byte code as a 48x8 bit matrix. By setting the most significant bit to 1 the top right pixel should come on.

![DMG-firstbit](/images/blog/almost-booting-the-gameboy-with-a-custom-logo/DMG-firstbit.webp)

Fantastic! Our assumption keeps working until we flip the fifth bit. Then this happens:

![DMG-fifthbit](/images/blog/almost-booting-the-gameboy-with-a-custom-logo/DMG-fifthbit.webp)

Each nibble corresponds to a row of 4 pixels added top to bottom. But when we reach the third byte this happens.

![DMG-17thbit](/images/blog/almost-booting-the-gameboy-with-a-custom-logo/DMG-17thbit.webp)

The first nibble of the third byte is added to the right of the first block. This pattern repeats for the first 24 bytes (top half of the logo) and then repeats (lower half of the logo). Here's a representation of how the upper/lower nibbles of the 48 bytes in the header end up forming the 8 rows of the logo:

```text
    1   3   5   7   9   11  13  15  17  19  21  23
    --  --  --  --  --  --  --  --  --  --  --  --
1 | c-  6-  c-  0-  0-  0-  0-  0-  0-  1-  8-  0-
2 | -e  -6  -c  -0  -3  -0  -0  -0  -0  -1  -8  -0

    2   4   6   8   10  12  14  16  18  20  22  24
    --  --  --  --  --  --  --  --  --  --  --  --
3 | e-  6-  0-  0-  7-  8-  0-  0-  0-  1-  8-  0-
4 | -d  -6  -d  -b  -3  -3  -c  -d  -8  -f  -9  -e

    25  27  29  31  33  35  37  39  41  43  45  47
    --  --  --  --  --  --  --  --  --  --  --  --
5 | d-  6-  d-  d-  b-  6-  6-  e-  d-  9-  b-  3-
6 | -c  -e  -d  -9  -b  -7  -e  -c  -d  -9  -b  -3

    26  28  30  32  34  36  38  40  42  44  46  48
    --  --  --  --  --  --  --  --  --  --  --  --
7 | c-  e-  d-  9-  b-  6-  0-  c-  d-  9-  b-  3-
8 | -c  -6  -d  -9  -b  -3  -e  -c  -c  -f  -9  -e
```

Armed with this knowledge and a healthy dose of python we should be able to extract the Nintendo logo from the header.

```python
from PIL import Image

header = bytes.fromhex(
    "ce ed 66 66 cc 0d 00 0b 03 73 00 83 "
    "00 0c 00 0d 00 08 11 1f 88 89 00 0e "
    "dc cc 6e e6 dd dd d9 99 bb bb 67 63 "
    "6e 0e ec cc dd dc 99 9f bb b9 33 3e "
)

logo = bytes(
    # Combine nibbles of every other byte 
    header[a + b + d] << c & 0xf0 | header[a + b + d + 2] >> 4 - c & 0x0F
    for a in (0, 24)  # Upper/lower 24 bytes
    for b in (0, 1)  # Even/odd bytes
    for c in (0, 4)  # Upper/lower nibbles
    for d in range(0, 21, 4)  # 6 bytes (48 bits) per row
)

Image.frombytes("1", (48, 8), logo).save("logo.bmp")
```

![DMG-Logo-Decoded](/images/blog/almost-booting-the-gameboy-with-a-custom-logo/DMG-Logo-Decoded.webp)

It works!

# Encoding a Logo

Now that we can decode a logo, encoding our own logo should just be a matter doing the same process in reverse. This is the logo I want to encode:

![DMG-mylogo](/images/blog/almost-booting-the-gameboy-with-a-custom-logo/DMG-mylogo.webp)

```python
from PIL import Image

logo = Image.open('mylogo.bmp').tobytes()

header = bytes(
    # Combine nibbles of every 6th byte (i.e. 1 row down)
    logo[a + b + d] << c & 0xF0 | logo[a + b + d + 6] >> 4 - c & 0x0F
    for a in (0, 24)  # Upper/lower 24 bytes
    for b in range(0, 6)  # 6 bytes (48 bits) per row
    for c in (0, 4)  # Upper/lower nibbles
    for d in (0, 12)  # Neighbor bytes in he header are 12 bytes apart in the logo
)

# Generate a valid gameboy rom by padding with 0x00 to fill out the header
with open('mylogo.gb', 'wb') as f:
    f.write(bytes(260) + header + bytes(28))
```

Let's check if it works. Here's the result in BGB:

![DMG-dodslaser](/images/blog/almost-booting-the-gameboy-with-a-custom-logo/DMG-dodslaser.webp)

BGB will complain that pretty much everything is broken in this ROM, and say that it would not play on a Gameboy. While this is true, the logo will still be scrolled, even on real hardware. This works because the logo is actually read twice by the bootloader. Once to be scrolled, and once again to be validated. [Some pirate gamecarts abuse this fact](http://fuji.drillspirits.net/?post=87) by replacing the logo in the header after it is read the first time, making a custom logo scroll while the header still passes validation. [This post on dhole's blog](https://dhole.github.io/post/gameboy_custom_logo/) shows how to do it with a homebrew cart emulator.

Implementing a fully booting pirate gameboy cartridge is beyond the scope of this project (hence "almost"). For now I'm happy to see my custom logo being scrolled on a Gameboy.

![DMG-hw](/images/blog/almost-booting-the-gameboy-with-a-custom-logo/DMG-hw.webp)

***UPDATE 2022-05-21** Rewrote most of the code to be more readable*