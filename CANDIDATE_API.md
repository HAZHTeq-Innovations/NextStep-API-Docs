**This API is deliberately unreliable, like real AI services under load. Handling that is part of the challenge.**

# NextStep Mock API

NextStep is an AI decision assistant: someone describes a messy situation, and NextStep breaks it into issues, puts them in order, recommends one next action, asks clarifying questions, and reassesses when things change. You'll build a frontend against this mock instead of a real AI model.

Responses can be slow, cut off, malformed, missing, contradictory, or rate limited, just like a real model behind a real network. How your app behaves when that happens counts as much as how it looks when everything works.

- **Base URL:** `https://nextstepmockapi.onrender.com`
- **OpenAPI spec:** `openapi.yaml`, for generating a client
- **JSON Schema for responses:** `GET /v1/schema`

## Quick start

```bash
curl -X POST https://nextstepmockapi.onrender.com/v1/situations \
  -H 'Content-Type: application/json' \
  -H 'X-Candidate-Id: you@example.com' \
  -H 'Idempotency-Key: 6f1c2b8e-3f0a-4e4b-9a57-0f5d2f7c9e10' \
  -d '{
        "text": "Viva is at 10am tomorrow, laptop won'\''t boot, my project partner has been ignoring my calls for 2 days, and my dad just got admitted to a hospital in Surat. I'\''m in Pune.",
        "locale": "en-IN",
        "client_time": "2026-10-02T23:55:00+05:30"
      }'
```

`GET /v1/scenarios` lists seven test situations to try. Any other text works too: you'll get a more generic analysis.

## Headers you can send

| Header            | Purpose                                                                                                                                                                                                                                        |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `X-Candidate-Id`  | Use the email address you'll put in the submission form. **Always send it**, on every request. It gives you your own rate limit (see [Rate limit](#rate-limit)), and the failures you see come in a sequence that is reproducible for your id. |
| `Idempotency-Key` | A unique value per user action (a UUID is ideal). Sending the same key again within 10 minutes returns the same response and creates nothing new. Reusing a key with a different body returns `422 idempotency_key_reused`.                    |
| `X-Chaos`         | Forces a specific failure mode so you can reproduce it on purpose. See [Chaos modes](#chaos-modes).                                                                                                                                            |
| `Content-Type`    | `application/json` on every POST.                                                                                                                                                                                                              |

Browsers' `EventSource` can't send headers, so the stream endpoint also accepts `candidate_id` and `chaos` as query parameters. Put `candidate_id` on every stream URL you build.

**About `Idempotency-Key`:** without one, a double tap on "Submit" creates two separate situations, and like a real model, they won't necessarily agree on what to do first. If a request times out, the situation may still have been created; retrying with the same key returns it instead of creating a duplicate.

## Endpoints

| Method | Path                          | What it does                                                                  |
| ------ | ----------------------------- | ----------------------------------------------------------------------------- |
| GET    | `/health`                     | `{ "status": "ok" }`                                                          |
| GET    | `/v1/scenarios`               | The seven test scenarios: `id`, `type`, `input`                               |
| GET    | `/v1/schema`                  | JSON Schema for the analysis response                                         |
| POST   | `/v1/situations`              | Submit a new situation. Returns version 1 of its analysis.                    |
| GET    | `/v1/situations/{id}`         | The latest version, for resuming after the app is killed or the page reloads. |
| POST   | `/v1/situations/{id}/answers` | Answer or skip clarifying questions. Returns a new version.                   |
| POST   | `/v1/situations/{id}/updates` | Report what changed. Returns a new version with a `changes` list.             |
| POST   | `/v1/situations/stream`       | Register a long input for streaming; returns a `stream_url`.                  |
| GET    | `/v1/situations/stream`       | Server-Sent Events version of `POST /v1/situations`.                          |

`/health`, `/v1/scenarios` and `/v1/schema` are never affected by chaos.

## Submitting a situation

`POST /v1/situations`

```json
{
  "text": "free-text description of the situation",
  "locale": "en-IN",
  "client_time": "2026-10-02T23:55:00+05:30"
}
```

- `text` (required): up to about 100KB. Long pastes are fine.
- `locale` (optional).
- `client_time` (optional but recommended): the user's current time **with UTC offset**. NextStep uses it to work out what "tomorrow", "tonight" and "Friday" mean; "tomorrow" sent at 23:55 is the next calendar day. Without it, server time in UTC is used. It's accepted on answers and updates too; if one of those leaves it out, NextStep carries on from the clock of your last request for that situation.

### Response

Every successful analysis, from any endpoint, has this shape:

```json
{
  "situation_id": "sit_8f2k1x",
  "version": 1,
  "server_time": "2026-10-02T23:55:03+05:30",
  "mode": "standard",
  "summary": "Four things are landing at once: your dad is in hospital in Surat, your viva is at 10am tomorrow, your laptop won't boot, and your project partner has gone quiet. The deciding question is whether you need to travel to Surat tonight, because that changes what happens with the viva. The laptop and your partner can each be handled in a few minutes.",
  "issues": [
    {
      "id": "iss_1",
      "title": "Dad admitted to hospital in Surat",
      "category": "family",
      "urgency": 5,
      "deadline": null,
      "depends_on": []
    },
    {
      "id": "iss_2",
      "title": "Viva at 10am tomorrow",
      "category": "work_study",
      "urgency": 5,
      "deadline": "2026-10-03T10:00:00+05:30",
      "depends_on": ["iss_3"]
    },
    {
      "id": "iss_3",
      "title": "Laptop won't boot before the viva",
      "category": "work_study",
      "urgency": 4,
      "deadline": "2026-10-03T09:00:00+05:30",
      "depends_on": []
    },
    {
      "id": "iss_4",
      "title": "Project partner not answering for 2 days",
      "category": "work_study",
      "urgency": 3,
      "deadline": null,
      "depends_on": []
    }
  ],
  "priorities": [
    {
      "rank": 1,
      "issue_id": "iss_1",
      "action": "Call whoever is with your dad and find out how serious it is and whether they need you in Surat tonight",
      "reason": "This decides everything else tonight. Surat is most of a night away from Pune, so travelling means the viva has to move; if you are not needed there tonight, the viva becomes your top priority.",
      "estimated_minutes": 10
    },
    {
      "rank": 2,
      "issue_id": "iss_2",
      "action": "Email your guide or examiner tonight: say your dad is in hospital and ask whether the viva could be moved or done online if you have to travel",
      "reason": "The viva is in about 10 hours. A short heads-up tonight keeps your options open whichever way the family call goes, and early notice is much easier to work with than a missed slot.",
      "estimated_minutes": 15
    }
  ],
  "next_action": {
    "text": "Call whoever is with your dad now and ask one question: do they need you in Surat tonight?",
    "issue_id": "iss_1",
    "why": "The answer decides whether tonight is about travelling or about the viva, and the call takes five minutes."
  },
  "clarifying_questions": [
    {
      "id": "q_1",
      "question": "Do you need to travel to Surat tonight?",
      "options": ["Yes, I need to go tonight", "No, family has it covered", "Not sure yet"],
      "skippable": false
    },
    {
      "id": "q_2",
      "question": "Is the viva in person or online?",
      "options": ["In person", "Online", "Not sure"],
      "skippable": true
    }
  ],
  "missing_information": [
    "How serious your dad's condition is",
    "Whether you need to travel to Surat tonight"
  ],
  "risk_flags": [],
  "confidence": {
    "level": "medium",
    "reasons": ["Not yet known whether you need to travel tonight, which could reorder everything"]
  },
  "changes": [],
  "support": null
}
```

(Shortened: the real response has four priorities and three questions.)

### Fields

| Field                  | Notes                                                                                                                                                                |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `situation_id`         | `sit_` followed by six characters. Store it: you need it to resume, answer and update.                                                                               |
| `version`              | Starts at 1 and goes up by one with every answer or update.                                                                                                          |
| `mode`                 | `standard`, `support`, `out_of_scope` or `needs_clarification`. See below.                                                                                           |
| `issues[].category`    | `work_study`, `money`, `family`, `health`, `housing`, `relationships`, `travel` or `other`.                                                                          |
| `issues[].urgency`     | Integer 1 (low) to 5 (high).                                                                                                                                         |
| `issues[].deadline`    | ISO 8601 in the user's UTC offset, or `null`.                                                                                                                        |
| `issues[].depends_on`  | Issues that need dealing with first.                                                                                                                                 |
| `priorities`           | What to do, in order. `rank` runs 1, 2, 3…; `issue_id` points at an issue.                                                                                           |
| `next_action`          | The one thing to do now. Can be `null` outside `standard` mode.                                                                                                      |
| `clarifying_questions` | `options` are suggested answers; an empty `options` list means free text. `skippable: false` questions can't be skipped.                                             |
| `missing_information`  | What NextStep doesn't know yet.                                                                                                                                      |
| `risk_flags`           | Machine-readable flags, e.g. `possible_scam_message`. Treat values you don't recognise as informational.                                                             |
| `confidence`           | `level` is `low`, `medium` or `high`, with plain-language `reasons`.                                                                                                 |
| `changes`              | Empty on version 1. Later versions list what changed: `{ "field": "issues.iss_2.deadline", "from": "Thursday 8 October", "to": "Friday 9 October", "reason": "…" }`. |
| `support`              | Only set in `support` mode.                                                                                                                                          |

### Modes

- **`standard`:** a normal plan. If there are issues, there's a `next_action`.
- **`needs_clarification`:** NextStep needs answers before it can order things properly. There's always at least one clarifying question; `next_action` may be `null`.
- **`out_of_scope`:** the request isn't something NextStep does (for example, writing an essay). `summary` explains what it can and can't help with, and a question may offer an alternative.
- **`support`:** the person may be in distress. `issues` and `priorities` are empty, `next_action` is `null`, and `support` contains a calm message, `resources` (helplines), and an `offer_to_continue`. **Don't show a task list in this mode.** Support responses are always served without chaos.

## Resuming: `GET /v1/situations/{id}`

Returns the latest version. Use it when the app restarts or the page reloads mid-flow. Unknown or expired ids return `404 not_found`.

## Answering questions: `POST /v1/situations/{id}/answers`

```json
{
  "answers": [
    { "question_id": "q_1", "answer": "No, family has it covered" },
    { "question_id": "q_3", "answer": null }
  ],
  "client_time": "2026-10-03T00:05:00+05:30"
}
```

`answer: null` skips a question. Answers can reorder priorities, change the mode, and close questions; the new version's `changes` says what moved. You'll get `422 unknown_question` for a question that isn't open on the latest version, and `422 question_not_skippable` for skipping one that can't be skipped.

## Reporting changes: `POST /v1/situations/{id}/updates`

```json
{
  "text": "Laptop is fixed, but the viva moved to Monday",
  "client_time": "2026-10-03T08:10:00+05:30"
}
```

NextStep reassesses: resolved issues drop off, corrected dates move, things that got worse jump to the top, and anything else is added to the list. `changes` records each of these.

## Streaming: `/v1/situations/stream`

A Server-Sent Events version of `POST /v1/situations`, so you can show progress while the analysis builds.

**Short inputs:** open the stream directly.

```
GET /v1/situations/stream?text=...&client_time=...&candidate_id=...
```

**Long inputs** (URLs stop working somewhere around 8–16KB): register the text first, sending `X-Candidate-Id` as usual, then open the returned URL. It already includes your `candidate_id`.

```
POST /v1/situations/stream   { "text": "...", "client_time": "..." }
→ { "stream_token": "stk_…", "stream_url": "/v1/situations/stream?token=stk_…&candidate_id=…", "expires_in_seconds": 600 }
```

Events, in order:

| Event      | `data`                                                                                             |
| ---------- | -------------------------------------------------------------------------------------------------- |
| `status`   | `{ "stage": "reading" \| "identifying_issues" \| "prioritising" \| "finalising", "message": "…" }` |
| `issue`    | One issue                                                                                          |
| `priority` | One priority                                                                                       |
| `done`     | The full analysis, same shape as `POST /v1/situations`                                             |

```js
const source = new EventSource(
  `${BASE}/v1/situations/stream?text=${encodeURIComponent(text)}&candidate_id=${encodeURIComponent(email)}`,
);
source.addEventListener('issue', (e) => addIssue(JSON.parse(e.data)));
source.addEventListener('done', (e) => {
  source.close(); // otherwise EventSource reconnects and starts a new analysis
  showResult(JSON.parse(e.data));
});
source.onerror = () => {
  /* the stream can stop early, without a done event */
};
```

A stream can end without `done`, go quiet, or be refused with a plain `429` before it starts.

## Errors

Error bodies look like this:

```json
{
  "error": "rate_limited",
  "message": "Too many requests. Try again in 12 seconds.",
  "request_id": "req_3f9a1c0b2d4e"
}
```

…except when a proxy in front of the service fails, in which case you get an HTML page instead.

| Status | `error`                                                                | Meaning                                                |
| ------ | ---------------------------------------------------------------------- | ------------------------------------------------------ |
| 400    | `invalid_request`, `invalid_json`, `invalid_chaos_mode`                | Fix the request                                        |
| 404    | `not_found`, `stream_token_not_found`                                  | Unknown or expired                                     |
| 413    | `payload_too_large`                                                    | Body over 100KB                                        |
| 422    | `unknown_question`, `question_not_skippable`, `idempotency_key_reused` | Valid JSON that can't be applied                       |
| 429    | `rate_limited`                                                         | Slow down; wait for the `Retry-After` header's seconds |
| 500    | `internal_error` (or an HTML page)                                     | Server failure; it's usually worth retrying            |
| 502    | none: the body is empty                                                | The gateway gave up waiting; treat it like a timeout   |

Every response from the API carries an `X-Request-Id` header, and browser code can read `Retry-After` and `X-Request-Id`. A `502` comes from the gateway in front of the API, not the API itself, so it has neither, and no CORS headers either.

## Chaos modes

Any request to `POST /v1/situations`, `/answers`, `/updates` or the stream may fail in one of these ways. Force one with `X-Chaos: <mode>` (or `?chaos=<mode>` on the stream) to build and test your handling.

| Mode            | What you get                                                                                                                                                                                                                                                                                                               |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `none`          | A normal response, after a realistic delay of a few seconds.                                                                                                                                                                                                                                                               |
| `slow`          | A valid response that takes a long time. On the stream, events arrive seconds apart.                                                                                                                                                                                                                                       |
| `timeout`       | Nothing for a full minute, then the request fails: a `502` with an empty body from the gateway. In a browser, `fetch` usually rejects with a network error instead, because the gateway's response has no CORS headers. On the stream: a couple of events, a minute of silence, then the connection closes without `done`. |
| `malformed`     | `200` with `Content-Type: application/json`, but the JSON is cut off part-way.                                                                                                                                                                                                                                             |
| `partial`       | Valid JSON that breaks the schema: a missing field, a wrong type, an unexpected extra field, or an `issue_id` that doesn't exist.                                                                                                                                                                                          |
| `server_error`  | `500`: sometimes a JSON error, sometimes an HTML error page. On the stream: a couple of events, then it closes without `done`.                                                                                                                                                                                             |
| `rate_limit`    | `429` with a `Retry-After` header.                                                                                                                                                                                                                                                                                         |
| `tie`           | A valid response where two priorities are both rank 1.                                                                                                                                                                                                                                                                     |
| `contradiction` | A valid response whose `next_action` doesn't match the priorities or contradicts what the user said.                                                                                                                                                                                                                       |
| `empty`         | `200`, valid shape, `standard` mode, but no issues and no priorities.                                                                                                                                                                                                                                                      |

`GET /v1/situations/{id}` only ever sees `slow` and `server_error`. Without `X-Chaos`, modes are picked at random, in a sequence that's reproducible for your `X-Candidate-Id`.

## Rate limit

Separately from chaos, there's a real limit of **60 requests per minute per `X-Candidate-Id`**. Going over returns `429` with `Retry-After`. Retrying in a tight loop will hit it; back off instead.

**Always send `X-Candidate-Id`** (or `candidate_id` on stream URLs). Requests without it share one limit per IP address. That means everyone on your college or hostel Wi-Fi, or on the same mobile network, so someone else can use it up for you.

## Your data

- Situations are kept **in memory only** and **deleted 24 hours after they're created**. After that, `GET` returns `404`.
- Request text is **never logged**. Logs keep only the time, method, path, status, duration, candidate id, chaos mode, and a one-way hash of the text.
- It's still a test service: please use the sample scenarios or made-up situations rather than real personal details.
