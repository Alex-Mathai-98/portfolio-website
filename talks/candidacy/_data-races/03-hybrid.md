### 3. Hybrid detection: lockset plus limited happens-before

**Paper:** Robert O'Callahan and Jong-Deok Choi, *Hybrid Dynamic Data Race Detection*, PPoPP 2003. [Read the full post]({% post_url 2026-11-10-hybrid-race-detection %}).

> Report a race only when **no common lock** was held **and** a deliberately **limited** happens-before relation cannot order the two accesses.

#### A small program with three fields

I'll use one small Java program, **JobServer**, for the whole section. A main thread **M** starts one worker **W**, and the two share three fields. **One field holds a real race, one is safe but fools lockset, and one is correctly locked.** Each field copies a case from the paper's own figures.

![Figure 1: The JobServer program](/assets/img/talks/candidacy/hybrid-figure-1-jobserver-code.svg)

***Figure 1: The JobServer program, built from cases in the paper's own figures.***

- `cfg` is the paper's `globalFlag`: written once before W starts, then only read.
- `hits` is the paper's `globalInt`: written by M and read by W with no lock.
- `clock` is the counter next to it, guarded by `clockLock`.

A detector sees one run. Here is the one we'll replay, in the order of the circled numbers:

![Figure 2: The observed run of JobServer](/assets/img/talks/candidacy/hybrid-figure-2-observed-run.svg)

***Figure 2: The observed run of JobServer, step by step, with the events each step emits. MEM = memory access, ACQ = lock acquire, REL = lock release, SND = message send, RCV = message receive (g<sub>s</sub> is the start message).***

Nothing goes wrong in this run: W happens to read 7. A good detector must still warn about `hits`, because the truth is this:

| Field | What protects it | Verdict | Why |
| --- | --- | --- | --- |
| `cfg` | `start()` | Safe | Written once before W exists, only read afterwards |
| `hits` | Nothing | **Real race** | In another schedule W3 runs before M4 and W reads 0 |
| `clock` | `clockLock` | Safe | Every access holds `clockLock` |

***Table 1: What a perfect detector would say about each JobServer field.***

A perfect detector reports `hits`, and nothing else. Let's see who manages it.

#### Lockset: cheap, but it cries wolf

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
| `hits` | M4 ↔ W3 | Real race |
| `clock` | none | Correct: both accesses hold `clockLock` |

***Table 2: The pairwise lockset check on the JobServer run.***

The real race is found, but so is one false alarm. It comes from synchronization that is not a lock: `start()`.

#### Pure happens-before: precise, but blind in a different way

Pure happens-before turns **every** interaction between threads into a message: `start`, `join`, `notify`, every lock release to the next acquire, and every memory write to later accesses of that location. Two accesses race only if no chain of messages orders them.

![Figure 3: The JobServer run as a space-time diagram](/assets/img/talks/candidacy/hybrid-figure-3-space-time.svg)

***Figure 3: The JobServer run as a space-time diagram. Pure HB also adds memory write-to-read edges, not drawn here.***

Following the arrows in Figure 3:

| Field | Pair | Path in Figure 3 | Pure HB says | Right? |
| --- | --- | --- | --- | --- |
| `cfg` | M1 ↔ W1 | M1 → `start()` → W1 | Ordered | ✓ |
| `hits` | M4 ↔ W3 | M4 → M5 releases `clockLock` → W2 acquires it → W3 | Ordered | ✗ **real race missed** |
| `clock` | M5 ↔ W2 | Through `clockLock` | Ordered | ✓ |

***Table 3: Pure happens-before on each JobServer field, following the arrows in Figure 3.***

No false alarms, but `hits` slips through. `clockLock` protects `clock`, not `hits`; the ordering is an accident of this schedule. Had W2 taken `clockLock` first, M4 and W3 would be unordered and pure HB would report them. This is the paper's Figure 4, and it is the key observation: **not every happens-before edge reflects real synchronization.**

##### Every pure HB race is a lockset race

The paper's Theorem 1 says pure HB never reports a pair that lockset would miss. Its proof is omitted, but it is three lines. If both accesses held a common lock *l*, the first thread must release *l* before the second acquires it. That release-to-acquire chain orders the two accesses, so pure HB would not report them.

The converse fails, as `cfg` shows. This is the "folk theorem": happens-before has fewer false positives than lockset, but more false negatives.

##### And it is expensive

The best earlier pure HB detector for Java, TRaDe, slowed programs 4 to 15 times relative to an *interpreter*, and JIT-compiled code is about ten times faster still. The cost comes from vector clocks. Every lock release and every memory write becomes a pending message, and each carries a vector with one entry per thread.

#### The hybrid: keep only the edges that mean something

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

The middle ring is where the trade lives. It holds a **real race pure HB misses** (`globalInt`, our `hits`) and a **false alarm pure HB avoids** (the paper's pooled object, handed between threads through a lock-protected pool). The paper accepts the second to get the first, plus a large speedup.

#### Vector clocks, the paper's way

The hybrid decides ⇢ with vector clocks: every live thread keeps one counter per thread, each message carries its sender's vector, and the receiver takes the entrywise max. (The [Lamport clocks post]({% post_url 2026-10-17-lamport-clocks %}) walks through the standard version.) The paper's version differs in two ways:

- **A stamp is one integer.** Each access is stored with only its own thread's entry, v, not the whole vector.
- **Only sends bump the clock.** A thread's own entry rises only right after it sends a message (`start`, `join`, `notify` or an annotated channel). Reads, writes and lock operations leave it alone, so long runs of accesses share one stamp.

The check is then one comparison:

> An earlier access by thread A, stamped v, is ordered before thread B's new access **iff B's entry for A ≥ v**: B has already heard of that point in A's history.

![Figure 5: Vector clocks on the JobServer run](/assets/img/posts/hybrid-race-detection/05-vector-clocks.png)

***Figure 5: Vector clocks on the JobServer run, written as (M-entry, W-entry). The start message tells W about M up to 1, so W1 can see M1 but W3 cannot see M4.***

The check runs online: each time a thread touches *m*, the new access is compared at that moment against the accesses stored for *m*. That is why the trade is deliberate: **few clock updates** (each O(\|T\|), only on sends and receives) and **many clock checks** (each O(1), on every access).

#### The hybrid on JobServer

Only one message is sent in the whole run: M3's `start()`. M's own entry goes from 1 to 2 right after it, so M1 carries stamp 1 and every later M access carries 2. W begins at (1, 1) and never changes. Here is every access that meets a stored access from the other thread:

| New access | Locks held | Compared with | Ordered? (B's entry for A ≥ v) | Locks shared? | Result |
| --- | --- | --- | --- | --- | --- |
| W1 READ `cfg` | {} | M1, v = 1 | **yes**: 1 ≥ 1 | no | no race |
| W2 WRITE `clock` | {clockLock} | M5, v = 2 | no: 1 < 2 | **yes** | no race |
| W3 READ `hits` | {} | M4, v = 2 | no: 1 < 2 | no | **race M4 ↔ W3** |

***Table 5: The hybrid check on every cross-thread comparison in the JobServer run.***

The hybrid gets all three right: the race on `hits` is reported, and `cfg` and `clock` stay silent.

Here is every detector on the same run:

| Field | Truth | Lockset | Lockset + thread locality | Pure HB | Hybrid |
| --- | --- | --- | --- | --- | --- |
| `cfg` | safe | false alarm | — | — | — |
| `hits` | race | found | found | **missed** | found |
| `clock` | safe | — | — | — | — |
| **Real found / false** |  | 1 / 1 | 1 / 0 | 0 / 0 | **1 / 0** |

***Table 6: Every detector on the JobServer run.***

"Lockset + thread locality" is the paper's **Detailed-NoHB** baseline, its stand-in for earlier lockset tools. It is lockset with a filter that ignores an object's accesses until a second thread touches it. That avoids the `cfg` alarm by luck of timing: M1 happens while only M has touched `srv`.
