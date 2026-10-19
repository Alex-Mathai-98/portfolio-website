---
title: "Can AI Write a Formal Model of a Real System? Notes on SysMoBench"
categories: [Technical]
math: false
mermaid: false
---

*A walkthrough of Cheng et al., "SysMoBench: Evaluating AI on Formally Modeling Complex Real-World Systems" (ICLR 2026).*

LLMs can now fix GitHub issues, write tests, and translate C to Rust. A harder question is whether they can *understand a system well enough to describe it formally*. SysMoBench is a benchmark built to answer exactly that. Its short answer: **yes for a 200-line spinlock, mostly no for a 2,000-line Raft implementation.**

The paper is a benchmark paper, so there is no new algorithm. What makes it worth reading is the evaluation design: how do you automatically grade a formal model when there is no single right answer?

## Why formal models of systems matter

Concurrent and distributed code fails in interleavings that tests rarely hit. Take two threads that each run `counter = counter + 1`. Each increment is really three steps (read, add, write), so the two threads can interleave in 20 different orders, and some of those orders lose an update. Five Raft servers with message drops and crashes have astronomically more interleavings than that.

A **model checker** such as TLC does not sample those interleavings. It enumerates every reachable state of a small instance and checks a property in each one. If anything breaks, you get the exact sequence of steps that leads there.

The catch is the model itself. A *protocol* model describes the Raft algorithm as published by Ongaro and Ousterhout. A *system* model describes **Etcd's** Raft, including its extra PreCandidate state, its message types and its heartbeat logic. System models find real code bugs, but they take months to write and constantly drift as code changes. If an LLM could draft them, formal verification would get much cheaper.

## A 60-second TLA+ primer

The "hello world" of TLA+ is Lamport's hour clock: a clock that only shows the hour and ticks forward forever.

In ordinary code, such a clock might look like this:

```python
import random, time

class HourClock:
    def __init__(self):
        self.hr = random.randint(1, 12)   # could be set to any hour at startup

    def tick(self):
        if self.hr == 12:
            self.hr = 1
        else:
            self.hr = self.hr + 1

clock = HourClock()
while True:
    time.sleep(3600)                      # wait an hour
    clock.tick()
    print(clock.hr)
```

The TLA+ model below describes the same behavior. Notice what it keeps and what it drops:

- **Kept:** the state (`self.hr` becomes `hr`), the startup choice (`__init__` becomes `Init`), and one tick (`tick()` becomes `Next`).
- **Dropped:** `sleep`, `print` and the `while` loop. Timing and output do not change which hours the clock can show, so the model leaves them out. `Spec` already says "keep taking steps forever."
- **Generalized:** `random.randint(1, 12)` becomes `hr \in 1..12`. The model does not pick one hour; it says *any* of them is possible, and TLC checks all 12.

```
---- MODULE HourClock ----
EXTENDS Naturals
VARIABLE hr

Init   == hr \in 1..12                          \* start at any hour
Next   == hr' = IF hr = 12 THEN 1 ELSE hr + 1    \* one tick
Spec   == Init /\ [][Next]_hr                    \* start with Init, then take Next steps

TypeOK == hr \in 1..12                           \* invariant: always a valid hour
====
```

Read it line by line:

- `VARIABLE hr` is the entire state of the system.
- `Init` says which states the system may start in: any of the 12 hours.
- `Next` describes one step. `hr'` ("hr prime") means the value of `hr` in the *next* state, so this line says "the next hour is one more, wrapping from 12 back to 1."
- `Spec` glues them together: begin in an `Init` state, and every later step is a `Next` step (or a step where nothing changes). Piece by piece, `Spec == Init /\ [][Next]_hr` reads:
  - `==` means "is defined as," like `def` in Python. Plain `=` is the equality test, as in `hr = 12`.
  - `/\` is logical AND (and `\/` is OR).
  - `Init`, used on its own, is a condition on the **first** state of an execution.
  - `[]` is the temporal operator "always": what follows must hold at **every** step of the execution, not just the first.
  - `[Next]_hr` is shorthand for `Next \/ hr' = hr`: each step is either a tick, or a step that leaves `hr` unchanged. Such do-nothing steps are called **stuttering** steps. They let the clock be part of a bigger model, where a minute hand might move while `hr` stays put.
  - Altogether: "the first state satisfies `Init`, and every step after that is either a tick or leaves `hr` alone."
  - One consequence: this `Spec` allows a clock that stutters forever, i.e., stops. It says what the clock *may* do, not that it *must* keep ticking. Requiring progress takes a **fairness** condition such as `WF_hr(Next)`, which comes back later when we discuss liveness.
- `TypeOK` is an **invariant**, a property that must be true in every reachable state.

Give this to the TLC model checker and it finds all 12 reachable states, checks `TypeOK` in each, and reports no errors. Now introduce a bug, say `ELSE hr + 2` without fixing the wraparound, and TLC reports a concrete trace such as 11 → 13 that breaks `TypeOK`.

That is the whole idea: describe states and steps, state what must always hold, and let the tool check every possibility. Real systems just have more variables and several actions that can fire in any order.

A TLA+ model is a state machine: variables, an initial state, and atomic **actions** that each say "if this guard holds, the variables change like this." Here is the core of a spinlock model:

```
Acquire(t) == /\ pc[t] \in {"trying", "spinning"}
              /\ IF lock_state = FALSE
                   THEN /\ lock_state' = TRUE
                        /\ pc' = [pc EXCEPT ![t] = "locked"]
                   ELSE /\ pc' = [pc EXCEPT ![t] = "spinning"]
                        /\ lock_state' = lock_state

MutualExclusion == Cardinality({t \in Threads : pc[t] = "locked"}) <= 1
```

Two modeling decisions hide in those few lines:

- **Invented state.** The Rust code has no `pc` variable; the modeler added it to track where each thread is.
- **Atomicity.** Testing and setting the lock happen in *one* action, because the real code uses an atomic `compare_exchange`. Split it into a read step and a write step, and TLC instantly finds two threads holding the lock. The model would be wrong, even though each line looks plausible.

That is the skill SysMoBench measures: reading code and choosing the right state, atomic steps and abstractions.

## The benchmark

Each task gives the AI a real system and asks for a TLA+ model plus a TLC configuration. To avoid arguments about the "right" level of detail, each task lists **actions that must be modeled** and **details that must be left out**. For the spinlock: model lock, unlock, the atomic compare-and-swap and spinning; skip the RAII guard and debug formatting.

There are 11 artifacts:

- **OS concurrency primitives** from the Asterinas Rust kernel: spinlock, mutex, rwmutex, ringbuffer (186–615 lines).
- **Production distributed systems:** Etcd Raft (Go), Redis Raft (C), Xline CURP (Rust), ZooKeeper leader election (Java, 5,360 lines).
- **Machine-generated Go** from the PGo compiler: a queue, a lock server and a Raft key-value store. Synthetic variable names make these hard to read for humans and LLMs alike.

## Grading without a gold model

This is the part I find most interesting. The authors refuse two easy options: comparing to a human-written reference model (many correct models exist, and real systems rarely have one) and LLM-as-a-judge (unreliable). Instead, four automatic checks run in sequence, each gating the next:

1. **Syntax.** Does the model parse with SANY? Partial credit comes from parsing each action on its own. A score of exactly 50% means every piece parses, but they don't fit together.
2. **Runtime.** Does TLC run it without errors? The score is the share of actions that actually fire without crashing.
3. **Conformance.** Does the model behave like the real code? The system is instrumented to log events; each log is replayed through the model with TLC. If the log shows a node becoming PreCandidate with its term unchanged, but the model jumps straight to Candidate and bumps the term, replay fails at that step.
4. **Invariants.** Does the model satisfy the system's correctness properties, such as mutual exclusion or "every acquired lock is eventually released"?

Two steps lean on an LLM: mapping the model's names to the log's names, and adapting invariant templates to the model's variables. The authors checked these by renaming and restructuring known-correct models; all 10 variants scored 100%.

The nice consequence is that adding a system takes days, not months. You need instrumentation and properties, not a reference model.

## What happened

Three simple agents were tested with Claude Sonnet 4, GPT-5, Gemini 2.5 Pro and DeepSeek R1: a **basic modeling** agent (read the code, write the model), a **code translation** agent (translate statement by statement, then assemble), and a **trace learning** agent (infer the model from logs alone).

- **Spinlock:** essentially solved. Every LLM passed syntax and runtime; conformance ranged from 80% to 100%.
- **Etcd Raft:** with the basic agent, only Claude Sonnet 4 got past syntax. Its model ran, but replayed only about 8% of real behavior. The translation agent did better, reaching about 15%.
- **Trace learning:** never produced a model that passed runtime checks.

A few lessons stood out to me.

- **Size and abstraction break today's LLMs.** Raft needs nested state (per-server logs, `nextIndex[i][j]`, in-flight messages), and the code is full of distractions.
- **Staying close to the code helps.** Statement-by-statement translation keeps the model from hallucinating logic, and from quietly reproducing a textbook Raft model it saw in training.
- **Liveness is the weak spot.** 41.9% of liveness properties failed versus 8.3% of safety properties, often because of missing or wrong fairness assumptions.
- **Read the metrics together.** One Etcd model passed 92% of invariants while matching only ~15% of real behavior. A model that does little can satisfy many safety properties for free.

## Are these models useful anyway?

Surprisingly, yes. Run on older code versions without any hint, AI-generated models rediscovered five real bugs, including a stale-read linearizability violation in Redis Raft and an Asterinas mutex where a *failed* `try_lock` released the lock.

How can models that mostly failed the conformance check still find bugs? Because a model does not need to be right about *everything* to catch a bug, only about the part where the bug lives.

- **Low scores came mostly from the big systems.** The Asterinas mutex and rwmutex are small, and models of small primitives scored high. That is where two of the five bugs were found.
- **Partial models are still useful.** Conformance measures how much of the *whole* system a model captures. A Raft model that gets leader election right but botches log replication can still expose an election bug.
- **Every bug report is checkable.** Model checking produces a concrete trace that leads to the violation. Engineers can replay that trace against the real code, so a false alarm from an inaccurate model gets caught rather than trusted.

In short, imperfect models work like a sharp but partial test: they cannot prove a system correct, but they can still find real problems.

The authors also tried two other specification languages instead of TLA+:

- [**Alloy**](https://alloytools.org) (Daniel Jackson, MIT): a lightweight modeling language based on relational logic, checked with the Alloy Analyzer by searching for counterexamples within small bounds.
- [**PAT**](https://pat.comp.nus.edu.sg) (Process Analysis Toolkit; Sun et al., CAV 2009, National University of Singapore): a model checker for concurrent and real-time systems, with models written in a CSP-style process language.

Results were near zero, with LLMs borrowing syntax from other languages. The authors' explanation: Alloy and PAT are far less used in industry than TLA+, so there is much less of them in LLM training data. That is a plausible hypothesis rather than something the paper measures, but it fits the errors, such as PAT models written in PRISM-style syntax.

## My take

SysMoBench's lasting contribution is its recipe: grade a model against the real system's traces and its expert-written properties, with partial credit at every stage.

**Main question I'd want answered:**

> Trace replay checks that observed executions are possible in the model, but not that the model forbids behaviors the code never shows. How do we catch over-permissive models?

Open questions I'd want answered next:

- Can the LLM that concretizes invariants accidentally weaken them, letting a broken model pass?
- Tasks list the actions to model. A real modeler must decide that too. How much harder is the task without that hint?
- Can conformance failures be fed back to the agent as a repair signal, the way test failures drive SWE-bench agents?

---

**Paper:** Qian Cheng, Ruize Tang, Emilie Ma, Finn Hackett, Peiyang He, Yiming Su, Ivan Beschastnikh, Yu Huang, Xiaoxing Ma, Tianyin Xu. *SysMoBench: Evaluating AI on Formally Modeling Complex Real-World Systems.* ICLR 2026. [arXiv:2509.23130](https://arxiv.org/abs/2509.23130)
