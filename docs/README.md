# Frigate VLM-driven alerting

Rule-based smart alerts on top of Frigate's per-object GenAI (VLM) descriptions,
integrated into the SgtBatten Frigate Notifications blueprint — **not** a parallel
notification system.

## Architecture (3 pieces)

```
[Frigate GenAI]
   |  frigate/tracked_object_update   {type:"description", id, description:"```json…```"}
   |     (async, ~seconds after event end, FENCED JSON, NO camera)
   v
PREPROCESSOR  (packages/frigate_vlm.yaml)
   - strip ```json fences  -> parse
   - DROP if not valid / not structured (no confidence+image_quality)
   - resolve id -> camera/label/zones via Frigate API /api/events/<id>
   - coerce types ("false"->false, "3"->3)   <-- prevents string-truthiness bugs
   |  frigate_vlm/events/<camera>   (clean contract, below)
   v
BLUEPRINT  (Stable_VLM.yaml — one automation per camera)
   - trigger on frigate_vlm/events/<camera>
   - gate on confidence/image_quality thresholds
   - evaluate per-camera vlm_rules template -> {notify, priority, message}
   - send / upgrade notification (shared tag => replaces baseline notification)
```

### Why a preprocessor (and not blueprint-only)

`frigate/tracked_object_update` contains **only `id` + `description`** — no camera,
no zones. Routing to per-camera rules requires resolving the camera, which needs a
lookup. The preprocessor does that once and isolates every Frigate quirk (fences,
async timing, type sloppiness, duplicate messages) behind a stable contract, so the
blueprint stays simple and survives Frigate upgrades.

## Contract: `frigate_vlm/events/<camera>`

```jsonc
{
  "id": "1700000000.123456-abcdef",
  "camera": "<camera>",
  "label": "person",
  "sub_label": null,
  "zones": ["<zone_a>", "<zone_b>"],
  "score": 0.77,
  "schema_version": 3,
  "vlm": {                       // fully type-coerced
    "schema_version": 3,
    "scene_observation": "An adult is walking up the driveway toward the door",
    "people": [{ "appearance": "adult, dark jacket", "apparent_age": "adult",
                 "location": "driveway", "action": "walking toward the door" }],
    "person_present": true, "likely_false_detection": false,
    "person_count": 1, "adults_count": 1, "children_count": 0,
    "confidence": "high", "image_quality": "good",
    "summary": "Adult walking up the driveway"
  }
}
```

## VLM JSON schema (the field contract)

The schema is **entirely up to your camera prompts** — the preprocessor and blueprint
never hardcode field names; rules reference whatever fields your prompt emits, always via
`default()`. The preprocessor is version-agnostic: it republishes whatever
`schema_version` the prompt emits, so you can evolve the schema without touching the
pipeline. Below is the **illustrative** field set used by the example rules; adapt freely.

**Core (recommended in every genai camera):**
`schema_version`, `scene_observation`, `people` (array — one object per clearly-visible
human: `appearance`/`apparent_age`/`location`/`action`), `person_present`,
`likely_false_detection`, `person_count`, `adults_count`, `children_count`,
`confidence` (`low|medium|high`), `image_quality` (`poor|fair|good`), `summary`.

> **Describe-first + grounding (recommended).** Have the prompt fill `scene_observation` and
> `people` FIRST, then the booleans, with `person_count == len(people)`. The rules then
> *reconcile*: act only when a real person is actually listed (`person_present` and not
> `likely_false_detection` and `people` non-empty), which suppresses hallucinated flags
> (e.g. "person on balcony" when the scene is empty). For a location-specific rule, cross-check
> `people` (e.g. someone actually on the balcony). See the example rules.
>
> **No identity.** Face/identity recognition is unreliable — don't ask the model "who".
> Classify by apparent age and drive alerts off behaviour, safety, and presence.
>
> `confidence` and `image_quality` must be named/valued consistently across all cameras —
> they drive the quality gate. Everything else is yours to define.

**Example pool-camera fields:** `child_supervised`, `child_in_water`, `child_near_pool_edge`.

**Example perimeter/front-camera fields:** `person_role` (`delivery|service_worker|generic`),
`package_being_delivered`, `package_being_taken`, `pedestrian_perimeter_passable`,
`vehicle_perimeter_passable` (conservative — true only when an opening is clearly visible),
`non_standard_approach`, `child_unsupervised_near_exit`, `suspicious_behavior`,
`suspicious_behavior_details`, `animal_in_frame`.

**Example elevated/balcony-camera fields:** `balcony_occupied`, `child_on_balcony`,
`child_climbing_balcony`, `loitering_on_street`, `person_observing_property`.

## Files

The repo mirrors your HA `/config` layout — each file copies to the matching `/config` path.

| Repo file | Goes to | Purpose |
|---|---|---|
| `custom_templates/frigate_vlm.jinja` | `/config/custom_templates/frigate_vlm.jinja` | macros: `parse_description`, `safe_get`, `vlm_clean_json` |
| `custom_templates/frigate_vlm_rules.jinja` *(your private copy — gitignored; start from `examples/`)* | `/config/custom_templates/frigate_vlm_rules.jinja` | your per-camera rule macros |
| `packages/frigate_vlm.yaml` | `/config/packages/frigate_vlm.yaml` | `rest_command` + preprocessor + siren-escalation automations |
| `packages/vlm_silence.yaml` | `/config/packages/vlm_silence.yaml` | per-camera `input_datetime` helpers for restart-safe "Silence this camera" |
| `blueprints/automation/frigate_vlm/Stable_VLM.yaml` | `/config/blueprints/automation/frigate_vlm/Stable_VLM.yaml` | the notification blueprint with the VLM branch |
| `examples/frigate_vlm_rules.example.jinja` | *(template — copy & adapt to the path above)* | anonymized starter rules |
| `examples/frigate_config.example.yaml` | *(reference — lives on the **Frigate** host, not HA)* | Frigate-side setup: Coral TPU + GenAI provider + per-camera prompt/zones |

## Install (step-by-step)

1. **Custom template** — copy `frigate_vlm.jinja` to `/config/custom_templates/frigate_vlm.jinja`
   (create the folder if needed). Custom templates are only re-read on a **full restart**.
2. **Enable packages** — in `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
3. **Packages** — copy both into `/config/packages/`:
   - `packages/frigate_vlm.yaml` (preprocessor + siren). **Edit the
     `rest_command.frigate_get_event` URL** inside it: replace `<FRIGATE_HOST>` with your
     Frigate host/IP (the unauthenticated internal API, usually port 5000), e.g.
     `http://192.168.x.x:5000`. This is a package config value, not a blueprint input.
   - `packages/vlm_silence.yaml` (restart-safe "Silence this camera"). **Add one
     `input_datetime` block per genai camera**, named `vlm_silenced_until_<camera>` to match
     your Frigate camera names. (Missing helper = silence no-ops for that camera; notifications
     still work.)
4. **Restart Home Assistant** (packages + custom templates need a restart, not just a reload).
5. **Verify the preprocessor** — see "Test the preprocessor" below.
6. **Blueprint** — copy `Stable_VLM.yaml` to
   `/config/blueprints/automation/frigate_vlm/Stable_VLM.yaml` (HA auto-detects it under
   Settings → Automations → Blueprints), or import by URL once the fork is pushed.
7. **One automation per camera** — create from the blueprint: pick the camera, set your usual
   notify device/group + Base URL, turn **Enable VLM Alerts** on, paste that camera's `vlm_rules`,
   and leave **Suppress Baseline Notifications** off until you trust the rules.

> **Attachment for VLM alerts — use an _event_-keyed image.** VLM alerts are
> object-level events, so they have an object/event id but **no review id**. Choose an
> **event/object-keyed** attachment (e.g. *Event GIF* / *Object Event GIF*, snapshot)
> rather than *Review GIF* — `review_preview.gif` is keyed by review id and will 404 on
> VLM alerts. The VLM branch honors whatever you select in **Initial Attachment**; this is
> just about picking an option that can resolve for object-level events.

> **Acknowledge button needs phone→HA reachability.** Receiving a push works over the
> cloud, but **tapping an action button POSTs an event back to HA**. If the phone can't
> reach your HA URL at tap time (e.g. VPN/Tailscale disconnected), the action silently
> fails (Android later shows an "unable to send" toast) and the siren won't stop. Ensure
> the companion app's URL is reachable (VPN connected, or a reachable external URL).

## "Silence this camera" (restart-safe)

The notification action **Silence this camera** pauses that camera's VLM alerts for
`silence_timer` minutes. Rather than disabling the automation (which breaks if HA restarts
mid-window — the old handler could leave a camera permanently off), it records a per-camera
timestamp in `input_datetime.vlm_silenced_until_<camera>` and the VLM notify gate suppresses
alerts while `now()` is before it. `input_datetime` persists across restarts and the gate is
purely time-based, so silence survives a restart and still expires correctly even if HA was
down past the expiry. Requires the helpers in `packages/vlm_silence.yaml` (one per camera).

## VLM Rules: inline vs file-backed (your choice — no blueprint change)

The blueprint's **VLM Rules** input is a template field. Two ways to supply rules:

**Option 1 — Inline.** Paste the full `{% if ... %}` template directly into the input.
Self-contained; edit in the automation UI.

**Option 2 — File-backed.** Keep rules in `custom_templates/frigate_vlm_rules.jinja`
as a per-camera macro, and set the input to a one-liner:
```jinja
{% from 'frigate_vlm_rules.jinja' import my_camera_rules %}{{ my_camera_rules(vlm, gate, vlm_zones, camera_name) }}
```
(See `examples/frigate_vlm_rules.example.jinja` for an anonymized starting point.)
Pros: rules live in version control next to the other macros, shared helpers, no UI
editing. Con: editing the file requires a **full HA restart** (custom_templates are
only re-read on restart), whereas inline edits apply on automation save.

Both produce the same `{notify, priority, title, message}` dict. Macros do NOT inherit
caller variables, so the scope (`vlm, gate, vlm_zones, camera_name`) is passed explicitly.

## Test the preprocessor

1. HA → Settings → Devices & Services → MQTT → Configure → **Listen to a topic**:
   `frigate_vlm/#`
2. Trigger a `person` event on a genai camera; wait ~10–30s for the description.
3. Expect one clean message on `frigate_vlm/events/<camera>` with `schema_version: 3`
   and booleans as real booleans.

## Status / roadmap

- [x] Schema v3 — describe-first (`scene_observation` + `people[]`), grounding (`person_present`/`likely_false_detection`), people[] reconciliation, no identity
- [x] Preprocessor (this folder)
- [x] Blueprint VLM branch — `blueprints/automation/frigate_vlm/Stable_VLM.yaml` (tailored)
- [x] Example rules (pool, gate, suspicious, delivery, package theft, balcony) with people[] reconciliation — anonymized template in `examples/frigate_vlm_rules.example.jinja` (real deployment file is gitignored)
- [x] Per-camera baseline-suppression toggle (`vlm_suppress_baseline`)
- [x] Restart-safe "Silence this camera" (`packages/vlm_silence.yaml` + time-based notify gate)
- [ ] Production soak (~1 week), then generalize + upstream PR to SgtBatten/HA_blueprints

## Generalization checklist (execute as the FINAL step, before the upstream PR)

The current build is **intentionally tailored** for fast iteration and easy debugging.
None of this is wasted — it's a refactor, not a rewrite, and the contract / `schema_version` /
single-`vlm_rules`-template decisions were all made to keep generalization cheap.

- [ ] **Inline the macros** — blueprints cannot import from `custom_templates`. Embed
      `parse_description` + `vlm_clean_json` directly in the blueprint, or push coercion into
      the documented `rest_command` helper.
- [ ] **Parameterize the Frigate API URL** — replace the `<FRIGATE_HOST>` placeholder with a
      config/input value.
- [ ] **Ship the camera-lookup as a documented `rest_command` snippet** — blueprints can't do
      HTTP or hold state, so the generic release is *blueprint + one `rest_command` block users
      paste into `configuration.yaml`*. (This disappears entirely if a future Frigate adds
      `camera` to `tracked_object_update` — check on 0.17+.)
- [ ] **Make the gate configurable** — required fields + confidence/quality thresholds as
      inputs, not a hardcoded `confidence`+`image_quality` assumption.
- [ ] **Make the VLM source topic + JSON paths inputs** — default to `frigate_vlm/events`
      (bridge) but allow pointing straight at Frigate's native topic.
- [ ] **Full notify parity** — the tailored vlm branch reuses the blueprint's dispatch; confirm
      telegram/TV/group all carry through for the generic version.
- [ ] **Docs + example `vlm_rules`** in the blueprint dir.
- [ ] **Soak ~1 week on real cameras first** — validates rule ergonomics and sane defaults
      before freezing the public config surface.
