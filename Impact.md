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
1. Keep **one** allowlist in node-engine, used by both Express and Socket.IO:
   ```ts
   /^https:\/\/([a-z0-9-]+\.)*incruiter\.com$/
   ```
2. Delete `cors({ origin: "*" })` from `server.ts`. Run the strict CORS middleware once, globally, **before** `tenantMiddleware`.
3. Return **403** for blocked websites instead of 500.
4. Run in **log-only mode for 1 week** first, so we find any real website we would accidentally block.
5. **Follow-up ticket:** add login checks to socket connections, `getTokenBeam` and `getInterviewInfo`. Delete `/test-merge-recording`.

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
Build the report **as soon as it's ready**, and keep a timer only as a backup.

The report is "ready" when:
1. the interview has ended,
2. all answers have arrived (the frontend sends `expectedAnswers` with `callEnd`),
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
Rollout steps:
1. Add logs to measure the real wait times.
2. Make the 6 minutes configurable through an env var.
3. Run the new check in "log only" mode.
4. Turn it on step by step.

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
1. Ignore repeat heartbeats from the same room within 25 seconds (fixes the 4× duplicates).
2. Write to MongoDB only:
   - on start,
   - on disconnect,
   - on end,
   - and once every 5 minutes as a backup.
3. The cron asks Redis "which rooms are silent?" instead of loading every interview from MongoDB.
4. **Safety rule:** if Redis is down, the cron must **not** mark anyone as disconnected.

Note: the cron lives in node-engine, which interview-system shares, so both teams must ship this together.

### Impact
| | Before | After |
|---|---|---|
| DB writes per interview | ~360 | ~10 (**97% fewer**) |
| DB writes/sec at 1,000 interviews | ~130 | ~3–4 |
| Cron work | Loads every live interview | One Redis query |
| Detect dropped candidate | ~1 hour | Can be minutes (Product decides) |

---

## Order of work
1. **CORS fix:** ~1 day, low risk.
2. **Report timer:** measure first, then switch on step by step.
3. **Socket/beam login checks:** most important for security.
4. **Heartbeat in Redis:** plan with the interview-system team.
