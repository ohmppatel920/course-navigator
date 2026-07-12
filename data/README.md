# Data — provenance & rights

## Files

| File | Contents | Committed? |
|------|----------|:---:|
| `courses.real.json` | 120 real courses (Spring 2026 offerings ⋈ Critical Review quality/workload) | **No — gitignored** |
| `cr_catalog_raw.json` | Raw Critical Review recent-review pull | **No — gitignored** |
| `courses.sample.json` | Public demo data: real (public) course codes/titles/times + **synthetic** ratings | Yes |

## Sources

- **Courses@Brown** (cab.brown.edu) — Brown's **public** course catalog. Course codes,
  titles, and meeting times come from here. Freely usable.
- **The Critical Review** (thecriticalreview.org) — Brown's student-run course-review
  publication. Course/professor ratings and **workload (hrs/week)** come from here.

## Data-rights posture (deliberately conservative)

The Critical Review has no published API, terms-of-use page, or `robots.txt` restriction,
and its stated purpose is to *help Brown students choose courses* — exactly this use. Even
so, to avoid overstepping:

1. **No public redistribution of Critical Review data.** The real ratings/workload live
   only in the gitignored `*.real.json` / `cr_catalog_raw.json`, used locally for the
   author's own course decisions and the pilot. The public demo ships
   `courses.sample.json`, whose **ratings are synthetic** (real course codes/titles/times
   are public CAB data).
2. **Attribution.** The app credits The Critical Review as the ratings source.
3. **Minimal collection.** Data was gathered via normal page loads in the author's own
   authenticated Brown session — a focused CS + Engineering subset — not a site-wide crawl.

## Refreshing the real data (local only)

Re-run the collection join against a fresh Critical Review pull + the CAB API for the target
term, then `courses.real.json` is regenerated. Keep it gitignored.
