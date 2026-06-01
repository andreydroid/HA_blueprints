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
PREPROCESSOR  (frigate_vlm_preprocessor.yaml — this folder)
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
  "schema_version": 1,
  "vlm": {                       // fully type-coerced
    "schema_version": 1,
    "person_count": 1, "adults_count": 1, "children_count": 0,
    "known_person_in_frame": true, "unknown_person_in_frame": false,
    "confidence": "high", "image_quality": "good",
    "summary": "A person is standing in the yard ..."
  }
}
```

## VLM JSON schema (the field contract)

The schema is **entirely up to your camera prompts** — the preprocessor and blueprint
never hardcode field names; rules reference whatever fields your prompt emits, always via
`default()`. The preprocessor is version-agnostic: it republishes whatever
`schema_version` the prompt emits, so you can evolve the schema without touching the
pipeline. Below is the **illustrative** field set used by the example rules; adapt freely.

**Core (recommended in every genai camera — the gate + identity basics):**
`schema_version`, `person_count`, `adults_count`, `children_count`,
`known_person_in_frame`, `unknown_person_in_frame`,
`confidence` (`low|medium|high`), `image_quality` (`poor|fair|good`), `summary`.

> `confidence` and `image_quality` must be named/valued consistently across all cameras —
> they drive the quality gate. Everything else is yours to define.

**Example pool-camera fields:** `child_supervised`, `child_in_water`, `child_near_pool_edge`.

**Example perimeter/front-camera fields:** `person_category`, `package_being_delivered`,
`package_being_taken`, `unknown_vehicle_on_property`, `pedestrian_perimeter_passable`,
`vehicle_perimeter_passable`, `non_standard_approach`, `child_unsupervised_near_exit`,
`suspicious_behavior`, `suspicious_behavior_details`, `animal_in_frame`.

**Example elevated/balcony-camera fields:** `balcony_occupied`, `child_on_balcony`,
`child_climbing_balcony`, `unknown_person_on_balcony`, `loitering_on_street`,
`person_observing_property`.

## Files

| File | Goes to | Purpose |
|---|---|---|
| `frigate-vlm.jinja` | `/config/custom_templates/frigate-vlm.jinja` | macros: `parse_description`, `safe_get`, `vlm_clean_json` |
| `frigate_vlm_preprocessor.yaml` | `/config/packages/frigate_vlm.yaml` | `rest_command` + preprocessor automation |
| `../Frigate_Camera_Notifications/Stable_VLM.yaml` | `/config/blueprints/automation/frigate_vlm/Stable_VLM.yaml` | the notification blueprint with the VLM branch |

## Install (step-by-step)

1. **Custom template** — copy `frigate-vlm.jinja` to `/config/custom_templates/frigate-vlm.jinja`
   (create the folder if needed). Custom templates are only re-read on a **full restart**.
2. **Enable packages** — in `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
3. **Preprocessor** — copy `frigate_vlm_preprocessor.yaml` to `/config/packages/frigate_vlm.yaml`.
   **Edit the `rest_command.frigate_get_event` URL** inside it: replace `<FRIGATE_HOST>`
   with your Frigate host/IP (the unauthenticated internal API, usually port 5000) —
   e.g. `http://192.168.x.x:5000`. This is a package config value, not a blueprint input,
   so it must be set here.
4. **Restart Home Assistant** (packages + custom templates need a restart, not just a reload).
5. **Verify the preprocessor** — see "Test the preprocessor" below.
6. **Blueprint** — copy `Stable_VLM.yaml` to
   `/config/blueprints/automation/frigate_vlm/Stable_VLM.yaml` (HA auto-detects it under
   Settings → Automations → Blueprints), or import by URL once the fork is pushed.
7. **One automation per camera** — create from the blueprint: pick the camera, set your usual
   notify device/group + Base URL, turn **Enable VLM Alerts** on, paste that camera's `vlm_rules`,
   and leave **Suppress Baseline Notifications** off until you trust the rules.

## VLM Rules: inline vs file-backed (your choice — no blueprint change)

The blueprint's **VLM Rules** input is a template field. Two ways to supply rules:

**Option 1 — Inline.** Paste the full `{% if ... %}` template directly into the input.
Self-contained; edit in the automation UI.

**Option 2 — File-backed.** Keep rules in `custom_templates/frigate-vlm-rules.jinja`
as a per-camera macro, and set the input to a one-liner:
```jinja
{% from 'frigate-vlm-rules.jinja' import my_camera_rules %}{{ my_camera_rules(vlm, gate, vlm_zones, camera_name) }}
```
(See `examples/frigate-vlm-rules.example.jinja` for an anonymized starting point.)
Pros: rules live in version control next to the other macros, shared helpers, no UI
editing. Con: editing the file requires a **full HA restart** (custom_templates are
only re-read on restart), whereas inline edits apply on automation save.

Both produce the same `{notify, priority, title, message}` dict. Macros do NOT inherit
caller variables, so the scope (`vlm, gate, vlm_zones, camera_name`) is passed explicitly.

## Test the preprocessor

1. HA → Settings → Devices & Services → MQTT → Configure → **Listen to a topic**:
   `frigate_vlm/#`
2. Trigger a `person` event on a genai camera; wait ~10–30s for the description.
3. Expect one clean message on `frigate_vlm/events/<camera>` with `schema_version: 1`
   and booleans as real booleans.

## Status / roadmap

- [x] Schema v1 standardized across camera prompts
- [x] Preprocessor (this folder)
- [x] Blueprint VLM branch — `Frigate_Camera_Notifications/Stable_VLM.yaml` (tailored)
- [x] Example rules (pool, gate, suspicious, delivery, package theft, balcony, family) — authored as per-camera macros; anonymized template in `examples/frigate-vlm-rules.example.jinja` (real deployment file is gitignored)
- [x] Per-camera baseline-suppression toggle (`vlm_suppress_baseline`)
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
