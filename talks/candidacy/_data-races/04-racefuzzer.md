### 4. RaceFuzzer: race-directed random testing

**Paper:** Koushik Sen, *Race Directed Random Testing of Concurrent Programs*, PLDI 2008. [Read the full post]({% post_url 2026-11-01-racefuzzer %}).

> Take each race a predictive detector suspects, and steer the thread scheduler until the two racing accesses are **one step apart in two different threads**. If that moment happens, the race is real.

#### A tiny program with four suspects

`LogService` has two threads. **M** sets up a config, signals readiness under a lock `L`, serves requests for a while, then shuts the logger down. **W** is a worker: it reads the config if the service is ready, then writes to the logger if the logger still exists. A launcher starts both threads together, so nothing like `start` or `join` orders them.

![LogService: the running example](/assets/img/posts/racefuzzer/01-logservice-code.png)

The paper's model assumes each statement touches at most one shared variable. That's why the null check (`w7`) and the call (`w8`) are separate statements. In practice this holds automatically, because the tool works on Java bytecode, where every field access is its own instruction.

Only `ready` is accessed under `L`. The other three shared variables are touched with no lock held:

![Every access to shared state, by thread](/assets/img/posts/racefuzzer/02-shared-accesses.png)

So any lock-based detector will suspect four pairs of statements. Here is what is actually true of each:

| Pair | Variable | Truth |
| --- | --- | --- |
| **P1** (m1, w6) | `cfg` | **False alarm.** W reads `cfg` only after seeing `ready == true`, which M sets after `m1`. The flag, guarded by `L`, orders them. |
| **P2** (m5, w1) | `lastActive` | **Real, benign.** A debugging field nobody reads. |
| **P3** (m6, w7) | `log` | **Real, harmless.** If the shutdown wins, the null check sees `null` and W skips the append. |
| **P4** (m6, w8) | `log` | **Real, harmful.** If the shutdown lands between the check and the use, W throws a `NullPointerException`. |

A good tool should stay quiet on P1, confirm P2 through P4, and show that P4 can crash.

#### Phase 1: where the suspects come from

RaceFuzzer doesn't invent its suspects. It takes them from an existing predictive detector. The paper uses the **hybrid** detector of O'Callahan and Choi (PPoPP 2003), but says any static or dynamic detector would do. This phase is not the paper's contribution.

The hybrid detector watches one run and flags two memory accesses when all of these hold:

- different threads, same memory location, at least one write;
- **no common lock** held (the lockset part, from Eraser);
- **not ordered** by `start`, `join` or `wait/notify` (a happens-before check that deliberately ignores lock release → acquire edges).

Formally, a memory event is `MEM(σ, m, a, t, L)`: thread `t`, executing statement `σ`, performs access `a` (READ or WRITE) on location `m` while holding lock set `L`. Two events are reported when:

$$
\begin{aligned}
& e_i = \mathrm{MEM}(\sigma_i, m_i, a_i, t_i, L_i) \;\wedge\; e_j = \mathrm{MEM}(\sigma_j, m_j, a_j, t_j, L_j) \\
& \wedge\; t_i \neq t_j \;\wedge\; m_i = m_j \;\wedge\; (a_i = \mathrm{WRITE} \vee a_j = \mathrm{WRITE}) \\
& \wedge\; L_i \cap L_j = \emptyset \;\wedge\; \neg(e_i \prec e_j) \;\wedge\; \neg(e_j \prec e_i)
\end{aligned}
$$

Here `≺` is happens-before built only from `start`, `join` and `wait/notify`. When the condition holds, the statements (σi, σj) become a racing pair.

Here is one run of LogService in which M runs up to `m5`, W runs to the end, and then M shuts down:

![One Phase 1 run, event by event](/assets/img/posts/racefuzzer/03-phase1-trace.png)

All four suspects are flagged, including the false alarm P1: the real ordering goes through `ready` and `L`, and the hybrid test ignores lock edges. Notice also that nothing crashed in this run. Phase 1 predicts P4 from a run where it did not crash. That's the payoff of lockset-style reasoning.

Two terms are worth keeping apart here:

- An **event race** is two specific executions in one run, such as events #8 and #9 above.
- A **racing pair** is the two *statements* behind them, here (m6, w8). Many event races, from many schedules, converge on the same pair, and the pair is what Phase 1 hands to Phase 2.

Phase 1 is not schedule-free either. It can only flag statements that actually executed: if W happens to read `ready == false`, it never runs `w6`, and P1 is never flagged. But that's only a coverage requirement. A happens-before detector needs coverage **and** a lucky ordering: the two accesses must also be unordered in the interleaving it saw.

#### Phase 2: postpone, then flip a coin

Phase 2 is the contribution. It takes **one** racing pair at a time, `RaceSet = {σ1, σ2}`, and runs the program under its own scheduler. Only one thread runs at a time, and every choice comes from a seeded random number generator:

![Algorithms 1 and 2, with the paper's line numbers](/assets/img/posts/racefuzzer/04-pseudocode.png)

The idea, step by step:

1. **Pick a random thread** that is enabled and not postponed.
2. **If its next statement isn't σ1 or σ2, just run it.** Everything outside the pair is uninteresting.
3. **If it is, look for a partner.** `Racing` returns `R`: the postponed threads that all wait at the same statement (σ1 or σ2) and are all about to access the same address as this thread, with at least one write.
4. **No partner yet: postpone the thread.** It waits, frozen one step before its racing access.
5. **Partner found: that's a real race.** Two threads are each one step from conflicting accesses to the same address, with nothing in between. RaceFuzzer reports it, then flips a coin. Heads: run this thread first. Tails: postpone it, and run every thread in `R` first.
6. **If every enabled thread is postponed, release one at random.** That thread runs its next statement unconditionally, which avoids an infinite loop. This is how a false pair ends: the partner never shows up, so the waiting thread is eventually let go.

Three details make this work.

- **The invariant.** At line 5, no two postponed threads ever race with each other. Every race is detected the moment it becomes complete, when a thread that isn't postponed arrives to finish the pair, and it's settled right away. That's why tails runs *all* of `R`, not just one thread: leaving one behind would put two racing threads in `postponed`, where nothing ever compares them.
- **The coin flip doesn't fix anything.** It only decides which of the two conflicting accesses goes first. Both orders are possible in real executions, and usually only one is harmful, so trying both at random is how RaceFuzzer finds out whether the race can crash the program.
- **Replay is free.** Because the seed fixes every choice, rerunning with the same seed reproduces the crash exactly. Nothing needs to be recorded.

#### Watching it work on LogService

To check the traces and numbers below, I wrote a small simulator of the algorithm and ran it on LogService.

##### P4: the crash

With `RaceSet = {m6, w8}`, one run goes like this:

1. W runs `w1`–`w7`. It sees a live logger at `w7`, so it heads for `w8`.
2. W arrives at `w8`. No partner is postponed yet, so **W is postponed**.
3. M runs through its setup and `serve()`. W just waits, however long that takes.
4. M arrives at `m6`. `Racing` finds W, so a **real race is reported**.
5. The coin says "run M first": `log = null`. M finishes.
6. Now everyone enabled is postponed, so W is released and runs `log.append(...)`: **NullPointerException**.

If the coin had gone the other way, W would have appended first and nothing would break. If M had reached `m6` first, M would have waited for W instead, and the race would be found from W's side. Over 2,000 seeds, the race was created in **100%** of runs and the crash happened in about **50%**.

##### P1: the partner never comes

With `RaceSet = {m1, w6}`, M gets postponed at `m1` before even setting `cfg`. W then reads `ready == false`, skips `w6`, and finishes. Everyone enabled is now postponed, so M is released and runs to the end. W can only reach `w6` after `m3`, which comes after `m1`, so the two threads can never meet. **0 of 2,000** runs reported a race. RaceFuzzer stays silent without anyone reading the code.

##### The scorecard

| Pair | Phase 1 | Race created | Crash | RaceFuzzer's verdict |
| --- | --- | --- | --- | --- |
| P1 (m1, w6) | suspected | 0% | — | not confirmed |
| P2 (m5, w1) | suspected | 100% | 0% | real |
| P3 (m6, w7) | suspected | 100% | 0% | real |
| P4 (m6, w8) | suspected | 100% | ≈ 50% | real **and harmful** |

Phase 1 gave four identical-looking warnings. Phase 2 sorted them into three tiers.
