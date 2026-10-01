# AOI Camera Upgrade — SWIR Line Scan Project Context

**Sources:**
1. Your alignment-meeting notes (15 Sep 2026)
2. `260915_SWIR_Line_Camera_Kickoff.pdf` — official kickoff deck, Marcus Wong, ViTrox (14–15 Sep 2026)
3. `1_5_5_X_SWIR_Setup_Guide_Line_RND_Version.pdf` — existing SW setup guide for the current area-scan SWIR feature (background/baseline reference only — describes the camera being replaced, not the new line-scan system)
4. Your mechanical-site follow-up meeting notes (~1 week ago) — new risks not covered in sources 1–3, see Section 12

**Status legend:** ✅ Resolved by kickoff deck · 🔁 Refined/corrected vs. your notes · 🟡 Still open

---

## 1. Project Summary

**Project:** Skyworks DSM board inspection — Area Scan → Line Scan. **Target delivery: early Q1.**

The kickoff deck adds the "why": the current area-scan system takes 55 seconds per panel using 138 area-scan shots, driven by 2,520 boards per panel, each only 0.75×0.96mm — small boards and a small field of view mean many shots, which becomes a bottleneck as throughput targets rise. ✅

Stated goal: continuous encoder-synced line scan, a pixel-shift SWIR sensor giving 2× resolution on the same optics, fewer passes for higher throughput, and a common Dalsa-plate interface for future sensors. ✅

---

## 2. Camera & Sensor

**Confirmed / expanded:**
- Camera: **i-tek TPL2KCL-40KS2**, SWIR InGaAs line scan, spectral range 950–1700nm
- Native: 1024px at 12.5µm; pixel-shift: 2048px at 6.25µm effective — these are sensor-plane (chip) pixel sizes
- Max line rate 40kHz, global TEC-cooled shutter, Camera Link interface
- 🔁 **Important clarification your notes didn't distinguish:** there are two different pixel-pitch numbers in play — the number on the sensor chip itself (12.5µm → 6.25µm after pixel-shift) vs. the number projected onto the PCB through the lens (5µm → 2.5µm after pixel-shift). The lens is described as 5µm native projection, giving 5µm native (1K) and 2.5µm pixel-shift (2K) resolution, over a ~5.1mm scan width — this matches your notes almost exactly, just now sourced to the Vision Optics team's actual slide.
- ✅ Validation baseline confirmed independently: the setup guide lists the **existing** SWIR camera as 5MP at 2560×2048, which — cross-checked against the kickoff FOV table (8.94×7.09mm at that resolution) — works out to ≈3.49µm/pixel, matching your "vs 5MPx 3.5um" note exactly.

**Still open:**
- 🟡 Formal validation protocol for 2.5µm pixel-shift image quality vs. the 3.5µm baseline is not yet defined — still needed (kickoff Next Step #1, "Vendor Q&A with i-tekvision," is the closest lead on this).

---

## 3. Resolution & Motion Synchronization

🔁 **This section needed real correction — the kickoff deck's numbers are more relaxed than your notes assumed, and here's why:**

- Your notes assumed gantry movement needed to stay within 2.5µm per snap, and that the encoder needed upgrading to 0.5µm. **Neither is what was actually designed.**
- The motor encoder used is 1µm/pulse (quadrature output), unchanged from what's already available — no encoder hardware upgrade needed.
- An Encoder Counting Unit applies a Division Factor of 5 (5 encoder pulses → 1 trigger pulse) before it reaches the camera/light trigger stage.
- The along-scan (Y-direction) target line pitch was deliberately set to 5µm — matching the lens's native resolution — not 2.5µm. The 2.5µm improvement is real, but it only applies **across** the scan line (X, via pixel-shift interleave), not along it.
- **Why 5µm and not 2.5µm in Y:** this isn't a compromise, it's a hard ceiling — the operating line rate is derated to 32kHz (80% of the 40kHz max), giving a 31.25µs line period, and 5µm ÷ 31.25µs solves to a 160mm/s stage speed. Doubling the along-scan resolution to 2.5µm would require doubling the trigger rate to 64kHz — above the camera's 40kHz ceiling. So the sensor's cross-scan resolution boost is essentially "free" (via pixel-shift), while along-scan resolution is capped by camera line rate, not gantry precision.
- Self-consistency check the deck itself runs: 160mm/s stage speed × 1µm/pulse encoder resolution = 160,000 pulses/s; divided by the Division Factor of 5 = 32,000 triggers/s = 32kHz, matching the operating line rate exactly — the mechanical and trigger paths are confirmed self-consistent.
- Exposure time is set to ≤80% of the line period (25µs), leaving a 20% guard band for trigger/readout.
- Capture only happens in the constant-velocity window — no capture during acceleration or deceleration — with a flat +30% cycle-time overhead applied for the accel/decel ramps.

**Still open:**
- 🟡 The actual ramp profile (accel/decel rate, ramp distance) is explicitly marked TBC with the motion/mechanical team — the +30% is a placeholder, not a measured number. Kickoff Next Step #3 ("Confirm encoder + accel spec," owned by Gary) is the direct follow-up on this.
- 🟡 The technical reference slide references "Kickoff Action Item #9," but the Next Steps slide only lists 7 numbered items — worth asking whether there's a longer action item list this deck doesn't fully show.

---

## 4. Lighting System

🔁 **Your "one light per snap?" question turns out to map directly onto a phased plan already laid out in the kickoff deck:**

- Baseline / simplest to commission: single channel, G-Sync Pro drives the LED continuously — one channel, always on. Timing: light always on, exposure (≤25µs) fits inside each 31.25µs trigger period.
- Two dual-channel paths are proposed, and they map onto the two options you wrote down:
  - **Per-Pass (your "rescan with different angles")** — ✅ marked as the **recommended starting point**: G-Sync Pro toggles the LED channel once per pass — pass 1 bright-field, pass 2 dark-field — with line-level timing unchanged from single-channel, and called "low risk".
  - **Per-Line (your "split lighting stride within a row")** — 🟡 explicitly still open: this is labeled "Advanced — feasibility TBC," where the Light Pulse Generator alternates strobe A/B every trigger, each channel effectively samples at half rate, and switch + LED settle + exposure must all fit inside 31.25µs — flagged as an "open item — confirm with Gary / G-Sync Pro spec".
- On exposure/heat: the deck flags that exposure is very short compared to conventional (0.025ms vs. ~5ms, ~200× brighter), so heat dissipation matters — recommending starting at 0.05ms (50µs) rather than jumping straight to the 25µs worst-case-derived value. This directly answers your exposure/heat note.
- **Optical geometry, new info not in your original notes:** the camera sits on the surface normal (0°); the SWIR line light is offset 8° from normal, focused onto the same scan line via a cylindrical rod lens. A separate coaxial (beam-splitter) light path is also shown — an LED array fires sideways into a 45° partial mirror, which redirects the beam straight down through the aperture onto the die, coaxial with the lens; reflected light returns through the same partly-transparent splitter into the camera. *(My inference, not stated outright in the deck: this pairing plausibly maps to bright-field = coaxial channel, dark-field = 8°-offset channel — worth confirming with Gary rather than assuming.)*

**Still open / not found in either source:**
- 🟡 Your "offset between planes ≤1.25µm" figure and the "25% light throughput" figure from your notes don't appear verbatim in either document. Given the newly-revealed 8° geometry and dual-channel timing constraints, they're plausibly related to one of those — but that's a guess, not a confirmed match. Worth asking directly whether these numbers came from a separate optical-design calc not in this deck.
- 🟡 **The 8° slant scan itself is called out as an unresolved item** — it appears as an explicit "⚠ 8° slant scan — open" line under the Vision Optics team's responsibilities.
- Useful background: the setup guide shows the *existing* area-scan SWIR system already runs a 9-angle lighting scheme (SpotLight + Coaxial + Ring Light combinations), all currently manually tuned with no auto-calibration — so the line-scan project's single/dual-channel starting point is a deliberate simplification versus what the platform is capable of long-term, not a limitation of the hardware.
- 🔴 **Update — this simplification is now a confirmed, not speculative, risk to inspection capability.** See new Section 14: reducing lighting angles turns out to conflict directly with how the existing die-crack detection algorithm actually works, and the throughput math changes materially depending on how many lighting conditions turn out to be necessary.

---

## 5. Image Retrieval & Software Integration

- Ownership is now clear: the Smart GUI team, led by TJ, owns image acquisition flow, view arrangement, and inspection logic for this project.
- Background from the existing SW architecture (not line-scan-specific, but the foundation the Smart GUI team will likely extend): the current system already distinguishes views by a `tool_head_type` field (0 = default mono, 1 = SWIR) and separates Normal / MZA / SWIR view compilation, with mixing of refdes types across a single view currently not supported.

**Still open:**
- 🟡 Your original note — that raw buffer retrieval and view-compilation crop don't currently match for line-scan output — isn't directly addressed in either document. The acquisition pipeline (Section 6 below) explains how the frame gets built and handed to software, but not how it reconciles with the existing view-compilation format. This still needs a design conversation with TJ's team specifically for line-scan (kickoff Next Step #7, "Begin Smart GUI wireframes," is the relevant follow-up).

---

## 6. Gantry, Encoder & Trigger Chain

🔁 **The kickoff deck describes a considerably more specific pipeline than "controller board triggers snap":**

Motor Encoder (1µm/pulse) → Encoder Counting Unit (÷5) → G-Sync Pro (master vision controller — generates both the camera pulse and the light pulse) → Camera Trigger (line-scan camera, 2×1K sensor rows, 40kHz max) and Light Trigger (LED driver, strobes in sync with each line) → raw line data → Frame Grabber (combines rows, builds frame, buffers via PCIe DMA) → PC/software (IKapLibrary/eGrabber receives the completed image buffer).

At the sensor level: bytes 0–1023 come from Sensor Row 1 (0 shift), bytes 1024–2047 from Sensor Row 2 (½-pixel shift); the frame grabber interleaves them into a single 2048-pixel combined line at 6.25µm effective (sensor-plane) pitch, and a frame is committed once the panel-length line count is reached.

**Still open:**
- 🟡 Frame rate on the G-Sync Pro side and the number of lighting channels (1 vs 2) are both still marked TBC in the team-responsibilities slide, even though the lighting section above shows a clear phased recommendation — worth reconciling which one is authoritative.

---

## 7. Calibration

No update — neither document addresses **gantry mapping error correction for curved motion** specifically for the line-scan system. The setup guide's `.jscale` calibration section covers the existing mono/area-SWIR camera scale calibration (per tool-head-id), which is a different kind of calibration (camera-to-gantry offset, not curved-motion path correction). 🟡 Still fully open.

---

## 8. FOV & Throughput — new section from the kickoff deck

Skyworks DSM panel: 28×90 boards, 0.75×0.96mm each → panel ≈49.4mm × 179.0mm.

| Option | FOV | Boards/Scan | Scans |
|---|---|---|---|
| Existing Area Camera — 5MP | 8.94×7.09mm | 5×4 | 138 |
| Area Camera — 1MP | 4.97×3.73mm | 3×2 | 450 |
| Line Scan — 2K Pixel Shift, 1K native | 5mm width | 3 in a row | 10 |
| Line Scan — 2K | 7mm width* | 4 in a row | 7 |

🟡 The 7mm figure is flagged by the team itself as needing reconciliation against the 5µm-lens math, which computes closer to ~5.1mm — this is Kickoff Next Step #6 ("Reconcile FOV assumptions," owned by Vision Optics).

Throughput: operating at 32kHz with a flat 30% accel/decel overhead, a worked example gives 1.12s × 1.30 = 2.9s per pass, × 10 passes = 29s total — versus the 55.2s baseline, claimed as 2.7×–5.4× faster than the current area-scan system even in the worst case.

---

## 9. Roles & Responsibilities — now with real names

| Team | PIC | Scope |
|---|---|---|
| Project Owner | **TJ** | Owns technologies & deliverables, aligns PMO on planning, coordinates all teams, oversees cost |
| PMO | **Chean Gay** | Owns project timeline; supports Project Owner and costing prep |
| Vision Optics | **Weng Yeong** | Camera & lens, Camera Link protocol, 5µm lens resolution, telecentric lens (on loan); open item — 8° slanted scan |
| Mechanical | **Weng Yeong / Kin How** (How, TBC) | Camera & lens holder, light source holder, Dalsa plate common interface |
| Line Scan Integration | **Gary** | Encoder-based triggering, G-Sync Pro acquisition (frame rate TBC), G-Sync Pro lighting (1 or 2 channels, TBC) |
| Smart GUI | **TJ & team** | Image acquisition flow, view arrangement, inspection logic |

**New names surfaced in the Master Gantt (Section 15) — roles inferred from task context, not stated explicitly, so worth confirming:**

| Name | Appears against | Inferred role |
|---|---|---|
| **Jian Yan** | Co-listed with Chean Gay as owner of "Machine Integration and Delivery (SWIR)" | Likely PMO/delivery co-owner alongside Chean Gay — worth confirming scope split between the two |
| **Yang** | "Work with Yang on I/O with SWIR lighting controller" (Encoder Triggering, Non-Encoder Trigger Board Route) | Likely electrical/firmware engineer supporting Gary's team on lighting-controller I/O |
| **BD Wan Ling** | "pending to work with BD Wan Ling, convert PO to line scan??" (SWIR RND #2 build) | Business Development — see Section 15c, this ties to a real commercial question (repurposing an existing PO'd machine), not just an engineering task |

---

## 10. Action Items — Kickoff's actual Next Steps, plus what's still uncovered

**Official next steps from the kickoff deck:**

| # | Action Item | Owner |
|---|---|---|
| 1 | Vendor Q&A with i-tekvision | Weng Yeong |
| 2 | Finalize frame grabber | Weng Yeong |
| 3 | Confirm encoder + accel spec | Gary |
| 4 | Decide lighting approach | Gary |
| 5 | Kick off mechanical design | Weng Yeong / Kin How |
| 6 | Reconcile FOV assumptions | Vision Optics |
| 7 | Begin Smart GUI wireframes | TJ & team |

**Gaps neither document closes yet — worth raising at the next sync:**

| # | Gap | Why it matters |
|---|---|---|
| 8 | Define pass/fail validation protocol for 2.5µm image quality vs. 3.5µm baseline | Not owned by any of the 7 items above; closest to #1 (vendor Q&A) but not the same task |
| 9 | Trace the source of your "1.25µm plane offset" and "25% coaxial throughput" figures | Not found in either document — may be from a calc/conversation not captured here |
| 10 | Reconcile "8° slant scan — open" (Vision Optics) with the dual-channel bright/dark-field lighting plan | Both are open items that may be the same underlying question |
| 11 | Clarify which is authoritative: the lighting recommendation (single→dual-per-pass→dual-per-line) vs. the "1 or 2ch TBC" still shown in the team-responsibilities slide | Two parts of the same deck don't fully agree on how settled this is |
| 12 | Locate "Kickoff Action Item #9" referenced in the motion-profile slide — the visible Next Steps list only goes to 7 | Possible missing appendix or a numbering mismatch worth flagging back to Marcus |
| 13 | Define how line-scan output will reconcile with the existing Normal/MZA/SWIR view-compilation architecture | Feeds directly into Smart GUI wireframing (#7) but isn't scoped as its own task yet |
| 14 | Get the actual DOF spec of the chosen lens/working distance, and the actual/spec'd conveyor rail coplanarity tolerance | Raised in the mechanical follow-up (Section 12a) — DOF is a hard optical limit; if the two numbers aren't both known, "is this a real risk" can't be answered either way |
| 15 | Confirm whether the 4.1667µm object-pixel figure (Section 12b) is the *actual current* lens plan or a contingency being stress-tested | Materially changes whether Section 3's "clean 5µm / 2.5µm, whole-number-divisible" design still holds as stated |
| 16 | Get the empirical basis for the reported 0.5µm encoder-precision stability ceiling (Section 12c) — vibration data, servo bandwidth test, or vendor spec? | Current 5µm line-pitch plan sits safely inside this ceiling, but the ceiling itself is asserted from mechanical-team experience, not yet documented |
| 17 | Confirm how many lighting conditions are actually required for reliable die-crack detection on line-scan hardware, and whether Per-Line multiplexing is feasible | Per Section 14c, this directly determines whether the project is still faster than the 55.2s area-scan baseline — 2+ conditions via Per-Pass repeats could erase the throughput advantage entirely |
| 18 | Test whether the 8°-offset "bar light" candidate behaves as directional (spotlight-like) or diffuse — don't assume it groups with coaxial | Section 14d — if it's closer to spotlight, it may partially resolve the crack-visibility/stability conflict rather than sitting on the same side of it |

---

*All ✅/🔁 items above are sourced directly from the kickoff deck and setup guide; 🟡 items are genuinely unresolved as of these two documents and are good candidates for your next meeting agenda.*

---

## 11. Milestones (Follow-Up Meeting) — Completion Criteria & Fallback Plans

🔁 **Update from the Master Gantt (Section 15): dates are no longer pending.** Milestone 1 maps to the Gantt's "Offline Station Benchmark Test (SWIR)" — **15-Sep to 30-Oct-2026**. Milestone 2 maps to "Machine Integration and Delivery (SWIR)" — **15-Sep-2026 to 28-Feb-2027**, with the final "Software Integration and Tester buyoff" sub-task ending **15-Feb-2027**. The M1-gates-M2 dependency you asked about is also now explicit in the sheet itself: Row 5 (Machine Integration) is marked as **Depends On Row 1** (Offline Bench Test) — so this isn't just a recommendation anymore, it's recorded as a formal dependency.

**One new discrepancy this surfaces, worth naming:** the original kickoff goal was "early Q1" delivery (Section 1). A Software Integration/Tester buyoff date of 15-Feb-2027, with Machine Integration running to 28-Feb, reads as **end of Q1**, not early Q1. Worth flagging this gap explicitly to TJ/Chean Gay rather than letting "the Gantt has dates now" be mistaken for "the original goal is on track" — those are two different questions.

Your boss set two milestones without a completion date yet ("pending"). Before anything else, get a date attached — work backward from the "early Q1" delivery target, and note the dependency below: **M1 gates M2.** If Milestone 1 doesn't land, Milestone 2 as currently scoped shouldn't start on schedule, because it commits real build effort around a camera/optics choice M1 is meant to validate. Worth saying that explicitly to your boss now, not after M2 is already underway.

### 11a. Milestone 1 — Bench Image-Quality Validation
**Objective:** Quick offline setup (loaned camera, frame grabber, light, lens + any available motion rig) to confirm image quality, including whether the 2K pixel-shifted image is usable.
**Owner:** Weng Yeong & Gary (exec) / TJ (oversight)

**What "done" should require:**
| # | Completion Criterion | Why it matters given the spec |
|---|---|---|
| 1 | Written pass/fail threshold defined **before** running the test (resolution, contrast/MTF, SNR — vs. the 3.5µm/5MP baseline) | This is exactly the "validation protocol" flagged as undefined in Section 2/gap #8 above — without it, "confirm image quality" is subjective |
| 2 | Pixel-shift interleave specifically checked for artifacts (ghosting, moiré, misregistration between Row 1 / Row 2) | The 2.5µm gain depends on Row 1/Row 2 combining cleanly — a bad interleave doesn't just reduce resolution, it can actively corrupt the image |
| 3 | Same physical test samples imaged on both the new SWIR line-scan camera and the existing 5MP/3.5µm area-scan camera | Needed for an apples-to-apples comparison, not two different baselines |
| 4 | Test run at a stage speed representative of the production target (160mm/s), not just whatever the bench rig can do at rest or slow-crawl | A slow bench pass can look fine and still miss motion blur, insufficient exposure, or vibration issues that only appear near 32kHz/160mm/s |
| 5 | Confirm material contrast on real production-relevant samples (actual die/board), not just a generic resolution test target | The whole reason for SWIR (950–1700nm) is a specific material contrast — a resolution chart alone won't prove that out |
| 6 | Explicit go/no-go decision recorded on native 1K mode (5µm, no pixel-shift) as well as 2K pixel-shift mode | Gives you a documented fallback path (see below) instead of discovering it under time pressure |

**Watch out for:** the loaned telecentric lens has a limited loan window (explicitly "on loan" in the kickoff deck) — build the test plan around that constraint, not around calendar convenience.

**Fallback plan if image quality fails:**
- **If only the pixel-shift interleave is bad, base optics are fine:** fall back to native 1K mode (1024px, 5µm) — same camera, same lens, no pixel-shift firmware/interleave in play. Lower resolution, but it's a mode the sensor already supports, not a new hardware ask. Re-run the FOV/throughput math (Section 8) at 5µm to see if the throughput case still holds.
- **If image quality is marginal either way:** don't force a go/no-go on limited samples — request a short extension (even a few days) with an expanded sample set before committing to Milestone 2's build effort. Cheaper to extend M1 than to discover the problem after Gary's team has built the full rig.
- **If material contrast genuinely isn't there at 950–1700nm for these dies:** this is a camera/vendor-selection-level failure, not a tuning problem — escalate to Weng Yeong for a vendor Q&A follow-up (ties to kickoff Next Step #1) and flag cost/schedule impact to TJ before any further build spend.

---

### 11b. Milestone 2 — Full System Integration
**Objective:** Build the R&D machine with encoder-driven triggering into G-Sync Pro, finalize the trigger/capture architecture, and integrate motion + lighting + camera control into VTHAL.
**Owner:** Gary (exec) / TJ (oversight)

**What "done" should require:**
| # | Completion Criterion | Why it matters given the spec |
|---|---|---|
| 1 | Accel/decel ramp profile confirmed with the motion/mechanical team (currently a placeholder +30% overhead) | The capture window's start/end is defined by where constant velocity begins/ends — get this wrong and you either capture distorted lines during ramp, or waste usable scan length being overly conservative |
| 2 | Lighting approach locked for this build (single-channel / dual-per-pass / dual-per-line) before wiring the trigger architecture | Per-line dual-channel needs G-Sync Pro's Light Pulse Generator to alternate strobes inside a single 31.25µs window — a materially different trigger design than single-channel. Don't leave this open while building |
| 3 | G-Sync Pro sustained 32kHz trigger rate validated on real hardware, not just on paper | A dropped trigger mid-panel corrupts the whole frame, since a frame = N lines stacked over the full panel length |
| 4 | Frame grabber confirmed to interleave Row 1/Row 2 into a clean 2048px line at full line rate, over a full multi-second panel scan (not a short burst) | Buffer overflow or PCIe DMA drops are more likely to show up over a sustained ~3s pass than a quick bench test |
| 5 | A defined path for how line-scan frames land in VTHAL's existing view-compilation model (`tool_head_type`, Normal/MZA/SWIR view logic) — even if it's a deliberately interim/simplified path | **Updated — this is more foundational than it first looked.** Per a clarification from the OP2 Semicon/Laser Line Scan project (tracked separately, `op2-semicon-laser-linescan-context.md`), VTHAL is the universal middleware between your software and every camera + frame grabber, and **it has never supported communication with a line-scan camera before — for either project.** So the open item here isn't just "extend the view-compilation model," it's "does VTHAL's core camera-communication layer handle line-scan acquisition at all yet." Worth confirming directly with whoever's liaising with the VTHAL team whether that foundational support exists or is still being scoped — this changes how big a task item #5 actually is |
| 6 | Mechanical build (Dalsa-plate interface, camera/lens holder, light source holder) signed off as stable by the Mechanical team before integration testing starts | Integration testing on a mechanically-unstable rig will produce noisy results that look like a software/trigger problem but aren't |

**New: shared cross-project dependency worth tracking explicitly.** The OP2 Semicon/Laser Line Scan project is independently liaising with the VTHAL team on this exact same gap (their camera is a different sensing technology, but the underlying ask — "make VTHAL understand line-scan" — is the same one). Whichever project's camera the VTHAL team studies first likely sets the technical pattern the other inherits, and VTHAL's own effort estimate is a shared critical-path item for both projects, not something either project controls unilaterally. Worth finding out if this is being coordinated across both projects or discovered independently by each.

**Watch out for:** don't let "trigger/capture architecture finalized" get declared while the ramp profile, lighting channel count, and G-Sync Pro frame rate are still marked TBC on the team-responsibilities slide — those TBCs are inputs to the architecture, not details to patch in afterward.

**Fallback plan if integration fails or underperforms:**
- **If 32kHz can't be sustained reliably:** drop to a lower operating line rate (with a correspondingly larger line pitch/lower resolution, or fewer boards per pass) rather than forcing it — then re-run the throughput math to confirm the project still beats the 55s baseline by a meaningful margin before committing to that as the new normal.
- **If dual-channel lighting (either mode) can't be made to work in time:** fall back to single-channel, always-on — it's already the deck's own documented "simplest to commission" baseline, so this isn't an improvised fallback. **Updated — this caveat is no longer speculative:** per Section 14, the existing area-scan crack-detection approach relies on multiple spotlight-current exposures specifically because a single fixed lighting condition unreliably captures die cracks. Confirm with whoever owns die-defect-detection before locking in single-channel — this is now a known, not hypothetical, risk to detection quality.
- **If VTHAL/view-compilation integration is the long pole:** decouple it from the rest of Milestone 2. Get the mechanical + trigger + capture pipeline proven first (raw frames in hand), and treat "clean integration into the existing inspection view architecture" as a fast-follow rather than a blocker — otherwise a software-architecture problem ends up stalling hardware bring-up that's actually going fine.
- **If cycle-time overhead comes in much worse than the +30% placeholder (e.g., 60%+):** this changes the project's business case, not just an engineering parameter — re-run the throughput comparison and flag it to TJ/Chean Gay directly, since it may affect whether "early Q1" is still realistic.

---

*Dates are now known (Section 15), but the M1→M2 handoff still deserves a real go/no-go conversation, not an automatic rollover — the Gantt records the dependency, it doesn't validate that M1's actual results will support moving on to M2 on schedule.*

---

## 12. Mechanical-Site Findings (Follow-Up Meeting, ~1 week ago) — New Risks & Assessment

These three items are new — they don't appear in the kickoff deck or setup guide, and they surface real tension between the optical resolution target and what the mechanical side can actually deliver. Treat all three as 🟡 open until quantified; I can validate the *engineering logic* behind each, but not the specific numbers, since none of those are in any document I have.

### 12a. Conveyor Rail Coplanarity vs. Line-Scan DOF

**Refined statement:** The conveyor is built from two rails. If the two rails aren't perfectly co-planar (i.e., one sits slightly higher than the other), the board's imaged surface will vary in height across the scan width by roughly that same amount. For the line-scan image to stay in focus, that height variation needs to be smaller than the optical system's Depth of Field (DOF) — the range of height over which the image stays acceptably sharp.

**How true is this?** The underlying principle is correct and not something to second-guess: DOF shrinks quickly as resolution improves — it's a basic property of imaging optics (roughly, DOF falls off with the square of the resolving power), so a system aiming for 2.5–5µm resolution will generally have a much shallower DOF than one imaging at 3.5µm/pixel on the existing area-scan camera. So "line-scan SWIR DOF is typically very low" is a reasonable expectation, not an exaggeration. What I can't confirm is whether it's actually a *problem* for this specific machine — that depends entirely on two numbers neither document states:
1. The actual DOF of the selected lens at its working distance (should come from Weng Yeong / the i-tek vendor Q&A — this is a natural add-on to kickoff Next Step #1).
2. The conveyor's actual or spec'd rail-to-rail coplanarity tolerance (Mechanical team / Kin How).

**What can be done beforehand:** This is a hardware-only problem — no image-processing step can recover a genuinely out-of-focus region, so it has to be solved mechanically or optically, not in software:
- Get both numbers above before Milestone 1 concludes, so "confirm image quality" (Section 11a) has a real pass/fail bar for this specific failure mode, not just resolution/contrast in general.
- If coplanarity turns out to exceed DOF: options are (a) a mechanical leveling/adjustment fixture on the rails (shims or an adjustable rail), (b) a lens/working-distance choice with more DOF margin (likely a resolution trade-off), or (c) a hold-down/flattening mechanism that forces the board flat regardless of rail height variance.
- Worth adding an explicit test to Milestone 1: image a test board with a *known, deliberately introduced* height offset between the two ends, to get an empirical sense of how much defocus actually degrades usable image quality — rather than relying on a specification number alone.

**Not clarified enough to call yet:** I don't have the lens's actual DOF number or the machine's actual/target coplanarity tolerance. Until both exist, "challenging" is a reasonable guess, not a confirmed risk — worth asking Weng Yeong and Kin How for these two numbers directly.

---

### 12b. Encoder Resolution vs. Image Resolution — the "Whole-Number Trigger" Problem

**Refined statement (cleaner phrasing):** For encoder-triggered line-scan acquisition — whether the camera is triggered in free-run or by hardware pulse — the along-scan sample spacing (the physical distance the gantry moves between one triggered line and the next) needs to be an exact whole-number multiple of the encoder's smallest step size. If it isn't, the trigger can't land at a perfectly even, repeatable physical interval every time; small rounding errors accumulate over the length of a scan, and the resulting image ends up geometrically stretched or compressed along the scan direction relative to the cross-scan direction.

**How true is this?** Yes — this is a real and well-known issue in encoder-synchronized line-scan imaging (it's the same underlying problem as line-rate/velocity mismatch in scanner and push-broom imaging systems generally). It's also good news for the *current* plan: recall from Section 3, the along-scan line pitch was deliberately set to 5µm, and the base encoder is 1µm/pulse — 5 ÷ 1 = 5, a clean whole number. That's exactly why it works with a Division Factor of 5. The design as documented in the kickoff deck doesn't have this problem.

**The new wrinkle:** you're describing a *different* scenario — a lens-selection roadblock producing 12.5µm (sensor pixel) → 4.1667µm (object-space), rather than the clean 5µm the kickoff deck assumes. 4.1667µm ÷ 1µm = 4.1667 — not a whole number, so this pitch cannot be hit exactly with an integer division of the 1µm encoder. This is a genuine conflict with the design in Section 3, not a minor rounding footnote.

**Update — now clarified (see Section 13):** confirmed as the actual pending choice, not a hypothetical. The ideal 2.5X lens (which would give the clean 5µm) turns out to be a highly-customized, hard-to-source spec in the market; the practical candidate is a 3X lens giving 4.1667µm (12.5 ÷ 3 = 4.1667, checks out exactly). So this is now a live design conflict, not a contingency — see Section 13b for the follow-on implications, including a new one: Milestone 1's bench validation may currently be running on the loaned *2.5X* lens rather than the 3X lens that's actually pending for production, which would mean M1's results don't carry over to the real optics.

**On your actual question — can post-processing fix the "overstretch"?**
Yes, with caveats:
- Because the mismatch between the achievable trigger pitch and the true 4.1667µm target is a *fixed, known ratio* (not a random or dynamic error), it can in principle be corrected with a simple linear rescale along the scan axis — not a complex distortion correction, just a one-time known scale factor.
- Useful concrete lead: the existing `.jscale` calibration file (from the setup guide, Section 4 of that digest) already carries a `cam_skew` and an `aspect_ratio` field per `tool_head_id`. That's very plausibly the exact mechanism meant for this kind of non-square-pixel correction — worth checking with whoever owns SW calibration whether the line-scan camera's future `tool_head_id` entry can just carry the correction factor there, rather than building a new mechanism.
- One real caution: the Die Defect Detection algorithm (setup guide, Section 10.1) filters on physical-unit thresholds — Crack Area Thres in µm, Width/Length Threshold — so this rescale needs to happen *before* those measurements are made, or defect sizing will be systematically wrong by whatever the stretch factor is. This is worth flagging to whoever owns the inspection algorithm work, not just the imaging pipeline.
- Rescaling does correct proportion/scale, but it cannot recover resolution that wasn't actually captured — if 4.1667µm is what you get, that's the real resolution; the correction just makes the image geometrically accurate at that resolution, not sharper.

**What can be done beforehand:** Push first on understanding *why* the ideal lens isn't available (cost, lead time, an optical/mechanical incompatibility?) — a lens swap that restores the clean 5µm relationship is simpler than building and validating a correction path. If the roadblock is truly unresolvable, scope the correction as a small, deterministic linear rescale (not a general distortion-correction problem) and confirm with SW calibration whether `.jscale`'s `aspect_ratio` field already covers this case.

---

### 12c. Gantry Encoder Precision Ceiling & Mechanical Stability

**Refined statement:** The gantry's position-triggering system currently supports pulse resolution down to 0.5µm reliably. Pushing finer than that risks instability in position compensation, because the gantry's mass makes it more sensitive to vibration — both from the floor (ground-borne) and from its own motion (e.g., motor cogging, acceleration-induced oscillation).

**How true is this?** The general engineering principle is sound and well-established in precision motion control: heavier gantries generally have lower mechanical bandwidth, and at very fine position scales, the practical resolution ceiling is usually set by vibration and structural resonance — not by the encoder's electronic counting resolution. A system can have a very fine encoder and still not achieve fine *positioning*, because real-world vibration amplitude can exceed the resolution you're trying to hit, turning the reading into noise rather than true position. So the concern itself is credible. What I can't verify is the *specific* 0.5µm number — that's empirical/measured data from the mechanical team's own testing, not something in any document I have, so it should be treated as a real but unverified data point until there's supporting evidence (a vibration measurement, a repeatability test, or a vendor spec for the stage).

**Good news for the current plan:** the project's committed line pitch is 5µm (Section 3), which is 10× coarser than the reported 0.5µm ceiling. As it stands, the current design doesn't need to push anywhere near this limit — worth stating that explicitly back to the team so this doesn't get treated as a blocker on the current plan. It becomes directly relevant only if a future along-scan resolution target moves finer (e.g., matching the 2.5µm cross-scan pixel-shift resolution in both axes, which would need roughly a 2.5µm line pitch — still coarser than 0.5µm, so likely still fine, but close enough to be worth testing rather than assuming).

**What can be done beforehand:**
- Ask for the underlying data behind the 0.5µm figure — knowing *why* (vibration test? servo bandwidth analysis? vendor datasheet?) tells you whether it's fixable (vibration isolation, a stiffer gantry, air bearings) or a hard ceiling for this machine generation.
- Since the current 5µm plan has comfortable margin, there's no urgency to resolve this before Milestone 2 — but it's worth logging now so it doesn't get rediscovered later if resolution targets tighten.
- **On Gary working independently at an offline station:** this is a sensible way to de-risk the trigger-generation engineering in parallel with the still-unresolved mechanical/lens questions above — reasonable to continue. One caveat worth raising with Gary directly: "offline station" and the real production machine likely differ in mass and vibration environment, so a positive result there is encouraging but should be re-validated once the actual Milestone 2 R&D machine is available, rather than treated as final proof of stability on the real hardware.
- 🔁 **Resolved by the Master Gantt (Section 15):** the gantry drive type — previously unspecified — is confirmed as a **ball screw**, not a linear motor. Worth noting for context: ball-screw drives generally carry more inherent backlash/compliance than direct-drive linear motors, which is broadly consistent with — not a contradiction of — the reported 0.5µm stability ceiling. It doesn't change the conclusion above (5µm has comfortable margin), but it's a plausible mechanical reason behind the ceiling, worth mentioning if you ask the mechanical team for the data behind that number.

---

### 12d. The Pattern Across All Three

All three findings are really one underlying tension, seen from three angles: **pushing resolution finer makes the mechanical tolerances tighter, and the machine's physical mass/construction is what's pushing back.** Finer resolution → shallower DOF (12a) → tighter coplanarity requirement. Finer/mismatched pitch → needs finer encoder division to stay whole-number-clean (12b) → runs straight into the mechanical stability ceiling (12c). Worth naming this explicitly in the next meeting rather than treating the three as unrelated line items — the real question underneath all three is *how much resolution the mechanical platform can actually support*, not just the camera/lens.

### 12e. Open Questions to Bring Back Before Making Calls

- What is the selected lens's actual DOF at working distance, and what is the conveyor's actual/spec'd rail coplanarity tolerance? (12a — both unknown, needed to know if this is a real risk at all). 🔁 **Partial update (Section 15):** the Master Gantt confirms DOF testing is explicitly scheduled as part of Milestone 1's "Free-run Image quality test" (Camera Testing: Min Exposure time, DOF) — so the plan now covers this, but the actual number is still pending since that task hasn't started.
- ~~Is the 4.1667µm object-pixel figure the camera/lens combination now actually being procured, or a fallback scenario under evaluation?~~ **Resolved (Section 13b): yes, it's the actual pending choice** — the ideal 2.5X lens is highly-customized/hard to source.
- **New:** is Milestone 1's bench validation currently using the loaned 2.5X (ideal) lens, or the 3X/4.1667µm lens that's actually pending for production? If it's the former, M1's image-quality results won't be representative of the real production optics (Section 13b).
- Can the existing `.jscale` `aspect_ratio` field absorb the along-scan rescale for the 4.1667µm mismatch, or does this need new SW work? Who owns confirming that — Smart GUI/TJ's team, or a separate calibration owner? (12b, 13b)
- What test or measurement produced the 0.5µm encoder-precision ceiling? Is it vibration data, a servo bandwidth limit, or a vendor spec? (12c)
- Is Gary's "offline station" representative of the real machine's mass/vibration environment, or a lighter test rig? Worth knowing before treating its results as predictive of the actual R&D machine. (12c)

---

## 13. Vision System — Component Selection (Pending, Not Finalized)

Latest compartment choices from your sourcing/BOM work — none finalized yet. Cross-checked against everything established so far.

### 13a. Camera — 2048 total sensor elements (2 rows of 1024), native or pixel-shift, pending price

**Correction applied:** the physical sensor has **2048 total light-sensing elements**, laid out as **2 rows of 1024 each** — not two separate camera variants with different photosite counts. This actually lines up exactly with the kickoff deck's own description of the i-tek TPL2KCL-40KS2 ("2K line-scan sensor, pixel-shift design: 2 rows of 1K sensors, Row 1 = 0 shift, Row 2 = ½-pixel shift"), so this is the same camera, now described more precisely than my last pass had it.

Where I still need your confirmation rather than assuming: the kickoff deck states **native resolution = 1024px @ 12.5µm** and **pixel-shift resolution = 2048px @ 6.25µm effective**. My working assumption, reconciling that with your correction, is that "native" means reading the two rows *without* the half-pixel-offset interleave applied (so you get 1024px-equivalent resolution even though 2048 elements physically exist), while "pixel-shift" is the processing mode that combines the two rows' known half-pixel offset into a genuinely higher-resolution 2048px line. **Is that the right reading, or does "native" mode actually output at 2048px too (just without the resolution enhancement the offset gives you)?** This matters beyond terminology — Section 11a's Milestone 1 completion criteria explicitly calls for a go/no-go recorded on "native 1K mode" as a fallback path, so I want the fallback's actual output resolution pinned down correctly before that milestone gets evaluated.

**Unchanged from before:** the "pending price" question — if pixel-shift is a costed option that might get dropped for budget, that's still worth an explicit yes/no, since the kickoff deck's resolution/throughput case (Sections 2, 8) assumes it's active.

### 13b. Lens — 3X (4.1667µm), ideal 2.5X (5µm) not sourceable

This confirms Section 12b's open question: the 4.1667µm pitch is the real pending choice, not a hypothetical. Logged rationale: the ideal 2.5X lens is a highly-customized spec that's hard to find in the market — useful to have that reasoning on record for later, since someone will ask "why not just use the lens the kickoff deck assumed."

**New risk this surfaces:** the kickoff deck describes the loaned telecentric lens (Milestone 1's bench test lens) without giving its exact reduction ratio. If that loaned lens is the *ideal* 2.5X (plausible — it's the one described as "on loan," consistent with a hard-to-source specialty item) rather than the 3X you're now pricing for production, **Milestone 1's image-quality results would be validating a different lens than what actually ships.** Worth confirming directly: which lens is on the bench right now?

**Practical path forward:** since the clean 2.5X isn't available, the along-scan rescale (linear, deterministic — see Section 12b) is the realistic fix. Worth formally scoping that now rather than treating it as a footnote, and re-confirming whether `.jscale`'s `aspect_ratio` field can carry it.

### 13c. Frame Grabber — Euresys Grablink Duo

Checked against Euresys' current published specs: Grablink Duo handles either two independent Camera Link Base cameras, or one Base/Medium/Full/72-bit/80-bit camera, with up to 1700 MB/s DMA bandwidth, 512 MB onboard buffer, and — notably — it runs on the **same eGrabber driver already named in the setup guide's Environment Setup section** (`Euresys eGrabber v24.09.0.5`), so this slots into the existing software stack rather than introducing a new driver dependency.

**Bandwidth sanity check:** at 2048px/line × 32kHz operating rate, even at a generous 16-bit depth, that's ~131 MB/s — under 8% of the card's 1700 MB/s ceiling. Even worst-case (40kHz max line rate, 2048px, 16-bit) only reaches ~164 MB/s. Bandwidth is not a risk here, by a wide margin.

**Worth confirming:** which Camera Link configuration (Base / Medium / Full) the i-tek camera actually outputs on its 2-row (2-tap) design — Grablink Duo supports all three, so this is more a documentation-completeness item than a real risk, but worth having on record.

**Side benefit worth noting:** Grablink Duo's dual-camera support lines up with the broader Primary/Auxiliary dual-camera direction already visible in the DSS setup guide (separate from this project, but same underlying platform) — not a requirement here, just a reason this choice has headroom beyond the immediate need.

### 13d. Cable — Camera Link

Matches the kickoff deck's stated interface for the i-tek camera exactly. No new risk — this is the expected, consistent choice given the camera and frame grabber above.

### 13e. SWIR Light Source — bar light vs. coaxial beam-splitter (either/or)

These map directly onto the two lighting geometries already shown in the kickoff deck's Optical Configuration slide (Section 4/12): the 8°-offset external light focused via a cylindrical rod lens (your "bar light"), and the coaxial beam-splitter box (your "coaxial light beam splitter") — same two concepts, now framed as a vendor/product decision rather than just a geometry diagram.

**This raises a real conflict worth flagging directly:** the kickoff deck's dual-channel lighting plan (Section 4) — "Pass 1 = bright-field, Pass 2 = dark-field" — reads as needing *both* a coaxial-style (bright-field) source and an off-axis-style (dark-field) source present at once, switching between them per pass. If you're choosing **one** physical light source type instead of both, that dual-channel bright/dark-field plan may not be achievable as currently described. **Worth asking Gary/Weng Yeong directly: does "either/or" here mean picking one geometry for the whole system (which would walk back the dual-channel plan), or is Channel A/B in the kickoff deck actually referring to something else — e.g., two drive settings on the same light type — and the bar-vs-coaxial choice is a separate, single-channel decision?** I don't have enough here to call which one is intended.

**One data point in coaxial's favor, for what it's worth:** the existing area-scan SWIR system (setup guide, Section 5) already runs a coaxial light channel among its 9 fixed angles (`SWIR_CL1–CL3`) — so coaxial SWIR lighting isn't a totally new concept at ViTrox, even though the line-scan-specific beam-splitter box hardware would still be new. The bar-light/8°-offset geometry has no equivalent precedent in the existing system. That's a mild point toward lower integration risk for coaxial, not a recommendation on its own — the bright/dark-field question above matters more.

### 13f. SWIR Lighting Controller — same vendor as the light source

Sensible default — matching the controller to the light source vendor avoids cross-vendor protocol/driver mismatches. No new risk here; this choice is really downstream of 13e, so it resolves once the light-source decision does.

---

**Summary of what needs an answer before these can be called finalized:**
1. Does "native" mode output 1024px (no interleave) or 2048px, given the sensor's confirmed 2048-total/2-row-of-1024 layout — and separately, is pixel-shift processing included at the price point being evaluated? (13a)
2. Which lens is actually on Milestone 1's bench right now — the loaned 2.5X, or the 3X being priced for production? (13b)
3. Does the bar-vs-coaxial choice foreclose the dual-channel bright/dark-field lighting plan, or are these independent decisions? (13e)

---

## 14. Inspection Capability vs. Throughput — the Lighting/Algorithm Conflict

This is a genuinely important new finding — it's the first thing in this whole document that directly threatens the project's core throughput claim, not just an open engineering detail.

### 14a. Why "fewer angles" matters so much more here than it did in area-scan

The existing area-scan SWIR system runs a fixed 9-angle sequence (Section 5/setup guide: `SWIR_SL1–3`, `SWIR_CL1–3`, `SWIR_RL1–2`, `SWIR_BL`). In area-scan, adding more angles is cheap — the camera is stationary at each FOV position, so extra angles just mean extra stationary snaps before moving to the next position. **In line-scan, that economics inverts:** each additional lighting condition doesn't mean one more snap, it means one more full-length pass down the entire panel (or, at minimum, down every one of the 10 width-strips needed to cover the panel — Section 8). That's the real reason you're exploring whether 1 angle is enough: in this architecture, lighting-condition count and total scan count are directly multiplied together, not added.

### 14b. What you found on crack visibility, and why it's plausible

- **Spotlight makes die cracks clearer than coaxial or ring light**, which is why most existing defect-detection algorithm work leans on spotlight. **Coaxial and ring are more stable, but being diffuse, wash out crack contrast.** This is a well-established machine-vision principle, not a quirk of this system: directional/grazing light creates shadow and specular contrast at surface discontinuities (like cracks), which is exactly what reveals them; diffuse/coaxial light suppresses those same shadows, which is what makes it more *stable* (less sensitive to part angle/orientation) but also what erases the crack contrast. Both properties come from the same physical cause, so this isn't a contradiction to resolve — it's a real trade-off.
- **Worth a cross-check against the setup guide:** the AI Die Defect Inspection algorithm's Few-shot mode has a stated *default* image plane of **Coaxial** ("change image plane if defects aren't clear on the default (Coaxial)"). That's a software default, not a claim that coaxial is best for cracks specifically — and your point above suggests engineers routinely override it toward spotlight in practice. Worth flagging this gap between the documented default and the real tuning behavior back to whoever owns that algorithm, since a new engineer reading only the setup guide would reasonably assume coaxial is the intended plane for die inspection.
- **At least 3 spotlight images (different currents), because it's not predictable beforehand which one reveals the crack** (die internal content, die body warpage, leadframe warpage all affect it unpredictably) — this reads as a bracketing/redundancy strategy: since no single fixed setting reliably works across all parts, using multiple settings increases the odds at least one of them captures the defect. This is sound, standard practice for handling part-to-part variability that isn't fully predictable in advance — not an inefficiency to eliminate, but a real requirement the current algorithm depends on.

### 14c. The throughput math this implies — worth running before anyone commits to a lighting plan

Using the kickoff deck's own numbers (2.9s per pass, 10 width-strip passes needed to cover the panel at single lighting = 29s total, Section 8), extending that to however many lighting conditions actually turn out to be necessary, **assuming each condition requires a full repeat of all 10 passes** (i.e., the "Per-Pass" toggling scheme from Section 4, applied per width-strip):

| Lighting conditions needed | Total passes | Total time | vs. 55.2s area-scan baseline |
|---|---|---|---|
| 1 (current kickoff assumption) | 10 | 29.0s | 1.9× faster |
| 2 (e.g., bright-field + dark-field) | 20 | 58.0s | **5% *slower*** |
| 3 (matching the area-scan spotlight-current approach) | 30 | 87.0s | **58% *slower*** |

**This is the single most consequential number in this document so far.** If reliable die-crack detection genuinely requires 2 or 3 lighting conditions per width-strip, and each condition requires a full repeated pass, the line-scan project doesn't just lose some of its throughput advantage — it could end up **slower than the system it's meant to replace.** This directly contradicts the "2.7×–5.4× faster, even worst-case" claim in Section 8, which appears to assume only the single-lighting case.

**This changes how important "Per-Line" dual-channel actually is.** Section 4 currently frames Per-Line multiplexing (alternating strobes within a single pass, rather than repeating whole passes) as an "advanced, feasibility TBC" nice-to-have. Given the numbers above, **Per-Line isn't a nice-to-have if more than one lighting condition is required — it may be the only way to preserve any meaningful throughput advantage at all**, since it's the only proposed mechanism that gets multiple lighting conditions without multiplying total pass count. Worth reframing this back to Gary: this isn't "confirm if the advanced timing works when convenient," it's "this may be load-bearing for the entire project's business case." Worth noting, too, that if 3 lighting conditions are genuinely needed, even Per-Line looks like a stretch — the timing budget (switch + LED settle + exposure, all inside 31.25µs) is already marked feasibility-TBC for 2 channels; fitting 3 into the same window is a materially harder ask, not a small extension of the same idea.

### 14d. Worth checking, not assuming

- **Is the 8°-offset "bar light" candidate (Section 13e) actually diffuse, or is it closer to spotlight-like?** It's described as focused through a cylindrical rod lens onto the scan line — that's a directional, concentrated beam, which is a different character from the coaxial box's deliberately diffused beam-splitter design. If the bar light turns out to behave more like a directional/spotlight source than a diffuse one, it may partially resolve this whole conflict rather than sitting on the same side of it as coaxial. This is worth an explicit bench check (ties into Milestone 1) rather than assuming "coaxial and bar light are both diffused" as your note groups them — that grouping may be right, but it's worth confirming given how much rides on it.
- **Post-processing is unlikely to substitute for a genuinely different lighting capture.** Since the reason different currents matter isn't simply "brighter vs. dimmer" but seems tied to how a specific defect interacts with a specific illumination condition (die content, warpage), this doesn't look like something a single capture plus digital enhancement could recover after the fact. Worth ruling this out explicitly with whoever owns the algorithm before spending effort on it as a shortcut.
- **Consider whether a two-stage/adaptive inspection could avoid uniformly multiplying every pass.** Area-scan doesn't need this because extra angles are cheap there — but line-scan's different cost structure (Section 14a) means it might be worth a first broad pass at one lighting condition to flag suspect regions, followed by targeted re-imaging of only the flagged areas under additional lighting conditions, rather than blanket-repeating all 10 width-strips for every lighting condition. This is a genuinely new idea for this architecture, not something carried over from area-scan practice, and would need validation that targeted re-imaging is even mechanically practical on this gantry (stop-and-snap vs. continuous scan).

### 14e. Your conclusion, and what I'd add to it

Your conclusion — that moving from area-scan to line-scan requires real effort on the inspection-capability side, and that limiting to a single angle looks impossible — holds up against the reasoning above. What I'd add: this isn't just an inspection-quality question anymore, it's now also a **throughput question**, per Section 14c's numbers. Worth raising both halves together with TJ/Gary rather than treating "does 1 angle work" and "is the project still faster than today" as two separate conversations — per the math above, they're the same conversation.

---

## 15. Master Gantt Chart — Alignment & Resolved Items

**Source:** `AOI Line Scan Delivery - Master Gantt.xlsx`, read directly (not edited) from the Google Drive link shared in this project. **Software Development (Item 4 and its sub-items) is deliberately excluded from this section** — that workstream is tracked in `software-development-context.md` separately; everything below comes from Items 1, 2, 3, 5, 6, and the sheet's Alignment table.

### 15a. Conflicts / Open Items This Resolves

| Prior open item | Resolution from the Gantt |
|---|---|
| Gantry drive type — ball screw vs. linear motor (Section 12c) | **Resolved: ball screw.** Stated twice in the sheet ("Z-Height Ball Screw" station requirement; "+ ball screw" on the 31-Oct SWIR offline test note). |
| Milestone 1 / Milestone 2 due dates — previously "pending" (Section 11) | **Resolved.** M1 ≈ 15-Sep to 30-Oct-2026; M2 ≈ 15-Sep-2026 to 28-Feb-2027 (Software Integration/Tester buyoff sub-task ends 15-Feb-2027). The M1-gates-M2 dependency is now a recorded `Depends On` relationship in the sheet itself, not just a recommendation. |
| Whether DOF and lighting-current/heat testing are actually planned for Milestone 1 (Sections 12a, 4) | **Confirmed as scheduled**, under "Free-run Image quality test": explicit sub-items for "Camera Testing (Min Exposure time, DOF)" and "Light Source Testing (Current cap, long emit durability, image quality against heat)." The specific numbers are still pending (Not Started), but the test plan itself now exists on paper. |
| Whether the SWIR line-scan camera is Primary or Auxiliary in the DSS framework (open across Sections 5/13) | **Clarified: Auxiliary**, in the official integration phase. The Alignment table's "SWIR Scan Official" row explicitly states "SWIR Line Scan Vision mount as Auxiliary Cam." (During the earlier bench-test phase it runs as "primary toolhead (Single Cam system)" — a simplified test-rig configuration, not a contradiction, just a different phase.) |

**Not resolved by the Gantt — still open:** which lens is physically on Milestone 1's bench (loaned 2.5X vs. the 3X pending for production — Section 13b); the "1.25µm plane offset" / "25% light throughput" figures (Section 4); how many lighting conditions are actually required for reliable crack detection (Section 14); the empirical basis for the 0.5µm encoder ceiling (Section 12c).

### 15b. Offline vs. Official Machine Alignment

This is the sheet's dedicated alignment table, reproduced directly since it answers exactly the "which machine, which phase, what's being tested" question:

| Start | End | Phase | Test Station | Station Requirement | Machine Setup | Items to Conduct | Remarks |
|---|---|---|---|---|---|---|---|
| — | 31-Oct-2026 | SWIR Scan **Test Station** (bench) | TBD | Z-Height, Ball Screw | SWIR Line Scan Vision installed as **primary toolhead** (single-cam system) | Image Quality Test (Free-Run); Vision Parts Benchmark and Finalization | Start date subject to parts-arrival ETA |
| 15-Nov-2026 | 15-Feb-2027 | SWIR Scan **Official** | **SWIR RND #1** | Z-Height | SWIR Line Scan Vision mounted as **Auxiliary Cam** | Encoder Triggering Integration Test; [SW] Integration Test; Rigidity Test | **SWIR Area Scan RND machine unavailable until the new SWIR RND #2 build is complete** |

**Two things worth flagging directly from this table:**

1. **The bench-test station is still "TBD."** Milestone 1's own test bench doesn't have a confirmed physical station yet, and its start is already gated on parts-arrival ETA — which Section 15c below shows is itself uncertain. Worth surfacing as a scheduling risk in its own right, not just a formality.
2. **The SWIR Area Scan machine goes dark during the official integration window.** Per the Remarks column, the existing area-scan SWIR RND machine won't be available from whenever "SWIR RND #2" construction starts until that new build completes — this is a real operational gap (Section 15c has the detail on why: the existing machine is effectively being rebuilt into SWIR RND #2). If anyone — support, other testing, a different team — needs area-scan SWIR capability during this window, it won't be there. Worth flagging this proactively rather than letting someone discover it by trying to use a machine that's mid-rebuild.

*(The Laser Line Scan project's rows from this same table — including the detail that the Laser camera gets temporarily mounted onto SWIR RND #1 as a third toolhead for its own bench testing — are tracked in `op2-semicon-laser-linescan-context.md`, since that's a different project's milestone, but it's a direct example of the two projects sharing physical hardware, not just a software/VTHAL dependency.)*

### 15c. New Findings Not Previously Captured

- **SWIR RND #2 is a DSS-based build, not OP2** — and its construction is what makes the existing SWIR Area Scan machine unavailable (15b). Note in the sheet: *"DSS Base, not OP2, with GSync Pro, pending to work with BD Wan Ling, convert PO to line scan??"* Two things worth surfacing from that one line: it confirms SWIR RND #2 sits on the DSS platform (consistent with Section 13's DSS cross-references), and it raises a **commercial question, not just an engineering one** — converting an existing Purchase Order to a line-scan configuration sounds like it could mean repurposing an already-committed customer or sales unit for R&D use. Worth confirming directly with TJ/BD Wan Ling what "convert PO" actually means here before assuming it's a routine internal parts reallocation.
- **A parallel, non-encoder-based triggering path is also being evaluated** ("Non Encoder Trigger Board Route," alongside the "Encoder Trigger Board Route" already documented in Section 6). The sheet doesn't detail the mechanism, so I'm not certain what "non-encoder" triggering means here mechanically — possibly a simpler timer- or software-triggered scheme evaluated as a fallback or comparison point. Worth a direct question to Gary on what this alternative actually is and why it's being run in parallel rather than sequentially.
- **A custom PCB is being designed for encoder signal capture**, referencing a "Delta Motion card" as a design reference (Delta is the same motion-control vendor named in the setup guide's registry notes on Delta driver init — consistent, not a new vendor). This means the trigger-chain hardware isn't fully off-the-shelf; there's custom board design and a JLCPCB fabrication order in the critical path, with a noted scheduling constraint (a company trip to Japan requires the PCBA purchase to be submitted before departure).
- **G-Sync Pro's vendor is Affinex** — not previously named in this document; the deck referred to it only by product name.
- **Parts-sourcing vendor names for SWIR Milestone 1:** Vital Vision, Vistech, Toutek Supp, KaiWei. As of the latest note, Vital Vision's parts ETD is still unknown and Vistech's is "late of Oct" — both bear directly on the "TBD start date, subject to parts ETA" caveat in 15b.

### 15d. Suggested Next Step

Given how much of this ties back to open items already logged in Sections 11–14, worth using this Gantt update as the occasion to close the loop on the two items it *doesn't* resolve but makes more urgent: which lens is actually on Milestone 1's bench, and whether the parts-ETD uncertainty (Vital Vision, Vistech) is going to push the 30-Oct Milestone 1 end date before anyone formally revises it.
