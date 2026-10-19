# The Flag Was Hiding Below the Picture

**Subtitle:** A DeadFace CTF challenge where no stego tool found anything — because the flag wasn't hidden *in* the image, it was cropped *out* of it.

I spent the first few minutes of this one running every steganography tool I could think of, and all of them came back with nothing. That emptiness turned out to be the actual hint.

This was "Keep Digging," an easy-to-medium **Steganography / File Manipulation** challenge from DeadFace CTF 2026. You get a single file, `keep_digging.jpg` — a picture of a man digging straight down into the ground, with the words "Keep Digging" on it. The flag is somewhere in that image. That's the whole brief.

## Where I started

The obvious first move with any image challenge is to check for an embedded payload, so I reached for the usual tools:

```bash
steghide extract -sf keep_digging.jpg
exiftool keep_digging.jpg
```

Neither gave me anything useful. No hidden archive, no suspicious metadata field, nothing in the comments. At that point a lot of people (me included, at first) start second-guessing the tools. But a clean result is a result: if nothing is embedded, the flag probably isn't embedded. It's hidden some other way.

So I went back to the picture itself. A man digging *downward*, and a title that says *keep digging*. Read literally, that's pointing at the bottom of the image. And there's a classic trick that fits: if a JPEG's header says the image is shorter than it really is, most viewers only draw the top part and quietly drop everything below. The pixels are still in the file. You just can't see them.

## Finding the height in the header

To test that, I opened the file in a hex editor and went looking for the frame header. This was a progressive JPEG, so the marker I wanted was **SOF2** — the bytes `FF C2`. I found it at offset 171 (`0xAB`):

```
FF C2 00 11 08  02 9E  01 C0 ...
                 ^^^^^  ^^^^^
                height  width
```

The two bytes right after the precision byte are the height, and the next two are the width:

```
02 9E  =  670 px  (height)
01 C0  =  448 px  (width)
```

So the file was telling every viewer "I'm 670 pixels tall." If the real content went lower than that, those extra rows were being hidden by that one number.

## Making the image taller

The fix is to overwrite the height with something larger so the decoder draws the full canvas. I bumped it from `02 9E` (670) up to `03 84` (900) with a short Python script. The height bytes sit at offset 176–177:

```python
with open('keep_digging.jpg', 'rb') as f:
    data = bytearray(f.read())

# overwrite the two height bytes (offset 176-177) with 900
data[176:178] = (900).to_bytes(2, 'big')

with open('keep_digging_fixed.jpg', 'wb') as f:
    f.write(data)
```

I opened `keep_digging_fixed.jpg`, and there it was — the flag text drawn into the dirt below where the original image used to stop:

```
deadface{D1g_d0wN_D33p!}
```

## Why this works

A JPEG carries its real pixel dimensions inside the SOF segment, and decoders simply trust that number instead of measuring the image data. Shrink the stored height and the viewer stops rendering early, so anything below the line disappears from view without a single byte being deleted. The image data is all still there. Restoring the height (or, like here, over-stating it) forces the decoder to paint the rest, flag included.

It's a neat reminder that an image file is a container with instructions, not just a bitmap. Change the instructions and you change what you're allowed to see.

## What I took from it

A few things stuck with me after this one:

When your tools come back empty, that's data too — it tells you the flag isn't where you first looked, so stop re-running the same scan and change your assumption. Image and file-format headers (dimensions, offsets, length fields) are a genuine hiding spot, and a hex editor plus a two-byte edit is often the entire exploit. And the challenge art was a literal instruction the whole time: "keep digging, down" was describing the image geometry, not being cute.

Not a hard challenge, but a good one to start a writeup series on, because it teaches a habit that keeps paying off: when the obvious approach fails, question the container, not just the contents.

---

**Medium excerpt:** Sometimes a flag isn't encrypted or embedded — it's just cropped out of view. Here's how a two-byte edit to a JPEG's height header brought a hidden DeadFace CTF flag back into the picture.

**Tags:** Cybersecurity · Capture The Flag · Steganography · Digital Forensics · Infosec

**Cover image concept:** A single image splitting at a horizontal "crop line" — the top half a normal photo, the bottom half emerging from darkness with faint text showing through, as if the picture continues past where it was cut. Alternative: a hex-editor view with the `FF C2 ... 02 9E` bytes highlighted.
