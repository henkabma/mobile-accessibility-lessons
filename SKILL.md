---
name: mobile-accessibility-lessons
description: Concrete, hard-won TalkBack/VoiceOver accessibility lessons distilled from three real apps (Uptimer, DirSpeed, Memorecorder) built for a blind developer. Load before or while building any mobile app UI (Jetpack Compose, SwiftUI/UIKit, or Kotlin Multiplatform) that needs to work well with a screen reader, or when debugging a reported TalkBack/VoiceOver problem.
---

# Mobile accessibility lessons

This is a working checklist, not a tutorial — every item here is a real bug or
near-miss found in one of three shipped-or-shipping apps built with and for a
blind developer (Uptimer: Jetpack Compose; DirSpeed: Compose + SwiftUI, ported;
Memorecorder: Kotlin Multiplatform, native UI per platform). Treat these as
things to actively check for, not background knowledge to recall passively.

The single biggest meta-lesson: **a build that compiles, or code that looks like
the textbook-correct pattern, is not verification.** Several of the bugs below
are copy-paste-correct Compose code that silently did nothing, or did the wrong
thing, and were only caught by actually running the app on a real device with a
screen reader. See "Verification discipline" at the end.

## Framework choice affects your accessibility floor

Native UI toolkits (Jetpack Compose, UIKit/SwiftUI) expose a real accessibility
tree largely for free — most native components are accessible by default.
Cross-platform frameworks like Flutter require building a *separate* Semantics
tree explicitly; nothing is accessible by default. If accessibility is a real
requirement (not a checkbox), this tips the framework decision, and no
emulator — on any framework — reliably validates real TalkBack/VoiceOver
behavior. Physical-device testing is required regardless of framework.

## Focus management

**TalkBack applies its own default initial focus, and it usually wins the
race.** When a screen appears, TalkBack picks a default focus target itself
(often the first focusable element, or a toolbar button) — frequently *before*
your own code gets a chance to request focus on the element that actually
matters. In Uptimer this showed up concretely as: a countdown/stopwatch screen
appears, TalkBack lands on the elapsed-time text (the first focusable node),
and because that text's spoken value changes every second, TalkBack narrates
the running count out loud, unprompted.

**The fix that actually works, confirmed on-device (Pixel 10 Pro, real
TalkBack): `Modifier.semantics { traversalIndex = ... }` on the sibling
nodes**, not a `FocusRequester`/`AccessibilityDelegate` chase. TalkBack's
"what gets focus when this screen appears" choice is driven by the semantics
tree's traversal order, independent of visual layout order — so put a lower
`traversalIndex` on the control that should get initial focus (e.g. `-1f` on
a Stop/Pause button) and a higher one on the element that should *not* (e.g.
`1f` on the ticking elapsed-time text), even though the text is drawn above
the button on screen:

```kotlin
Text(
    text = formatElapsed(elapsed),
    modifier = Modifier.semantics {
        contentDescription = elapsedSpoken
        traversalIndex = 1f   // visited/focused after the button below, despite being drawn first
    }
)
// ...
Button(
    onClick = onStop,
    modifier = Modifier.semantics { traversalIndex = -1f }  // gets initial focus
) { Text("Stop") }
```

**A previously-documented approach here — installing a `View.AccessibilityDelegate`
on the parent view to listen for `AccessibilityEvent.TYPE_VIEW_ACCESSIBILITY_FOCUSED`
and steal focus back — was tried and confirmed NOT to work** (built, compiled
cleanly, looked textbook-correct, had zero effect on-device: TalkBack kept
narrating the elapsed text regardless). Reasonable guess at why: Compose's
virtual accessibility nodes don't necessarily route through the real-View
`requestSendAccessibilityEvent` bubbling chain the same way genuine child
Views do, so a delegate on the ComposeView's parent may just never see the
event it's listening for. Unconfirmed — but confirmed *not working* either
way, so don't reach for it; use `traversalIndex` instead. A plain
`FocusRequester.requestFocus()` (with a short delayed fallback, e.g.
`delay(600)`, in case nothing else claims focus at all) is still worth keeping
alongside `traversalIndex` as a minor nicety for keyboard/switch-access
navigation, but it does not reliably move TalkBack's *spoken* cursor for touch
users by itself — don't rely on it alone for that.

**Re-arm/re-check on every `ON_RESUME`, not just once.** An Activity is not torn
down when merely backgrounded, so a one-shot `LaunchedEffect(Unit)` won't
rerun when the user returns to the app — verify focus behavior still holds
every time a screen returns to the foreground, not just on first appearance.
(iOS mirror: reset VoiceOver focus on every `scenePhase` return to `.active`,
not just on first appear.)

**Headings.** Compose exposes *nothing* as a heading by default — TalkBack's
"jump by heading" navigation control only works on elements explicitly marked
with `Modifier.semantics { heading() }`. If a screen has section labels, mark
them, mirroring iOS `Form` `Section` headers (which VoiceOver's rotor already
understands natively).

**Back navigation.** If a "screen" is really just a Composable swapped in by
local state (not a real back-stack entry), the system back gesture falls
through to the Activity's default behavior — exiting the app — unless you add
an explicit `BackHandler` that returns to the logical parent screen.

## Live-updating values

**Split the stable label from the changing value.** Put the label in
`contentDescription` (read once when focus arrives) and the live value in
`stateDescription` instead of concatenating both into one `contentDescription`
string on every update:

```kotlin
Modifier.clearAndSetSemantics {
    contentDescription = label          // read once, on focus
    stateDescription = accessibilityValue  // re-announced on each live update
    if (announceChanges) liveRegion = LiveRegionMode.Polite
}
```

Without this split, every update re-reads the full "label: value" string,
which is exactly the kind of repetition that makes an app feel unusable with a
screen reader running.

**Don't wire a live region straight to a noisy or fast-changing raw value.**
A value that changes every second (GPS speed, elapsed time) will flicker or
chatter constantly if you announce every change. Two techniques, often
combined:
- Apply a change threshold — only treat an update as "real" once it differs
  from the last-announced value by a meaningful amount (e.g. ~5 km/h, not every
  0.3 km/h GPS jitter).
- For values that update on a fixed short cadence purely for visual/haptic
  purposes (an elapsed-time counter, say), consider *not* giving that specific
  text a live region at all — that's exactly what distinct vibration/earcon
  patterns are for instead (see "Non-visual signal design" below). A live
  region is for meaningful state changes, not a running clock.

A forced "still alive" update after N seconds of no qualifying change can be a
reasonable middle ground for a value the user needs reassurance is still
tracking (e.g. "no GPS signal" vs. "value hasn't moved") — but don't add that
reassurance ping to every value; it can just become more chatter.

## Dark mode isn't optional, and "correct-looking" theme code can still do nothing

Dark mode support belongs in the same category as screen-reader support here:
a platform convention the user actually relies on daily, not a cosmetic
nice-to-have — treat a request for "an app" as implicitly including working
dark mode unless told otherwise.

**`MaterialTheme(colorScheme = ...)` alone paints nothing.** This is a real bug
that shipped and was only caught by screenshotting the app on-device with the
system in dark mode — the code looked completely idiomatic and still failed:

```kotlin
// Looks correct. Compiles. Does nothing visible.
MaterialTheme(colorScheme = if (isSystemInDarkTheme()) darkColorScheme() else lightColorScheme()) {
    content()
}
```

`MaterialTheme` only *publishes* the color scheme via `CompositionLocal` — it
does not draw a background. Without a `Surface` (or `Scaffold`) inside it that
actually reads `MaterialTheme.colorScheme.background` and paints it, the
screen keeps showing whatever background was already there (typically a
light, hardcoded XML window background from the Activity's theme), so the app
looks permanently light regardless of system setting or what color scheme was
actually computed — confirmed by forcing `darkColorScheme()` unconditionally
and seeing zero visual change until the `Surface` wrapper was added. Put the
fix once, centrally, in the shared root theme composable so every future
screen inherits it automatically instead of relying on each screen to
remember it:

```kotlin
@Composable
fun AppTheme(content: @Composable () -> Unit) {
    val colors = if (isSystemInDarkTheme()) DarkColors else LightColors
    MaterialTheme(colorScheme = colors) {
        Surface(modifier = Modifier.fillMaxSize(), color = MaterialTheme.colorScheme.background) {
            // safeDrawingPadding here, not on the Surface itself, so the background still
            // paints edge-to-edge behind the status/nav bars; only the content is inset.
            Box(modifier = Modifier.safeDrawingPadding()) {
                content()
            }
        }
    }
}
```

The `safeDrawingPadding()` half of that isn't a dark-mode issue per se, but it
was found in the same on-device screenshot pass and belongs in the same root
wrapper: content draws edge-to-edge by default on modern Android, so without
it a screen's title sits under the status bar clock/icons.

**When asked to give a specific element a specific color, stop and ask what
happens in the other theme before writing a literal color value.** A request
like "make this button purple" or "make the header dark blue" is really a
request about *one* theme unless the user says otherwise. A hardcoded
`Color(0xFF...)` will not adapt, and can end up low-contrast, jarring, or
outright invisible (e.g. a dark-on-dark button) once the system switches
theme. Before hardcoding a literal color:
- Prefer an existing role from the theme's `ColorScheme` (`primary`,
  `secondary`, `tertiary`, `error`, and their `on*`/`*Container` pairs) over a
  literal value wherever the requested color is close to one of those roles —
  it gets light/dark handling for free.
- If the request is genuinely a custom brand/one-off color that doesn't map to
  an existing role, define *both* a light and a dark value for it (in the same
  place the rest of the palette is defined) rather than one literal color used
  in both themes, and actually check it in both — don't assume a color chosen
  while looking at light mode still has enough contrast against a dark
  background, or vice versa.
- If it's ambiguous which theme the user means, or whether they want it to
  vary by theme at all, ask, rather than silently picking one reading (this is
  the same "surface ambiguous options explicitly" discipline that applies to
  any user request with more than one reasonable interpretation).

## Merged semantics for compound controls

A bare `Switch`/`Checkbox` next to a `Text` label are two separate
accessibility nodes by default — TalkBack announces "switch, off" with no idea
what it's a switch *for*. Fix: wrap the whole row in a modifier that merges
descendant semantics and handles the tap, and make the control itself
`onCheckedChange = null` so it doesn't double-handle the click:

```kotlin
Row(
    modifier = Modifier.toggleable(
        value = checked,
        onValueChange = { ... },
        role = Role.Switch // or Role.Checkbox
    )
) {
    Switch(checked = checked, onCheckedChange = null)
    Text("Keep counting after target time")
}
```

This is an easy one to reintroduce on every new toggle row in every new
screen — it was caught and fixed independently in two different projects.

## Content descriptions must disambiguate repeated items

A row of otherwise-identical icon buttons (e.g. a delete icon on every item in
a list) needs a *per-item* content description, not one generic string reused
everywhere: `"Delete warning 3"`, not just `"Delete"` on every row. A
screen-reader user swiping through the list has no other way to tell which
control they're on.

## Custom actions instead of extra visible controls

To add functionality (skip forward/back, jump to first/last, switch units)
without cluttering a deliberately minimal visible UI, expose it as a platform
accessibility custom action instead of a new button:
- Compose: `customActions = listOf(CustomAccessibilityAction(label) { ... })`
  in a `semantics` block — shows up in TalkBack's local context menu.
- iOS: `UIAccessibilityCustomAction` / SwiftUI `accessibilityAction` — shows up
  in VoiceOver's rotor.

**Long-press is a safe secondary gesture** for the same idea on the visible
control itself: TalkBack/VoiceOver both translate a long-press to
double-tap-and-hold, so "long-press previous/next to jump to first/last" works
for touch and screen-reader users with the same one interaction, no separate
button needed.

## Text input pitfalls (numeric fields especially)

- **`OutlinedTextField`/`BasicTextField` default to multi-line.** Without
  `singleLine = true` and an explicit `imeAction` (e.g. `ImeAction.Done`), the
  keyboard's enter/return key can insert a literal newline into the value
  instead of closing the keyboard. A value like `"1\n"` then silently fails
  `toIntOrNull()`/`toLongOrNull()` and falls back to some default — a warning
  typed as "1 minute" can end up behaving like "0", with no error shown
  anywhere.
- **Selecting a pre-filled example value on focus is unreliable.** The natural
  idea — select-all the text so the first keystroke replaces it — loses a race
  against the tap's own cursor-placement gesture handling, which runs after
  (or interleaved with) the focus-changed callback and silently overrides the
  selection. Typing "2" over a pre-filled "4" can produce "42" instead of "2".
  **Clearing the field to empty on its first focus** (guarded by a
  `remember { mutableStateOf(false) }` "has this been focused before" flag) is
  the version that actually holds up.
- **Don't use `FocusManager.clearFocus()` just to dismiss the keyboard.**
  Clearing focus doesn't leave nothing focused — it can move focus to another
  focusable element (observed: back to the very first field on the whole
  screen). If that element is *also* wired with "clear on first focus" logic,
  you've just silently wiped an unrelated field the user never touched. Use
  `LocalSoftwareKeyboardController.current?.hide()` instead — it dismisses the
  keyboard without touching which element is logically focused.
- **An implicit "blank/zero means X" convention is fragile — prefer an
  explicit control.** A repeat-interval field that silently means "no repeat"
  when left blank or zero is exactly the kind of ambiguity that produces
  wrong behavior nobody notices until it fires wrong, and it's also harder for
  a screen reader to convey than an explicit state. An explicit checkbox
  ("Repeat: on/off", disabling — not hiding — the related field when off) is
  clearer for both sighted and screen-reader use.

## Non-visual signal design (the general version of "reduce chatter")

Whatever the output channel — speech, vibration, an earcon — the same
principle recurs: **don't repeat information the user has already been given
unless the underlying state actually changed.** Concretely:
- Speech: read a value's label once on focus arrival; live updates announce
  only the changed value (see "Live-updating values" above).
- Vibration: if a pattern encodes compound state (e.g. "which minute, which
  sub-marker"), only re-encode the part that changed since the *previous*
  signal, not the whole state every time — repeating a long "which minute"
  vibration on every 15-second tick within the same minute is exactly the
  vibration equivalent of TalkBack repeating a label on every value update.
- Give the *final*/terminal event in a sequence (target reached, recording
  stopped) a distinctly different signal so it doesn't need to be decoded the
  same way as an ordinary tick.

## Speak what a screen reader should actually say

A visual abbreviation isn't necessarily a valid word for text-to-speech: "km/u"
read aloud by a screen reader isn't a recognized unit. If a value will be
read aloud, provide the spoken form explicitly ("kilometer per uur") separately
from what's shown on screen, rather than assuming the abbreviation is fine
either way.

Watch for this in field *labels* too, not just live values — a label like
"Every (s)" for a seconds-interval input is exactly the kind of thing this
project's own guidance warns against, and it still slipped through once
(shipped, then had to be corrected to "Every (seconds)" after the fact).
A single stray letter in parentheses is easy to miss during review because it
reads fine *visually* — the whole point is that it doesn't read fine *aloud*,
which is invisible unless you specifically ask "what would this sound like
spoken?" for every short label, not just the obviously-abbreviated ones.

**A duration formatted as "MM:SS" gets misread as a clock time, not a
duration.** This is a distinct, easy-to-miss case of the same general
principle, found live on a real device: a count-up timer showing "16:00"
elapsed got read aloud as "four o'clock", and "15:52" as "8 minutes to four"
— technically a correct reading of that *string shape*, completely wrong for
what it *means*. Any TTS engine's own text-normalization actively looks for
"H:MM"/"HH:MM"-shaped text and converts it to a spoken clock-time reading;
there is no way to opt out of that heuristic by formatting the string
differently, only by not handing it the raw "MM:SS" text at all. Fix: give
the element an explicit spoken-form `contentDescription` that spells the
duration out in words ("15 minutes and 52 seconds", or just "16 minutes"
when the seconds component is exactly zero — don't say "and 0 seconds"),
independent of what's shown on screen:

```kotlin
Text(
    text = "15:52",                                     // what's shown
    modifier = Modifier.semantics { contentDescription = "15 minutes and 52 seconds" } // what's said
)
```

Use real plurals (`pluralStringResource`/`<plurals>`, not a hand-rolled
`if (n == 1)`) for the minutes/seconds words — "1 minutes" is exactly the kind
of small grammatical wrongness that makes an app sound unpolished read aloud
even though it's invisible in a visual review. Note this has to be computed
*before* entering a `Modifier.semantics { }` block, not inside it — that
lambda's receiver (`SemanticsPropertyReceiver`) isn't a composable context, so
`pluralStringResource`/`stringResource` calls have to happen at the normal
composable call site first and get captured as a plain `String`.

This same MM:SS trap applies to any duration/elapsed-time display, on any
platform — it's specific to the *visual shape* of the text a TTS engine sees,
not to Compose or Android particularly.

## Layout resilience at large accessibility font sizes

Large text scaling (a real, commonly-used accessibility setting, not an edge
case) will wrap unbounded text across multiple lines. In a fixed-height
container like a top app bar, that can grow the bar tall enough to push the
*entire scrollable body* off-screen with no way to reach it. Cap chrome/label
text that must stay compact to `maxLines = 1` with `TextOverflow.Ellipsis`, and
actually test the screen with a large font-scale setting enabled — this class
of bug is invisible at default text size.

## Don't fight TalkBack for the audio stream

If the app plays its own tones/audio, don't route it through
`USAGE_ASSISTANCE_ACCESSIBILITY` / `STREAM_ACCESSIBILITY` unless you actually
want it tied to TalkBack's own speech volume — hardware volume keys and
independent volume control silently stop working as expected otherwise. Use
`STREAM_MUSIC` (or whatever's actually appropriate) and route volume keys to
match it explicitly.

## Reliability underpins accessibility for a non-visual app

For an app whose entire value proposition is a non-visual/non-auditory signal
(e.g. vibration) firing at the right time regardless of screen state, exact
timing reliability *is* an accessibility requirement, not just a nice-to-have.
On Android, `AlarmManager.setAlarmClock()` needs the app to declare
`USE_EXACT_ALARM` (a declare-only permission for apps whose core function is
timers/alarms) or it throws at runtime with no warning at compile time — check
this explicitly for anything that must fire while the screen is off or locked.

## Verification discipline

- **A successful build is not verification of runtime behavior** — permission
  flows, layout at runtime, and whether a feature actually fires all need an
  actual run. This generalizes past accessibility: treat "it compiles" as
  necessary, never sufficient, for anything UI- or runtime-facing.
- If no device is connected, check for an available emulator/AVD
  (`emulator -list-avds`) before treating "no connected device" as a stopping
  condition — a project may already have a purpose-built test AVD sitting
  unused (e.g. a `talkback-test` AVD with TalkBack pre-configured).
- `adb shell uiautomator dump` (or similar accessibility-tree introspection) is
  a useful first pass for checking content descriptions and merged semantics,
  but it has real, known blind spots for how Compose's merged-semantics nodes
  get represented — it is not a substitute for an actual TalkBack pass, only a
  cheap early filter.
- Re-verify accessibility-sensitive Compose internals after any Compose BOM
  bump — some of these patterns (like `traversalIndex` governing TalkBack's
  default-focus choice) are observed behavior, not a documented API contract,
  and can change silently. Prefer documented semantics properties
  (`traversalIndex`, `heading()`, `liveRegion`, ...) over reaching into
  `AndroidComposeView`'s internal View-level `AccessibilityDelegate` wiring —
  the latter was tried for initial-focus control in Uptimer and confirmed not
  to work at all, likely because Compose's virtual accessibility nodes don't
  route through the real-View event-bubbling chain that hook expects.

## Quick pre-ship checklist

- [ ] The root theme composable wraps content in a `Surface` that reads
      `MaterialTheme.colorScheme.background` (not just `MaterialTheme(colorScheme = ...) { content() }`),
      plus `safeDrawingPadding()` around the content.
- [ ] Verified on-device (a screenshot is enough) that the app actually
      renders dark when the system is in dark mode — not just that the code
      looks right.
- [ ] Any element given a specific literal color was checked against "what
      does this look like in the other theme" — either it's a theme role
      (`primary`, `error`, etc.) or it has explicit light *and* dark values.
- [ ] Every toggle (`Switch`/`Checkbox`) row has merged semantics via
      `Modifier.toggleable`/`selectable`, not a bare control beside a `Text`.
- [ ] Every repeated icon/button in a list has a per-item, disambiguating
      content description.
- [ ] Numeric/text fields: `singleLine = true`, explicit `imeAction`, and
      "clear on first focus" (not select-on-focus) for pre-filled examples.
- [ ] Nothing dismisses the keyboard via `FocusManager.clearFocus()`.
- [ ] Section labels are marked as headings; the initial and post-resume
      accessibility focus target is deliberate (via `traversalIndex` on the
      relevant siblings, not a `View.AccessibilityDelegate` hack), not
      whatever TalkBack defaults to.
- [ ] Live-updating text splits label (`contentDescription`) from value
      (`stateDescription`), and fast/noisy values are thresholded or
      deliberately excluded from a live region.
- [ ] Every short field/button label was read back mentally *as speech*, not
      just visually proofread — no bare-letter unit abbreviations like "(s)".
- [ ] Any "MM:SS"-shaped duration/elapsed-time text has an explicit
      spelled-out `contentDescription` ("15 minutes and 52 seconds"), so TTS
      normalization can't misread it as a clock time.
- [ ] Any custom app audio uses a stream independent of
      `STREAM_ACCESSIBILITY`.
- [ ] Screen tested at a large accessibility font-size setting.
- [ ] Screen actually tested with TalkBack/VoiceOver turned on, on a real
      device — not just compiled, not just checked via `uiautomator dump`.
