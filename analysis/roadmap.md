# DarkSlide — Prioritized Roadmap

> Goal: bring DarkSlide to best-in-class open-source parity with (and beyond) Negative Lab Pro. Phased so each phase de-risks the next. Estimates assume 1–2 engineers familiar with the codebase.

Legend — Priority: P0 (critical) … P3 (nice-to-have). Difficulty/Time/Impact/Risk: Low/Med/High.

## Phase 1 — Stabilization & parity harness
Make the imaging layer safe to change before changing it.

| Task | Pri | Diff | Time | Impact | Risk | Depends on |
|---|---|---|---|---|---|---|
| Golden-image parity tests: CPU-8bit vs CPU-float vs WGSL | P0 | Med | 1–2 wk | High | Low | — |
| Extract single per-pixel pipeline **spec/core** (one source of truth) | P0 | Med | 2–3 wk | High | Med | parity tests |
| Convert long positional signatures to a `PipelineContext` object | P1 | Low | 1 wk | Med | Low | core |
| Consolidate per-stock data into one typed `FilmStock` record | P1 | Med | 1 wk | Med | Low | — |

## Phase 2 — Precision & RAW fidelity (the quality unlock)
| Task | Pri | Diff | Time | Impact | Risk | Depends on |
|---|---|---|---|---|---|---|
| Float / ≥16-bit internal render buffer; remove mid-pipeline 8-bit quantization | P0 | Med | 2–3 wk | High | Med | Phase 1 core |
| Real **16-bit PNG/TIFF export** from the float raster (+ real deflate) | P0 | Med | 1–2 wk | High | Low | 16-bit buffer |
| **Linear-RAW render path**: drop `SRgb`/`Calibrate` from `decode_raw`, invert scene-referred | P0 | High | 3–4 wk | High | High | 16-bit buffer |
| Move exposure/contrast/tone math to scene-linear domain | P1 | Med | 2 wk | Med/High | Med | 16-bit buffer |

## Phase 3 — Color science (authentic look)
| Task | Pri | Diff | Time | Impact | Risk | Depends on |
|---|---|---|---|---|---|---|
| Per-stock **density characteristic curves** (replace single γ + linear scale) | P1 | High | 3–4 wk | High | Med | Phase 2 |
| Measured / fitted per-stock color renderings (patch-based, not hand-tuned matrices) | P1 | High | 4–6 wk | High | Med | curves |
| Scene-referred **auto WB/neutralization** after inversion | P1 | Med | 2 wk | Med/High | Med | Phase 2 |
| ΔE2000 + optional cLUT ICC input support | P2 | Med | 2 wk | Med | Low | — |

## Phase 4 — Professional workflow
| Task | Pri | Diff | Time | Impact | Risk | Depends on |
|---|---|---|---|---|---|---|
| **Local/masked adjustments** (radial/linear/brush) | P1 | High | 4–6 wk | High | Med | Phase 1 |
| Roll-level sync + "apply to selection" across tabs | P1 | Med | 2 wk | Med | Low | — |
| **CLI / headless** batch conversion (Rust or Node) | P2 | Med | 2–3 wk | Med | Low | Phase 1 core |
| Lens correction / vignetting; perspective correction | P2 | High | 3–4 wk | Med | Med | — |
| Lightroom/Capture One **bridge or export recipe** | P3 | High | 4 wk | Med | High | — |

## Phase 5 — Performance optimization
| Task | Pri | Diff | Time | Impact | Risk | Depends on |
|---|---|---|---|---|---|---|
| GPU export path (reuse WGSL render for full-res export) | P1 | Med | 2 wk | Med/High | Med | Phase 2 |
| Worker pool for batch / contact sheet | P2 | Med | 1–2 wk | Med | Low | — |
| WASM/SIMD core shared with the CPU pipeline | P2 | High | 3–4 wk | Med | Med | Phase 1 core |
| Fuse per-pixel passes; float Gaussian intermediates | P2 | Low | 1 wk | Low/Med | Low | core |

## Phase 6 — UI modernization & extensibility
| Task | Pri | Diff | Time | Impact | Risk | Depends on |
|---|---|---|---|---|---|---|
| Break up `App.tsx` / `imageWorker.ts` into feature modules | P1 | Med | 2–3 wk | Med | Med | Phase 1 |
| Plugin/extension surface for stocks & renderings | P2 | High | 3–4 wk | Med | Med | FilmStock record |
| Sign/notarize macOS; sign Windows; harden Linux packaging | P2 | Med | 1–2 wk | Med | Low | — |

## Phase 7 — Advanced color science (frontier)
| Task | Pri | Diff | Time | Impact | Risk | Depends on |
|---|---|---|---|---|---|---|
| Spectral dye-density model / negative characteristic inversion | P2 | High | 6–8 wk | High | High | Phase 3 |
| Lab-emulation renderings (Frontier/Noritsu-style) | P2 | High | 4–6 wk | High | High | Phase 3 |
| Learned dust/scratch segmentation; IR/ICE channel support | P3 | High | 4–6 wk | Med | Med | — |

## Critical path
Phase 1 (core + parity) → Phase 2 (float/16-bit + linear RAW) → Phase 3 (color science). These three phases close the largest gaps with NLP; Phases 4–7 add differentiation and polish.
