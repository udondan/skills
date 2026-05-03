---
name: browser-automation
description: How to interact with a Chrome browser via MCP Chromium tools (mcp__chromium__*). Use this skill whenever the user asks you to click, fill, hover, select, navigate, scroll, submit, or automate anything in a browser — even if they just say "go to X and do Y" or "open the page and fill in the form". Any task that involves a browser and requires touching a page element should use this skill. Do not skip the snapshot step even for simple one-click tasks.
---

# Browser Automation via MCP Chromium

When working with a browser through MCP Chromium tools, the core challenge is finding the right element to interact with. Pages are complex — many elements have no obvious selector, some share class names, and raw HTML can be misleading about what's actually visible. The approach here avoids all that fragility by taking a **page snapshot first**: a JS script that labels every visible interactive element with a stable `data-claude-ref` attribute, giving you a clean, human-readable map of what's on screen before you touch anything.

## The snapshot script

Run this via `mcp__chromium__evaluate` before any interaction. It clears stale refs, finds all visible interactive elements, and returns a numbered list with labels.

```javascript
(function() {
  document.querySelectorAll('[data-claude-ref]').forEach(el => el.removeAttribute('data-claude-ref'));
  const interactive = [
    'a[href]', 'button', 'input:not([type="hidden"])', 'select', 'textarea',
    '[role="button"]', '[role="link"]', '[role="menuitem"]', '[role="tab"]',
    '[role="checkbox"]', '[role="radio"]', '[role="combobox"]',
    '[role="option"]', '[role="switch"]', '[tabindex]:not([tabindex="-1"])'
  ];
  const seen = new Set();
  const elements = [];
  for (const el of document.querySelectorAll(interactive.join(','))) {
    if (!seen.has(el)) { seen.add(el); elements.push(el); }
  }
  const visible = elements.filter(el => {
    const r = el.getBoundingClientRect();
    if (r.width === 0 || r.height === 0) return false;
    const s = window.getComputedStyle(el);
    if (s.visibility === 'hidden' || s.display === 'none' || s.opacity === '0') return false;
    const cx = r.left + r.width / 2;
    const cy = r.top + r.height / 2;
    const hit = document.elementFromPoint(cx, cy);
    return hit === el || el.contains(hit);
  });
  const lines = [];
  visible.forEach((el, i) => {
    const ref = `ref${i + 1}`;
    el.setAttribute('data-claude-ref', ref);
    const tag = el.tagName.toLowerCase();
    const role = el.getAttribute('role') || tag;
    const type = el.getAttribute('type') ? `[${el.getAttribute('type')}]` : '';
    const label =
      el.getAttribute('aria-label')?.trim() ||
      el.getAttribute('title')?.trim() ||
      el.getAttribute('placeholder')?.trim() ||
      el.textContent?.trim().replace(/\s+/g, ' ').slice(0, 60) ||
      el.getAttribute('name') ||
      el.getAttribute('value')?.slice(0, 30) ||
      '(no label)';
    const disabled = el.disabled || el.getAttribute('aria-disabled') === 'true' ? ' [disabled]' : '';
    lines.push(`[${ref}] ${role}${type}${disabled}: "${label}"`);
  });
  return `=== PAGE SNAPSHOT (${lines.length} elements) ===\n` + lines.join('\n');
})()
```

The output looks like:
```
=== PAGE SNAPSHOT (12 elements) ===
[ref1] a: "Home"
[ref2] a: "Products"
[ref3] button: "Sign in"
[ref4] input[text]: "Search"
[ref5] button: "Search"
...
```

## Workflow

**Step 1 — Navigate** (if needed): Use `mcp__chromium__navigate` to open the page.

**Step 2 — Snapshot**: Run the snapshot script via `mcp__chromium__evaluate`. Read the element list and find the target by its meaning or label — not by position.

**Step 3 — Interact**: Use the ref as the CSS selector. For example:
- Click: `mcp__chromium__click` with selector `[data-claude-ref="ref3"]`
- Fill a text field: `mcp__chromium__fill` with selector `[data-claude-ref="ref4"]`
- Hover: `mcp__chromium__hover` with selector `[data-claude-ref="ref2"]`
- Select a dropdown option: `mcp__chromium__select` with selector `[data-claude-ref="ref7"]`

**Step 4 — Re-snapshot after change**: After any navigation, form submission, modal open/close, or anything that meaningfully changes the DOM, re-run the snapshot before the next interaction. Refs from the previous snapshot are stale.

## When to re-run the snapshot

Re-run whenever:
- The URL changes
- A modal, drawer, or overlay opens or closes
- The page loads new content dynamically (e.g. a search result appears)
- You clicked something and the page state visibly changed

You don't need to re-run between two interactions on the same static page state.

## Matching elements by meaning

The label in the snapshot comes from aria-label, title, placeholder, text content, name, or value — in that priority order. Match by intent:
- "the search box" → find `input[text]` with label containing "search" or "Search"
- "the sign-in button" → find `button` or `[role="button"]` with label like "Sign in"
- "the username field" → find `input[text]` or `input[email]` with label like "Username" or "Email"

If multiple elements could match, prefer the one that's most specific to the user's intent. If still ambiguous, take a screenshot (`mcp__chromium__screenshot`) to orient yourself visually.

## Handling difficult pages

- **Shadow DOM / iframes**: The snapshot script runs in the top-level document. If the target is inside a shadow root or iframe, you may need to evaluate inside that context explicitly.
- **Dynamically loaded content**: If the snapshot returns 0 elements or fewer than expected, the page may still be loading. Wait briefly and re-run.
- **No matching element**: If you can't find the right element in the snapshot, take a screenshot to see what's on screen, then decide whether to scroll, wait, or refine the search.
- **Disabled elements**: The snapshot marks disabled elements — don't try to interact with them. Look for an alternative (e.g., fill in required fields first).
