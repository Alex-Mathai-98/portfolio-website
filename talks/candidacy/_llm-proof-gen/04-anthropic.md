### 16. Anthropic: library-level verification of memory safety

**Source:** Anthropic's research post on [formal verification](https://www.anthropic.com/research/formal-verification), with the public proofs at [theorem-labs/oss-verify-proofs](https://github.com/theorem-labs/oss-verify-proofs). The proofs are stated in LPS, Theorem's Rocq model of a Linux process running a RISC-V binary ([theorem-labs/rocq-linux-process-semantics](https://github.com/theorem-labs/rocq-linux-process-semantics)). [Read the full post]({% post_url 2026-11-06-frama-c-vs-anthropic-verification %}).

> Instead of writing a spec in C terms and letting Frama-C generate goals, compile the library's fuzz tests to RISC-V machine code and state one generic theorem: "this fuzz test never crashes, for any input under 64 MB". Claude then has to prove that theorem in Rocq.

#### The idea

AutoRocq and LemmaNet prove goals that Frama-C generates from a human-written spec. Here there is no per-program spec at all:

- **One statement for every program.** Each fuzz target is built twice, plain and with AddressSanitizer (ASan). The theorem, called `agrees`, is the same for every library: on every input within the limits, both builds are error-free and behave the same. ASan aborts on any bad memory access, so "the ASan build is error-free" means no memory errors.
- **The binary, not the source.** The proof is about the machine code that actually runs, so a compiler bug cannot hide a crash.
- **"Memory error" is defined by the machine model.** No one writes `\valid_read` checks; a load from a byte that no readable segment holds is simply a fault in the model.

***Table 1:*** *Two ways to say "no memory errors"*

|  | Frama-C (AutoRocq, LemmaNet) | Anthropic / LPS |
| --- | --- | --- |
| What is analysed | The C source | The compiled RISC-V binary |
| What "memory error" means | ACSL checks such as `\valid_read(p)`, added by a human or by Frama-C's RTE plugin | A `MachineFault` in the machine model, or an `abort()` from ASan |
| Who writes the spec | Humans, or the RTE plugin | Nobody: every program gets the same statement |
| What gets proved | Many small goals, one per check | One theorem about every execution of the whole binary |
| Loop invariants | Written in the annotations, before goals exist | Stated and proved inside the Rocq proof |
{: .wrap}

#### The machine model

For each instruction, LPS fetches it and passes it through two layers:

![Figure 1: One machine step in LPS](/assets/img/posts/frama-c-vs-anthropic-verification/machine-step-layers.png)

***Figure 1:*** *One machine step in LPS. Layer 1 (`exec`) is the RISC-V chip; Layer 2 (`interp_eff_accesses`) is the Linux process around it. LPS models only Linux.*

- **Layer 1 (`exec`)** turns the instruction into requests, such as `ELoad` ("read n bits at address a") or `EStore`. It never touches memory.
- **Layer 2 (`interp_eff_accesses`)** answers each request against memory. A readable load continues; an unreadable one returns `MachineFault`.
- **Fail-closed:** a step the model cannot interpret returns `MachineUnmodeled`, which also counts as an error, so it can never pass as safe.

#### One example, both ways

The fuzz test reads the first byte of its input, guarded so it never reads an empty buffer:

```c
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
  if (size > 0) {
    uint8_t x = data[0];   // the only memory access
  }
  return 0;
}
```

![Figure 2: The whole proof, both ways: Frama-C next to LPS](/assets/img/posts/frama-c-vs-anthropic-verification/frama-c-vs-lps-side-by-side.png)

***Figure 2:*** *The same program proved both ways. Left: Frama-C on the C source. Right: LPS on the RISC-V binary. Highlighted: the memory check, the goal it becomes, and the step that proves it. The LPS proof is pseudocode.*

Reading Figure 2 row by row:

1. **The program.** Frama-C's RTE plugin inserts `assert \valid_read(data + 0)` before the read. The binary has no assertion, just a `beqz` (skip if `size == 0`) and an `lbu` (load one byte).
2. **The goal.**
   - *Frama-C:* the assert becomes `wp_goal`: from "all `size` bytes are readable" and `0 < size`, prove "1 byte is readable".
   - *LPS:* a generic theorem: for any start state whose input bytes are readable, the harness never reaches a crash.
3. **The proof.**
   - *Frama-C:* unfold `valid_rd`, then `lia`: the whole buffer fits in its block and the buffer is not empty, so its first byte fits too.
   - *LPS:* step through the instructions. `beqz` touches no memory, so split on `size = 0`. In the `size > 0` case, Layer 1 turns `lbu` into an `ELoad`, and Layer 2 faults unless the byte is readable. So the proof **must** prove `readable s0.(mem) s0.(a0)`, from the input hypothesis with i = 0.

**The key point:** both proofs have the same core, "one byte fits in a non-empty buffer". Frama-C writes the obligation down in advance; in LPS it appears by itself at the `lbu`, as a case of Layer 2 that the proof has to rule out.

#### What it buys, and what it costs

- **No spec writing:** "never crashes" needs no human to describe what the program should do.
- **The compiler is not trusted**, since the proof is about the binary.
- **The guarantee is narrower:** no crash for inputs under 64 MB, on static, single-threaded RISC-V Linux, using ASan's notion of a crash. It says nothing about computing the right answer.
- **The proofs are huge:** the reasoning a Frama-C user puts into annotations moves into the proof. The post reports about 600,000 lines of Rocq for a typical library.
