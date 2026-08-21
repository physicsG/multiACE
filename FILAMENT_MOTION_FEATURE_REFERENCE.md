# multiACE filament-motion feature engineering reference

This document captures reusable lessons from building Bowden-path calibration and verification. It is intended as a starting point for future features that select a spool, move filament, interpret sensors, display a route, or persist path-specific configuration.

The detailed calibration UX and state machine remain in [CALIBRATION_TAB.md](CALIBRATION_TAB.md). This reference focuses on architectural rules and failure modes that apply beyond calibration.

## 1. The core mental model

Keep four kinds of state separate:

| State | What it answers | Examples |
|---|---|---|
| Inventory | What is inserted in an ACE? | Slot ready or empty, RFID, material, colour |
| Topology | Which ACE and slot can reach which toolhead? | Mode, `ace_heads`, `head_ace`, manual or feeder heads |
| Route/session | Which path did this operation select? | ACE, slot, head, scope, session ID |
| Physical | Where is the filament tip now? | Parked, between ACE and splitter, at splitter, at toolhead sensor |

These states are related but are not interchangeable.

- A ready slot contains a preloaded spool; it is not proof that its filament occupies the external Bowden path.
- A clear toolhead sensor does not prove the splitter and Bowden path are empty.
- A selected card in the browser is user intent; it does not prove that Klipper selected the same route.
- `head_source` records the known source of a loaded head; it is not a continuously measured tip coordinate.
- Material metadata describes a spool; it does not establish that this spool is the one currently moving.

Every motion feature should build one explicit route descriptor and use it for validation, commands, status, and visualization:

```text
route = ACE index + slot index + toolhead index + operating mode/scope
```

Never reconstruct different versions of the route independently in the UI, web backend, and Klipper.

## 2. Authoritative sources and where to look

### Runtime layers

| Question | Authoritative source | Useful implementation |
|---|---|---|
| Is an ACE connected? | Klipper `ace` object | `MultiAce.get_status` in `multiace/klipper/extras/ace.py` |
| Which protocol is it using? | Per-ACE protocol instance exposed by Klipper | `aces[].protocol` in the same status payload |
| Which slots are present or moving? | ACE gate and slot status | `_gate_status_per_ace`, `_info_per_ace`, normalized as `aces[].slots[]` |
| Which head is ACE-driven? | Klipper topology configuration | `ace_heads`, `head_ace`, `head_manual`, `head_feeder` |
| Which spool loaded a head? | Klipper route history | `head_source` |
| Is filament at the toolhead? | U1 `eN_filament` motion sensor | `_calibration_sensor` and normalized `filament_at_extruder` |
| Is a print or swap active? | Klipper and related components | `print_stats`, `swap_in_progress`, `ace_bg_swap.busy` |
| Where is a calibration tip estimated? | Calibration session | `calibration.tip_position_mm` |
| What values apply to one path? | Klipper effective config lookup | `get_load_length`, `get_retract_length`, `get_swap_retract_length` |
| What metadata should be displayed? | Override, then RFID, then derived print-task data | `_parse_state` in `multiace/web/backend/main.py` |
| Is v2 filament movement observable? | ACE 2 Pro feed telemetry | `get_feed_info` decoder sampling |

The browser should normally consume the normalized `/api/state` result. The web backend obtains this from Moonraker with `/printer/objects/query` for the objects listed in `ACE_OBJECTS` in `multiace/web/backend/main.py`.

When debugging disagreement between layers, inspect them in this order:

1. Klipper `ace` object status: route, mapping, session, active device and raw ACE state.
2. `filament_feed left` and `filament_feed right`: individual toolhead sensor fields.
3. Normalized `/api/state`: what the web backend decided to expose.
4. Browser state and selected controls: what the UI rendered.
5. The exact G-code or API payload sent when motion started.

### Important source files

- `multiace/klipper/extras/ace.py`: hardware commands, topology, effective configuration, collision guards, calibration state and status.
- `multiace/klipper/extras/filament_feed_ace.py`: toolhead loading and unloading mechanics and sensor interaction.
- `multiace/web/backend/main.py`: Moonraker queries, normalization, validation, config persistence and web API dispatch.
- `multiace/web/frontend/app.js`: UI-derived state, default selection, route locking and API payload construction.
- `multiace/web/frontend/index.html`: user-visible states and available controls.
- `multiace/web/mockups/calibration-tab.html`: proposed UX that may be ahead of production behavior.
- `multiace/README.md`: installation, modes, hardware topology and operational prerequisites.

Do not treat the mockup or browser state as proof of firmware behavior. Confirm production behavior in `ace.py` and the API payload.

## 3. Indexing and route identity

Internal ACE, slot and head indices are zero-based. The UI displays them as one-based:

```text
internal ace=0 slot=0 head=3
display  ACE 1  Slot 1  T4
```

Keep conversion at presentation boundaries. Logs should state whether values are internal or display indices.

A session must store at least:

```text
session_id, ace, slot, head, scope, state, tip_position_mm
```

Every action after start must include the session ID. Every new or repeated start must include the displayed route. Klipper must compare the submitted route with the stored session before deciding to reuse it.

### Lesson: Verify again must not mean reuse blindly

A completed Slot 4 verification was once reused after the user selected Slot 1. The graph showed Slot 1 while Klipper still commanded the Slot 4 session. The general rule is:

- If submitted route and stored route match, a completed measured session may be reused.
- If they differ and the old route is parked, create a new transient session from the selected route's effective saved configuration.
- If they differ and the old route is not parked, reject the request.

The route shown in the graph must be the same route serialized into the command.

## 4. Topology and toolhead selection

multiACE supports different physical topologies.

### Multi mode

The slot maps directly to the corresponding toolhead. Calibration requires `slot == head`.

### Head mode

All slots in an ACE can share one ACE-driven head through a combiner. Resolve the toolhead using `ace_heads` and `head_ace`; do not assume T1 or derive the head from the slot number.

When exactly one ACE is connected, selecting it automatically is reasonable. The initial toolhead must still be resolved from topology. A single ACE does not imply the first toolhead.

Manual and stock-feeder heads are not ACE routes. Reject them in Klipper even if the browser accidentally offers them.

### Selection defaults

- Auto-select the only connected ACE when the user has not made a choice.
- Auto-select the first ready slot only before the user has deliberately selected a slot.
- Auto-select the mapped ACE-driven toolhead from live topology.
- Never overwrite an explicit user choice during a harmless state refresh.
- Once filament leaves preload, force the UI back to the authoritative session route.

## 5. Spool presence and ACE preload

Inserting filament starts an ACE-owned internal preload. The ACE advances filament to its own ready position before multiACE commands any external Bowden movement.

Use that automatic-preload position as coordinate zero. The distance travelled inside the ACE before this point is deliberately unknown and does not need to be measured.

### Slot availability

Klipper gate values are:

```text
-1 unknown
 0 empty
 1 available
```

The web backend additionally normalizes motion/error states such as loading, unloading, feeding, assist and error.

Protocol strings differ. ACE Pro v1 can report `empty1`, while v2 reports `empty`. Empty detection must use the shared prefix rule rather than equality with only `empty`.

Require all of the following before starting a route from preload:

- selected ACE connected;
- selected gate available;
- slot returned to ready after preload;
- selected toolhead is ACE-driven;
- selected toolhead sensor explicitly clear;
- printer idle;
- no print, swap, background load/unload, or calibration movement.

Multiple slots may legitimately be ready at the same time. Do not infer the intended motion slot from the first ready spool after the user has selected another one. Helpers such as `_first_loaded_slot_for_ace` are appropriate only for documented fallback/display behavior, not for a safety-critical explicit move.

### Metadata is not placement evidence

The display metadata priority is:

1. user slot override;
2. ACE RFID data;
3. derived print-task data;
4. empty/unknown.

This priority is useful for labels and colours. It must never decide which motor to move.

## 6. Filament detection semantics

Use `filament_at_extruder` for the selected toolhead when deciding whether the tip reached or cleared the toolhead. It corresponds to the U1 `filament_motion_sensor eN_filament` object used by calibration.

Do not substitute the generic `filament_detected` field. That field represents an upstream feed/channel condition and may remain true while filament is merely preloaded in the ACE.

Treat sensor values as three-state data:

```text
true   filament detected at the selected toolhead sensor
false  selected toolhead sensor explicitly clear
null   sensor unavailable or state unknown
```

Unknown is not clear. A missing sensor must block automatic motion that depends on it.

A clear toolhead sensor also does not mean the entire route is clear. The tip can be between the ACE, splitter and toolhead. Combine the sensor with the session coordinate and state.

The useful hierarchy is:

- Sensor transition: authoritative at the physical toolhead endpoint.
- Explicit visual confirmation: authoritative at a visible splitter checkpoint.
- Commanded distance and elapsed time: an estimate between anchors.
- v2 decoder movement: supporting evidence for motion/slip, not an absolute coordinate.

## 7. Coordinate model

Use the ACE preload position as `P0`:

```text
P0 = 0
Ps = load_length - swap_retract_length
Pt = raw sensor-edge load distance or effective saved retract_length
```

The marker maps `P0..Ps` to the ACE-to-splitter segment and `Ps..Pt` to the splitter-to-toolhead segment. Each drawn segment has its own percentage because the diagram is not physically to scale.

Saved valid anchors are sufficient to draw a parked tip at P0 after restart. Show a question mark only when anchors or physical position are genuinely unknown.

Keep raw sensor-edge measurements separate from allowances. For example, a load allowance may affect a saved load command but must not silently change the raw round-trip return coordinate.

## 8. Collision prevention invariants

Collision prevention must exist in both the UI and Klipper. UI disabling is guidance; firmware validation is authority.

### Route lock

Lock ACE, slot, scope and toolhead selection whenever:

- `tip_position_mm > 0`;
- calibration or verification is moving;
- verification is manually paused;
- verification is held at the toolhead or splitter;
- a cancelled or failed session leaves physical position uncertain.

While locked, pin the displayed route to the session route. Reject a different route even if it arrives through a direct API call or stale browser.

Unlock only when the route is known parked at ACE preload or has been explicitly cleared through a recovery/preparation operation.

### Shared-path rules

- Never feed a second spool into an occupied combiner, splitter or shared Bowden path.
- Do not use another ready slot as evidence that switching is safe.
- Do not clear route ownership merely because the toolhead sensor changed to clear.
- Do not allow selector changes during a checkpoint hold or pause.
- On disconnect, timeout, cancellation or protocol failure, preserve enough session identity to recover the same route.
- Prefer rejecting an uncertain move over guessing which spool is physically active.

### Preparation scope

Preparation should unload only the toolhead connected to the ACE route being calibrated. Unload-all is broader than necessary and can disturb unrelated paths.

When the toolhead sensor is occupied but `head_source` is unknown, preparation must submit the ACE and slot selected in the calibration UI. Firmware may accept that route only after validating topology, selected-slot gate state, current calibration occupancy, and conflicts with other loaded routes. The normal dashboard command historically tolerated a missing source by using the active ACE; calibration must never use that fallback. Once accepted, keep the recovered route in memory so all nested unload stages target the same spool and a failed or cancelled unload can be retried safely.

An automatic unload can prove that the selected toolhead sensor became clear. It cannot prove that loose filament was removed from a visible splitter/output path. Keep automatic sensor evidence separate from the user's clean-path acknowledgement.

## 9. Motion and state-machine design

### Measurement versus verification

Measurement and verification have different goals:

- Measurement uses controlled increments near an unknown endpoint. The current load procedure begins with 500 mm and then uses 100 mm increments while polling the sensor.
- Verification already knows the anchors. Each leg should be one bounded command rather than repeated 500/100 mm bursts.

The verification workflow is:

1. preload to toolhead in one continuous command;
2. stop at the toolhead sensor and wait for manual continuation;
3. toolhead to splitter in one continuous command;
4. stop at the splitter and wait for visual confirmation;
5. splitter to preload in one continuous command.

Manual pause/resume can divide a leg, but automatic chunking must not.

If the configured toolhead endpoint is reached while its sensor remains clear, enter the bounded `verify_toolhead_adjust` recovery hold rather than failing immediately. Accept only signed 1, 2, 5, or 10 mm jogs, cap cumulative correction at +/-50 mm, poll the toolhead sensor during movement, and transition to the normal toolhead hold as soon as it activates. Keep the signed correction diagnostic; never silently rewrite calibration values from a verification jog.

### Command acknowledgement is not motion completion

An ACE command callback confirms acceptance or reports an error. Completion is determined from ACE ready state plus elapsed-time safeguards. Cached ready state may still be visible immediately after sending a command, so do not mark a move complete too early.

The calibration timer currently waits until elapsed time reaches a substantial fraction of the expected move duration before accepting ready as completion, and applies an additional timeout margin.

### Sensor-driven stop

Poll the selected toolhead sensor throughout positive movement. Stop the ACE immediately on activation. Do not wait for the remainder of a long requested feed.

Record the estimated trigger coordinate separately from the calibrated target so timing and stop-latency differences remain diagnostic rather than silently altering configuration.

### Pause and resume

On pause:

1. estimate and commit the partial coordinate;
2. send stop directly;
3. remember the previous moving phase;
4. stop the timer;
5. resume by commanding only the remaining distance.

The displayed paused position is approximate because motion can continue briefly during deceleration.

### Cancellation must bypass blocked queues

A cancellation queued behind a blocking unload runs too late. Preparation unload therefore uses the direct Klipper webhook `multiace/calibration_unload_cancel`, exposed by the web backend as `/api/calibration/unload-cancel`.

Use this pattern for future long-running synchronous operations:

- register an out-of-band cancellation endpoint;
- set a cancellation flag;
- stop hardware immediately where supported;
- check the flag at safe points inside the operation;
- preserve an unknown/non-parked state if exact position cannot be proven.

Keep Stop and cancel available while moving, manually paused, and waiting at checkpoints.

### Session concurrency

- Increment the session ID for each new route session.
- Include it in every action request.
- Reject actions for a stale session rather than applying them to the new route.
- Treat page reload as a projection refresh, not as a reason to reset physical state.
- Never infer that an HTTP request finishing means the physical move finished; poll the authoritative state.

## 10. ACE protocol differences

### ACE Pro v1

- No equivalent per-slot `get_feed_info` telemetry is exposed by the current integration.
- Distance is a commanded-distance estimate.
- Repeatability runs are important.
- Empty state strings can differ from v2, notably `empty1`.

### ACE 2 Pro v2

- Provides per-slot feed information including steps, requested length and decoder value.
- Reports firmware motion states such as preloading.
- Supports continuous decoder sampling during movement.

Decoder guidance:

- Sample repeatedly during the move and use min/max span.
- Do not rely only on a before/after delta because the value may reset or hold between commands.
- Normalize the unsigned 64-bit value when required.
- Use span and outward/return symmetry as diagnostics for movement and slip.
- Do not claim that decoder count is an absolute filament-tip coordinate until its unit and firmware behavior are verified.
- Do not let decoder diagnostics alter the commanded coordinate model silently.

## 11. Effective configuration and persistence

Klipper lookup priority is:

```text
per-slot value under the selected ACE
then per-ACE value
then global value
```

Use the existing lookup methods in `ace.py` rather than duplicating precedence:

- `get_load_length`
- `get_retract_length`
- `get_swap_retract_length`

The frontend must calculate preview anchors with the same precedence. It must use measured session values only when the session route matches the currently selected route. Otherwise, showing old measured anchors beside a new slot creates the same class of UI/runtime disagreement as the Slot 4 verification bug.

Entire-ACE calibration writes shared ACE defaults. Single-slot calibration writes the path-specific override. Do not remove unrelated slot overrides when applying an entire-ACE value.

Config writes should use the existing conflict-aware read/merge/write path:

- retain a content hash to detect concurrent changes;
- create a backup;
- show the exact diff;
- do not save automatically;
- persist only after explicit review;
- show the required restart state.

## 12. UI rules learned from the feature

- Initialize selectors only after live state and topology are available.
- Distinguish a visually selected control from an acknowledged backend route.
- Disable controls from physical/session state, not only from a local HTTP busy flag.
- Pin selectors to the session while the route is locked.
- Show the tip marker in the existing ACE-splitter-toolhead graphic instead of duplicating position in an unrelated line.
- Use saved anchors immediately after restart when the route is known parked.
- State which sensor is clear; avoid broad claims such as all ACE-driven sensors are clear when only one selected sensor was evaluated.
- Use plain action language such as Use clear T4 as starting point instead of opaque state language such as Empty baseline explicitly confirmed.
- Separate automatic evidence from user confirmation.
- Show why an action is disabled, especially route locks and unavailable sensors.
- Keep expert corrections bounded and contextual rather than exposing permanent unrestricted motor controls.
- Never silently write a manual verification correction into configuration.

The graph, labels and controls should all derive from the same route/session object. If the graph says Slot 1, the next command must contain Slot 1.

## 13. Deployment and runtime version lessons

The feature spans two independently loaded runtimes:

- frontend/backend files served by the standalone multiACE web service;
- `ace.py` loaded into the running Klipper process.

Updating web files does not replace the Python class already loaded in Klipper. A web restart can therefore show a new button that calls an old G-code implementation.

After changing `ace.py`, restart Klipper or reboot the printer. For changes involving boot scripts, USB/serial discovery or the full installed package, use a full printer reboot as directed by the installer.

Useful deployment checks:

- inspect the JavaScript served by the printer for a newly added symbol;
- inspect `/api/state` for newly added status fields;
- invoke no motion until the web and Klipper versions agree;
- hard-refresh the browser if the served bundle is correct but the page is stale;
- remember that `/etc/init.d/S98multiace-web` owns the standalone web service;
- do not assume reinstall succeeded merely because the page loads.

Convert recognizable old-runtime errors into actionable restart guidance, but do not hide the underlying error in logs.

## 14. Diagnostics and observability

Log route identity at the start of every physical operation:

```text
operation, session ID, ACE, slot, head, phase, requested distance
```

For movement, also retain:

```text
start coordinate, direction, speed, start time, move ID,
sensor transitions, ACE response errors, completion reason,
decoder min/max/span when available
```

The calibration implementation uses a move sequence ID and session ID so delayed callbacks cannot mutate a newer move.

When reproducing a route bug, capture:

1. `/api/state` before the click;
2. selected values rendered by the page;
3. request payload sent to `/api/calibration/action`;
4. Klipper calibration status immediately after start;
5. the ACE and slot index in the hardware request;
6. sensor state and final session state.

The most useful invariant to log is:

```text
displayed route == submitted route == session route == hardware request route
```

## 15. Common failed assumptions

| Failed assumption | Correct rule |
|---|---|
| One connected ACE means T1 | Resolve the mapped ACE-driven head from topology |
| First ready spool is always intended | Use it only as an untouched default; explicit selection wins |
| A ready spool occupies the Bowden path | Ready normally means parked at the ACE preload position |
| Clear toolhead sensor means clear route | The tip may be elsewhere in the Bowden/splitter path |
| `filament_detected` is the toolhead endpoint | Use `filament_at_extruder` for the selected head |
| Unknown sensor means clear | Unknown must block sensor-dependent automation |
| The selector controls the active backend slot | Submit and verify the full route on every start |
| Verify again can reuse any completed session | Reuse only an exact route match |
| UI disabling prevents collisions | Enforce the same invariant in Klipper |
| A queued cancel stops a blocking unload | Use an out-of-band stop/cancel path |
| Decoder count is absolute distance | Treat it as movement/slip telemetry until proven otherwise |
| Verification should use measurement bursts | Known verification legs can be one bounded command each |
| The splitter can be skipped | It is a user-confirmed physical checkpoint |
| Reaching a checkpoint should auto-continue | Wait for explicit intervention at toolhead and splitter |
| Web reinstall means Python is live | Restart Klipper or reboot after `ace.py` changes |
| Unload all is safest preparation | Clear only the selected route and avoid disturbing unrelated paths |

## 16. Regression scenarios for future features

Every filament-motion feature should test at least these cases:

### Route selection

- One connected ACE mapped to T4: initial route shows T4, not T1.
- Slots 1 and 4 both ready: selecting Slot 1 moves only Slot 1.
- Complete a Slot 4 operation, select Slot 1, then repeat: the new session and hardware request use Slot 1.
- Switch scope between entire ACE and one slot while parked: effective values follow the selected scope.

### Collision prevention

- Pause with the tip between ACE and splitter: every route selector remains locked.
- Hold at the toolhead: another slot cannot be selected through UI or direct API.
- Hold at the splitter: another slot cannot be selected through UI or direct API.
- Cancel mid-route: route identity remains available for recovery and no different spool can start.
- Return to preload: selectors unlock only after position zero is authoritative.

### Sensors and unknown state

- Toolhead sensor false: start is allowed only if all other guards pass.
- Toolhead sensor true: start from preload is rejected.
- Toolhead sensor unavailable: start is rejected as unknown, not treated as clear.
- Sensor trips during a long outbound command: ACE receives stop immediately.
- Sensor remains triggered after full return: verification fails rather than reporting parked.

### Motion control

- Each verification leg produces one full remaining-distance request.
- Manual pause stops the current leg and resume sends only the remaining distance.
- Toolhead and splitter holds require explicit continuation.
- Stop and cancel is available while moving, paused and held.
- ACE disconnect, rejected command and timeout all end in a recoverable failed/cancelled state.

### Protocol and deployment

- Run the same workflow on v1 with estimated labels and on v2 with decoder diagnostics.
- Reinstall web files without restarting Klipper: the UI reports an actionable runtime mismatch rather than proceeding unsafely.
- Reload the page during movement or a checkpoint hold: the UI reconstructs and locks to the live session.
- Browser cache contains the old bundle: version/symbol checks distinguish cache from old installation.

## 17. Template for a new filament-motion feature

Before implementation, write down:

1. Route identity: ACE, slot, head and topology constraints.
2. Physical origin: how parked is proven.
3. Endpoints: which are sensor-driven, distance-driven or visually confirmed.
4. Collision domain: which other routes share hardware with this route.
5. State machine: every moving, held, paused, complete, failed and cancelled state.
6. Guards: printer, swap, connection, gate, sensor, mode and route-lock checks.
7. Motion bounds: maximum distance, speed and timeout.
8. Stop behavior: direct hardware stop and out-of-band cancellation if needed.
9. Telemetry: authoritative signals versus diagnostics.
10. UI projection: how selectors, graph and actions derive from session state.
11. Persistence: effective precedence, review, backup and restart behavior.
12. Recovery: what the user does after interruption or unknown position.
13. Deployment: which processes must restart.
14. Regression tests: especially two ready spools and stale-session reuse.

Do not start motor-control implementation until the physical origin, collision domain and recovery path are explicit.

## 18. Known limitations and follow-up work

- Verification fine-position controls are implemented in firmware, API, and the production UI; their stop latency and +/-50 mm recovery bound still require validation on each supported ACE generation.
- The real unit/scaling and cross-firmware stability of the ACE 2 Pro decoder should be measured before using it for anything beyond diagnostics.
- Stop latency and physical coast introduce uncertainty into elapsed-time position estimates.
- Backlash, splitter friction, material stiffness and spool drag can make outward and return paths asymmetric.
- Automatic-preload repeatability should be measured per ACE generation, slot and material.
- Recovery after power loss cannot assume the last persisted coordinate is still physically correct; require a clean-route procedure.

These limitations should appear as explicit confidence or recovery states, not be hidden behind a precise-looking number.
