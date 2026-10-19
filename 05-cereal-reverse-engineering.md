# Reversing a Windows Crackme: Custom Base64, SHA-1 and RC4

**Subtitle:** Why you can't just patch the password check out of this one — and how reading the binary gave up the real key.

This was my first proper reverse-engineering challenge of DeadFace CTF 2026, and it taught me a lesson that a lot of beginners (me included) have to learn the hard way: sometimes bypassing the check gets you nowhere, because the check *is* the key.

The target was a 32-bit Windows binary, `re01-win-x86.exe`, that asks for a password. The brief even gave an example string, `deadface{I-For-One-Welcome-Our-New-Buggy-Overlords}`, but made a point of saying that was a *format sample*, not the real flag.

## Getting oriented with strings

Before opening a disassembler, I ran `strings` and read the symbol names. That alone sketched the whole architecture:

```
Please enter the password:
Flag decode error
_base64_decode
_ENCODED_HASH_UTF16
_SHA1Init / _SHA1Update / _SHA1Final
_base64_UTF16
re01.c
```

So the program hashes your password with SHA-1, compares it against a stored hash, and uses something base64-related to decode the flag. That combination usually means the password isn't just checked — it's also the decryption key, so the flag never exists in cleartext inside the file.

## Confirming it in Ghidra

I loaded the binary into Ghidra. The real logic lived in the compiler's startup code rather than a tidy `main`, which threw me for a moment, but following the cross-references from the "Please enter the password" string led straight to it. The flow was:

1. Read the password.
2. SHA-1 it and format the digest as a 40-character hex string.
3. `strcmp` that hex against a stored value decoded from `.data`.
4. On a match, RC4-decrypt the flag using the password as the key.

That step 4 is the trap. If you patch step 3 to always say "access granted," step 4 still runs RC4 with *your wrong password*, so the flag decrypts to garbage. The only way through is the real password.

## The custom base64 alphabet

The stored hash and the flag ciphertext were base64, but standard base64 gave nonsense. Reading the binary showed why: a custom alphabet labelled `_base64_UTF16`, starting with `Z`. I pulled the full 64-character alphabet straight out of the file:

```
ZKj9n+yf0wDVX1s/5YbdxSo=ILaUpPBCHg8uvNO4klm6iJGhQ7eFrWczAMEq3RTt2
```

Decoding the stored hash with that alphabet (a manual 6-bits-per-symbol decode) produced a clean 40-character SHA-1 digest:

```python
custom = 'ZKj9n+yf0wDVX1s/5YbdxSo=ILaUpPBCHg8uvNO4klm6iJGhQ7eFrWczAMEq3RTt2'
enc = '1uH71dHc1FggXcIML9pcId1vXo1gXdwvXuK8XO5cIoIMIO5FLu0MXZ22'
idx = {c: i for i, c in enumerate(custom)}
bits = ''.join(format(idx[c], '06b') for c in enc)
out = bytes(int(bits[i:i+8], 2) for i in range(0, len(bits)//8*8, 8))
print(out)   # 68158678a3f9d76a3d1ca12d20b2d6af9bd3f290
```

## Cracking the hash

That SHA-1 digest cracked instantly on CrackStation to `cookiecrisp` — the "favourite cereal" the challenge name hinted at. Running the binary and entering it gave:

```
Access granted
Good job! You are ALMOST there... : https://cerealkiller.lyttonlabs.org/unwanted-guests/ick.png
```

The final flag was rendered inside that `ick.png` image, which I read and submitted. I'll be honest here: I didn't copy the exact flag string down at the time, so I'm marking it as **not recorded** rather than reconstructing it from memory. The password recovery — `cookiecrisp` — is the part that mattered and is fully verified.

## Why it works

The design is deliberately patch-proof. The password is both the authentication secret (through its SHA-1) and the RC4 decryption key, so forcing the comparison to pass doesn't help — you'd decrypt with the wrong key and get noise. The custom base64 alphabet was the real speed bump: it reuses the base64 symbol set in a shuffled order, and even drops `=` into the middle, which is exactly why an off-the-shelf decoder fails and quietly hands you garbage instead of an error.

## What I took from it

Strings and symbol names are the cheapest recon you can do, and here they revealed the entire SHA-1 / RC4 / base64 structure before I disassembled anything. When a password doubles as a decryption key, there's no shortcut — you have to recover the real value. And don't trust a "base64" string just because it decodes to *something*; if the output is garbage, suspect a custom alphabet and pull it from the binary. Last practical note: if you can't find `main`, the compiler may have fused it into startup code — follow cross-references from a known string instead of hunting for a function that isn't there.

---

**Medium excerpt:** You can't patch your way past this DeadFace CTF crackme — the password is also the decryption key. Here's how reading the binary gave up a custom base64 alphabet, a SHA-1 hash, and the real password.

**Tags:** Cybersecurity · Capture The Flag · Reverse Engineering · Ghidra · Infosec

**Cover image concept:** A Ghidra decompiler pane with the SHA-1 → strcmp → RC4 flow visible, or a clean diagram "password → SHA-1 → check → RC4 → flag" showing the password feeding both the check and the decrypt. Signals RE without being a generic hacker stock photo.
