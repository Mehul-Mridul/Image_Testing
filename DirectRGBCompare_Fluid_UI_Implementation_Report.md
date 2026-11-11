# DirectRGB Compare — Fluid UI / Chrome Motion Release

**Type:** Integrated Windows-oriented source upgrade built on **Premium Studio + Adaptive UI Recovery**, not a replacement prototype.  
**Runtime certification:** **NOT VERIFIED** — source developed/tested on Linux without Windows, PySide6, Win32/D3D11, live libmpv, or FastStone installed.  
**Source color contract:** `policy.py`, `mpv_native.py`, `validation.py`, `color_math.py` SHA-256 match the prior Premium Studio release exactly.

## 1. Evidence-based feel audit (baseline before this change)

The scores are **source-review estimates**, not ratings derived from Windows user input or observed animation. Do not treat them as tested UX measurements.

| Aspect | Before /5 | Source evidence | After status |
|---|---:|---|---|
| Input feedback | 2 | QSS `:pressed` existed, but few distinct pointer-down affordances | Press feedback now immediate on new transport buttons; **runtime needs verification** |
| Direct manipulation | 3 | QSlider mapped cursor to position; splitter drag called expensive candidate search per mouse move | Direct manual splitter path, non-spring thumb; **runtime needs verification** |
| Interruptibility | 1 | Dock toggles directly called show/hide; no interruption model | Reversible Qt-only content fades from current opacity; **runtime needs verification** |
| Spring/motion | 1 | No shared spring model; theme only | Reusable critically damped velocity-preserving spring engine; **runtime needs verification** |
| Spatial consistency | 2 | Panels snapped open/closed and active selection border changed immediately | Anchored dock fades and selected-pane chrome strip; no unsafe native-pane movement |
| Materials/hierarchy | 3 | Graphite/Obsidian/Midnight QSS theme, dark panels and native near-black video host | Reuses restrained dark surfaces, refined transport states; **visual result needs verification** |
| Reduced motion | 1 | No setting | Persistent Reduce Motion, optional Windows accessibility preference; **runtime needs verification** |

## 2. Actual changes and interaction behavior

| Interaction | Normal path | Interrupted/reversed path | Bounds / failure behavior | Reduced-motion alternative |
|---|---|---|---|---|
| Play/step buttons | Immediate down-state followed by subtle chrome outline settling | Pointer-out / release retargets the spring without resetting state | Always invoke real transport action via original callback; no animation gates | Instant highlight reset |
| Active pane | Existing selection updates immediately; new 3px header strip animates | Repeated pane changes preserve spring velocity and retarget | Active indicator lives above native video host | Immediate strip state |
| Timeline | Pointer directly sets slider value; separate 16ms lightweight clock while playing | Scrubbing pauses group, coalesces decode previews at 95ms, commits exact requested target on release; resumes prior play state when appropriate | Slider clamped 0 to maximum-duration horizon; shorter videos keep prior EOS handling | Same direct, unsmoothed positioning; only optional hover chrome stops animating |
| Sidebar | QDockWidget appears immediately; its *Qt-only contents* fade over 170ms | Close/open during animation starts from actual current opacity (135ms close) | Dock geometry managed by Qt; `Ctrl+Shift+R` restores browser instantly | No fade; immediate show/hide |
| Splitter | Native pane rectangles follow manual divider position immediately; chosen topology held stable during drag | Reverse drag tracks pointer without interpolating coordinates | Split ratios clamped 0.10–0.90; pointer release/focus loss cancels grab | Identical direct dragging |
| Layout switching | Existing adaptive solver still chooses layout when loading/resizing | Mid-action changes request new layout; no video/HWND tweening | Native video frames never blended/crossfaded; no video window reparenting | Same immediate layout change |
| Zoom/pan | Existing cursor-centred zoom and direct pan | New UI does not add time-based interpolation | Existing source-to-viewport and DPI geometry; native-event routing needs Windows check | Same direct pan/zoom |
| Menus | Context menu appears immediately from the relevant toolbar control | Immediate native Qt dismissal | Do not layer animated translucent widgets over videos | No motion |
| Loading | Existing asynchronous metadata probe and per-pane status | New choices supersede stale probes as in previous release | No source file changes; no new decoder paths | Same loading behavior |

### Precision and synchronization

The **longest-duration comparison timeline remains intact**, with offsets respected. Shorter clips still hold their last valid frame on a best-effort basis. `ComparisonTimeline` and the underlying `MpvPanePlayer` are **unchanged**.

The new 16ms UI timer updates only the slider/text from the existing monotonic comparison clock; the legacy decoder/PTS/dropped-frame poll remains at 150ms. It neither decodes frames nor requests seeks.

During scrubbing, slider position follows input, and the **requested** timestamp is displayed. A bounded 95ms coalescing timer submits asynchronous seeks; while decoding catches up the visible source frame may lag. This UI does **not** falsely claim a preview frame is already presented or that an old mpv command can always be cancelled after dispatch. A final explicit seek is submitted on release.

## 3. Motion specification

| Element | Implementation | Parameters | Why |
|---|---|---|---|
| General selection chrome | `motion.py:Spring`, `fluid_interactions.py:MotionSystem` | Exact damped solution; natural frequency 3.2 Hz, damping ratio 1.0; Qt timer 16ms while active | Variable frame time stability and velocity continuity |
| Active pane | `ActivePaneStrip` | 3.5 Hz, ζ=1 | No bounce around inspection content |
| Transport button | `FluidButton` | Press immediate, hover/release 4.8 Hz, ζ=1 | Instant response with gentle settling |
| Timeline hover | `PremiumTimeline` | 4.8 Hz, ζ=1; animates only track thickness/handle radius | Precision value remains fully unsmoothed |
| Browser/diagnostics fade | `PanelTransitions` | 170ms open / 135ms close; Qt `QPropertyAnimation`, OutCubic | Opacity is not modeled as physical inertia; reversed from current opacity |
| Splitter | `AdaptiveWorkspace` | No spring/inertia, pointer-direct | Essential pixel/position control |
| Playback cursor | `ComparisonTimeline` | No animation; monotonic source at up to 16ms *UI paint* updates | No timestamp interpolation beyond the existing comparison clock |

The spring uses an analytical solution, rather than a per-frame Euler approximation: it behaves consistently with variable elapsed intervals. Velocity continuity is verified with deterministic unit tests. No motion or material compositor is installed on `NativeVideoHost`.

**Momentum projection is deliberately not used.** It is inappropriate for timestamps, video panning, or exact splitter coordinates. No overshoot is used in the installed springs (ζ=1).

## 4. Materials, typography and protected surfaces

Theme tokens remain in `theme.py`. Default Graphite: window `#0C0E12`, panel `#151A21`, raised `#1D242D`, elevated `#27313D`, border `#354151`, text `#E9EEF4`, accent `#78B7E6`. Obsidian and Midnight remain in the Appearance menu. Chrome uses subtle faux glass (near-opaque Qt QSS), not actual DWM blur of the video. Qt font stack is **Segoe UI Variable → Segoe UI → system fallback**, where supported.

The **video host remains opaque and visually independent**; `policy.py`, `mpv_native.py`, `validation.py`, and `color_math.py` were not changed. No blur, opacity, QGraphicsEffect, crossfade, or spring transform is applied to the native video child HWND. Splitter changes alter the size of its parent pane only, preserving the prior Fit/native video logic.

## 5. Architectural changes

**Created:**
- `directrgb/motion.py`: Qt-independent exact spring physics, validation and reduced-motion snap.
- `directrgb/fluid_interactions.py`: centralized Qt driver; chrome button, active-pane strip and Qt-only dock content transitions.
- `directrgb/system_motion.py`: read-only Windows `SPI_GETCLIENTAREAANIMATION` query with safe fallback.
- `tests/test_fluid_motion.py`, `tests/test_fluid_ui_contract.py`: offline motion, resize and source-guard tests.
- `tools/fluid_gui_smoke.py`, `CHECK_FLUID_GUI.bat`: **Windows-only** real Qt smoke-check script. Does NOT certify D3D11 or RGB.
- `docs/FLUID_CPU_MICROBENCHMARK.json` and this report.

**Modified:**
- `directrgb/premium_ui.py`: real transport controls replaced with fluid Qt buttons; added pane chrome and Reduce Motion menu.
- `directrgb/timeline_widget.py`: precise pointer dragging retained; spring only for chrome hover geometry, semantic accessibility name.
- `directrgb/vnext_gui.py`: interruptible drawer target state, coalesced scrub preview, 16ms UI-only clock, diagnostics, setting persistence and motion cleanup.
- `directrgb/adaptive_layout.py`, `directrgb/adaptive_workspace.py`: direct O(number-of-panes) layout path while splitter is grabbed; draw minimal divider cues in parent gaps.
- `directrgb/styles.py`: Segoe UI Variable preference.
- `directrgb/app_settings.py`: backward-compatible reduced-motion/system-preference settings.
- `README.md`: release navigation.

**Not changed:** libmpv interface, decoder choices, RGB policy, pixel comparison routines, source reference assets, installers and PyInstaller flags. New PySide6 widgets are in `directrgb/` and picked up via import at build.

## 6. Performance

This environment has **no Windows Qt/libmpv runtime**. No claims about 4K FPS, actual input latency, GPU workload, pixel equality or native HWND stability are possible.

A CPU-only Python microbenchmark on this Linux container (see JSON report) measured median values for this instance:
- Existing full four-pane layout search with preference/previous: **15.891ms**.
- New direct manual-divider computation: **0.01239ms**.
- 1,000 analytical spring steps: **0.54ms**.

These figures show reduced *layout computation* while dragging, **not measured desktop frame pacing or actual user-perceived 4K performance**. The full adaptive search is still used when finding a best layout outside of active manual dragging.

## 7. Verification status and release gates

| Test | Status |
|---|---|
| Static/source Python syntax checks | PASS |
| Offline Python test suite incl. frozen RGB core | PASS — see `docs/FLUID_TEST_RESULTS.json` |
| Qt window construction, button press, browser transition, timeline scrub | NOT RUN — run `CHECK_FLUID_GUI.bat` on Windows |
| Same-video two/three/four pane playback | NOT VERIFIED |
| 2K/4K during splitter drag, frame pacing and memory | NOT VERIFIED |
| Mixed DPI 100/125/150/200%, two displays | NOT VERIFIED |
| Rapid sidebar open/close while video plays | NOT VERIFIED |
| Native HWND focus, z-order and clipping | NOT VERIFIED |
| Windows reduced-motion preference | NOT VERIFIED |
| FastStone CMS-OFF numerical parity | NOT VERIFIED |
| GPU frame/P TS sync and VFR last-frame hold | NOT VERIFIED |

Windows acceptance procedure:
1. Run `RUN_FIRST.bat` if dependencies are missing, otherwise `build_windows.bat` in **this new source folder**.
2. Run `CHECK_FLUID_GUI.bat`, then `CHECK_PREMIUM_GUI.bat`, and `CHECK_WINDOWS_GUI.bat`. Archive their outputs.
3. Test 2/3/4 videos while scrubbing, change selected pane, click Play/Pause repeatedly, resize dividers fast, open and reverse the browser panel mid-transition, toggle reduced motion.
4. Repeat with mixed-duration clips (e.g. 2s/8s), mixed orientation, VFR and 2160×3840 content. Verify progress to the longest clip and explicit shorter-clip ended state.
5. Test both with Windows scaling 100%, 125%, 150%, 200%; test native child HWND mouse/keyboard focus and compare configured RGB outputs.
6. Perform independent native RGB capture and FastStone reference comparison before any production color-certification claim.
7. Revert to the old Premium Studio build if the new Qt effects introduce focus, compositing or frame-pacing regressions.

## 8. Do / Don't engineering decisions

| DO | DON'T / why rejected |
|---|---|
| Animate only surrounding chrome selection and buttons | Never crossfade or blur video HWND pixels: would compromise inspection |
| Keep scrub thumb fixed under mouse and coalesce actual seeks | Never spring video time, frame index or PTS: would be misleading |
| Use analytical velocity-preserving springs for decorative state | Never use fixed per-frame Euler or state restart: can jump under load |
| Compute manual divider geometry directly under drag | Never rerun the expensive global layout search every mouse move |
| Persist reduced-motion, respect Windows accessibility when possible | Never make animation mandatory for understanding an error or selection |

**Bottom line:** The redesigned fluid Qt chrome is implemented and checked by pure-code tests. Native Windows layout quality, real input latency, four-way 4K performance, and DirectRGB/FastStone pixel-parity still need real Windows testing.
