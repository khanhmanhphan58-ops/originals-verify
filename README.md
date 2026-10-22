# Originals · Provably Fair Verifiers

Static, dependency-free verifier pages for Originals games. Each page recomputes a finished round from the revealed seeds entirely in the browser (Web Crypto); nothing is sent to a server.

| Game | ID | Page |
|---|---|---|
| Stellar Rush | 26001 | `stellar_rush.html?s=<serverSeed>&c=<clientSeed>&n=<nonce>&d=<meteors>` |

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
