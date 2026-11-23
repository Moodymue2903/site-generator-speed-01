# Design: a faster drop-in for makefun-site 1.0.0

`run(input)` returns exactly what makefun-site 1.0.0's `build` returns for one
case's `{ site, templates, files }` input: the output tree as a JSON object of
path -> file text, byte for byte. The reference (kit
`baselines/makefun-site/vendor/makefun-site/`) was read in full and is the
authority wherever behaviour differs; `solve.js` is written from scratch.

## The Markdown subset (only what makefun-site uses)

Blocks, in order: fenced code (``` or ~~~, info string is a language class),
ATX headings 1-6 (each with an id = the unique slug of its heading text),
thematic breaks (`---`, `***` and the `___` / spaced variants the reference
accepts), blockquotes (lines opened on `>`, one level, no nesting), flat lists
(no nesting; `-`, `*`, `+` bullets and `1.` / `1)` ordered, a list item
continues while its lines are indented or blank-then-indented), paragraphs
(a paragraph continues until a blank line, fence, heading or thematic break;
a quote or list line inside a paragraph is paragraph text).

Inline: backslash escapes (only the escapable set), code spans (a single
backtick pair — the reference does not do multi-backtick runs), `**` strong,
single `*` emphasis, `[text](target)` links (the target must be non-empty
and hold no whitespace), and HTML escaping of `& < > " '`. `_` is always
itself. Raw HTML is text, never markup. No setext headings, no indented code,
no tables, no reference links, no autolinks.

Headings of level 2 and 3 feed the table of contents (`level`, `id`, `html`);
the excerpt is the first paragraph's inline HTML, or `""`.

## The template dialect (only what makefun-site uses)

- `{{ path }}` escaped, `{{{ path }}}` raw. A path is names joined by `.`.
  Lookup: the first name from the innermost loop scope holding it (`loop`
  before the loop element), else the page's values; each next name an own
  property or undefined. As text: a string itself, a number in JavaScript's
  digits, `true`/`false`, an array's items joined by `", "`, else `""`.
  Truthy: not undefined / null / false / `""` / 0 / an empty array.
- `{% for x in path %} ... {% endfor %}` — `x` bound, `loop` = index from 1,
  first, last, length.
- `{% if path %} ... {% else %} ... {% endif %}`, `{% if not path %}`.
- `{% include "name" %}` — same scope, at most 8 deep; a missing template or
  a deeper include renders nothing.
- A `{% %}` tag alone on its line (only spaces/tabs beside it) takes the
  whole line plus its `\n`.
- Parse errors are reported as `template "<name>": <reason>` and that
  template is left out (unclosed `{{` / `{{{` / `{%`, not a path, unknown
  tag, for/if not closed or closed out of order).

## Front matter, slugs, links, report

- Front matter: `---` opens, the next exact `---` closes. Lines blank
  (spaces/tabs) or `#` comments are skipped; else `key: value`
  (`/^([a-z_]+):(?: (.*))?$/`, one space after the colon). Keys: title, date,
  tags, draft, layout, slug, summary. Errors, first one wins: `no front
  matter`, `the front matter is not closed`, `line <n> is not "key: value"`,
  `unknown key "<k>"`, `key "<k>" is given twice`, `key "<k>" has no value`,
  the key's own reason; then `no title` when there is none.
- A source is refused (reported, dropped) for a front-matter error, for
  living under `tags/` or `page/`, for a post with no date, or for a layout
  that is no template that parses; a `draft: true` source is reported as a
  draft and dropped.
- Slug: NFKD, drop combining marks, A-Z lowered, runs of non-alphanumerics
  to one `-`, none at either end, cut to 64 (a trailing `-` at the cut
  dropped), `untitled` when empty. Unique in its directory in source order
  (first of `s`, `s-2`, `s-3`, ... not taken); at the root, `posts`, `tags`
  and `page` are taken from the start. A page is written to
  `<directory>/<slug>/index.html`; its url is `<base>/<directory>/<slug>/`
  where `<base>` is `site.base_url` with trailing slashes dropped.
- Links: a target with a scheme, or opening on `/` or `#`, is kept as
  written. Else it is resolved from the source's directory (`.` / `..`
  steps, never above the root) with an optional `#` fragment: the url of the
  page built from that source, or `<base>/<path>` of a static file, fragment
  after it. A target naming neither is reported once as
  `broken-link <source> -> <target>` and kept as written.
- Collections: posts newest date first, equal dates by slug; each post's
  `newer` is the one before it, `older` the one after (null at either end).
- Tags: each post's tags, each slug once (first spelling kept), in its own
  order. A tag is all posts whose tags have its slug, named as first met
  walking posts in collection order; url `<base>/tags/<slug>/`. Tags are
  ordered by post count (most first), then slug. A page's tags link to the
  tag of the same slug, or url `""` where no post has that tag.
- Layouts: every page through its layout with `page`; the index through
  `index`, `site.per_page` posts a page (`index.html`, then
  `page/2/index.html` and on; at least one, even with no posts) with `posts`
  and `pager`; each tag through `tag` at `tags/<slug>/index.html`; `tags`
  through `tags/index.html`. An index/tag/tags page whose template is missing
  is not written. Every template also sees `site`, `tags`, `recent`. A
  static file is written at its own path unless a built page has that path
  (reported `shadowed <path>: a built page has its path`).
- A card: `title`, `url`, `date` (string or `""`), `date_long`, `summary`
  (its `summary` key or `""`), `excerpt` (its body's first paragraph's HTML
  or `""`), `reading_minutes` (its body's words — runs of non-whitespace —
  over 200, rounded up, at least 1), `tags` (each `name`, `slug`, `url`). A
  page is a card plus `slug`, `content` (its body's HTML), `toc`, `newer` /
  `older` (each `title`, `url`, `date`, `date_long`, or null; null for a
  page), `is_post`.
- `build-report.txt`: every refusal, draft, broken link and template that
  did not parse, sorted in code-unit order, then
  `built <n> pages, <n> posts, <n> tags and <n> static files`, every line
  ending in `\n`.

## Optimisation levers

- **Markdown inline is the hot path.** A single native regex scan
  (`/[\\`*\[&<>"']/g`) finds the next inline construct or HTML special
  character; plain-text runs between matches are copied in bulk with no
  escaping pass (the regex already excludes every special character).
  The old per-character loop with a per-run `escapeHtml` was ~20% slower.
  (One real bug found by fuzzing: a literal `*` / backtick / `[` left in the
  plain run was pushed twice when the next match re-sliced from the old
  position. Fixed by marking the plain run pushed up to the match before
  handling it. 100,000 random strings now match the reference inline
  exactly.)
- **Fused lex + render.** Markdown blocks are rendered as they are lexed —
  no token array, no second pass. Direct character checks per line, no
  per-line regex battery.
- **Fast paths where the reference pays for generality.** `escapeHtml`
  scans for the first special character and returns the string unchanged
  when there is none. `slugify` has an ASCII fast path (no NFKD /
  `\p{M}` regex); the full reference logic runs for non-ASCII. Front
  matter, dates, tags, layouts and slugs are checked by direct character
  scans, not regexes. `wordsOf` counts word runs with a two-loop scan, no
  regex and no array.
- **Templates parsed once per case; rendering reuses scope objects.**
  A `{% for %}` execution creates one vars object, one `loop` object and
  one child scope, mutated across iterations instead of allocated per
  iteration. All renders share one root scope for `site` / `tags` /
  `recent` instead of spreading them into a fresh object per page.
  Numbers and booleans skip the HTML escape (they render as digits /
  `true` / `false`, which hold no special character).
- **Cards computed once per source** and reused for the post map, the
  index/tag/recent lists and the page object (the reference recomputes a
  post's card twice).
- **No state leaked between cases.** All state is per-build, locals only;
  nothing is cached across `run` calls.

## Measured

Local, Node 22.23.5, this VM, `run()` only over the 3 practice cases
(`bench.mjs`, `process.cpuUsage()` around the 3 `run()` calls):

| entry | median CPU (3 cases) |
|---|---|
| house baseline (makefun-site 1.0.0) | 617,730 µs (11 runs) |
| this entry | 204,763 µs (11 runs) |

Speedup ~3.0x. `grade-local --driver process`: `passed=1`,
`cases_passed=3/3`, L1 regrade reproduced byte for byte.
