# Testing instead of trusting: case studies from LLM-built projects

## Foreword

> "A computer can never be held accountable, therefore a computer must never make a management decision."
>
> IBM training material, 1979

Everything in my public repositories was built with LLM coding tools. My work was the planning, orchestration, and validation: setting the objectives and parameters, directing the model's workflows in parallel, and testing instead of trusting the output. The model produces code that compiles, passes its own checks, and arrives with a confident summary, but errors in interpretation and judgment can still appear and accumulate if unchecked.

The cases below are examples of the types of mistakes that even an advanced model can make, and a qualified engineer is needed to detect and remediate.

## Mismatched criteria and descriptors

### The "cap" that didn't cap

**Project:** [OpenRCT2 screensaver](https://github.com/Ogrueb/openrct2-screensaver), a plugin that moves the camera between chosen viewpoints.

**Problem:** Camera pans had to be slow enough to read. I had rated speeds by eye (500 px/s at close zoom, 100 px/s less per zoom level out), and the model reported applying a 100 px/s cap. Pans were running at 1,900 px/s.

**Error:** The value named "cap" only decided when the camera zoomed out before a pan. Nothing limited speed, and the model's summary described the name, not the behavior.

**Testing:** I asserted the definition: "If the 'cap' doesn't actually function as a cap, it's mis-named." The value was renamed to what it was, a zoom threshold, and a real speed limit was added, tuned from my ratings instead of a default.

### The ride-pan chord test

**Project:** [OpenRCT2 screensaver](https://github.com/Ogrueb/openrct2-screensaver), the view generator that picks what the camera shows.

**Problem:** Views meant to follow a roller coaster's track sometimes traced the park's edge, left the park, or, in the model's words, "tracked nothing visible".

**Error:** The model's test accepted any straight line with track within three tiles of every point. In a dense mesh of coaster track almost any line passes, so the generator picked chords through the mesh instead of paths along a ride.

**Testing:** I reviewed every generated view, park by park. One pan left the park and another followed nothing. Neither could lie along a ride, which pointed at the selection test rather than the camera code the model was debugging. The fix requires one connected stretch of track (a breadth-first search with a detour limit of 1.5 times the straight-line distance), plus checks that the frame is filled and the track visible. I refused per-park tuning across two more rounds, because a heuristic that needs a special case per park is not a heuristic.

### Commonalities

In both cases the code matched the model's reading of the task, not the one that would match the intended result.

## Invalid assumptions and missing context

### Game pixels and the scale factor

**Project:** [OpenRCT2 screensaver](https://github.com/Ogrueb/openrct2-screensaver).

**Problem:** Views I chose on a 4K monitor came out tighter on a 1080 display. The model blamed the resolution and built a 3840×2160 reference into the view scaling.

**Error:** OpenRCT2 renders at an internal scale factor. Differences in results were attributed to the varying resolutions between tests, instead of the fitting function on different aspect ratios.

**Testing:** I asserted the single reference: 1920×1080 game pixels, fitted to whatever aspect the display has via the scaling factor. The code or plugin API couldn't explain a perceptual change, so the model made the most-probable, least-complex assumption, which had to be explicitly corrected.

### A retired signing key

**Project:** [nonstandard-notes](https://github.com/Ogrueb/nonstandard-notes), a fork of the Standard Notes Android app, built and signed with my own key.

**Problem:** After publishing the fork, I moved its builds to a new release key and retired the environment file holding the dev key's settings. In the next session Gradle failed with a keystore error. The model said the keystore in the build location was "not your master key. It's a different file", labeled it unknown, and proposed a two-key setup.

**Error:** The commands still sourced the retired file, loading the dev key's password and alias against the release keystore. The model had it backwards: the "stray" file was the correct release key, as its own plan from earlier that day said.

**Testing:** I ran the model's proposed checks: a byte comparison and a keystore listing showed that the old password opened the old key, not the file in the build location. I identified the commands' key reference as the error and rejected the two-key setup to re-assert the correct key, and the model added a fingerprint check to catch future mismatches.

### The GDI+ replay under a transform

**Project:** [FTR-header-animations](https://github.com/Ogrueb/FTR-header-animations), a reverse-engineering of the animated headers in the Windows Vista, 7, and 10 file-operation dialogs, with a Python renderer that reproduces them to within about one level of 255.

**Problem:** To separate renderer error from recording noise, the model built a harness that replayed the decoded draw lists through Windows' own GDI+ library as a calibration reference, drawing "as DUI paints". Radial gradient fills came out in the wrong place.

**Error:** The harness drew under a GDI+ world transform, which misplaces path-gradient brushes. The real player transforms paths to device space first and folds the shape's matrix into the brush.

**Testing:** I ran the replay in a Windows virtual machine and compared it with my lossless captures. Radial fills were off while everything else agreed. With points transformed before drawing, the replay matched to within about 0.7 of a level out of 255. Unnoticed, the calibration reference would have been wrong, and the renderer fitted to a mistake.

### Commonalities

Each failed at a boundary the model could not see: a display setting, a key file, a graphics library's rule. Each catch was a fact about that boundary, incorporated into explicit parameters.

## Unit tests pass, then fail in the field

### The pan harness that reported arrival

**Project:** [OpenRCT2 screensaver](https://github.com/Ogrueb/openrct2-screensaver).

**Problem:** Each park's last view pans back to its first before the next park loads. The model's harness simulated every pan and reported each one arriving in time. On screen, the return pan was far too slow.

**Error:** The harness skipped the zoom-out leg of the return pan, so it measured a shorter move than the real one.

**Testing:** I visually checked the full loop. The fix was a stepped zoom-out with the rotation at the end.

### The Qt 6 port

**Project:** [Lightscreen-Rd](https://github.com/Ogrueb/Lightscreen-Rd), a maintained fork of the Lightscreen screenshot tool, moved to Qt 6 and from qmake to CMake in staged pull requests.

**Problem:** The port compiled cleanly, and the model marked it done while tests were still running. Behavior had changed silently in three places: the Windows event handler wasn't called, area selections were misaligned, and transparencies flattened.

**Error:** None of the three caused a compiler error or a crash. A clean build and a green checklist were consistent with all three.

**Testing:** Before porting, I had the plan add behavior tests for settings and the area-selection dialog, and run CI on Windows as well as Linux. After the port, the area-dialog tests caught the misaligned selections (330 of 331 passing), and the Windows CI tests caught the flattened transparency. All three regressions were fixed before merge.

### Header text attributed to ClearType

**Project:** [FTR-header-animations](https://github.com/Ogrueb/FTR-header-animations).

**Problem:** The header strips carry the dialog's text over the animation. The model's README called the text "close rather than identical" to Windows and blamed ClearType, which cannot be reproduced exactly.

**Error:** It wasn't ClearType. The renderer drew the text at four times the size and scaled it down, discarding the font's hinting, and the image library's shaping engine placed letters at unhinted fractional widths. The text came out 1 to 3 px wider than Windows draws it, masked by a fitted offset, and the model's validation had passed it.

**Testing:** I asked for additional methods to isolate the residual. A brightness-only comparison showed the shape error exceeded the color error, so the cause was geometry, not anti-aliasing. Drawing with the basic engine at final size matched the recordings to the pixel, and results were regenerated and rechecked.

### Commonalities

In each case the model's own tests passed, and the catch came in places the model didn't anticipate a problem. A headless key test passed while the keys were dead, mock tests were gated behind plugin pull requests, and fixes were shown only in the model's own renders. A test the model writes checks the model's understanding. Only a live run checks what it didn't think of.

## (Anti)Patterns

- **Mismatched criteria** are caught by holding the model's terms against the behavior, and by reviewing the output against the specification rather than the summary.
- **Invalid assumptions** are caught with context the model doesn't hold, converted into explicit facts and testable criteria.
- **False passes** are caught by live runs, by questioning residuals the model has discounted, and by adding independent tests and CI.

## Added safeguards

| Rule | Project | Case |
| --- | --- | --- |
| A value is not a cap, a limit, or a filter until its behavior matches the name. | OpenRCT2 screensaver | The "cap" that didn't cap |
| Tune from rated ladders, not from the model's default guesses. | OpenRCT2 screensaver | The "cap" that didn't cap |
| No special-casing to make a heuristic pass review. If it needs one, the heuristic is wrong. | OpenRCT2 screensaver | The ride-pan chord test |
| Record the environment as facts the model must reconcile with: the display, the scale factor, the build configuration. Put them in the project instructions, not in a chat message a later session will not see. | OpenRCT2 screensaver | Game pixels and the scale factor |
| One signing key. Verify its fingerprint before any build, and never source the retired key's file. | nonstandard-notes | A retired signing key |
| Build safety-net tests for behavior before porting code. | Lightscreen-Rd | The Qt 6 port |
| Measure deviations and isolate their causes, instead of naive compensations. | FTR-header-animations | The GDI+ replay; Header text attributed to ClearType |
| Check output files against the model's claims, and contradict an erroneous diagnosis. | All | Throughout |
