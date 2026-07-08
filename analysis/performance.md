# DarkSlide — Performance Review

> Sources: `imagePipeline.ts`, `imageWorker.ts`, `gpu/WebGPUPipeline.ts`, `exportEncoder.ts`, `rawImport.ts`.

## What is already good
- **Off-main-thread**: all heavy imaging runs in a Web Worker; the UI thread stays responsive.
- **WebGPU tiled render** with a CPU fallback (`tiledRender.wgsl` mirrors `processImageData`'s uniforms).
- **Multi-resolution preview levels** + **tiled reads** (`buildPreviewLevels`, `handlePrepareTileJob`/`handleReadTile`) so full-res pixels are only touched on demand.
- **Pinned, cached analysis**: film base, density balance, residual base offset, highlight density are computed once per document on downsampled frames and reused for preview and export (deterministic + cheap).
- **Sampling caps**: film-base and balance estimators bound reads (`ANALYSIS_MAX_SAMPLES ≈ 260k`, strides in `computeDensityBalance`/`computeResidualBaseOffset`).
- **Transferable ArrayBuffers** across the worker boundary; **reused scratch buffers** for Gaussian passes (`getScratchBuffers`).
- **Memory eviction**: `estimateMemoryBytes` + `handleEvictPreviews` bound resident documents.

## Bottlenecks & issues

### 1. 8-bit quantization mid-pipeline (quality + rework)
- Even the "float" path quantizes to 8-bit to index the curve LUT (`imagePipeline.ts:1371-1377`, `1572-1578`). This caps precision and forces a second 8-bit round for any high-depth ambition. **Impact:** banding; blocks real 16-bit export. **Fix:** float curve evaluation / higher-res LUT; keep buffers float. *Expected: eliminates banding, unlocks 16-bit; small CPU cost.*

### 2. Duplicated per-pixel loops
- `processImageData` and `processFloatRaster` are ~180 lines each of near-identical logic (`imagePipeline.ts:1211` vs `:1411`), plus the WGSL copy. **Impact:** 3× maintenance, drift risk, no single place to optimize. **Fix:** one shared per-pixel core (or generate the loop). *Expected: maintainability win; enables a single SIMD/GPU optimization to benefit all paths.*

### 3. Repeated full-image passes on export/preview
- Inversion, spatial NR, sharpen, and histogram are separate passes over the image (`processImageData` then `applyNoiseReduction` then `applySharpen` then `accumulateHistogram`). **Fix:** fuse per-pixel stages; compute histogram inline (already partly done). *Expected: ~15–30% fewer memory passes on large exports.*

### 4. RAW down-convert to 8-bit before render
- 16-bit RAW is decoded (`rgb16ToRgba8`) to an 8-bit preview buffer for rendering; the 16-bit buffer is only used for film-base estimation and is dropped past `MAX_HIGH_DEPTH_RAW_PIXELS` (`rawImport.ts:76-97`). **Impact:** the highest-fidelity data never reaches the converter. **Fix:** render from the high-depth buffer through a float pipeline. *Expected: major quality gain on RAW; more memory.*

### 5. Gaussian H-pass 8-bit round-trip
- `separableGaussianBlur` writes the horizontal pass back to `Uint8ClampedArray` before the vertical pass (`imagePipeline.ts:1165-1173`). **Impact:** precision loss + extra clamp/round. **Fix:** keep float between passes (or use the WGSL blur). *Expected: cleaner NR/sharpen, minor speed.*

### 6. `JSON.stringify` in the undo hot path
- `useHistory.push`/`commitInteraction` compare states via `JSON.stringify` and `structuredClone` on every commit (`useHistory.ts:18,40`). For large settings this is O(size) per interaction. **Impact:** minor now, scales poorly. **Fix:** structural equality / dirty flag. *Expected: negligible→noticeable on rapid edits.*

### 7. Export encoders are single-threaded JS
- `encodePng`/`encodeTiff` build full byte arrays on the worker; 16-bit PNG uses **stored (uncompressed) deflate** (`binaryEncoding.deflateStore`). **Impact:** large files, slower writes. **Fix:** real deflate (fflate/pako) or a Rust encoder command on desktop. *Expected: much smaller 16-bit PNGs, faster export.*

### 8. Contact sheet / batch are sequential
- `handleContactSheet` and `batchProcessor` process frames one at a time in a single worker. **Fix:** a small worker pool (multiple `imageWorker` instances) for batch/contact-sheet. *Expected: near-linear speedup with cores on batch jobs.*

## SIMD / GPU / parallel opportunities
- **GPU:** extend the existing WGSL render to the export path so large exports use the GPU (currently CPU float/8-bit paths dominate export). GPU histogram shader already exists (`histogram.wgsl`).
- **SIMD:** the per-pixel core is a natural fit for WASM SIMD if a shared Rust/WASM core replaced the JS loops.
- **Parallel:** worker pool for batch; tile-parallel render already structurally supported.

## Priority ranking (impact × effort)
1. Float/≥16-bit render buffer (unblocks quality + 16-bit) — **high impact, medium effort**.
2. RAW linear/float render path — **high impact, medium effort**.
3. De-duplicate pipeline into one core — **medium impact, medium effort, de-risks everything else**.
4. Worker pool for batch/contact sheet — **medium impact, low-medium effort**.
5. Real deflate for 16-bit PNG — **low-medium impact, low effort**.
