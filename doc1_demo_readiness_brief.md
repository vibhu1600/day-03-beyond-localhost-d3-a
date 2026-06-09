# Demo Readiness Verification Brief

Student: Vibhuti Uttarde
Assigned incident: D3-A
GitHub branch: day-03-beyond-localhost-d3-a

---

## 1. Readiness Decision

**BLOCK**

The Analyze Resume feature — the entire point of this demo — returns 503 out of the box. Three separate config bugs in `.env` cause this. The recruiter workflow was never tested end to end before this was called ready.

---

## 2. Critical Workflow Declared

Declared before touching anything:

```
Recruiter selects candidate
-> clicks Analyze Resume
-> sees score and signals
-> updates candidate status
-> dashboard reflects the result
```

---

## 3. Before-Fix Evidence

| Evidence Type | Exact Evidence | What It Proved |
|---|---|---|
| Health | `GET /health → 200 { "status": "ok" }` | Backend is alive. Nothing more. |
| Login | `POST /api/login → 200 { token, recruiter }` | Auth works. Unrelated to the workflow. |
| Candidate list | `GET /api/candidates → 200`, 4 records returned | Data loads. Still not the workflow. |
| Analyze Resume | `POST /api/analyze/c_101 → 503 "Resume analysis is disabled in this environment"` | Core feature is off. The recruiter hits a dead end immediately. |
| Backend log | `[WARN] Resume analysis requested but ENABLE_RESUME_ANALYSIS=false` `[ERROR] Resume analysis disabled in current environment` | Confirms the 503 is intentional — a hard gate in routes.js, not a crash. |
| Code — routes.js | `if (process.env.ENABLE_RESUME_ANALYSIS !== "true") → return 503` | `.env.example` ships this as `false`. Copied as-is. Never changed. |
| Code — routes.js (production block) | Checks `AI_PROVIDER` and `AI_SERVICE_KEY` both present, then checks `AI_PROVIDER === "mock"`. Both are empty in `.env.example`. | Even if the flag was fixed, this would be the next failure. Two more blockers stacked behind the first. |
| Code — mockAi.js | `RESUME_STORAGE_PATH || "./tmp/resumes"` → builds path `./tmp/resumes/c_101.txt` | Files actually live at `./data/resumes/`. That directory doesn't exist. Third blocker. |
| Code — candidates.js | Module-level `const candidates = [...]`. No database. No file write. | All score and status changes vanish on server restart. Silent data loss. |

---

## 4. Root Causes

| Symptom | Root Cause | Evidence | Confidence |
|---|---|---|---|
| 503 on every analysis call | `ENABLE_RESUME_ANALYSIS=false` in `.env.example`, never corrected | routes.js first conditional | High |
| Would 500 after fixing flag | `AI_PROVIDER` and `AI_SERVICE_KEY` both empty. Production mode requires both, and `AI_PROVIDER` must equal `"mock"` exactly. | routes.js production block | High |
| Would 500 after fixing provider | `RESUME_STORAGE_PATH=./tmp/resumes` but files are at `./data/resumes` | mockAi.js + repo file tree | High |
| Silent data loss on restart | In-memory JS array, no persistence layer | candidates.js, no DB in package.json | High |

---

## 5. Fixes Applied

| Fix | Why | File touched |
|---|---|---|
| `ENABLE_RESUME_ANALYSIS=true` | Removes the 503 gate | `backend/.env` |
| `AI_PROVIDER=mock` | Only accepted value per routes.js. Hint confirmed: check the route, don't add a new provider. | `backend/.env` |
| `AI_SERVICE_KEY=hiresignal-mock-key` | Just needs to be non-empty. No real API call is made. | `backend/.env` |
| `RESUME_STORAGE_PATH=./data/resumes` | Corrects the path to where the files actually are | `backend/.env` |

No source code was changed. All four fixes are `.env` only.

Fixed `.env`:
```
PORT=5000
NODE_ENV=production
JWT_SECRET=careerforge_day3_secret
ENABLE_RESUME_ANALYSIS=true
AI_PROVIDER=mock
AI_SERVICE_KEY=hiresignal-mock-key
RESUME_STORAGE_PATH=./data/resumes
```

---

## 6. After-Fix Verification

| Check | Result | Evidence |
|---|---|---|
| `/health` | Pass | `200 { status: "ok" }` |
| Login | Pass | `200 { token, recruiter }` |
| Candidate list | Pass | `200`, 4 candidates |
| Analyze Resume | Pass | `POST /api/analyze/c_101 → 200 { candidate: { screeningScore: 78 }, analysis: { score: 78, recommendation: "Needs Review", signals: ["Detected debugging exposure"] } }` |
| Score/signals visible | Pass | `analysis.score`, `recommendation`, `signals[]` all present in response |
| Status update | Pass | `PATCH /api/candidates/c_101/status { "status": "shortlisted" } → 200` |
| Dashboard state | Pass (within session) | Card and summary reflect changes correctly |
| Refresh/reload | Risk | Browser refresh re-fetches from backend. If server was restarted, scores and statuses reset to seed values silently. |

---

## 7. Remaining Risks

- No persistence. Any server restart wipes all recruiter work silently.
- `Dashboard.jsx` code wasn't fully visible in materials. The README itself warns: "if the score does not change after analysis, review the dashboard refresh path." That path needs to be verified in the running UI.
- Only tested on `c_101`. All four candidates should be analyzed before declaring the fix complete.

---

## 8. AI / Founder Assumption Audit

Founder claim: *"Health is green. Login works. Candidate data loads. We're good."*

That's three checks on three different code paths — none of which touch the analyze route. The health endpoint is a single `res.json({ status: "ok" })`. It proves the process is alive. It says nothing about whether `ENABLE_RESUME_ANALYSIS` is set, whether the file path resolves, or whether the AI config is present.

AI used: Claude — to trace code paths across files.
Everything verified against actual lines in the codebase. No claim made without a file reference.

---

## 9. Founder Update

```
Quick update before the demo.

The analyze feature was disabled by default in the config — three env variables
were either wrong or empty. Fixed them, no code changes. Workflow runs end to end now.

One thing to flag: all data is in-memory. If the server restarts during the demo,
analysis scores and status changes reset silently. Don't restart the backend mid-demo.

Recommend we block the demo today and reschedule once we've confirmed the full 
workflow is stable. The fixes are done, but we haven't properly tested all four 
candidates yet and the state risk is real.
```
