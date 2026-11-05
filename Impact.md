# InCBot Video System: 3 Fixes (Short Version)

Each fix has three parts: **Problem**, **Solution** and **Impact**.

---

## 1. CORS (which websites can talk to our server)

### Problem
- **Two CORS rules that disagree.** [server.ts:323](incbot-video-system/src/server.ts#L323) says "allow every website" (`origin: "*"`). A second rule ([corsMiddleware.ts](incbot-video-system/src/node-engine/middlewares/corsMiddleware.ts)) says "only `*.incruiter.com`". Nobody can easily tell which rule wins.
- **The socket rule has a bug.** [websocket.ts:63](incbot-video-system/src/websocket.ts#L63) uses `/\.?incruiter\.com$/`, which also allows `https://evilincruiter.com` and plain `http://` (tested).
- **Blocked websites get a 500 error.** In our dashboards this looks like the server crashed.
- **CORS only stops browsers.** A script can call our API directly. Some endpoints and socket events don't check who the caller is. For example, anyone with a `roomId` can call `getTokenBeam` and watch a live interview, or `joinRoom` and read live transcripts.

### Solution
1. **One allowlist.** In `node-engine/middlewares/corsMiddleware.ts`, export a single `isAllowedOrigin()` helper that Express and Socket.IO both use:
   ```ts
   /^https:\/\/([a-z0-9-]+\.)*incruiter\.com$/   // https only, anchored at both ends
   ```
2. **One CORS layer** (`server.ts`):
   - Delete `cors({ origin: "*" })`.
   - Mount `CORS_PERMISSIONS` once, globally, **before** `tenantMiddleware`, so error responses carry CORS headers too.
   - Remove the per-route copies.
3. **403 instead of 500.** Inside `corsMiddleware`, check the origin first and return `403 Origin not allowed`, with the origin logged. Today it passes an error to `next()`, which `errorHandler` turns into a 500.
4. **Sockets** (`websocket.ts`): replace the local regex with `isAllowedOrigin()`, using Socket.IO's `allowRequest` hook so it also covers direct WebSocket connections.
5. **Rollout:**
   - The HTTP change can ship directly. Bad origins are already blocked on `/incbot`; only the status code changes.
   - The socket rule gets stricter, so run it in **log-only mode for 1 week** (`SOCKET_ORIGIN_ENFORCE=false`), then enforce.
6. **Follow-up ticket (security):**
   - **Socket auth:** the client sends its interview token on connect (`auth: { token }`). The server verifies it like `USER_PERMISSIONS` does, then allows `joinRoom` only for that interview's room.
   - **Beam token:** protect `getTokenBeam` with a secret header (stored in Key Vault).
   - **Unprotected routes:** delete `/test-merge-recording`, and protect `getInterviewInfo` / `getInterviewToken`.

### Impact
| | Before | After |
|---|---|---|
| Fake domains like `evilincruiter.com` | Allowed | Blocked |
| Number of CORS rules | 3 | 1 |
| Blocked request in logs | 500 (looks like a crash) | 403 (clear) |
| Effect on real users | — | None |

---

## 2. Report waits a fixed 6 minutes

### Problem
- When an interview ends, `callEnd` waits **6 minutes** before building the report ([callEndService.ts:327](incbot-video-system/src/api/incbot/services/callEndService.ts#L327): `const delay = 360000`).
- The wait happens **even if every answer was already scored**.
- During the wait, the employer's dashboard shows **"In Progress"** for an interview that already ended.
- **Why the timer exists:** the server can't tell when two things are finished:
  - the last answer's audio may still be uploading from the browser,
  - the full call recording may not be saved yet.

  So it waits long enough for the worst case.

### Solution
Build the report **as soon as it's ready**, and keep the timer only as a backup.

The report is "ready" when:
1. the interview has ended,
2. all answers have arrived: the frontend sends `expectedAnswers` with `callEnd`, and the server counts arrivals in Redis (`INCR` in `transcribe()`),
3. no answers are still being scored (we already have this counter: [pendingEvaluations.ts](incbot-video-system/src/pendingEvaluations.ts)),
4. the recording URL is saved (AI-conversation interviews only).

How it works:
```
callEnd           → schedule the report job with a 10-min backup timer
answer scored     → check "ready?"  ─┐
answer uploaded   → check "ready?"   ├─ if ready → job.promote() → report runs now
recording saved   → check "ready?"  ─┘
backup timer ends → report runs anyway (same as today)
```

**Today vs. new**
| | Today | New |
|---|---|---|
| When we check | 6 min after `callEnd`, then every 30 s (polling) | Each time something finishes |
| What we check | Pending counter only | All 4 conditions |
| Worst case | Report built without late answers | Backup timer fires; never slower than today |

**How the check runs (no loop):** `triggerReportIfReady()` doesn't run on its own. It's called at the end of code that already runs when work finishes, like a `socket.on` handler that runs once per message. Each call reads Redis, decides "ready or not", and exits.

Calls are added in 3 places:
- `callEnd()`: the interview ended.
- `uncountOnSuccess` in [queueRegistry.ts](incbot-video-system/src/queueRegistry.ts): an answer was scored. This runs once per answer job, after the existing `notePendingDone()`.
- `saveRecording()`: the recording was saved.

What each call does:
```
              triggerReportIfReady()          (new file: reportReadiness.ts)
                        │
                        ▼
         ┌──────────────────────────────┐
         │ Interview ended?             │
         │ All answers arrived?         │
         │ No answers still scoring?    │
         │ Recording saved?             │
         └──────────────┬───────────────┘
                        │
            ┌───────────┴───────────┐
            ▼                       ▼
      any "no" → stop         all "yes" → start report now
     (a later call will              (job.promote())
       check again)
```

Example: 3 answers and a recording.
```
┌───────────────────────────────┐
│ 10:00:00  Candidate clicks End│
│ callEnd()                     │
│   → triggerReportIfReady()    │
└──────────────┬────────────────┘
               ▼
        Answers left: 3   Recording: ✗   → NOT READY, stop
               │
               ▼
┌───────────────────────────────┐
│ 10:00:10  Answer 1 scored     │
│ uncountOnSuccess()            │
│   → triggerReportIfReady()    │
└──────────────┬────────────────┘
               ▼
        Answers left: 2   Recording: ✗   → NOT READY, stop
               │
               ▼
┌───────────────────────────────┐
│ 10:00:20  Recording saved     │
│ saveRecording()               │
│   → triggerReportIfReady()    │
└──────────────┬────────────────┘
               ▼
        Answers left: 2   Recording: ✓   → NOT READY, stop
               │
               ▼
┌───────────────────────────────┐
│ 10:00:30  Answer 2 scored     │
│ uncountOnSuccess()            │
│   → triggerReportIfReady()    │
└──────────────┬────────────────┘
               ▼
        Answers left: 1   Recording: ✓   → NOT READY, stop
               │
               ▼
┌───────────────────────────────┐
│ 10:00:45  Answer 3 scored     │
│ uncountOnSuccess()            │
│   → triggerReportIfReady()    │
└──────────────┬────────────────┘
               ▼
        Answers left: 0   Recording: ✓   → READY ✅
               │
               ▼
┌───────────────────────────────┐
│ 🚀 Report starts now          │
│    (today: 10:06:00)          │
└───────────────────────────────┘
```
Whatever finishes last triggers the final check, so nothing needs to poll.

**Code changes**
- **New `src/reportReadiness.ts`:** `triggerReportIfReady(roomId)` runs the 4 checks, then calls `job.promote()` on the waiting report job.
- **`callEndDelay()`:**
  - Add a Redis "run once" key (`SET NX`), so the report is never built twice.
  - Give the 30-second re-queued job a known `jobId`, so it can be promoted.
- **Gaps to fill:**
  - Add `roomId` to coding jobs.
  - If `expectedAnswers` is missing, wait at least 30 s after `callEnd`.
  - If another service saves the recording URL, it must call us (e.g. `POST /incbot/recordingReady/:roomId`).
- **Status:** add a `reportStatus` field instead of changing `status`.

**Rollout:**
1. Measure the real wait times.
2. Make the delay configurable (`REPORT_START_DELAY_MS`).
3. Run the new check in log-only mode.
4. Enable gradually.

### Impact
| | Before | After |
|---|---|---|
| Wait before the report starts | 6 to 9.5 min | Usually under 1 min |
| Status after the interview | "In Progress" (wrong) | "Generating report" (correct) |
| Reports missing an answer | Possible | Less likely |
| Server cost | — | Same |

---

## 3. "Candidate is online" heartbeat writes to MongoDB

### Problem
- The browser sends `interview-active` every 30 seconds from **4 places at once**.
- Each one **writes to MongoDB** ([websocket.ts:319](incbot-video-system/src/websocket.ts#L319)). That's about **360 writes per interview**, or about **130 writes a second** at 1,000 live interviews.
- The cron that reads this data ([CronHelper.ts:1175](incbot-video-system/src/node-engine/helpers/CronHelper.ts#L1175)) runs only every **10 minutes**. It flags a candidate only after **1 hour** of silence.
- So we write very often but read rarely: lots of database cost for little benefit.

### Solution
Keep "last seen" in **Redis** (fast, in memory). Write to MongoDB only when something important happens.
```ts
// on each heartbeat: fast, no database write
await redis.zAdd("incbot:presence", { score: Date.now(), value: `${tenantId}:${roomId}` });
```
Main code changes:
- **New `node-engine/helpers/presence.ts`:** `markSeen()`, `lastSeen()` and `forget()`. It goes in node-engine because the cron in interview-system needs it too.
- **Heartbeat handler** (`websocket.ts`):
  - Write to Redis instead of MongoDB.
  - Ignore repeats from the same room within 25 s (`SET NX EX 25`), which fixes the 4× duplicates.
  - Write to MongoDB only once every 5 minutes as a backup.
  - If Redis is down, write to MongoDB as today.
- **On disconnect** (`websocket.ts`): save the last-seen time to MongoDB when the candidate's socket drops.
- **On call end** (`callEndFn()`): read last-seen from Redis for the "Interview Ended" row, then remove the room from Redis.
- **The cron** (`CronHelper.incbotNetworkIssue()`):
  - Use the **newer** of the Redis and MongoDB times.
  - **If Redis is down, skip the run**, so we never mark people disconnected because data is missing.

Rollout:
1. Confirm interview-system uses the same Redis.
2. Ship the cron change (reader) in both services first.
3. Ship the heartbeat change (writer).

### Impact
| | Before | After |
|---|---|---|
| DB writes per interview | ~360 | ~10 (**97% fewer**) |
| DB writes/sec at 1,000 interviews | ~130 | ~3–4 |
| Cron work | Loads every live interview | Same query, but reads last-seen from Redis |
| Detect dropped candidate | ~1 hour | Can be minutes (Product decides) |

---

## Order of work
1. **CORS fix:** ~1 day, low risk.
2. **Report timer:** measure first, then switch on step by step.
3. **Socket/beam login checks:** most important for security.
4. **Heartbeat in Redis:** plan with the interview-system team.
