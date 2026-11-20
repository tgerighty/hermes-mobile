# Duo layout verification — 2026-11-19

Supplement to [goncharik/hermes-mobile#181](https://github.com/goncharik/hermes-mobile/pull/181).
The production fix is the existing work by tgerighty. This contribution extends its tests
and records native simulator evidence; it changes no production source.

## Reproduction and result

On unchanged upstream `52dae9b` (1.0, build 96), the unfolded portrait display is blank.
Opening the sidebar reveals the selected chat inside that narrow sidebar. Landscape also
leaves the chat in the sidebar and the main detail area empty. This reproduces without a
server: all captures use the existing public `HERMES_DEMO=chat` fixture.

On PR #181 head `7ca22c27095d0f4224b21d8768c2724eb092a245`, the selected chat fills the
unfolded portrait detail area; unfolded landscape shows the session list and chat side by
side. The folded display keeps the single chat stack. A synthetic unsent draft (`Duo `)
survived folding, rotation and unfolding. The composer remained above the software keyboard
on both screens. Light and dark appearances were checked.

These PNGs are unedited `simctl io screenshot` captures from Xcode 27.2 / iOS 27.2, with
the hinge and orientation controlled in Device Hub. No physical-device, live-server, microphone,
recording, or notification test is claimed. The demo transport is inert; running-turn socket
and queued-work preservation are covered by the existing reducer suite, not live-server testing.

Measured native panels: outer **1398×2034 pixels at 3×** (466×678 points); inner
**2007×2853 pixels at 3×** (669×951 points), with width/height exchanged in landscape.
Safari's content viewport is not used to size these native checks.

| Case | Before | After |
| --- | --- | --- |
| Unfolded portrait | [Blank detail](before-inner-portrait-blank.png), [chat in sidebar](before-inner-portrait-sidebar.png) | [Full chat detail](after-inner-portrait.png) |
| Unfolded landscape | [Narrow chat / blank detail](before-inner-landscape.png) | [List and chat side by side](after-inner-landscape.png) |
| Folded display | Not changed by the fix | [Portrait](after-outer-portrait.png), [landscape](after-outer-landscape.png) |
| Software keyboard | Not a separate baseline defect | [Inner](after-inner-keyboard.png), [outer](after-outer-keyboard.png) |
| Dark appearance | Not a separate baseline defect | [Inner landscape](after-inner-dark-landscape.png), [outer keyboard](after-outer-dark-keyboard.png) |

All text is from the repository's demo fixture or the synthetic draft. No personal screenshots,
credentials, real server addresses or private chat history are included.

## Automated checks

- **42 iOS tests passed, zero failures**, on PR #181's exact production source plus these test
  additions: AppLayoutTests (5), ChatColumnLayoutTests (4), ComposerTextViewTests (32),
  TranscriptKeyboardScrollTests (1).
- **2,970 package tests passed**, with **two known issues** reported by the suite. HermesKit
  is byte-identical between upstream `52dae9b` and PR #181 head.
- The complete pixel-baseline snapshot suite was not run: this machine has iOS 27 runtimes,
  while the repository's golden snapshots require iOS 26. Baselines were not re-recorded.
- An earlier baseline layout-test run reached the expected regular-layout assertion, then
  stalled in XCTest failure-report symbolication and was interrupted. It is not counted as a
  completed failing test run; the native before captures above are the completed visual reproduction.

From a generated workspace, the iOS checks are reproducible with:

```sh
xcodebuild test \
  -workspace HermesMobile.xcworkspace -scheme HermesMobile -configuration Debug \
  -destination 'platform=iOS Simulator,name=iPhone Duo' \
  -only-testing:HermesMobileTests/AppLayoutTests \
  -only-testing:HermesMobileTests/ChatColumnLayoutTests \
  -only-testing:HermesMobileTests/ComposerTextViewTests \
  -only-testing:HermesMobileTests/TranscriptKeyboardScrollTests \
  -skipMacroValidation -parallel-testing-enabled NO \
  -test-timeouts-enabled YES -maximum-test-execution-time-allowance 60 \
  CODE_SIGNING_ALLOWED=NO
./scripts/test.sh
```

The per-invocation macro-validation flag permits the project's existing package macros on
this contributor machine. No global Xcode trust setting was changed.

`AppLayoutTests` now checks that the single transcript and composer are descendants of the
actual `UISplitViewController` secondary controller, and that regular layout has no sidebar
chat marker. Size cases at 669×951, 951×669 and 466×678 check layout classification only;
the native PNGs separately demonstrate real display behavior.
