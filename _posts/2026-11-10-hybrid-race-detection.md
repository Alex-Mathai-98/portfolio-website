---
title: "Hybrid Race Detection: Lockset, Checked Against Just Enough Happens-Before"
categories: [Technical]
math: true
mermaid: false
---

Dynamic data race detectors come in two families, and each fails in its own direction. **Lockset** detectors such as Eraser are cheap and barely care about the schedule, but they cry wolf whenever correct code synchronizes without locks. **Happens-before** detectors never cry wolf, but they are expensive, and an unrelated lock hand-off can hide a real race from them.

In this post I walk through O'Callahan and Choi's *Hybrid Dynamic Data Race Detection* (PPoPP 2003), which combines the two. Its idea fits in one sentence:

> Report a race only when **no common lock** was held **and** a deliberately **limited** happens-before relation cannot order the two accesses.

I assume you know locksets and Lamport's happened-before relation. My earlier posts on [Eraser]({% post_url 2026-10-30-eraser-lockset %}) and [Lamport clocks]({% post_url 2026-10-17-lamport-clocks %}) cover both, and the Lamport post includes vector clocks.

## 1. A small program with five fields

I'll use one 18-line Java program, **JobServer**, for the whole post. A main thread **M** starts one worker **W**, and the two share five fields. **Two fields hold real races, two are safe but fool lockset, and one is correctly locked.** Each field copies a case from the paper's own figures.

![Figure 1: The JobServer program](/assets/img/posts/hybrid-race-detection/01-jobserver-code.png)

***Figure 1: The JobServer program, stitched together from the paper's Figures 1, 2 and 4.***

- `cfg` is the paper's `globalFlag`: written once before W starts, then only read.
- `w` is the paper's `childThread`: W clears it, while M later reads it and calls `interrupt()`.
- `hits` is the paper's `globalInt`, and `clock` is the counter next to it, guarded by `clockLock`.
- `Buf.data` is the paper's recycled `BigObject`, handed between threads through `BufPool`. The pool's `put` and `take` are `synchronized`, with no `wait`/`notify`.

A detector sees one run. Here is the one we'll replay, in the order of the circled numbers:

![Figure 2: The observed run of JobServer](/assets/img/posts/hybrid-race-detection/02-observed-run.png)

***Figure 2: The observed run of JobServer, step by step, with the events each step emits. MEM = memory access, ACQ = lock acquire, REL = lock release, SND = message send, RCV = message receive (g<sub>s</sub> is the start message).***

The run does **not** crash. M11 reads `null` and skips the call. A good detector must still warn about it, because the truth is this:

| Field | What protects it | Verdict | Why |
| --- | --- | --- | --- |
| `cfg` | `start()` | Safe | Written once before W exists, only read afterwards |
| `w` | Nothing on W's side | **Real bug** | If W7 lands between M11's null check and `interrupt()`, M throws a NullPointerException |
| `hits` | Nothing | **Real race** | In another schedule W3 runs before M4 and W reads 0 |
| `clock` | `clockLock` | Safe | Every access holds `clockLock` |
| `Buf.data` | Ownership handed over through the pool | Safe | Only the thread that took the buffer touches it |

***Table 1: What a perfect detector would say about each JobServer field.***

A perfect detector reports `w` and `hits`, and nothing else. Let's see who manages it.

## 2. Lockset: cheap, but it cries wolf

The paper treats the program as a black box that emits events: memory accesses `MEM(m, a, t)`, lock acquires and releases `ACQ(l, t)` / `REL(l, t)`, and thread messages `SND(g, t)` / `RCV(g, t)`. Every detector is then a **predicate over pairs of events**. Lockset's predicate is:

$$
\begin{aligned}
\text{IsPotentialLocksetRace}(i,j) =\;& e_i = \mathrm{MEM}(m_i,a_i,t_i) \wedge e_j = \mathrm{MEM}(m_j,a_j,t_j) \\
&\wedge t_i \ne t_j \wedge m_i = m_j \wedge (a_i = \mathrm{WRITE} \vee a_j = \mathrm{WRITE}) \\
&\wedge L_i(t_i) \cap L_j(t_j) = \emptyset
\end{aligned}
$$

Two different threads, the same location, at least one write, and **no lock held at both accesses**. L<sub>i</sub>(t) is the set of locks thread t holds just before event i.

Two details differ from Eraser:

- **Pairs, not a running intersection.** Eraser intersects one candidate set with every access, which can go empty even when every conflicting pair shares a lock. Three threads writing under {a, b}, {b, c} and {a, c} are mutually exclusive, yet Eraser warns. The pairwise check stays silent.
- **One report per location.** The tool reports the first race on a location and stops watching it. So it only needs to store a set of `(a, t, L)` tuples per location, and drops any access whose tuple is already there.

On JobServer:

| Field | First racing pair reported | Correct? |
| --- | --- | --- |
| `cfg` | M1 ↔ W1 | False alarm: `start()` orders them |
| `w` | M2 ↔ W7 | Right field, **wrong pair** (the dangerous one is W7 ↔ M11) |
| `hits` | M4 ↔ W3 | Real race |
| `clock` | none | Correct: both accesses hold `clockLock` |
| `Buf.data` | M6 ↔ W5 | False alarm: the pool hands the buffer over |

***Table 2: The pairwise lockset check on the JobServer run.***

Two real races found, two false alarms, and one report that points at the harmless pair. Every false alarm comes from synchronization that is not a lock: `start()` and the pool.

## 3. Pure happens-before: precise, but blind in a different way

Pure happens-before turns **every** interaction between threads into a message: `start`, `join`, `notify`, every lock release to the next acquire, and every memory write to later accesses of that location. Two accesses race only if no chain of messages orders them.

![Figure 3: The JobServer run as a space-time diagram](/assets/img/posts/hybrid-race-detection/03-space-time.png)

***Figure 3: The JobServer run as a space-time diagram. Pure HB also adds memory write-to-read edges, not drawn here.***

Following the arrows in Figure 3:

| Field | Pair | Path in Figure 3 | Pure HB says | Right? |
| --- | --- | --- | --- | --- |
| `cfg` | M1 ↔ W1 | M1 → `start()` → W1 | Ordered | ✓ |
| `w` | W7 ↔ M11 | No path from W7 to M11 | **Reported** | ✓ right pair |
| `hits` | M4 ↔ W3 | M4 → M5 releases `clockLock` → W2 acquires it → W3 | Ordered | ✗ **real race missed** |
| `clock` | M5 ↔ W2 | Through `clockLock` | Ordered | ✓ |
| `Buf.data` | M6 ↔ W5, W5 ↔ M9 | Through the pool's lock | Ordered | ✓ |

***Table 3: Pure happens-before on each JobServer field, following the arrows in Figure 3.***

No false alarms, but `hits` slips through. `clockLock` protects `clock`, not `hits`; the ordering is an accident of this schedule. Had W2 taken `clockLock` first, M4 and W3 would be unordered and pure HB would report them. This is the paper's Figure 4, and it is the key observation: **not every happens-before edge reflects real synchronization.**

### 3.1 Every pure HB race is a lockset race

The paper's Theorem 1 says pure HB never reports a pair that lockset would miss. Its proof is omitted, but it is three lines. If both accesses held a common lock *l*, the first thread must release *l* before the second acquires it. That release-to-acquire chain orders the two accesses, so pure HB would not report them.

The converse fails, as `cfg` and `Buf.data` show. This is the "folk theorem": happens-before has fewer false positives than lockset, but more false negatives.

### 3.2 And it is expensive

The best earlier pure HB detector for Java, TRaDe, slowed programs 4 to 15 times relative to an *interpreter*, and JIT-compiled code is about ten times faster still. The cost comes from vector clocks. Every lock release and every memory write becomes a pending message, and each carries a vector with one entry per thread.

## 4. The hybrid: keep only the edges that mean something

The paper's move is to split the work. **Locks are judged by the lockset check, which understands them.** Happens-before is kept only for the few events that really order threads, and lock hand-offs and memory accesses stop being messages.

| Message source | Send | Receive | Pure HB | Hybrid |
| --- | --- | --- | --- | --- |
| `t2.start()` | caller | t2 as it begins | ✓ | ✓ |
| `t2.join()` | t2 as it ends | caller when join returns | ✓ | ✓ |
| `o.notify()` / `notifyAll()` | the notifier | every thread waiting on `o` | ✓ | ✓ |
| User-marked methods (e.g. `put` / `take`) | the marked send | the marked receive |  | ✓ (optional) |
| Lock hand-off | after `REL(l)` | the next `ACQ(l)` | ✓ |  |
| Memory | after each write to *m* | later accesses to *m* | ✓ |  |

***Table 4: Thread messages, and which detector uses them.***

Call the resulting relation ⇢ (the paper draws an arrow with a dot over it). The hybrid check is the lockset check plus one line:

$$
\text{IsPotentialHybridRace}(i,j) \;=\; \text{IsPotentialLocksetRace}(i,j) \;\wedge\; \neg(i \dashrightarrow j) \;\wedge\; \neg(j \dashrightarrow i)
$$

That gives two consequences at once:

- **Few messages, so cheap clocks.** Threads start, join and notify rarely compared with how often they touch memory or locks.
- **Lock edges can no longer hide races.** `clockLock` stops ordering `hits`. Mutual exclusion is handled by the lockset half of the check.

The reports nest neatly:

![Figure 4: Lockset, hybrid and pure happens-before reports as nested sets](/assets/img/posts/hybrid-race-detection/04-nested-sets.png)

***Figure 4: Lockset, hybrid and pure happens-before reports as nested sets, with the paper's own examples placed.***

- **Hybrid ⊆ lockset**, because the hybrid adds a condition to the lockset check.
- **Pure HB ⊆ hybrid**, because of Theorem 1, and because ⇢ has fewer edges than →: a pair that → cannot order, ⇢ cannot order either.

The middle ring is where the trade lives. It holds a **real race pure HB misses** (`globalInt`, our `hits`) and a **false alarm pure HB avoids** (the pooled object, our `Buf.data`). The paper accepts the second to get the first, plus a large speedup.

## 5. Vector clocks, the paper's way

The hybrid decides ⇢ with vector clocks: every live thread keeps one counter per thread, each message carries its sender's vector, and the receiver takes the entrywise max. (The [Lamport clocks post]({% post_url 2026-10-17-lamport-clocks %}) walks through the standard version.) The paper's version differs in two ways:

- **A stamp is one integer.** Each access is stored with only its own thread's entry, v, not the whole vector.
- **Only sends bump the clock.** A thread's own entry rises only right after it sends a message (`start`, `join`, `notify` or an annotated channel). Reads, writes and lock operations leave it alone, so long runs of accesses share one stamp.

The check is then one comparison:

> An earlier access by thread A, stamped v, is ordered before thread B's new access **iff B's entry for A ≥ v**: B has already heard of that point in A's history.

![Figure 5: Vector clocks on the JobServer run](/assets/img/posts/hybrid-race-detection/05-vector-clocks.png)

***Figure 5: Vector clocks on the JobServer run, written as (M-entry, W-entry). The start message tells W about M up to 1, so W1 can see M1 but W3 cannot see M4.***

The check runs online: each time a thread touches *m*, the new access is compared at that moment against the accesses stored for *m*. That is why the trade is deliberate: **few clock updates** (each O(\|T\|), only on sends and receives) and **many clock checks** (each O(1), on every access).

## 6. The hybrid on JobServer

Only one message is sent in the whole run: M3's `start()`. M's own entry goes from 1 to 2 right after it, so M1 and M2 carry stamp 1 and every later M access carries 2. W begins at (1, 1) and never changes. Here is every access that meets a stored access from the other thread:

| New access | Locks held | Compared with | Ordered? (B's entry for A ≥ v) | Locks shared? | Result |
| --- | --- | --- | --- | --- | --- |
| W1 READ `cfg` | {} | M1, v = 1 | **yes**: 1 ≥ 1 | no | no race |
| W2 WRITE `clock` | {clockLock} | M5, v = 2 | no: 1 < 2 | **yes** | no race |
| W3 READ `hits` | {} | M4, v = 2 | no: 1 < 2 | no | **race M4 ↔ W3** |
| W5 WRITE `Buf.data` | {} | M6, v = 2 | no: 1 < 2 | no | **race M6 ↔ W5** (false alarm) |
| W7 WRITE `w` | {} | M2, v = 1 | **yes**: 1 ≥ 1 | no | no race; W7 is stored |
| M11 READ `w` | {srv} | W7, v = 1 | no: 0 < 1 | no | **race W7 ↔ M11** |

***Table 5: The hybrid check on every cross-thread comparison in the JobServer run.***

The `w` rows are the ones I like best. Lockset alone reported M2 ↔ W7, the harmless pair, and then stopped watching `w`. The hybrid knows `start()` orders M2 before W7, so it keeps looking and reports **W7 ↔ M11**, the pair that can crash. The paper saw the same effect on real programs: lockset's first report on a field was sometimes false, while the hybrid skipped it and found a true race on the same field.

Score so far: both real races found, `cfg` correctly silent, and one false alarm left on `Buf.data`.

## 7. Removing the pool false alarm

The `Buf.data` alarm is the paper's **object-pool problem**: ownership moves through the pool's lock, and the hybrid deliberately ignores lock hand-offs. The tool offers two fixes:

- **Automatic, for blocking channels.** If a class calls `wait` or `notify`, every release of that class's locks becomes a send and the next acquire a receive. A blocking queue is caught this way. `BufPool` never blocks, so it is not.
- **Manual, for everything else.** The user marks `put` as a send and `take` as a receive.

With `BufPool` marked, the new messages move the clocks. M7's `put` bumps M to (3, 0), and W's `take` at W4 lifts W to (2, 1). At W5 the check against M6 now passes: 2 ≥ 2, ordered. The return trip works the same way, so M9 is ordered after W5. The real races survive. W3 still knows M only up to 1, below M4's stamp of 2. And W's send at W6 lifts W7's stamp to 2, while M11 knows W only up to 1.

The annotation is **trusted, not checked**. It tells the detector that everything a thread did before `put` happens before everything the taker does after `take`, for all data, not just the buffer. A race on some other field that happens to straddle a `put`/`take` in the observed run is then hidden, exactly as `clockLock` hid `hits` from pure HB. (A thread that keeps writing a buffer after returning it is still caught: its clock moves past the send.)

Here is every detector on the same run:

| Field | Truth | Lockset | Lockset + thread locality | Pure HB | Hybrid | Hybrid + pool marked |
| --- | --- | --- | --- | --- | --- | --- |
| `cfg` | safe | false alarm | — | — | — | — |
| `w` | bug | found (wrong pair) | found | found | found | found |
| `hits` | race | found | found | **missed** | found | found |
| `clock` | safe | — | — | — | — | — |
| `Buf.data` | safe | false alarm | false alarm | — | false alarm | — |
| **Real found / false** |  | 2 / 2 | 2 / 1 | 1 / 0 | 2 / 1 | **2 / 0** |

***Table 6: Every detector on the JobServer run.***

"Lockset + thread locality" is the paper's **Detailed-NoHB** baseline, its stand-in for earlier lockset tools. It is lockset with a filter that ignores an object's accesses until a second thread touches it (section 8.3). That avoids the `cfg` alarm by luck of timing: M1 happens while only M has touched `srv`.

## 8. Making it fast

Storing and comparing every access is hopeless, so the detector **throws away accesses that can never be the only witness of a race**. A new access e<sub>n</sub> on location *m* is **redundant** if every future access that would race with it also races with something already stored. Since the tool reports one race per location, dropping e<sub>n</sub> loses nothing.

### 8.1 Lockset-subset

The first rule drops e<sub>n</sub> when one stored access e<sub>i</sub> is at least as dangerous in every respect:

$$
\begin{aligned}
\text{IsRedundantLocksetSubset}(i,n) =\;& e_i = \mathrm{MEM}(m,a_i,t_i) \wedge e_n = \mathrm{MEM}(m,a_n,t_n) \\
&\wedge (a_i = \mathrm{WRITE} \vee a_i = a_n) \wedge t_i = t_n \\
&\wedge L_i(t_i) \subseteq L_n(t_n) \wedge V_i(t_i)(t_i) = V_n(t_n)(t_n)
\end{aligned}
$$

Same thread, an access type at least as strong, **fewer or equal locks held**, and **the same clock stamp**. The stamp condition is the paper's addition to an earlier rule, and it matters. Suppose M reloads `cfg` after starting W. The reload looks exactly like M1 (a write by M with no locks), so without the stamp test it would be dropped. But M1 is ordered before W1 by `start()` and the reload is not, so dropping it loses a real race. With the test, the reload has stamp 2 instead of 1 and is kept.

Because clocks move only on sends, a thread spends long stretches at one stamp. In a real server, W's loop would repeat W1 thousands of times, and all the repeats collapse into one stored tuple.

### 8.2 Oversized-lockset

The second rule is new in this paper. It catches reads of effectively read-only data made under **assorted unrelated locks**, where no stored lockset is a subset of another.

![Figure 6: The oversized-lockset argument on four reads of cfg](/assets/img/posts/hybrid-race-detection/06-oversized-lockset.png)

***Figure 6: The oversized-lockset argument on four reads of `cfg`.***

A future write that races with the new read but with none of the stored ones must hold a lock from every stored lockset: a **hitting set**. If even the smallest hitting set has more than N locks, where N is the most locks any thread holds at once, no such write can exist and the new read is redundant. Minimum hitting set is NP-complete, so the tool uses a cheap lower bound: count sets that are pairwise disjoint. N itself comes from the largest lockset seen in a first, cheap run.

### 8.3 Two runs

![Figure 7: The detector's architecture](/assets/img/posts/hybrid-race-detection/07-architecture.png)

***Figure 7: The detector's architecture: two runs of the program with different instrumentation (after the paper's Figure 5).***

The tool runs the program twice. A cheap **simple mode** watches every field with an Eraser-like running intersection and a **thread-locality filter**: an object's accesses are ignored until a second thread touches it. Only the fields it flags get the expensive hybrid check in the second, **detailed** run. On JobServer, simple mode flags `w`, `hits` and `Buf.data`.

The filter is schedule-sensitive, and that is a real cost. Had M run M4 before W1, `srv` would still have been M's alone, the write to `hits` would be skipped, and simple mode would never flag `hits`. The detailed run would then never look at it.

## 9. What it found on real programs

The authors ran eleven Java programs, from small Java Grande kernels up to the Resin and Tomcat web servers (over 67,000 and 54,000 lines). These are the programs where something was reported:

| Program | Simple run (slowdown) | Detailed run, selected fields (slowdown) | Hybrid: bug–benign–false fields | Lockset baseline: bug–benign–false fields |
| --- | --- | --- | --- | --- |
| hedc | not timed | not timed | 4–0–0 | 3–0–1 |
| tsp | 16.87× | 2.50× | 1–0–2 | 0–0–2 |
| mtrt | 4.23× | 1.08× | 2–3–0 | 2–2–7 |
| raytracer | 26.89× | 1.04× | 1–0–0 | 0–0–0 |
| specjbb | 2.94× | 2.21× | 0–1–1 | 0–1–1 |
| resin | 1.91× | 2.06× | 1–18–57 | 1–25–152 |
| tomcat | 2.68× | 1.26× | 3–6–13 | 2–6–17 |

***Table 7: Highlights from the paper's Tables 2 and 3. The lockset baseline is Detailed-NoHB (the hybrid with happens-before off and the thread-locality filter on). hedc is not CPU-bound and was not timed.***

What I take from it:

- **The detailed run is cheap once fields are selected**: 1.04× to 2.5×. Detailed mode on *every* field is often unusable (190× on raytracer, no finish on specjbb and resin), so the two-run design is not optional.
- **Happens-before cuts false alarms where it matters**: 57 false fields on Resin instead of 152, 0 on mtrt instead of 7.
- **It also finds more bugs**: one more bug field in hedc, tsp, raytracer and Tomcat, partly through the same effect as `w`.
- **The happens-before checks are nearly free.** Turning them off sometimes made runs *slower*, because more races got reported; one Tomcat report reached 25 MB.
- **Both redundancy rules earn their keep.** Without them most detailed runs on all fields did not finish in 30 minutes; specjbb needs oversized-lockset even on selected fields.

The bugs ranged from possible aborts (hedc) and hangs (Resin) to data corruption (Tomcat, mtrt) and wrong answers (tsp). In most cases the programmer seems not to have realized the data was shared at all. Most real races, though, were benign by design: double-checked locking, lazy initialization, timestamps, statistics counters. The authors note many of these are only safe on the JVMs of the time.

## 10. References

- Robert O'Callahan and Jong-Deok Choi. *Hybrid Dynamic Data Race Detection*. PPoPP 2003.
- Stefan Savage, Michael Burrows, Greg Nelson, Patrick Sobalvarro and Thomas Anderson. *Eraser: A Dynamic Data Race Detector for Multithreaded Programs*. ACM TOCS, 1997.
- Leslie Lamport. *Time, Clocks, and the Ordering of Events in a Distributed System*. Communications of the ACM, 1978.
- Jong-Deok Choi, Keunwoo Lee, Alexey Loginov, Robert O'Callahan, Vivek Sarkar and Manu Sridharan. *Efficient and Precise Datarace Detection for Multithreaded Object-Oriented Programs*. PLDI 2002.
- Anne Dinning and Edith Schonberg. *Detecting Access Anomalies in Programs with Critical Sections*. ACM/ONR Workshop on Parallel and Distributed Debugging, 1991.
- Koushik Sen. *Race Directed Random Testing of Concurrent Programs*. PLDI 2008.
- Cormac Flanagan and Stephen Freund. *FastTrack: Efficient and Precise Dynamic Race Detection*. PLDI 2009.
