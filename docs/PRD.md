# PRD — Cairn *(working title)*
### An AI course-decision copilot for Brown's shopping period

**Author:** Ohm Patel · **Status:** Draft v0.1 (pending review) · **Last updated:** 2026-07-12

> Working name "Cairn" (a stacked-stone trail marker) — placeholder, to be validated.

---

## 1. Problem

Brown's open curriculum is a gift and a burden: **~5,400 sections** are offered each
semester with no distribution requirements to narrow the field. During the two-week
**shopping period**, a student samples 5–6 courses and has days to decide which ~4 to
keep. The decision hinges on three things that are *scattered across tools*:

- **Quality** — is the professor good? (the Critical Review)
- **Workload** — how many hours/week will this cost me? (Critical Review)
- **Fit** — does it match my interests, level, and schedule? (Courses@Brown + my own head)

Students bounce between the Critical Review, Courses@Brown, friends' group chats, and a
spreadsheet, then decide under time pressure with incomplete information. There is no
single place that **combines quality + workload + fit** and helps you actually *decide*.

*(What we are NOT solving: requirement/concentration tracking — Brown's **ASK** already
owns that. We deliberately stay out of it.)*

## 2. Target user

**Primary:** a Brown undergraduate actively shopping courses (weighted toward under-
classmen and anyone with open electives — i.e. most of campus, given the open curriculum).

**First pilot user:** the author + classmates (accessible, honest feedback).

## 3. Insight / why now

- The signals already exist but are **siloed**; nobody has combined them into a decision.
- The **workload signal is the hidden gold** — students consistently under-weight it and
  end up overloaded. Surfacing "this is a 5-star course but 14 hrs/week" is uniquely useful.
- LLMs make **conversational discovery + personalized reasoning** cheap for the first time:
  "I liked CSCI 0300 and want something applied but under 8 hrs/week" is now answerable.

## 4. Goals & non-goals

**Goals**
- G1 — Take a student from *choice overload* → *a defensible keep/drop decision* in one session.
- G2 — Make **workload** and **quality** first-class in every recommendation.
- G3 — Weave **discovery** into the decision (surface courses they'd value but didn't know).

**Non-goals (v1)**
- Requirement/degree tracking (ASK owns it).
- Full catalog coverage — a **focused, real subset** is enough to prove the product.
- Registration/enrollment actions — we advise, we don't register.
- A two-sided marketplace / social features (no liquidity to make them work).

## 5. Success metrics

**North star:** *Decision confidence* — % of users who say Cairn helped them decide what to
keep (post-session 1-question survey). Target ≥ 70% of pilot users answer "yes."

**Supporting**
- *Activation:* % of visitors who build a shortlist of ≥ 3 courses. Target ≥ 50%.
- *Depth:* median courses compared per session. Target ≥ 4.
- *AI quality (the eval):* **grounded-reasoning pass rate** — % of AI recommendations whose
  cited facts (rating, hours) match the dataset with zero fabricated numbers. Target ≥ 95%.
- *(Stretch, if timeline allows)* return in a later shopping period.

## 6. Core user flow

```
1. Tell Cairn about you        "I'm a sophomore, like CS + econ, hate heavy reading,
   (chat)                        want to keep it under ~10 hrs/week total."
        ↓  (discovery)
2. Cairn suggests candidates    grounded in real data: surfaces courses that fit,
   → added to your shortlist    each with quality · workload · why-it-fits
        ↓
3. Decision board               side-by-side: rating, prof, workload, meeting time,
   (keep / maybe / drop)         schedule conflicts flagged; you sort + decide
        ↓  (reasoning)
4. Cairn's take                 "Keep CSCI 0300 (strong, 8 hrs) and ECON 1110; DATA 1500
                                 conflicts with your Th afternoon and pushes you to 15 hrs."
```

## 7. Scope

**MVP (v1) — build now**
- Chat-based discovery over the real 113-course dataset.
- Decision board: add/compare/keep-drop, showing rating · prof · workload · meets.
- **Schedule-conflict detection** from meeting times (deterministic, no AI needed).
- AI recommendation that is **grounded** (only cites facts present in the dataset).
- Web app, works from the hosted demo; a "sample" dataset for the public version.

**v1.1 — after pilot feedback**
- Expand dataset (the CAB API + CR data pipeline already proven — can scale to full catalog).
- "Total workload" budget across the shortlist; balance warnings.

**Later**
- Personalization from courses-taken; save/return across sessions.

## 8. Key product decisions & tradeoffs

| Decision | Choice | Why |
|---|---|---|
| Job | Shopping-period *decision* (not requirement tracking) | ASK owns requirements; the decision is the unmet gap |
| Interaction | Hybrid chat → decision board | Discovery needs chat; deciding needs a comparable view |
| Data | Real, focused subset (113 courses) | Real enough to be useful + pilotable; sidesteps full-scrape ToS |
| AI role | **Reason over data, never invent facts** | Trust: a course tool that hallucinates a rating is worse than nothing |
| Schedule conflicts | Deterministic code, not the LLM | LLMs are unreliable at time math; this must be exact |

## 9. AI design + evaluation *(the part that makes this a real AI product)*

- **Grounding:** the model is given the structured course rows and instructed to cite only
  those facts (rating, hours, meeting time). Any number it states is checked against the row.
- **Guardrail:** a post-check flags or drops any AI claim whose numbers don't match the data.
- **Eval set:** ~15 student scenarios (e.g. "light workload, quantitative, mornings free")
  with a rubric — did it recommend in-dataset courses, respect the constraint (workload/
  schedule), and cite correct numbers? Report the grounded-reasoning pass rate as a metric.
- **Failure modes handled:** no matching course → says so; conflicting constraints → surfaces
  the tradeoff instead of guessing.

## 10. Data

- **Source:** Critical Review (quality/workload) ⋈ Courses@Brown (Spring 2026 offerings +
  meeting times), joined on course code. Collected via the author's authenticated session,
  modest focused subset.
- **Shape (per course):** code, title, professor, meets, course_rating, prof_rating,
  avg_hours, class_size, reviewed_edition, dept.
- **Rights:** real Critical Review–derived data stays **local/gitignored**; the public repo
  and hosted demo ship a **derived/sample** dataset, clearly labeled. (Data-licensing
  restraint is itself part of the product story.)

## 11. Risks

| Risk | Mitigation |
|---|---|
| AI fabricates a rating → destroys trust | Grounding + deterministic post-check (the eval gate) |
| Small dataset feels thin to a real user | Curated across 19 depts; honest "not in dataset yet" messaging |
| Uncertain pilot access | Product must be valuable at N=1 (no marketplace); metrics work with a small pilot |
| Data freshness / ToS | Focused subset, local-only real data, sample for public |

## 12. Rollout / pilot

1. Ship the MVP with the real dataset (local) + sample (public/hosted).
2. Pilot with 10–30 classmates during a shopping-adjacent window; capture the north-star
   survey + activation/depth from lightweight analytics.
3. Iterate on the top failure the pilot surfaces; write it up as the PM case study.
