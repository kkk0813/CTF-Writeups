# It's Oops PM

**Platform:** Hack The Box (CTF Try Out)  
**Category:** Hardware  
**Difficulty:** Very Easy  
**Concepts:** Hardware Trojan, Backdoor Logic, VHDL, Multiplexer (MUX), Crypto-Processor / TPM

## Objective
A remote "satellite" service asks for a **16-bit binary input**. We are given the design files for the chip that runs behind that service. The goal is to read the chip's logic, find the hidden trigger built into it, and send that trigger to the service to unlock the flag.

## Background (for anyone new to hardware)
Before the walkthrough, three words that show up a lot:

* **VHDL** is a language used to *describe* what a chip does. It is plain text, like code, so we can just read it. It is not a compiled program — it is more like a blueprint for wiring.
* A **crypto-processor / TPM** is a small chip whose job is to keep a secret **key** locked inside. The outside world can send it data to be encrypted, but it should **never** be able to see the key itself. That is the chip's one big promise.
* A **multiplexer (MUX)** is just a switch. It looks at a "select" wire and, based on that wire, decides which of two inputs to pass through to the output. Think of a train track switch: one lever decides which of two tracks the train takes.

Keep the TPM's promise in mind — *the key must never come out* — because this challenge is all about a chip that breaks it.

## Analysis
Unzipping the challenge file gave five files:

```
backdoor.vhdl
encryption.vhdl
key.vhdl
tpm.vhdl
schematic.png
```

`schematic.png` is a hand-drawn diagram showing the blocks: **INPUT → CRYPTO → MUX → output**, with a **KEY** block feeding in and a **LOGIC** block on a side path. That side path is the clue.

Reading the files one at a time:

**`key.vhdl`** — hands out a fixed secret key and nothing else:
```vhdl
constant key : STD_LOGIC_VECTOR(15 downto 0) := "0110001111100001";
```

**`encryption.vhdl`** — takes our input and the key, mixes them together with XOR, and produces an encrypted output. This is the normal, safe path.

**`backdoor.vhdl`** — this is the important one. It watches our input and turns on a single flag wire `B` **only if** our input exactly matches one hidden value:
```vhdl
constant pattern : STD_LOGIC_VECTOR(15 downto 0) := "1111111111101001";
...
if D = pattern then
    B <= '1';   -- input matched the secret! raise the flag
else
    B <= '0';   -- normal input, do nothing
end if;
```

**`tpm.vhdl`** — the top-level file that wires everything together. It connects our input to all three blocks above, then uses this switch at the end:
```vhdl
case B is
    when '1'   =>  Output <= Key         -- backdoor ON  -> leak the KEY
    when others =>  Output <= Encrypted  -- backdoor OFF -> normal encryption
```

That `case` statement **is** the MUX from the schematic, and the backdoor's flag `B` is the select wire.

So the whole chip works like this:

* **Any normal input** → `B = 0` → the MUX passes the *encrypted* output. Safe. The key stays hidden.
* **The one magic input** → `B = 1` → the MUX flips and passes the *raw key* straight to the output.

In plain words: this chip has a secret knock. Type the right 16 bits and it stops doing its job and hands you the secret it was supposed to protect.

## Payload & Execution
We do **not** need to do any encryption math. The answer is just the backdoor's hidden pattern:

```
1111111111101001
```

Connect to the service and send it:

```bash
echo "1111111111101001" | nc 154.57.164.69 30530
```

* `nc` opens a network connection to the satellite service at that IP and port.
* `echo "..." |` types the 16 bits into it automatically, so we do not have to type at the prompt by hand.

When the service receives the magic pattern, the backdoor flag goes high, the MUX flips, and the service returns the key/flag.

> **Bit-order tip:** the value is declared as `(15 downto 0)`, which means the **leftmost** character is the highest bit (bit 15). We send it left-to-right exactly as written. If the service ever rejected it, the next thing to try would be the reversed string `1001011111111111`, just to rule out the service reading the bits in the opposite order.

## Why It Works
The chip looks completely normal almost all the time. There are 65,536 possible 16-bit inputs, and 65,535 of them behave perfectly — input goes in, encrypted output comes out, key stays secret. Only **one** input out of all of those flips the switch.

That is exactly what makes this a **hardware trojan**: a piece of logic hidden inside a chip that behaves normally until it sees a secret trigger, then does something bad. Because it almost never activates, normal testing would never catch it — the testers would have to guess the exact magic value, which is basically impossible by luck.

The trojan is built from just two small parts working together:
1. A **comparator** (`backdoor.vhdl`) that watches for one exact value.
2. A **MUX** (in `tpm.vhdl`) whose select wire is controlled by that comparator, letting it swap the safe output for the secret one.

Once you can spot the pattern "*input feeds a comparator, and that comparator picks which thing the MUX outputs*," you can recognize this kind of backdoor anywhere.

## Real-World Meaning
In real life, a backdoor like this is a serious problem:

* **It silently leaks secrets.** A crypto chip is trusted to never reveal its keys. A hidden trigger lets anyone who knows the secret input pull the keys out, with nothing suspicious showing up in software logs.
* **It can be slipped in through the supply chain.** Chips pass through many companies — designers, factories, packagers. A bad actor at any step could insert a trojan, and the customer who buys the chip usually cannot see inside the real silicon the way we just read these text files.
* **It hides from testing.** Because it only wakes up on a rare trigger, ordinary quality tests pass it as "working fine."

This is a real research area (look up "hardware trojans" and "supply-chain security"). Note that this challenge *calls* the block a TPM for flavor, but it is really a simple crypto-processor, not a real spec-compliant Trusted Platform Module.

## Mitigation
How defenders try to stop trojans like this in the real world:

1. **Golden-reference comparison** — compare a suspect chip against a known-good "golden" chip to spot logic that should not be there.
2. **Side-channel / power analysis** — measure things like power use or timing; hidden logic can leave tiny physical fingerprints.
3. **Formal verification of the design** — mathematically prove the chip only does what the spec allows, with no extra hidden paths.
4. **Trusted design and fabrication** — control who can touch the design and where it is manufactured, so no one can quietly add a "secret knock."

The core lesson: a secure chip should have **no hidden input** that changes its behavior. If one exists, the chip's security promise is already broken.
