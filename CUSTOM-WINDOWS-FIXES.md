# Custom Windows test batch — 29 October 2026

Published and body-verified as upstream [issue #10046](https://github.com/storytold/photocraft/issues/10046)
on 29 October 2026. The published body begins with the report below; this local
publication note was added after posting.

Report from **Nostella** (@tomgardenart), summarizing work in a custom Windows
PhotoCraft checkout. These changes were tested in that checkout and included in
a local optimized build. This branch publishes their actual source and tests.
Seven focused draft code PRs have subsequently been opened against current
upstream; their links are below. Submission does not mean acceptance or merge.

## Completed local work

| Area | Observed problem and resulting change | Main implementation locations |
|---|---|---|
| Windows pen barrel button | Normalize the existing physical button service into secondary clicks before widgets handle input; suppress associated tip clicks and handle release/focus loss. Physical Wacom confirmation remains pending. Already submitted separately as [PR #9923](https://github.com/storytold/photocraft/pull/9923), currently open and unmerged. | `crates/ui-egui/src/input/stylus/pen_button.rs`, raw input hook in `app.rs` |
| Define Brush Preset menu | Include pixel-selection presence in the shared menu cache key. This fixes an open menu staying disabled when a preview selection changes without a document revision. Select All and marquee brush creation are covered. | `crates/ui-egui/src/menus/native_menu.rs`, `menus/tests.rs` |
| Sampled brush cursor | Pencil and Pencil-mode Eraser use the tip-shaped continuous OS cursor shared with Brush. Correct solid rectangle corners and contour centering. Invalidate a stored-tip cursor cache when its preview bitmap is replaced by the full bitmap, even if the stored key is unchanged. | `crates/paint/src/outline.rs`, `crates/ui-egui/src/tools/brush_cursor.rs`, canvas tool input |
| Resized sampled tips | Bilinear interpolation was mixing cropped edge texels with an imaginary transparent border, widening fringes and clipping Pencil corners. Extend edge texels and clip support with a document-pixel-sized antialias band. Interior soft values still interpolate normally. | `crates/paint/src/tile.rs`, `render.rs`, `render/reference.rs` |
| Intermittent missing marching ants | Selections still constrained edits while the outline was invisible. Bounds/outline caches now retain shallow copy-on-write surfaces so pointer fingerprints remain valid through edits and allocation reuse. Regression tests cover initially empty masks and canvas create/open/switch sequences; the user's exact original sequence remains unconfirmed. | `crates/ui-egui/src/app/thumbs.rs`, `canvas.rs`, app cache state |
| Selected pixels across documents | Extend the existing tab/tiled-canvas layer transfer to selected pixels. `layer.copyToDocument` accepts `selectionOnly`; placement, conversion and destination Undo reuse existing paths. The source is preserved, and a floating preview is restored when transfer arms. | `crates/engine/src/cmd/layer_copy_cmds.rs`, `crates/ui-egui/src/panels/layer_transfer.rs`, canvas tabs/drag |
| Locked Move-tool drag | The apparent marquee after merging was traced to a locked layer. Show a two-second **Layer is Locked** overlay on a locked-hit drag instead of a misleading layer-selection box/modal error. Empty-space layer boxing remains available. | `crates/ui-egui/src/tools/move_lock.rs`, `move_box.rs` |
| Magic Wand/Bucket crisp-edge defaults | Wand anti-aliasing generated partial mask coverage outside the marching ants' 50% contour. Fills and strokes therefore blended with previous colors there. Default Wand and Bucket coverage to crisp pixels, each with an independent persisted Anti-alias opt-in. This is a deliberate default-policy choice for pixel work, not a claim that intentional antialiasing is inherently broken. Other tools and explicit feathering remain unchanged. Imported actions with missing `AntA` preserve their former behavior; newly exported bucket actions encode the new default. | Engine selection/paint/action commands; UI `state/tool_options.rs`, options bar and tool dispatch |

## Custom-checkout profile finding

Suspected lost brushes/settings were associated with two different profile
locations: launcher versus direct executable launch. Same-profile data loss was
not reproduced. The custom repo executable now defaults to the launcher's profile,
with explicit environment/portable overrides taking precedence. Both existing
profiles were preserved; no automatic migration or merge was performed. This
repository-specific `apps/photocraft/src/app_dirs.rs` change is separate from a
general upstream persistence fix.

## Investigated without a confirmed fix

- **Adjustment-preview edge softening:** rendered Levels and Hue/Saturation
  comparisons at 800% matched committed edges. No speculative rendering patch.
- **Pixelated soft feather on a 183 × 313 image:** reviewed Krita's official
  [display settings](https://docs.krita.org/en/reference_manual/preferences/display_settings.html)
  and [brush tips](https://docs.krita.org/en/reference_manual/brushes/brush_settings/brush_tips.html).
  Canvas magnification and brush coverage are separate concerns; no clear defect
  was established. Excluded from fixes at the reporter's request.

## Verification and limits

- Paint suite: **133 passed**. Broad engine run: **2,331 passed, 15 ignored**,
  with one spooler timing failure passing in isolation. Broad UI run: **4,184
  passed, 18 ignored**; two new headless harness-cleanup failures were corrected
  and passed focused reruns. These runs preceded the final Wand-default change.
- Final selection/paint/action focused suite: **126 engine and 6 UI tests passed**.
  Also passed: 17 layer-copy, 7 transfer, 10 cursor, 2 lock-overlay and 21 native
  profile-resolution tests. Visual checks covered cursor/edge behavior, preview
  comparisons, and lock overlays at 1×/2× scaling.
- Formatting, strict all-target clippy for changed crates/native app, dependency
  layers, and wasm checks passed (24 crates plus the HEIF feature).
- Optimized Windows build passed. A disposable-profile smoke test drove the
  release UI's default Wand, filled an angular region white and checked all
  **1,024 exported pixels**: selected pixels exactly white, outside pixels
  unchanged. The resulting 1200% screenshot was visually inspected. Uniform
  24 MP Wand calls took approximately **316 ms crisp / 231 ms smoothed** in one
  warm process; these are smoke timings, not a before/after benchmark.
- The broad adversarial-command sweep is **not green**: unchanged
  `file.print.page`, `file.printers`, and `file.printOneCopy` each exceeded 4 seconds
  twice, including outside the sandbox with printing disabled. No changed-command
  failure was reported. Printer code and timeout thresholds were not altered.
- The new build has been opened for user testing. Physical tablet behavior,
  persistence across real restarts, and the original intermittent UI sequence
  still need user confirmation. Private profiles, credentials and personal
  artwork are not included in this report.


## Using this source branch

This branch publishes the completed code and regression tests from the tested
custom build, based on upstream commit `eb6c1abf75c60dda13a3de0aa4e7bdd5f4c43915`.
It does not implement the next-batch interpolation or Extras requests. The
repository-specific app-directory adjustment only activates in the host project
layout; ordinary standalone clones use the normal PhotoCraft profile rules.

```powershell
git clone --branch custom/windows-tested-fixes-2026-10-29 https://github.com/tomgardenart/photocraft.git
cd photocraft
cargo build --locked --release -p photocraft
```

Follow `docs/development.md` for Windows compiler prerequisites and the optional
CRAFT_FONTS_DIR font bundle. This publishes source, not a signed binary release.
Focused upstream pull requests are tracked in the host project's handoff.

## Focused upstream code submissions

- [#10054 — sampled-tip scaling](https://github.com/storytold/photocraft/pull/10054)
- [#10055 — Define Brush menu refresh](https://github.com/storytold/photocraft/pull/10055)
- [#10056 — crisp Wand/Bucket defaults](https://github.com/storytold/photocraft/pull/10056)
- [#10057 — selection-cache lifetime](https://github.com/storytold/photocraft/pull/10057)
- [#10058 — sampled-tip Pencil cursors](https://github.com/storytold/photocraft/pull/10058)
- [#10059 — selected pixels across documents](https://github.com/storytold/photocraft/pull/10059)
- [#10060 — locked Move overlay](https://github.com/storytold/photocraft/pull/10060)

These draft branches are independent adaptations onto upstream `69b8fd9c1`,
not replacements for the older tested source snapshot on this branch. In a
combined review checkout, 143 engine, 77 paint and 196 UI focused tests passed;
formatting, strict all-target clippy, layers and wasm checks passed. The rendered
Wand check and two overlay tests also passed. The same three printer-command
timeouts remain the broad-sweep limitation. Quick performance checks are being
recorded in the PR descriptions. Hardware/user-workflow confirmation is separate.
