# AshGrid

**A platform for running structured self-experiments and reading the results honestly.**

[**Try it → ashgrid.co**](https://ashgrid.co) · iOS (App Store) and web · US, adults 18+

People already experiment on themselves — a new supplement, an earlier caffeine cutoff, ten
minutes of morning light. What's usually missing is the part that makes it an actual
experiment: a fixed starting point, the same questions asked the same way every time, and an
honest read of what changed.

AshGrid records one immutable starting snapshot per experiment, keeps question wording and
scales constant for the entire run, and reports change against that fixed point with the
sample size and uncertainty stated rather than collapsed into a score.

---

<p align="center">
  <img src="screenshots/home.png"    width="24%" alt="Active experiments" />
  <img src="screenshots/checkin.png" width="24%" alt="Daily check-in" />
  <img src="screenshots/result.png"  width="24%" alt="Result against the starting snapshot" />
  <img src="screenshots/stacks.png"  width="24%" alt="Stacks" />
</p>

---

## The engineering problems worth talking about

**Daily self-reports are serially correlated, which makes a naive t-test overconfident.**
Today's sleep score resembles yesterday's, so treating 20 check-ins as 20 independent samples
overstates confidence and manufactures false positives. The statistics engine estimates each
user's own lag-1 autocorrelation, derives an effective sample size
`n_eff = n(1 − r) / (1 + r)`, and recomputes the test at that adjusted `n` and degrees of
freedom. With `r ≤ 0` or too few points it reduces exactly to the classic one-sample test —
the correction only ever shrinks confidence, never inflates it.

**Correcting history without rewriting it.** Check-ins, consent, results, and verdicts are
append-only: a correction is a *new* event referencing the prior one. This isn't a convention
the application layer is trusted to honor — a PostgreSQL trigger raises an exception on any
`UPDATE` or `DELETE` across 32 tables, with a single narrow exception for GDPR/CCPA erasure
that is scoped to one user id and verified per row at runtime.

**Merging duplicate submissions when history can't move.** A community submission starts
tracking immediately on its own private protocol shell, so the user isn't blocked on review.
When moderation later confirms it duplicates a catalog entry, runs can't simply be repointed —
observation events are keyed to the questions actually asked, and moving them would orphan the
history. Merge is therefore a per-run decision: runs whose primary metric the target also
tracks are repointed; the rest stay put, still fully functional, with the reason recorded.

**Telling a real confound from a scheduling accident.** Two overlapping experiments are only
evidence of combination if they genuinely overlapped. Overlap is measured against each run's
own window and classified — sustained (≥80%), partial, or negligible (<20%). Partial overlaps
are excluded from *both* the "together" and "alone" cohorts, because silently demoting an
ambiguous run into "alone" is wrong in the other direction and much harder to notice.

**Offline check-ins that can't double-submit or vanish.** Check-ins written without
connectivity go to a WAL-mode SQLite outbox keyed by an idempotency key, so a retried flush
can't duplicate a submission. Failures are classified before queueing — only network errors,
timeouts, 429s and 5xx are retried; permanent 4xx never are — and one permanently-failing item
can't block the rest of the queue.

## Architecture

```
iOS + Web client                      API                        Data
─────────────────────────             ───────────────────        ──────────────────────
React Native / Expo Router      →     FastAPI (Python)     →     PostgreSQL (Supabase)
TypeScript                            101 REST endpoints         66 tables · RLS on all
TanStack Query · Zustand              Redis rate limiting        Append-only event model
SQLite offline outbox                 Idempotency keys           90 migrations
Apple Health / Health Connect         Deterministic stats        Governed aggregates
```

Two deployed services, deliberately separate: a stateless API that must stay fast, and an
isolated cron process for the maintenance pipeline — a scheduler living inside a web dyno runs
once per instance and silently double-processes the moment it scales past one.

Heavy work never happens in the request path. Statistics, aggregate recomputation, retention,
and outbox drain all run as scheduled jobs.

## Scale

| | |
|---|---|
| REST endpoints | **101** across 14 routers |
| Database tables | **66**, row-level security on every one |
| RLS policies | **69**, split per operation rather than one blanket rule |
| Append-only tables | **32**, immutability enforced by database trigger |
| Migrations | **90**, verified in CI against in-memory Postgres (WASM) |
| Automated tests | **472** backend · **254** frontend, gating every merge |
| Statistical pipelines | **5**, all deterministic — no model decides a number |

## Design principles

**Observations, never medical claims.** AshGrid is not a medical device and does not diagnose,
prescribe, or tell anyone something "works." Every AI-generated string is re-linted against a
banned-claim pattern set *before* it is stored or returned — the model cannot talk its way past
the filter.

**Statistics that admit uncertainty.** Too little data reads "inconclusive" rather than
producing a confident-looking number. Community aggregates publish only once at least 30
distinct people have run the same thing, because an average of three results is noise wearing
the costume of a finding.

**Provenance on every value.** Each reading carries whether it came from the person or their
wearable, and that tag follows it through analysis into the result.

**The record is the product.** Nothing is overwritten. Results are versioned so a better method
never silently rewrites someone's history.

## Tech

**Client** — React Native, Expo Router, TypeScript, TanStack Query, Zustand, SQLite, HealthKit / Health Connect
**API** — Python, FastAPI, Pydantic, NumPy, SciPy, Redis
**Data** — PostgreSQL (Supabase), row-level security, pgvector
**Infra** — Docker (multi-stage, non-root), GitHub Actions CI, static web export on CDN

---

### On source availability

The application source is kept private. AshGrid handles personal health data, and its
authorization model, moderation logic, and safety filters are not things worth publishing a map
of. This repository is a description of the work rather than the work itself.

Happy to walk through architecture, specific tradeoffs, or code in an interview.

**Built by [Ashley Pandey](https://github.com/ashleypandeyxx)** · [ashgrid.co](https://ashgrid.co)
