# Breaking Single-Byte XOR Without Brute Force

**Subtitle:** A DeadFace CTF crypto warm-up that you can solve by hand, if you know which byte to look at first.

Single-byte XOR is one of the first things you meet in CTF cryptography, and "X Marks the Spot" is about as clean an example as you'll find. The nice part is that you don't have to brute-force all 256 keys. The ciphertext tells you the key if you know what to look for.

This was an easy **Cryptography / XOR** challenge. The title is the giveaway: an X usually means XOR. You're handed one hex string and nothing else:

```
02362b2964302c2164372c25202b3337642b226417342127303621641021272c2a2b282b233d7e6420212520222527213f3c2b361b751b263d30771b70301b701b3075297739
```

## Reading the ciphertext before decrypting it

Before writing any code, I looked at the bytes. One of them, `64`, keeps showing up, and it tends to sit between groups of other bytes — right where you'd expect spaces between words in a sentence.

That's the whole crack. In normal English text the most common character by far is the space, `0x20`. If this is a single-byte XOR of an English sentence, then the most common ciphertext byte is almost certainly an encrypted space. XOR is its own inverse, so recovering the key from that one assumption is a single operation:

```
key = 0x64 XOR 0x20 = 0x44   →  ASCII 'D'
```

## Decrypting with the key

With the key guessed, decrypting the rest is three lines of Python:

```python
hex_str = "02362b2964302c2164372c25202b3337642b226417342127303621641021272c2a2b282b233d7e6420212520222527213f3c2b361b751b263d30771b70301b701b3075297739"
raw = bytes.fromhex(hex_str)

print(bytes(b ^ 0x44 for b in raw).decode())
```

And it comes straight out as readable text:

```
From the shadows of Spectre Technology: deadface{xor_1_byt3_4t_4_t1m3}
```

```
deadface{xor_1_byt3_4t_4_t1m3}
```

## Why the space trick works

XOR with a single repeating byte is weak for one specific reason: it preserves the frequency structure of the plaintext. Every space in the message becomes the same ciphertext byte, every 'e' becomes another fixed byte, and so on. So the statistics of English survive the encryption. The most frequent ciphertext byte lines up with the most frequent plaintext character, which in ordinary prose is the space.

Once you assume "most common byte = space," you get the key for free, and because XOR is symmetric, the same key both encrypts and decrypts. No brute force, no wordlist — just one observation about how often a byte appears.

## What I took from it

Even the flag text, `xor_1_byt3_4t_4_t1m3`, is basically the lesson: attack one byte at a time. The habit worth keeping is to look at ciphertext before you automate anything. A repeated byte in an XOR challenge is nearly always an encrypted space, and frequency analysis will beat brute force any time the cipher leaves the plaintext's structure intact. For a stronger scheme you'd move to a repeating multi-byte key and tools like `xortool`, but the thinking starts here.

---

**Medium excerpt:** You don't need to brute-force single-byte XOR. If you can spot the encrypted space in the ciphertext, the key falls out in one step — here's how I solved a DeadFace CTF crypto challenge by hand.

**Tags:** Cybersecurity · Capture The Flag · Cryptography · XOR · Infosec

**Cover image concept:** A line of hex with one repeating byte (`64`) highlighted across the string, and a small "⊕ 0x20 = key" annotation. Clean monospace-on-dark look that signals "crypto" without stock-photo padlocks.
