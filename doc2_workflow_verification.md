# Workflow Verification Evidence

Student: [Your Name]
Variant ID: D3-A

---

## Critical Workflow Declared Before Fixes

```
Recruiter selects candidate
-> clicks Analyze Resume
-> sees score and signals
-> updates candidate status
-> dashboard reflects the result
```

---

## Before-Fix Evidence

| Step | Expected Behavior | Actual Behavior | Evidence |
|---|---|---|---|
| Select candidate | 4 candidate cards load with name, role, score, status | Works — cards load correctly | `GET /api/candidates → 200`, 4 records |
| Analyze resume | Click button → score + signals appear | FAILS — `503 "Resume analysis is disabled in this environment"` | Backend log: `[WARN] ENABLE_RESUME_ANALYSIS=false` |
| Review score/signals | Score and signals shown on card | Not reachable — blocked by 503 | N/A |
| Update status | Dropdown change → card updates | API works but state is in-memory — server restart wipes it | `PATCH /status → 200`, but candidates.js has no DB |
| Dashboard reflects result | Summary counts update | Works within session only — no persistence | In-memory array confirmed in candidates.js |

---

## After-Fix Verification

*Fixes applied: `ENABLE_RESUME_ANALYSIS=true`, `AI_PROVIDER=mock`, `AI_SERVICE_KEY=hiresignal-mock-key`, `RESUME_STORAGE_PATH=./data/resumes`*

| Step | Passed? | Evidence |
|---|---|---|
| Select candidate | Yes | `GET /api/candidates → 200`, 4 candidates load |
| Analyze resume | Yes | `POST /api/analyze/c_101 → 200 { screeningScore: 78, recommendation: "Needs Review", signals: ["Detected debugging exposure"] }` |
| Review score/signals | Yes | `analysis.score`, `analysis.recommendation`, `analysis.signals[]` all returned and populated |
| Update status | Yes | `PATCH /api/candidates/c_101/status { "status": "shortlisted" } → 200`, card reflects change |
| Dashboard reflects result | Yes (within session) | Summary and card state correct — but resets on server restart |

---

## Remaining Risk

- State is in-memory. Server restart = all scores and statuses silently reset to defaults. Recruiter sees original seed data with no warning.
- `Dashboard.jsx` score refresh behaviour needs live UI confirmation — not just API verification.
- All four candidates (c_101–c_104) should be tested individually before the pilot, not just c_101.
