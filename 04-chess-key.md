# A Vigenère Key Hidden in a Chess Game

**Subtitle:** The cipher was the easy part. The real challenge was finding which 2026 Magnus Carlsen game held the key.

Some crypto challenges hide the hard work inside the maths. This one hid it in the real world. "Chess Key" was a high-difficulty **Cryptography / OSINT** challenge, and the Vigenère decryption at the end was almost an afterthought — the actual puzzle was reconstructing the key from a specific chess game you had to go and find.

## The clue

You start with a short ciphertext:

```
nZ3ipU8_yy3p5_hvyKi0
```

A forum thread titled "Carlsen gave me an idea" laid out the rules:

- An 8×8 chessboard acts as a letter-lookup layer. A provided image, `chesskey.png`, maps each square to a letter.
- The Vigenère key comes from **White's moves only**, taken from a 2026 Magnus Carlsen game that opens with the English Opening (1. c4).
- Digits and underscores in the ciphertext are left untouched.

From the board image, the square-to-letter mappings I needed were: c4=L, g3=S, g2=G, f3=F, h4=I, f5=W, c3=R, d3=X, g4=F, e4=E, h6=A, g1=V, d4=P.

## The OSINT part

The thread told me *how* to build the key but not *which* game. The constraints were specific enough to search on, though: a 2026 Carlsen game, English Opening, 1. c4. That pointed to **Carlsen vs. Aleksandar Indjic, 13 January 2026**. White's opening moves:

```
1. c4   2. g3   3. Bg2   4. Nf3   5. Nh4   6. Nf5   7. Nc3
8. d3   9. g4   10. Bxe4  11. Nxh6  12. Rg1  13. d4 ...
```

Reading each move's destination square through the board map gives the key letters:

```
c4 g3 g2 f3 h4 f5 c3 d3 g4 e4 h6 g1 d4
 L  S  G  F  I  W  R  X  F  E  A  V  P
```

So the Vigenère key is `LSGFIWRXFEAVP`.

## Decrypting

With the key in hand, it's a standard Vigenère decrypt — subtract the key letter from the cipher letter, mod 26 — while leaving digits and underscores alone and keeping the original letter case:

```
word 1  nZ3ipU8 :  n-L=c  Z-S=H  i-G=c  p-F=k  U-I=M   -> cH3ckM8
word 2  yy3p5   :  y-W=c  y-R=h  p-X=s                 -> ch3s5
word 3  hvyKi0  :  h-F=c  v-E=r  y-A=y  K-V=P  i-P=t    -> cryPt0
```

Joining the three parts gives the flag:

```
deadface{cH3ckM8_ch3s5_cryPt0}
```

## Why it works

Underneath, this is just a Vigenère cipher. What makes it a high-difficulty challenge is that the key isn't a word you can guess — it's encoded as chess moves, and you can only reconstruct it by doing three separate things correctly: recognising the cipher, running the OSINT to identify the exact game, and applying the board-to-letter lookup without slipping. Miss any one and the key is wrong. Case was preserved on purpose so the mixed-case flag (`cH3ckM8`) would survive the decrypt.

## What I took from it

The biggest lesson here was that "crypto" challenges aren't always about the crypto. A key can live in a public, searchable data source, and the game is finding it. A single datable clue — a 2026 Carlsen game with the English Opening — is enough to narrow an OSINT search fast if you treat the constraints as search terms. And a small practical habit paid off: when a flag is clearly mixed-case and full of leetspeak, preserve case and leave non-alphabet characters untouched, or you'll mangle a correct answer at the last step.

---

**Medium excerpt:** The cipher in this DeadFace CTF challenge was a plain Vigenère — but the key was buried in a specific 2026 Magnus Carlsen game you had to track down first. Here's the full chain from chessboard to flag.

**Tags:** Cybersecurity · Capture The Flag · Cryptography · OSINT · Chess

**Cover image concept:** A chessboard overlaid with letters on the squares, a move arrow (c4) pointing to its mapped letter (L), and a faint Vigenère grid behind it. Ties the three ideas (chess, OSINT, cipher) together in one frame.
