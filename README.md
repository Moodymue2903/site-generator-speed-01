# site-generator-speed-01

[![make.fun](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/competitions/site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/Moodymue2903/site-generator-speed-01/rank.svg)](https://make.fun/competitions/site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/Moodymue2903/site-generator-speed-01/reward.svg)](https://make.fun/competitions/site-generator-speed-01-run-20261122)

A faster drop-in for make.fun's own `makefun-site` 1.0.0, built for the
[site-generator-speed-01-run-20261122](https://make.fun/competitions/site-generator-speed-01-run-20261122)
competition ("Build a static site faster, 22 November 2026").

`run(input)` returns exactly what `makefun-site` 1.0.0 returns for a generated
site — the output tree (each page's `index.html`, the index and tag pages,
`build-report.txt`, and each static file) as a JSON object of path to file
text, byte for byte — but about 3x faster than the reference on the practice
set (median ~205 µs vs ~618 µs for the house baseline over the 3 practice
cases, measured locally with Node 22).

## How it works

`makefun-site` is a general tool; this is a single-purpose reimplementation
of its exact behaviour, specialised for the one job:

- **Markdown:** a fused lexer + renderer covering only the subset
  `makefun-site` uses (fenced code, ATX headings with ids, thematic breaks,
  blockquotes, flat lists, paragraphs; inline code spans, `**` strong,
  single `*` emphasis, `[text](target)` links, backslash escapes). No token
  array, no per-line regex battery — direct character checks, plain-text runs
  copied in bulk, and a single native regex scan for the next inline
  construct or HTML special character.
- **Templates:** the dialect only (`{{ path }}` escaped, `{{{ path }}}` raw,
  `{% for %}` with `loop`, `{% if %}` / `{% else %}` / `{% if not %}`,
  `{% include %}` to depth 8). Templates are parsed once per case; rendering
  reuses loop scopes instead of allocating per iteration, and shares one root
  scope for `site` / `tags` / `recent`.
- **Slugs and front matter:** direct character scans instead of regexes, with
  an ASCII fast path for `slugify` (full NFKD behaviour kept for non-ASCII).
- **No per-file overhead the reference pays:** no plugin pipeline, no
  intermediate file objects, minimal allocations (no state leaked between
  cases).

The reference implementation's code was read to learn its exact behaviour and
is the authority wherever the two differ; `solve.js` is written from scratch
and shares no code with it.

## Files

- `solve.js` — dependency-free ES module. `run(input)` returns the output
  tree for one case's `{ site, templates, files }` input.
- `main.ts` — the driver: reads every case from `/input/cases.json`, answers
  each with `run`, writes `/output/answers.json`.
- `DESIGN.md` — the design: the Markdown subset, the template features, and
  the optimisation levers.
- `LICENSE` — MIT.

## Verify locally

```sh
node cli/makefun.mjs grade-local --kit <kit> --bot <this repo> --driver process
```

Practice set: `passed=1`, `cases_passed=3/3`, L1 regrade reproduced byte for
byte.
