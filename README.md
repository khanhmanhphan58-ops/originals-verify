# Originals · Provably Fair Verifiers

Static, dependency-free verifier pages for Originals games. Each page recomputes a finished round from the revealed seeds entirely in the browser (Web Crypto); nothing is sent to a server.

| Game | ID | Page |
|---|---|---|
| Stellar Rush | 26001 | `stellar_rush.html?s=<serverSeed>&c=<clientSeed>&n=<nonce>&d=<meteors>` |
| Tower Legend | 26002 | `tower_legend.html?s=<serverSeed>&c=<clientSeed>&n=<nonce>&d=<difficulty 0-4>` |
| Plinko | 26003 | `plinko.html?s=<serverSeed>&c=<clientSeed>&n=<nonce>&r=<rows 8-16>&k=<risk 1-3>&b=<bias 0-2>` |
| Bingo | 26004 | `bingo.html?s=<serverSeed>&c=<clientSeed>&n=<nonce>&cards=<card>_<card>…` (each card: 25 comma-separated numbers row by row, centre 0; optional) |

## Stellar Rush algorithm

- `H(x)` = SHA-256 of the UTF-8 text `x`, as lowercase hex. Seeds and hashes are used as hex text, never decoded to bytes.
- Before the bet the game commits to `H(serverSeed)`.
- `mac = HMAC-SHA256(key = serverSeed, message = clientSeed + ":" + nonce)`.
- Base deck: `7,2,19,25,1,13,5,24,14,6,15,9,22,16,3,17,18,20,8,21,4,12,10,23,11`.
- `S(items, seed)`: `h = H(seed)`; the item at input index `i` gets key `h[i:] + h[:i]`; stable ascending sort by key.
- `order = S(S(base, mac), H(mac))`; the first K numbers are the meteor planets (K = 1, 3, 5, 10).
- The round advances planet 1 → 25 and crashes at the smallest meteor number on the path.

Test vector (checked on every page load): server seed = 64 × `0`, client seed `player`, nonce 1 →
hash `60e05bd1…6250f55`, HMAC `a5cc9801…eebea5100`, order `7 19 11 17 20 12 2 25 14 6 3 4 21 10 8 9 15 18 23 22 16 1 5 24 13`.

## Tower Legend algorithm

- Rules (columns / traps / rows): Easy 4/1/9, Medium 3/1/9, Hard 2/1/9, Extreme 3/2/6, Nightmare 4/3/6.
- Before the bet the game commits to `SHA256(serverSeed)` (lowercase hex of the UTF-8 text).
- For each row `r` (from 0, bottom row first): `mac = HMAC-SHA256(key = serverSeed, message = clientSeed + ":" + nonce + ":" + r)` as 32 bytes.
- Candidates `[1 … columns]`. For `j = 0 … columns − traps − 1`: `u` = big-endian uint32 of bytes `[4j, 4j+4)`, `index = floor(u × remaining / 2^32)`, remove that candidate; it is a safe tile.
- The removed candidates sorted ascending are the row's safe tiles; the rest are traps.
- Multiplier after `n` safe rows: `floor(98 × columns^n / (columns − traps)^n) / 100`.

Test vector (checked on every page load): server seed = 64 × `0`, client seed `player`, nonce 1, Easy →
row 0 HMAC `649dc01d…c9a6f9c5`, safe tiles `[2,3,4] [1,2,4] [1,3,4] [1,2,3] [1,2,4] [1,2,3] [1,2,4] [1,2,3] [1,2,4]`.

## Plinko algorithm

- Before the bet the game commits to `SHA256(serverSeed)`. Every ball, demo balls included, takes the next nonce of the seed pair, starting from 1.
- `mac = HMAC-SHA512(key = serverSeed, message = clientSeed + ":" + nonce)` as 64 bytes.
- Row `i` (0 … rows − 1) uses bytes `[4i, 4i+4)`: `x = b0/256 + b1/256² + b2/256³ + b3/256⁴`; right when `x > 0.5`, otherwise left.
- Slot = number of rights. Multiplier = pay table `[risk][rows][bias][slot]`; payout = amount × multiplier, truncated to 4 decimals.
- Risk 1 Regular, 2 High, 3 Nightmare; bias 0 Classic, 1 Left, 2 Right. Left doubles Classic on slots with `2 × slot < rows`, keeps the middle slot when rows is even, pays 0 elsewhere; Right mirrors Left.

Test vector (checked on every page load): server seed = 64 × `0`, client seed `player`, nonce 1, 16 rows →
HMAC `965f9445…522b9883`, path `1011000110010100`, slot 7.

## Bingo algorithm

- Before the bet the game commits to `SHA256(serverSeed)`. Every round, demo rounds included and whatever the number of cards, takes the next nonce of the seed pair, starting from 1.
- 20 balls are drawn without replacement from 1 … 51. Ball `i` (0 … 19): `h = HMAC-SHA256(key = serverSeed + ":" + i, message = clientSeed + ":" + nonce)` as lowercase hex; `start = int(h[0:15], 16) mod 47`; `num = int(h[start:start+14], 16) >> 3`; `index = floor(num × left / 2^53)` with `left = 51 − i`; the ball is the number at `index` of the remaining numbers in ascending order.
- A card is 5 × 5: 24 different numbers from 1 … 51 and a free centre (written as 0). 12 lines: rows, columns and both diagonals; a line wins when its 5 cells are all drawn.
- Multiplier per card by winning lines: 1 → 5, 2 → 50, 3 → 260, 4 → 520, 5 → 1000, 6+ → 10000, none → 0. Card payout = amount per card × multiplier, truncated to 4 decimals; the round pays the sum over its cards.
- The page floors the exact integer product. Verifiers that apply `parseInt` to the floating-point product differ only when the product is below 10⁻⁶.

Test vectors (checked on every page load): server seed = 64 × `0`, client seed `player` →
nonce 1 balls `15 19 18 13 16 21 7 34 51 28 3 4 17 25 46 20 10 14 39 24` (ball 0 HMAC `4d7fdabc…15dca4c6`), nonce 3 balls `36 21 30 46 28 45 12 16 34 1 50 39 38 6 5 32 44 13 29 42`; with nonce 1, card `15,19,18,13,16,1,2,5,6,8,9,11,0,12,22,23,26,27,29,30,31,32,33,35,36` wins row 1 (5×).
