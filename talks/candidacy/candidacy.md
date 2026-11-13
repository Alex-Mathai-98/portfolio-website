---
title: "PhD Candidacy Exam: Paper Summaries"
layout: page
permalink: /candidacy/
toc: true
math: true
mermaid: false
---

<!-- AIDEV-NOTE: paper files (_data-races/*.md) started as verbatim copies of the linked _posts; edits
     in either place do not flow to the other. -->

Summaries of the papers I read for my PhD candidacy exam, grouped by topic. For each paper I keep only the part of my full post that runs the method end to end on a worked example. The full posts add the background, results and limitations.

<!-- AIDEV-NOTE: keep at least one raw <h2>: for pages, page.content is still Markdown when the layout
     runs, so toc-status.html never sees "<h2" and the TOC stays off. Later sections can use ##. -->
<h2 id="data-races-how-to-find-them">Data races: How to find them?</h2>

A data race is two accesses to the same memory location from different threads, at least one of them a write, with nothing ordering them. 
I will discuss four data-race detection techniques that build on each other:

<!-- 1. **Lamport** defines happened-before, the ordering every race detector reasons about, and the clocks that compute it.
2. **Eraser** ignores ordering and checks a locking discipline instead: was one lock held for every access?
3. **Hybrid detection** reports a race only when both checks agree: no common lock, and no ordering from a deliberately limited happens-before.
4. **RaceFuzzer** takes the races these detectors suspect and steers the scheduler to make each one actually happen. -->

<!-- AIDEV-NOTE: precision/recall scale (pure HTML; styles in the <style> block below). Positions are qualitative:
     pure HB reports ⊆ hybrid ⊆ lockset (Hybrid post, Fig. 4); RaceFuzzer has no false alarms but only confirms
     races its Phase 1 detector suspects. Below 640px it becomes a vertical list. -->
<div class="tradeoff-scale" role="figure" aria-label="Precision versus recall of the four approaches (five papers)">
  <div class="ts-ends">
    <span><strong>&larr; More precise</strong><small>fewer false alarms</small></span>
    <span><strong>More recall &rarr;</strong><small>fewer missed races</small></span>
  </div>
  <div class="ts-track">
    <div class="ts-line" aria-hidden="true"></div>
    <div class="ts-mark ts-p0">
      <span class="ts-dot" aria-hidden="true"></span>
      <span class="ts-name">Lamport + Vector Clocks</span>
      <span class="ts-cite">Lamport 1978 · Fidge & Mattern 1988</span>
      <span class="ts-note">Pure happens-before, computed exactly with vector clocks. No false alarms on the schedule it sees, but an unrelated lock hand-off can hide a real race.</span>
    </div>
    <div class="ts-mark ts-p1">
      <span class="ts-dot" aria-hidden="true"></span>
      <span class="ts-name">RaceFuzzer</span>
      <span class="ts-cite">Sen 2008</span>
      <span class="ts-note">Reports only races it actually makes happen, but can only confirm races its detector suspected.</span>
    </div>
    <div class="ts-mark ts-p2">
      <span class="ts-dot" aria-hidden="true"></span>
      <span class="ts-name">Hybrid</span>
      <span class="ts-cite">O'Callahan &amp; Choi 2003</span>
      <span class="ts-note">Lockset plus a limited happens-before. 
      <!-- Its reports sit between the two: pure HB &sube; hybrid &sube; lockset. -->
      </span>
    </div>
    <div class="ts-mark ts-p3">
      <span class="ts-dot" aria-hidden="true"></span>
      <span class="ts-name">Eraser (lockset)</span>
      <span class="ts-cite">Savage et al. 1997</span>
      <span class="ts-note">Flags every access without a common lock, so it misses the least but has false positives due to non-lock synchronization.</span>
    </div>
  </div>
  <!-- <p class="ts-caption">Positions are qualitative: they show the order of the trade-off, not measured values.</p> -->
</div>

<!-- AIDEV-NOTE: each paper lives in its own file under _data-races/ (underscore = not published on its own).
     Keep a blank line around every include so kramdown sees the headings. -->

{% include_relative _data-races/01-lamport.md %}

{% include_relative _data-races/02-eraser.md %}

{% include_relative _data-races/03-hybrid.md %}

{% include_relative _data-races/04-racefuzzer.md %}

## Concurrency Bugs are more than "just" Data Races

Data-race detectors look for one kind of mistake. A study of 105 real bugs shows that most concurrency bugs are about broken intentions, atomicity or order, and many of them involve no data race at all.

{% include_relative _concurrency-bugs/01-lu-learning-from-mistakes.md %}

## Concurrency Fixing

Finding a concurrency bug is only half the job. These two papers fix races automatically: LOOM installs a developer's workaround into a running program, and AFix synthesizes a lock-based patch from a bug report.

{% include_relative _concurrency-fixing/01-loom.md %}

{% include_relative _concurrency-fixing/02-afix.md %}

## Software Verification — the entire spectrum

{% include_relative _verification/01-spectrum.md %}

## LLM Agents + Model Checking

Model checkers prove properties exhaustively, but someone has to write the model or the specification. BMC-Agent and PAT-Agent let LLM agents write them and keep the checker as the judge: BMC-Agent infers function contracts for C and Rust code, and PAT-Agent writes a whole system model from English. SysMoBench measures how well LLMs can model real systems in the first place.

{% include_relative _llm-model-checking/01-bmc-agent.md %}

{% include_relative _llm-model-checking/02-pat-agent.md %}

{% include_relative _llm-model-checking/03-sysmobench.md %}

## LLM Agents + Deductive Verification

Deductive verifiers such as Verus prove code correct for every input, but only if someone supplies the right loop invariants and proof hints. When an LLM writes them and the proof fails, the verifier's error message says little about why. ExVerus turns that failure into concrete counterexamples the LLM can reason about.

{% include_relative _llm-deductive/01-exverus.md %}

## LLM + Proof Generation

Interactive provers such as Rocq give the strongest guarantee, but every proof step must be written out. AutoRocq lets an LLM agent write those steps, searching Rocq's libraries for the lemmas it needs. LemmaNet goes further and has the LLM invent new helper lemmas that no library contains. Anthropic's work drops the per-program spec altogether and proves whole libraries free of memory errors at the binary level.

{% include_relative _llm-proof-gen/01-basics.md %}

{% include_relative _llm-proof-gen/02-autorocq.md %}

{% include_relative _llm-proof-gen/03-lemmanet.md %}

{% include_relative _llm-proof-gen/04-anthropic.md %}

## Research Directions

{% include_relative _research-directions/01-concurrency-fixing-agents.md %}

{% include_relative _research-directions/02-verifying-concurrent-software.md %}

## References

Papers covered on this page, in the order they appear. Each title links to its section above.

**Data races: How to find them?**

1. Leslie Lamport. [Time, Clocks, and the Ordering of Events in a Distributed System](#1-lamports-algorithm-happens-before-algorithm). *Communications of the ACM*, 1978.
2. Colin J. Fidge. [Timestamps in Message-Passing Systems That Preserve the Partial Ordering](https://www.cs.uky.edu/~manivann/cs570/Fidge.pdf). *Australian Computer Science Conference*, 1988.
3. Friedemann Mattern. [Virtual Time and Global States of Distributed Systems](https://www.vs.inf.ethz.ch/publ/papers/VirtTimeGlobStates.pdf). *Workshop on Parallel and Distributed Algorithms*, 1988.
4. Bernadette Charron-Bost. Concerning the Size of Logical Clocks in Distributed Systems. *Information Processing Letters*, 1991.
5. Stefan Savage, Michael Burrows, Greg Nelson, Patrick Sobalvarro and Thomas Anderson. [Eraser: A Dynamic Data Race Detector for Multithreaded Programs](#2-lockset-analysis-eraser). *SOSP* 1997 and *ACM Transactions on Computer Systems*, 1997.
6. Robert O'Callahan and Jong-Deok Choi. [Hybrid Dynamic Data Race Detection](#3-hybrid-detection-lockset-plus-limited-happens-before). *PPoPP*, 2003.
7. Koushik Sen. [Race Directed Random Testing of Concurrent Programs](#4-racefuzzer-race-directed-random-testing). *PLDI*, 2008. [doi:10.1145/1375581.1375584](https://doi.org/10.1145/1375581.1375584).

**Concurrency Bugs are more than "just" Data Races**

{:start="8"}
8. Shan Lu, Soyeon Park, Eunsoo Seo and Yuanyuan Zhou. [Learning from Mistakes: A Comprehensive Study on Real World Concurrency Bug Characteristics](#5-learning-from-mistakes-real-world-concurrency-bug-characteristics). *ASPLOS*, 2008.

**Concurrency Fixing**

{:start="9"}
9. Jingyue Wu, Heming Cui and Junfeng Yang. [Bypassing Races in Live Applications with Execution Filters](#6-loom-bypassing-races-in-live-applications). *OSDI*, 2010. [USENIX page](https://www.usenix.org/conference/osdi10/bypassing-races-live-applications-execution-filters).
10. Guoliang Jin, Linhai Song, Wei Zhang, Shan Lu and Ben Liblit. [Automated Atomicity-Violation Fixing](#7-afix-automated-atomicity-violation-fixing). *PLDI*, 2011.

**LLM Agents + Model Checking**

{:start="11"}
11. Youcheng Sun, Jiawen Liu, Daniel Kroening and Jason Xue. [Agentic Model Checking](#9-bmc-agent-agentic-model-checking). [arXiv:2605.21434](https://arxiv.org/abs/2605.21434), 2026.
12. Xinyue Zuo, Yifan Zhang, Hongshu Wang, Yufan Cai, Zhe Hou, Jun Sun and Jin Song Dong. [PAT-Agent: Autoformalization for Model Checking](#10-pat-agent-autoformalization-for-model-checking). *ASE*, 2025. [arXiv:2509.23675](https://arxiv.org/abs/2509.23675).
13. Qian Cheng, Ruize Tang, Emilie Ma, Finn Hackett, Peiyang He, Yiming Su, Ivan Beschastnikh, Yu Huang, Xiaoxing Ma and Tianyin Xu. [SysMoBench: Evaluating AI on Formally Modeling Complex Real-World Systems](#11-sysmobench-evaluating-ai-on-formally-modeling-complex-real-world-systems). *ICLR*, 2026. [arXiv:2509.23130](https://arxiv.org/abs/2509.23130).

**LLM Agents + Deductive Verification**

{:start="14"}
14. Yang, Sun, Wu, Caridad, Yuan, Yao, Lu and Pei. [ExVerus: Verus Proof Repair via Counterexample Reasoning](#12-exverus-verus-proof-repair-via-counterexample-reasoning). [arXiv:2603.25810](https://arxiv.org/abs/2603.25810), 2026.

**LLM + Proof Generation**

{:start="15"}
15. Tu et al. [Agentic Verification of Software Systems](#14-autorocq-agentic-verification-of-software-systems). *FSE*, 2026. [doi:10.1145/3808164](https://doi.org/10.1145/3808164).
16. Zhao, Tu, Liu, Rinard and Roychoudhury. [Automated Lemma Discovery in Agentic Program Verification](#15-lemmanet-automated-lemma-discovery-in-agentic-program-verification). *ASE*, 2026. [doi:10.1145/3832783.3834356](https://doi.org/10.1145/3832783.3834356).
17. Anthropic. [Library-level verification of memory safety](#16-anthropic-library-level-verification-of-memory-safety). [Research post](https://www.anthropic.com/research/formal-verification); proofs at [theorem-labs/oss-verify-proofs](https://github.com/theorem-labs/oss-verify-proofs).

**LLM Agents for Crash Resolution**

{:start="18"}
18. **kGym.** kBench: A Benchmark & Platform to Test LLMs on Linux Kernel Crash Resolution. *NeurIPS*, 2024. [arXiv:2407.02680](https://arxiv.org/abs/2407.02680). [Project](https://github.com/Alex-Mathai-98/kGym-Kernel-Gym).
19. **Live-kGym.** Outrunning LLM Cutoffs: A Live Kernel Crash Resolution Benchmark for All. *ICML*, 2026. [arXiv:2602.02690](https://arxiv.org/abs/2602.02690).
20. **kAgent.** kAgent: An Execution-Guided Crash Resolution Agent for the Linux Kernel. *DL4C Workshop at ICML*, 2025. [arXiv:2504.20412](https://arxiv.org/abs/2504.20412).

**LLM Code Slop Reduction**

{:start="21"}
21. **TRIM.** TRIM: Reducing AI-Generated Code Slop via Agent Trajectory Minimization. Under submission, 2026. [arXiv:2607.18161](https://arxiv.org/abs/2607.18161).

<!-- AIDEV-NOTE: h2 = topic section (big, ruled; styled below), h3 = paper, h4 = paper subsection. Each gets
     a ">" toggle that wraps what follows it in a .fold-body, up to the next heading of the same or higher level
     (so an h2 nests its papers and an h3 its subsections). h4s are numbered "<paper>.<n>" here, not in the .md:
     <paper> is the h3's leading "N." (else its position). This runs before the layout's tocbot start-up
     (document listeners fire before window ones), so the TOC shows the numbers and toggles do not break it.
     Moves all nodes, not just elements: kramdown emits $$ math as bare \[...\] text nodes between elements.
     Everything starts expanded; following a link into collapsed content opens every collapsed level above it.
     Figures are lazy and unsized, so after a jump they load late and push the target down: settle()
     loads the ones above the target right away, then re-aligns the target once they are in.
     h5 (accent bar) and h6 (indented) are restyled below so sub-parts read as semi-titles.
     No // comments in the script: the compress layout minifies HTML. -->
<style>
  .tradeoff-scale { margin: 2rem 0 1.5rem; }
  .ts-ends { display: flex; justify-content: space-between; gap: 1rem; margin-bottom: .6rem; }
  .ts-ends span { display: flex; flex-direction: column; line-height: 1.3; }
  .ts-ends span:last-child { text-align: right; }
  .ts-ends small { color: var(--text-muted-color); font-size: .8rem; }
  .ts-track { position: relative; min-height: 15.5rem; }
  .ts-line {
    position: absolute; top: .55rem; left: 0; right: 0; height: 2px; background: var(--text-muted-color);
  }
  .ts-line::before, .ts-line::after {
    content: ""; position: absolute; top: -5px; border: 6px solid transparent;
  }
  .ts-line::before { left: -2px; border-right: 9px solid var(--text-muted-color); border-left: 0; }
  .ts-line::after { right: -2px; border-left: 9px solid var(--text-muted-color); border-right: 0; }
  .ts-mark { position: absolute; top: 0; width: 23%; display: flex; flex-direction: column; }
  .ts-dot {
    width: 1.1rem; height: 1.1rem; border-radius: 50%; background: var(--link-color);
    border: 3px solid var(--card-bg, var(--main-bg)); box-sizing: border-box; margin-bottom: .5rem;
  }
  .ts-name { font-weight: 600; color: var(--heading-color, var(--text-color)); line-height: 1.3; }
  .ts-cite { color: var(--text-muted-color); font-size: .75rem; font-style: italic; line-height: 1.3; }
  .ts-note { color: var(--text-muted-color); font-size: .82rem; line-height: 1.45; margin-top: .2rem; }
  .ts-p0 { left: .9rem; align-items: flex-start; text-align: left; }
  .ts-p1 { left: 37%; transform: translateX(-50%); align-items: center; text-align: center; }
  .ts-p2 { left: 63%; transform: translateX(-50%); align-items: center; text-align: center; }
  .ts-p3 { right: .9rem; align-items: flex-end; text-align: right; }
  .ts-pm { left: 50%; transform: translateX(-50%); align-items: center; text-align: center; }
  .ts-caption { color: var(--text-muted-color); font-size: .8rem; margin-top: .4rem; }
  @media (max-width: 640px) {
    .ts-ends { display: none; }
    .ts-track { min-height: 0; padding-left: 1.6rem; }
    .ts-track::before { content: "\2191  More precise"; display: block; font-weight: 600; margin-bottom: .6rem; }
    .ts-track::after { content: "\2193  More recall"; display: block; font-weight: 600; margin-top: .2rem; }
    .ts-line { top: 2.2rem; bottom: 2.2rem; left: .5rem; right: auto; width: 2px; height: auto; }
    .ts-line::before, .ts-line::after { display: none; }
    .ts-mark {
      position: relative; left: auto; right: auto; width: auto; transform: none;
      align-items: flex-start; text-align: left; margin-bottom: 1rem;
    }
    .ts-dot { position: absolute; left: -1.6rem; top: .1rem; margin: 0; }
  }
  .content table.wrap th, .content table.wrap td { white-space: normal; min-width: 7rem; vertical-align: top; }
  .content table.wrap code { white-space: nowrap; }
  .content table.compact th, .content table.compact td { padding: .4rem .55rem; text-align: center; }
  .content table.compact td:first-child { text-align: left; }
  .content table.compact thead th { white-space: normal; }
  article .content h5 {
    font-size: 1.2rem; font-weight: 600; margin-top: 2.2rem; padding: .15rem 0 .15rem .7rem;
    border-left: 4px solid var(--link-color);
  }
  article .content h6 {
    font-size: 1.02rem; font-weight: 600; margin: 1.6rem 0 .6rem 1rem; color: var(--heading-color, var(--text-color));
  }
  article .content h2 {
    font-size: 1.875rem; font-weight: 500; margin-top: 3rem; padding-bottom: .5rem;
    border-bottom: 2px solid var(--main-border-color, rgba(127, 127, 127, .35));
  }
  .fold-toggle {
    border: 0; background: none; color: inherit; padding: 0 .4rem 0 0; cursor: pointer;
    font-size: .85em; vertical-align: .05em; line-height: 1;
  }
  .fold-toggle i { transition: transform .2s ease; }
  .fold-toggle[aria-expanded="true"] i { transform: rotate(90deg); }
  .fold-body[hidden] { display: none !important; }
</style>
<script>
  document.addEventListener('DOMContentLoaded', function () {
    const content = document.querySelector('article .content');
    if (!content) return;
    let n = 0;
    const fold = function (heading, stop, label) {
      const body = document.createElement('div');
      body.className = 'fold-body';
      body.id = 'fold-body-' + (n++);
      let next = heading.nextSibling;
      while (next && !stop.test(next.nodeName)) {
        const after = next.nextSibling;
        body.appendChild(next);
        next = after;
      }
      heading.after(body);
      const btn = document.createElement('button');
      btn.type = 'button';
      btn.className = 'fold-toggle';
      btn.setAttribute('aria-expanded', 'true');
      btn.setAttribute('aria-controls', body.id);
      btn.setAttribute('aria-label', 'Collapse or expand this ' + label);
      btn.innerHTML = '<i class="fa-solid fa-angle-right fa-fw"></i>';
      btn.addEventListener('click', function () {
        const open = btn.getAttribute('aria-expanded') === 'true';
        btn.setAttribute('aria-expanded', String(!open));
        body.hidden = open;
      });
      heading.prepend(btn);
    };
    let paper = 0, sub = 0;
    content.querySelectorAll('h2, h3, h4').forEach(function (h) {
      if (h.tagName === 'H2') { paper = 0; return; }
      if (h.tagName === 'H3') {
        const m = h.textContent.trim().match(/^(\d+)\./);
        paper = m ? Number(m[1]) : paper + 1;
        sub = 0;
        return;
      }
      const num = document.createElement('span');
      num.className = 'heading-num';
      num.textContent = paper + '.' + (++sub) + ' ';
      h.prepend(num);
    });
    content.querySelectorAll('h4').forEach(function (h) { fold(h, /^H[234]$/, 'subsection'); });
    content.querySelectorAll('h3').forEach(function (h) { fold(h, /^H[23]$/, 'paper'); });
    content.querySelectorAll('h2').forEach(function (h) { fold(h, /^H2$/, 'section'); });
    const openFor = function (id) {
      let el = id && document.getElementById(decodeURIComponent(id));
      while (el) {
        const body = el.parentElement && el.parentElement.closest('.fold-body');
        if (body && body.hidden) {
          body.hidden = false;
          document.querySelector('[aria-controls="' + body.id + '"]').setAttribute('aria-expanded', 'true');
        }
        el = body;
      }
    };
    const settle = function (id) {
      const target = id && document.getElementById(decodeURIComponent(id));
      if (!target) return;
      const waiting = [...content.querySelectorAll('img')].filter(function (img) {
        return !img.complete && (img.compareDocumentPosition(target) & Node.DOCUMENT_POSITION_FOLLOWING);
      });
      if (!waiting.length) return;
      waiting.forEach(function (img) { img.loading = 'eager'; });
      Promise.all(waiting.map(function (img) {
        return new Promise(function (done) {
          img.addEventListener('load', done, { once: true });
          img.addEventListener('error', done, { once: true });
        });
      })).then(function () {
        if (location.hash === '#' + id) target.scrollIntoView();
      });
    };
    const go = function (id) { openFor(id); settle(id); };
    document.addEventListener('click', function (e) {
      const a = e.target.closest('a[href*="#"]');
      if (a && a.pathname === location.pathname) go(a.hash.slice(1));
    }, true);
    window.addEventListener('hashchange', function () { go(location.hash.slice(1)); });
    go(location.hash.slice(1));
  });
</script>
