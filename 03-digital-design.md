# A Cipher Built Out of Logic Gates

**Subtitle:** A DeadFace CTF puzzle that turns AND, OR, XOR and NOT into a tiny decoding machine.

Most crypto challenges reuse a known cipher. This one invented its own, and the "key" was a column of logic-gate operations. "Digital Design" was a medium **Cryptography / Digital Logic** challenge, and solving it is really about modelling the scheme correctly before you write a line of code.

## The clue

The challenge came from a forum post by a DeadFace member called lilith, with two parallel blocks. First, a set of binary groups:

```
001100000 10110011 001110101 100111010 010101000
101111101 100101100 00001100 10001010 100000010 0100001110
```

And second, a matching set of letter groups made only of X, O, N and A:

```
XXONO NXNOA OAOON NXXAO NAAXO XANXO ANOXA ANOXN NONOX ANXAX AOAAX
```

Those letters are logic gates: **A = AND, O = OR, X = XOR, N = NOT**. Each binary group pairs with one gate group. AND, OR and XOR each take two input bits; NOT takes one. If you walk the gates across the bits, each group collapses to a short binary value, and A1Z26 (1→A, 2→B, …, 26→Z) turns that into a letter.

## Working one group by hand

Take the first group, `001100000`, with gates `XXONO`. You consume bits from the binary string as each gate demands:

```
X on (0,0) -> 0
X on (1,1) -> 0
O on (0,0) -> 0
N on (0)   -> 1
O on (0,0) -> 0
```

That leaves `00010`, which is 2, which is **B**. Do that for all eleven groups and you spell out the flag word.

## Automating the decode

Rather than hand-crank eleven groups, I scripted it. The key detail is reading the right number of bits per gate — two for AND/OR/XOR, one for NOT:

```python
bins = "001100000 10110011 001110101 100111010 010101000 101111101 100101100 00001100 10001010 100000010 0100001110".split()
gates = "XXONO NXNOA OAOON NXXAO NAAXO XANXO ANOXA ANOXN NONOX ANXAX AOAAX".split()

flag_text = ""
for b, g in zip(bins, gates):
    idx = 0
    bits = ""
    for op in g:
        if op == 'N':
            bits += str(1 - int(b[idx])); idx += 1
        else:
            x, y = int(b[idx]), int(b[idx+1]); idx += 2
            if op == 'A': bits += str(x & y)
            elif op == 'O': bits += str(x | y)
            elif op == 'X': bits += str(x ^ y)
    flag_text += chr(ord('A') + int(bits, 2) - 1)

print(f"deadface{{{flag_text}}}")
```

The decoded word came out as `BINARYLOGIC`:

```
deadface{BINARYLOGIC}
```

## Why it works

There's no secret key here in the cryptographic sense. The scheme is a small bit-level machine: each gate letter is an instruction that reads one or two bits and writes a single bit, the written bits form a 5-bit number, and A1Z26 maps 1–26 onto the alphabet. The only thing that trips people up is bit alignment — if you feed NOT two bits or AND one, the whole group shifts and every letter after it comes out wrong. Get the input widths right and it decodes cleanly.

## What I took from it

Custom encodings look intimidating until you describe them in plain terms: "read N bits, apply this operation, emit the result." Once I wrote that sentence, the code wrote itself. A1Z26 is also worth keeping in your back pocket — it shows up constantly as the final layer of multi-stage puzzles, so when you end up with small numbers between 1 and 26, try mapping them to letters before assuming you've gone wrong.

---

**Medium excerpt:** This DeadFace CTF challenge didn't use a known cipher — it built one out of AND, OR, XOR and NOT gates. Here's how I modelled it as a tiny bit-machine and decoded the flag.

**Tags:** Cybersecurity · Capture The Flag · Cryptography · Logic · Infosec

**Cover image concept:** A row of logic-gate symbols (AND/OR/XOR/NOT) feeding bits into letters, or a clean diagram of `001100000 + XXONO → 00010 → B`. Reinforces the "bits become letters" idea.
