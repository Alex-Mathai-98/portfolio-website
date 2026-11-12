### 7. AFix: automated atomicity-violation fixing

**Paper:** Guoliang Jin, Linhai Song, Wei Zhang, Shan Lu and Ben Liblit, *Automated Atomicity-Violation Fixing*, PLDI 2011. [Read the full post]({% post_url 2026-11-10-afix-atomicity-violation-fixing %}).

> Given a bug report (p, c, r), hold one lock on every statement on some path from p to c, and take the same lock around r. Then r can never run between p and c.

#### The running example: Apache's buf\_write

Two threads append to a shared log buffer by running the same function, buf\_write. A detector reports two atomicity violations on it, A and B. For each one, Figure 1 marks p and c (Thread 1's two accesses) and r (Thread 2's access that slips between them).

![Figure 1: The Apache buf_write bug: two atomicity violations reported on four lines](/assets/img/posts/afix-atomicity-violation-fixing/01-apache-buf-write.png)

***Figure 1:*** *The Apache buf\_write bug: two atomicity violations reported on four lines.*

In **A**, Thread 2 writes `buf_len` between Thread 1's size check (line 2) and its copy (line 6), so the copy uses a larger `buf_len` than the check approved and can overflow.

In **B**, Thread 2 writes `buf_len` between Thread 1's copy (line 6) and its write-back (line 8), so Thread 1 overwrites Thread 2's update with a stale length and the log is corrupted. Here p is line 6, and line 8 is both c (in Thread 1) and r (in Thread 2).

Now try to fix it with locks the obvious way. Each attempt below is from the paper's introduction.

***Table 1:*** *Three obvious patches for Figure 1 and why each deadlocks.*

| Patch attempt | What goes wrong | Fixed in |
| --- | --- | --- |
| For A: lock before line 2, unlock after line 6 | When the check fails, the function returns at line 4 still holding the lock; the next call to `buf_write` deadlocks | Fixing one bug in one function |
| For B: one critical region over lines 6–8, another over line 8 (the r access), same lock | Line 8 is inside the 6–8 region, so the thread re-acquires a lock it already holds | Pairing the p–c region with r |
| Both fixes, each with its own lock L1 and L2 | One thread holds L1 (taken at line 2) and waits for L2 before line 6; another holds L2 (taken at line 6) and waits for L1 before line 8 | Fixing many reports together |

All three are the same mistake: placing lock and unlock by looking at the reported lines instead of at every control-flow path and every other patch.

**Obvious?**: AFix's final patch for this function is one lock, taken before line 2 and released before line 4 and after line 8 (see "Apache, end to end" below).

#### Background: what CTrigger hands to AFix

**AFix is a fixer, not a finder: its input is a list of (p, c, r) triples, each a confirmed failure where a remote access r lands between two consecutive accesses p and c by one thread to the same shared variable.**

##### CTrigger in two phases

CTrigger (Park, Lu and Zhou, ASPLOS'09) targets single-variable atomicity violations. It is active testing in the RaceFuzzer mould: predict first, then force.

- **Detection.** Monitor a few runs of the program, predict which unserializable interleavings could happen under the same input, and output the instructions p (preceding), c (current) and r (remote) for each.
- **Testing.** Inject delays at selected points to force each predicted interleaving. Report the triples whose interleaving actually makes the program fail.

A triple is reported only when its interleaving gives a result no serial order of the two threads could give. On one variable x, that leaves four patterns.

![Figure 2: The four interleaving patterns CTrigger reports, with p and c in Thread 1 and r in Thread 2](/assets/img/posts/afix-atomicity-violation-fixing/02-ctrigger-patterns.png)

***Figure 2:*** *The four interleaving patterns CTrigger reports, with p and c in Thread 1 and r in Thread 2. Which pattern a reported triple shows depends on the underlying code: whether p, r and c read or write x.*

Because p and c are **consecutive** accesses to x, no other instance of p or c lies between them. AFix's region algorithm (below) relies on this.

##### The naive fix

Given one triple, the naive patch acquires a lock before p and before r and releases it after c and after r. The authors implemented it as a baseline. It fixes **none** of the eight bugs and deadlocks in **seven**. It fails in four distinct ways, each of which a later section of AFix is designed to avoid. Figures 3a–3d walk through each one step by step, showing which thread does what and how many threads it takes.

![Figure 3a: Naive fix, case 1: an early return leaks the lock](/assets/img/posts/afix-atomicity-violation-fixing/03a-naive-early-return.png)

***Figure 3a:*** *Naive fix, case 1: an early return leaks the lock. T1 returns at line 4 still holding L, so the next lock(L), by T2 or by T1 itself, never succeeds. One thread is enough. Amber lines are the locks the naive rule adds.*

![Figure 3b: Naive fix, case 2: p and c in different functions](/assets/img/posts/afix-atomicity-violation-fixing/03b-naive-different-functions.png)

***Figure 3b:*** *Naive fix, case 2: p and c in different functions. The lock is taken in close() and released in open(), so when close() runs twice with no open() in between, T1 blocks on its own lock. One thread is enough.*

![Figure 3c: Naive fix, case 3: r inside the p–c region](/assets/img/posts/afix-atomicity-violation-fixing/03c-naive-r-inside-region.png)

***Figure 3c:*** *Naive fix, case 3: r inside the p–c region. p, c and r are all in buf\_write(), so both threads run the same code, and the lock added for r also runs inside T1's own region. Line 8 is both c and r, so T1 takes L before line 6 and again before line 8, and deadlocks with itself. One thread is enough.*

![Figure 3d: Naive fix, case 4: two patches, two locks](/assets/img/posts/afix-atomicity-violation-fixing/03d-naive-two-patches.png)

***Figure 3d:*** *Naive fix, case 4: two patches, two locks. T2 ends up holding L2 and needing L1, while T1 holds L1 and needs L2: a circular wait that takes two threads. Amber lines are patch A's lock L1, grey lines patch B's lock L2.*

#### AFix at a glance

**AFix copies a developer's fixing workflow in four steps: understand the bug, patch each report, merge the patches, then test and monitor them.** Steps 2 and 3 are static and took at most one second per program in the evaluation.

![Figure 4: The AFix pipeline](/assets/img/posts/afix-atomicity-violation-fixing/04-afix-pipeline.png)

***Figure 4:*** *The AFix pipeline. The two highlighted steps are the paper's main contribution.*

Fixing one triple (p, c, r) means making the code from p to c mutually exclusive with r. The paper breaks that into four sub-tasks:

1. **Put p and c in one critical region on every path**, without a double lock, a double unlock, an unlock without a lock or a deadlock (see the next two subsections).
2. **Put r in its own critical region.** This is easy: lock before r, unlock after it.
3. **Make the two regions cooperate**, which takes more than giving them the same lock (see "Do the two regions overlap?" below).
4. **Choose the lock** (see "Choosing the lock" below).

Implementation notes: AFix is built on LLVM 2.7 and rewrites the program's bitcode with POSIX `pthread` locks. CTrigger reports instruction addresses, so AFix maps each one back to a source line and never places a lock or unlock in the middle of a line, even if that widens a region slightly. This keeps patches readable.

#### Fixing one bug in one function

**AFix holds the lock at exactly the CFG nodes that lie on some path from p to c: it computes that set as P ∩ C, puts a lock on every edge into it and an unlock on every edge out of it, in time linear in the CFG.** This section assumes p and c are in the same function and the function is not recursive; The next subsection removes both assumptions.

##### Computing the protected nodes

The paper's recipe:

1. **P:** search forward from p, treating c as if it had no successors.
2. **C:** search backward from c, treating p as if it had no predecessors.
3. **Protected nodes = P ∩ C:** reachable from p, and able to reach c, without passing through p or c again on the way.

Then AFix inserts `lock` on every edge from an unprotected node to a protected one, and `unlock` on every edge from a protected node to an unprotected one. To put code "on" an edge x → y, it splits the edge with a new node n (x → n → y) and puts the code in n.

Two searches and an intersection are each linear in nodes plus edges, so the whole step is linear in the size of the function's CFG.

Figure 5 runs the four steps on Apache report A from Figure 1, with p = line 2 and c = line 6.

![Figure 5: Computing the protected nodes for Apache report A: the forward search P from p, the backward search C from c, their intersection, and the lock and unlock edges where control enters or leaves that set](/assets/img/posts/afix-atomicity-violation-fixing/05-protected-set-apache-a.png)

***Figure 5:*** *Computing the protected nodes for Apache report A: the forward search P from p, the backward search C from c, their intersection, and the lock and unlock edges where control enters or leaves that set.*

##### The three traps, replayed

![Figure 6: Naive lock placement vs. AFix's edge-based placement on three traps](/assets/img/posts/afix-atomicity-violation-fixing/06-three-traps.png)

***Figure 6:*** *Naive lock placement vs. AFix's edge-based placement on three traps.*

To check the rule, I replayed it in a short script on the paper's examples. Table 2 lists the result; the Apache rows are the two reports from Figure 1.

***Table 2:*** *Protected nodes and lock edges computed for four examples (replayed in a script).*

| Example | P ∩ C | lock on edge | unlock on edges |
| --- | --- | --- | --- |
| (a) branch | p, c | entry → p | p → else, c → join |
| (b) loop | p, loop head, c | pre-loop → loop head | c → after |
| Apache report A (2, 6, 8) | lines 2, 3, 6 | entry → 2 | 3 → 4 (the return), 6 → 8 |
| Apache report B (6, 8, 8) | lines 6, 8 | 3 → 6 | 8 → exit |

Figure 7 draws each row of Table 2 as a control-flow graph.

![Figure 7: The four examples of Table 2 as control-flow graphs: the protected nodes P ∩ C, the edge that gets lock() and the edges that get unlock()](/assets/img/posts/afix-atomicity-violation-fixing/07-four-cfgs.png)

***Figure 7:*** *The four examples of Table 2 as control-flow graphs: the protected nodes P ∩ C, the edge that gets lock() and the edges that get unlock(). The lock placement replays the rule above.*

Two things to notice:

- In (b) the loop head is protected because it lies on the path p → head → c. That is why the lock moves out of the loop. It also means the lock is held while the loop condition is checked, even on a run that never executes p. The region is slightly larger than strictly needed, but correct.
- In Apache report A, the return path gets its own unlock on edge 3 → 4. This is exactly what the first naive patch in Table 1 missed.

##### What AFix guarantees

Because the lock is taken only when control enters the protected set and released only when it leaves, two facts hold: the lock is held at every protected node, and never at an unprotected one. The paper's guarantees follow from those two facts.

***Table 3:*** *Correctness questions the paper answers for one patched function.*

| Question | Answer | Why |
| --- | --- | --- |
| Are p and c inside the region on every path? | Yes | Every node on any p → c path is in P ∩ C |
| Can the patch add a double lock? | No | A lock is placed only on edges leaving unprotected nodes, where the lock is not held |
| Can it add a double unlock or an unlock without a lock? | No | An unlock is placed only on edges leaving protected nodes, where the lock is held |
| Can it add a data race? | No | Locks only remove interleavings: every schedule of the patched program was possible before |
| Can it add a deadlock? | Not provable in general for C/C++ | Handled by a static check plus timed locks (see timed locks below) |

The first three answers depend on nobody else taking the same lock inside the region. Recursion breaks that (see reentrant locks below), and so does an r region nested inside the p–c region.

##### Why this policy

The authors implemented other placement policies, some of which give smaller regions for certain control-flow shapes. They kept this one because it is simple to reason about, composes well across functions and gives readable patches, and they saw no excessive slowdown from it. It also ends the region as soon as c can no longer be reached, as on the else edge in (a).

#### Deadlocks, recursion and p, c in different functions

**The basic rule can't prove deadlock freedom, breaks under recursion and assumes p and c share a function; AFix handles these with timed locks, reentrant locks and by moving the region up to the innermost function both call stacks share.**

##### Possible deadlocks: timed locks

AFix scans each new critical region for operations that can block. Here that means the p–c region from the basic rule: p, c and every statement on any path between them (the protected set P ∩ C). It looks for:

- lock acquisitions, condition waits, barriers and thread joins, which are easy to find;
- ad-hoc spin loops: AFix conservatively treats **any loop inside the region that touches heap or global variables** as a possible spin loop.

If the region has none, a deadlock is impossible and AFix uses an ordinary `pthread_mutex_lock`. If it has one, AFix uses `pthread_mutex_timedlock`, with a default time-out of **10 seconds**. A circular wait then ends in a time-out instead of a hang, and the run-time system works out whether it was a real deadlock.

> **Major drawback.** The cost is that **a timed-out lock gives up atomicity**: the thread goes on without the lock, so the bug can still happen on that run. The paper names this as one of the two cases where AFix can fail.

Figures 8a–8c walk through the problem step by step on a small example: the original bug, AFix's fix drawn on the control-flow graphs, and how that fix can deadlock.

![Figure 8a: The original code and the atomicity violation (p, c, r): T2 sets shared to NULL between T1's check p and its use c, and T1 crashes](/assets/img/posts/afix-atomicity-violation-fixing/08a-blocking-call-bug.png)

***Figure 8a:*** *The original code and the atomicity violation (p, c, r): T2 sets shared to NULL between T1's check p and its use c, and T1 crashes. The existing lock M does not prevent it because T1 does not hold M at p or c. Illustrative example, not from the paper.*

![Figure 8b: AFix's fix on the control-flow graphs](/assets/img/posts/afix-atomicity-violation-fixing/08b-blocking-call-cfg.png)

***Figure 8b:*** *AFix's fix on the control-flow graphs. In T1, lock(L) goes on the edge into the protected set and unlock(L) on both edges out of it (the end of the if and the false branch). In T2, r gets its own one-node region. The existing lock(M) lies inside T1's protected set, which is what makes the fix risky.*

![Figure 8c: With the fix applied: T1 holds L and blocks on M inside the region, while T2 holds M and waits for L before r](/assets/img/posts/afix-atomicity-violation-fixing/08c-blocking-call-timed-lock.png)

***Figure 8c:*** *With the fix applied: T1 holds L and blocks on M inside the region, while T2 holds M and waits for L before r. A plain lock hangs forever; AFix's timed lock ends the wait after 10 s but loses atomicity on that run. Illustrative example, not from the paper.*

##### Recursion: reentrant locks

If a function containing an AFix region can call itself, directly or through other calls, while inside that region, the thread will try to take a lock it already holds (Figure 6c). AFix finds these cases with reachability on the static call graph. Figures 9a–9c walk through it step by step: the original bug, AFix's fix drawn on the control-flow graph, and what the recursive call does to that fix.

![Figure 9a: The original code and the bug: T2 sets aPtr to NULL between T1's read p and its use c inside the recursive function foo()](/assets/img/posts/afix-atomicity-violation-fixing/09a-recursion-bug.png)

***Figure 9a:*** *The original code and the bug: T2 sets aPtr to NULL between T1's read p and its use c inside the recursive function foo(). Illustrative example.*

![Figure 9b: AFix's fix on the control-flow graph: lock(L) on the edge into foo()'s protected set and unlock(L) on the edge out, plus a one-node region around r in T2](/assets/img/posts/afix-atomicity-violation-fixing/09b-recursion-cfg.png)

***Figure 9b:*** *AFix's fix on the control-flow graph: lock(L) on the edge into foo()'s protected set and unlock(L) on the edge out, plus a one-node region around r in T2. The protected set contains the recursive call, which re-enters foo() and reaches lock(L) while L is still held.*

![Figure 9c: What happens at the recursive call](/assets/img/posts/afix-atomicity-violation-fixing/09c-reentrant-lock.png)

***Figure 9c:*** *What happens at the recursive call. With a plain mutex, T1 blocks on the lock it already holds. With AFix's reentrant lock, T1 re-enters by raising the count, and L opens for T2 only when the count returns to 0.*

When an AFix region can be reached from another AFix region that uses the same lock, AFix uses a **reentrant lock**: a mutex plus an owner thread ID and a nesting count. Everywhere else it uses plain mutexes, which are faster.

***Table 4:*** *Which lock AFix inserts, and when.*

| Situation in the region | Lock AFix uses |
| --- | --- |
| No blocking operations, no recursion | `pthread_mutex_lock` |
| A possibly blocking operation (lock, wait, barrier, join, suspected spin loop) | `pthread_mutex_timedlock`, 10 s default |
| Region reachable from another region with the same lock | Reentrant lock (owner ID + count) |

##### p and c in different functions

**In short:** when p and c sit in different functions, AFix moves the critical region up into the innermost function that calls both, and protects those two calls.

**The problem.** The paper's MySQL example rotates the log: `newlog()` calls `close()` (p) and then `open()` (c). Another thread's `insert()` checks whether the log is open in between, sees it closed, and silently drops its log entry.

![Figure 10a: The bug: T2's insert() reads the log state after close() and before open(), and drops its entry](/assets/img/posts/afix-atomicity-violation-fixing/10a-mysql-bug.png)

***Figure 10a:*** *The bug: T2's insert() reads the log state after close() and before open(), and drops its entry. p and c are writes in two different functions.*

**Why the basic algorithm doesn't work here.** The basic rule finds the region inside one function's control-flow graph, and no single CFG here contains both p and c. The two obvious workarounds both put lock and unlock in different functions. Splitting the lock between `close()` and `open()` is the naive patch of Figure 3b, which deadlocks. A full interprocedural analysis could compute a region that spans functions, but it would have the same split, which composes badly; the authors note that developers' own patches almost always keep lock and unlock in one function.

![Figure 10b: Why the basic rule can't be applied directly: close()'s CFG has no c and open()'s CFG has no p, so P ∩ C is empty in both](/assets/img/posts/afix-atomicity-violation-fixing/10b-mysql-why-section5-fails.png)

***Figure 10b:*** *Why the basic rule can't be applied directly: close()'s CFG has no c and open()'s CFG has no p, so P ∩ C is empty in both. The two workarounds AFix rejects are listed on the right.*

**AFix's solution: move up to a common caller.** AFix does what a developer would. Figure 10c runs these steps on the MySQL example, and Figure 10d shows the resulting code.

1. Compare the call stacks CTrigger reported for p and for c.
2. Take **f**, the innermost function on their common prefix.
3. If p is not directly in f, replace p with the call node in f that leads to p. Do the same for c.
4. Run the basic rule inside f.

![Figure 10c: AFix's solution on MySQL: the call stacks of p and c agree up to newlog(), so p and c are replaced by the calls close() and open(), and the basic rule runs on newlog()'s CFG](/assets/img/posts/afix-atomicity-violation-fixing/10c-mysql-move-up-to-newlog.png)

***Figure 10c:*** *AFix's solution on MySQL: the call stacks of p and c agree up to newlog(), so p and c are replaced by the calls close() and open(), and the basic rule runs on newlog()'s CFG. The lock goes before close() and the unlock after open().*

![Figure 10d: The resulting patch: lock(L) before close() and unlock(L) after open() inside newlog(), plus r's own region in insert()](/assets/img/posts/afix-atomicity-violation-fixing/10d-mysql-resulting-patch.png)

***Figure 10d:*** *The resulting patch: lock(L) before close() and unlock(L) after open() inside newlog(), plus r's own region in insert().*

**The cost.** The new p and c are whole call sites, so the region now covers everything `close()` and `open()` do. Any blocking operation inside those calls would also be inside the region; the paper doesn't say how deep the timed-lock scan follows calls.

#### Pairing the p–c region with r, and choosing a lock

**If reaching r already implies being inside the p–c region, AFix drops r's own lock operations; otherwise it keeps them, and it guards both regions with one new global lock.** This removes the second deadlock in Table 1.

##### Do the two regions overlap?

Let f be the function holding the p–c region (after the substitution above). AFix checks r's reported call stack:

- **f is not on r's stack:** the regions can't overlap, so r keeps its lock and unlock.
- **r is directly in f:** they overlap if r is one of the protected nodes.
- **f is on r's stack, but r is deeper:** find the call node in f that leads to r; they overlap if that call node is protected.
- **Recursion puts f on the stack several times:** if any of f's calls leading to r is inside the p–c region, r's lock operations are redundant.

When they overlap, r's lock and unlock are removed; otherwise taking the lock at r would be a double lock. In Apache report B (6, 8, 8), r is line 8, which is in the protected set {6, 8}, so AFix drops the separate region around line 8.

This is checked only for **the stack configuration CTrigger reported**. If another path reaches r outside the p–c region and also fails, AFix relies on CTrigger to report that path separately.

##### Choosing the lock

For a single report, AFix creates **a new global lock** and uses it at the boundaries of both regions.

Reusing an existing lock would be nicer, especially if p and c are already inside a critical section. But the lock would have to be live and reachable at r and on every edge into or out of the p–c region. For heap locks reached through linked data structures that is hard, and harder still when r is in another function with different locals.

Global locks have fixed names, so AFix does try one kind of reuse: if a global lock already protects the intended p–c region in the same function, AFix uses it and adds no lock operations around p–c. None of the eight evaluation bugs had such a lock.

##### When AFix can still fail

The paper names two cases:

1. **A timed lock times out** (see timed locks above). Atomicity is lost on that run; the run-time system reports it so the patch can be refined.
2. **p or c has several dynamic instances that touch different memory.** The p–c pair that needs protection may not be the consecutive pair AFix protects. The authors call this theoretically possible but very rare, and illustrate it with a made-up example.

**Case 2, step by step.** The paper's made-up example has a loop in which c runs more than once. Figure 11a shows the bug, Figure 11b where the basic rule puts the lock, Figure 11c the patch as code, and Figure 11d why the patch still lets the bug through.

**The problem.** A detector reports static lines, not runs. Here c is one line that runs twice, and only its second run writes the same location as p.

![Figure 11a: The code and the bug: c runs twice and only the second run writes p's location, array(1)](/assets/img/posts/afix-atomicity-violation-fixing/11a-rare-case-bug.png)

***Figure 11a:*** *The code and the bug: c runs twice and only the second run writes p's location, array\[1\]. To fix the bug, r must be kept out of the window from p to that second run. Code from a made-up example in the paper; the paper gives no r, so the r here is illustrative.*

**What AFix does.** The basic rule stops the forward search at the first c it reaches, so the region ends right after that first run.

![Figure 11b: AFix's region on the control-flow graph: the protected set is {p, loop test, body before c, c}](/assets/img/posts/afix-atomicity-violation-fixing/11b-rare-case-cfg.png)

***Figure 11b:*** *AFix's region on the control-flow graph: the protected set is {p, loop test, body before c, c}. Because the rest of the loop body is outside the set, unlock(L) goes right after c and lock(L) goes on the back edge.*

**The patch as code.** Figure 11c shows the same lock and unlock edges written into the source.

![Figure 11c: The same patch as code: the lock is released after every run of c and taken again before the next one](/assets/img/posts/afix-atomicity-violation-fixing/11c-rare-case-patch.png)

***Figure 11c:*** *The same patch as code: the lock is released after every run of c and taken again before the next one.*

**What goes wrong.** Between the two runs of c, L is free, so r can take it and run in exactly the window the fix was meant to close.

![Figure 11d: Running the patch: L is free between the first and second runs of c, so r can still land between p and its real partner](/assets/img/posts/afix-atomicity-violation-fixing/11d-rare-case-run.png)

***Figure 11d:*** *Running the patch: L is free between the first and second runs of c, so r can still land between p and its real partner.*

This case is a direct consequence of the basic rule: a path that passes through c once already counts as having reached c, so the region ends there.

#### Fixing many reports together

**AFix first patches each report on its own, then drops patches that another patch subsumes and merges patches whose lock-guarded regions contain each other's p, c or r, repeating until nothing more merges.** For Apache this turns two reports, two locks and a cross-patch deadlock into one lock, one lock call and two unlock calls.

##### Removing redundant patches

Each patch has two regions: p–c and r. A patch is fully redundant if another patch subsumes both. If only its p–c region is subsumed, AFix still drops that region, and switches r's region to the subsuming patch's lock; r is a single instruction, so a separate region for it costs little.

Under AFix's placement rule, region p–c is subsumed by region p′–c′ exactly when its CFG nodes are a subset of p′–c′'s: the outer lock is already held at every inner node.

Across functions, let f hold p–c, let f′ hold p′–c′, and let n be the call node in f′ that eventually leads to f. Then p′–c′ subsumes p–c if and only if:

1. f′ is on the common prefix of p's and c's call stacks, and
2. n is inside the p′–c′ region.

The full call stacks CTrigger reports, and the rule that each region's lock and unlock live in one function, are what keep this check simple.

##### When to merge

For each patch i, AFix keeps two node sets:

- **Anchorsᵢ**: the nodes of the bug it was made for, initially {pᵢ, cᵢ, rᵢ}.
- **Criticalᵢ**: every node inside a region guarded by patch i's lock: the pᵢ–cᵢ protected nodes plus rᵢ.

$$
\text{merge}(i, j) \iff \mathit{Anchors}_i \cap \mathit{Critical}_j \neq \emptyset \;\lor\; \mathit{Anchors}_j \cap \mathit{Critical}_i \neq \emptyset
$$

In words: merge two patches when one patch's lock already guards an access of the other's bug. The merged patch gets the union of both Anchors sets and both Critical sets and is itself a candidate for further merging, until no patch's p, c or r lies inside another patch's guarded region.

##### How to merge

1. **Recompute the boundaries.** Put an unlock on every edge leaving the merged Critical set and a lock on every edge entering it. Lock operations that now sit strictly inside the set disappear.
2. **Use one lock variable.** Pick either patch's lock arbitrarily and use it in every lock and unlock of the merged patch.

This merge runs only on regions in the same function, and it also covers intraprocedural redundancy. AFix removes interprocedural redundant patches first (8.1), then merges.

![Figure 12: Three scenarios where patches should or must merge, before and after AFix's merge](/assets/img/posts/afix-atomicity-violation-fixing/12-merge-scenarios.png)

***Figure 12:*** *Three scenarios where patches should or must merge, before and after AFix's merge.*

##### Apache, end to end

Putting it all together on Figure 1. My script reproduces each step:

1. Report A (2, 6, 8): protected {2, 3, 6}. r = line 8 is outside, so it gets its own L1 region.
2. Report B (6, 8, 8): protected {6, 8}. r = line 8 is inside, so its separate region is dropped.
3. Anchors\_B = {6, 8} lies inside Critical\_A = {2, 3, 6, 8}, so the patches merge.
4. Merged Critical = {2, 3, 6, 8}: lock on entry → 2, unlock on 3 → 4 and on 8 → exit.

![Figure 13: The Apache bug's two patches, each correct alone, and the merged patch AFix applies](/assets/img/posts/afix-atomicity-violation-fixing/13-apache-merged.png)

***Figure 13:*** *The Apache bug's two patches, each correct alone, and the merged patch AFix applies.*

The result matches the paper exactly: one lock before line 2, unlocks before line 4 and after line 8. The paper notes the merged patch closely resembles the developers' own fix.

**The whole technique on one page.** Figure 14 runs AFix end to end on the Apache bug: the input, the patch for each report with its control-flow graph and how its lock edges were chosen, the merge test, and the merged patch.

![Figure 14: AFix end to end on Apache](/assets/img/posts/afix-atomicity-violation-fixing/14-afix-end-to-end.png)

***Figure 14:*** *AFix end to end on Apache. Steps 1a and 2a compute each report's p–c region (forward search, backward search, intersection) and place lock and unlock on the edges that cross its boundary. Steps 1b and 2b handle r separately: A's r (line 8) lies outside its p–c region and gets its own region on L1, while B's r is already inside, so no region is added. Step 3 applies the merge test and shows the deadlock the two separate patches would cause. Step 4 takes the union of the guarded nodes and recomputes the boundary, which leaves one lock L, one lock call and two unlock calls.*

##### What merging buys

***Table 5:*** *Effects of merging on patch quality.*

| Property | Effect | Caveat |
| --- | --- | --- |
| Readability | Far fewer locks; one lock in five of the six multi-report bugs | Judged by manual inspection |
| Performance | Usually better: fewer lock operations without much larger regions | Not guaranteed; larger regions can cut concurrency |
| Correctness | Same or better, because regions only grow | Helps most when the detector misreports the root cause (see the results below) |
| Deadlock between AFix patches | Impossible, by the paper's argument: two AFix locks that one thread could hold at once would have been merged | The argument is about regions inside one function |

#### Results: overall patch quality

***Table 6:*** *Overall patch quality. Fixed = bug gone, no new bug, negligible slowdown; partial = failures reduced without hurting performance; bad = new bug, little effect or intolerable slowdown. For manual patches, partial means developers first submitted incomplete patches.*

| Bug | Naive | Unmerged | Merged | Manual |
| --- | --- | --- | --- | --- |
| FFT | bad | bad | partial | fixed |
| PBZIP2 | bad | bad | bad | fixed |
| Apache | bad | bad | fixed | fixed |
| MySQL1 | bad | fixed | fixed | fixed |
| MySQL2 | bad | fixed | fixed | partial |
| Mozilla1 | bad | fixed | fixed | partial |
| Mozilla2 | bad | fixed | fixed | fixed |
| Cherokee | bad | fixed | fixed | fixed |

Merged AFix fixes six of eight; unmerged fixes five, losing Apache to the cross-patch deadlock of Figure 13. For MySQL2 and Mozilla1 the merged patch beats the developers' first attempts. The merged PBZIP2 patch lowers failures but is rated bad because of its slowdown (10.4).
