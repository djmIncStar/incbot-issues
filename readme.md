# InCBot Video System — Three Improvement Proposals

**Service:** `incbot-video-system` (the backend that runs the AI interview)
**Scope:** (1) Browser access rules (CORS), (2) faster interview reports, (3) cheaper "candidate is online" tracking
**Status:** Proposal, for review by Engineering, Product and Leadership

---

## Summary

| # | Issue | In one line | Who feels it | Effort | Priority |
|---|---|---|---|---|---|
| 1 | **CORS and access control** | Our rules on which websites may talk to the server are inconsistent, and one of them can be fooled by a look-alike domain. A related, more serious gap: some endpoints and all live-interview channels don't check who is calling. | Security, compliance, customer trust | Small (CORS fix: ~1 day). Larger for the auth follow-up | **High** |
| 2 | **Report waits a fixed 6 minutes** | Every report waits 6 minutes after the interview ends, even when all the work finished in seconds, and the employer sees "In Progress" the whole time. | Employers (main customer), PM, Sales | Medium (~1–2 weeks incl. measurement) | **High** |
| 3 | **"Online" heartbeat written to the database** | Every live interview writes to the main database about 8 times a minute just to say "still here", while a dropped candidate is only flagged about 1 hour later. | Infra cost, database stability at scale | Medium (~1 week; touches a shared library) | **Medium** |

Glossary for non-engineers:
- **Server / backend:** the program on our cloud machines that the interview page talks to.
- **Database (MongoDB):** permanent storage for interviews and reports. Reliable, but each write has a cost.
- **Redis:** a very fast, in-memory store we already run. Good for short-lived, frequently changing data.
- **Queue / job:** a to-do list for slow background work (scoring answers, building reports).
- **CORS:** a browser safety rule. Our server tells the browser which websites may read its responses.

---

# Issue 1: CORS and access control

### Plain-English summary
Imagine a building with two receptionists who follow different guest lists. One waves everyone through, the other checks names. The live-interview entrance has a third list that would admit "evil-incruiter.com" because it ends in "incruiter.com". Separately, some doors have no lock at all for visitors who don't come through a browser.

### How it works today
```
Browser request ─► [Tenant lookup] ─► [CORS #1: "allow everyone (*)"] ─► [CORS #2: "allow *.incruiter.com only"] ─► route
Live-interview socket ─► [CORS #3: separate, looser pattern] ─► socket events (no identity check)
```

### Problem statement
1. **Two contradictory CORS layers on the HTTP API.**
   - [src/server.ts:323-327](incbot-video-system/src/server.ts#L323-L327) applies `cors({ origin: "*" })` to **every** request.
   - The strict allowlist (`CORS_PERMISSIONS` → [node-engine/middlewares/corsMiddleware.ts](incbot-video-system/src/node-engine/middlewares/corsMiddleware.ts)) is then added per route group.
   - The first layer answers every browser "pre-flight" check with "anyone allowed" before the strict layer runs. `/api-docs` and unknown routes only ever get the "anyone" rule.
   - The real rule is therefore hard to work out, and the next developer can easily make it worse.

2. **Blocked websites get "500 Internal Server Error".** When the strict layer rejects an origin, the error reaches [errorHandler.ts](incbot-video-system/src/common/middleware/errorHandler.ts), which always returns 500. In monitoring, a blocked or misconfigured caller looks like a server crash. That hides real outages and creates noise.

3. **The live-interview channel (Socket.IO) has its own, weaker allowlist.**
   - [src/websocket.ts:63](incbot-video-system/src/websocket.ts#L63) uses the pattern `/\.?incruiter\.com$/`, which isn't anchored to the start of the origin and doesn't require `https`.
   - Tested: `https://evilincruiter.com` → **allowed**, `http://incruiter.com` (unencrypted) → **allowed**.
   - The HTTP list in node-engine is correct, so the two lists have already drifted apart.

4. **Error responses can lose their CORS headers.**
   - `tenantMiddleware` runs **before** CORS ([server.ts:321](incbot-video-system/src/server.ts#L321)).
   - If the tenant lookup fails, the response has no CORS headers, so the browser shows a generic "CORS error" instead of the real one.
   - This wastes debugging time for frontend developers and support.

5. **The bigger, related risk: CORS isn't authentication.** CORS only restrains *browsers*. A script or `curl` sends no `Origin` header, which is explicitly allowed (`corsMiddleware.ts`: "Allow requests with no origin"), or it can fake one. So these places need their own identity checks, and today they have none:

   | Entry point | What anyone with a `roomId` can do today | Evidence |
   |---|---|---|
   | `GET /incbot/getTokenBeam/:roomId/:mode` | Get a LiveKit token with `canSubscribe: true`, i.e. **join and watch/listen to a live interview** | [interviewRouter.ts:629-633](incbot-video-system/src/api/incbot/interviewRouter.ts#L629-L633) has no permission middleware; grant at [interviewService.ts:3275-3281](incbot-video-system/src/api/incbot/services/interviewService.ts#L3275-L3281) |
   | Socket `joinRoom` | Receive live transcripts (`stt:final`), candidate code (`receiveCode`) and AI voice audio for that room | [websocket.ts:456](incbot-video-system/src/websocket.ts#L456): no check on who joins |
   | Socket `stt:start` / `stt:audio` | Use our paid speech-to-text minutes | [websocket.ts:607-660](incbot-video-system/src/websocket.ts#L607-L660) |
   | `POST /incbot/test-merge-recording/:roomId` | Trigger a heavy video-merge job (CPU + storage cost) | [interviewRouter.ts](incbot-video-system/src/api/incbot/interviewRouter.ts) (end of file), marked "TEST" |
   | `GET /getInterviewInfo/:roomId`, `GET /getInterviewToken/...` | Read interview details | No permission middleware on the route |

   A `roomId` is a MongoDB ObjectId. It's partly time-based and appears in URLs and API paths, so it shouldn't be treated as a secret password.

### Proposed solution
**Phase A: fix CORS (small, low risk)**
1. **One allowlist, defined once.** Export a single list from node-engine and use it for both HTTP and Socket.IO. Anchor it and require HTTPS: `^https://([a-z0-9-]+\.)*incruiter\.com$`, plus `http://localhost:<port>` for local development only.
2. **One CORS layer.** Delete the global `cors({ origin: "*" })` in `server.ts` and apply the strict middleware once, globally, **before** `tenantMiddleware`, so every response (including errors) carries correct headers.
3. **Return `403 Forbidden: origin not allowed`** for rejected origins instead of 500, and log the rejected origin.
4. **Roll out in "report-only" mode first.** For about one week, log what *would* be blocked without blocking it. This catches any legitimate origin we forgot (customer white-label domains, staging hosts, the recording "beam", mobile webviews) before it breaks.

**Phase B: add real identity checks (separate tickets, higher value)**
1. **Socket handshake authentication.** The client sends its interview token in `socket.handshake.auth`. The server verifies it and only allows `joinRoom` for the room that token belongs to.
2. **Protect `getTokenBeam`** with a service-to-service secret or a signed, short-lived beam token. Only our recorder should be able to get one.
3. **Remove `/test-merge-recording`** from production builds, and add permission middleware to `getInterviewInfo` / `getInterviewToken`, or confirm they return only non-sensitive data.
4. *(Related finding, outside this proposal.)* The tenant is chosen from the client-supplied `x-tenant-id` header ([TenantMiddleware.ts:7](incbot-video-system/src/node-engine/middlewares/TenantMiddleware.ts#L7)), and the auth middleware doesn't check that the logged-in user belongs to that tenant. This should get its own review.

### Expected impact
| Area | Today | After |
|---|---|---|
| Look-alike domains on live channel | Allowed (`evilincruiter.com`, plain `http`) | Blocked |
| Rules developers must reason about | 3 lists, 2 HTTP layers | 1 list, 1 layer |
| Blocked request in monitoring | Shows as **500 server error** | Shows as **403**, with the origin logged |
| Frontend debugging of tenant errors | Hidden behind "CORS error" | Real error visible |
| Live interview privacy (Phase B) | Anyone with a `roomId` can watch/listen | Only the candidate, our bot and our recorder |
| User-visible change for legitimate users | — | None (verified by report-only week) |

### Risks and mitigations
- **A legitimate origin gets blocked.** Mitigated by the report-only week.
- **Moving the Host-header check to all routes affects health probes.** `corsMiddleware` also rejects requests whose `Host` isn't `*.incruiter.com`. Confirm how the Kubernetes probes call `/health-check` before making it global. If they call the pod IP directly, exempt the health routes.

### Open questions
- Which non-`incruiter.com` origins legitimately call us (white-label customers, staging, beam)?
- What `Host` header do the Kubernetes probes and internal services (LiveKit agent, interview-system cron) send?

---

# Issue 2: The report waits a fixed 6 minutes after every interview

### Plain-English summary
When an interview ends, we start a **6-minute timer** and only then begin building the report, just in case some answers are still being processed. Most of the time everything was ready long before. The employer waits for nothing, and the dashboard shows "In Progress" for an interview that has already finished.

### How it works today
```
Candidate clicks End
   │
   ▼
POST /callEnd ─► status set to "In Progress" ─► schedule report job  ⏱ wait 6:00 (fixed)
                                                         │
                                                         ▼
                                    "Are any answers still being scored?"  (Redis counter)
                                       ├─ yes → wait 30 s more, ask again (up to +3:30)
                                       └─ no  → build report: transcribe full recording,
                                                AI scoring, deepfake & AI-text checks
                                                → final status → email employer
```
- 6-minute delay: [callEndService.ts:327](incbot-video-system/src/api/incbot/services/callEndService.ts#L327) (`const delay = 360000`)
- Status set to "In Progress" at call end: [callEndService.ts:314-318](incbot-video-system/src/api/incbot/services/callEndService.ts#L314-L318)
- "Still scoring?" check and 30 s re-checks (max 210 s): [callEndService.ts:28-35](incbot-video-system/src/api/incbot/services/callEndService.ts#L28-L35), [L548-605](incbot-video-system/src/api/incbot/services/callEndService.ts#L548-L605)
- Counter: [src/pendingEvaluations.ts](incbot-video-system/src/pendingEvaluations.ts). It goes up when an answer is queued ([transcribeService.ts:1358](incbot-video-system/src/api/incbot/services/transcribeService.ts#L1358), [codingService.ts:169](incbot-video-system/src/api/incbot/services/codingService.ts#L169)) and down when its job succeeds ([queueRegistry.ts](incbot-video-system/src/queueRegistry.ts)).
- Report builder: `skillsMetricV2` ([callEndService.ts:2007](incbot-video-system/src/api/incbot/services/callEndService.ts#L2007))

### Problem statement
1. **Every report pays at least 6 minutes of idle waiting**, and up to **9.5 minutes** before report work even begins. Report processing time comes on top of that.
2. **The wait is a guess, not a check.** The code already has a precise "is anything still being scored?" check (the counter). The fixed 6 minutes is a safety margin for things the counter **cannot see**:
   - **(a) Late uploads:** the last answer's audio may still be uploading from the candidate's browser when End is clicked. The counter only increases once that upload *arrives*, so for a moment it reads 0 even though an answer is coming.
   - **(b) The full recording:** AI-conversation interviews transcribe the whole call recording (`response.recording`, [callEndService.ts:2124](incbot-video-system/src/api/incbot/services/callEndService.ts#L2124)). Nothing tracks whether it has been uploaded yet.

   Because these two signals are missing, the timer has to be long enough for the *worst* case, and every interview pays it.
3. **Employers see a misleading status.** The interview shows "In Progress" for 6–10+ minutes after the candidate left. That drives "is it stuck?" support tickets and lowers trust in the product.
4. **The guess can still be wrong.** If an upload or recording takes longer than the window, the report is built without it. The code logs this case ("scoring this interview with evaluations still outstanding — the report may be missing a coding section, an MCQ…", [callEndService.ts:596-603](incbot-video-system/src/api/incbot/services/callEndService.ts#L596-L603)). A longer timer makes this rarer; it doesn't make it impossible.

### Proposed solution: "start the report when it's ready, not when the timer says so"
Replace the guess with an explicit **readiness check**, triggered by events, and keep a timer only as a **safety net**.

**Definition of "ready to build the report"** (all must be true):
1. The interview has ended (`interviewEndDate` set by `callEnd`).
2. **All expected answers have arrived.** At `callEnd`, the frontend sends how many answer uploads it started (`expectedAnswers`). The server compares that with how many it received. This closes gap (a).
3. **No answers are still being scored** (existing counter = 0).
4. **For AI-conversation interviews: the recording URL exists.** This closes gap (b).

**How the trigger works**
```
callEnd ──► schedule SAFETY-NET report job (e.g. 10 min, same as today's worst case)
             + run readiness check now

Each of these events re-runs the readiness check:
   • an answer finishes scoring (counter decremented)
   • an answer upload arrives
   • the recording URL is saved

Readiness check passes ──► promote the safety-net job to "run now" (BullMQ job.promote())
Safety net fires first   ──► build the report anyway, log exactly what was missing (as today)
```
The safety-net job uses the existing fixed ID `callEndDelay-<roomId>`. Promoting it means there is still only **one** report job per interview, so duplicates are impossible. A "report generated" marker in Redis or MongoDB guards against re-runs after the job is removed. The code already uses this pattern for the report email ([callEndService.ts:37-60](incbot-video-system/src/api/incbot/services/callEndService.ts#L37-L60)).

**Status shown to the employer:** set a distinct status such as **"Generating report"** at `callEnd` instead of "In Progress", so the dashboard tells the truth while the report is built. This needs a PM decision plus a small dashboard change in interview-system.

**Rollout plan (low risk, data first)**
1. **Measure (2–3 days).** Add logs for:
   - the time from `callEnd` to the last answer upload,
   - the time from `callEnd` to the counter reaching 0,
   - the time from `callEnd` to the recording URL being saved,
   - report build duration.

   This tells us the real p50/p95/p99 instead of guessing.
2. **Make the delay configurable** (`REPORT_START_DELAY_MS`, default stays 6 min). This is a safe change that can be reverted without a deploy.
3. **Ship the readiness check behind a flag in "shadow mode":** log *when the report would have started* without changing behaviour. Compare with step 1.
4. **Turn it on** for internal and practice interviews, then a few customers, then everyone.

### Expected impact
| Metric | Today | After (expected, to be confirmed by step 1) |
|---|---|---|
| Idle wait before report work starts | **6:00 minimum**, up to 9:30 | Typically **under 1 minute** (bounded by the slowest answer or recording upload) |
| Employer sees an accurate status after the interview | No ("In Progress" for 6–10+ min) | Yes ("Generating report" → "Completed") |
| Reports built with missing sections | Possible when uploads are slower than the window, logged | Rarer: we wait for the actual signals, and the safety net still covers the worst case |
| Compute cost | — | Unchanged (same work, started earlier) |
| Support tickets like "report not ready / stuck" | Baseline | Expected to drop (measure before/after) |

**Business value:** the report is what the employer pays for. Delivering it 5+ minutes sooner after every interview makes the product feel instant, and helps in sales demos and high-volume hiring drives where recruiters review candidates back to back.

### Risks and mitigations
- **The frontend can't report `expectedAnswers` reliably** (tab closed, crash). Fall back to comparing the question list with the answers stored in `TranscriptBank`, and keep the safety net.
- **Counter drift if Redis has a hiccup.** The safety net still fires, so the worst case is today's behaviour, never worse.
- **Recording uploaded by another service.** Who writes `response.recording` for AI-conversation interviews needs confirming (see open questions). That writer must trigger the readiness check, or the check must poll briefly.

### Open questions
- Which component saves the full recording URL for AI-conversation interviews, and how long after call end does it usually arrive?
- PM: should employers see a "Generating report" status? Does the dashboard need changes in interview-system?
- What is the acceptable worst-case report time (the safety-net timer)?

---

# Issue 3: Store "candidate is online" in Redis, not the main database

### Plain-English summary
During an interview, the candidate's browser keeps saying "I'm still here" every 30 seconds, from **four places at once**. Each message is written to the main database. Meanwhile, the background check that uses this data only runs every 10 minutes and only flags a dropped candidate after an **hour** of silence. We pay for a database write every few seconds to answer a question we only ask every 10 minutes.

### How it works today
```
Candidate browser (4 emitters × every 30 s)
      │  socket event "interview-active"
      ▼
incbot-video-system ──► MongoDB: InCBotInterview.interviewActiveTimestamp = now
                                   ▲
interview-system cron (every 10 min, node-engine CronHelper.incbotNetworkIssue)
   reads ALL "In Progress" interviews, flags "Network Disconnected" if
   (silent > 1 h  OR  started > 2 h ago  OR  attempt > 2)  AND  silent > 2 min
```
- Writer: [websocket.ts:319-454](incbot-video-system/src/websocket.ts#L319-L454) (`interview-active` handler; the code comment documents the four emitters firing every 30 s)
- Reader 1, the disconnect cron: [node-engine/helpers/CronHelper.ts:1175-1255](incbot-video-system/src/node-engine/helpers/CronHelper.ts#L1175-L1255), which reschedules itself every 10 min
- Reader 2, the "Interview Ended" timestamp at call end: [callEndService.ts:400-430](incbot-video-system/src/api/incbot/services/callEndService.ts#L400-L430), [proctoringEndTimestamp.ts](incbot-video-system/src/api/incbot/services/proctoringEndTimestamp.ts)

### Problem statement
1. **Hottest database write in the product.**
   - Each live interview writes **~8 times per minute**, about **360 writes per 45-minute interview**.
   - At 1,000 concurrent interviews that's **~130 writes/second** to the busiest collection, purely to record "still here".
   - The team already moved this write off Mongoose to cut its cost (INC-1615, documented in the handler). That helped, but each write still touches a large (~20 KB) interview document and its indexes, in the per-tenant database.
2. **4× duplication.** Four emitters send the same heartbeat, and the server writes all of them.
3. **Every heartbeat sets up a database context first** (`applyTenantContext(tenantId)`) using a `tenantId` the browser supplies.
4. **Slow detection despite the frequent writes.**
   - The cron runs every 10 minutes, and its rule needs **1 hour** of silence (unless the interview is over 2 h old or on attempt 3+).
   - A candidate whose connection dies is typically flagged "Network Disconnected" **60–70 minutes later**.
   - The data is updated every 30 s, but the decision is made at most every 10 minutes.
5. **The cron loads every in-progress interview** in full (with the creator populated) every 10 minutes to check one timestamp field.

Note: point 4 is partly a **product rule** (the 1-hour grace). This proposal makes faster detection *possible*. Whether to use it is a PM decision.

### Proposed solution
**Presence lives in Redis; MongoDB only records meaningful changes.**

```
Candidate heartbeat ──► Redis: ZADD incbot:presence <now> "<tenantId>:<roomId>"     (microseconds, no DB)
                        (dedupe: ignore if this room already wrote in the last 25 s)

MongoDB is written only when something changes:
   • interview starts / resumes        (already done by callStart)
   • socket disconnects                (new: we know instantly)
   • interview ends                    (callEnd copies the last-seen time from Redis)
   • backstop: at most once every 5 min, so data survives a Redis loss

Disconnect cron ──► ZRANGEBYSCORE incbot:presence 0 (now − threshold)
                    → only the stale rooms, then check those few interviews in MongoDB
```

**Details for engineers**
1. **Writer** ([websocket.ts](incbot-video-system/src/websocket.ts) `interview-active`):
   - Validate `roomId` (keep the existing guards).
   - `ZADD` with the server time.
   - Dedupe with `SET presence:lock:<roomId> NX EX 25`.
   - Write MongoDB only if the last persisted value is older than 5 minutes.
2. **Readers:**
   - `CronHelper.incbotNetworkIssue` (node-engine, shared with interview-system) reads Redis first. It falls back to the MongoDB timestamp per room if Redis has no entry.
   - `callEndFn` / `resolveEndRowTimestamp` read last-seen from Redis, falling back to MongoDB.
3. **Clean-up:** `ZREM` on `callEnd`. Trim old entries periodically, as a safety net for interviews that never end.
4. **Safety rule (critical):** if Redis is **unavailable**, the cron must **not** flag anyone. "No data" must never be read as "everyone disconnected". It should fall back to the 5-minute MongoDB backstop.
5. **Faster detection (optional, PM decision):** the socket `disconnect` event already tells us the moment a candidate drops. The cron can run every 1–2 minutes cheaply, because it now reads one Redis key instead of every in-progress interview. The 1-hour grace can then be tuned separately.

### Expected impact
| Metric | Today | After |
|---|---|---|
| DB writes per 45-min interview for presence | ~360 | **~10** (start, end, disconnects, 5-min backstop) — **~97% fewer** |
| DB writes/sec at 1,000 concurrent interviews | ~130 | **~3–4** |
| Duplicate heartbeats processed | 4× | 1× (deduplicated) |
| Disconnect cron work per run | Loads every in-progress interview | One Redis query + only the stale interviews |
| Time to detect a dropped candidate | ~60–70 min (current rule) | Can be cut to minutes if Product wants; unchanged otherwise |
| Cost | — | Lower database load: room to grow without upgrading the MongoDB tier; Redis cost is negligible (one small entry per live interview) |

### Risks and mitigations
- **Redis outage** → the cron's fallback rule (never flag on missing data) plus the 5-minute MongoDB backstop.
- **Shared library.** `CronHelper` lives in node-engine, which interview-system also uses. Both services must be deployed with the change, and interview-system must reach the same Redis. Ship the reader change first (Redis-first with MongoDB fallback, harmless on its own), then the writer change.
- **"Last seen" becomes up to 5 minutes stale in MongoDB.** Any other report or service reading `interviewActiveTimestamp` straight from MongoDB will see slightly older values. Grep both repos for readers before rollout. Known readers today: the cron, `callEndFn`, and the mock-interview cron (separate collection, out of scope).

### Open questions
- Does interview-system (which runs the cron) share the same Redis instance as incbot-video-system?
- PM: is the 1-hour "Network Disconnected" grace intentional? Would faster detection help, e.g. auto-notifying the employer within minutes?
- Frontend: can the four heartbeat emitters be reduced to one? That's a simple extra saving on network traffic.

---

## Suggested order of work
1. **Issue 1, Phase A** (CORS fix in report-only mode): about a day, removes a real weakness, no user impact.
2. **Issue 2, steps 1–2** (measure + configurable delay): low risk, gives us real numbers.
3. **Issue 1, Phase B** (socket and beam-token auth): highest security value; plan with the frontend team.
4. **Issue 2, steps 3–4** (readiness check, shadow → rollout).
5. **Issue 3** (presence in Redis), coordinated with the interview-system team because of the shared cron.
