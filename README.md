# HA_blueprints — Frigate Notifications **+ VLM alerting** (fork)

A fork of **[SgtBatten/HA_blueprints](https://github.com/SgtBatten/HA_blueprints)** that adds a
**rule-based, VLM-driven alerting layer** on top of the excellent Frigate Camera Notifications
blueprint.

Stock Frigate notifications alert on *any* detection in a zone. This fork adds smart alerts
driven by Frigate's per-object **GenAI / VLM** scene descriptions, so you can notify on what is
actually *happening* — "unsupervised child near the pool", "person at the gate at night",
"package being taken", "delivery" — instead of "person detected, again".

It is **not** a parallel notification system: the VLM logic is wired **into** the SgtBatten
blueprint as an extra branch, reusing the same notification dispatch, tags, and actions.

> 📖 **Full architecture, the field contract, step-by-step install, and usage live in
> [`docs/README.md`](docs/README.md).** Start there. Frigate-0.17 prompt-shape changes are in
> [`docs/UPGRADE_FRIGATE_0.17.md`](docs/UPGRADE_FRIGATE_0.17.md).

## What this fork adds (vs. upstream)

| Piece | File(s) | What it does |
|---|---|---|
| **Preprocessor** | `packages/frigate_vlm.yaml` | Normalizes Frigate's async, fenced, camera-less `tracked_object_update` GenAI output into a clean per-camera MQTT contract (`frigate_vlm/events/<camera>`): strips ```` ```json ```` fences, resolves `id → camera/zones` via the Frigate API, and type-coerces values (so `"false"` stops being truthy). |
| **VLM branch in the blueprint** | `blueprints/automation/frigate_vlm/Stable_VLM.yaml` | New trigger on the contract topic; gates on `confidence`/`image_quality`; evaluates a per-camera **VLM Rules** Jinja template → `{notify, priority, title, message, siren?}`; reuses the blueprint's existing notification dispatch (shared tag replaces the baseline notification). Plus a per-camera **Suppress Baseline Notifications** toggle. |
| **Helper macros** | `custom_templates/frigate_vlm.jinja` | `parse_description`, `vlm_clean_json` (fence-strip + type coercion). |
| **Siren escalation** | `packages/frigate_vlm.yaml` | Rules can opt into an escalating, acknowledge-able camera-siren pattern (urgent / soft / scareoff profiles). |
| **Restart-safe "Silence this camera"** | `packages/vlm_silence.yaml` + blueprint | Silence records a per-camera `input_datetime` timestamp and the notify gate is time-based — so an HA restart mid-window can't leave a camera permanently disabled (the original turn-off/delay/turn-on could). |
| **v3 prompt + rule grasp** | `examples/*` | Describe-first prompts (`scene_observation` + a `people[]` array), grounding flags (`person_present`, `likely_false_detection`), conservative gate fields, and **no identity** (face/family recognition proved unreliable — alert on behaviour/safety/presence). Rules *reconcile* booleans against `people[]` to kill hallucinated alerts. |

Everything is designed around a stable `frigate_vlm/events/<camera>` contract and a
`schema_version`, so the schema can evolve without touching the pipeline.

## How it differs from the original, concretely

- The upstream blueprints under **`Frigate_Camera_Notifications/`** are kept **untouched** as the
  fork baseline (still importable below).
- The VLM work lives alongside them in `blueprints/`, `packages/`, `custom_templates/`,
  `examples/`, and `docs/` — mirroring an HA `/config` layout so each file copies to the matching
  path. See the file table in [`docs/README.md`](docs/README.md#files).
- Your real rule file (`custom_templates/frigate_vlm_rules.jinja`) is intentionally
  **gitignored** (it contains your zones/identities); start from
  [`examples/frigate_vlm_rules.example.jinja`](examples/frigate_vlm_rules.example.jinja).

## Getting started

1. Read [`docs/README.md`](docs/README.md) — architecture + the `frigate_vlm/events/<camera>` contract.
2. Set up the Frigate side from [`examples/frigate_config.example.yaml`](examples/frigate_config.example.yaml)
   (GenAI provider + per-camera describe-first prompt).
3. Install the HA side (custom template, both packages, blueprint) — install steps are in the docs.
4. Adapt [`examples/frigate_vlm_rules.example.jinja`](examples/frigate_vlm_rules.example.jinja)
   into your own `custom_templates/frigate_vlm_rules.jinja`, then create one automation per camera.

---

## Upstream baseline (unchanged)

The original SgtBatten **Frigate Camera Notifications** blueprint is included unmodified under
[`Frigate_Camera_Notifications/`](Frigate_Camera_Notifications). It sends a notification when a
Frigate event fires for the selected camera — initially the detection thumbnail, with actionable
buttons to view the clip and snapshot.

STABLE: [![Import blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/SgtBatten/HA_blueprints/main/Frigate_Camera_Notifications/Stable.yaml)

BETA: [![Import blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/SgtBatten/HA_blueprints/main/Frigate_Camera_Notifications/Beta.yaml)

Full credit for the underlying notification blueprint goes to
[SgtBatten/HA_blueprints](https://github.com/SgtBatten/HA_blueprints).
