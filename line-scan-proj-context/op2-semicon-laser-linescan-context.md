# OP2 Semicon — Laser Line Scan Auxiliary Camera — Project Context

**Status:** Early-stage. **Product owner: YuenKhang** — not your project, but the software development work overlaps heavily with yours (both need line-scan support added to VTHAL, see Section 5), so this is tracked here as a **watchlist item**, not a primary deliverable. Most specs below are working understanding from a single conversation, several now clarified below and flagged where still open.

**Related but tracked separately from:** the SWIR Line Scan / Skyworks DSM project (`swir-line-scan-project-context.md`). Same broad category of work — adding a line-scan auxiliary camera to an existing AOI platform — but a different customer, a different sensing technology (laser triangulation vs. SWIR spectral imaging), a different product owner, and a different timeline. Kept as its own document so the two don't get tangled, with cross-project notes in Section 5 where they genuinely connect — and Section 5 now carries the most important connection: **both projects are blocked on the same unresolved VTHAL capability gap.**

---

## 1. Project Summary

- **Platform:** OP2 Semicon — an **existing ViTrox product line**, but per your clarification, **severely untested on actual customer production sites so far.** That's an important compounding factor for the timeline risk in Section 4: it's not just the Laser module that's unproven at the Feb on-site upgrade — the base platform itself carries its own unproven-in-production risk, independent of the auxiliary camera work.
- **Ownership:** Laser Line Scan product owner is **YuenKhang**. Software development for it overlaps significantly with your SWIR project (Section 5), which is why you're watchlisting it rather than treating it as unrelated.
- **Phased delivery:**
  - **Mid-November:** ship the OP2 Semicon platform to the customer site as a single-camera machine (i.e., without the Laser Line Scan module).
  - **15-February-2027** (refined from "mid-February" — Section 6): on-site upgrade at the customer's site, adding the Laser Line Scan module as an **auxiliary camera**. Confirmed in the Master Gantt as "On-site Upgrade to DSS Line Scan" — also confirms this integrates onto a **DSS-based** platform, same underlying architecture family as the SWIR project's target machine.
- Framed as "another similar line scan project" to the SWIR/Skyworks work — same category of effort, different specifics.

---

## 2. Laser Line Scan Module — Hardware Overview

| Aspect | Detail |
|---|---|
| Integration style | **All-in-one unit** — lighting controller and frame grabber combined into a single module |
| Data transfer | **Ethernet cable, TCP/IP protocol** |
| Lighting per snap | Two conditions: **visible light** and **405nm UV (violet) light** |
| Native sensor | **3200 elements** |
| Resolution | **12µm / 5µm** |
| Onboard buffer | Up to **32,000 lines** of image pixels |
| Visible-light output | Greyscale image, **"short" (16-bit)** format |
| UV (405nm) output | **3D height map**, in **16-bit or 32-bit** format |

**Reading the specs, with flags where I'm inferring rather than confirming:**

- **Integration style vs. the SWIR project:** this all-in-one design (lighting controller + frame grabber in one unit) is architecturally simpler than the SWIR project's discrete G-Sync Pro + Grablink Duo + separate light source/controller — fewer boxes, fewer interconnects to validate. Whether that simplicity holds up depends on what's inside that single unit, which isn't detailed yet.
- **Ethernet/TCP-IP vs. Camera Link:** a materially different data path from the SWIR project's Camera Link/PCIe DMA. Different bandwidth ceiling, different latency characteristics, different cabling. Worth its own throughput sanity check once frame size and target line rate are known — the same kind of check that confirmed the SWIR project's Grablink Duo has comfortable headroom (Section 13c of the SWIR doc), but this is a different transport entirely so that check doesn't carry over.
- **405nm + height map = laser triangulation, not spectral imaging.** The pairing of a UV/violet laser line with a height-map output is the signature of laser-line triangulation profilometry (a projected stripe, height inferred from how much it laterally shifts as surface height varies) — a fundamentally different measurement principle from the SWIR project's material-contrast imaging. Worth keeping in mind that these two projects, despite both being "line scan," are solving different problems (3D shape/height vs. 2D material contrast) — the mechanical/optical risk categories from the SWIR project's Section 12 (DOF, encoder-pitch matching) don't automatically transfer here without re-deriving them for a triangulation geometry, which has its own separate DOF/standoff/baseline-angle considerations.
- **"12µm / 5µm" — correction applied:** these are **two candidate resolutions, still undecided which to go for** — not a chip-pitch-vs-object-space-pitch pair the way the SWIR camera's spec worked out. Good that I asked rather than locked in the earlier reading, since it was wrong. One thing still worth a light confirm when the decision firms up (not urgent): does the "3200 native elements" figure apply to both resolution candidates, or does it belong to just one of them? A finer pitch covering the same field width would generally need more elements, so it's plausible these aren't simply two settings on one fixed sensor.

- **New: buffer capacity is a real, concrete input to the 12µm-vs-5µm decision, not just a quality trade-off.** Using your panel-size assumption ("safely presumed similar across semicon production setup") and the Skyworks DSM panel (~179mm) as the closest reference point I have for what "similar" looks like in this program:
  - At **12µm/line**: 179,000µm ÷ 12µm ≈ **14,917 lines** for a full panel-length pass — about 47% of the 32,000-line onboard buffer. Comfortable single-pass margin.
  - At **5µm/line**: 179,000µm ÷ 5µm = **35,800 lines** — this **exceeds the 32,000-line buffer by ~12%**. A full panel-length scan wouldn't fit in one continuous onboard-buffered pass at this resolution.
  - If 5µm is the resolution you actually want, that's not necessarily disqualifying, but it does mean extra engineering the 12µm option avoids: either splitting the scan into multiple shorter passes that each fit under 32,000 lines, or streaming continuously off the onboard buffer via the Ethernet/TCP-IP link rather than buffer-then-transfer (which shifts the bottleneck to sustained network throughput instead of buffer depth — a different check, not yet done). Worth surfacing this trade-off explicitly in the resolution decision rather than treating it as purely an image-quality call.
  - Caveat: this whole calculation rides on "panel size similar to Skyworks DSM's ~179mm" being a reasonable stand-in — if this project's actual target panel is meaningfully longer or shorter, both numbers move proportionally.
- **16-bit vs. 32-bit height-map format** sounds like a precision/range trade-off (coarser but smaller vs. finer or floating-point but larger) rather than two different products — worth confirming which is the actual target for this project, since it affects buffer sizing and downstream processing load.

---

## 3. Open Questions

**Resolved this round:**
- ~~Is "OP2 Semicon" an existing product line?~~ Yes — existing product line, but untested at customer production sites (Section 1).
- ~~Does 5µm refer to object-space resolution through the lens, or two separate sensor modes?~~ Neither reading was right — 12µm and 5µm are two undecided resolution candidates (Section 2).
- ~~Is there an in-house bench-validation step before the Feb on-site upgrade?~~ Yes — an offline in-house station is being set up for integration testing, running in parallel with machine-shipping prep (Section 4).
- ~~What board/panel size is this designed against?~~ Presumed similar to other semicon production setups — used as the basis for the buffer-capacity check in Section 2.
- ~~Who owns this, any overlap with the SWIR team?~~ YuenKhang owns it; software-side overlap with your SWIR work is real (Section 5), which is why it's on your watchlist.

**Still open, and now the most important one in this whole document:**
- **VTHAL has never supported communication with any line-scan product before** (Section 5) — this affects both projects, not just this one. Liaison with the VTHAL team is underway to scope feasibility and effort. Given this gates both projects' software integration, effort estimation and timing here deserve to be treated as the critical path, not a line item.
- You mentioned the VTHAL team needs to **"occupy the sole camera"** for their study. Is that the same physical unit and the same activity as the in-house offline integration station (Section 4), or two separate efforts competing for one piece of hardware? If they're separate, that's a scheduling conflict worth resolving explicitly. If they're the same effort, worth confirming the sequencing: does VTHAL's feasibility study need to conclude before the "integration test" can meaningfully proceed (since a real integration test presumably needs at least a working VTHAL communication path to test against)? 🔁 **Partial update (Section 6):** the Master Gantt confirms real physical-hardware sharing is happening — the Laser Scan Camera is scheduled to be temporarily mounted onto **SWIR RND #1** (the SWIR project's own bench machine) as a third toolhead for its Oct–Nov bench testing. That doesn't resolve whether the "sole camera" comment refers to this same arrangement, but it does confirm cross-project hardware sharing is real and already scheduled, not hypothetical — worth checking directly whether this creates a scheduling conflict with whatever the SWIR team needs SWIR RND #1 for during that same window.
- Which height-map format — 16-bit or 32-bit — is the actual target, and why the choice matters (precision needs vs. buffer/bandwidth budget)?
- Does this module integrate through the same VTHAL Primary/Auxiliary architecture the DSS setup guide describes, or does the Ethernet/TCP-IP interface mean a separate acquisition pathway outside that framework? (Partially answered by Section 5's clarification that VTHAL is the universal middleman regardless of camera type — but the DSS-specific Primary/Auxiliary registry mechanism is still a separate question from "does it go through VTHAL at all.")

---

## 4. Timeline Risk Worth Naming

Two risk layers now, not one:
1. **The base OP2 Semicon platform itself** is going to a live customer production site in November despite being, in your words, "severely untested" there — a risk that exists independently of the Laser module.
2. **The Laser Line Scan module** is scheduled for on-site integration in February. The good news: there is an in-house offline station planned for integration testing beforehand, running in parallel with shipping prep — this is exactly the kind of bench-gate the SWIR project's Milestone 1 represents, so the "no lab to retreat to" risk I flagged earlier is genuinely being addressed, not ignored.

**What still needs answering to know if that gate is solid:** the offline station's usefulness depends on timing relative to the VTHAL liaison (Section 3/5) — if VTHAL's line-scan support isn't ready in time, the offline station may only be able to validate hardware/Ethernet-link-level integration, not a true end-to-end software pipeline test. Worth confirming the offline station's testing scope explicitly matches what's actually achievable given where VTHAL's support stands at the time.

---

## 5. Cross-Project Notes (vs. the SWIR/Skyworks Project)

**VTHAL, clarified:** VTHAL is a middleware library sitting between your AOI software and the camera + frame grabber — your software always triggers VTHAL to snap (which includes firing the lighting) and retrieves the image buffer from wherever VTHAL places it. Neither your software nor this project's talks to hardware directly; VTHAL is the universal go-between for every camera type.

**The single most important cross-project fact from this conversation: VTHAL has never supported communication with any line-scan product before — not this Laser module, and not the SWIR line-scan camera either.** That reframes something in the SWIR document worth flagging back there: Milestone 2's completion criterion #5 ("a defined path for how line-scan frames land in VTHAL's existing view-compilation model") assumed the gap was in the *view-compilation layer* specifically. Per this clarification, the gap may be more foundational — VTHAL's core camera-communication layer itself has no line-scan concept yet, for either project. I've flagged this back in the SWIR document (see its Section 6/11b).

**What this means practically:**
- Both projects are liaising with the VTHAL team on the same underlying capability gap, at the same time. Whichever project's camera gets used for VTHAL's feasibility study likely sets the technical pattern the other project inherits — worth knowing whether that's being coordinated deliberately or happening by whichever project reaches the VTHAL team first.
- Effort estimation and timing from the VTHAL team is the critical path for the software side of both projects, not just this one — worth tracking explicitly rather than letting it sit as an assumption that "VTHAL integration" is a routine step.
- The DSS Dual-Camera Setup Guide's Auxiliary Camera generalization (Mono/Color/SWIR) is a separate question from "does VTHAL support line-scan at all" — even if this Laser module were to slot into that Primary/Auxiliary registry pattern, it still needs VTHAL's core communication layer to understand line-scan acquisition first. The DSS framework question (does it need a 4th auxiliary type) is downstream of the more basic VTHAL gap, not a substitute for it.

**Other connections:**
- The all-in-one lighting-controller-plus-frame-grabber design is architecturally simpler than the SWIR project's discrete G-Sync Pro + Grablink Duo + separate light controller — fewer boxes, fewer interconnects. Whether that simplicity survives VTHAL integration depends on what VTHAL's line-scan support ends up looking like for each project's hardware shape.
- Both projects independently reached the same design lesson from opposite directions: the SWIR project discovered its `.jscale` calibration file already has an `aspect_ratio` field seemingly built for exactly the kind of non-square-pixel correction its lens mismatch now needs (Section 12b/13b of that doc). If this Laser module's eventual resolution choice (Section 2) turns out to have a similar whole-number-divisibility question against whatever triggers its line acquisition, that same existing mechanism is worth checking here too, rather than re-deriving a correction approach from scratch.

---

## 6. Master Gantt Chart — New Findings

**Source:** `AOI Line Scan Delivery - Master Gantt.xlsx`, read directly (not edited). Software Development items are excluded here, consistent with the SWIR document.

- **Candidate model numbers, not previously known:** parts-sourcing notes for the Laser Offline Station Benchmark reference **8020, 8060K, 8061** as candidate models still being decided ("decide model, consider convert to sales for SW testing"). Worth confirming with YuenKhang whether these map onto the 12µm/5µm resolution decision (Section 2) — i.e., are these different resolution variants, or different vendors for the same spec?
- **The trial part has a time limit, similar to the SWIR project's loaned lens:** noted as "part available for 1 month" — with a suggestion to convert it to an outright purchase specifically to enable ongoing SW testing without racing the loan window. Worth flagging as a parallel to the SWIR project's own "loaned telecentric lens" time pressure (Section 12b/13b of that doc) — both projects are currently working against a borrowed-hardware clock, independently.
- **Alignment table rows for this project** (full table, including the SWIR side, is in `swir-line-scan-project-context.md` Section 15b):

| Start | End | Phase | Test Station | Station Requirement | Machine Setup | Items to Conduct | Remarks |
|---|---|---|---|---|---|---|---|
| 9-Oct-2026 | 15-Nov-2026 | Laser Scan **Test Station** (bench) | **SWIR RND #1** | Z-Height | Laser Scan Camera temporarily attached as a **3rd toolhead**, simple mount | Image Quality Test; [SW] Camera Comm Test with vendor SDK | TBD |
| 15-Nov-2026 | 15-Feb-2027 | Laser Scan **Official** | **OP2 RND (new build)** | Z-Height | Laser Scan Camera mounted as **Auxiliary Cam**; original SWIR Area Scan camera moved to a new DSS build | Encoder Triggering Integration Test; [SW] Integration Test; Rigidity Test | — |

**The most notable thing in this table: the Laser camera's own bench testing runs on the SWIR project's machine (SWIR RND #1), not a dedicated Laser test rig.** That's the concrete version of the "sole camera" resource-sharing question flagged in Section 3 — worth checking directly with Gary/Weng Yeong whether this Oct–Nov window conflicts with anything the SWIR project itself needs that same machine for.

**Also notable:** the existing SWIR Area Scan camera is described as being relocated to "a new DSS build" once the Laser module goes official — this is the same DSS rebuild referenced in the SWIR document's Section 15c ("SWIR RND #2"), so both projects' hardware transitions are tangled together at the machine level, not just at the software/VTHAL level.
