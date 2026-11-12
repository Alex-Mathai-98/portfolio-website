### 15. LemmaNet: automated lemma discovery in agentic program verification

**Paper:** Zhao, Tu, Liu, Rinard and Roychoudhury, *Automated Lemma Discovery in Agentic Program Verification*, ASE 2026, [doi:10.1145/3832783.3834356](https://doi.org/10.1145/3832783.3834356). [Read the full post]({% post_url 2026-11-06-lemmanet-automated-lemma-discovery %}).

> Before proving a low-level verification condition, have the LLM restate it in source-code terms and prove that simple version. Then have it write helper lemmas that carry the same argument into Frama-C's memory model.

#### The gap: retrieving lemmas vs. inventing them

- **AutoRocq retrieves.** In `wp_goal`, both helper lemmas (`Z.abs_le`, `Z.square_le_mono_nonneg`) already existed in Rocq's standard library, and `Search` found them.
- **Why invent lemmas?** A Frama-C VC hides a simple idea under low-level detail. It is easier for the LLM to first solve the problem informally, at the level of the source code, and then state each step of that argument as a lemma written with Frama-C's own definitions. With those lemmas, the formal proof is short.
- **LemmaNet invents them.** It is built on AutoRocq, which it uses as its proof agent, and adds a lemma synthesizer and a lemma adapter around it.

#### The example: `hex2bin` from the Linux kernel

`hex2bin` turns a hex string into bytes. The loop invariant to check says `src` never drops below its starting value, kept in the ghost variable `osrc`:

```c
//@ ghost char *osrc = src;          // ghost variable: remembers the start
/*@ loop invariant osrc <= src; ... */
while (count--) {
  int hi = hex_to_bin(*src++);
  int lo = hex_to_bin(*src++);
  ...
}
```

`src` moves forward by 2 every iteration, so the invariant is almost trivially preserved: if `osrc <= src` before, then `osrc <= src + 2` after.

##### Step 1: the goal Frama-C produces

Frama-C turns that one-line preservation check into a Rocq goal with about 35 hypotheses. Here `a1` is `osrc` and `a2` is `src`; most hypotheses are elided:

```coq
Theorem wp_goal :
  forall (t: Z -> Z) (t1: addr -> Z) (t2: Z -> Z)
         (a: addr) (i: Z) (i1: Z) (i2: Z) (a1: addr) (a2: addr),
  (* ... counters, offsets, bit arithmetic ... *)
  (* ... memory model and type facts ... *)
  addr_le a1 a2 ->                    (* osrc <= src: the invariant before the iteration *)
  (* ... what hex_to_bin returned, the other loop invariant ... *)
  addr_le a1 (shift a2 2).            (* osrc <= src + 2 *)
```

- **Only one hypothesis matters:** `addr_le a1 a2`. The rest describes memory, integer types and bit operations.
- **Even so, `lia` cannot finish.** `addr_le` and `shift` are Frama-C memory-model definitions, not arithmetic. Getting from `addr_le a1 a2` to `addr_le a1 (shift a2 2)` needs facts that connect them, and no library has those.
- **AutoRocq alone fails on this goal.**

##### Step 2: a short, source-level version

The key idea: first reason at the level of the source code. An LLM reads the C code and specs and writes what the invariant really says, using Frama-C's pointer representation (a block `base` plus a position `offset`; `shift p k` moves the offset by `k`):

```coq
Definition addr_le_model (p q : addr) : Prop :=
  base p = base q /\ offset p <= offset q.

Lemma goal_osrc_le_src_preserved :
  forall osrc src : addr,
  addr_le_model osrc src -> addr_le_model osrc (shift src 2).
Proof.
  intros osrc src [Hb Hoff].
  split.
  (* base osrc = base (shift src 2) *)
  - unfold shift, base in *; simpl in *; exact Hb.
  (* offset osrc <= offset (shift src 2) *)
  - unfold shift, offset in *; simpl in *; lia.
Qed.
```

- `intros osrc src [Hb Hoff]` splits the hypothesis into `Hb : base osrc = base src` and `Hoff : offset osrc <= offset src`.
- `split` makes two goals, one per half of the "and".
- **Same block:** unfolding `shift` turns `base (shift src 2)` into `base src`, so `Hb` closes it.
- **Offset only grows:** unfolding turns `offset (shift src 2)` into `offset src + 2`, so `lia` closes it from `Hoff`.

**The catch:** `addr_le_model` is the LLM's own definition. Nothing guarantees it matches Frama-C's `addr_le`, so this proof is only a guide. The target is still Frama-C's goal.

##### Step 3: helper lemmas that bridge the two levels

From the short proof, the LLM knows *why* the property holds: same block, and the offset only grows. It now states each step in Frama-C's own terms (`addr_le`, `shift`), as helper lemmas with their own proofs:

| Step of the short proof | Helper lemma that carries it into Frama-C |
| --- | --- |
| From `osrc <= src`, both pointers are in the same block (`Hb`) | `HL2_addr_le_same_base`: from `addr_le p q`, get `base p = base q` |
| Moving `src` forward keeps `osrc` at or before it (`Hoff`, then `lia`) | `HL1_addr_le_shift_same_base`: same base, `addr_le p q` and `0 <= k` give `addr_le p (shift q k)` |
{: .wrap}

***Table 1:*** *How each step of the short proof becomes a helper lemma.*

```coq
Lemma HL1_addr_le_shift_same_base :
  forall p q k, base p = base q -> addr_le p q ->
  (0 <= k)%Z -> addr_le p (shift q k).

Lemma HL2_addr_le_same_base :
  forall p q, addr_le p q -> base p = base q.
```

Rocq compiles each lemma with its proof. A lemma whose proof fails is discarded, along with any lemma that depends on it.

##### Step 4: the real proof

With HL1 and HL2 proved above it, Frama-C's goal takes three steps. Here it is with one bullet per goal:

```coq
Proof.
  intros ... ; cbn.
  (* Goal: addr_le a1 (shift a2 2) *)
  apply (HL1_addr_le_shift_same_base a1 a2 2).
  (* 3 goals are now open: A, B, C *)
  - (* Goal A: base a1 = base a2 *)
    apply (HL2_addr_le_same_base _ _ Hle_a1a2).
  - (* Goal B: addr_le a1 a2 *)
    exact Hle_a1a2.
  - (* Goal C: 0 <= 2 *)
    lia.
Qed.
```

- `apply HL1` works backwards: to prove `addr_le a1 (shift a2 2)`, it is enough to prove HL1's three premises.
- HL2 proves Goal A from the hypothesis `Hle_a1a2 : addr_le a1 a2`, which also closes Goal B directly, and `lia` closes Goal C.
- The paper writes the last two bullets as one line, `all: try (exact Hle_a1a2); try lia.`: try both tactics on every open goal and keep whatever works.

#### How LemmaNet works

- **Offline lemma synthesizer (before proving):**
  - an LLM reads the sliced C code and specs, then writes and proves the short version (Step 2), with up to five tries to make it compile;
  - it compares that with Frama-C's goal and proposes helper lemmas with proofs, plus a plan for where to use each (Step 3).
- **Online lemma adapter (during proving):** an offline lemma may not fit exactly, for example a type mismatch (`Z` vs. `addr`) or a clashing definition. Using Rocq's error message, the agent fixes or strengthens the lemma, and adds it as an `assert` with its own proof.
- **Still trustworthy:** the final proof is always of Frama-C's original goal, unproved statements never enter the context, and `Qed` rechecks everything.

#### Results

| Tool | VCs proved (of 941) |
| --- | --- |
| LemmaNet | **364** |
| AutoRocq | 287 |
| Copra | 240 |
| CoqHammer | 123 |

***Table 2:*** *VCs proved out of 941: 641 from SV-COMP plus 300 from NTP4VC, which includes the Linux kernel and Contiki OS. Every tool uses GPT-5.2.*

- **Harder NTP4VC set:** 66 vs. 40 for AutoRocq (+60%).
- **Ablation:** without reading the program source, LemmaNet drops from 364 to 301. Proving VCs benefits from understanding the program, not just the maths.
- **What the lemmas do:** most are about memory and addresses (28%). In 88% of the uniquely proved VCs that use helper lemmas, a lemma bridges the program-level idea and Frama-C's encoding, just like HL1 and HL2.
