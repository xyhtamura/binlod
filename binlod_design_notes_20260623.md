# Binlod & Tabota — Generator / Granular Design Notes

*Forward-looking design note, not a handover. Captures decisions from the
June 2026 design conversation on **Binlod** (the granular generator) and the
**Tabota** features it forces. Read alongside `tabota_handover_20260627.md` and
`tabota_chart_model.md`. The MVP and an editing shell are now built; this remains
the **contract**, with build deltas flagged inline and dated (see §5a, §11a).*

*Lineage note: extends the handover's §8 deferred list (granular synth,
B-layer). The granular synth referenced there is now named **Binlod**.*

---

## 0. Names

- **Binlod** — Hiligaynon for *seed / grain*. The granular generator
  application. Keeps the agrarian register of Tabota (田, rice paddies) and the
  rice-paddy → grains-of-rice pun that is load-bearing, not decorative.
- Capability matrix (doubles as fidelity ladder):

  | | output: MIDI | output: audio |
  |---|---|---|
  | **input: MIDI** | lossy corner (MVP) | — |
  | **input: Tabota** | high-fidelity | high-fidelity master |

  Tabota-in is the authored master; MIDI-in borrows note-ons to skip authoring.

---

## 1. The core realization: Binlod is a *peer view*, not a downstream consumer

Binlod is **a different way of playing back, rendering, and interpreting what
you can already draw on the Roll** — not a tool that sits downstream of Tabota.
This collapses "author-in-Tabota-then-feed" and "author-in-Binlod" into one
design: they are read-only vs read-write over **one shared document spine**.

**Propagation does not come from sharing UI components.** It comes from a shared
*core*. If both shells import the same model + resolver, a B-layer change lands
once and both views speak the new grammar for free. Three tiers:

- **model** — document + mutation API + change events. Shared. B-layer here → free to both.
- **resolve** — `tabota-resolve.js`, already pure and out. Shared. B-layer here → free to both.
- **primitives** — draw-shaped-curve-between-pins, hit-test, drag-to-seconds,
  per-span grid, affect logic. View-agnostic. If B-layer is *expressed in terms
  of these*, free to both.

Only genuinely view-specific rendering stays per-shell (Roll draws notes; Binlod
draws grain clouds) — and that part *should not* propagate, because it is where
the two tools differ on purpose. **The art is maximizing tier 3.**

**Framework asymmetry is fine.** Extract the core as vanilla ES modules. The Roll
keeps its working chrome; Binlod (greenfield) may use React for *its* chrome
while importing the same vanilla core. Symmetry of core, not of framework. This
de-risks the move — no rewrite of a working app to enable sharing.

**Externalizing `script.js` ≠ state sharing.** A file split buys code-sharing;
"multiple things update when one does" is a *state* problem. Separate pages each
instantiate their own document; syncing them needs BroadcastChannel /
SharedWorker / persisted-doc-plus-change-feed. This is the expensive 80% and is
**only required if Binlod must write in v1.** A read-only Binlod needs only the
shared resolver + a load path, and still inherits B-layer for free.

**Litmus test:** this second view is the test of whether the primitives are
actually extracted. If building Binlod forces copy-paste from the Roll, tier 3
doesn't exist yet. It is also the best forcing function for the **conformance
fixtures** — the first time a *different* interface reads the same resolved
model, "does my spec port" stops being theoretical.

---

## 2. The generator abstraction: `generator → events → sink`

The "family of sequencers" is not a family of apps. They are **generators**:
functions `(points, shapes, params, seed) → events`. Two orthogonal axes:

- **generator** — granular density, arp/ostinato, etc.
- **sink** — audio grains | MIDI port | document writer

Binlod's granular generator + audio sink = granular synth.
Same generator + MIDI sink = the granular MIDI sequencer.
Different generator (arp) + same core = ostinato tool.

Factor these two axes and the "family" is a small grid, not N apps. The real
artifact is a **generator interface** — one plugin point in the shared model.

---

## 3. The load-bearing distinction: rate curves vs value curves

Both are the *same object* (shaped curve, `shapeK` vocabulary, Simpson
integration for nonlinear shapes) in **opposite roles**:

- **rate curve** — *generates instants*. λ(t) integrated (cumulative) → events
  placed at equal increments of running integral. This is **literally the tempo
  engine**: `bpm integrated → beats`; `density integrated → grains`. Same math.
- **value curve** — *sampled at* instants something else produced. velocity(t),
  cc(t), pitch(t), read at each grain time; never generates one. Decorates.

The whole granular generator in one line: **one rate curve emits the instants;
N value curves are read there, each under its own lens.**

Jittered velocity is therefore *not* per-grain authoring — it is a value curve
sampled at grain instants, plus jitter. Density, velocity, pitch = several
simultaneous charts over one clock, sampled where the rate chart fires. This is
the existing parallel-charts-over-the-numéraire architecture, unchanged.

---

## 4. f(x) on the y-axis = promoting the curve from decoration to content

"Write f(x) on the y axis" is **not a new primitive.** The tempo curve and the
pitch slide are already f(x) over the clock. The move is to let a lane's
*content* be a curve, not just point-notes — un-gating an engine that exists.

Precise model correction (keep this sharp):

- A **lens is a chart**: it says what a y-value *means* (pitch-Hz, velocity
  0–127, CC-74, density λ).
- A **curve is content** drawn in that space.
- **lens + curve = an interpretable lane.** You do not have "f(x) as a lens";
  you have "a curve read under a lens." The different y-axis lenses are
  velocity, CC, density, pitch — each a chart; the f(x) is content under one.

**Rosetta, pointed downstream:** f(x) is the master; MIDI gets the sampled
render. Export is a sampling that **does not invert** — round-trip MIDI→Tabota
recovers events, never the function that birthed them. The curve is the score;
the notes are its shadow on the MIDI wall. Fully compatible with **baked**:
baking commits the curve's output to concrete events while the curve stays in
the doc as editable master. Baked freezes the shadow, doesn't discard the light.
Conservation holds — f(x) is conserved; baking is a projection.

---

## 5. The point-duration note = existing note, third DOF (anchor) unlocked

Edge-only notes pin onset + end; meaning lives at the onset or smears across the
span. **Rise-and-hit has its salient instant in the interior** — the hit is the
climax, approach before, decay after, footprint past it on both sides. Edge-only
notes structurally cannot say "the important instant is inside me."

Resolution: a third pin — **start, anchor, end** — a 3-DOF element the **DOF law
already governs**. Three pins → renders; leave the anchor free → it floats to
center / integral-balance; the sides hang off it.

**Decision:** this is the **existing note with its third DOF unlocked**, *not* a
new element type. Anchor-pin optional. ("The common must not be a prison.") Most
users will adjust the length of the before/after (head/tail), so head/tail are
the handles people actually grab.

**Unification with affect modes:** the anchor's two sides *are* the affect-mode
deposit. `before` = the [start, anchor] shape; `after` = the [anchor, end]
shape; `neither`/`both` fall straight out. The affect-mode system was already a
*latent* point-duration note — a deposit at an *implicit* point with two sides.
The point-duration note exposes that point: pinnable, given extent. Not a new
mechanism — the anchor the affect system was already depositing at.

**Lens-relative meaning (the most Tabota thing here):**
- under the **density lens** → one mound-cluster: extent = cloud footprint,
  anchor = attractor, two shapes = crescendo-in / decrescendo-out, optional
  anchor grain = the hit (fired by default; a per-heap toggle — see §5a).
- under a **pitch lens** → an accent, or a slide's arrival instant.
Same structural feature; meaning is lens-relative.

---

### 5a. Anchor grain — default, not invariant (revised 2026-06-28)

The onset grain was first specified as **guaranteed** (P(onset)=1 — the pin always
strikes). In build it became a **per-heap toggle**, `fire onset grain`, defaulting
ON. The pin still *locates* the onset visually — a hollow ring marks it when the
grain is suppressed — but whether a grain *sounds* there is now a DOF the author
owns per heap.

Reason: a silent onset is musically real — a cloud that swells toward an instant
it never articulates. The guarantee was a prison; the default keeps the common
case free without foreclosing the other. Restatement of the conservation reading:
the anchor **coordinate** is conserved, not a grain at it. ("The common must be
easy, but must not be a prison.")

## 6. Open composition rule (decide before writing the generator)

Curves now live at **two scales**:

- **lane/voice curves** — global to a voice, stacked over all its notes
  (the per-voice velo/CC curves).
- **per-element curves** — the point-duration note's own internal before/after
  shape.

**When a note with an internal shape sits in a voice that also has a value
lane, which wins — multiply, override-on-span, or add?** This composition rule
is the real decision. It is the *same* question as global tempo vs local
note-glide, so the **region-vs-note resolution from the tempo engine may port
directly.** Resolve before writing a line of generator.

---

## 7. MIDI reality (constraints to encode eventually)

- **Velocity is static, per-note, pinned at note-on.** There is no continuous
  velocity and **no global voice velocity**. This is the protocol, not a choice.
- **Channel CC** (mod wheel, etc.) is per-voice = per-channel — global to the voice.
- **Per-note continuous expression** requires **poly aftertouch** (per-note
  pressure) or, more completely, **MPE** (each note on its own channel carrying
  its own bend / CC74 / pressure).
- **Therefore: per-voice/per-note *value curves* cannot ride plain MIDI.** They
  force the **MPE rung** of the fidelity ladder. Plain MIDI = static velocity +
  channel CC + poly-AT; per-note value curves = MPE. The value-curve work must
  know its sink.
- **Each instantiated voice has its own global track, all drawable on.** Tabota
  must eventually model per-voice tracks as first-class drawable surfaces.

---

## 8. Baked vs generative (decided: baked, for now)

- **Baked** — generator runs once, emits concrete events; those are the score.
  Deterministic, conserves seconds, fits everything that exists. The MVP and v1.
- **Generative** (deferred) — document stores the *generator spec + seed
  policy*; events realized at playback with fresh entropy. The score becomes an
  **instruction**, not an event list. This is the achronous / instruction-based
  (Cage / Happenings) reach arriving as a natural consequence of generators, not
  a bolted-on feature. Resolves cleanly via DOF law: generator **pins its
  boundaries** (extent fixed, seconds conserved at edges); indeterminacy lives
  *inside* the extent as free handles realized at playback. Same shape as a note
  with free DOF. The score commits to *when the cloud is*, not *which grains
  land*. Not v1.

---

## 9. Novelty (defensible framing — claim it precisely)

The entire shipping granular category is **audio-domain granulation**: a grain
is a slice of a sample buffer (Quanta 2, Arturia FRAGMENTS, GrainDust,
Grainferno, GranuRise, Refractalizer). The field also **skews real-time /
performance**: modulation by LFO / envelope / step-seq / MPE, live input,
randomize-on-the-fly, grain layout as a transient consequence of modulators.

Binlod's two differentiators sit in an empty room:

1. **Event-domain, not audio-domain.** The grain is a MIDI event routed to an
   external instrument (BFD3, Battery, drum machine). The sink is the *font*;
   the lacing is the *text*. Rosetta facing downstream — sinks are projection
   charts of one authored pattern.
2. **Baked & editable, not live & ephemeral.** Existing plugins cannot hand you
   the grain layout as an object — it's a side effect of modulators. Binlod's is
   a document: λ(t) authored offline, deterministic, openable, with jittered
   velocity + CC as additional sampled value curves. Infinite lookahead at no
   latency cost — acausal shaping a real-time engine structurally cannot do.

**Honest claim (narrow, defensible):** not "density-driven MIDI" (Max patches /
probability sequencers exist), but **granular density as the authoring paradigm
for an editable, baked event score that drives external instruments** —
audio-grains and MIDI-clusters as the same operation differing only in sink.

---

## 10. Build order (dependency graph, not linear)

The user's planned track:
1. Pending Tabota work (lens frames).
2. Extensive graphic tools (pen tool with splines, etc.).
3. Plain f(x) layer for general use.
4. MIDI integration.

Corrected dependencies:

- **Generator core** (rate curve λ(t) → integrate → place grains at equal area)
  depends on **nothing** in Tabota. Standalone math — the tempo engine pointed
  sideways. Buildable & provable in a Node harness today.
- **f(x) layer splits on a fork:** is the density curve **drawn freehand**
  (needs pen/spline tooling → step 3 depends on step 2) or **composed from
  shapeK primitives** (reuses tempo vocabulary → needs neither)? The MVP is the
  **parametric** path: linear/exp/log/smooth + head/tail. The freehand f(x) is a
  later enrichment gated on the pen tool.
- Net: graphic-composer work (1–2) proceeds on the Tabota track for its own
  sake; the **generator core proceeds in parallel as a standalone** (waits on
  nothing). They meet only at MIDI integration. Not blocked either direction.

---

## 11. The MVP (Binlod v0 — generator test harness)

**Framing:** this is a **generator test harness**, not the product shape. The
product is authored-offline; the harness borrows MIDI's note-ons so authoring
need not be built yet. Take a MIDI **file** in, *not* a live stream — a
real-time MIDI-in→out granulator is exactly the live-performance effect Binlod
differentiates itself from. File-in / baked / file-out keeps it offline from
line one. The MVP is secretly a **headless point-duration note** (anchor pinned
at note-on, head/tail as the handles) — the throwaway prototype is the real
element with no chrome.

**Spec:**

```
MIDI file in
  → each note-on becomes an anchor point (onset grain optional — §5a)
  → params: head length, tail length, before-shape, after-shape, density
  → integrate λ over [anchor − head, anchor + tail]
  → emit grain note-ons (carry pitch through; scatter velocity by a value
    curve + jitter)
MIDI file out
```

No Tabota, no pen tool, no audio. Exercises **only** the spray — the novel,
risky part — and the exact engine everything else wraps.

---

## 11a. Build deltas (2026-06-28)

The MVP shipped, then grew an editing shell. What build added beyond §11's
headless spec:

- **Per-heap overrides.** Params are no longer global-only. Each heap carries a
  sparse `ov` dict; effective value = `ov[key] ?? global`. Empty on load. The
  side panel is **scope-swapping** ("scope is the door"): nothing selected edits
  the global default; a selection edits those heaps' overrides; mixed values show
  an ember marker. The global/per-heap split echoes §6's region-vs-note shape but
  on the **param** axis (which global default vs which per-heap value), not the
  **value-curve** axis — §6 proper (a note's internal shape vs a voice lane)
  remains open (§12.1).
- **Per-heap stable-id seeding.** Determinism is keyed to a stable heap id, not a
  shared rng walk: `heapSeed(seed,id) = ((seed·2654435761) ^ (id·40503) ^
  0x9e3779b9) >>> 0`. So add / delete / move / re-roll of one heap never reshuffles
  another's scatter — the precondition that makes per-heap overrides and editing
  non-destructive, and one the original walk-based bake quietly violated.
- **Editing shell (chrome arrived).** §11 called the prototype "the real element
  with no chrome"; it now has chrome, borrowed from the Roll on purpose:
  select-modes (replace / include / exclude, Shift-add / Alt-remove), marquee
  incl/excl, x/y zoom, ruler-drag scroll, undo/redo. Move is **time-only** (pitch
  never re-homed); add snaps to **occupied pitches only**. Binlod is a supplement
  to the Roll and a local MIDI editor, not a general MIDI editor — the chrome
  stops at that boundary.
- **Audition.** A Web-Audio preview (noise-burst for ch9, triangle for pitched)
  for feel-testing the bake. Not a sink — the sinks remain file-out MIDI (and,
  later, audio). Flagged: needs an ear.

None of this touches the cores in §3–§8: one rate curve emits instants, value
curves are sampled there, the bake is baked-and-editable. The deltas are shell
and determinism, not model.

## 12. Open forks (next decisions, in rough order)

1. **Value-curve composition rule** (§6): note's internal shape vs voice lane
   curve — multiply / override / add. Port from tempo region-vs-note resolution.
2. **MVP grain payload as concrete JSON** — note number, velocity, CC bundle,
   and (for percussion) which articulation / sample slot in the target. The
   percussion-axis maps directly: y-chart picks kit piece, λ(t) picks when,
   velocity curve picks how hard.
3. **Read-only vs writable Binlod for v1** — sets whether cross-page state sync
   (BroadcastChannel / SharedWorker) is needed at all.
