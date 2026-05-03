# browser-automation

A Claude Code skill for reliable browser automation via MCP Chromium tools.

## What it does

Instead of guessing CSS selectors from raw HTML, this skill gives Claude a structured approach: before touching any element, run a snapshot script that labels every visible interactive element on the page with a stable `data-claude-ref` attribute. Claude then clicks, fills, or hovers by ref — not by selector.

This makes automation more reliable on dynamic pages, pages with ambiguous markup, and multi-step flows where the DOM changes between interactions.

## Installation

```bash
claude skill install browser-automation
```

Requires an MCP server that exposes `mcp__chromium__*` tools (navigate, evaluate, click, fill, etc.). Compatible with [mcp-chromium](https://github.com/anthropics/mcp-chromium) and similar Chromium-based MCP servers.

## Usage

Just ask Claude to interact with a browser — the skill triggers automatically:

> "Go to https://news.ycombinator.com and click the 'new' link"

> "Open https://duckduckgo.com, search for 'Raspberry Pi 5', and submit"

> "Go to the GitHub repo at axios/axios, open the Issues tab, and click the first open issue"

Claude will always run the snapshot before touching anything, and re-run it after any navigation or DOM change.

## Origin

The idea is directly lifted from how `claude --chrome` works internally.

`claude --chrome` connects Claude to your real browser via the Claude in Chrome extension. When Claude needs to interact with a page, it calls `take_snapshot` — which returns the page's **accessibility tree** with every interactive element assigned a stable UID. Claude finds the target by meaning, then calls `click(uid: "...")`. It never has to parse HTML or guess a CSS selector.

This skill approximates that same pattern for MCP Chromium servers, which only expose raw CSS-selector-based tools (`mcp__chromium__click(selector)`). Instead of an accessibility tree, a small JS snippet injected via `mcp__chromium__evaluate` walks the live DOM, finds all visible interactive elements, assigns `data-claude-ref="ref1"`, `ref2`, etc., and returns the same kind of semantic list. Claude then clicks by ref instead of UID.

The result is the same: Claude reasons about elements by what they *mean*, not where they are in the markup.

## Why this approach

The naive approach — deriving selectors from raw HTML — breaks in a few common situations:

- **Dynamic content**: the HTML you read before JS runs doesn't match what's visible on screen
- **Ambiguous elements**: many elements share class names or tag types; labels disambiguate
- **Multi-step flows**: after a click the DOM changes completely; old selectors are stale
- **Shadow DOM / framework-rendered UI**: structure in the HTML rarely maps 1:1 to what's clickable

The snapshot script interrogates the live DOM — what's actually rendered and visible — and gives Claude a human-readable map (`[ref3] a: "new"`) to reason about rather than raw markup.

## Benchmark results

Tested against 6 evals across simple, dynamic, and multi-step scenarios:

| | With skill | Without skill |
|---|---|---|
| Pass rate | **100%** | 43% |
| npm dynamic search — tool calls | 7 | 8 |
| GitHub multi-step — tool calls | 6 | 8 |
| Wikipedia TOC (many similar links) — tool calls | **5** | **12** |

Without the skill, the agent succeeded on simple tasks but hit token-limit errors on `get_content`, needed screenshot fallbacks, and made syntax errors in ad-hoc `evaluate` calls — resulting in 2.4× more tool calls on complex pages.

## Comparison to vercel-labs/agent-browser

[agent-browser](https://github.com/vercel-labs/agent-browser) (226K installs) uses the same conceptual loop — snapshot → click by ref → re-snapshot — but as a native Rust CLI binary rather than an in-browser JS snippet. Its refs come from the accessibility tree (`@e1`, `@e2`) rather than DOM injection.

We ran the same evals against both, using `--headed` mode for agent-browser to match our MCP Chromium's non-headless configuration. Two findings:

**1. Snapshot size.** The a11y tree on real pages is large. agent-browser's `snapshot -i` returned 48K characters for the GitHub repo homepage, 81K for a Wikipedia article, and 34K for the npmjs.com package page. Our script filters to *visible interactive elements only* — it ignores structural nodes, headings, and invisible elements — so the output stays compact regardless of page complexity. On token-constrained tasks this is a meaningful difference.

**2. Command count.** On equivalent tasks:

| Task | Our skill (MCP calls) | agent-browser (Bash calls) |
|---|---|---|
| npm search + click package | **7**, no retries | 19, 1 session retry |
| GitHub repo → Issues → first issue | **6** | 8 |
| Wikipedia TOC → History | **5** | 8, 1 syntax error |

The command count difference comes partly from retries and errors, partly from agent-browser needing explicit `get url` and `close` calls that our approach doesn't require.

The tools are complementary rather than competing: agent-browser is a full-featured standalone tool with its own browser management; this skill is a lightweight overlay for environments that already have an MCP Chromium server running.

## Skill structure

```
browser-automation/
├── SKILL.md        # Skill instructions + snapshot script
└── evals/
    └── evals.json  # 6 eval prompts with assertions
```
