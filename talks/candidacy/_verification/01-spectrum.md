### 8. The range of verification techniques: model checking to interactive proof

**Post:** [Read the full post]({% post_url 2026-11-09-verification-techniques %}).

The three approaches differ in who writes the proof and how much the result covers:

- Bounded model checking needs no proof, but covers only runs up to a bound.
- Rocq covers everything, but you write every step.
- Verus sits in between: you write the key facts (invariants), and Z3 works out the rest.

<!-- AIDEV-NOTE: proof-effort/guarantee scale; reuses the .tradeoff-scale styles in candidacy.md (.ts-pm = centred mark). -->
<div class="tradeoff-scale" role="figure" aria-label="Verification techniques from least proof effort to strongest guarantee">
  <div class="ts-ends">
    <span><strong>&larr; Less proof to write</strong><small>result holds only up to a bound</small></span>
    <span><strong>Stronger guarantee &rarr;</strong><small>holds for every input, more proof to write</small></span>
  </div>
  <div class="ts-track">
    <div class="ts-line" aria-hidden="true"></div>
    <div class="ts-mark ts-p0">
      <span class="ts-dot" aria-hidden="true"></span>
      <span class="ts-name">Bounded model checking</span>
      <span class="ts-cite">e.g. CBMC · LLM agent: <a href="#9-bmc-agent-agentic-model-checking">BMC-Agent</a></span>
      <span class="ts-note">No proof to write: the tool unrolls loops up to a bound and checks every run within it.</span>
    </div>
    <div class="ts-mark ts-pm">
      <span class="ts-dot" aria-hidden="true"></span>
      <span class="ts-name">SMT-based deductive verification</span>
      <span class="ts-cite">e.g. Verus · LLM agent: <a href="#12-exverus-verus-proof-repair-via-counterexample-reasoning">ExVerus</a></span>
      <span class="ts-note">You write the key facts (invariants); Z3 proves the rest, for every input.</span>
    </div>
    <div class="ts-mark ts-p3">
      <span class="ts-dot" aria-hidden="true"></span>
      <span class="ts-name">Interactive proof</span>
      <span class="ts-cite">e.g. Rocq · LLM agent: <a href="#14-autorocq-agentic-verification-of-software-systems">AutoRocq</a></span>
      <span class="ts-note">You write every proof step; the checker verifies each one.</span>
    </div>
  </div>
</div>

#### The spectrum

![Figure 1: Where SMT-based deductive verification sits between bounded model checking and interactive proving](/assets/img/posts/verification-techniques/vt-01-verification-spectrum.png)

***Figure 1:*** *Where SMT-based deductive verification sits between bounded model checking and interactive proving. Examples of agentic papers at each point: BMC-Agent and Avocado (CBMC), ExVerus (Verus), AutoRocq (Rocq).*

#### Hello world: one claim, three tools

The program starts at x = 0 and adds 2, n times. The claim: the result is 2n. All three versions below were run and pass (CBMC 5.95, Verus 0.2025.07.12, Rocq/Coq 8.18).

![Figure 2: The same claim proved with CBMC, Verus and Rocq](/assets/img/posts/verification-techniques/vt-02-hello-world.png)

***Figure 2:*** *The same claim proved with CBMC, Verus and Rocq. Green: the claim being proved. Blue: what you must add so the tool can prove it.*

***Table 1:*** *What each version needs and what it proves*

| Tool | What you add beyond the code | What gets proved | How the tool gets there |
| --- | --- | --- | --- |
| CBMC | A bound on n and an unrolling limit (`--unwind 101`) | The claim for every n ≤ 100, and nothing beyond | Unrolls the loop 100 times and asks a SAT solver for a failing run |
| Verus | One loop invariant, `x == 2 * i`, plus `decreases` | The claim for every u32 value of n, with no bound: x is a u64, so 2n always fits | Generates three checks for the loop; Z3 proves each one.<br>(1) Entry: at i = 0, x = 0, the invariant holds (0 == 2 · 0).<br>(2) One iteration: if x == 2 · i and i < n, then after the body x + 2 == 2 · (i + 1).<br>(3) Exit: when the loop stops, i = n, so x == 2 · n. |
| Rocq | A proof script: induction on n, then `lia` for the arithmetic | The claim for every natural number n | Checks each step you write; nothing is searched for you |
{: .wrap}

Three things to notice:

- **The bound plays a different role.** In CBMC the bound is essential: the tool can only unroll a fixed number of iterations. Verus needs no bound: the invariant works for any n, and making `x` a `u64` means 2n always fits (`2^64` at max). Rocq is valid for infinitely large values.
- **The invariant is the whole proof in Verus.** `x == 2 * i` is exactly the fact the Rocq proof establishes by induction. Verus asks you only for that fact and lets Z3 do the induction step for you.
- **Rocq usually proves a functional version.** The loop is written as a recursive function, which is standard in Rocq.
