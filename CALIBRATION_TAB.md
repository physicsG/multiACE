# multiACE calibration tab

Status: Calibration and verification implementation present in the working tree
Target UI: `multiace/web/frontend`
Target firmware: `multiace/klipper/extras/ace.py`
Mockup: [`multiace/web/mockups/calibration-tab.html`](multiace/web/mockups/calibration-tab.html)
Engineering reference: [`FILAMENT_MOTION_FEATURE_REFERENCE.md`](FILAMENT_MOTION_FEATURE_REFERENCE.md)

## Current implementation

The first functional slice is present in the working tree:

- the live web UI has a **Calibration** tab between Dashboard and Config;
- connected-unit, unit/slot scope, route, readiness, jog, checkpoint, review, and config-apply controls are wired to backend endpoints;
- clean-slate preparation reads the selected toolhead's actual extruder sensor, unloads only that toolhead when necessary, then requires a separate user confirmation; the unload has a direct, out-of-band cancel path rather than a queued cancel macro;
- Klipper exposes a guarded, cancellable calibration state machine with bounded feed/retract movement;
- the existing ACE → splitter → toolhead diagram resolves effective configured anchors immediately, then tracks the estimated filament tip directly on its two tube segments;
- a non-heated verification round trip supports manual pause/resume and automatically holds at the toolhead sensor and measured splitter checkpoint;
- ACE 2 Pro decoder span is sampled as supporting telemetry, while ACE v1 is explicitly labelled as a commanded-distance estimate;
- applying a completed result uses the existing conflict-aware config save/backup path.

This slice still requires printer hardware validation of the new verification controls. Decoder-to-millimetre conversion remains future work, so displayed in-motion positions are timing/command estimates and the splitter checkpoint still requires visual confirmation.

## Summary

Add a **Calibration** tab between **Dashboard** and **Config**. It lists every connected ACE, lets the user calibrate one unit or one slot, measures the feed path from a fresh automatic-preload reference to the toolhead sensor, and provides one manual retract checkpoint:

1. filament has cleared the Y/4-way splitter.

After that checkpoint, the remaining retract is automatic: cumulative retract stops when it equals the measured outward load distance, returning to the same ACE automatic-preload reference without requiring a view inside the ACE.

The resulting values map directly to multiACE's existing configuration hierarchy:

| Measurement | Unit-level key | Slot-level key | Meaning |
| --- | --- | --- | --- |
| Load distance | `[ace N] load_length` | `[ace N] load_length_S` | Post-preload position to toolhead sensor |
| Swap retract | `[ace N] swap_retract_length` | `[ace N] swap_retract_length_S` | Total retract from the toolhead sensor until the tip safely clears the shared junction |
| Preload-reference return | `[ace N] retract_length` | `[ace N] retract_length_S` | Same raw distance as the measured outward trip, returning to the fresh automatic-preload reference |

`N` and `S` are zero-based in the config, regardless of `display_index_base`.

This is feasible on both protocol generations, but the quality of automatic measurement differs:

- **ACE 2 Pro / protocol v2:** preferred. It supports start/stop feed commands and `GET_FEED_INFO` telemetry (`steps`, `length`, and a hardware decoder value). multiACE already reads this decoder and describes it as real filament movement. Calibration can sample it throughout a move, detect slip, and improve the measured value.
- **ACE Pro / protocol v1:** usable with incremental commanded moves and the U1 toolhead sensor. V1 exposes start/stop motion but no equivalent `feed_info` telemetry in multiACE. The result is therefore a commanded-distance estimate, not an encoder-confirmed measurement. The UI should label it **estimated** and recommend two runs.

## Why insertion already moves filament

Putting filament into an ACE inlet is a separate **preload** operation owned by the ACE firmware. An inlet/gate detection event starts the slot motor and the ACE moves the filament to its internal ready position. The official Anycubic manuals describe this as automatic pre-feeding after filament is detected. ACE 2 Pro reports the slot state as `preloading` over protocol v2.

That means multiACE does not need to know the distance from the spool to the ACE's internal gate. Its configured `load_length` begins at the repeatable, post-preload position and covers only the external route to the printer/toolhead sensor.

The automatic preload position is the calibration origin, `P0`. The absolute distance travelled inside the ACE before `P0` is deliberately unknown and cancels out: calibration measures outward from `P0` and returns by the same raw distance. `feed_length` must remain `0`, otherwise an additional multiACE feed shifts the origin after automatic preload.

The exact ACE 2 Pro preload stop rule is not publicly documented. The available evidence shows:

- a firmware-controlled `preloading` state;
- four independent feed motors and a buffering mechanism;
- per-slot feed telemetry including motor steps, requested length, and a decoder/filament-motion reading;
- automatic preload starting after insertion is detected.

The safest conclusion is that the ACE's own sensors and motor/decoder feedback control the internal preload. It is not calculating the user's external Bowden length at insertion time. External travel only starts when the printer later requests a feed of a specified length.

Sources:

- [Anycubic ACE 2 Pro product page](https://store.anycubic.com/products/ace-2-pro)
- [Anycubic Kobra 4 Combo manual (ACE 2 Pro)](https://wiki.anycubic.com/k4/Kobra%204%20Combo%20%E7%94%A8%E6%88%B7%E6%89%8B%E5%86%8C-EN-20260407.pdf)
- [Anycubic Kobra S1 Combo manual](https://wiki.anycubic.com/kobra-s1/anycubic_kobra_s1_combo%E7%94%A8%E6%88%B7%E6%89%8B%E5%86%8C-v1.3.pdf)
- multiACE v2 protocol implementation: `multiace/klipper/extras/ace_protocol_v2.py`

## User experience

### Tab layout

The Calibration tab contains:

1. a readiness banner (printer idle, no swap/background operation, sensors reachable);
2. cards for connected ACE units only;
3. a calibration scope selector: **Entire ACE** or **Single slot**;
4. the resolved physical route, for example `ACE 1 · Slot 2 → Toolhead T2`;
5. a seven-step wizard beginning with an explicit clean-slate baseline;
6. live measured/commanded distances and sensor status;
7. a config diff and **Apply to config** action.

### Filament-tip route indicator

The route is a topology diagram, not a physically scaled Bowden tube. It does not add a second progress rail: the filament marker is rendered directly on the existing ACE → splitter and splitter → toolhead lines.

On opening the tab, resolve the physical ACE-to-toolhead anchor from effective `retract_length` and the junction offset from effective `swap_retract_length`, using normal override precedence:

1. selected per-slot `*_S` override;
2. selected `[ace N]` value;
3. global `[ace]` default.

`load_length` is not used as the configured physical coordinate because it may include a feed allowance beyond the sensor edge. During a new measurement, the raw measured load distance and raw retract distance are equal, so that raw value becomes the live toolhead anchor.

This means an already-calibrated route is useful immediately and does not regress to “unknown” merely because a new calibration session has not started. New measurements replace the corresponding configured anchor as soon as they become available. A question mark is reserved for a genuinely missing/invalid configured route or an indeterminate physical position.

Once calibration has established both anchors, use the fresh ACE preload as coordinate zero:

```text
ACE preload P0       = 0
splitter Ps          = load_length - swap_retract_length
toolhead sensor Pt   = raw load distance (active measurement)
                       or effective retract_length (saved route)
```

Positions from `P0..Ps` map onto the existing ACE → splitter line; positions from `Ps..Pt` map onto the existing splitter → toolhead line. Each line uses its own local percentage. This keeps the marker physically meaningful even though the two real tube sections are not drawn to scale.

During motion the firmware publishes `tip_position_mm`. It is derived from completed commanded movement plus elapsed movement in the active bounded move. The UI labels it **estimated** and updates the marker continuously for moving, manually paused, and endpoint-recovery states. Sensor activation is authoritative at the toolhead endpoint; user observation is authoritative when measuring the splitter checkpoint.

Disconnected configured units can appear in a collapsed “Unavailable” area, but cannot be calibrated.

### Scope

**Entire ACE** uses a representative slot/path selected by the user and writes `[ace N]` defaults. This is appropriate when all four routes are physically the same.

**Single slot** writes `*_S` overrides under `[ace N]`. This is the recommended scope when:

- tubes have different lengths;
- a 4-way splitter has unequal branch lengths;
- routing or friction differs per spool position;
- the ACE feeds different toolheads or different splitter inputs.

The UI should never describe slot calibration as spool-material calibration. The value follows the physical **slot/path**, even when the spool is replaced.

### Route resolution

- In `multi` mode, default to the normal slot-to-head route already used by multiACE. Allow the user to confirm the resolved toolhead.
- In `head` mode, use the ACE-driven head configured by `head_ace_*`; all slots may share that head through a combiner.
- When exactly one ACE is connected, select it automatically. Resolve its designated ACE-driven toolhead from `ace_heads` plus `head_ace` and select that head instead of defaulting to T0.
- Select the first slot whose live state is `ready`. Preserve an explicit user selection afterward rather than continually snapping back as status refreshes arrive.
- In `normal` mode, show the tab but require switching to `multi` or `head` before starting because there is no active ACE route.
- If a route is ambiguous, require explicit toolhead selection. Do not guess and move filament.

## Calibration wizard

### 1. Establish a clean slate

Preconditions:

- printer is idle (not printing or paused);
- no swap, background load, load, unload, or feed-assist operation is active;
- no other calibration session exists.

The wizard must deliberately establish a known empty baseline before choosing the measured path:

1. Inspect `filament_at_extruder` for the toolhead resolved by the selected calibration route. This is the `eN_filament` toolhead sensor used by calibration. Do not use the generic `filament_detected` field here: that field reflects the upstream feed port and may remain true while filament is preloaded in the ACE but the toolhead is empty.
2. If that toolhead is occupied, offer **Unload selected toolhead** using `ACE_UNLOAD_HEAD HEAD=N CALIBRATION_PREPARE=1 ACE=A SLOT=S`. The explicit ACE/slot route is mandatory for preparation. Firmware validates it against the configured topology, gate state, any tracked source, and other loaded routes; it must not fall back to the currently active ACE when `head_source` is unknown.
3. Wait until that workflow finishes and re-read the selected toolhead sensor; do not trust only the command return.
4. Require the selected sensor to report explicitly clear. An unavailable/unknown sensor is not equivalent to empty.
5. Ask the user to remove any loose filament left in the output tube, Y/4-way splitter, or toolhead entrance.
6. Clear a cancelled/failed calibration session and stale route selection before accepting the baseline.
7. Once the sensor is clear, present **Use clear TN as starting point**. Record the clean-baseline acknowledgement only when the user clicks it; completion of an automatic unload must not silently confirm the baseline.

The status copy is stateful rather than a permanently positive checklist label:

- **Selected TN: toolhead sensor is clear** when `filament_at_extruder == false`;
- **Selected TN: filament detected at toolhead sensor** when it is `true`;
- **Selected TN: toolhead sensor state unavailable** when it is absent or unknown.

After a successful unload the wizard stays on step 1 with the sensor shown clear and the clean starting point awaiting confirmation. This keeps sensor verification (automatic evidence) distinct from the user's acknowledgement that the visible splitter/output route is actually clean.

The selected-toolhead unload must be directly cancellable. Do not enqueue a
cancel G-code behind `ACE_UNLOAD_HEAD`: Klipper cannot execute it until the
blocking unload has already returned. Preparation uses a Klipper webhook that
sets a cooperative cancel flag out-of-band, immediately stops ACE feed/rollback
and feed assist, turns off the selected heater, and aborts at cancellation
checkpoints throughout homing, heating, tip forming, probing, and retract.
After cancellation the position is treated as unknown and the clean-slate step
must be repeated.

Changing the ACE, slot, toolhead, or scope invalidates this acknowledgement and returns the wizard to this step. Cancelling a calibration does the same because filament may be left between checkpoints.

This step does not eject spools from the ACE. Filament should remain captured at the ACE's normal internal ready/parked position.

### 2. Prepare the selected route

Preconditions:

- selected ACE is connected and ready;
- selected slot is not empty (`gate_status == available`);
- selected toolhead sensor is present and currently clear;
- selected head is ACE-driven and not in manual/TPU or stock-feeder mode;
- the clean-slate acknowledgement from step 1 is still valid.

Instructions:

1. Remove and reinsert filament in the selected ACE inlet so the reference is fresh.
2. Wait for automatic ACE preload to complete and the slot to return to `ready`.
3. Confirm that only the selected slot enters the displayed physical route.
4. Manually verify that the splitter/output tube is clear and the displayed toolhead is correct.
5. Record an explicit **Use ACE preload position and confirm route** acknowledgement.

The **Start calibration** button becomes active only after both preparation steps pass.

### 3. Measure load distance

Feed at a conservative calibration speed. Do not call `ACE_LOAD_HEAD` or `FEED_AUTO LOAD=1`, because those flows can select a tool, heat the nozzle, seat filament, purge, and mutate material bookkeeping.

Recommended control loop:

1. Disable feed assist for the selected ACE/slot.
2. Start with one bounded 500 mm move, followed by 100 mm increments until the sensor is reached or the guarded maximum is exhausted.
3. Poll the selected U1 toolhead sensor between increments and while a v2 move is active.
4. Stop the ACE immediately when the toolhead sensor becomes active.
5. Record cumulative commanded movement.
6. On v2, continuously sample `get_feed_info` during the move and store decoder min/max/span. Do not rely only on a before/after decoder delta: the current code notes that the value may reset or hold between commands.
7. Mark the result as `encoder-confirmed`, `encoder-warning`, or `estimated`.

The feed command must still have a hard maximum (default: current effective `load_length` plus a guarded allowance, never an unlimited feed). If that ceiling is reached without the toolhead sensor, stop and fail safely.

The raw sensor-edge measurement should be retained separately from the applied value. If a load safety allowance is desired, show it explicitly:

```text
sensor edge       2068 mm
load allowance      20 mm
configured value  2088 mm
```

Do not silently add a hidden allowance.

### 4. Mark the swap-clear position

With filament sitting at the toolhead sensor, retract in user-selected jog steps: `−5`, `−10`, `−25`, `−50`, `−100`, `−250`, and `−500 mm`.

After each completed step, update:

- cumulative commanded retract;
- v2 decoder movement/span and slip warning;
- toolhead sensor state;
- ACE slot state;
- a prominent **Stop** control.

The user visually checks the Y/4-way splitter. When the filament tip has fully passed the junction, they click **Mark splitter cleared**. Store the cumulative distance at that instant as the raw swap checkpoint.

Offer an explicit clearance allowance field. The final value is:

```text
swap_retract_length = raw splitter checkpoint + user-visible clearance allowance
```

The UI should recommend—not force—a small allowance and explain that too little can collide with the next filament, while too much slows every swap.

### 5. Return to the automatic-preload reference

After the splitter checkpoint, the user clicks **Return to preload reference**. The firmware automatically retracts the remaining bounded distance until cumulative retract exactly equals the raw sensor-edge load measurement. No visual inspection inside the ACE is requested or implied.

`retract_length` is the **total cumulative retract from the toolhead sensor**, not the extra distance after the splitter checkpoint:

```text
raw retract_length = raw load sensor-edge distance
raw retract_length > raw swap checkpoint > 0
```

The load allowance is not added to `retract_length`: normal loading stops on the toolhead sensor, so the return must mirror the raw movement that actually reached that sensor. The firmware rejects manual jogs that would pass the measured preload reference.

On ACE 2 Pro, compare accumulated outward and return decoder spans and expose their delta as diagnostics. Decoder telemetry checks symmetry/slip; it does not establish an absolute tip coordinate. ACE v1 uses commanded-distance symmetry and should be repeated to assess spread.

### 6. Verify the calibrated route

After calibration, offer one non-heated round trip before Save:

1. Start at the known ACE automatic-preload coordinate (`0 mm`).
2. Issue one bounded move for the complete calibrated load distance while continuously polling the selected toolhead sensor and moving the route marker.
3. Stop immediately when the toolhead sensor activates and wait for explicit **Continue to splitter** confirmation. Do not reverse automatically.
4. After confirmation, issue one bounded retract for the complete toolhead-to-splitter distance and stop at `load_length - swap_retract_length`.
5. After the user confirms the splitter checkpoint, issue one bounded retract for the complete remaining splitter-to-preload distance. Confirm that the toolhead sensor clears and report verification complete only after the ACE reports ready at coordinate zero.
6. Permit **Pause verification** during every moving leg. Stop the ACE directly, retain the estimated partial-move coordinate, and change the control to **Resume verification**. Resuming commands only the remaining distance.
7. Keep **Stop & cancel** available independently of pause. Cancellation invalidates the clean-slate acknowledgement because the physical position may no longer match the parked reference.
8. On v2, collect decoder span separately for the verification round trip so verification telemetry does not alter the original calibration comparison.

The normal verification path therefore contains three continuous movement commands: preload to toolhead, toolhead to splitter, and splitter to preload. Both the toolhead and splitter are explicit user-confirmed holds, but there are no automatic 500/100 mm bursts within a leg. A manual pause position is explicitly approximate because the ACE can travel slightly between the stop request and physical deceleration.

Verification is deliberately separate from measurement so a mistaken checkpoint does not immediately become persistent configuration. Applying without verification remains possible, but a completed round trip is shown as **Verified** and can be repeated before saving.

After a restart, a saved route may bypass the measurement acknowledgements when live state already proves the safe starting condition: the selected slot is `ready` at ACE preload, the mapped toolhead sensor is explicitly clear, the printer is idle, and no swap/background movement is active. A prominent **Verify parked route** action primes a transient verification session from effective `retract_length` and `swap_retract_length`; it does not overwrite configuration or require another measurement pass. If those live conditions are not satisfied, the button remains disabled and the normal preparation workflow remains available.

Every verification start, including **Verify again**, submits the ACE, slot, toolhead, and scope displayed in the selector. A completed session may be reused only when all four values still match. If the displayed route changed while parked, firmware primes a new transient session from that route's effective saved values instead of silently reusing the previous slot.

When `tip_position_mm > 0`, or verification/calibration is moving, manually paused, or held at the toolhead/splitter, lock the ACE, slot, scope, and toolhead selectors to the session route. Klipper must enforce the same rule and reject a different verification route even if requested outside the UI. This prevents a second spool entering an occupied Bowden/splitter path. Unlock only after the route returns to ACE preload or the selected route is explicitly cleared through preparation.

Installing updated frontend files restarts the web service, but copying a new `ace.py` does not replace the Python class already loaded inside Klipper. After an upgrade that adds calibration G-code behavior, restart Klipper or reboot the printer before using the new verification action. If an old runtime responds with “complete calibration before verification,” the UI translates that into a restart instruction.

### Verification fine positioning

Do not fail verification immediately when the saved toolhead endpoint is reached while the toolhead sensor is still clear. Stop the bounded automatic move and enter `verify_toolhead_adjust` instead. This is a recovery hold, not unrestricted manual control.

The verification panel then exposes two clearly labelled rows:

- **Toward toolhead:** `+1`, `+2`, `+5`, `+10 mm`;
- **Toward ACE:** `−1`, `−2`, `−5`, `−10 mm`.

Safety and UX rules:

1. Use the same conservative calibration speed and allow only one bounded jog at a time.
2. Limit cumulative correction to ±50 mm around the saved endpoint. Reaching the limit without a sensor transition fails with an actionable route/friction warning.
3. Poll the selected toolhead sensor throughout positive movement and stop the ACE immediately when it activates; do not wait for the remainder of that jog.
4. Keep **Stop & cancel** directly available. Disable other jog buttons while a jog is active.
5. Update the inline route marker and show the exact signed correction beside it. The marker may remain visually held at the toolhead node when the correction extends beyond the drawn route.
6. Once the sensor trips, transition to the normal `verify_toolhead` hold and wait for explicit **Continue to splitter** confirmation.
7. Record the signed endpoint correction in the verification result. Do not silently add it to `load_length` or `retract_length`; show it during review so the user can deliberately recalibrate or apply a future correction workflow.

The production firmware, API, and calibration tab implement this state as `verify_toolhead_adjust` with `verify_toolhead_adjusting` while a jog is active. The interactive mockup demonstrates the same flow with a saved endpoint that stops 7 mm short.

### 7. Review and apply

Show:

- scope and exact config keys;
- previous effective value;
- measured raw value;
- explicit allowance;
- proposed saved value;
- measurement confidence;
- validation result.

**Apply to config** uses the existing config read/merge/write path, including the SHA-1 conflict check and `.bak` creation. It writes either:

```ini
[ace 0]
load_length: 2088
swap_retract_length: 872
retract_length: 1915
```

or:

```ini
[ace 0]
load_length_2: 2110
swap_retract_length_2: 895
retract_length_2: 1942
```

The tab should then show the existing full-printer-restart banner. The MVP does not need to mutate the loaded Python values in place.

## Capability and confidence matrix

| Capability | ACE Pro / v1 | ACE 2 Pro / v2 |
| --- | --- | --- |
| Detect filament present in slot | Yes | Yes |
| Automatic inlet preload | Yes | Yes; reported as `preloading` |
| Command bounded feed/retract | Yes | Yes |
| Stop an active feed | Yes | Yes |
| Stop on U1 toolhead sensor | Yes, coordinated by multiACE | Yes, coordinated by multiACE |
| Read actual per-slot movement | Not exposed by current v1 integration | `get_feed_info` decoder telemetry |
| Automatic load measurement | Incremental commanded-distance estimate | Encoder-assisted, subject to hardware validation |
| Manual splitter checkpoint plus automatic preload-reference return | Yes | Yes |
| Slip diagnostics | Indirect/timeouts only | Decoder versus commanded movement |

The tab should not be restricted to ACE 2 Pro. Instead, display the confidence badge and adapt the validation policy.

## Firmware design

### State ownership

Keep the authoritative calibration session in `ace.py`, not only in the browser or FastAPI process. A page refresh or WebSocket reconnect must not orphan an active motor operation.

Suggested status object:

```python
calibration = {
    "session_id": "...",
    "state": "idle|prepared|feeding|at_sensor|retracting|retract_ready|swap_marked|returning|complete|failed|cancelled",
    "ace": 0,
    "slot": 2,
    "head": 2,
    "scope": "ace|slot",
    "protocol": "v1|v2",
    "commanded_feed_mm": 0,
    "commanded_retract_mm": 0,
    "decoder_span": None,
    "feed_decoder_span": None,
    "return_decoder_span": None,
    "decoder_return_delta": None,
    "park_reference": "ace_preload",
    "toolhead_sensor": False,
    "swap_retract_length_mm": None,
    "retract_length_mm": None,
    "error": None,
}
```

Expose it from `ace.get_status()` so the existing WebSocket state stream updates the tab.

### Suggested G-code surface

Use narrow commands with state validation on every transition:

```text
ACE_CALIBRATION_START ACE=0 SLOT=2 HEAD=2 SCOPE=slot
ACE_CALIBRATION_FEED
ACE_CALIBRATION_RETRACT LENGTH=25
ACE_CALIBRATION_MARK MARK=swap
ACE_CALIBRATION_RETURN
ACE_CALIBRATION_VERIFY ACTION=start
ACE_CALIBRATION_VERIFY ACTION=jog LENGTH=5
ACE_CALIBRATION_VERIFY ACTION=continue
ACE_CALIBRATION_CANCEL
ACE_CALIBRATION_RESET
```

`ACE_CALIBRATION_CANCEL` must be accepted from every non-idle state.

### Suggested HTTP surface

The FastAPI layer remains a thin Moonraker adapter:

```text
GET  /api/calibration
POST /api/calibration/start
POST /api/calibration/action
POST /api/calibration/apply
```

Example start body:

```json
{"ace": 0, "slot": 2, "head": 2, "scope": "slot"}
```

Example action body:

```json
{"session_id": "...", "action": "retract", "length": 25}
```

The backend should reject stale session IDs and must never construct unrestricted G-code from arbitrary action strings.

## Safety requirements

- A single global calibration lock; no concurrent calibration, swap, load, unload, dryer motion, or background operation.
- Calibration disabled while printing or paused.
- Start requires a freshly confirmed baseline in which the selected calibration toolhead sensor reports clear; unknown sensor state is not clear.
- Every move is bounded by a configured maximum and a deadline.
- A visible and always-enabled **Stop & cancel** button. Preparation unload
  cancellation must bypass the G-code queue through the dedicated Klipper
  webhook; it may not be implemented as the next queued macro.
- Cancel stops feed/retract, disables feed assist, restores sensor enable state, restores the previously active ACE, and leaves the filament captured.
- Toolhead sensor edge is debounced before stopping/recording.
- Reject starting when the selected toolhead sensor is already active.
- Reject a slot that becomes empty during the operation.
- Stop on ACE disconnect, protocol error, stalled decoder, or page-independent watchdog timeout.
- Never heat or extrude during calibration; the filament should only reach the toolhead sensor.
- Do not update `head_source`, RFID/material identity, `print_task_config`, or snapshot state.
- Do not save automatically. Persist only after review and successful validation (or an explicit expert override with a warning).

## Config precedence and form integration

The existing lookup order is already correct:

1. `[ace N] key_S`;
2. `[ace N] key`;
3. global `[ace] key` (with existing head/slot fallbacks for load);
4. built-in default.

After Apply, update the in-browser `configForm.perAce[N]` values too, so switching to Config immediately shows the same numbers. A later config reload remains the source of truth.

Unit-level calibration should not delete existing per-slot overrides. Show a warning that those slots will continue to override the new unit value. Slot-level Apply changes only the selected slot.

## Implementation phases

### Phase 1: safe MVP

- Calibration tab and connected-unit cards.
- Clean-slate preparation action, sensor recheck, and explicit empty-baseline/route confirmations.
- Explicit ACE/slot/head route selection.
- Incremental feed to toolhead sensor for both protocols.
- Manual splitter checkpoint followed by an automatic return to the measured preload reference.
- Commanded-distance results and v1/v2 confidence badges.
- Config diff, backup, conflict-aware write, restart banner.
- Cancel/watchdog and all precondition checks.

### Phase 2: v2 telemetry

- Continuous `get_feed_info` sampling during every move.
- Decoder span capture that accounts for reset/hold behavior.
- Slip ratio/warnings and encoder-assisted result.
- Hardware test matrix across ACE 2 Pro firmware versions.

### Phase 3: repeatability tools

- Two/three-run calibration and median/spread display.
- Per-slot “calibrate all” queue with explicit user checkpoints.
- Calibration history and last validated date/firmware version.

## Acceptance criteria

- Calibration appears between Dashboard and Config.
- Only connected ACEs can start a session.
- Calibration cannot start until the clean-slate baseline and physical route are explicitly confirmed; changing route or scope invalidates both.
- Unit and slot scopes write the correct zero-based config keys.
- Feed always stops at or before the guarded maximum and stops promptly on toolhead sensor activation.
- The splitter checkpoint records its cumulative total; the final retract automatically stops at the raw load distance.
- `retract_length == raw load_length > raw swap checkpoint` is enforced, and manual retract cannot pass the preload reference.
- Cancel works in every active step without requiring the browser to remain connected.
- Normal load/unload material bookkeeping remains unchanged.
- Applying results creates a backup and does not overwrite a concurrently edited config.
- v1 results are visibly labeled estimated; v2 results show decoder health/confidence.
- The user can complete a validation round trip before saving.

## Open hardware questions to validate

1. Confirm the unit and scaling of v2 `decoder` across current ACE 2 Pro firmware versions.
2. Confirm whether v2 `length` is the active command target, the completed distance, or a lifetime/session counter.
3. Measure how quickly STOP takes effect at 20, 50, and 80 mm/s.
4. Determine the smallest reliable incremental command on v1 and v2.
5. Measure automatic-preload repeatability per slot and material; no visual checkpoint inside the ACE is assumed.
6. Decide default load and splitter-clear allowances from repeated hardware tests; do not guess them in production.
