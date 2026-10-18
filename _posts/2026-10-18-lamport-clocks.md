---
title: "What is Concurrency?"
categories: [Technical]
math: false
mermaid: false
---

In 1978, Leslie Lamport published an eight-page paper, *Time, Clocks, and the Ordering of Events in a Distributed System*. It is one of the most cited papers in computer science and a big part of why he won the Turing Award. Its central idea fits in one sentence: **in a distributed system, "time" really means causality.**

This post walks through the paper from the ground up, with a small worked example at every step.

## The last seat on the plane

Alice and Bob both try to book the last seat on a flight, through two different servers.

- Alice books through **server A** at true time 10:00:00.010.
- Bob books through **server B** at true time 10:00:00.020.

Alice was first. But server A's clock runs 50 ms fast, so its log says **10:00:00.060**. Server B's clock is accurate, so its log says **10:00:00.020**. Sort the two logs by timestamp, and the seat goes to Bob.

Nobody did anything wrong. Every machine's clock drifts a little, and no machine can read another machine's clock without sending a message, which takes an unknown amount of time. **Physical timestamps from different machines cannot be trusted to say which event came first.**

Lamport's answer is to stop asking *when* an event happened, and ask a different question instead: **could this event have influenced that one?** Every machine can answer that using only the messages it actually receives.

## Causality, not time

Lamport models a distributed system as a set of **processes**. Each process does one thing after another, so its events are in a fixed order. Sending a message is an event, and receiving one is an event.

He then defines **happened before**, written **a → b**, as the smallest relation satisfying three rules:

1. **Same process:** if a comes before b in the same process, then a → b.
2. **Messages:** if a sends a message and b receives it, then a → b.
3. **Chains:** if a → b and b → c, then a → c.

In plain terms, **a → b means information could have flowed from a to b.** If neither a → b nor b → a, the two events are **concurrent**. Neither could have influenced the other, so neither came "first" in any sense the system can observe.

Here is the running example used throughout this post: three processes, P, Q and R, exchanging three messages.

![Space-time diagram: processes P, Q and R exchange messages m1, m2 and m3. Each event is labeled with its Lamport timestamp; r2 jumps from 1 to 5.](/assets/img/posts/lamport-clocks/spacetime-diagram.svg)

To check whether a → b, look for a path from a to b that moves forward along process lines and message arrows.

- **p1 → r2:** p1 → p2 → (m1) → q3 → q4 → (m2) → r2. Yes.
- **r1 → p4:** r1 sends m3, p4 receives it. Yes.
- **r1 vs q4:** there is no path either way, so they are **concurrent**. That holds even though r1 is drawn to the left.

A note on the word **"smallest"**: it simply means that → is the **transitive closure** of the three rules. It contains every pair the rules force, and not a single pair more.

## Logical clocks: a counter is enough

A **logical clock** is just an integer counter kept by each process. It has nothing to do with seconds. Its only job is to satisfy the **Clock Condition**:

> If a → b, then C(a) < C(b).

If a could have influenced b, then a gets the smaller number. Two implementation rules, which Lamport calls IR1 and IR2, are enough to guarantee it:

- **IR1:** tick your counter between any two of your own events.
- **IR2:** stamp every message with your counter. When you receive a message with stamp T, move your counter past T.

In code, the whole algorithm is a few lines:

```python
C = 0  # this process's logical clock

def local_event():
    global C
    C += 1
    return C

def send(msg):
    global C
    C += 1
    msg.T = C
    transmit(msg)

def receive(msg):
    global C
    C = max(C, msg.T) + 1
    return C
```

Run it on the example above and you get the numbers in the diagram. The interesting moment is **r2**. R's clock reads 1, but the message m2 arrives stamped 4, so R jumps straight to **5**. A slow process gets pulled forward the moment it learns about later events.

### The catch: it only works in one direction

If a → b, then C(a) < C(b); but this does **not** mean that

> If C(a) < C(b), then a → b.

## What is concurrent?

If two events have **no happened-before relationship** in either direction (neither a → b nor b → a), they are **concurrent**, written a ∥ b.

In the example, r1 and q4 are concurrent: no path of messages connects them. Concurrent does not mean simultaneous, and it does not mean equal timestamps (r1 has 1, q4 has 4). It only means that neither event could have influenced the other.

## One order everyone agrees on

Happened-before is a **partial** order: concurrent events are left unordered. Many algorithms need more than that. They need **one sequence of all events that every process agrees on**.

Lamport writes this total order as **⇒**: a ⇒ b means "a comes before b in the agreed order (i.e., numbering system)", whether or not a actually happened before b.

Lamport's construction takes two steps:

1. **Sort by timestamp.** This puts every causal pair in the right order, because of the Clock Condition.
2. **Break ties by process ID** (P ≺ Q ≺ R). Only concurrent events can share a timestamp, so the tie-break never contradicts causality.

This gives a lexicographic order on (timestamp, process ID). Lamport writes it as **⇒**. For the example:

| (C, process) | Events in ⇒ order |
| --- | --- |
| (1, ·) | p1, q1, r1 |
| (2, ·) | p2, q2 |
| (3, ·) | p3, q3 |
| (4, ·) | p4, q4 |
| (5, ·) | r2 |

Two things are worth seeing here.

- **The order is fixed when the tie-break rule is kept constant.** Other clock values or another tie-break rule would give a different, equally valid order. Only → is a real property of the execution.
- **That is fine.** No process needs the "true" order. What matters is that every process computes the same one, on its own. Each event carries a fixed stamp chosen by its sender, and everyone sorts with the same rule. Same inputs and same rule give the same output, with no coordination.

Compare this with sorting by **arrival time**. One process might receive A then B, while another receives B then A, and the two would disagree.

## Mutual exclusion without a coordinator

Here is the payoff. Several processes share one resource, such as a lock on a file, and we want three guarantees:

- **Safety:** only one process holds the resource at a time.
- **Fairness:** requests are granted in causal order. If request A → request B, then A goes first.
- **Liveness:** every request is eventually granted.

### Why a central coordinator isn't enough

The obvious design sends every request to one coordinator, which serves them in arrival order. That breaks fairness:

1. P1 sends a request to the coordinator.
2. P1 then sends some other message to P2.
3. Having received it, P2 sends its own request to the coordinator.
4. P2's request happens to arrive first.

P1's request → P2's request, but the coordinator serves P2 first. **Arrival order is not causal order.**

> **Key idea:** Rather than having a central coordinator, make the processes communicate with each other through messages.

### Lamport's algorithm

Assume channels are reliable and FIFO. Every process keeps its own request queue, sorted by ⇒, and follows five rules:

1. **Request:** send a timestamped request to everyone and add it to your own queue.
2. **On a request:** add it to your queue and send back a timestamped ack.
3. **Release:** remove your request and send "release" to everyone.
4. **On a release:** remove that process's request from your queue.
5. **Enter** when (i) your request is first in your queue, and (ii) you have received a message timestamped later than your request from every other process.

Each process checks rule 5 on its own. Nobody is in charge.

### The key insight is check (ii)

Check (i) alone is not safe. Your request may look first only because an earlier request from someone else is still in transit.

Check (ii) closes that gap. Suppose P1 requested at time 2 and has received an ack from P2 stamped 3. Anything P2 sent earlier, including any request stamped 2 or lower, was sent **before** that ack. Because channels are FIFO, it also **arrived** before the ack. So once P1 holds a later message from everyone, it knows its queue is complete up to its own request.

### Watching it run

<iframe id="lamport-mutex" src="/assets/html/lamport-clocks/lamport-mutex.html"
        title="Lamport mutual exclusion, step by step"
        width="100%" height="1150" style="border:0" loading="lazy"></iframe>
<script>
  /* AIDEV-NOTE: same-origin iframe; fit height to content, mirror Chirpy's theme. No // comments: HTML is minified. */
  (function () {
    const frame = document.getElementById("lamport-mutex");
    const root = document.documentElement;
    frame.addEventListener("load", function () {
      const doc = frame.contentDocument;
      if (!doc) return;
      const syncTheme = function () {
        const mode = root.getAttribute("data-mode");
        if (mode) doc.documentElement.setAttribute("data-theme", mode);
        else doc.documentElement.removeAttribute("data-theme");
      };
      const fit = function () { frame.style.height = doc.body.scrollHeight + "px"; };
      syncTheme();
      new MutationObserver(syncTheme).observe(root, { attributes: true, attributeFilter: ["data-mode"] });
      new ResizeObserver(fit).observe(doc.body);
      fit();
    });
  })();
</script>

The cost is **3(N − 1) messages** per entry: N − 1 requests, N − 1 acks and N − 1 releases.

## The bigger idea: replicated state machines

Mutual exclusion turns out to be one instance of a general recipe, which may be the most influential idea in the paper:

> **If every replica applies the same deterministic commands in the same order, every replica ends up in the same state.**

The total order ⇒ supplies "the same order". A replica runs a command stamped T only after it has heard from every other process with a stamp of at least T, so nothing earlier can still arrive.

**Example: a replicated bank account.** Three replicas each hold a balance of 100. Two commands are issued concurrently:

- **(3, A):** deposit 50
- **(3, B):** withdraw 120 if the balance is at least 120

Without an agreed order, replicas can diverge. Deposit-then-withdraw leaves 30. Withdraw-then-deposit rejects the withdrawal and leaves 150. With ⇒ and A ≺ B, every replica runs the deposit first, and every replica ends at **30**.

This recipe underlies Paxos, Raft and almost every replicated database built since.

### The limitation Lamport points out himself

The recipe needs **every** process to take part, so one crashed process stalls everyone. Lamport makes a subtle observation here: with logical time alone, a crashed process and a very slow one look exactly the same. You only decide "it crashed" because you waited "too long", and "too long" is a physical-time idea.

## Where logical clocks fall short

Logical clocks only see causality that travels through **system messages**. Lamport's example shows what that misses:

1. You submit request A on a computer in New York.
2. You phone a friend in San Francisco and say "go ahead".
3. Your friend submits request B.

In reality, A caused B. But no system message connects them, so inside the system A and B are concurrent. If the San Francisco machine's counter happens to be lower (say 12 against 900), B gets the smaller timestamp and is served first. The phone call was invisible to the system.

There are two ways out:

- **Bring the causality into the system.** Tell your friend A's timestamp, and have B stamped later than it. This works, but it puts the burden on users.
- **Use real clocks that are synchronized well enough.** Any outside causal chain takes some minimum time μ, bounded at the very least by the speed of light. If every pair of clocks agrees to within ε ≤ μ (ignoring a tiny drift term), the effect is always stamped after its cause.

Lamport then shows how to keep clocks that close with a simple rule: **on receiving a message, never set your clock backward, only forward**, to at least the sender's stamp plus the minimum delay. He proves that clocks stay within about

> **ε ≈ d (2κτ + ξ)**

where d is the network diameter, κ the clock drift rate, τ the interval between messages, and ξ the unpredictable part of message delay.

Plug in typical numbers (d = 3 hops, τ = 10 s, κ = 10⁻⁶, ξ = 1 ms) and you get ε ≈ **3 ms**. Almost all of that comes from ξ. **Network jitter, not clock quality, limits synchronization.** That lesson still holds for NTP and PTP today.

## Why it still matters

Nearly every idea in these eight pages grew into its own field.

| Idea in the paper | What it became |
| --- | --- |
| Happened-before (→) | The standard model of causality, and the basis of happens-before race detectors such as ThreadSanitizer |
| Timestamps can't detect concurrency | Vector clocks (Fidge and Mattern, 1988): C(a) < C(b) exactly when a → b |
| Same commands, same order | State machine replication: Paxos, Raft, replicated logs |
| Failure needs physical time | Timeouts, failure detectors, and the FLP impossibility result (1985) |
| Physical clocks prevent anomalies | Google Spanner's TrueTime, which waits out clock uncertainty before committing |
| Mixing logical and physical time | Hybrid Logical Clocks (2014) |

The lasting lesson is the change of question. Distributed systems rarely need to know **what time** something happened. They need to know **what could have caused what**, and to agree on **one order** everyone can compute alone. Lamport showed that a counter and two rules give you both.

## Further reading

- Lamport, *Time, Clocks, and the Ordering of Events in a Distributed System*, Communications of the ACM, 1978
- Lamport, *The Part-Time Parliament*, 1998: Paxos, the fault-tolerant follow-up
- Fidge (1988) and Mattern (1989): vector clocks
- Corbett et al., *Spanner: Google's Globally-Distributed Database*, OSDI 2012
- Kulkarni et al., *Logical Physical Clocks*, 2014
