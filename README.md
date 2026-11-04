# Cairn — a Brown course-decision copilot

*Working title.* Turn Brown's overwhelming shopping period into a decision: describe what
you want, get **grounded** course suggestions (quality · workload · fit), then compare them
on a board that flags **schedule conflicts** and **total workload** — so you keep the right
four courses, not the first four you tried.

**Why:** Brown's open curriculum offers ~5,600 sections with no requirements to narrow them.
The signals that actually drive the decision — professor quality, workload, schedule fit —
are scattered across the Critical Review, Courses@Brown, and group chats. Cairn combines them.

See [`docs/PRD.md`](docs/PRD.md) for the full product spec (problem, user, metrics, AI eval plan).

## Status — MVP (working)

- ✅ **Full catalog**: the demo loads all **5,628 sections** of Brown's Fall 2026 catalog
  (113 subjects, 1,760 distinct courses) from the public Courses@Brown catalog, pulled 3 Nov 2026
- ✅ **Two-stage funnel**: a deterministic prefilter narrows those 5,628 sections to at most
  **15 candidates**, and only then does the ranking stage cut to at most **5**. Both bounds are
  named constants (`PREFILTER_K`, `FINAL_K`) at the top of the script
- ✅ **Discovery**: describe your constraints → ranked, **grounded** suggestions (never invents a rating)
- ✅ **Decision board**: keep / maybe / drop, with rating · workload · meeting time
- ✅ **Schedule-conflict detection** — exact, computed from meeting times (not guessed)
- ✅ **Total-workload** tally with a heavy-load warning
- ⏳ Next: natural-language chat via an LLM (deterministic ranker works today, keyless);
  pilot with classmates

## The funnel

The prefilter is a pure function of (catalog, parsed query). No model, no network, no
randomness, and ties break on course code so the same query always returns the same list.
It applies the hard constraints (workload cap, day avoidance, course level, interest match),
collapses multiple sections of one course to a single representative, orders what is left,
and cuts to `PREFILTER_K`. Only then does ranking run, capped at `FINAL_K`.

In the gated pilot an LLM orders the candidate set and the server re-attaches the
authoritative ratings afterwards, so a model can never emit a number that is not in the
data. This keyless public build runs the identical funnel with the local scorer standing in
for the model. The live demo prints the counts for every search, so the narrowing is
visible rather than asserted.

## Run it

```bash
python3 -m http.server 8291   # then open http://localhost:8291
```

The app loads `data/catalog.json` (the full public Fall 2026 catalog, committed) as its base
layer, then overlays ratings and workload from `data/courses.local.json` (real Critical Review
data, local only) if present, else from `data/courses.sample.json` (illustrative). Courses with
no review data show `—` rather than an invented number. It's a single static file, no build step.

## Data & rights

Ratings/workload are from **The Critical Review**; offerings/times from **Courses@Brown**.
The real Critical Review data is **local-only and gitignored** — the public repo/demo ships
a sample with **synthetic ratings** (real, public course codes/titles/times). See
[`data/README.md`](data/README.md) for the full provenance and the deliberately conservative
data-rights posture.
