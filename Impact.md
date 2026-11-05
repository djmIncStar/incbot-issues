# InCBot Video System: 3 Fixes (Short Version)

Each fix has three parts: **Problem**, **Solution** (files to change, code flow, rollout) and **Impact**.

---

## 1. CORS (which websites can talk to our server)

### Problem
- **Two CORS rules that disagree.** [server.ts:323](incbot-video-system/src/server.ts#L323) says "allow every website" (`origin: "*"`). A second rule ([corsMiddleware.ts](incbot-video-system/src/node-engine/middlewares/corsMiddleware.ts)) says "only `*.incruiter.com`". Nobody can easily tell which rule wins.
- **The socket rule has a bug.** [websocket.ts:63](incbot-video-system/src/websocket.ts#L63) uses `/\.?incruiter\.com$/`, which also allows `https://evilincruiter.com` and plain `http://` (tested).
- **Blocked websites get a 500 error.** In our dashboards this looks like the server crashed.
- **CORS only stops browsers.** A script can call our API directly. Some endpoints and socket events don't check who the caller is. For example, anyone with a `roomId` can call `getTokenBeam` and watch a live interview, or `joinRoom` and read live transcripts.

### Solution

#### Part A: one clean CORS rule (~1 day)

**Files to change**
| File | Change |
|---|---|
| `node-engine/middlewares/corsMiddleware.ts` | Export one shared allowlist. Return 403 instead of throwing |
| `src/server.ts` | Remove `cors({ origin: "*" })`. Mount CORS once, before `tenantMiddleware` |
| `src/websocket.ts` | Use the shared allowlist instead of the local regex |

**Step 1: one allowlist, exported from node-engine** (`corsMiddleware.ts`)
```ts
export const ALLOWED_ORIGINS = [
  /^https:\/\/([a-z0-9-]+\.)*incruiter\.com$/, // https only, anchored at both ends
  /^http:\/\/localhost:\d+$/,                  // local development
];
export const isAllowedOrigin = (origin?: string) =>
  !!origin && ALLOWED_ORIGINS.some((r) => r.test(origin));
```

**Step 2: reject with 403, not a thrown error** (inside the same middleware). Today the `origin` callback passes an `Error` to `next()`, and [errorHandler.ts](incbot-video-system/src/common/middleware/errorHandler.ts) turns any error into a 500. Check the origin first instead:
```ts
const origin = req.headers.origin;
if (origin && !isAllowedOrigin(origin)) {
  logger.warn({ message: "CORS: origin blocked", origin, path: req.path });
  return res.status(403).json({ message: "Origin not allowed" });
}
// origin is either absent (server-to-server, curl) or already validated
return cors({ origin: true, credentials: true })(req, res, next);
```
Leave the existing Host-header check and the Postman key bypass as they are.

**Step 3: mount it once, first** (`server.ts`)
```ts
// before                                   // after
app.use(tenantMiddleware);                  app.use(CORS_PERMISSIONS);   // 1. CORS first
app.use(cors({ origin: "*" }));             app.use(tenantMiddleware);   // 2. then tenant
...                                         ...
app.use("/incbot", CORS_PERMISSIONS, r);    app.use("/incbot", r);       // per-route copy removed
```
With CORS first, even a tenant-lookup error goes back with correct CORS headers, so the frontend sees the real error instead of a fake "CORS error".

**Step 4: sockets use the same list** (`websocket.ts`). Use Socket.IO's `allowRequest` hook. It runs for every new connection, including direct WebSocket upgrades:
```ts
import { isAllowedOrigin } from "@/node-engine/middlewares/corsMiddleware";

const ENFORCE = process.env.SOCKET_ORIGIN_ENFORCE === "true";

const io = new SocketIOServer(server, {
  transports: ["websocket"],
  allowRequest: (req, cb) => {
    const ok = isAllowedOrigin(req.headers.origin);
    if (!ok) console.warn(`[socket] origin not allowed: ${req.headers.origin}`);
    cb(null, ok || !ENFORCE); // log-only until ENFORCE=true
  },
});
```
Delete the old `allowedDomains` array and the `cors.origin` callback.

**Rollout**
- **HTTP change:** low risk. Bad origins are *already* blocked on `/incbot` today; only the status code changes, from 500 to 403. Ship it directly.
- **Socket change:** this *tightens* the rule (no more `http://`, no look-alike domains), so first deploy with `SOCKET_ORIGIN_ENFORCE=false` for 1 week. Check the logs for blocked origins. If they're all junk, set it to `true`.

#### Part B: real identity checks (follow-up tickets)

CORS can't stop scripts, so the server must check **who** is calling.

| Where | Change |
|---|---|
| **Socket connect** (`websocket.ts`) | Add `io.use()` auth middleware. The client sends its token in `io(url, { auth: { token } })`. The server looks it up the same way `USER_PERMISSIONS` does (`EncryptedToken` → JWT → `interviewId`, `tenantId`). Save the result on `socket.data.auth` |
| **`joinRoom`** | Allow only if `roomId` belongs to `socket.data.auth.interviewId`. Remove the "no roomId → generate a new room" branch |
| **`interview-active`, `stt:*`, `tts:*`** | Use `tenantId` from `socket.data.auth`, never from the event payload. Reject if not authenticated |
| **`GET /getTokenBeam`** | Add a `BEAM_PERMISSION` middleware: the recorder sends a secret header (stored in Key Vault), compared with `crypto.timingSafeEqual` |
| **`/test-merge-recording`** | Delete the route |
| **`getInterviewInfo`, `getInterviewToken`** | Check what the pre-join page sends. If the candidate has no token yet, strip sensitive fields from the response |

Code flow for socket auth:
```
client: io(url, { auth: { token } })
   → io.use(): EncryptedToken.findOne({ encryptedAccessToken: token, status: "active" })
             → jwt.verify(accessToken) → { interviewId, tenantId }
             → socket.data.auth = { interviewId, tenantId, role }
   → joinRoom(roomId): InCBotInterview.findOne({ _id: interviewId, roomId }) ? join : reject
```
Rollout: (1) the frontend starts sending the token, while the server ignores it. (2) The server checks it but only logs failures. (3) Enforce with `SOCKET_AUTH_ENFORCE=true`.

Note: the candidate token is created in `callStart` ([transcribeService.ts:600-633](incbot-video-system/src/api/incbot/services/transcribeService.ts#L600-L633)). Confirm the frontend joins the socket room only after `callStart`, or issue the token earlier.

### Impact
| | Before | After |
|---|---|---|
| Fake domains like `evilincruiter.com` | Allowed | Blocked |
| Number of CORS rules | 3 | 1 |
| Blocked request in logs | 500 (looks like a crash) | 403 (clear) |
| Effect on real users | — | None |
| Watch a live interview with only a `roomId` (Part B) | Possible | Blocked |

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
Build the report **as soon as it's ready**. Keep the timer only as a backup.

**The report is "ready" when all of these are true**
1. The interview has ended.
2. All answers have arrived (the frontend tells us how many to expect).
3. No answers are still being scored (we already have this counter: [pendingEvaluations.ts](incbot-video-system/src/pendingEvaluations.ts)).
4. The recording URL is saved (AI-conversation interviews only).

**Files to change**
| File | Change |
|---|---|
| `src/reportReadiness.ts` (**new**) | `checkReportReady()` and `triggerReportIfReady()` |
| `callEndService.ts` → `callEnd()` | Save "ended" + `expectedAnswers` in Redis. Make the delay configurable. Call the trigger |
| `callEndService.ts` → `callEndDelay()` | Add a "run only once" guard. Give re-queued jobs a findable ID |
| `transcribeService.ts` → `transcribe()` | Count answers received |
| `queueRegistry.ts` → `uncountOnSuccess` | Call the trigger after an answer finishes scoring |
| `codingService.ts` | Add `roomId` to the coding job data (today it only has `interviewId`) |
| `callEndService.ts` → `saveRecording()` | Call the trigger after the recording is saved |
| Frontend `callEnd` request | Send `expectedAnswers` = number of `/transcribe` uploads it started |

**Redis keys used** (all expire after 1 day)
```
report:ended:<roomId>      → time callEnd happened
report:expected:<roomId>   → expectedAnswers sent by the frontend
report:received:<roomId>   → INCR on every /transcribe that arrives
report:jobId:<roomId>      → ID of the report job currently waiting
report:started:<roomId>    → "run once" guard
```

**Step 1: new module `src/reportReadiness.ts`**
```ts
export async function checkReportReady(roomId: string, interviewId: string) {
  const missing: string[] = [];

  if (!(await redisClient.exists(`report:ended:${roomId}`))) missing.push("not-ended");

  const expected = Number(await redisClient.get(`report:expected:${roomId}`));
  const received = Number(await redisClient.get(`report:received:${roomId}`) ?? 0);
  if (expected && received < expected) missing.push(`answers ${received}/${expected}`);

  if ((await pendingCount(roomId, interviewId)) > 0) missing.push("still-scoring");

  if (await needsRecording(roomId) && !(await hasRecording(roomId))) missing.push("recording");

  return { ready: missing.length === 0, missing };
}

export async function triggerReportIfReady(roomId: string, interviewId: string) {
  const { ready, missing } = await checkReportReady(roomId, interviewId);
  if (!ready) return;

  if (process.env.REPORT_FAST_START !== "true") {          // shadow mode
    logger.info({ message: "report would start now", roomId, msSinceEnd: /* now - ended */ 0 });
    return;
  }
  const jobId = await redisClient.get(`report:jobId:${roomId}`);
  const job = jobId ? await queues[getQueueNameFor("callEndDelayNew")]?.getJob(jobId) : null;
  if (job && (await job.isDelayed())) await job.promote();  // run the waiting job now
}
```

**Step 2: `callEnd()` changes**
```ts
await redisClient.set(`report:ended:${roomId}`, Date.now(), { EX: 86400 });
if (input.expectedAnswers) await redisClient.set(`report:expected:${roomId}`, input.expectedAnswers, { EX: 86400 });

const delay = Number(process.env.REPORT_START_DELAY_MS ?? 360000);  // was hard-coded
await addJob("callEndDelayNew", ..., { delay, jobId: `callEndDelay-${roomId}` });
await redisClient.set(`report:jobId:${roomId}`, `callEndDelay-${roomId}`, { EX: 86400 });

void triggerReportIfReady(roomId, interviewId);   // everything may already be done
```
If the frontend doesn't send `expectedAnswers` (old app version, tab crash), skip check 2. In that case, never promote earlier than **30 seconds** after `callEnd`, to give in-flight uploads time to land.

**Step 3: call the trigger wherever "something finished"**
```ts
// transcribeService.transcribe()  – an answer upload arrived
await redisClient.incr(`report:received:${roomId}`);

// queueRegistry.uncountOnSuccess  – an answer finished scoring
await notePendingDone(kind, id);
void triggerReportIfReady(roomId, interviewId);

// callEndService.saveRecording()  – recording URL saved
void triggerReportIfReady(roomId, interviewId);
```
Before `callEnd`, the trigger is a cheap no-op (check 1 fails), so it's safe to call any time.

**Step 4: make `callEndDelay()` safe to call early or twice**
- **Run once:** right after the "still scoring?" check passes:
  ```ts
  const first = await redisClient.set(`report:started:${roomId}`, "1", { NX: true, EX: 86400 });
  if (first === null) return;   // another run already built this report
  ```
  If the report build throws, delete the key so a retry can run. This is the same pattern as the existing mail guard ([callEndService.ts:37-60](incbot-video-system/src/api/incbot/services/callEndService.ts#L37-L60)).
- **Findable re-queue:** today the "still scoring, wait 30 s" path ([callEndService.ts:568](incbot-video-system/src/api/incbot/services/callEndService.ts#L568)) re-adds the job **without** an ID, so the trigger can't find it. Give it `jobId: callEndDelay-<roomId>-<n>` and store that in `report:jobId:<roomId>`.

**Step 5: dashboard status (needs a PM decision).** Don't change the `status` value: many queries and crons filter on `"In Progress"`. Add a separate field instead, for example `reportStatus: "generating" | "ready"`, which the dashboard shows.

**Code flow, end to end**
```
callEnd ──► save ended/expected ──► add job (6-min backup) ──► trigger()
                                                                  │
answer uploaded ──► received++                                    │
answer scored   ──► pending-- ──► trigger() ─┐                    │
recording saved ──────────────► trigger() ───┼──► checkReportReady()
                                             │       ├─ not ready → do nothing
                                             │       └─ ready     → job.promote()
backup timer ends ──────────────────────────────────► callEndDelay() runs anyway (as today)
```

**Rollout**
1. Ship the Redis keys and logs only. Measure the real time from `callEnd` to "everything done".
2. Turn on `REPORT_START_DELAY_MS`, still at 6 min.
3. Turn on the trigger in shadow mode (`REPORT_FAST_START=false`): it only logs "would start now".
4. Set `REPORT_FAST_START=true` for practice interviews, then a few customers, then everyone.

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
Keep "last seen" in **Redis**. Write to MongoDB only when it matters.

**Files to change**
| File | Change |
|---|---|
| `node-engine/helpers/presence.ts` (**new**) | Shared helper: `markSeen`, `lastSeen`, `forget`. Lives in node-engine because interview-system's cron needs it too |
| `src/websocket.ts` → `interview-active` | Write to Redis. Write to MongoDB at most once per 5 min |
| `src/websocket.ts` → `disconnect` | Save the last-seen time to MongoDB when the candidate's socket drops |
| `callEndService.ts` → `callEndFn()` | Read last-seen from Redis first. Clean up the Redis entry |
| `node-engine/helpers/CronHelper.ts` → `incbotNetworkIssue()` | Use the newest of Redis and MongoDB. Skip the run if Redis is down |

**Step 1: new helper `node-engine/helpers/presence.ts`**
```ts
const KEY = "incbot:presence";                       // sorted set: member → last-seen time
const member = (tenantId: string, roomId: string) => `${tenantId}:${roomId}`;

/** Returns false for duplicate beats; persist=true at most once every 5 minutes. */
export async function markSeen(redis, tenantId: string, roomId: string) {
  const fresh = await redis.set(`incbot:presence:beat:${roomId}`, "1", { NX: true, EX: 25 });
  if (fresh === null) return { counted: false, persist: false };   // 4 emitters → 1 write

  await redis.zAdd(KEY, { score: Date.now(), value: member(tenantId, roomId) });
  const persist = await redis.set(`incbot:presence:db:${roomId}`, "1", { NX: true, EX: 300 });
  return { counted: true, persist: persist !== null };
}

export async function lastSeen(redis, tenantId: string, roomId: string): Promise<Date | null> {
  const score = await redis.zScore(KEY, member(tenantId, roomId));
  return score ? new Date(score) : null;
}

export async function forget(redis, tenantId: string, roomId: string) {
  await redis.zRem(KEY, member(tenantId, roomId));
}
```

**Step 2: heartbeat handler** (`websocket.ts`, `interview-active`)
```ts
// keep the existing guards (roomId present, valid timestamp, interviewEndDate: null)
let persist = true;                                   // Redis down → old behaviour
if (redisClient?.isReady) {
  ({ persist } = await markSeen(redisClient, tenantId, roomId));
}
if (persist) {
  await InCBotInterview.collection.updateOne(        // existing write, now rare
    { roomId: new mongoose.Types.ObjectId(roomId), interviewEndDate: null },
    { $set: { interviewActiveTimestamp: new Date() } },
  );
}
```
If Redis is unavailable, it simply writes to MongoDB as today, so nothing breaks.

**Step 3: on candidate disconnect** (`websocket.ts`, `disconnect`). Copy the Redis last-seen time into MongoDB. MongoDB then holds an exact value right when the candidate drops, which is the moment that matters.

**Step 4: call end** (`callEndService.callEndFn()`, [L400-430](incbot-video-system/src/api/incbot/services/callEndService.ts#L400-L430))
```ts
const seen = await lastSeen(redisClient, tenantId, roomId);
const interviewActiveTimestamp = newest(seen, existing?.interviewActiveTimestamp);
// pass interviewActiveTimestamp into resolveEndRowTimestamp(...) as today
await forget(redisClient, tenantId, roomId);
```

**Step 5: the cron** (`CronHelper.incbotNetworkIssue()`)
```ts
if (!redisClient?.isReady) {
  logger.warn({ message: "presence: Redis down, skipping network-disconnect check this run" });
  return;  // never flag people because data is missing
}
for (const interview of interviews) {
  const seen = newest(
    await lastSeen(redisClient, tenantId, String(interview.roomId)),
    interview.interviewActiveTimestamp,              // MongoDB backup value
  );
  // existing rule unchanged, but use `seen` instead of interview.interviewActiveTimestamp
}
```
Using the **newest** of the two values keeps this correct during rollout, while some pods still write only to MongoDB.

**Code flow**
```
heartbeat ─► dedupe (25 s) ─► Redis ZADD ─► every 5 min: MongoDB write (backup)
disconnect ─► MongoDB: last seen
callEnd ─► read Redis last seen ─► "Interview Ended" row ─► remove from Redis
cron ─► Redis down? skip : compare newest(Redis, MongoDB) with the existing rule
```

**Rollout**
1. Confirm that interview-system uses the **same Redis** as this service. The cron runs there.
2. Ship `presence.ts` and the cron change (reader) in **both** services first. Nothing writes to Redis yet, so it behaves exactly as today.
3. Ship the heartbeat change (writer) in incbot-video-system. Watch the MongoDB write rate drop.
4. Optional, if Product wants: run the cron more often and shorten the 1-hour rule.

### Impact
| | Before | After |
|---|---|---|
| DB writes per interview | ~360 | ~10 (**97% fewer**) |
| DB writes/sec at 1,000 interviews | ~130 | ~3–4 |
| Duplicate heartbeats processed | 4× | 1× |
| Detect dropped candidate | ~1 hour | Can be minutes (Product decides) |
| If Redis goes down | — | Falls back to today's behaviour; no false "disconnected" |

---

## Order of work
1. **CORS Part A:** ~1 day, low risk.
2. **Report timer:** steps 1–3 (measure, shadow mode), then switch on step by step.
3. **CORS Part B (socket/beam login checks):** most important for security; needs frontend work.
4. **Heartbeat in Redis:** ship together with the interview-system team (shared cron).
