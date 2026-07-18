# FlagCasino

**Platform:** Hack The Box (CTF Try Out)  
**Category:** Reverse Engineering  
**Difficulty:** Very Easy  
**Concepts:** Binary Reversing, Pseudo-Random Number Generators (PRNG), `srand`/`rand` Determinism, Ghidra, ELF Analysis

## Objective
A compiled Linux program called `casino` is given. Running it starts a guessing "casino" game. There is no flag printed anywhere in the file. The goal is to read the program's logic, understand how it checks input, and recover the flag that is hidden in its data.

## Background (for anyone new to reversing)
A few ideas to know first:

* A **compiled binary** is a program that has already been turned from human-readable source code into machine code. You cannot just open it and read it like a `.txt` file — you need tools to translate it back into something readable.
* **Ghidra** is a free tool (made by the NSA) that does this translation. It **decompiles** machine code into C-like pseudocode you can read.
* A **PRNG** (pseudo-random number generator) is the code behind functions like `rand()`. It is *not* truly random — it produces numbers from a starting value called a **seed**. The important part: **the same seed always produces the same sequence.** If you know the seed, the "random" numbers are 100% predictable.
* `srand(x)` sets the seed to `x`. `rand()` then produces the next number in that seed's sequence.

That last point is the entire challenge.

## Analysis

### Step 1: First look (`file` and `strings`)
```bash
file casino
strings casino
```
`file` reported a **64-bit ELF, not stripped**. "Not stripped" is good news — it means function names survive, so in Ghidra we can find functions called `main`, `check`, and `banner` by name instead of guessing.

`strings` showed the game text (`WELCOME TO ROBO CASINO`, `CORRECT`, `INCORRECT`) and the imported functions: `srand`, `rand`, `scanf`, `printf`. Seeing `srand` in something called "casino" is a big hint that predictable randomness is the theme. Notably, the flag itself was **not** in the strings output — so it is not stored as plain text.

### Step 2: Read `main` in Ghidra
After importing `casino` into Ghidra and letting it auto-analyze, the decompiled `main` revealed the whole game loop (simplified):

```c
local_c = 0;                                   // round counter
while (true) {
    if (0x1c < local_c) {                      // 0x1c = 28; after 29 rounds you win
        puts("HOUSE BALANCE $0...");
        return 0;
    }
    scanf("%c", &local_d);                     // read ONE character from you
    srand((int)local_d);                       // seed the PRNG with YOUR character
    iVar1 = rand();                            // take the first rand() output
    if (iVar1 != *(int *)(check + local_c*4)) {// compare to stored value check[round]
        puts("INCORRECT");
        exit(-2);                              // one wrong answer = game over
    }
    puts("CORRECT");
    local_c = local_c + 1;
}
```

Reading this tells us everything:
* The loop runs **29 rounds** (0 through 28), so the flag is **29 characters** long.
* Each round it takes **one character**, uses that character as the **seed**, generates one `rand()` value, and compares it to a stored number `check[round]`.
* `check` is a **table of 29 numbers** stored in the program's data.

### Step 3: The key insight
The seed is *the character we type*. Because `srand(c); rand()` always gives the same result for the same character `c`, the table is really saying:

> "For each position, which character, when used as the seed, produces this stored number?"

So each of the 29 stored numbers maps back to exactly **one** character — and those characters, in order, are the flag. The flag was never encrypted; it was stored as *"the `rand()` output you get when you seed with each flag character."*

## Payload & Execution
There is no need to play the game by hand. There are only a small number of possible characters, so we can try each one offline: for every printable character, seed with it, run `rand()`, and see which stored value it matches.

### Step 1: Get the `check` table
In Ghidra's **Symbol Tree**, `check` appears under **Labels** (not Functions), because it is data, not code. Jumping to it in the Listing shows the raw bytes. The 29 values are stored as 4-byte little-endian integers (29 × 4 = 116 bytes). Using **Copy Special → Byte String** gives the raw bytes to work with.

### Step 2: Reverse the table into the flag
```python
from ctypes import CDLL
import struct

libc = CDLL("libc.so.6")   # use the SAME rand() the binary uses

# 116 bytes copied from Ghidra (the `check` table)
hexbytes = """
be 28 4b 24 05 78 f7 0a 17 fc 0d 11 a1 c3 af 07 33 c5 fe 6a a2 59 d6 4e
b0 d4 c5 33 b8 82 65 28 20 37 38 43 fc 14 5a 05 9f 5f 19 19 20 37 38 43
80 93 14 63 99 b2 5a 61 33 c5 fe 6a b8 cf 6f 6c 20 37 38 43 37 a2 3d 0f
33 c5 fe 6a 99 b2 5a 61 b8 82 65 28 fc 14 5a 05 94 49 e4 3a e9 df d7 06
a2 59 d6 4e cd 4a cd 0c 64 ed d8 57 99 b2 5a 61 2a bc e9 22
"""
raw = bytes(int(b, 16) for b in hexbytes.split())
NUM = len(raw) // 4
targets = struct.unpack("<" + "i"*NUM, raw)   # 29 little-endian ints

flag = ""
for t in targets:
    for c in range(0x20, 0x7f):    # printable ASCII only (see note below)
        libc.srand(c)
        if libc.rand() == t:
            flag += chr(c)
            break
print(flag)
```

Running it:
```bash
python3 solve.py
```

**Result:**
```
HTB{r4nd_1s_v3ry_pr3d1ct4bl3}
```

The flag literally states the lesson: `rand` is very predictable.

## Why It Works
`rand()` is deterministic: one seed always gives one output. The program used each flag character as a seed and stored the resulting `rand()` value in a table. Since that mapping goes both ways, we simply tried every printable character, found which one reproduced each stored value, and read the flag straight out.

Two details that made the solver correct:
* **Same libc.** `rand()`'s exact algorithm is not standardized — different systems can produce different sequences. Calling the real `libc.so.6` through Python's `ctypes` guarantees we use the *same* `rand()` the binary used. (Our Kali box is Debian-family, matching the Debian-built binary.)
* **Printable range only.** The code does `srand((int)local_d)` on a `char`. A `char` can be signed, so bytes above 127 would become *negative* seeds in the real program but stay positive in a naive `0–255` loop — a mismatch that could cause false matches. Flag characters are all printable (`0x20`–`0x7e`), so restricting the search to that range avoids the problem entirely.

A neat detail: repeated values in the table (e.g. `33 c5 fe 6a` appears three times) correspond to repeated characters in the flag — a visible fingerprint of the "seed = character" design.

## Real-World Meaning
The "predictable random" problem is a real and serious bug class, not just a CTF gimmick:

* **Weak randomness breaks security.** If a program uses `rand()` (or seeds a PRNG with something guessable like the current time) to generate passwords, session tokens, password-reset codes, or encryption keys, an attacker who can guess or reproduce the seed can predict all of those "secret" values.
* **`rand()` is not for security.** The C `rand()` function is fine for games or simulations, but it is not cryptographically secure. Its output is predictable and, given a few outputs, its future values can often be computed.
* **Seeds must be unpredictable.** Real systems must use a cryptographically secure random source, seeded from true system entropy, for anything security-related.

## Mitigation
For developers, the fixes are straightforward:

1. **Use a cryptographically secure RNG** for any security purpose — for example `getrandom()` / `/dev/urandom` on Linux, or a library CSPRNG — never plain `rand()`.
2. **Never seed with predictable values** like `time(NULL)`, a counter, or user input when the output must be unguessable.
3. **Do not store secrets as reversible transforms.** Here the flag was recoverable because each character mapped to a single stored value. Secrets should be protected with proper cryptography, not obscured with a predictable function.
