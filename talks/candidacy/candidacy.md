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
<h2 id="data-races-and-how-to-find-them">Data races: And how to find them?</h2>

A data race is two accesses to the same memory location from different threads, at least one of them a write, with nothing ordering them. 
I will discuss four data-race detection techniques that build on each other:

<!-- 1. **Lamport** defines happened-before, the ordering every race detector reasons about, and the clocks that compute it.
2. **Eraser** ignores ordering and checks a locking discipline instead: was one lock held for every access?
3. **Hybrid detection** reports a race only when both checks agree: no common lock, and no ordering from a deliberately limited happens-before.
4. **RaceFuzzer** takes the races these detectors suspect and steers the scheduler to make each one actually happen. -->

<!-- AIDEV-NOTE: precision/recall scale (pure HTML; styles in the <style> block below). Positions are qualitative:
     pure HB reports ⊆ hybrid ⊆ lockset (Hybrid post, Fig. 4); RaceFuzzer has no false alarms but only confirms
     races its Phase 1 detector suspects. Below 640px it becomes a vertical list. -->
<div class="tradeoff-scale" role="figure" aria-label="Precision versus recall of the four approaches">
  <div class="ts-ends">
    <span><strong>&larr; More precise</strong><small>fewer false alarms</small></span>
    <span><strong>More recall &rarr;</strong><small>fewer missed races</small></span>
  </div>
  <div class="ts-track">
    <div class="ts-line" aria-hidden="true"></div>
    <div class="ts-mark ts-p0">
      <span class="ts-dot" aria-hidden="true"></span>
      <span class="ts-name">Lamport</span>
      <span class="ts-note">Pure happens-before. No false alarms on the schedule it sees, but an unrelated lock hand-off can hide a real race.</span>
    </div>
    <div class="ts-mark ts-p1">
      <span class="ts-dot" aria-hidden="true"></span>
      <span class="ts-name">RaceFuzzer</span>
      <span class="ts-note">Reports only races it actually makes happen, but can only confirm races its detector suspected.</span>
    </div>
    <div class="ts-mark ts-p2">
      <span class="ts-dot" aria-hidden="true"></span>
      <span class="ts-name">Hybrid</span>
      <span class="ts-note">Lockset plus a limited happens-before. 
      <!-- Its reports sit between the two: pure HB &sube; hybrid &sube; lockset. -->
      </span>
    </div>
    <div class="ts-mark ts-p3">
      <span class="ts-dot" aria-hidden="true"></span>
      <span class="ts-name">Eraser (lockset)</span>
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

<!-- AIDEV-NOTE: h2 = topic section (big, ruled; styled below), h3 = paper. Each gets a ">" toggle that wraps
     what follows it in a .fold-body: an h3 up to the next h2/h3, an h2 up to the next h2 (so it nests papers).
     Moves all nodes, not just elements: kramdown emits $$ math as bare \[...\] text nodes between elements.
     Everything starts expanded; following a link into collapsed content opens every collapsed level above it.
     Figures are lazy and unsized, so after a jump they load late and push the target down: settle()
     loads the ones above the target right away, then re-aligns the target once they are in.
     No // comments in the script: the compress layout minifies HTML. -->
<style>
  .tradeoff-scale { margin: 2rem 0 1.5rem; }
  .ts-ends { display: flex; justify-content: space-between; gap: 1rem; margin-bottom: .6rem; }
  .ts-ends span { display: flex; flex-direction: column; line-height: 1.3; }
  .ts-ends span:last-child { text-align: right; }
  .ts-ends small { color: var(--text-muted-color); font-size: .8rem; }
  .ts-track { position: relative; min-height: 10.5rem; }
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
  .ts-note { color: var(--text-muted-color); font-size: .82rem; line-height: 1.45; margin-top: .2rem; }
  .ts-p0 { left: .9rem; align-items: flex-start; text-align: left; }
  .ts-p1 { left: 37%; transform: translateX(-50%); align-items: center; text-align: center; }
  .ts-p2 { left: 63%; transform: translateX(-50%); align-items: center; text-align: center; }
  .ts-p3 { right: .9rem; align-items: flex-end; text-align: right; }
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
