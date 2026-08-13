# Archon Quest Auto-Bot — Natlan → Snezhnaya (Linux / GE-Proton)

## Context

The goal is a bot that plays Genshin Impact's Archon Quest storyline unattended, from
Chapter V (Natlan) through Chapter VII (Snezhnaya), on a Linux host running the game
under GE-Proton. Archon Quests are long (each act runs 1–3 hours of walking, dialogue,
cutscenes, and scripted combat) and highly linear, which makes them a good automation
target: the route is fixed, the objectives are ordered, and failure modes are mostly
"got stuck on terrain" rather than "made a wrong strategic choice".

Scope is three chapters, ~13 acts:

| Chapter | Name | Acts | Versions |
|---|---|---|---|
| V | Natlan — *Incandescent Ode of Resurrection* | Acts I–V + Interlude *All Fires Fuel the Flame* | 5.0–5.3 |
| VI | Nod-Krai / Sumeru — *Song of the Welkin Moon* | Acts I–VII | 6.0–6.3 |
| VII | Snezhnaya — *Everwinter Without Mercy* | Act I (more to come) | 7.0 |

Chapter VII shipped **2026-08-12**, one day before this plan was written. There is no
community route data for it, and the wiki act list for Chapter VI Acts II–III and the
VI/VII boundary is inconsistent across sources. So the bot cannot be built from scraped
guides or from per-act macro recordings — new acts would always outrun the data.

The resolution is that **Genshin already ships the route data**. The quest system
continuously publishes where to go: the tracked-objective marker on the map, the compass
arrow on the minimap, the distance readout on the marker, and the objective text in the
tracker. A bot that reads the game's own guidance and steers to minimize
distance-to-objective needs no recorded path and generalizes to acts that do not exist
yet. Recorded routes survive only as an *optional cache* for the handful of places where
reactive steering is slow (vertical interiors, mazes). The act list is likewise
enumerated from the in-game Quest menu rather than hardcoded.

The user chose **assisted autonomy with human handoff**: the bot drives traversal,
dialogue, and known combat, and hands control back on anything it doesn't recognize.

### Detectability: what "undetectable like GE-Proton" can and cannot mean

GE-Proton is safe in practice because there is nothing to detect. The client binary is
unmodified, its memory is untouched, its traffic is normal. The only signal is an
*absent* kernel driver, and HoYoverse has chosen for years not to enforce on it.

This bot can match that on the **client side** exactly, and the design below commits to it:

| Surface | GE-Proton | This bot | Enforced by |
|---|---|---|---|
| Client binary modified | no | no | never ship a patcher |
| Code injected / DLL loaded | no | no | no injection path exists |
| Memory written | no | **no** | build-time grep test fails on write syscalls |
| Hooks / detours | no | no | reads are `process_vm_readv` only |
| Network traffic altered | no | no | bot never touches the socket |
| Input origin | kernel input stack | kernel input stack (`uinput`) | virtual device is indistinguishable from a USB keyboard |

What GE-Proton does *not* produce, and this bot unavoidably does, is **server-side
behavioral telemetry**: movement traces, input inter-arrival distributions, session
length, action cadence, objective-completion timing. HoYoverse's realistic detection
path for a well-built bot is statistical analysis of that stream, not client
introspection. That risk can be *shaped* — made to sit inside the human distribution —
but it cannot be driven to zero, and no amount of client-side cleanliness removes it.

Concretely, the residual risk is roughly: far lower than any Windows cheat tool or
memory-writing bot, meaningfully higher than plain GE-Proton, and dominated by how
disciplined the humanization and session budgeting are. Hence: **the account running
this is expendable and is never the user's main.** The safety manager (Layer 6) is
load-bearing, not decoration.

---

## Architecture

Python 3.12 (CV + OCR ecosystem, fast iteration), numpy for the hot pixel paths.
Everything runs on the **Linux host**, outside the Proton prefix. The game process is
never modified.

```
archonbot/
  platform/        # capture, input, process attach — the only OS-specific code
    capture.py     #   X11 XShm / Wayland PipeWire portal → numpy BGR frames
    input.py       #   /dev/uinput virtual kbd+mouse
    attach.py      #   locate GenshinImpact.exe under Proton, process_vm_readv reads
  perception/
    ocr.py         #   PaddleOCR PP-OCRv5 (CPU or ROCm/CUDA), cropped to UI regions
    templates.py   #   cv2.matchTemplate for static UI (dialogue arrow, F-prompt, ...)
    detect.py      #   YOLOv8n for NPC markers, quest waypoints, chests, enemies
    screenstate.py #   classifier: OPEN_WORLD | DIALOGUE | CUTSCENE | MAP | COMBAT | LOADING | ...
  world/
    state.py       #   fused world model (position, HP, party, quest stage, screen state)
    memory.py      #   optional read-only offsets; degrades to vision on signature miss
  nav/
    objective.py   #   read quest marker + distance; map pixel ↔ world transform
    teleport.py    #   pick nearest waypoint to objective, teleport
    steer.py       #   reactive heading loop + recovery ladder
    hints.py       #   self-written per-location fixes (learned, not recorded)
  combat/
    engine.py      #   rotation executor, dodge-on-telegraph, revive/retry
  quest/
    runtime.py     #   generic objective-following loop — the core
    verbs.py       #   objective text → TALK/GOTO/FIGHT/INTERACT/FOLLOW/WAIT
    dialogue.py    #   generic advance + choice selection
    acts/          #   OPTIONAL per-act overrides; absence is fine
  supervisor/
    handoff.py     #   pause, screenshot, notify human, wait for resume
    safety.py      #   session budgets, humanization, kill switch
  record.py        #   frame harvesting for model training (not route recording)
  main.py
```

### Layer 1 — Platform (Linux-specific, the part no existing tool gives us)

**Capture.** Detect the session type at startup. On X11, `XShmGetImage` on the game
window (~1–2 ms/frame). On Wayland, open one long-lived `xdg-desktop-portal` ScreenCast
session over PipeWire and pull frames from the stream — do *not* re-prompt per frame.
Target 10 fps steady, burst to 30 during combat. Normalize everything to a canonical
1920×1080 coordinate space so all templates and UI regions are resolution-independent.

**Input.** Create a virtual keyboard + mouse via `/dev/uinput` (python-evdev). This is
the right choice on Linux specifically: uinput events enter the kernel input stack, so
Wine/X11/Wayland receive them identically to a physical device — no XTEST quirks, and
it works on Wayland where XTEST doesn't. Requires the user to be in the `input` group
or a udev rule granting access to `/dev/uinput`.

All input goes through a humanizer:
- key hold durations sampled from a log-normal fit of recorded human play, not `sleep(0.1)`
- mouse camera turns follow a Bezier curve with an overshoot-and-correct tail
- 1–3 frame jitter on every action boundary
- deliberate small idle gaps between quest steps

**Process attach.** Find the `GenshinImpact.exe` PID inside the Proton prefix, read
`/proc/<pid>/maps` to locate `UserAssembly.dll` / `GameAssembly.dll`, then use
`process_vm_readv(2)` for reads. This is the hybrid path the user asked about, and the
reason it's acceptable here rather than on Windows:

- `mhyprot` is a Windows kernel driver; it does not load under Proton, so there is no
  handle-stripping / callback-based blocking of foreign readers.
- `process_vm_readv` requires no ptrace attach, no injection, no DLL, no hooking. The
  game's address space is not written to and its code is not modified. From the game
  process's perspective nothing happened.
- What HoYoverse *can* see is server-side: impossible movement, teleport writes,
  superhuman input timing, 24/7 sessions. Reads cause none of that. **Writes are
  categorically forbidden in this codebase** — no speed hacks, no coordinate writes, no
  packet touching. That is the line that separates "reads a value" from "trips
  behavioral detection".

**Offsets are versioned and disposable.** `world/memory.py` resolves each field by
byte-pattern signature scan at startup, not by hardcoded offset. Signatures live in
`config/offsets/<game_version>.toml`. If a scan fails — which it will on every patch —
the bot logs it and **silently degrades that field to the vision pipeline**. Memory is
an accelerator, never a dependency. A `--vision-only` flag must produce a fully working
bot; CI runs the test suite in that mode.

Fields worth reading (all read-only): player world position + rotation, party HP,
current quest ID/stage, loading-screen flag. Nothing else.

### Layer 2 — Perception

Four cooperating detectors, cheapest-first:

1. **Screen-state classifier** runs every frame. Small CNN (or, to start, a set of
   template/histogram checks) over downscaled frames → one of `OPEN_WORLD`, `DIALOGUE`,
   `DIALOGUE_CHOICE`, `CUTSCENE`, `MAP`, `COMBAT`, `LOADING`, `DOMAIN_ENTRY`,
   `PUZZLE_UI`, `UNKNOWN`. This gates everything else — no OCR runs during a cutscene.
2. **Template matching** for static UI: the interact `F` prompt, dialogue continue
   arrow, "Auto" toggle, teleport-confirm button, revive prompt. Cheap, exact, stable.
3. **OCR** (PaddleOCR PP-OCRv5) on cropped regions only: quest objective text
   (top-left tracker), dialogue choice lines, waypoint names on the map, act titles in
   the Quest menu. Never full-frame.
4. **YOLOv8n** for the things templates can't do: quest waypoint markers, NPC nameplate
   markers, enemy positions. Trained on frames harvested by `record.py` — note this is
   frame *labeling* data for the models, not route recording; it is collected once and
   reused across all regions.

`UNKNOWN` for more than N consecutive frames is a handoff trigger, not an error.

### Layer 3 — Quest runtime: one generic loop, not a script per act

The runtime is **not** a per-act script interpreter. It is a single objective-following
loop that works on any quest, because every Archon Quest objective reduces to one of a
small set of verbs, and the game tells us which:

```
until act complete:
    obj = read_objective()          # OCR tracker text + marker + distance
    verb = classify(obj.text)       # see table below
    dispatch(verb, obj)             # navigate / talk / fight / interact / wait
    if screen_state in (DIALOGUE, CUTSCENE):  advance()      # always safe, always first
    if no progress for T:  escalate → handoff
```

Objective text is classified into a verb by keyword + a small text classifier, trained
on the objective strings the bot itself harvests (they are extremely formulaic across
all seven regions):

| Objective text pattern | Verb | Action |
|---|---|---|
| "Talk to …", "Speak with …" | `TALK` | navigate to marker, `F`, advance dialogue, auto-pick choices |
| "Go to …", "Head to …", "Return to …" | `GOTO` | navigate to marker, arrive |
| "Defeat …", "Clear out …" | `FIGHT` | navigate, engage with combat profile |
| "Investigate …", "Search …", "Pick up …" | `INTERACT` | navigate, `F` on nearest prompt |
| "Follow …", "Escort …" | `FOLLOW` | maintain 5–10 m from moving marker |
| "Wait for …", "Watch …" | `WAIT` | idle until objective changes |
| unmatched | `UNKNOWN` | handoff |

Dialogue handling is fully generic and needs no per-act data: advance on any dialogue
screen; on a choice, prefer the option the objective text mentions, else the option that
is not "(Ask about …)" side-branch phrasing, else the last one — Genshin's plot-advancing
choice is overwhelmingly the terminal one. This alone covers the majority of Archon
Quest runtime.

**Act files are optional overrides, not requirements.** `quest/acts/*.json` exists only
to annotate the known-hard bits — a boss profile to use, an arrival radius to widen, a
step to hand off on unconditionally:

```jsonc
{
  "id": "ch07_act01", "chapter": 7, "act": 1,
  "title": "Everwinter Without Mercy",
  "requires": { "ar": 40, "completed": ["ch06_act07"] },
  "overrides": [
    { "when_objective_matches": "Defeat the .*",
      "combat_profile": "boss_unknown", "action": "handoff" },
    { "when_objective_matches": "Cross the frozen .*", "arrive_radius": 12.0 }
  ]
}
```

An act with **no file at all still runs** on the generic loop. That is the property that
makes this survive Snezhnaya Acts II+ in 7.1, 7.2, … — new content needs zero authoring,
and overrides are written only in response to an actual handoff.

**Act ordering is discovered, not hardcoded.** On startup the bot opens the Quest menu,
OCRs the Archon Quest chapter/act list, and picks the first incomplete act. This
sidesteps the wiki inconsistencies around Ch. VI Acts II–III and gives a correct resume
point after any break.

### Layer 4 — Navigation: follow the game's own guidance

No recorded macros. The navigator is a closed loop around the objective marker the game
already draws. This is the piece that makes the bot generalize to unreleased content.

**Step 1 — read the objective.** Open the map (`M`). The tracked Archon Quest draws a
marker; OCR the objective text and the marker's distance label, and take the marker's
pixel position. The map's pixel↔world transform is a known affine per region (scale +
offset, calibrated once against three known teleport waypoints), so marker pixels
convert to approximate world coordinates. When the memory path is live, read the
objective's world position directly and skip the projection.

**Step 2 — teleport to the best entry point.** Enumerate unlocked waypoints and Statues
visible on the map, pick the one minimizing straight-line distance to the objective,
click, confirm. Long-distance travel is therefore always teleport, never walking —
which sidesteps open-world 3D pathing almost entirely and, usefully, is also what a
human does.

**Step 3 — reactive steering to the marker.** A control loop, not a path replay:

```
loop at 10 Hz:
    bearing = angle to objective          # from compass arrow, or memory position
    yaw_error = bearing - camera_yaw
    mouse_dx = clamp(Kp * yaw_error)      # humanized Bezier, not a jump
    hold W;  sprint if stamina > 40 and distance > 30m
    if distance not decreasing for 3s:  escalate(recovery)
    if distance < arrive_radius:        done
```

The **distance readout is the error signal** — it is a scalar the game hands us for
free, and hill-climbing on it needs no map of the terrain.

**Step 4 — recovery ladder** when distance stops falling (the only hard part, since
Teyvat has cliffs):
1. jump (small ledges)
2. strafe ±45° for 2 s and re-aim (thin obstacles, tree trunks)
3. look up — if the objective is *above*, climb; if below, glide (`X` from height)
4. detour sampling: pick the best of 8 candidate headings by 3 s trial, keep the one
   that reduces distance
5. re-teleport to the next-nearest waypoint and re-approach from a different side
6. handoff

Steps 1–5 resolve the overwhelming majority of Genshin terrain because the game is
built to be traversable from most directions.

**Optional acceleration, not a dependency.** Two caches may be layered on later:
- `nav/hints.json` — learned per-location fixes ("at this objective, glide from the
  north cliff"), written automatically whenever recovery step 4 or 5 succeeds. The bot
  gets faster at content it has seen without anyone recording anything.
- an offline navmesh built from extracted terrain heightmaps, enabling true A*. Only
  worth it if reactive steering measures below ~85 % first-try arrival.

`record.py` still exists, but demoted: its job is harvesting labeled frames for training
the perception models (M1) and optionally capturing a hint for a location the recovery
ladder keeps failing on. It is **not** on the critical path for adding a new act.

### Layer 5 — Combat

Archon Quest combat is mostly trivial trash plus a few scripted setpieces. Ship a
**profile** system rather than a general combat AI:
- `trash_wave`: lock-on, rotate a fixed skill order (E → E → swap → E → NA chain),
  dodge on red-telegraph detection, disengage when no enemies detected for 4 s.
- `boss_<name>`: authored per fight, or `handoff` if not authored.
- Universal: HP below 40 % → retreat and heal; party wipe → accept revive, retry once,
  then hand off.

Any unrecognized health-bar-with-name (i.e. a named boss with no profile) is a handoff.
This is the assisted-autonomy contract: the bot does not gamble on setpieces.

### Layer 6 — Supervisor: handoff and safety

**Handoff** pauses input, captures a screenshot + last 30 s of frames as a clip, and
notifies the user (desktop notification via `notify-send`, plus optional ntfy/Telegram
push). The bot then waits. The user plays the hard bit and presses a resume hotkey; the
bot re-syncs by OCRing the objective tracker and jumping to the matching step. Every
handoff is logged with its cause — the handoff log is the backlog of what to script next.

**Safety manager**, enforced in code and non-optional:
- Global kill switch on a hotkey and on mouse-jiggle-by-human detection.
- Session budget: default max 2.5 h continuous, then a mandatory 30–90 min idle break.
  Randomized daily start time. No 24/7 operation.
- Refuses to run if co-op mode is detected — never automate anything that touches other
  players.
- Hard assertion that no write path exists: a unit test greps the codebase for
  `process_vm_writev`, `ptrace(PTRACE_POKE`, and fails the build on a hit.
- Startup banner requiring explicit acknowledgement that this is a burner account.

---

## Milestones

| # | Deliverable | Done when |
|---|---|---|
| M0 | Platform layer | Capture 10 fps from the Proton window on both X11 and Wayland; uinput moves the character in-game |
| M1 | Perception baseline | Screen-state classifier ≥98 % on a 2k-frame hand-labeled set; OCR reads the objective tracker reliably |
| M2 | Dialogue autopilot | Bot clears an entire dialogue-heavy quest segment unattended, including choices, with no per-quest data |
| M3 | Objective navigation | Teleport-to-best-waypoint + reactive steering reaches an arbitrary quest marker; ≥85 % first-try arrival across 40 sampled objectives, recovery ladder handles the rest |
| M4 | Generic runtime | Verb classifier + dispatch completes Ch. V Act I end-to-end with **no act file present**, ≤3 handoffs |
| M5 | Chapter V complete | All 5 acts + interlude; overrides written only where handoffs actually occurred; Natlan trash combat profiles |
| M6 | Chapter VI | 7 acts incl. Sumeru segments; `nav/hints.json` self-learning demonstrably reducing handoffs run-over-run |
| M7 | Chapter VII | Act I completes with zero authored content, proving the generalization claim on material released after the bot was built |

Build-order note: there is deliberately **no "record the routes" milestone**. M3 is the
load-bearing one — if reactive steering hits its arrival rate, every later milestone is
mostly combat profiles and patience. If it lands below ~70 %, revisit the offline
navmesh option before continuing to M4.

---

## Verification

**Per-layer, offline:**
- Perception: a fixture corpus of labeled frames (`tests/frames/`) checked into git-lfs;
  `pytest` asserts classifier + OCR + template hit rates. Runs in CI with no game.
- Runtime: the step interpreter is tested against a **mock perception source** that
  replays a recorded frame sequence, so act scripts can be regression-tested without
  launching Genshin.
- Safety: static test asserting no memory-write symbols; unit tests on session-budget
  and kill-switch logic.

**In-game, per act:**
1. Run with `--dry-run`, which executes perception and decision-making but routes input
   to a logger instead of uinput. Confirm the decision trace matches what a human would do.
2. Run live on the burner account with the human present, timing it and counting
   handoffs. Acceptance: act completes, handoffs ≤3, no deaths that aren't recovered.
3. Re-run the same act in `--vision-only` to prove the memory path is a pure accelerator.
4. After a game patch, run the offset signature scan; confirm failures degrade to vision
   rather than crashing or, worse, reading garbage coordinates.

**The generalization test (the one that matters):** point the bot at an act it has never
seen and for which **no act file, no route, and no hint exists** — ideally a World Quest
in a region already covered, since those are plentiful and structurally identical. If it
completes on the generic loop, the design claim holds and Snezhnaya Acts II+ will work on
release day. If it only works on acts that were tuned by hand, the design has quietly
regressed into macro replay and needs revisiting.

**Behavioral-plausibility check:** log input inter-arrival times and movement traces for
a bot session and a human session of the same act, and compare the distributions. If the
bot is distinguishable by eye on a histogram, it is distinguishable by a query. Tune the
humanizer until it is not — this is the only lever that exists against server-side
detection.

**End-to-end:** a fresh burner account at the required AR runs Chapter V Act I through
Chapter VII Act I in sequence across scheduled sessions, resuming correctly after each
break via act-list discovery.

---

## Explicitly out of scope: 100 % region exploration

Worth recording why, since it looks like a natural extension and is not one.

- **The technique inverts.** This plan needs no recorded routes because the game
  publishes an objective marker. Undiscovered chests, oculi, and puzzle rewards have no
  marker by design, so an exploration bot must ingest external coordinate databases and
  walk authored routes — reintroducing exactly the dependency Layer 4 removes.
- **Puzzles dominate.** Archon Quests are dialogue, walking, and scripted fights.
  Exploration is gated behind hundreds of distinct per-region mechanics, so under the
  assisted-autonomy contract the handoff rate goes from a few per act to near-constant.
- **The risk profile is worse.** Natlan→Snezhnaya Archon content is ~30 h and resembles
  a player binging story. 100 % across three regions is several hundred hours of
  repetitive collection with efficient pathing and no story engagement — the exact
  behavioral signature server-side detection exists to find. Client-side cleanliness is
  unchanged; the telemetry side is not, and humanization cannot disguise volume.
- **Reuse if ever revisited:** platform, perception, combat, steering, and supervisor
  layers carry over (~70 %). The objective-verb loop — the actual intelligence here —
  does not.

## Open items to settle during M0

- Confirm the exact act list and titles for Ch. VI Acts II–III and the Ch. VI→VII
  boundary by OCRing the in-game Quest menu (wiki sources disagree; the game is truth).
- Decide GPU vs CPU for PaddleOCR/YOLO based on the host — a dGPU shared with the game
  under Proton may cost more fps than it saves latency.
- Verify `/dev/uinput` permissions and whether the user's compositor allows a persistent
  PipeWire ScreenCast session without re-prompting.
