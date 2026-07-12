# Cairn — a Brown course-decision copilot

*Working title.* Turn Brown's overwhelming shopping period into a decision: describe what
you want, get **grounded** course suggestions (quality · workload · fit), then compare them
on a board that flags **schedule conflicts** and **total workload** — so you keep the right
four courses, not the first four you tried.

**Why:** Brown's open curriculum offers ~5,400 sections with no requirements to narrow them.
The signals that actually drive the decision — professor quality, workload, schedule fit —
are scattered across the Critical Review, Courses@Brown, and group chats. Cairn combines them.

See [`docs/PRD.md`](docs/PRD.md) for the full product spec (problem, user, metrics, AI eval plan).

## Status — MVP (working)

- ✅ **Discovery**: describe your constraints → ranked, **grounded** suggestions (never invents a rating)
- ✅ **Decision board**: keep / maybe / drop, with rating · workload · meeting time
- ✅ **Schedule-conflict detection** — exact, computed from meeting times (not guessed)
- ✅ **Total-workload** tally with a heavy-load warning
- ⏳ Next: natural-language chat via an LLM (deterministic ranker works today, keyless);
  pilot with classmates; expand beyond the current 120-course subset

## Run it

```bash
python3 -m http.server 8291   # then open http://localhost:8291
```

The app loads `data/courses.local.json` (real data, local only) if present, else falls back
to `data/courses.sample.json` (illustrative). It's a single static file — no build step.

## Data & rights

Ratings/workload are from **The Critical Review**; offerings/times from **Courses@Brown**.
The real Critical Review data is **local-only and gitignored** — the public repo/demo ships
a sample with **synthetic ratings** (real, public course codes/titles/times). See
[`data/README.md`](data/README.md) for the full provenance and the deliberately conservative
data-rights posture.
