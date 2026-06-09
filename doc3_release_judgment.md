# Release Judgment Decision

Student: Vibhuti Uttarde
Variant ID: D3-A

Final decision:

- [ ] Ship
- [x] **Block**
- [ ] Conditional

---

## Decision Evidence

| Evidence | Supports | Why It Matters |
|---|---|---|
| `ENABLE_RESUME_ANALYSIS=false` in `.env.example` — 503 on every analysis call | Block | The recruiter's only job in this demo is to analyze resumes. That feature was off by default. Setup guide says copy the example and run — no mention of editing it. |
| `AI_PROVIDER` and `AI_SERVICE_KEY` both empty — would 500 in production even after fixing the flag | Block | Three cascading blockers. You can't find the second and third until the first is fixed. This wasn't a single oversight — the entire env config was never set up for the workflow. |
| `RESUME_STORAGE_PATH=./tmp/resumes` — directory doesn't exist | Block | The fix is one line. But shipping a config that points to a nonexistent path means the feature was never run end to end in this environment. |
| All fixes are `.env` only — no code defects | Block (not permanent) | Fixes are trivial, but that doesn't mean the product was ready. It means it wasn't verified. The pre-fix state is what we're judging. |
| In-memory state — restart resets all recruiter work silently | Block | No database. No warning on data loss. A server hiccup mid-demo and the recruiter sees original scores again with no explanation. |

---

## Blockers

1. `.env` was never configured for the workflow. Default config breaks analysis at three points.
2. No persistence — data loss on restart is silent and unrecoverable without re-running analysis.
3. Full workflow was not verified across all four candidates before declaring readiness.

---

## Conditions If Proceeding

If the team insists on proceeding conditionally:

1. `.env` must be corrected before starting — `ENABLE_RESUME_ANALYSIS=true`, `AI_PROVIDER=mock`, `AI_SERVICE_KEY=<non-empty>`, `RESUME_STORAGE_PATH=./data/resumes`
2. All four candidates analyzed in a pre-demo dry run
3. Server must not be restarted during the demo
4. Pilot recruiter is told upfront: scores and status are session-only in this prototype

---

## Final Release Defense

The product failed the one workflow that defines its value.

Health check passed. Login passed. Candidates loaded. None of that touches the analyze route. The founder used liveness as a proxy for readiness — that's the exact trap Day 3 is built around.

Three config bugs in `.env`, all stacked on the same code path, all pointing to the same conclusion: the analyze workflow was never run in this environment before being called demo-ready. The fixes are simple. That's not a reason to ship — it's evidence that verification didn't happen.

Block. Fix the config. Run the full workflow on all four candidates. Then reschedule.
