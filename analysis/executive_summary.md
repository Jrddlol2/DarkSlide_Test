# DarkSlide — Executive Summary

> A principal-engineer technical audit of DarkSlide compared against Negative Lab Pro (NLP). Analysis only — no production code was modified. Companion documents: `architecture.md`, `pipeline.md`, `algorithms.md`, `feature_inventory.md`, `performance.md`, `nlp_comparison.md`, `technical_debt.md`, `roadmap.md`.

## What DarkSlide is
A free, open-source film-negative-to-positive converter that runs both **in the browser** and as a **Tauri desktop app** (React 19 + TypeScript frontend, Rust backend). All processing is local. The desktop build adds RAW decoding (`rawler`), a live scanning-session folder watcher, native OS integration, and an auto-updater.

## Overall assessment
DarkSlide is **substantially more sophisticated than a typical hobby project**. The domain layer — density-domain inversion, automatic orange-mask/film-base estimation with confidence scoring, numeric ICC parsing, deterministic pinned parameters with a diagnostic report — is thoughtfully engineered and genuinely well tested. Its main limitations are **precision (8-bit internal), RAW source fidelity (inverts from display-encoded, camera-calibrated data), and a heuristic color model**, plus concentration of complexity in three very large files.

Against NLP, DarkSlide **wins on access** (free, offline, no Adobe, browser + desktop) and **transparency** (inspectable, deterministic conversions), but **trails on color fidelity, RAW handling, 16-bit output, local edits, and catalog/workflow integration**.

## Image-quality impact ranking (Phase 7)
The factors that most influence conversion quality, ranked by their leverage on the final image in *this* codebase:

1. **Negative inversion precision (8-bit vs float)** — biggest ceiling on quality; causes banding and blocks 16-bit.
2. **RAW source fidelity (linear scene-referred vs sRGB-encoded)** — determines highlight/color headroom.
3. **Per-stock color rendering / channel balancing** — determines whether the "look" is authentic.
4. **Orange-mask / density normalization** — already strong; correct base = correct everything downstream.
5. **White-balance / neutralization** — currently border-biased gray-world; matters on mixed-light scans.
6. **Highlight handling & tone reproduction** — good heuristics, but display-domain math limits accuracy.
7. **Shadow color balance / residual base** — handled well via residual-offset + crush guard.
8. **Contrast mapping & film-profile tone** — competent and tunable.

## The five highest-impact improvements toward NLP parity

1. **Adopt a float / ≥16-bit internal render buffer and remove the mid-pipeline 8-bit quantization** (`imagePipeline.ts:1371-1377`, `1572-1578`). This single change eliminates banding, unlocks true 16-bit PNG/TIFF export (currently always degraded — `exportEncoder.ts:422`), and is the foundation for every other quality gain.

2. **Invert from linear scene-referred RAW.** Drop `SRgb`/`Calibrate` from `decode_raw` (`src-tauri/src/lib.rs:104-111`) and render the negative from the high-depth linear buffer instead of the 8-bit sRGB preview (`rawImport.ts:60-97`). This closes the largest color/highlight-fidelity gap with NLP.

3. **Replace the hand-tuned color model with measured per-stock renderings.** Move from single-γ + linear `densityScale` + hand-tuned 3×3 matrices (`constants.ts`) to per-stock density **characteristic curves** and fitted color renderings. This is what makes NLP's "lab look" authentic.

4. **Unify the three pipeline implementations behind one core with golden-image parity tests.** CPU-8bit, CPU-float, and WGSL are kept in manual parity today (`imagePipeline.ts` + `tiledRender.wgsl`); a single source of truth de-risks improvements #1–#3 and cuts triple maintenance.

5. **Add local/masked adjustments and scene-referred auto white balance.** Radial/linear/brush masks plus post-inversion neutralization bring DarkSlide's editing flexibility closer to the Lightroom-backed NLP workflow.

## Bottom line
DarkSlide already does the *hard, correct* thing — a physically-grounded, inspectable inversion with strong automatic base estimation. The path to NLP-class results is not a rewrite: it is **higher numerical precision, a linear-RAW front end, and measured color science**, delivered on top of a de-duplicated pipeline core. Executed in the phased order in `roadmap.md`, these would make DarkSlide the strongest open-source film-conversion tool available.
