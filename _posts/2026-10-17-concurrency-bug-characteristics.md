---
title: "Concurrency - Basics, Bug Types, Root Causes, & Fixes"
categories: [Technical]
math: false
mermaid: false
---

## Tutorial Outline

Content has been taken from *Learning from Mistakes — A Comprehensive Study on Real World Concurrency Bug Characteristics* (Lu, Park, Seo and Zhou, ASPLOS 2008).

**How to read this tutorial**

- **Part 1** builds the concurrency vocabulary you need. 

- **Part 3** explains how the study was done.

- **Parts 4–7** walk through the four result areas, one finding at a time, each tied to a code example from the paper.

- **Part 8** covers the rest, critiques the study, and places it in history.

- The **cheat sheet** at the end lists all 13 findings and gives self-check questions.

## Part 1 — Concurrency basics you need first

A concurrency bug is a bug that appears only under some orderings of threads. Everything in the paper rests on five ideas: threads, interleavings, locks, and three classic bug types.

### 1.1 Threads share memory

A **thread** is an independent stream of instructions.

- Threads in one process share the same heap and globals, so two threads can read and write the same variable. This is the key difference from multi-processing: threads of one process share the same memory, while each process gets its own separate copy.

- The OS scheduler decides when each thread runs, and it can switch threads between any two machine instructions. Two threads can run simultaneously on two separate CPU cores, or share a single core through intermittent context switching.

### 1.2 Interleavings: why `count++` is not one step

In C, `count++` compiles to roughly three instructions:

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-01.svg" alt="Figure 1" />
</figure>

***Figure 1:*** *count++ compiles to three machine steps (load, add, store).*

Run it in two threads, starting from `count = 0`. One possible ordering of their instructions — an **interleaving** — is:

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-02.svg" alt="Figure 2" />
</figure>

***Figure 2:*** *One bad interleaving of count++ in two threads loses an update.*

The program is correct under most interleavings and wrong under a few. That is why these bugs are hard: the same input can pass 999 times and fail once. The number of interleavings grows **exponentially** with the number of instructions, so you cannot test them all. Keep this in mind; Finding 8 is about shrinking this space.

### 1.3 Locks and critical sections

A **lock** (mutex) lets only one thread at a time run a protected region, called a **critical section**:

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-03.svg" alt="Figure 3" />
</figure>

***Figure 3:*** *A mutex makes count++ one uninterruptible critical section.*

**Key point: what locks do and do not give you.**

- **Locks give mutual exclusion:** "not at the same time."

- **Locks do not give ordering:** "A before B."

This distinction drives several of the paper's findings.

Locks also differ in what happens when a thread tries to take a lock it already holds:

- **Non-reentrant lock:** it does not track who holds it. If the owner calls `lock()` again, it waits for itself forever, which is a **self-deadlock** (Part 1.7). A default `pthread_mutex_t` in C works this way.

- **Reentrant (recursive) lock:** it remembers its owner and keeps a count. The owner can lock it again without waiting (the count goes up), and the lock is released only when the count returns to 0. Examples are Java's `synchronized` and `ReentrantLock`, or a pthread mutex created with `PTHREAD_MUTEX_RECURSIVE`.

The same `outer()` / `inner()` example from Figure 10 (section 1.7), now with a reentrant lock `m`, run by thread T1:

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-04.svg" alt="Figure 4" />
</figure>

***Figure 4:*** *Reentrant lock walk-through: the owner can re-lock, and only the outermost unlock releases.*

The owner check on lines 8 and 15 prevents the self-deadlock. The count makes sure only the outermost `unlock()` on line 4 releases the lock. The unlocks on lines 18 and 11 just lower the count, so `inner()` cannot accidentally free the lock while `outer()` is still in its critical section.

### 1.4 Data race

A **data race** is two accesses to the same variable, from different threads, with at least one write, and no common lock or other synchronization between them. The `count++` example is a data race.

**The key question when hunting for a data race:**

> ***"If these two threads ran simultaneously on two CPUs, could they access the same memory at the same time?"***
>
> If yes, and at least one access is a write, you have a data race.

Two subtleties the paper relies on:

> **Subtlety 1 — Not every race is a bug.** A *benign race* is an intentional race whose every interleaving still gives the intended result. The classic case is a spin flag:

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-05.svg" alt="Figure 5" />
</figure>

***Figure 5:*** *A benign race on a spin flag (racy accesses shaded).*

This is a data race: Thread 1 writes `done` while Thread 2 reads it, with no lock. But walk through the possible orders:

- Thread 2 reads `done` before the write → sees 0 → keeps spinning and reads again.

- Thread 2 reads `done` after the write → sees 1 → leaves the loop and starts phase 2.

Every order ends the same way: phase 2 starts only after phase 1 is done. A race detector will still flag it, so tools that report every race produce false alarms. That is one reason the paper does not use "data race" as a bug category.

*Caveat for modern code:* in today's C/C++ a plain racy `int` here is undefined behaviour. The compiler may keep `done` in a register and spin forever. Write it with `atomic_int` (or `std::atomic<bool>`), which keeps the same logic but makes it well defined.

> **Subtlety 2 — No race does not mean no bug.** Every access below is protected by the same lock `m`, so a race detector reports nothing. The program is still wrong because it lets the user overdraw the account.

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-06.svg" alt="Figure 6" />
</figure>

***Figure 6:*** *Bank withdrawal: every access is locked, but the check (line 4) and the act (line 6) are separate.*

One bad interleaving:

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-07.svg" alt="Figure 7" />
</figure>

***Figure 7:*** *Two withdraw(80) calls interleave and overdraw the account.*

**Ask the Key question:** *"If these two threads ran simultaneously on two CPUs, could they access the same memory at the same time?"* **No**, because every access to `balance` holds lock `m`. So there is no data race.

The bug is that the check and the act were meant to be **one atomic step**, and another thread slipped in between. That is an atomicity violation (next section). The fix is to hold the lock across both the check and the act.

> **Fix pattern — data race:** Add Locking to make every conflicting access synchronized.
>
> - Put every read and write of the shared variable under the **same lock**, e.g. `lock(m); count++; unlock(m);`.
>
> - Or use an **atomic operation**, e.g. `atomic_fetch_add(&count, 1)`. For an intentional flag like `done`, declare it `atomic_int`.
>
> - Removing the race is not the whole job: as Subtlety 2 shows, the code can still be wrong at a higher level.

### 1.5 Atomicity violation

The programmer *assumes* a group of accesses runs as one indivisible unit, but nothing enforces it.

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-08.svg" alt="Figure 8" />
</figure>

***Figure 8:*** *Atomicity violation: Thread 2 runs between Thread 1's check and use.*

The check and the use were meant to be atomic together. The fix is to hold one lock across both: take the lock at step 1 and release it only at step 9, by deleting the unlock at step 3 and the lock at step 7. Thread 2's steps 4–6 then cannot run until Thread 1 has finished the check (step 2) and the use (step 8).

> **Fix pattern — atomicity violation:** Change Locking to make the whole check-then-act region one unit.
>
> - **Widen the critical section:** hold one lock across the check and the use (steps 1–9 above).
>
> - **Read once into a local:** copy the shared value into a local variable, then check and use the local, so a later change cannot affect you.
>
> - **Optimistic retry:** do the work, then check the shared value has not changed; if it has, redo it (the MySQL fix in Part 6).

### 1.6 Order violation

The programmer *assumes* operation A always happens before operation B in another thread, but nothing enforces it.

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-09.svg" alt="Figure 9" />
</figure>

***Figure 9:*** *Order violation: Thread 2 reads worker before Thread 1 assigns it.*

The line `worker = create_thread(run);` is really two steps: the call that starts Thread 2 (step 1) and storing its result in `worker` (step 4). Thread 2 can run in between and read `worker` at step 3, before it is set.

Putting locks around both lines does not help: the lock still lets Thread 2 enter first. You need a condition variable, a flag, or a reordering of the code. This is the paper's Figure 2 (drawn as blog Figure 13 in 4.2.1).

> **Fix pattern — order violation:** enforce "A before B" explicitly; a lock alone cannot.
>
> - **Make B wait for A:** use a condition variable, semaphore or flag. For example, Thread 2 runs `wait_until(worker_ready);` before `use(worker);`, and Thread 1 signals after assigning `worker`.
>
> - **Reorder the code:** do A before B can possibly start, e.g. finish initializing the data before calling `create_thread`, or pass it to the new thread as an argument.
>
> - **Check and skip:** if B may legitimately run first, have it check whether A has happened and skip or retry (the condition-check fixes in Part 6).

### 1.7 Deadlock

A **deadlock** happens when threads wait for a resource that will never be released, so none of them can proceed. It comes in two forms.

#### Self-deadlock (one thread)

A single thread tries to acquire a non-reentrant lock it already holds (see the lock types in 1.3). It waits for itself forever. No other thread is involved, so no interleaving is needed: it fails the same way every time.

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-10.svg" alt="Figure 10" />
</figure>

***Figure 10:*** *Self-deadlock: one thread waits for a non-reentrant lock it already holds.*

The paper finds these are surprisingly common: 7 of the 31 deadlocks (Finding 4). Because only one thread is involved, they are also the easiest to detect.

#### Cross-deadlock (two or more threads)

Two or more threads each hold a resource and wait for a resource another thread holds, forming a cycle. The classic case is **lock-order inversion**: Thread 1 takes A then B, while Thread 2 takes B then A.

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-11.svg" alt="Figure 11" />
</figure>

***Figure 11:*** *Cross-deadlock from lock-order inversion between two threads.*

### 1.8 Summary

| Term                | What goes wrong                                 | Do locks fix it?                             |
|---------------------|-------------------------------------------------|----------------------------------------------|
| Data race           | Unsynchronized conflicting accesses             | Yes, by definition                           |
| Atomicity violation | A region meant to be indivisible is interrupted | Usually, if the lock covers the whole region |
| Order violation     | A meant-to-be-earlier operation runs later      | Often no: needs ordering, not exclusion      |
| Deadlock            | Circular wait on resources                      | Locks cause it; fix by changing acquisition  |

## Part 3 — Methodology: how the 105 bugs were chosen and classified

The authors keyword-searched bug trackers, randomly sampled about 500 reports, and kept the 105 that were verified concurrency bugs: 74 non-deadlock and 31 deadlock.

### 3.1 The three research directions the study wants to inform

| Direction                      | Open question the paper asks                                       | Example of an assumption being tested                                                    |
|--------------------------------|--------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| **Bug detection**              | Do existing detectors cover the bugs that really occur?            | Race and atomicity detectors look at one variable at a time. Is that enough?             |
| **Testing and model checking** | Can we test a small set of interleavings and still find most bugs? | Testing every interleaving is impossible. How many threads and accesses must we control? |
| **Language design**            | Would transactional memory (TM) prevent these bugs?                | TM expresses "this region is atomic." Do bugs need anything else?                        |

### 3.2 The four applications

All four are mature C/C++ programs with 9–13 years of history and 1–4 million lines of code. They cover both ends of how concurrency is used:

| Application | Kind            | Why it uses threads                                | Non-deadlock bugs | Deadlock bugs |
|-------------|-----------------|----------------------------------------------------|-------------------|---------------|
| MySQL       | Database server | Serve many client requests at once                 | 14                | 9             |
| Apache      | Web server      | Serve many client requests at once                 | 13                | 4             |
| Mozilla     | Browser suite   | Keep the GUI responsive while background work runs | 41                | 16            |
| OpenOffice  | Office suite    | Same as Mozilla                                    | 6                 | 2             |
| **Total**   |                 |                                                    | **74**            | **31**        |

### 3.3 The collection pipeline

1.  **Keyword filter.** Search 500,000+ bug reports for words like *race, deadlock, synchronization, lock, mutex, atomic, compete*. This yields thousands of candidates.

2.  **Random sample.** Pick about 500 reports that have a clear root-cause description, source code and a fix. Random sampling avoids cherry-picking interesting bugs.

3.  **Manual verification.** Read each one and keep only bugs caused by a programmer's *wrong assumption about concurrent execution*. Result: 105 bugs.

4.  **Deep reading.** For each bug, examine the report, source, all patches (including failed ones) and developer discussion.

### 3.4 The three classification dimensions

Each bug is labelled along three axes, defined in the paper's Table 2:

- **Pattern** — which programmer intention was violated: *Atomicity*, *Order*, or *Other*. Deadlocks are studied separately and not sub-divided.

- **Manifestation** — the smallest set *S* of memory accesses such that enforcing one specific order on *S* guarantees the bug appears. From *S* they count threads, variables and accesses.

- **Fix strategy** — how the final patch fixed it, plus any buggy intermediate patches.

Two design choices are worth noticing:

> **Design choice 1 — Pattern is by root cause, not by fix.** The paper's Figure 2 order bug (the Mozilla mThread bug, blog Figure 13) *could* be patched with a lock, but it is still labelled *Order*, because the violated intention was ordering.

> **Design choice 2 — "Data race" is not a pattern.** A race may be benign (a spin flag), and a race-free program can still be buggy (Part 1.5).
>
> ***So races are a symptom, not a category.***

## Part 4 — Bug patterns: what kind of mistake? (Findings 1–2)

72 of the 74 non-deadlock bugs fall into just two patterns: atomicity violation or order violation. The second pattern alone covers about a third (24 bugs), and 2008-era detectors did not look for it.

| Application | Atomicity | Order  | Other | Total  |
|-------------|-----------|--------|-------|--------|
| MySQL       | 12        | 1      | 1     | 14     |
| Apache      | 7         | 6      | 0     | 13     |
| Mozilla     | 29        | 15     | 0     | 41     |
| OpenOffice  | 3         | 2      | 1     | 6      |
| **Overall** | **51**    | **24** | **2** | **74** |

The columns sum to 77, not 74: three bugs fit both atomicity and order and are counted in both.

### 4.1 Atomicity violation — paper's Figure 1 (MySQL `ha_innodb.cc`)

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-12.svg" alt="Figure 12" />
</figure>

***Figure 12:*** *The paper's Figure 1 (MySQL `ha_innodb.cc`): an atomicity violation, step by step.*

**The intention:** S1 and S2 form one unit, so the value checked is the value used. **The violation:** S3 slips between them. This is the check-then-act shape from Part 1.5. Programmers write it because they think sequentially; two adjacent lines *feel* atomic.

### 4.2 Order violation

The paper shows that order intentions come in more than one shape:

| Flavour                       | Example                                              | What should come first                                              |
|-------------------------------|------------------------------------------------------|---------------------------------------------------------------------|
| Write → read, one variable    | Paper's Figure 2 (blog Figure 13)                    | Assign `mThread` before Thread 2 reads it                           |
| Write → write, one variable   | Paper's Figure 4, Mozilla `macio.c` (blog Figure 14) | Set `io_pending = TRUE` before the async callback sets it `FALSE`   |
| Group → group, many variables | Paper's Figure 5, Mozilla JS engine (blog Figure 15) | All `js_MarkAtom` calls before `js_UnpinPinnedAtom` frees the atoms |

#### 4.2.1 Write → read: Mozilla `nsthread.cpp` (paper's Figure 2)

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-13.svg" alt="Figure 13" />
</figure>

***Figure 13:*** *The paper's Figure 2 (Mozilla `nsthread.cpp`): an order violation, step by step.*

**The intention:** `mThread` is assigned before Thread 2 reads it. It *seems* guaranteed, because Thread 1 creates Thread 2. But the new thread can start running before `PR_CreateThread` returns and the assignment happens. Thread 2 then dereferences an uninitialized pointer.

#### 4.2.2 Write → write: Mozilla `macio.c` (paper's Figure 4)

Walk through the paper's Figure 4, because it is subtle:

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-14.svg" alt="Figure 14" />
</figure>

***Figure 14:*** *The paper's Figure 4 (Mozilla `macio.c`): a write-write order violation that causes a hang.*

If the I/O finishes very fast, S4 runs *before* S2. Then S2 sets the flag to TRUE after the "done" signal, and S3 spins forever: a **hang**. The developers' fix was to swap S1 and S2 (Part 6).

#### 4.2.3 Group → group: Mozilla JS engine (paper's Figure 5)

Several threads run `js_DestroyContext`. Each thread that is not the last one calls `js_MarkAtom`, which may touch elements of the shared `atoms` array. The last thread calls `js_UnpinPinnedAtom`, which frees them. The intended order is between two **groups** of operations: every mark must finish before the unpin. If the unpin runs first, a later mark touches freed atoms and the program crashes. Part 6.4 shows how the developers fixed this bug, and how their first fix failed.

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-15.svg" alt="Figure 15" />
</figure>

***Figure 15:*** *The paper's Figure 5 (Mozilla `jscntxt.c`, `jsgc.c`): a group-to-group order violation, step by step.*

### 4.3 The "Other" category — paper's Figure 3 (MySQL)

Only 2 bugs fit neither pattern. In one, a monitor thread crashes the server if any thread waits on a lock longer than `fatal_timeout`, assuming that means deadlock. Under heavy load (2,048 worker threads), normal waits exceeded the timeout and the server kept killing itself. The broken assumption was about **performance**, not atomicity or order.

### 4.4 Findings and implications

> **Finding 1.** 97% (72/74) of non-deadlock bugs are atomicity or order violations. **Implication:** detection, testing and language design should focus on these two patterns first.

> **Finding 2.** 32% (24/74) are order violations. **Implication:** we need new tools for order bugs. A race detector or atomicity checker misses them, because two accesses can be perfectly locked and still run in the wrong order.

**Why this mattered:** before this paper, "concurrency bug" was nearly synonymous with "data race" or "deadlock." Naming *order violation* as a first-class category, and showing it is common, redirected a decade of tool research.

## Part 5 — Manifestation: what does it take to trigger a bug? (Findings 3–8)

Real concurrency bugs are **"shallow"**. This is the most practically important part of the paper for testing.

- **Few threads:** 101 of 105 bugs need at most two threads.

- **Few accesses:** 97 of 105 (92%) need an order among at most 4 memory accesses.

- **One thread = self-deadlock only:** the 7 single-thread bugs are all self-deadlocks, where a thread waits on a lock it already holds. No interleaving is involved.

- **Two threads for everything else**, with four exceptions: 2 bugs need more than two threads, and 2 need an external event, such as another program touching the same file.

### 5.1 The key definition: manifestation condition

The **manifestation condition** of a bug is the smallest set *S* of memory accesses plus a specific order on *S*, such that if that order happens — however it is forced — the bug is **guaranteed** to appear.

**Worked example — the paper's Figure 1 (blog Figure 12).** *S* = {S1, S3, S2}. The order is S1 → S3 → S2. So:

- threads = 2 (Thread 1 runs S1 and S2; Thread 2 runs S3)

- variables = 1 (`thd->proc_info`)

- accesses = 3

Every other instruction in the program, and every other thread, is irrelevant. That is what makes a small manifestation condition so valuable: a tester only has to force three accesses into one order.

### 5.2 How many threads? (Findings 3–4)

| Bug type          | 1 thread | 2 threads | \> 2 threads | Needs external event |
|-------------------|----------|-----------|--------------|----------------------|
| Non-deadlock (74) | 0        | 71        | 1            | 2                    |
| Deadlock (31)     | 7        | 23        | 1            | 0                    |

> **Finding 3.** 96% (101/105) of bugs involve no more than two threads. **Implication:** test threads *pairwise*. Instead of exploring all orderings of 1,000 threads, explore orderings between each pair.

**But don't bugs show up under heavy load?** Yes, and the paper reconciles this. Heavy load creates more context switches and contention, which raises the *probability* of hitting the bad order between two threads. The condition itself still involves only two. Load is a dice-roll multiplier, not a structural requirement.

The 2 "environment" bugs depend on something outside the program. In one Mozilla bug, another process had to modify the same file at the same time.

> **Finding 4.** 22% (7/31) of deadlocks involve a single thread, typically re-acquiring a lock it already holds. **Implication:** these are easy to detect without reasoning about other threads, so eliminate them first.

This is the self-deadlock drawn in Figure 10 (section 1.7).

### 5.3 How many variables? (Findings 5–7)

| Bug type          | Involves 1 variable / resource | Involves more than 1 | Breakdown of "more than 1"                     |
|-------------------|--------------------------------|----------------------|------------------------------------------------|
| Non-deadlock (74) | 49                             | 25                   | Not split further in the paper                 |
| Deadlock (31)     | 7                              | 24                   | 23 involve 2 resources, 1 involves more than 2 |

> **Finding 5.** 66% (49/74) of non-deadlock bugs involve one variable. **Implication:** the single-variable focus of race detectors (Eraser, RaceTrack) and atomicity checkers (AVIO, SVD) is a reasonable simplification.

> **Finding 6.** 34% (25/74) involve multiple variables. **Implication:** we need new detectors for multi-variable bugs, because single-variable tools miss them by construction.

**Worked example — the paper's Figure 6 (Mozilla text frame; blog Figure 16).** Three fields together describe a string region:

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-16.svg" alt="Figure 16" />
</figure>

***Figure 16:*** *The paper's Figure 6 (Mozilla text frame): a multi-variable concurrency bug, step by step.*

Between Cut and Reflow, the three fields are mutually inconsistent. If Thread 1 paints in that window, it indexes past the end of the shortened string and crashes.

The key insight: look at any **single** variable and nothing is wrong. Reading `mContent` before or after Thread 2's change is fine on its own. The bug is that Thread 1 reads the *trio* while Thread 2 is half-way through updating it. A per-variable detector sees no problem. The hard research question is *which* variables are semantically linked; the same group later built MUVI to infer such correlations from code.

> **Finding 7.** 97% (30/31) of deadlocks involve at most two resources. **Implication:** deadlock testing can check the acquire/release order of each pair of resources.

### 5.4 How many accesses? (Finding 8)

Even with 2 threads and 1 variable, a thread might touch that variable millions of times. So the authors count the accesses in *S*.

| Accesses in *S* | Non-deadlock | Deadlock           |
|-----------------|--------------|--------------------|
| 1               | 0            | 7 (self-deadlocks) |
| 2               | 22           | 3                  |
| 3               | 33           | 20                 |
| 4               | 12           | 0                  |
| \> 4            | 7            | 1                  |

> **Finding 8.** 90% (67/74) of non-deadlock bugs and 97% (30/31) of deadlocks manifest deterministically once an order among ≤ 4 accesses is enforced. **Implication:** test all orders among every small group of accesses. This turns an exponential search into a polynomial one.

**Why exponential vs polynomial? A small calculation.** Take two threads, each executing *n* shared-memory accesses.

Use the MySQL bug from Figure 12, and give each thread 3 accesses to `thd->proc_info`:

- **Thread 1:** a1 (`S1` check), a2 (`S2` use), a3

- **Thread 2:** b1, b2 (`S3` write NULL), b3

The bug fires exactly when **a1 → b2 → a2** happens. That is 3 accesses, so this bug has depth *k* = 3.

**Option 1 — try every interleaving.** Merging two lists of *n* accesses gives "2n choose n" schedules. For *n* = 3 that is 20; for *n* = 10 it is 184,756; for *n* = 30 it is about 1.2 × 10^17. The count grows **exponentially**, so this is hopeless at real sizes.

**Option 2 — force only small groups.** Pick every group of *k* = 3 accesses that spans both threads, and try each order of just that group while the rest runs freely.

- With 6 accesses there are "6 choose 3" = 20 groups. Dropping the 2 single-thread groups leaves **18**.

- Program order is fixed inside a thread (a1 always precedes a2), so each group has only **3** possible orders. For the group {a1, a2, b2}:

| Forced order     | Result                                                              |
|------------------|---------------------------------------------------------------------|
| b2 → a1 → a2     | NULL is set first, so the check fails and nothing is printed. Safe. |
| **a1 → b2 → a2** | Check passes, then NULL, then use. **Crash.**                       |
| a1 → a2 → b2     | The print finishes before NULL is set. Safe.                        |

- That is 18 × 3 = **54 small experiments**. Because {a1, b2, a2} is one of the groups, one experiment is **guaranteed** to force the crashing order.

**In general:** picking a group of *k* accesses and then ordering it is the same as picking an ordered list of *k* accesses. So there are at most C(2n, k) × k! = 2n(2n − 1)…(2n − k + 1) ≈ **(2n)^k experiments**, which is polynomial in *n* for a fixed *k*. Program order inside each thread cuts the real number further: 54 instead of the bound of 120 here.

**The difference is evident for large *n*:**

| Accesses per thread (*n*) | Every interleaving | Groups of 3 × 3 orders |
|---------------------------|--------------------|------------------------|
| 3                         | 20                 | 18 × 3 = 54            |
| 10                        | 184,756            | 900 × 3 = 2,700        |
| 30                        | ~1.2 × 10^17       | 26,100 × 3 ≈ 78,000    |

Finding 8 says *k* ≤ 4 catches about 92% of bugs, so a tester can bound *k* and still find most of them. This also gave empirical backing to **iterative context bounding** (the CHESS line of work, cited as \[28\]), which explores schedules with few preemptions first.

> **One honest caveat.** Knowing *which* 4 accesses to order is still hard, and forcing that order needs control over inputs and scheduling. The paper explicitly leaves "how to leverage this" as future work.

## Part 6 — How were the bugs fixed? (Findings 9–10)

The authors expected most fixes to add or change locks. They were wrong: locks fixed only 27% of non-deadlock bugs, and 30% of Mozilla bugs got at least one wrong patch before the right one.

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-17.svg" alt="Figure 17" />
</figure>

***Figure 17:*** *Fix strategies for the 74 non-deadlock bugs (Lu et al., ASPLOS'08, Table 8).*

### 6.1 Why locks are not enough

There are two reasons.

1.  **Locks cannot express order.** The paper's Figure 7 (MySQL; not drawn in this post): Thread 1 sets `dynamicId`; Thread 2 reads it in `status()`. Wrapping both in a lock still lets Thread 2 go first and read an uninitialized value. A mutex says "not together," never "A first."

2.  **Even when locks would work, they may be a poor choice.** They cost performance, and adding a lock can create a new deadlock.

### 6.2 The non-lock strategies, with examples

#### 6.2.1 Condition check (COND, 19 bugs)

Add a flag or a consistency check instead of preventing the bad interleaving.

- **\[Order Violation\]** *Spin on a flag* to enforce order: wait until the other thread signals it is done.

  <figure>
  <img src="/assets/img/posts/concurrency-bugs/figure-18.svg" alt="Figure 18" />
  </figure>

  ***Figure 18:*** *Spin-flag fix: the paper's Figure 9(b) final fix for its Figure 5 order bug (blog Figure 15). Thread 1 spins at step 4 until Thread 2 clears `gcLevel` at step 6.*

- **\[Atomicity Violation\]** *Check consistency, then skip or retry.* For the paper's Figure 6 (the text-frame bug, blog Figure 16), Mozilla added `if (strlen(mContent) >= mOffset + mLength)` before `putc`, skipping the paint if the trio is inconsistent. (The paper doesn't label this bug's pattern explicitly. It is tagged atomicity here because Thread 1 reads the three fields in the middle of Thread 2's update to them.)

  <figure>
  <img src="/assets/img/posts/concurrency-bugs/figure-19.svg" alt="Figure 19" />
  </figure>

  ***Figure 19:*** *Consistency-check fix for the paper's Figure 6 text-frame bug (blog Figure 16).*

- **\[Atomicity Violation\]** *Optimistic retry* (the paper's Figure 8, MySQL `btr0sea.c`):

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-20.svg" alt="Figure 20" />
</figure>

***Figure 20:*** *The MySQL `btr0sea.c` bug before the fix. The paper shows only the fixed code; this buggy version is reconstructed from its description.*

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-21.svg" alt="Figure 21" />
</figure>

***Figure 21:*** *The paper's Figure 8 fix (MySQL `btr0sea.c`): optimistic retry.*

This does not stop the bad interleaving; it **detects and recovers** from it. It is faster than locking when conflicts are rare. Notice it is a hand-written transaction, which matters in Part 7.

#### 6.2.2 Code switch (Switch, 10 bugs)

Reorder statements so the intended order holds naturally. The paper's Figure 4 (blog Figure 14) was fixed by swapping S1 and S2: set `io_pending = TRUE` *before* starting the async read, so the callback can never run first.

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-22.svg" alt="Figure 22" />
</figure>

***Figure 22:*** *Code-switch fix for the paper's Figure 4 bug (blog Figure 14): S2 now runs before S1, so the callback's S4 can only come after it. The paper describes this fix in words; the code is reconstructed from that description.*

#### 6.2.3 Design change (Design, 19 bugs)

Change data structures or algorithms so the sharing disappears.

- MySQL \#7209: `HASH::current_record` was shared but did not need to be. Developers made it per-thread. No sharing, no bug.

- Mozilla \#201134: copy the shared `nsCertType` into a local, operate on the local, write it back once.

- Mozilla \#131447: redesign message queueing to tolerate a reply arriving before its callback is registered.

These fixes require understanding program **semantics**, not just which accesses conflict.

> **Finding 9.** Adding or changing locks fixed only 20 of 74 non-deadlock bugs. **Implication:** there is no silver bullet. A tool that only says "these accesses aren't under a common lock" is not enough. Developers also need the bug pattern (atomicity or order?) and the consistency condition that must hold.

### 6.3 Deadlock fixes

| Strategy                 | What it means                                 | Bugs (of 31) |
|--------------------------|-----------------------------------------------|--------------|
| Give up resource         | Simply don't take one of the locks            | 19           |
| Change acquisition order | Make all threads take locks in the same order | 7            |
| Split resource           | Break one big lock into finer locks           | 1            |
| Other                    | —                                             | 4            |

> **Finding 10.** 61% (19/31) of deadlocks were fixed by giving up a lock. **Implication:** this can trade a deadlock for an atomicity violation. Fixed deadlocks deserve extra checking.

Dropping a lock removes the circular wait, but whatever the lock protected is now unprotected. In some reports developers *said* the fix introduced a new non-deadlock bug and shipped it anyway, betting it would be rare.

### 6.4 Mistakes during fixing

Using Mozilla's patch histories (57 bugs), the authors examined every intermediate patch.

- 17 of 57 bugs had at least one buggy patch; there were 23 buggy patches in total.

- 6 only made the bug **rarer**, without removing it.

- 5 introduced **new concurrency bugs**.

- 12 introduced **new non-concurrency bugs**.

**Worked example — the paper's Figure 9.** For the paper's Figure 5 order bug (blog Figure 15; `js_MarkAtom` must precede `js_UnpinPinnedAtom`):

<figure>
<img src="/assets/img/posts/concurrency-bugs/figure-23.svg" alt="Figure 23" />
</figure>

***Figure 23:*** *The paper's Figure 9(a): the incomplete first fix. The window between steps 1 and 2 is still open, so the mark can run after the unpin. The final fix, the paper's Figure 9(b), is drawn as blog Figure 18 in 6.2.1.*

Attempt 1 still breaks if Thread 2 passes its check *just before* Thread 1 sets `LANDING`, then marks after the unpin. The window shrank but did not close. The final fix adds a second flag, `gcLevel`, so Thread 1 waits for any in-progress mark to finish. This is a classic lesson: a fix that makes a race rarer looks like a fix in testing.

**Takeaway:** developers reasoning about concurrency make mistakes at a high rate even when they know a bug exists. That motivates tools that validate concurrency patches.

## Part 7 — Would transactional memory have prevented these bugs? (Findings 11–13)

Basic TM could cleanly prevent 41 of 105 bugs, could help 44 more if practical problems were solved, and cannot help 20 because they are about order, not atomicity.

### 7.1 TM in one example

**Transactional memory** lets a programmer mark a region `atomic` and leaves the mechanism to the runtime or hardware. The system runs the region optimistically, tracks what it reads and writes, and if another thread interfered, **aborts, rolls back and re-executes** it. To the programmer it looks as if the region ran alone.

The paper's Figure 1 bug (blog Figure 12) becomes:

``` c
atomic {
    if (thd->proc_info)
        fputs(thd->proc_info, ...);
}
```

No lock to choose, no lock ordering to get wrong, no deadlock. Recall the MySQL retry loop from Part 6 (the paper's Figure 8): developers hand-wrote exactly what TM does automatically.

The paper evaluates only **basic** TM — atomicity and isolation — not every advanced design.

### 7.2 Results

| Verdict                      | Bugs (of 105) | Typical bug                                                            |
|------------------------------|---------------|------------------------------------------------------------------------|
| TM can help                  | 41            | Atomicity violations and deadlocks with small, simple critical regions |
| TM might help, with concerns | 44            | Atomic intent, but the region is hard to transactionalize              |
| — rollback problem           | 30            | Region does I/O or system calls                                        |
| — region too long            | 8             | e.g. a whole garbage-collection pass                                   |
| — code nature                | 6             | Code structure is hard to turn into a transaction                      |
| Little help                  | 20            | Order intentions                                                       |

> **Finding 11.** TM can help avoid 41 of 105 bugs (39%). **Implication:** TM is a promising language feature, though not a panacea.

### 7.3 Concerns (Finding 12)

**Rollback.** Look again at the TM version of the paper's Figure 1 (blog Figure 12). `fputs` writes to a file. If the transaction aborts after the write, the runtime cannot un-print the bytes. Irreversible operations — I/O, network sends, many system calls — break the abort-and-retry model. This was the biggest concern (30 bugs).

**Size.** Some Mozilla atomic regions spanned the whole garbage collector. Hardware TM tracks reads and writes in the cache, so a huge footprint overflows it.

> **Finding 12.** TM could help 44 more bugs if these concerns were addressed. **Implication:** combine TM with system support (e.g. deferring I/O until commit) and hybrid hardware/software TM for long transactions.

### 7.4 What TM cannot express (Finding 13)

`atomic` means "run as if alone." It does not mean "run after X." Take the paper's Figure 2 (blog Figure 13) again: wrapping each thread's code in `atomic` still lets Thread 2's transaction commit first.

The paper highlights a nasty order sub-type found in 7 bugs: **"A before B, or A not at all."**

``` c
// Thread 1: keeps inserting           // Thread 2: at some point
cache_insert(e1);                      cache_destroy();
cache_insert(e2);  // must be skipped if the cache is already destroyed
```

The developers did not want Thread 2 to wait for all insertions. They wanted late inserts silently skipped. Neither a lock nor a basic transaction states that intent directly.

> **Finding 13.** 20 of 105 bugs cannot benefit from basic TM, because the violated intention is about order. **Implication:** languages need features for expressing order, not only atomicity.

The paper points to TM designs with richer semantics, such as `retry` and `orElse` from Haskell's composable memory transactions. `retry` lets a transaction block until a condition holds, which can encode "wait until A has happened."

**Connecting the threads of the paper:** order violations show up three times — as a common pattern (Part 4), as bugs that locks cannot fix (Part 6), and as bugs that TM cannot prevent (Part 7). That consistent message is a big reason the paper was so influential.

## Part 8 — Other observations, critique, and legacy

The paper closes with evidence that concurrency bugs are severe and hard to debug, and its findings shaped a decade of follow-up research.

### 8.1 Reading critically

A good senior-level reading asks where the conclusions could break.

| Question                           | Why it matters                                                                                               |
|------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Selection by keywords              | Bugs whose reports never say "race" or "lock" are invisible, which may bias toward easily recognized bugs.   |
| Only fixed, well-documented bugs   | Bugs that are hard to explain or never fixed are excluded; they may be deeper (more threads, more accesses). |
| Manifestation is judged by hand    | "Smallest set *S*" is the authors' reasoning, not a machine-checked minimum.                                 |
| C/C++ desktop and server apps only | Kernels, GPU code, distributed systems, Java and message-passing languages may differ.                       |
| Mozilla dominates                  | 57 of 105 bugs come from one codebase, so its style weighs heavily on the percentages.                       |
| Fixed-size sample                  | 105 bugs is large for hand analysis, but small cells (e.g. 2 "Other") carry little statistical weight.       |

None of these invalidates the study; the authors flag most of them in Section 2.3. They tell you how far to generalize.

### 8.2 Takeaways for you as a programmer

1.  When you write two lines that must happen together, ask: *what if another thread runs between them?*

2.  When you assume "X has happened by now," ask: *what enforces that?* A lock does not.

3.  When several fields must stay consistent, protect them as a group, or make them immutable or thread-local.

4.  When you fix a deadlock by dropping a lock, re-check what that lock was protecting.

5.  A fix that makes a concurrency bug rarer is not a fix. Reproduce first, then verify the fix under forced interleavings.

## Cheat sheet and self-check

All 13 findings on one page, followed by questions to test your understanding.

### All 13 findings

| \#  | Area      | Finding                                                            | Implication                                          |
|-----|-----------|--------------------------------------------------------------------|------------------------------------------------------|
| 1   | Pattern   | 97% (72/74) of non-deadlock bugs are atomicity or order violations | Focus tools on these two patterns                    |
| 2   | Pattern   | 32% (24/74) are order violations                                   | Build order-violation detectors                      |
| 3   | Threads   | 96% (101/105) involve ≤ 2 threads                                  | Test threads pairwise                                |
| 4   | Threads   | 22% (7/31) of deadlocks involve 1 thread                           | Catch self-deadlocks first; they are easy            |
| 5   | Variables | 66% (49/74) of non-deadlock bugs involve 1 variable                | Single-variable detectors are a fair start           |
| 6   | Variables | 34% (25/74) involve \> 1 variable                                  | Build multi-variable detectors                       |
| 7   | Resources | 97% (30/31) of deadlocks involve ≤ 2 resources                     | Test resource acquisition pairwise                   |
| 8   | Accesses  | ~92% of bugs need an order on ≤ 4 accesses                         | Exponential testing becomes polynomial               |
| 9   | Fixes     | Only 20/74 non-deadlock fixes used locks                           | Tools must explain pattern and consistency condition |
| 10  | Fixes     | 19/31 deadlocks fixed by giving up a lock                          | Re-check "fixed" deadlocks for new bugs              |
| 11  | TM        | TM helps 41/105                                                    | Promising feature                                    |
| 12  | TM        | TM could help 44 more, with I/O rollback and size solved           | Add system support, hybrid TM                        |
| 13  | TM        | 20/105 cannot benefit: order intentions                            | Languages need order primitives                      |

### Self-check questions

1.  Explain why a program can be **data-race-free** and still have an atomicity violation. Give a three-line example.

2.  Why is the paper's Figure 2 bug (blog Figure 13) classified as **order** even though a lock-based patch could fix it?

3.  Write down the manifestation condition (set *S*, order, threads, variables, accesses) for the paper's Figure 4 (blog Figure 14) `io_pending` bug.

4.  Bugs seem to appear mainly under heavy load. How does the paper reconcile this with "≤ 2 threads"?

5.  Why can't a per-variable race detector find the paper's Figure 6 bug (blog Figure 16), even if it reports every race perfectly?

6.  Two threads each make 20 shared accesses. Roughly compare the number of full interleavings with the number of access pairs to test.

7.  The MySQL `btr0sea.c` retry loop does not prevent the bad interleaving. Why is it still a correct fix?

8.  Fixing a deadlock by dropping a lock can create which kind of bug? Why?

9.  Why does `fputs` inside an `atomic` block worry TM designers?

10. Pick one threat to validity and describe a concrete way a modern follow-up study could address it.

### Answer hints

1.  Lock each access separately; the check and the use are each protected, but not together (Part 1.5).

2.  Patterns are classified by root cause, the violated intention, not by fix (Part 3.4).

3.  *S* = {S2 write, S4 write} with S4 before S2 (S3's read then spins); 2 threads, 1 variable, 2–3 accesses depending on whether you count S3.

4.  Load raises the probability of the bad order between two threads; it does not add threads to the condition (Part 5.2).

5.  No single variable is accessed wrongly; the problem is the consistency of three variables together (Part 5.3).

6.  "40 choose 20" ≈ 1.4 × 10^11 interleavings versus about 400 pairs × 2 orders.

7.  It detects interference and redoes the work, so the result is always consistent (Part 6.2).

8.  An atomicity violation, because what the lock protected is now unprotected (Part 6.3).

9.  Printing cannot be undone if the transaction aborts (Part 7.3).

10. Open answer; e.g. avoid keyword bias by sampling from crash reports or sanitizer (TSan) reports.
