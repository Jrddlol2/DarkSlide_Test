# DarkSlide — Feature Inventory

> Complete inventory of user-visible features, from the UI, hooks, worker handlers, and constants. "Desktop only" = requires the Tauri build.

## Supported formats
- **Input (all builds):** JPEG, PNG, WebP, TIFF (`utils/tiff.ts`, UTIF).
- **Input (desktop only):** RAW — DNG, CR3, NEF, ARW, RAF, RW2 (`RAW_EXTENSIONS`, `decode_raw`).
- **Output:** PNG, TIFF, JPEG, WebP (`exportEncoder.ts`). Bit depth 8 always; **16-bit selectable for PNG/TIFF but currently degrades to 8-bit** (`exportOptions.ts:12`, `exportEncoder.ts:422`).

## Editing controls (`ConversionSettings`, `types.ts`)
- Exposure, contrast, saturation, temperature, tint.
- Black point, white point, highlight protection, shadow recovery, midtone contrast.
- Per-channel balance (R/G/B), flare correction, residual base correction toggle.
- **Curves**: master + red/green/blue (`CurvesControl.tsx`), composed with lab-style curves.
- **Black & white mode**: per-channel luminance mixing + split-tone (`blackAndWhite.{redMix,greenMix,blueMix,tone}`).
- **Sharpen** (radius, amount) and **noise reduction** (luminance strength).
- **Crop**: non-destructive, ratio presets (3:2, 4:5, 1:1, 6×7, 6×4.5, …) + rotation (`CropPane.tsx`, `previewLayout.ts`).
- **Film base sample**: manual eyedropper or auto estimate; roll-level base.
- **Light-source correction**: 11 profiles incl. CineStill CS-LITE modes (`LIGHT_SOURCE_PROFILES`), plus custom light sources (`useCustomLightSources`).

## Film profiles (`FILM_PROFILES`, `constants.ts`)
- **40+ built-in color and B&W stocks** across Kodak, Fuji, Ilford, CineStill, Harman/Phoenix, Lomography, Foma, Rollei, etc. Each carries default settings, an optional `colorMatrix`, `tonalCharacter`, and (color) a density-balance preset.
- Categorized (`category`), searchable browser (`PresetsPane.tsx`), with per-stock reference fixtures in `src/test/fixtures/reference/`.

## Presets
- Custom user presets with folders, tags, sorting, search (`useCustomPresets`, `presetStore.ts`).
- Export/import as `.darkslide` JSON files.
- Quick-export presets (format/quality/bit-depth/size) (`quickExportStore.ts`).

## Batch & multi-image
- **Tabbed documents** — work on many images at once (`useDocumentTabs`, `TabBar.tsx`).
- **Batch export** a whole roll, optionally applying a preset to every frame (`batchProcessor.ts`, `BatchModal.tsx`); color-vs-mono decision unified across preview/single/batch (README v1.0.0).
- **Contact sheet** generation — grid overview (`ContactSheetModal.tsx`, `handleContactSheet`).

## Roll / session management
- **Rolls**: group frames with film-stock metadata + filmstrip sidebar (`useRolls`, `rolls.ts`, `RollInfoModal.tsx`).
- **Scanning sessions** (desktop only): live folder watch that imports frames as the scanner writes them (`useScanningSession`, `watcher.rs`, secondary window).

## Undo / redo / history
- 50-step JSON-snapshot undo/redo (`useHistory.ts`), slider drags coalesced via begin/commit interaction.
- Reset adjustments (`Cmd/Ctrl+Shift+R`).

## Preview / view modes
- Real-time preview with multi-resolution levels + tiled rendering for large images.
- **Before/after comparison** (`Cmd/Ctrl+/`).
- Zoom & pan, zoom-to-fit / actual size, magnifier loupe (`MagnifierLoupe.tsx`, `useViewportZoom`).
- **Live histogram**, per-channel (`Histogram.tsx`).
- Crop overlay toggle; adjustments/profiles pane toggles.

## Dust & scratch removal
- **Manual**: paint over dust/hairs/scratches; content-aware fill (`DustPane.tsx`, `DustOverlay.tsx`).
- **Auto-detect** (experimental): marks likely defects for review; sensitivity + max-radius controls (`dustDetection.ts`).

## Color management
- Input profile auto-detect from embedded ICC or override; working/output profiles sRGB, Display P3, Adobe RGB, linear.
- Display-P3 preview when the monitor supports it (`getPreferredPreviewDisplayProfile`).
- Correct ICC embedding in PNG/TIFF exports.

## Metadata
- EXIF read/write (piexifjs); orientation applied on RAW import; output profile description embedded.
- Sidecar settings (`sidecarSettings.ts`) and diagnostic report of exact conversion parameters (`buildConversionParametersDebug`).

## Keyboard shortcuts (`useAppShortcuts.ts`, native menu accelerators in `lib.rs`)
| Action | Shortcut |
|---|---|
| Import | Cmd/Ctrl+O |
| Export | Cmd/Ctrl+E |
| Batch export | Cmd/Ctrl+Shift+E |
| Open in editor | Shift+Cmd/Ctrl+O (macOS) |
| Close image | Cmd/Ctrl+W |
| Reset adjustments | Cmd/Ctrl+Shift+R |
| Toggle before/after | Cmd/Ctrl+/ |
| Toggle crop overlay | Cmd/Ctrl+Alt+C |
| Toggle adjustments pane | Cmd/Ctrl+\ |
| Toggle profiles pane | Cmd/Ctrl+Shift+\ |
| Scanning session | Cmd/Ctrl+Shift+W |
| Zoom fit / actual / in / out | Cmd/Ctrl+0 / 1 / = / - |
| Settings | Cmd/Ctrl+, |
| Pan/compare | Space (hold) |

## Desktop-only integrations
- Native file dialogs, native menus, recent-files menu.
- **Open in external editor** (Photoshop/Affinity/etc.) — macOS only (`lib.rs:455`).
- Auto-update notifications (gated by build env; `useAutoUpdate`, `UpdateBanner.tsx`).
- Open-with / dock file handling (macOS).

## Configuration options
- Settings modal (`SettingsModal.tsx`): preferences persisted in `preferenceStore.ts`.
- Custom light sources, custom presets, quick-export presets, recent files — all persisted locally.

## Not present (gaps vs a pro tool)
- ❌ No CLI / headless / scripting interface.
- ❌ No plugin/extension system.
- ❌ No Lightroom Classic / Capture One integration (NLP is a LrC plugin).
- ❌ No grain synthesis, lens correction/vignetting, or perspective correction.
- ❌ No true 16-bit render/export path; no scene-linear RAW inversion.
- ❌ No ΔE2000 / cLUT ICC support.
- ❌ No local/masked adjustments (only global + curves + spot dust).
