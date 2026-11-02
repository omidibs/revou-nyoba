# Implementation Plan: Cat Petting Clicker

## Overview

The Cat Petting Clicker is a zero-build, browser-only single-page app built with HTML5, Tailwind CSS (Play CDN), and Vanilla JavaScript ES modules. The implementation is sequenced to validate pure logic first with property-based tests (fast-check under `node --test`), then layer on the DOM/audio side-effectful modules, and finally wire everything together in a thin event-driven controller. Pure functions (Reaction Engine mapping, bounded random generators, counter math) are dependency-free and testable without a DOM; impure modules (SoundManager, ReactionRenderer) depend only on browser APIs and are tested with jsdom, fake timers, and a mocked `HTMLAudioElement`.

## Tasks

- [ ] 1. Set up project structure and test infrastructure
  - [-] 1.1 Create the project directory structure
    - Create `js/` for ES modules, `assets/sounds/` for audio cues, and `tests/` for property and unit/integration tests
    - _Requirements: (infrastructure)_
  - [-] 1.2 Configure package.json and test tooling
    - Create `package.json` with `"type": "module"` for ES module support
    - Add `fast-check` as a dev dependency for property-based tests
    - Add `jsdom` as a dev dependency for DOM/impure module tests
    - Add an npm `test` script running `node --test`
    - _Requirements: (infrastructure)_

- [ ] 2. Implement the reaction engine and its property tests
  - [~] 2.1 Implement `js/reaction-engine.js`
    - Export the `ReactionType` enum (`happy`, `purring`, `sleeping`, `zoomies`)
    - Export `REACTION_RANGES` mapping each reaction to its inclusive `[min, max]` range: Happy `[1,40]`, Purring `[41,70]`, Sleeping `[71,90]`, Zoomies `[91,100]`
    - Implement `selectReaction(percentage)` returning exactly one reaction for any integer in `[1,100]`, returning `null` (never throwing) for out-of-range or non-integer inputs
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6_
  - [~] 2.2 Write property test for reaction partition
    - Create `tests/reaction-partition.property.test.js`
    - **Property 1: Reaction partition is complete and disjoint**
    - **Validates: Requirements 3.1, 3.2, 3.3, 3.4, 3.5**
    - Generate random integers `p` in `1..100`; assert `selectReaction(p)` returns exactly one reaction and the implied ranges are disjoint, contiguous, and partition `1..100` with no gaps or overlaps
  - [~] 2.3 Write property test for out-of-range no-crash
    - Create `tests/reaction-out-of-range.property.test.js`
    - **Property 2: Out-of-range inputs do not crash**
    - **Validates: Requirement 3.6**
    - Generate invalid `p` (negatives, `>100`, floats, `NaN`, non-numbers); assert `selectReaction(p)` returns `null` and never throws

- [ ] 3. Implement the random generators and their property tests
  - [~] 3.1 Implement `js/random.js`
    - `nextPercentage(random)` -> integer in `[1,100]` via `1 + floor(random() * 100)`
    - `nextPettingVolume(random)` -> number in `[0.2, 0.5]` via `0.2 + random() * 0.3`
    - `nextPettingCue(random)` -> `'pop'` | `'meowy'`
    - `nextZoomiesCue(random)` -> `'zoom'` | `'boing'`
    - `nextZzzCount(random)` -> integer in `[1,3]` via `1 + floor(random() * 3)`
    - `nextSpeedLinePositions(rect, count, random)` -> `count` positions whose centers lie within `100 px` of the Cat bounding box (pure helper used by the renderer and Property 7)
    - All functions accept an optional `random` source defaulting to `Math.random` for deterministic/property-based tests
    - _Requirements: 2.1, 2.2, 6.3, 7.2, 7.3_
  - [~] 3.2 Write property test for percentage range
    - Create `tests/percentage-range.property.test.js`
    - **Property 3: Generated percentage is always in range**
    - **Validates: Requirement 2.1**
    - Call `nextPercentage()` repeatedly (with an injected random source); assert each result is an integer in `[1,100]`
  - [~] 3.3 Write property test for petting volume range
    - Create `tests/petting-volume.property.test.js`
    - **Property 4: Petting sound volume is always in the 20-50% band**
    - **Validates: Requirement 2.2**
    - Call `nextPettingVolume()` repeatedly; assert each result is in `[0.2, 0.5]`
  - [~] 3.4 Write property test for Zzz count bounds
    - Create `tests/zzz-count.property.test.js`
    - **Property 6: Sleeping Zzz count is bounded**
    - **Validates: Requirement 6.3**
    - Generate Sleeping Zzz counts; assert the count is an integer in `[1,3]`
  - [~] 3.5 Write property test for speed-line proximity
    - Create `tests/speed-line-proximity.property.test.js`
    - **Property 7: Zoomies speed-lines stay within proximity**
    - **Validates: Requirement 7.2**
    - Generate random Cat bounding boxes and offsets; assert each placed speed-line center lies within `100 px` of the box and at least `4` speed-lines are produced

- [ ] 4. Implement the pet counter and its property test
  - [~] 4.1 Implement `js/pet-counter.js`
    - Hold `count` state as a non-negative integer starting at `0`
    - `getCount()` returns the current count
    - `increment()` adds 1, re-renders, and returns the new count
    - `render()` sets the counter element text to `` `Total Pets: ${count} ⭐` ``
    - _Requirements: 2.3, 9.1, 9.2_
  - [~] 4.2 Write property test for counter increment and display
    - Create `tests/pet-counter.property.test.js`
    - **Property 5: Counter increment updates count and display**
    - **Validates: Requirements 2.3, 9.1, 9.2**
    - Generate random non-negative `n`; after `increment()` assert the count is `n + 1` and the rendered text equals `` `Total Pets: ${n + 1} ⭐` ``

- [~] 5. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Create the HTML page with Tailwind config and fonts
  - [~] 6.1 Build `index.html` structure, Tailwind config, and Google Fonts
    - Page shell: pastel gradient background transitioning `#FFDEED` (top) to `#B5FFFC` (bottom); aspect-square Cat container keeping the Cat centered and occupying `>= 50%` of its width/height; bubble-style counters using `bg-white/30`, `backdrop-blur-lg`, `rounded-full`, `border-white/40`; overlay containers above the Cat
    - Tailwind Play CDN with inline `tailwind.config`: `fontFamily.sans = ['Quicksand', 'Varela Round', 'sans-serif']`; `wiggle` keyframe with `>= 2` stops producing a `10-20 px` horizontal `translateX` shake; `zzzFade` keyframe with `>= 2` stops fading `Zzz...` in; `animation.wiggle = 'wiggle 0.1s linear infinite'` (50 ms half-cycle) and `animation.zzzFade = 'zzzFade 1.5s ease-in-out forwards'`
    - Google Fonts links for Quicksand and Varela Round; load `js/app.js` as `type="module"`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 8.1, 8.2_

- [ ] 7. Implement the sound manager and its unit tests
  - [~] 7.1 Implement `js/sound-manager.js`
    - `play(cue, { volume, loop })` constructs `new Audio('assets/sounds/<cue>.mp3')`, clamps volume to `[0,1]`, sets `loop`, calls `play()` with an attached `.catch()` that swallows rejections, and returns the element or `null` on construction/load error
    - `playPettingSound()` plays `nextPettingCue()` at `nextPettingVolume()` (20-50%)
    - `playReactionSound(reaction)` — Purring: `play('purr', { volume: 0.5, loop: true })`; Zoomies: `play(nextZoomiesCue(), { volume: 0.5 })`
    - `stop(audio)` pauses and resets `currentTime` to `0`, swallowing errors
    - _Requirements: 2.2, 2.4, 5.2, 5.3, 7.3, 7.4_
  - [~] 7.2 Write unit tests for the sound manager
    - Create `tests/sound-manager.test.js`
    - Mock `HTMLAudioElement`; assert volume is clamped to `[0,1]`, rejections are swallowed, and `null` is returned on construction error
    - Assert `playPettingSound()` picks `pop`/`meowy` at a 20-50% volume
    - Assert `playReactionSound('purring')` plays `purr.mp3` at `0.5` with `loop = true`, and `playReactionSound('zoomies')` plays one of `zoom`/`boing` at `0.5`
    - Assert `stop()` pauses and resets `currentTime`
    - Assert a rejecting `play()` does not interrupt an in-progress pet action flow
    - _Requirements: 2.4, 5.2, 5.3, 7.3, 7.4_

- [ ] 8. Implement the reaction renderer and its unit tests
  - [~] 8.1 Implement `js/reaction-renderer.js`
    - Maintain an `active` handle `{ type, timers, audios, elements }` (or `null`)
    - `apply(reactionType)`: call `clear()` first, then branch — Happy adds `animate-bounce` plus a `YAY!😊 (+5 Aura)` text overlay and schedules removal `>= 2000 ms`; Purring adds `animate-pulse`, plays `purr` (loop), and schedules cleanup at `2000 ms`; Sleeping adds `scale-90 rotate-6` plus `nextZzzCount()` `Zzz...` emojis with `animate-zzz-fade`; Zoomies adds `animate-wiggle` plus `>= 4` speed-line emojis within `100 px` (via `nextSpeedLinePositions`), plays `zoom`/`boing`, and schedules cleanup in `[2000, 3000] ms`
    - `clear()`: clears all timers, removes reaction-specific classes/elements, stops all audios, and sets `active = null`
    - _Requirements: 4.1, 4.2, 4.3, 5.1, 5.2, 6.1, 6.2, 6.3, 7.1, 7.2, 7.3, 7.4_
  - [~] 8.2 Write unit tests for the reaction renderer
    - Create `tests/reaction-renderer.test.js`
    - Happy: assert `animate-bounce` and overlay text `YAY!😊 (+5 Aura)`; advance fake timers `>= 2000 ms`; assert both removed; assert interrupting with another reaction clears the bounce and overlay
    - Purring: assert `animate-pulse` and `purr` plays at `0.5` with `loop = true`; advance `2000 ms`; assert class removed and audio stopped; repeat with a rejecting audio mock to confirm the animation still runs
    - Sleeping: assert `scale-90` and `rotate-6` on the Cat; assert Zzz elements carry `animate-zzz-fade`
    - Zoomies: assert `animate-wiggle` applied, `>= 4` speed-lines within `100 px`, and one of `zoom`/`boing` plays; repeat with a rejecting audio mock to confirm animation and speed-lines still render; advance `2-3 s`; assert cleanup
    - _Requirements: 4.1, 4.2, 4.3, 5.1, 5.2, 5.3, 6.1, 6.2, 6.3, 7.1, 7.2, 7.3, 7.4_

- [ ] 9. Implement the app controller and wire the modules together
  - [~] 9.1 Implement `js/app.js`
    - `init()`: query DOM elements (Cat, counter, overlay containers), attach the Cat click listener, render the initial counter `Total Pets: 0 ⭐`
    - `handlePetAction()`: generate `nextPercentage()`, call `selectReaction` (return early if `null`), `petCounter.increment()`, `soundManager.playPettingSound()` (fire-and-forget), then `reactionRenderer.apply(reaction)`
    - Initialize the controller on DOMContentLoaded
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 9.1_

- [ ] 10. Write integration and UI rendering tests
  - [~] 10.1 Write integration tests for the pet action flow
    - Create `tests/pet-action.integration.test.js`
    - Use jsdom plus fake timers; simulate a Cat click and assert the counter increments, a reaction is rendered, and a petting sound fires (fire-and-forget)
    - Assert rapid clicks interrupt the previous reaction cleanly (no orphaned overlays)
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 4.3_
  - [~] 10.2 Write UI rendering tests for the visual requirements
    - Create `tests/ui-rendering.test.js`
    - Assert the Cat container is `aspect-square` and the Cat occupies `>= 50%` of it
    - Assert the background gradient resolves to `#FFDEED -> #B5FFFC`
    - Assert counter elements carry `bg-white/30`, `backdrop-blur-lg`, `rounded-full`, `border-white/40`
    - Assert the loaded font is Quicksand or Varela Round
    - Assert `tailwind.config` registers `wiggle` and `zzzFade` keyframes with `>= 2` stops each, and that Zoomies applies `animate-wiggle` and Sleeping applies `animate-zzz-fade`
    - Assert the initial counter text is `Total Pets: 0 ⭐`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 8.1, 8.2, 8.3, 8.4, 9.1_

- [~] 11. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation: pure logic is validated with property tests before any DOM code is written
- Property tests validate universal correctness properties of the pure logic; unit/integration tests validate rendering, timing, and side effects
- Each property test lives in its own file under `tests/` so property tests can run in parallel without file-write conflicts
- Audio asset files (`pop.mp3`, `meowy.mp3`, `purr.mp3`, `zoom.mp3`, `boing.mp3`) must be placed in `assets/sounds/` for runtime playback; the SoundManager degrades gracefully when files are missing, so their absence does not block development or tests
- Running the app requires a static server (e.g., `python -m http.server` or VS Code Live Server) because ES modules and audio assets are fetched via HTTP; run this manually in a terminal

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2"] },
    { "id": 1, "tasks": ["2.1", "3.1", "4.1", "6.1"] },
    { "id": 2, "tasks": ["2.2", "2.3", "3.2", "3.3", "3.4", "3.5", "4.2", "7.1"] },
    { "id": 3, "tasks": ["7.2", "8.1"] },
    { "id": 4, "tasks": ["8.2", "9.1"] },
    { "id": 5, "tasks": ["10.1", "10.2"] }
  ]
}
```
