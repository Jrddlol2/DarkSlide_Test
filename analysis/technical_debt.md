# DarkSlide — Technical Debt & Refactoring Opportunities

> Ordered roughly by leverage. Each item lists effort (S/M/L) and impact (Low/Med/High). No code was changed.

## Architecture rating justification

| Category | Score | Why |
|---|---|---|
| Maintainability | 6 | Strong tests + types, but complexity concentrated in 3 giant files. |
| Readability | 7 | Clear naming, good comments explaining *why* (esp. imaging). |
| Modularity | 6 | `utils/` is well factored; `App.tsx`/worker are not. |
| Coupling | 5 | `App.tsx` knows about nearly everything; worker protocol is wide. |
| Cohesion | 6 | Imaging modules cohesive; orchestrators do too much. |
| Scalability | 6 | Tiling/preview levels scale; export/batch single-threaded. |
| Extensibility | 5 | Adding a stock = editing several parallel maps; no plugin surface. |
| Duplication | 4 | Pipeline implemented 3× (8-bit CPU, float CPU, WGSL). |
| Performance | 6 | GPU render good; 8-bit precision + JS export drag it down. |
| Memory | 6 | Eviction + sampling caps good; some avoidable full copies. |
| Thread safety | 8 | Worker isolation + Rust `Mutex`; `catch_unwind` on FFI. |
| GPU readiness | 7 | Real WGSL pipeline already exists with CPU parity. |
| Cross-platform | 6 | macOS-first; Win/Linux experimental; editor-open macOS-only. |

## Large files / god modules
- **`App.tsx` (~2960 lines)** — root orchestrator holding tabs, panes, import/export, settings, shortcuts. **Effort L, Impact High.** Split into feature controllers/contexts (import, export, editing, rolls).
- **`imageWorker.ts` (~2060 lines)** — decode, preview levels, tiling, analysis caches, dust, export, contact sheet in one file. **Effort L, Impact High.** Split by responsibility (document store, tiling, analysis, export).
- **`imageWorkerClient.ts` (~1820) / `App.test.tsx` (~3520)** — large but mostly mechanical.
- **`constants.ts` (~1460)** — data-heavy (fine), but mixes data + logic (`resolveLightSourceIdForProfile`, etc.). **Effort S.** Move film data to JSON/data modules.

## Duplicated logic (highest structural risk)
- **Three pipeline copies** must be kept in manual parity: `processImageData` (`imagePipeline.ts:1211`), `processFloatRaster` (`:1411`), and `tiledRender.wgsl`. Comments even flag the "exact parity" requirement (`colorProfiles.ts:419`). **Effort M, Impact High.** Extract one per-pixel spec; generate/share the loop; add golden-image parity tests across all three (partial parity tests exist).
- **Effective-settings/tonal-character resolution** is copy-pasted at the top of `buildProcessingUniforms`, `processImageData`, and `processFloatRaster` (`imagePipeline.ts:938-945`, `1235-1242`, `1435-1442`). **Effort S.**

## Magic numbers / hardcoded constants
- Many perceptual thresholds are inline (e.g. highlight threshold `200/255`, shadow knee `0.25`, contrast formula constants, WB chroma/luma bands in `autoAnalysis.ts`). `FILM_BASE_CONFIDENCE` is a good model — **centralize the rest** into named, documented constants. **Effort S, Impact Med.**
- `LUMA_R/G/B = 0.299/0.587/0.114` (Rec.601) duplicated across files; some use it for perceptual luma where Rec.709/linear luma would be more correct. **Effort S.**

## Precision / correctness debt (also quality — see `pipeline.md`)
- **8-bit quantization before the curve LUT** in the float path (`imagePipeline.ts:1371-1377`, `1572-1578`) — a design compromise that blocks true high-depth output. **Effort M, Impact High.**
- **RAW `SRgb` + `Calibrate` develop** (`lib.rs:104-111`) bakes display encoding + camera color into the source of the inversion. **Effort M, Impact High.**
- **Display-domain tone math** — exposure/contrast applied on encoded values, not scene-linear. **Effort M, Impact Med/High.**

## Poor abstractions / API smells
- **Very long positional parameter lists**: `processImageData`, `processFloatRaster`, `buildProcessingUniforms`, `resolveDensityInversionParams`, `computeResidualBaseOffset` each take 10–24 positional args. Easy to mis-order; hard to extend. **Effort M, Impact Med.** Introduce a `PipelineContext`/options object.
- **Parallel per-stock maps** (`COLOR_MATRICES`, `TONAL_CHARACTERS`, `FILM_STOCK_DENSITY_PRESETS`, `FILM_PROFILES`) keyed by string id with no compile-time guarantee they stay in sync. **Effort M.** One `FilmStock` record per stock.

## Dead / degraded paths
- 16-bit export exists in the encoders but is **unreachable** (always degrades) — either wire the high-depth path or mark it explicitly experimental in the UI (README already hedges). **Effort M.**
- Removed features referenced in history (flat-field, H&D pipeline — README v0.8.2) appear cleaned up; verify no orphaned settings remain in `types.ts`/presets. **Effort S.**

## Testing debt
- Excellent unit coverage on pipeline math, color, film base, dust, export. Gaps: **cross-path parity** (CPU vs WGSL) golden images, RAW-develop-to-render integration, and end-to-end export byte-level tests for 16-bit. **Effort M, Impact Med.**

## Platform debt
- "Open in editor" macOS-only; Windows/Linux unsigned + experimental. Document support tiers and add CI smoke tests per platform. **Effort M.**

## Suggested remediation order
1. Extract a **single pipeline core** + parity golden tests (de-risks all imaging work).
2. Introduce **float/≥16-bit buffers**; remove mid-pipeline quantization.
3. Add a **linear-RAW render path**.
4. Break up `App.tsx` and `imageWorker.ts`.
5. Consolidate per-stock data into one typed record; centralize magic numbers.
