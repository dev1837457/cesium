# Camera angle getter drift: cloud validation handoff

## Goal and exact source

Finish the official lint/browser test stage for this focused Cesium fix. This is correctness validation, not a performance or hardware benchmark. The candidate has bounded CPU evidence and independent AI review approving the **next test stage**, not merge readiness.

- Repository: https://github.com/dev1837457/cesium
- Branch: `fix/camera-angle-getter-drift`
- Canonical upstream/base: `CesiumGS/cesium`, `main` at `ad366f071293b28bdf330c822965b8bd0bdeb781`.
- **Implementation commit:** `44506ff8e47b92e10089a6a457d2ee3cc74160f3`. This contains only Camera.js, CameraSpec.js, SceneSpec.js, CHANGES.md and CONTRIBUTORS.md.
- This handoff is a separate documentation commit. Do not include it in an eventual upstream code PR; use the implementation commit plus subsequently validated focused fixes.
- Issue: https://github.com/CesiumGS/cesium/issues/12546
- Existing diagnosis: https://github.com/CesiumGS/cesium/issues/12546#issuecomment-2773123704
- Implementation patch SHA256 before committing: `eb156320e7f3d89f84e3cc0ed914d2d69784f8c92c3412f672d9640fe28930c3`.
- Camera.js SHA256: `76e4d3915683f1d3de1ff0c90d15a4c56a517631c88126c5e52b13541b49fb96`.

Start by verifying the checkout, commit ancestry, these files and the runtime/dependencies actually used. Do not silently substitute an unrelated Cesium release or test an unmodified installed package instead of the checked-out source.

## Root cause and critical invariants

Old heading/pitch/roll getters temporarily call `_setTransform` into ENU and back. Repeated round trips change public pose and derived caches by roundoff; registering even a no-op `camera.changed` listener enables internal getter reads. The accumulated changes can cross View's EPSILON15 movement detector and cause unnecessary movement events/request-render decisions.

The fix calculates orientation in scratch coordinates. It retains normal `updateMembers` lazy updates, including normalization of genuinely caller-dirtied vectors, but does not temporarily rewrite an initialized camera pose/reference frame. Exact preservation is required **after initialization**, not a blanket prohibition on legitimate first-use cache updates.

Preserve these details when testing or making a minimal correction:

- Same projection ellipsoid and actual-transform conversion for 3D, Columbus View and 2D; same MORPHING undefined results and angle wrapping/singularity formulas.
- For transformed-basis compatibility: cross **raw transformed** direction/up before normalizing direction, up and right. Then use the legacy determinant/EPSILON2 condition and Gram–Schmidt up correction, reciprocal-then-multiply scalar order, and right recomputation without an extra normalization. All results stay in preallocated scratch.
- Do not replace this with camera save/restore, app-level orientation snapping, a global epsilon increase, suppressed micro-motion or a cache freeze. EPSILON2 is the existing basis orthonormalization rule, not a widened motion tolerance.

A review caught and corrected a real intermediate-candidate regression for a triaxial ellipsoid. Keep the numeric CameraSpec regression:

    const e = new Ellipsoid(3000000, 2600000, 2000000);
    // Camera's scene uses GeographicProjection(e) and ellipsoid e.
    camera.setView({
      destination: Cartesian3.fromDegrees(0, 0, 1000000, e),
      orientation: { heading: 0.3, pitch: -0.7, roll: 0.3 },
    });
    camera.lookAtTransform(FixedFrameTransforms.eastNorthUpToFixedFrame(
      Cartesian3.fromDegrees(45, 20, 0, e), e));
    // After normal world/view initialization:
    // legacy and current roll = 0.41386328940640293
    // incomplete intermediate candidate = 0.48018484130842065

The literal expected legacy roll is important: taking an expected angle from the candidate only tests repeatability. Existing triaxial ENU frames can themselves be nonorthonormal. Bounded tests found 52 initialized view/inverse matrix products nonidentity **identically on base and candidate**; this fix preserves that legacy behavior and does not claim to repair unrelated frame construction.

## Existing evidence versus outstanding work

Completed CPU-only evidence (Node 24.22.1; engine 26.4.0/core 0.1.0 math):

- Twelve 600-cycle Camera/View scenarios: zero spurious detections on the candidate, exact initialized public/world/view state preserved. Identity no-op listener baseline 93 -> 0.
- Exact issue #12546 sphere/offset at an instantaneous CPU flight endpoint: 24 -> 0 detector returns.
- 450 seeded adversarial cases / 1350 original-getter comparisons, including triaxial ellipsoids, random frames and a custom skew projection: zero differences after the correction; the intermediate candidate had 18 roll mismatches. Strengthened candidate runs use 600 repeated reads per case.
- 360 deterministic mode/frame/projection/ellipsoid/pose cases / 1080 comparisons: zero circular angle differences, 600-cycle exact preservation and exact initialized legacy world/view-state parity.
- Translation/rotation/zoom/frustum and changed/moveStart/moveEnd positives; 600 intentional 5e-15 rotations retained and detected in both tested frames; legitimate dirty normalization preserved.
- Focused Node-assert adapter executed 39 selected actual CameraSpec bodies: 20 pre-existing + 19 new, all pass. Reverting only Camera.js to base makes all 19 new stability tests fail; restoring passes. This adapter is **not Jasmine/Karma**, is not checked in, and is not a cloud validation command.
- Changed-source syntax, formatting and whitespace checks passed. Independent AI re-review replayed CPU evidence and found no further proven defect in its bounded review.

**Not completed:** official ESLint; actual Jasmine/Karma Camera/Scene/controller suites; the three new real Scene draw/settle tests; original Viewer/low-targetFrameRate behavior; broader suite. CPU detector returns are **not drawn-frame counts**. No hardware-performance or exhaustive correctness claim is supported.

## Cloud setup and dependency safety

Use an isolated cloud job, not another person's production checkout. First record Node/npm/browser versions, OS, exact commit, dependency resolution/lockfile digest and available test tooling. Repository requires Node >=22.17.0; choose a supported runtime and report the actual version.

**Do not install any dependency or browser artifact published less than 24 hours ago.** Verify exact resolved versions against official registry/release metadata and save publication-time URLs/timestamps. A lockfile date or an old package name is not age proof. Prefer already installed, verified artifacts. If official age cannot be established, ask rather than assuming permission. Do not silently update the lockfile to evade this rule.

Inspect `package.json`, `gulpfile.js`, `Specs/karma.conf.cjs` and the contributor build/testing guides before setup. In this revision the root prepare script is:

    gulp prepare && husky && node scripts/isCI.js || playwright install --with-deps

Do **not** blindly run that lifecycle or download browsers/system packages. After exact dependency age checks and authorization, `npm ci --ignore-scripts` can avoid automatic lifecycle execution; audit any required dependency-specific build steps before explicitly running them. The inspected `gulp prepare` copies existing dependency assets/Jasmine runner files into generated locations. A possible explicitly audited sequence is:

    ./node_modules/.bin/gulp prepare
    npm run build

These are setup instructions, not claims that setup has already run. Do not use auto-installing npx as a substitute for missing tools. Do not change host services/security or use privileged package installation without separate permission. Do not modify CI workflows or dispatch Actions. This fork publication's documentation commit uses a skip marker to avoid an unintended push deployment; that is not test evidence.

## Prove a real WebGL context before render conclusions

Software rendering (SwiftShader/llvmpipe) is acceptable for **CPU-backed correctness** here; no physical GPU is required. Do not assume it exists. Verify the actual browser binary, installed launcher and version-specific supported flags. Record the exact command/flags/environment. Keep any job-only launcher/wrapper separate from production changes; do not invent a repository browser flag.

Using the same binary and flags as Karma:

1. Create an actual WebGL2 context, or an explicitly recorded WebGL1 fallback. Record VERSION, SHADING_LANGUAGE_VERSION, VENDOR, RENDERER and unmasked renderer/vendor through WEBGL_debug_renderer_info when available. If the extension is unavailable, say so; do not fabricate the renderer.
2. Compile/link real vertex/fragment shaders, draw a full-screen triangle to a small canvas with a known nonblack output, and verify gl.readPixels against that color (for example RGBA [51,153,204,255] with a documented one-byte rounding tolerance). Assert compile/link success, no GL error and meaningful pixels. Clearing a canvas or merely importing Scene is not shader/draw proof.
3. Record the result and then run the real Scene tests. If context creation/shader/readback fails, report the blocker and the unexecuted render gates; do not claim render correctness.

**Cesium's `--webglStub` does not satisfy this requirement.** It is a no-op WebGL stub, not SwiftShader or another real software renderer. Do not use it to certify the Scene draw/idle tests or pixel checks.

## Official commands to validate

These command forms are supported by the inspected package scripts and gulp test implementation. Verify the installed karma-chrome-launcher provides ChromeHeadless and points to the proven browser. If not, use a documented installed launcher and record the substitution/blocker. Any software-rendering flags must be validated for the actual browser/launcher; they are not supplied by `--browsers` alone.

    npm run eslint
    npm run test -- --workspace engine --includeName "Scene/Camera" --browsers ChromeHeadless --failTaskOnError
    npm run test -- --workspace engine --includeName "Scene/ScreenSpaceCameraController" --browsers ChromeHeadless --failTaskOnError
    npm run test -- --workspace engine --includeName "stays idle in requestRenderMode with camera" --browsers ChromeHeadless --failTaskOnError
    npm run test -- --workspace engine --includeName "Scene/Scene" --browsers ChromeHeadless --failTaskOnError
    npm run test -- --workspace engine --browsers ChromeHeadless --failTaskOnError
    npm run test -- --browsers ChromeHeadless --failTaskOnError

**Keep `--failTaskOnError`:** gulp's default can otherwise resolve despite a failing Karma exit. Preserve exit status if using pipelines (for example Bash pipefail), and save actual Jasmine selected/executed/passed/failed counts; a zero-test filter is not a pass. View detector regressions are in CameraSpec and SceneSpec; do not invent a missing ViewSpec suite.

Run the three new Scene cases for control/no-op changed listener/getter polling. They use normal initializeFrame -> render, fixed time, requestRenderMode and maximumRenderTimeChange=Infinity, settling before 600 observations, then **actual postRender/frameNumber** checks. Verify positive translation, rotation, zoom, frustum-only change, explicit requestRender and immediate mode transition still draw and settle. Repeat the timing/settling cases enough to detect flakiness. Do not substitute RAF/render-call counts for draws or loosen expectations merely to make a failure green.

Also exercise the original issue's Viewer/top-down/low-targetFrameRate scenario; separate its event/timing result from the instantaneous CPU endpoint evidence.

## Baseline/fix controls and failure handling

Use the **same verified environment, browser flags, timing and tests** for baseline and candidate. Keep the new tests present on baseline: in a disposable checkout at the implementation commit, restore only `packages/engine/Source/Scene/Camera.js` from base `ad366f071293b28bdf330c822965b8bd0bdeb781`, rebuild and run the new regression filters. Never revert the tests or edit the original source branch just to manufacture a red. Use isolated build outputs, and verify served bundle/source hashes so a stale fixed bundle cannot be mistaken for baseline.

Require baseline regression failure and candidate success, plus the real-motion/explicit-request controls. The numeric roll literal itself should match untouched base; the full new numeric spec also requires repeated exact-state preservation, which the old getter does not provide.

If official tests find a real defect, diagnose and make the smallest focused correction preserving angles, lazy semantics, no observation-induced pose changes and intentional micro-motion. Do not blindly change expectations or EPSILON15. Run a revert/restore control for the new correction and seek independent review. Keep unrelated legacy matrix geometry, benchmark work and app-level workarounds out of this patch.

## Deliverable and publication boundaries

Return a concise cloud results document with:

- Exact tested source/base/head SHAs, runtime/dependency/browser versions and official artifact-age proof.
- Exact setup/test commands, flags, selected test counts, exit codes and complete relevant failure/pass outputs.
- Verified context/renderer/shader/readback evidence; baseline/fixed/restore results and repeatability.
- All unexecuted suites, environmental blockers, numerical caveats and any minimal code fixes with regression evidence.

Keep results/launcher diagnostics and this handoff separate from the implementation commit; a later upstream PR should contain only the focused code/tests/change log/contributor metadata plus validated fixes. Sanitize logs: no credentials, private account data or unrelated environment/profiler dumps. Work only within the dedicated fork branch under the later cloud job's authorization. Do not update main, force-push, create upstream issues/comments/PRs or dispatch CI workflows. An upstream PR still needs explicit owner approval and freshly reconciled upstream ancestry/revalidation.

AI disclosure must remain truthful: implementation, tests, this handoff and independent reviews used OpenClaw with OpenAI GPT-6.1. AI self/independent review is **not** an assertion that the account owner personally reviewed the content. Resolve required submitter/human acknowledgment before any future upstream submission.
