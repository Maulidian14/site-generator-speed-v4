[![make.fun · site-generator-speed-01-run-20261122](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/Maulidian14/site-generator-speed-v4/rank.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/Maulidian14/site-generator-speed-v4/reward.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)

# Fast Static Site Generator (v4)

A faster drop-in for makefun-site 1.0.0, built for the make.fun bounty
[Fast Static Site Generator](https://make.fun/bounty/site-generator-speed-01-run-20261122).
Same answers as makefun-site 1.0.0, byte for byte, for every input — only the
work is cheaper. `solve.js` is a dependency-free ES module whose `run(input)`
builds the whole site; `main.ts` is the driver the harness runs.

## What makes it fast

v4 is a fresh, measurement-guided rewrite. Profiling v3 showed the cold-start
cost dominating: in a fresh process the optimising compiler's background
threads and the per-template code generation cost more CPU than the faster
code they produce ever pays back — and every thread's CPU counts. So v4
attacks the cold start itself:

- **Join, don't append.** Every renderer collects its pieces in an array and
  joins once per page, paragraph and template. Appending strings with `+=`
  builds deep rope trees that the collector and the final `JSON.stringify`
  then have to flatten; one join is far cheaper than that flattening. This
  is the single biggest win in v4: it cut the run's CPU by about a third.
- **No code generation.** Templates are rendered by a tree walk over a flat
  binding stack — no `new Function`, no per-template compilation. Loop
  variables ride the stack instead of a fresh scope object per iteration,
  and a for-loop makes one loop object per render and rewrites its fields
  each iteration.
- **Optimising compiler off.** `main.ts` sets `--no-opt` (alongside
  `--no-incremental-marking`, as the house baseline does) before importing
  `solve.js`. Measured locally: about 14% less CPU on the whole run.
- **Word count folded into the Markdown pass.** Every line is counted as it
  is consumed, so the body is never scanned a second time.
- **Index-walked Markdown.** The block pass walks the source by index: no
  `split()` into a line array, no per-line offset table, no per-line slices
  for classification.
- **Regex-driven inline rendering.** The next special character is found with
  one regex exec (a C++ scan) instead of a per-character JavaScript loop,
  and plain runs are copied verbatim between specials.
- **Less allocation churn.** Block output accumulates by string append,
  front matter is scanned in place, and the four render roots are built once
  per site instead of once per page.

## What was measured and rejected

- Rendering directly to the output JSON string (instead of building the
  answers object and calling `JSON.stringify` once) is slower: bulk
  `JSON.stringify` beats every hand-rolled escaper and every per-page
  stringify variant.
- Memoising link resolution is slower than resolving each link: building the
  memo key costs more than the resolution it saves.
- V8's `--single-threaded` and larger semi-spaces help only as command-line
  flags, which the harness does not pass; they have no effect through
  `setFlagsFromString`.
