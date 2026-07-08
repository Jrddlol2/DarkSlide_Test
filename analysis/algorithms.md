# DarkSlide — Algorithm Review

> Each significant imaging algorithm, with purpose, I/O, mathematics, assumptions, limitations, and improvements.

## 1. Density-domain negative inversion
- **Where:** `imagePipeline.ts:512` (`applyDensityInversion`), staged in `applyInversionStage:698`.
- **Purpose:** invert a scanned negative into a positive in the physically meaningful film-density domain.
- **Inputs:** encoded channel value; working profile; `baseDensity`, `densityScale`, `gamma`.
- **Output:** positive channel 0..1.
- **Math:** `T = decode(v)`, `D = max(0, −log₁₀T − D_base)·scale`, `out = 1 − 10^(−D/γ)`, `γ = 2.2`.
- **Assumptions:** transmittance ∝ decoded channel; base sample represents Dmin; one display gamma fits all stocks; per-layer contrast differences are a single linear scale.
- **Limitations:** no per-channel characteristic-curve (H&D) shape; single global γ; operates on 8-bit-quantized data downstream.
- **Improvements:** per-channel density LUTs derived from real stock characteristic curves; keep the density buffer in float end-to-end.

## 2. Clear-film-base estimation (region + confidence)
- **Where:** `rawImport.ts:202` (`estimateFilmBaseCore`), gates in `constants.ts:431` (`FILM_BASE_CONFIDENCE`).
- **Purpose:** find the orange mask / Dmin reference automatically.
- **Inputs:** RGB(A) pixels, dimensions, stride, channel scale (8- or 16-bit).
- **Outputs:** `FilmBaseEstimate { sample, source, confidence, rejectedCandidates, clamped }`.
- **Math/logic:** grid cells over the 3–12% border/rebate band → per-cell mean + max per-channel std-dev; candidate = bright (`lum ≥ 96`) ∧ low-texture (`std ≤ 14`) ∧ not blown; 4-connected clustering; `confidence = 0.3·size + 0.4·uniformity + 0.3·brightness`; final sample = 64-bin mode-cluster mean of the winning cluster.
- **Assumptions:** clear base is present at the frame edge/rebate, brighter and flatter than image content.
- **Limitations:** borderless/tightly-cropped scans have no rebate → conservative fallback; single global base ignores lateral base-density variation.
- **Improvements:** optional user rebate ROI; roll-level base averaging (already partly supported via `source:'roll'`); 2D base gradient model.

## 3. Density balance (per-channel dye contrast)
- **Where:** `imagePipeline.ts:611` (`computeDensityBalance`); presets `constants.ts:374`.
- **Purpose:** equalize C-41 layer gamma mismatch (blue layer is lower contrast).
- **Math:** collect midtone densities `−log₁₀(v/base)` per channel; take 20–80% trimmed means; `scaleR = meanG/meanR`, `scaleB = meanG/meanB`, clamp [0.4, 2.0]; **reject** (hold neutral) on clamp hits or marginal sample counts (`source:'clamp-rejected'`).
- **Limitations:** heuristic; needs enough neutral midtones; presets are hand-tuned constants (`scaleB ≈ 0.53–0.74`).
- **Improvements:** fit to reference patches; per-stock measured curves instead of a single blue scalar.

## 4. Flare / veiling-glare floor
- **Where:** `flareEstimation.ts:1` (`estimateFlare`).
- **Purpose:** per-channel black-flare floor subtracted symmetrically from base and image.
- **Math:** 0.5th-percentile of each channel histogram → floor; applied as `v − floor·strength` (`applyFlareCorrection`).
- **Limitations:** global scalar per channel; assumes uniform flare.
- **Improvements:** spatially-varying flare / lens-model deglare.

## 5. Color-space conversion + ICC parsing
- **Where:** `colorProfiles.ts` (`convertRgbBetweenProfiles:835`, `parseInputIccProfile:684`, `chromaticAdaptationMatrix:99`).
- **Purpose:** decode input profile to a working profile; identify/honor embedded ICC.
- **Math:** TRC decode → linear → 3×3 (linear→XYZ D65 ∘ XYZ→linear) → TRC encode; Bradford CAT for D50↔D65; TRC classification by sampling to sRGB-or-power-law within tolerance (`classifyTrcEvaluator:590`).
- **Limitations:** matrix/TRC profiles only (no cLUT/A2B); one shared TRC across channels.
- **Improvements:** support cLUT profiles via sampled 3D LUT; Little-CMS-style engine.

## 6. sRGB ↔ Lab ↔ XYZ + ΔE
- **Where:** `colorScience.ts`.
- **Purpose:** perceptual distance for profile identification / neutral matching.
- **Math:** standard sRGB↔XYZ↔CIELAB (D65, exact κ/ε), ΔE = ΔE76 (Euclidean).
- **Limitations:** ΔE76 only (no ΔE2000).
- **Improvements:** ΔE2000 for perceptual accuracy where it matters.

## 7. Auto exposure / white balance / contrast
- **Where:** `autoAnalysis.ts`.
- **Exposure:** percentile-based midpoint shift + black/white points.
- **WB:** chroma-weighted neutral-border gray-world with midtone weighting, two-pass relaxation, warm nudge for color negatives.
- **Contrast:** IQR/range compression → boost; midtone lift curve point.
- **Mono detection:** chroma + normalized cross-channel residual statistics.
- **Limitations:** border-biased WB assumes neutral surroundings; gray-world fails on strongly colored scenes.
- **Improvements:** scene-referred WB after inversion; illuminant estimation from specular highlights.

## 8. Adaptive highlight recovery & tonal character
- **Where:** `imagePipeline.ts:244` (`applyAdaptiveHighlightRecovery`), `constants.ts` `TONAL_CHARACTERS`.
- **Purpose:** filmic highlight shoulder + shadow lift/midtone anchor per stock.
- **Math:** shoulder softness `1 − protection·shoulder^rolloff`; rolloff scaled by highlight-density estimate; shadow lift via gamma below 0.5.
- **Limitations:** heuristic constants per stock; not a fitted film response.
- **Improvements:** replace triples with measured per-stock tone curves.

## 9. Dust / hair / scratch detection & inpainting
- **Where:** `dustDetection.ts`, `dustRemoval.ts`, `dustGeometry.ts` (README v0.9.0).
- **Purpose:** auto-mark and repair defects; manual paint-to-repair.
- **Approach:** orientation filter + Hessian line-likeness for scratches; per-defect width measured along its path; **structure+texture** repair copying grain from a parallel donor strip; texture/noise-floor vetoes to cut false positives on grain.
- **Limitations:** experimental; quality varies with scan/film (README); no IR/ICE channel use.
- **Improvements:** learned defect segmentation; use scanner IR channel when present.

## 10. Frame / crop detection
- **Where:** `frameDetection.ts`.
- **Purpose:** find the image frame within the holder for auto-crop.
- **Approach:** edge/border detection returning a `FrameDetectionResult` with angle/confidence.
- **Improvements:** perforation-aware detection; multi-frame strip segmentation.

## 11. Separable Gaussian (shared primitive)
- **Where:** `imagePipeline.ts:1112` (`gaussianBlur1D`), `separableGaussianBlur:1165`.
- **Math:** 1D kernel `σ = 0.65r + 0.35`, two passes; scratch buffers reused.
- **Limitations:** clamped edges; 8-bit intermediate in the H-pass round-trip.
- **Improvements:** float intermediate; GPU offload (already available via WGSL).

## 12. Export encoders (PNG/TIFF)
- **Where:** `exportEncoder.ts`, `binaryEncoding.ts`.
- **Purpose:** self-contained encoders with correct ICC embedding.
- **Notes:** manual PNG uses stored (uncompressed) IDAT for 16-bit; TIFF is a minimal little-endian IFD writer. Correct but not size-optimal for 16-bit.
- **Improvements:** real deflate for 16-bit PNG; predictor/LZW for TIFF.

## Third-party algorithm usage
- **`rawler`** (Rust) performs demosaic/calibrate/develop; DarkSlide selects the step list (`lib.rs:104-111`) and takes 16-bit sRGB output.
- **UTIF** decodes TIFF; **piexifjs** reads/writes EXIF. Everything else (inversion, color, film base, ICC, encoders) is **custom** DarkSlide code.
