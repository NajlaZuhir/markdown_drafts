# AI Manager — Integration Guide for Unity/VR Devs

> **Last updated:** 2026-08-16
> Feel free to ask questions on the channle.

---

## 1. The big picture

Every time a trainee speaks or does something in VR — talking to an NPC, picking something up — that moment gets sent to our backend as **one HTTP request**. The backend figures out what should happen (who was addressed, what they meant, how the character responds, whether it was the right thing to do) and sends back **one response** telling Unity what to do next: what to say, through which character, and what animation/action to play.

That's the whole loop. One endpoint, one request, one response, repeated every turn.

---

## 2. The endpoint

```
POST /api/ai/turn
Content-Type: application/json
Accept: application/json
```

Everything goes as **raw JSON in the request body** — not form data, not query parameters. If you're using `UnityWebRequest`, make sure you set `Content-Type: application/json` explicitly (the built-in `.Post()` helper defaults to form-encoding, which will break this).

**Auth note (updated):** this endpoint is now **behind authentication** — every request must include `Authorization: Bearer <token>`, using the token returned by your login flow. A request without a valid token gets `401`. See §5a for what this means for `session_id` specifically.

---

## 3. Why there's only one endpoint, in Abubaker's words

This design was deliberate, not accidental — here's the reasoning as originally explained by Abubaker, who built the original AI Manager pipeline:

> There are three types of client turns, and each one can trigger a different logic path inside the pipeline — for example, a spoken message versus an action that just updates the trainee's progress in the database. When I say "different route," I mean different *logic inside the pipeline*, not a different API endpoint — all three turn types go through the same `POST /api/ai/turn` endpoint. The pipeline branches internally based on the values you send, not based on the URL.
>
> Every variable in the request schema also exists as a real column in our database — so the request shape isn't arbitrary, it mirrors the actual data model. As we cover more procedures and the use cases get more complex, this pipeline can grow more branches. If it gets complicated enough, we may eventually split into two or three separate API routes to keep things manageable — but for now, one endpoint is the agreed design.

**In plain terms:** there are 3 kinds of turns (see `turn_kind` below), and the backend decides internally what to do with each one. You never need to call a different URL for a different kind of turn — just set the right fields for the kind of turn it is.

---

## 4. The 3 turn types (`turn_kind`)

| `turn_kind` | When to use it | What happens |
|---|---|---|
| `"voice"` | The trainee spoke | Full pipeline runs — the backend figures out who they addressed, what they meant, and generates a spoken reply + possible action. |
| `"action"` | The trainee physically did something, no speech | No reply is generated (nothing to say) — the backend just evaluates whether the action was correct and updates progress. |
| `"system"` | Reserved for non-trainee-initiated updates | Currently minimal use — check with backend before relying on this one. |

---

## 5. Required vs. optional fields — the real contract

**This is not a "send whatever you can" situation for these fields — they're strictly validated.** If a required field is missing, the request is rejected with a `422` error before anything else runs.

### Always required, every request
| Field | Type | Notes |
|---|---|---|
| `session_id` | integer | **Required — see §5a, this one has a real prerequisite step** |
| `procedure_id` | integer or string | Which procedure this is |
| `scenario_id` | integer | Must be a real scenario id |
| `mode` | string | Exactly `"training"` or `"evaluation"` — nothing else |
| `turn_kind` | string | Exactly `"voice"`, `"action"`, or `"system"` |
| `npcs_in_scene` | array | At least 1 entry |
| `npcs_in_scene[].npc_id` | integer | Must be a real NPC id |
| `npcs_in_scene[].name` | string | Max 100 characters |

### Required only for voice turns
| Field | Type | Notes |
|---|---|---|
| `user_input.text` | string | The trainee's speech-to-text output. Required and can't be empty when `turn_kind` is `"voice"`. |

### Required only for action turns
| Field | Type | Notes |
|---|---|---|
| `student_action` | string | Describes what the trainee physically did |

### Optional, but send whenever you have the data
| Field | Type | Notes |
|---|---|---|
| `step_id` | string | Which step of the procedure the trainee is currently on — **see §7, this one needs exact string matching with us** |
| `user_input.lang` | string | e.g. `"en"`, max 10 chars |
| `focus` | object | See §6 below — send it whenever gaze can be determined |
| `focus.npc_id` | integer | Must be a real NPC id |
| `focus.confidence` | number | Between `0` and `1` — see §6, this isn't decorative |
| `npcs_in_scene[].role` | string | e.g. `"patient"`, `"nurse"` |
| `context` | object | Free-form extra info, e.g. `{"patient_status": "critical"}` |

Anything you send outside these field names is silently ignored — the backend only reads what's declared here.

---

## 5a. `session_id` has a real prerequisite — a session must exist first

You can't just make up a `session_id` or reuse an old one. It **must belong to the authenticated user** making the request, and there's no way to skip this: `session_id` doesn't just have to *exist*, the backend checks it's owned by whoever's token is on the request.

**Before your first turn in a scenario, call one of these:**

```
POST /api/session/training
POST /api/session/evaluation
```
```json
{ "procedure_id": 2, "scenario_id": 2 }
```

Both require the same `Authorization: Bearer <token>` header. The response gives you the `session_id` to use for every subsequent turn in that run:

```json
{ "session_id": 3, "procedure_id": 2, "scenario_id": 2, "type": "training", "status": "in_progress" }
```

Send a `session_id` that doesn't belong to you (someone else's, a made-up number, an old one from a different user) and you'll get a `422` explaining exactly that — not a silent failure, so it should be easy to debug if it ever comes up.

---

## 6. `focus` and `target_npc` — how "who's being talked to" actually works

This is the part that comes up most, so worth spelling out clearly.

**`focus` is what YOU send in** — it's just gaze. Whatever NPC the trainee is currently looking at, resolved entirely on your side (Unity's gaze ray). Send it honestly every turn, including your real confidence level — don't always send `1.0` regardless of certainty, since the backend actually uses that number as a threshold check. If confidence is too low and nothing else in the request clarifies who's being addressed, the backend will deliberately ask for clarification rather than guess wrong.

**`target_npc` is what comes back in the response** — the backend's actual decision on who was addressed. Important: **a spoken name always overrides gaze.** If the trainee is looking at NPC A but says NPC B's name, `target_npc` will be NPC B — the backend resolves this conflict for you, you don't need to.

**What you need to do with it:** read `target_npc.npc_id` from the response, look it up in your own local registry (built when the scenario loads — see §8), and route the reply text + action to *that* character, not necessarily the one that was being looked at.

---

## 7. `step_id` — the one field with no validation net

There's no fixed list of allowed values for `step_id` — it's whatever step key each procedure has been authored with backend-side (e.g. `"ppe_sterile_prep"`, `"landmark_identification"`). **If you send a key that doesn't match exactly, the request doesn't error — it just silently loses step context**, which can be confusing to debug since nothing visibly breaks. Always confirm the exact step keys with us per procedure rather than guessing or reusing keys across procedures.

---

## 8. Where NPC ids actually come from

Before your first turn in a scenario, you need to already know which `npc_id` corresponds to which character in your scene — the request doesn't tell you this, you tell the request. That mapping is expected to come from fetching the scenario's NPC roster at load time. **This part of the API is not fully working yet** — flag directly with backend before relying on it, so we can confirm the right way to fetch this for you.

---

## 9. What comes back — the response

```json
{
  "target_npc": { "npc_id": 6, "name": "Sarah", "role": "nurse" },
  "response_text": "Here are the sterile gloves.",
  "action_command": "AnimHandOverGloves",
  "intent": { "name": "action_confirm", "confidence": 0.95 },
  "object_ref": null,
  "evaluation": {
    "target_correct": true,
    "intent_recognized": true,
    "action_correct": true,
    "step_appropriate": true
  },
  "emotion": "urgent",
  "meta": { "provider": "openai", "model": "gpt-4o-mini", "latency_ms": 1613 }
}
```

What to actually do with each part:
- **`target_npc` + `response_text`** → feed to your on-device TTS, voiced as that character.
- **`action_command`** → dispatch into your animation system for that same character.
- **`evaluation`** → optional, for scoring/feedback UI if you want to show it. Not required for the simulation to function.
- **`meta`** → diagnostic info, not meant for the trainee.

If a turn produces no reply (e.g. addressing a non-conversational character, or an action turn), `response_text` will be an empty string and `action_command` will be `null` — that's expected, not an error.

---

## 10. What procedures exist right now

Two, fully seeded and tested end-to-end:

| Procedure | `procedure_id` | `scenario_id` | Notes |
|---|---|---|---|
| NG Tube Feeding | `2` | `2` | Conscious adult patient, addresses the nurse or patient directly |
| Chest Tube Intubation | `3` | `3` | Trauma patient (non-conversational) — nurse is the default addressee |

(A note on Abubaker's earlier explanation, since it referenced "NG-2, one of two seeded procedures" — that was accurate at the time it was recorded, but a second procedure (Chest Tube) has since been fully built and tested, so there are now two real procedures with full coverage, not just one plus a placeholder.)

Use the seeded ids above to test connectivity — don't use `procedure_id: 1` / `scenario_id: 1`, those don't correspond to either real procedure.

---

## 11. Where to find working examples

The Postman collection (`Esculapio.AI.postman_collection.json`, "AI Manager" folder) has 16 real, passing request/response examples — 8 for each procedure — covering every scenario type described above (spoken name, gaze fallback, correct/incorrect actions, ambiguous turns, etc.).

**Updated for the new auth/session requirements** and re-verified live end-to-end (register → login → create session → turn request, plus both rejection cases). Run the folders in this order for it to work correctly:

```
Authentication → User Profile → Voice Profile → VR Sessions → AI Manager
```

`VR Sessions` creates the session NGT's requests use; the `Chest Tube` sub-folder inside `AI Manager` has its own equivalent first request (`00 — Start Training Session (Chest Tube)`) so it doesn't collide with NGT's session. If you're unsure how a specific case should be structured, check there first before asking.
