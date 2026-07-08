# DarkSlide vs Negative Lab Pro (NLP)

> NLP comparisons are based on publicly documented behavior and its well-known workflow (a Lightroom Classic plugin built around scene-referred RAW conversion, Frontier/Noritsu/"Lab" color renditions, per-roll analysis, and Camera Raw as the editing surface). DarkSlide claims are grounded in this repository.

## Workflow model — the biggest structural difference
- **NLP** lives **inside Lightroom Classic**. You RAW-convert, NLP reads the linear demosaiced data, writes a converted rendering, and you keep editing with Lightroom's full toolset (masks, spot heal, sync, catalog, metadata). It is a *workflow plugin*, not a standalone app.
- **DarkSlide** is a **standalone app** (browser + Tauri) with its own importer, editor, rolls, and exporter. Self-contained and free, but not integrated with a DAM/catalog and without Lightroom's local-adjustment tooling.

**Implication:** professionals already in a Lightroom catalog get less friction from NLP; hobbyists and non-Adobe users get a complete, free, offline tool from DarkSlide.

## Color accuracy & color science
- **NLP:** scene-referred conversion from **linear RAW**, multiple emulated lab "profiles" (Frontier/Noritsu/Lab), sophisticated per-emulsion color renderings and neutralization; the industry reference for out-of-box color.
- **DarkSlide:** density-domain inversion is physically sound, but color is shaped by **hand-tuned 3×3 matrices + heuristic tonal-character triples + a per-channel density scale** (`constants.ts`), and RAW is inverted from **sRGB-encoded, camera-calibrated** data (`lib.rs:104-111`), not linear scene-referred RAW.
- **Verdict:** NLP is ahead on out-of-box color fidelity and "lab look" authenticity. DarkSlide's framing is correct but its color model is coarser.

## Film-inversion quality
- Both invert in a density-aware way. DarkSlide's base-anchored black point, symmetric flare subtraction, crush guard, and pinned-deterministic parameters are a genuinely strong, inspectable design (`imagePipeline.ts`).
- NLP's advantage is the downstream **per-stock color rendering** and scene-referred data, not the inversion arithmetic per se.

## Orange-mask handling
- **DarkSlide:** automatic region/confidence film-base estimator with rebate detection and fallbacks (`estimateFilmBaseCore`), plus manual picker and roll-level base. This is a real strength and arguably more transparent than NLP's black-box base handling.
- **NLP:** robust, largely automatic; users rarely touch it.
- **Verdict:** roughly comparable; DarkSlide is more inspectable, NLP is more "just works" on edge cases.

## White balance / neutralization
- **NLP:** in-Lightroom WB picker on linear data + its own neutralization; strong.
- **DarkSlide:** neutral-border gray-world auto WB (`analyzeColorBalance`) + additive temp/tint. Works but border-biased and not chromatic-adaptation-correct.
- **Verdict:** NLP ahead.

## Scene balancing / tone
- **NLP:** per-image and roll-aware tone rendering with lab-accurate contrast.
- **DarkSlide:** percentile exposure, IQR-based midtone contrast, adaptive highlight shoulder, per-stock tonal character — competent and tunable but heuristic; tone math runs on display-encoded values, not scene-linear.

## Batch processing
- **NLP:** batch-convert selections, sync settings across a roll via Lightroom.
- **DarkSlide:** batch export a roll with optional preset (`batchProcessor.ts`), contact sheets, rolls, and a live scanning-session watcher (desktop). DarkSlide's **scanning-session folder watch** is a feature NLP does not natively provide.

## Speed
- **NLP:** bounded by Lightroom/ACR; conversion is fast, editing is Lightroom-fast.
- **DarkSlide:** WebGPU render + tiling is responsive, but export is single-threaded JS and RAW is down-converted to 8-bit before render. Comparable interactively; DarkSlide loses on high-res export throughput.

## Non-destructive editing & presets
- **NLP:** fully non-destructive via Lightroom history/catalog + develop settings.
- **DarkSlide:** non-destructive crop, 50-step undo, sidecar settings, exportable `.darkslide` presets with folders/tags. Solid, though history is app-session-scoped, not a catalog.

## Export
- **NLP:** exports through Lightroom (TIFF/JPEG, 16-bit, full color management).
- **DarkSlide:** PNG/TIFF/JPEG/WebP with correct ICC embedding, **but 16-bit currently degrades to 8-bit** (`exportEncoder.ts:422`). **Verdict:** NLP ahead until DarkSlide ships a true high-depth path.

## Professional usability
- **NLP:** catalog, metadata, masks, sync, tethering via Lightroom — professional-grade end-to-end.
- **DarkSlide:** clean focused UI, rolls, contact sheets, keyboard shortcuts, diagnostics — very good for a standalone, but no masks/local edits and no DAM.

---

## Summary tables

### Strengths of DarkSlide
- Free, open-source, offline, browser **and** desktop; no Adobe dependency.
- Physically-grounded, deterministic, **inspectable** inversion (diagnostic parameter report).
- Strong automatic film-base/orange-mask estimation with confidence + crush guard.
- Numeric ICC input-profile parsing (honors linear scan profiles).
- Scanning-session folder watch, rolls, contact sheets, 40+ film presets.

### Strengths of NLP
- Scene-referred conversion from linear RAW; authentic lab color renderings.
- Deep Lightroom integration (catalog, masks, sync, metadata, 16-bit export).
- Best-in-class out-of-box color and neutralization on difficult stocks.
- Mature, widely validated across scanners/cameras.

### Weaknesses of DarkSlide
- Inverts from sRGB-encoded, camera-calibrated RAW, not linear scene-referred.
- 8-bit internal precision; 16-bit export degrades to 8-bit.
- Heuristic color model (hand-tuned matrices) vs measured dye/lab models.
- No local/masked adjustments, no lens correction, no catalog/DAM.
- Display-domain tone math (not scene-linear).

### Weaknesses of NLP (publicly known)
- Paid; requires Lightroom Classic (Adobe subscription/ecosystem).
- Closed-source; conversion internals are a black box.
- Tied to Adobe's release/compatibility cadence.

### Features unique to each
- **DarkSlide only:** browser build, scanning-session watcher, exportable `.darkslide` presets, open diagnostics, no-cost/offline.
- **NLP only:** Lightroom catalog + masks + sync, authentic Frontier/Noritsu/Lab renderings, scene-referred RAW pipeline.

### Missing from DarkSlide (highest leverage)
1. Scene-referred **linear RAW** inversion.
2. True **≥16-bit float** render + export path.
3. Measured **per-stock color/lab renderings** (not hand-tuned matrices).
4. **Local/masked** adjustments.
5. Catalog/metadata workflow or a Lightroom bridge.

### Already done well by DarkSlide
- Density-domain inversion mechanics and base anchoring.
- Automatic orange-mask/film-base estimation.
- Color-management correctness (ICC parse/embed, profile transforms).
- Determinism/inspectability (pinned params + diagnostic report).
- Batch/roll/contact-sheet/scanning-session ergonomics.
