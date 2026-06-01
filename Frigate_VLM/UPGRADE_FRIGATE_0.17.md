# Frigate 0.17 — upgrade impact on the VLM alerting pipeline

Research + analysis of how upgrading the Frigate **server** (currently 0.15.2-3bda638)
to **0.17.x** affects this project. Captured 2026-06-01.

**TL;DR:** the core data path is unchanged, so the preprocessor / blueprint / rules
need **zero code changes** for the upgrade. The only required work is a **Frigate-side
config migration** (per-camera `genai.*` keys move under `genai.objects.*`). Skipping it
silently kills GenAI → all VLM alerts go dark while baseline alerts keep working.

---

## ✅ Core data path is UNCHANGED (verified against 0.17 docs)

The entire pipeline hinges on the per-object description MQTT message. In 0.17 it is
**identical** to 0.15:

- **Topic:** `frigate/tracked_object_update`
- **Payload:**
  ```json
  { "type": "description", "id": "1607123955.475377-mxklsc",
    "description": "The car is a red sedan moving away from the camera." }
  ```

So `❷ preprocessor`, the `frigate_vlm/events/<camera>` contract, and all rule macros
keep working. This is the payoff of isolating Frigate behind a stable contract — the
0.17 breaking change is config-side, not data-side.

`frigate/reviews` and `frigate/events` structures: no documented changes.
`/api/events/<id>` (used by the preprocessor for id→camera resolution): no documented changes.

---

## ⚠️ BREAKING CHANGE on upgrade — GenAI config restructure

> "The global `genai` config now only configures the provider. Other fields have moved
> under `objects -> genai`."

**Current (0.15) per-camera shape:**
```yaml
cameras:
  <camera>:
    genai:
      enabled: true
      use_snapshot: true
      objects: [person]
      required_zones: [<zone_a>, <zone_b>, ...]
      prompt: |
        ...
```

**0.17 shape (move everything except provider under `genai.objects`):**
```yaml
# top-level: provider only
genai:
  provider: gemini
  api_key: ...
  model: gemini-2.5-flash

cameras:
  <camera>:
    genai:
      objects:
        enabled: true
        use_snapshot: true
        objects: [person]
        required_zones: [<zone_a>, <zone_b>, ...]
        prompt: |
          ...
```
> Exact nesting must be confirmed against the 0.17 docs at migration time — treat the
> above as the shape, verify the precise keys before applying.

**Failure mode if skipped:** GenAI silently stops generating descriptions →
`frigate/tracked_object_update` goes quiet → every VLM alert disappears, while baseline
SgtBatten alerts keep firing. This is the SAME failure signature as the brace-crash and
the alert-zone bugs we already hit — easy to misdiagnose. After migrating, re-capture
`frigate/tracked_object_update` to confirm descriptions still flow.

---

## 🎯 New 0.17 features relevant to this project

1. **Native review summaries + structured classification (dangerous / suspicious / normal).**
   Frigate itself now generates a title + description + threat class per *review item*
   (distinct from our per-object descriptions).
   - The blueprint's dormant **`genai-event` branch** (reads `metadata.title` /
     `shortSummary` / `potential_threat_level`) was built for exactly this and never
     fires on 0.15. On 0.17 it **may start firing** — be aware it could activate and
     produce notifications we haven't tuned. Check/observe after upgrade.
   - Strategic: some suspicious-activity rules could eventually lean on Frigate's native
     classification instead of custom prompt fields. Not urgent — our per-object
     structured JSON is richer — but relevant for the generalization/PR.

2. **Per-camera GenAI toggle via MQTT:**
   - `frigate/<camera>/object_descriptions/set` (ON/OFF) + `/state`
   - `frigate/<camera>/review_descriptions/set` (ON/OFF) + `/state`
   Operational lever — e.g. disable description generation per camera on a schedule via
   an HA automation, complementing the rules.

3. **Semantic Search Triggers** — Frigate-native "do X when an object matches this
   description/image." Conceptual overlap with our rule engine but runs inside Frigate;
   not a replacement (can't drive the escalating HA siren/notification/ack logic). Good
   to know it exists.

4. **YOLOv9 on Coral** — directly relevant to **small-child detection**. "Improved
   accuracy over the default mobiledet model" on USB Coral. Could let you raise the person
   `threshold`/`min_score` back up (fewer false positives) while still detecting a small
   child — a better fix than a lowered-threshold workaround (e.g. `min_score: 0.4 /
   threshold: 0.6`) on a pool/safety camera. Worth A/B testing post-upgrade.

Other 0.17 items (not directly relevant): CUDA Graphs / Intel NPU / Apple Silicon
detector performance, state vs object classification models.

---

## Recommended upgrade procedure (when ready — not mid-soak)

The preprocessor / blueprint / rules need NO changes. Only Frigate-side config does.

1. **Back up** the current working Frigate config.
2. **Migrate** every camera's `genai.*` → `genai.objects.*`; move provider to global `genai:`.
   Keep prompts byte-identical (incl. the doubled `{{ }}` braces and schema_version v2 fields).
3. **Restart Frigate**, trigger a person event, and **re-capture** `frigate/tracked_object_update`
   — confirm a structured description with `schema_version: 2` still arrives.
4. Confirm a `frigate_vlm/events/<camera>` message still lands (preprocessor end-to-end).
5. **Watch the dormant `genai-event` branch** — if native review summaries now publish in
   a way the blueprint's genai branch consumes, decide keep vs. suppress.
6. **Test YOLOv9 on Coral** for child detection; if better, re-raise person thresholds.
7. Decide: adopt native review classification, or keep custom per-object prompts (likely keep).

## Sources
- Frigate 0.17.0 release — https://github.com/blakeblackshear/frigate/releases/tag/v0.17.0
- 0.17.0 release discussion #22137 — https://github.com/blakeblackshear/frigate/discussions/22137
- Frigate MQTT integration docs — https://docs.frigate.video/integrations/mqtt/
