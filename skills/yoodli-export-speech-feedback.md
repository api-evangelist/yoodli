---
name: Export Yoodli coaching feedback for a session
description: Retrieve a Yoodli recording's rubric goal scores, coaching feedback, reviewer comments and transcript, including how to obtain the speechId in the first place.
api: openapi/yoodli-api-openapi.yml
base_url: https://app.yoodli.ai/api
operations:
  - GET /v3/speeches/{speechId}/feedback
generated: '2026-09-04'
method: generated
source: openapi/yoodli-api-openapi.yml + https://developers.yoodli.ai/docs/web-embed-api
---

# Export coaching feedback for a Yoodli session

> **Operation handles.** The published spec declares **no `operationId`**; the step is addressed by method + path.

## Getting a `speechId` — read this first

There is **no operation that lists or searches recordings.** `GET /v3/speeches/{speechId}/feedback` is the
only Speech operation Yoodli publishes, and it requires an id you already hold. There are exactly two
documented ways to obtain one, and neither is a server-side callback:

1. **The Web Embed `score_complete` postMessage event.** If you embed a Yoodli activity in your own page
   (iframe on `https://app.yoodli.ai`), the iframe posts `score_complete` to the parent window carrying
   `speechId`, `score` (0-100), `speechShareId`, `roleplayId`, `recordingId`, `recordingDuration` and
   `completedAt`. Verify `event.origin === 'https://app.yoodli.ai'` before trusting the message, then hand
   the `speechId` to your backend yourself. See `asyncapi/yoodli-web-embed-events.yml`.
2. **Out of band** — a human copies it from the application.

**Yoodli publishes no server-side webhook.** A backend is never notified that a session finished. If you are
building an unattended pipeline, the embed event is your only trigger.

## The call

`GET /v3/speeches/{speechId}/feedback`

- Auth: `Authorization: Bearer <API key>`.
- Optional `sections` query parameter selects what to return. Valid values (`RTFeedbackExportSection`):
  - `goals` — goal/rubric scores and their feedback
  - `coaching_feedback` — coaching feedback remarks
  - `user_comments` — reviewer/user comments
  - `transcript` — the full transcript
  Omit it to get everything; narrow it when you only need scores, since the transcript dominates the payload.
- Rate-limit category **Fast API — 100 calls per minute**.

## Responses

| Status | Meaning |
|---|---|
| `200` | The recording's AI coaching feedback (`FeedbackJsonResponse`) |
| `400` | Invalid query parameters — usually an unrecognised `sections` value |
| `403` | Not authorized to download this recording's feedback |
| `404` | Recording not found, or not accessible to the caller |

Note `403` and `404` are both used to mean "you cannot see this", so do not treat a 404 as proof the
recording does not exist.

## Handling the data

This payload contains a **transcript of a person speaking** and coaching remarks about them. Treat it as
personal data: it is the most sensitive thing this API returns. Yoodli's own posture is on its privacy
policy (SOC 2 Type 2, GDPR) at https://yoodli.ai/privacy and its trust center at https://trust.yoodli.ai/.

Capture the `x-request-id` response header — Yoodli support asks for it first.
