---
name: presentation
description: >
  Creating animated technical presentations using RevealJS.
  Always applies Aidan's brand colours and typography.
  Use when communicating a design proposal, system explanation,
  feature walkthrough, or technical decision to an audience.
---

# Presentations

Technical presentations should communicate clearly and look considered.
Slides built with RevealJS are code — version-controllable, portable, and
consistent with a brand rather than dependent on a presentation tool's defaults.

---

## Brand

Always apply Aidan's brand palette and typography. No exceptions.

| Role | Name | Hex | Usage |
|---|---|---|---|
| Primary / accent | Marsala | `#964F4C` | Headers, highlights, borders, key callouts |
| Background | Cloud Dancer | `#F0EEE9` | Slide backgrounds, light sections |
| Text / dark | Black Beauty | `#27272A` | Body text, code, dark backgrounds |

Typography:
- **Display / headings**: a serif or distinctive display font — never Inter or Roboto
- **Body**: clean, readable, high contrast
- **Code**: monospace — `JetBrains Mono` or `Fira Code` preferred

---

## Base RevealJS Template

Every presentation starts from this template. **Dark mode is the default.**
A toggle button is included on every slide — press it to switch to light mode for
corporate or stakeholder settings where a dark deck may not land as well.

```html
<!DOCTYPE html>
<html lang="en" data-theme="dark">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>{{TITLE}}</title>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/reveal.js/5.1.0/reveal.min.css" />
  <style>
    /* ── Brand tokens ───────────────────────────────────────── */
    :root {
      --marsala:       #964F4C;
      --cloud-dancer:  #F0EEE9;
      --black-beauty:  #27272A;
      --marsala-dim:   #7a3f3c;
      --marsala-light: #c47f7c;
    }

    /* ── Dark mode (default) ────────────────────────────────── */
    [data-theme="dark"] {
      --slide-bg:      var(--black-beauty);
      --slide-text:    var(--cloud-dancer);
      --heading-color: var(--marsala);
      --muted-color:   var(--cloud-dancer);
      --code-bg:       #1a1a1c;
      --code-text:     var(--marsala-light);
      --hr-color:      var(--marsala);
      --callout-bg:    rgba(150, 79, 76, 0.12);
      --toggle-bg:     rgba(240, 238, 233, 0.08);
      --toggle-color:  var(--cloud-dancer);
      --toggle-label:  "☀ light";
    }

    /* ── Light mode ─────────────────────────────────────────── */
    [data-theme="light"] {
      --slide-bg:      var(--cloud-dancer);
      --slide-text:    var(--black-beauty);
      --heading-color: var(--marsala);
      --muted-color:   var(--black-beauty);
      --code-bg:       #e8e6e1;
      --code-text:     var(--marsala-dim);
      --hr-color:      var(--marsala);
      --callout-bg:    rgba(150, 79, 76, 0.08);
      --toggle-bg:     rgba(39, 39, 42, 0.08);
      --toggle-color:  var(--black-beauty);
      --toggle-label:  "☾ dark";
    }

    /* ── Reveal overrides ───────────────────────────────────── */
    .reveal-viewport { background: var(--slide-bg); transition: background 0.3s ease; }

    .reveal .slides section {
      background: var(--slide-bg);
      color: var(--slide-text);
      font-family: 'Georgia', 'Times New Roman', serif;
      text-align: left;
      padding: 2rem 3rem;
      height: 100%;
      box-sizing: border-box;
      transition: background 0.3s ease, color 0.3s ease;
    }

    /* ── Typography ─────────────────────────────────────────── */
    .reveal h1, .reveal h2 {
      font-family: 'Georgia', serif;
      color: var(--heading-color);
      text-transform: none;
      letter-spacing: -0.02em;
      line-height: 1.1;
      margin-bottom: 1rem;
    }
    .reveal h1 { font-size: 2.8rem; }
    .reveal h2 { font-size: 2rem; }
    .reveal h3 {
      font-size: 1.2rem;
      color: var(--muted-color);
      text-transform: uppercase;
      letter-spacing: 0.12em;
      opacity: 0.6;
      font-weight: 400;
      margin-bottom: 0.5rem;
    }
    .reveal p, .reveal li {
      font-size: 1.1rem;
      line-height: 1.7;
      color: var(--slide-text);
    }

    /* ── Code blocks ────────────────────────────────────────── */
    .reveal pre {
      background: var(--code-bg);
      border-left: 3px solid var(--marsala);
      border-radius: 4px;
      padding: 1.2rem 1.5rem;
      font-size: 0.85rem;
      width: 100%;
      margin: 1rem 0;
      transition: background 0.3s ease;
    }
    .reveal code {
      font-family: 'JetBrains Mono', 'Fira Code', monospace;
      color: var(--code-text);
    }

    /* ── Accent elements ────────────────────────────────────── */
    .reveal .accent { color: var(--marsala); font-weight: 600; }
    .reveal .muted  { opacity: 0.5; font-size: 0.9rem; }
    .reveal .label  {
      display: inline-block;
      background: var(--marsala);
      color: var(--cloud-dancer);
      padding: 0.15rem 0.6rem;
      border-radius: 3px;
      font-size: 0.75rem;
      letter-spacing: 0.08em;
      text-transform: uppercase;
    }

    /* ── Divider / rule ─────────────────────────────────────── */
    .reveal hr {
      border: none;
      border-top: 1px solid var(--hr-color);
      opacity: 0.3;
      margin: 1.5rem 0;
    }

    /* ── Title slide ────────────────────────────────────────── */
    .title-slide {
      display: flex !important;
      flex-direction: column;
      justify-content: flex-end;
      padding-bottom: 3rem !important;
      border-left: 6px solid var(--marsala);
      padding-left: 3rem !important;
    }
    .title-slide h1 { font-size: 3.5rem; margin-bottom: 0.5rem; }
    .title-slide .subtitle {
      color: var(--slide-text);
      opacity: 0.6;
      font-size: 1.1rem;
      margin-bottom: 0.5rem;
    }
    .title-slide .meta {
      color: var(--marsala);
      font-size: 0.9rem;
      letter-spacing: 0.08em;
      text-transform: uppercase;
    }

    /* ── Two-column layout ──────────────────────────────────── */
    .cols {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 2rem;
      align-items: start;
    }
    .cols.wide { grid-template-columns: 2fr 1fr; }

    /* ── Callout box ────────────────────────────────────────── */
    .callout {
      background: var(--callout-bg);
      border-left: 3px solid var(--marsala);
      padding: 1rem 1.2rem;
      border-radius: 0 4px 4px 0;
      margin: 1rem 0;
    }

    /* ── Fragment animations ────────────────────────────────── */
    .reveal .fragment { transition: all 0.3s ease; }

    /* ── Theme toggle button ────────────────────────────────── */
    #theme-toggle {
      position: fixed;
      top: 1rem;
      right: 1rem;
      z-index: 1000;
      background: var(--toggle-bg);
      color: var(--toggle-color);
      border: 1px solid currentColor;
      border-radius: 4px;
      padding: 0.3rem 0.7rem;
      font-family: 'JetBrains Mono', 'Fira Code', monospace;
      font-size: 0.7rem;
      letter-spacing: 0.05em;
      cursor: pointer;
      opacity: 0.5;
      transition: opacity 0.2s ease, background 0.3s ease, color 0.3s ease;
    }
    #theme-toggle:hover { opacity: 1; }
  </style>
</head>
<body>

<!-- Theme toggle — visible on every slide -->
<button id="theme-toggle" title="Toggle light / dark mode">☀ light</button>

<div class="reveal">
  <div class="slides">

    <!-- Title slide -->
    <section class="title-slide">
      <div class="meta">{{DATE}} · {{CONTEXT}}</div>
      <h1>{{TITLE}}</h1>
      <div class="subtitle">{{SUBTITLE}}</div>
    </section>

    <!-- Content slides go here -->

  </div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/reveal.js/5.1.0/reveal.min.js"></script>
<script>
  Reveal.initialize({
    hash: true,
    transition: 'fade',
    transitionSpeed: 'fast',
    backgroundTransition: 'fade',
    controls: false,
    progress: true,
    center: false,
    width: 1280,
    height: 720,
  });

  // ── Theme toggle ─────────────────────────────────────────
  const btn = document.getElementById('theme-toggle');
  const html = document.documentElement;

  btn.addEventListener('click', () => {
    const next = html.dataset.theme === 'dark' ? 'light' : 'dark';
    html.dataset.theme = next;
    btn.textContent = next === 'dark' ? '☀ light' : '☾ dark';
  });
</script>
</body>
</html>
```

---

## Slide Patterns

### Standard content slide

```html
<section>
  <h3>Context label</h3>
  <h2>Slide Title</h2>
  <hr>
  <p>Opening point that frames what follows.</p>
  <ul>
    <li class="fragment">First point — revealed on advance</li>
    <li class="fragment">Second point</li>
    <li class="fragment">Third point</li>
  </ul>
</section>
```

### Two-column slide

```html
<section>
  <h2>Comparison or split content</h2>
  <hr>
  <div class="cols">
    <div>
      <span class="label">Before</span>
      <pre><code>// old approach</code></pre>
    </div>
    <div>
      <span class="label">After</span>
      <pre><code>// new approach</code></pre>
    </div>
  </div>
</section>
```

### Callout / key point slide

```html
<section>
  <h2>Key Decision</h2>
  <hr>
  <div class="callout">
    <p><span class="accent">Core principle:</span> the statement being emphasised.</p>
  </div>
  <p class="fragment">Supporting context that follows the callout.</p>
</section>
```

### Section divider

```html
<section style="display:flex; flex-direction:column; justify-content:center;">
  <h3 style="opacity:0.4;">Part 02</h3>
  <h1>Section Title</h1>
</section>
```

---

## Animation Defaults

- **Slide transition**: `fade` — professional, not distracting
- **Fragment reveals**: use `.fragment` on list items and callouts to control pacing
- **Speed**: `fast` — snappy, not sluggish

Do not use `zoom`, `convex`, or `cube` transitions — they undermine a technical,
professional tone.

---

## Content Rules

- One idea per slide — not one topic per slide
- Never paste more than ~8 lines of code on a slide — use a callout or abbreviate
- Diagrams from the `diagramming` skill can be embedded as inline Mermaid or SVG
- No bullet points longer than one line — if it needs more, it is a paragraph, not a bullet
- Use `.fragment` to pace reveals — do not dump all content at once
- Labels (`<span class="label">`) replace slide headers for categorisation within a deck

---

## Presentation Types

| Purpose | Structure |
|---|---|
| Technical design proposal | Problem → Options → Recommendation → Trade-offs → Next steps |
| System explanation | Context → Components → Interactions → Constraints |
| Feature walkthrough | Goal → Demo flow → Implementation notes → Open questions |
| Decision record | Background → Decision → Consequences → Alternatives considered |
