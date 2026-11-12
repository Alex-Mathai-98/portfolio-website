### 13. Proof generation basics

**Background:** my tutorial [Proof Solving in Rocq: From First Proofs to Advanced Tactics]({% post_url 2026-11-05-proof-solving-in-rocq %}) covers everything below in depth.

> Rocq is a proof checker: you state a claim, give it step-by-step instructions (tactics), and it accepts the proof only if every step is valid.

#### A proof, end to end

```coq
Theorem plus_O_n : forall n : nat, 0 + n = n.   (* a name and a claim        *)
Proof.                                         (* start the proof           *)
  intros n.                                    (* tactics, one by one       *)
  simpl.
  reflexivity.
Qed.                                           (* Rocq rechecks and saves it *)
```

While inside a proof, Rocq shows the **proof state**: the **context** (what you may assume) above a line, and the **goal** (what is left to show) below it. Each tactic changes the proof state:

| After | Proof state | What the tactic did |
| --- | --- | --- |
| `Proof.` | `⊢ forall n : nat, 0 + n = n` | Nothing assumed yet |
| `intros n` | `n : nat ⊢ 0 + n = n` | "Let `n` be any number": moves `n` into the context |
| `simpl` | `n : nat ⊢ n = n` | Runs the definition of `+`, which returns `n` when the first argument is `0` |
| `reflexivity` | no goals left | Both sides are the same |
{: .wrap}

***Table 1:*** *The proof state after each tactic of `plus_O_n`. `⊢` separates the context from the goal.*

#### What makes it a good target for LLMs

- **Every step is checked.** `Qed` makes Rocq's kernel recheck the whole proof, so a wrong LLM guess can only fail. It can never produce a false "proved".
- **Errors are local.** Rocq reports which tactic failed and on which goal, so an agent can retry one step instead of the whole proof.
- **Tactics can split a goal.** Tactics such as `destruct` (case split) or `replace` (with a side goal) turn one goal into several, so a proof is really a **tree** of goals.

#### The tactics used in the next two papers

| Tactic | In plain words |
| --- | --- |
| `assert (H : P) by t` | Prove the fact `P` with tactic `t`, then add it to the context as `H` |
| `apply L` | Use lemma `L` backwards: to prove its conclusion, it is enough to prove its premises |
| `destruct H` | Split an "or" hypothesis into one goal per case |
| `replace a with b in H` | Rewrite `a` to `b` inside `H`, leaving `b = a` as a side goal |
| `subst` | Replace a variable by what it is equal to |
| `exfalso` | Turn the goal into `False`: use when the hypotheses contradict each other |
| `lia` | Decides linear integer arithmetic (it cannot multiply two unknowns) |
| `t1; t2` | Run `t1`, then `t2` on every goal `t1` leaves |
| `Search pattern` | Not a tactic: asks Rocq for library lemmas whose statement matches `pattern` |
{: .wrap}

***Table 2:*** *Tactics used in the AutoRocq and LemmaNet proofs below.*

#### Where the goals come from

Both papers prove goals extracted from C programs. **Frama-C**'s WP plugin reads C code annotated with ACSL specifications (pre/postconditions and loop invariants) and turns each check into a Rocq theorem, a **verification condition (VC)**. Proving every VC proves the C code meets its specification.
