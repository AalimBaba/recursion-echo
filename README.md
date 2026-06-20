# RECURSION // ECHO

## Project Overview & Concept

**RECURSION // ECHO** is a five-stage temporal puzzle-action survival game built as a single-file HTML5 application. The player controls a responsive green neon cursor inside a cyberpunk vector grid, collects three glowing data shards, and survives autonomous firewall drones during a strict 10.00 second loop. When the timer expires and all three shards have been collected, the timeline rewinds: the player returns to the bottom-center origin point, and the completed run is instantiated as a translucent cyan **Echo Instance** that replays the player's exact historical coordinate path.

The central design tension is that every success increases the spatial complexity of the next run. The player is not only navigating around hazards, but also around their own past state. By loop five, the playfield contains multiple simultaneous historical trajectories, forcing the player to solve a real-time path planning problem under time pressure.

The core technical problem solved by the project is **historical-state spatial navigation**. Each loop records the live player's world-space position every animation frame into a runtime sequence array. On subsequent loops, those serialized coordinate samples are replayed frame-by-frame as active collision participants. The player must route around the current hazard field while avoiding any Euclidean intersection with prior self-state. A collision with a drone or Echo Instance triggers a timeline-collapse exception with a diagnostic reason.

## Technical Stack & Architecture

### Stack

- **Pure ECMAScript 6+**: The entire simulation, state machine, input layer, deterministic playback system, collision detection, and renderer are implemented with vanilla JavaScript.
- **HTML5 Canvas API**: All entities are rendered procedurally with vector primitives, including the player cursor, Echo Instances, hazard drones, data shards, particles, origin marker, and grid matrix.
- **Custom Responsive CSS3**: The HUD, modal overlay, footer telemetry, responsive playfield, neon styling, and visual scanline treatment are implemented with embedded CSS.
- **Zero third-party dependencies**: No engines, frameworks, libraries, images, audio files, fonts, or external CDNs are used. The project is ready for GitHub Pages deployment as a single `index.html` file.

### Command/Playback Pattern

The game implements a coordinate-based Command/Playback Pattern. Instead of recording abstract button presses, the simulation serializes the resolved world-space transformation of the player after input integration:

```js
currentHistory.push({
  x: player.x,
  y: player.y
});
```

That frame-by-frame runtime sequence becomes the authoritative playback command stream for an Echo Instance. When a loop completes successfully, the active history array is copied into the echo stack:

```js
echoes.push(currentHistory.map((p) => ({ x: p.x, y: p.y })));
```

During later loops, the engine selects the echo sample matching the current frame index. This produces exact historical replay in the same coordinate space as the live simulation, making paradox collision checks deterministic and inspectable.

### Deterministic Time-Delta Game Loop

The runtime uses `requestAnimationFrame` and a capped time delta to decouple gameplay physics from monitor refresh rate. Movement, drone velocity, visual particles, animation phase, and loop timing all advance using `dt` in seconds:

```js
const dt = Math.min(0.05, Math.max(0, (stamp - lastStamp) / 1000));
update(dt);
draw();
```

This architecture prevents high-refresh displays from accelerating gameplay and prevents a background-tab resume from creating a single large physics jump. The result is a uniform simulation model across common 60 Hz, 120 Hz, 144 Hz, and variable-refresh displays.

### Euclidean Distance Matrix

All gameplay interaction uses precise Euclidean distance checks:

```js
const dx = a.x - b.x;
const dy = a.y - b.y;
return Math.sqrt(dx * dx + dy * dy);
```

The collision matrix is intentionally O(N), where N is the active count of drones, shards, and echo histories. This is appropriate for the designed entity count and keeps the implementation straightforward, deterministic, and easy to audit. Each frame evaluates:

- Player versus firewall drones.
- Player versus current-frame Echo Instance positions.
- Player versus uncollected data shards.

A paradox occurs when the live player radius overlaps an Echo Instance radius. The failure message identifies the echo loop number and frame index, turning collision detection into a visible diagnostic system rather than a black-box fail state.

## Gameplay & Interface Preview

![Main Gameplay State](placeholder_link)

**Description:** Showing the active player cursor, running vector grid, and autonomous hazard drones.

![Temporal Paradox Collision Screen](placeholder_link)

**Description:** Showing the Timeline Collapse game-over matrix indicating a spatial collision with a past loop instance.

![Mission Success Decryption Core](placeholder_link)

**Description:** Showing the termination panel after surviving all 5 cascading historical timelines.

## System Implementation & Production Code

The canonical deployable source is `index.html`. The following annotated listing contains the complete HTML, CSS, and JavaScript implementation. Every non-empty source line is followed by a direct explanatory comment for engineering review, portfolio presentation, and recruiter-facing code walkthroughs.

```html
<!DOCTYPE html>
<!-- Line 1: Declares standards-mode HTML so the browser uses modern layout, canvas, and scripting behavior. -->
<html lang="en">
<!-- Line 2: Opens the root document element and sets the page language for accessibility and browser metadata. -->
<head>
<!-- Line 3: Starts the metadata section that configures encoding, viewport behavior, title, and embedded styles. -->
  <meta charset="UTF-8">
  <!-- Line 4: Forces UTF-8 decoding so all source text, labels, and CSS tokens parse consistently. -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <!-- Line 5: Locks the viewport to responsive device dimensions and prevents zoom drift during pointer play. -->
  <title>RECURSION // ECHO</title>
  <!-- Line 6: Sets the browser tab title to the game name for GitHub Pages and portfolio presentation. -->
  <style>
  /* Line 7: Begins the embedded CSS block, preserving the single-file deployment constraint. */
    :root {
    /* Line 8: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      --bg: #03040b;
      /* Line 9: Defines a reusable custom property for the neon visual palette used across the interface. */
      --panel: rgba(3, 12, 18, 0.72);
      /* Line 10: Defines a reusable custom property for the neon visual palette used across the interface. */
      --green: #33ff33;
      /* Line 11: Defines a reusable custom property for the neon visual palette used across the interface. */
      --red: #ff3366;
      /* Line 12: Defines a reusable custom property for the neon visual palette used across the interface. */
      --cyan: #00ffff;
      /* Line 13: Defines a reusable custom property for the neon visual palette used across the interface. */
      --amber: #ffec66;
      /* Line 14: Defines a reusable custom property for the neon visual palette used across the interface. */
      --muted: #7df8ff;
      /* Line 15: Defines a reusable custom property for the neon visual palette used across the interface. */
    }
    /* Line 16: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    * { box-sizing: border-box; }
    /* Line 18: Draws a crisp vector-style boundary that matches the terminal instrumentation theme. */

    html,
    /* Line 20: Contributes a focused visual or layout rule to the single-file responsive interface. */
    body {
    /* Line 21: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      width: 100%;
      /* Line 22: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      height: 100%;
      /* Line 23: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      margin: 0;
      /* Line 24: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      overflow: hidden;
      /* Line 25: Controls clipping behavior to prevent scrollbars or visual spill from disrupting play. */
      background: radial-gradient(circle at 50% 45%, #071329 0%, #03040b 52%, #000 100%);
      /* Line 26: Applies procedural color, gradient, or panel treatment without using external image assets. */
      color: var(--green);
      /* Line 27: Defines a reusable custom property for the neon visual palette used across the interface. */
      font-family: "Courier New", Courier, monospace;
      /* Line 28: Sets typography to a monospace terminal style consistent with the game fiction. */
      letter-spacing: 0;
      /* Line 29: Contributes a focused visual or layout rule to the single-file responsive interface. */
      touch-action: none;
      /* Line 30: Contributes a focused visual or layout rule to the single-file responsive interface. */
      user-select: none;
      /* Line 31: Contributes a focused visual or layout rule to the single-file responsive interface. */
    }
    /* Line 32: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    body::before {
    /* Line 34: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      content: "";
      /* Line 35: Contributes a focused visual or layout rule to the single-file responsive interface. */
      position: fixed;
      /* Line 36: Sets positioning behavior so overlays, pseudo-elements, or containers align predictably. */
      inset: 0;
      /* Line 37: Contributes a focused visual or layout rule to the single-file responsive interface. */
      pointer-events: none;
      /* Line 38: Contributes a focused visual or layout rule to the single-file responsive interface. */
      background:
      /* Line 39: Applies procedural color, gradient, or panel treatment without using external image assets. */
        linear-gradient(rgba(51, 255, 51, 0.035) 50%, rgba(0, 0, 0, 0.05) 50%),
        /* Line 40: Contributes a focused visual or layout rule to the single-file responsive interface. */
        linear-gradient(90deg, rgba(255, 51, 102, 0.015), rgba(0, 255, 255, 0.018), rgba(51, 255, 51, 0.015));
        /* Line 41: Contributes a focused visual or layout rule to the single-file responsive interface. */
      background-size: 100% 4px, 7px 100%;
      /* Line 42: Applies procedural color, gradient, or panel treatment without using external image assets. */
      mix-blend-mode: screen;
      /* Line 43: Contributes a focused visual or layout rule to the single-file responsive interface. */
      opacity: 0.55;
      /* Line 44: Contributes a focused visual or layout rule to the single-file responsive interface. */
      z-index: 4;
      /* Line 45: Contributes a focused visual or layout rule to the single-file responsive interface. */
    }
    /* Line 46: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .shell {
    /* Line 48: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      width: 100vw;
      /* Line 49: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      height: 100vh;
      /* Line 50: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      display: grid;
      /* Line 51: Uses CSS Grid to create deterministic HUD or shell layout tracks. */
      grid-template-rows: auto minmax(0, 1fr) auto;
      /* Line 52: Contributes a focused visual or layout rule to the single-file responsive interface. */
      gap: 10px;
      /* Line 53: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      padding: clamp(10px, 2.4vmin, 22px);
      /* Line 54: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
    }
    /* Line 55: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .hud,
    /* Line 57: Contributes a focused visual or layout rule to the single-file responsive interface. */
    .footer {
    /* Line 58: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      width: min(980px, 100%);
      /* Line 59: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      margin: 0 auto;
      /* Line 60: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      border: 1px solid rgba(0, 255, 255, 0.35);
      /* Line 61: Draws a crisp vector-style boundary that matches the terminal instrumentation theme. */
      background: var(--panel);
      /* Line 62: Defines a reusable custom property for the neon visual palette used across the interface. */
      box-shadow: 0 0 24px rgba(0, 255, 255, 0.11), inset 0 0 18px rgba(51, 255, 51, 0.035);
      /* Line 63: Adds neon glow treatment in CSS to reinforce the cyberpunk terminal aesthetic. */
      backdrop-filter: blur(8px);
      /* Line 64: Contributes a focused visual or layout rule to the single-file responsive interface. */
    }
    /* Line 65: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .hud {
    /* Line 67: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      display: grid;
      /* Line 68: Uses CSS Grid to create deterministic HUD or shell layout tracks. */
      grid-template-columns: repeat(4, 1fr);
      /* Line 69: Contributes a focused visual or layout rule to the single-file responsive interface. */
      gap: 1px;
      /* Line 70: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      min-height: 68px;
      /* Line 71: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
    }
    /* Line 72: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .cell {
    /* Line 74: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      padding: 12px clamp(9px, 2vw, 18px);
      /* Line 75: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      border-right: 1px solid rgba(0, 255, 255, 0.18);
      /* Line 76: Draws a crisp vector-style boundary that matches the terminal instrumentation theme. */
      min-width: 0;
      /* Line 77: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
    }
    /* Line 78: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .cell:last-child { border-right: 0; }
    /* Line 80: Draws a crisp vector-style boundary that matches the terminal instrumentation theme. */

    .label {
    /* Line 82: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      color: rgba(125, 248, 255, 0.72);
      /* Line 83: Applies a palette color that distinguishes player, warning, telemetry, or interface text. */
      font-size: 11px;
      /* Line 84: Sets typography to a monospace terminal style consistent with the game fiction. */
      line-height: 1.2;
      /* Line 85: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
    }
    /* Line 86: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .value {
    /* Line 88: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      margin-top: 4px;
      /* Line 89: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      color: var(--green);
      /* Line 90: Defines a reusable custom property for the neon visual palette used across the interface. */
      font-size: clamp(16px, 2.8vmin, 24px);
      /* Line 91: Sets typography to a monospace terminal style consistent with the game fiction. */
      font-weight: 700;
      /* Line 92: Sets typography to a monospace terminal style consistent with the game fiction. */
      text-shadow: 0 0 9px rgba(51, 255, 51, 0.72);
      /* Line 93: Adds neon glow treatment in CSS to reinforce the cyberpunk terminal aesthetic. */
      white-space: nowrap;
      /* Line 94: Contributes a focused visual or layout rule to the single-file responsive interface. */
      overflow: hidden;
      /* Line 95: Controls clipping behavior to prevent scrollbars or visual spill from disrupting play. */
      text-overflow: ellipsis;
      /* Line 96: Controls clipping behavior to prevent scrollbars or visual spill from disrupting play. */
    }
    /* Line 97: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .value.warn { color: var(--red); text-shadow: 0 0 9px rgba(255, 51, 102, 0.72); }
    /* Line 99: Defines a reusable custom property for the neon visual palette used across the interface. */
    .value.cyan { color: var(--cyan); text-shadow: 0 0 9px rgba(0, 255, 255, 0.72); }
    /* Line 100: Defines a reusable custom property for the neon visual palette used across the interface. */

    .stage-wrap {
    /* Line 102: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      position: relative;
      /* Line 103: Sets positioning behavior so overlays, pseudo-elements, or containers align predictably. */
      width: min(92vmin, calc(100vw - 20px), 760px);
      /* Line 104: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      aspect-ratio: 1;
      /* Line 105: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      align-self: center;
      /* Line 106: Contributes a focused visual or layout rule to the single-file responsive interface. */
      justify-self: center;
      /* Line 107: Contributes a focused visual or layout rule to the single-file responsive interface. */
      border: 1px solid rgba(51, 255, 51, 0.82);
      /* Line 108: Draws a crisp vector-style boundary that matches the terminal instrumentation theme. */
      background: #040713;
      /* Line 109: Applies procedural color, gradient, or panel treatment without using external image assets. */
      box-shadow:
      /* Line 110: Adds neon glow treatment in CSS to reinforce the cyberpunk terminal aesthetic. */
        0 0 38px rgba(51, 255, 51, 0.18),
        /* Line 111: Contributes a focused visual or layout rule to the single-file responsive interface. */
        0 0 62px rgba(0, 255, 255, 0.1),
        /* Line 112: Contributes a focused visual or layout rule to the single-file responsive interface. */
        inset 0 0 42px rgba(0, 255, 255, 0.08);
        /* Line 113: Contributes a focused visual or layout rule to the single-file responsive interface. */
    }
    /* Line 114: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    canvas {
    /* Line 116: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      display: block;
      /* Line 117: Contributes a focused visual or layout rule to the single-file responsive interface. */
      width: 100%;
      /* Line 118: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      height: 100%;
      /* Line 119: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      image-rendering: auto;
      /* Line 120: Contributes a focused visual or layout rule to the single-file responsive interface. */
    }
    /* Line 121: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .overlay {
    /* Line 123: Declares the modal overlay used for the title, failure, and mission-success states. */
      position: absolute;
      /* Line 124: Sets positioning behavior so overlays, pseudo-elements, or containers align predictably. */
      inset: 0;
      /* Line 125: Contributes a focused visual or layout rule to the single-file responsive interface. */
      display: none;
      /* Line 126: Contributes a focused visual or layout rule to the single-file responsive interface. */
      place-items: center;
      /* Line 127: Contributes a focused visual or layout rule to the single-file responsive interface. */
      padding: clamp(18px, 5vmin, 46px);
      /* Line 128: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      background: rgba(0, 2, 8, 0.88);
      /* Line 129: Applies procedural color, gradient, or panel treatment without using external image assets. */
      z-index: 3;
      /* Line 130: Contributes a focused visual or layout rule to the single-file responsive interface. */
    }
    /* Line 131: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .overlay.is-active { display: grid; }
    /* Line 133: Declares the modal overlay used for the title, failure, and mission-success states. */

    .modal {
    /* Line 135: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      width: min(560px, 100%);
      /* Line 136: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      border: 1px solid currentColor;
      /* Line 137: Draws a crisp vector-style boundary that matches the terminal instrumentation theme. */
      color: var(--cyan);
      /* Line 138: Defines a reusable custom property for the neon visual palette used across the interface. */
      background: rgba(1, 9, 15, 0.93);
      /* Line 139: Applies procedural color, gradient, or panel treatment without using external image assets. */
      padding: clamp(18px, 4vmin, 34px);
      /* Line 140: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      text-align: center;
      /* Line 141: Contributes a focused visual or layout rule to the single-file responsive interface. */
      box-shadow: 0 0 32px rgba(0, 255, 255, 0.22), inset 0 0 22px rgba(0, 255, 255, 0.08);
      /* Line 142: Adds neon glow treatment in CSS to reinforce the cyberpunk terminal aesthetic. */
    }
    /* Line 143: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .modal h1 {
    /* Line 145: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      margin: 0 0 12px;
      /* Line 146: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      color: var(--red);
      /* Line 147: Defines a reusable custom property for the neon visual palette used across the interface. */
      font-size: clamp(24px, 6vmin, 46px);
      /* Line 148: Sets typography to a monospace terminal style consistent with the game fiction. */
      line-height: 1;
      /* Line 149: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      text-shadow: 0 0 18px currentColor;
      /* Line 150: Adds neon glow treatment in CSS to reinforce the cyberpunk terminal aesthetic. */
      word-break: break-word;
      /* Line 151: Contributes a focused visual or layout rule to the single-file responsive interface. */
    }
    /* Line 152: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .modal p {
    /* Line 154: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      margin: 0 auto 22px;
      /* Line 155: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      max-width: 44ch;
      /* Line 156: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      color: #d9ffff;
      /* Line 157: Applies a palette color that distinguishes player, warning, telemetry, or interface text. */
      font-size: clamp(14px, 2.3vmin, 18px);
      /* Line 158: Sets typography to a monospace terminal style consistent with the game fiction. */
      line-height: 1.45;
      /* Line 159: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
    }
    /* Line 160: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    button {
    /* Line 162: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      min-height: 44px;
      /* Line 163: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      border: 1px solid var(--green);
      /* Line 164: Defines a reusable custom property for the neon visual palette used across the interface. */
      background: rgba(51, 255, 51, 0.06);
      /* Line 165: Applies procedural color, gradient, or panel treatment without using external image assets. */
      color: var(--green);
      /* Line 166: Defines a reusable custom property for the neon visual palette used across the interface. */
      padding: 11px 18px;
      /* Line 167: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      font: 700 15px/1 "Courier New", Courier, monospace;
      /* Line 168: Sets typography to a monospace terminal style consistent with the game fiction. */
      cursor: pointer;
      /* Line 169: Contributes a focused visual or layout rule to the single-file responsive interface. */
      box-shadow: 0 0 14px rgba(51, 255, 51, 0.16), inset 0 0 12px rgba(51, 255, 51, 0.06);
      /* Line 170: Adds neon glow treatment in CSS to reinforce the cyberpunk terminal aesthetic. */
    }
    /* Line 171: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    button:hover,
    /* Line 173: Contributes a focused visual or layout rule to the single-file responsive interface. */
    button:focus-visible {
    /* Line 174: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      outline: none;
      /* Line 175: Contributes a focused visual or layout rule to the single-file responsive interface. */
      background: var(--green);
      /* Line 176: Defines a reusable custom property for the neon visual palette used across the interface. */
      color: #031008;
      /* Line 177: Applies a palette color that distinguishes player, warning, telemetry, or interface text. */
      box-shadow: 0 0 22px rgba(51, 255, 51, 0.55);
      /* Line 178: Adds neon glow treatment in CSS to reinforce the cyberpunk terminal aesthetic. */
    }
    /* Line 179: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .footer {
    /* Line 181: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      min-height: 44px;
      /* Line 182: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      display: flex;
      /* Line 183: Uses Flexbox for compact alignment of footer telemetry and controls. */
      align-items: center;
      /* Line 184: Contributes a focused visual or layout rule to the single-file responsive interface. */
      justify-content: space-between;
      /* Line 185: Contributes a focused visual or layout rule to the single-file responsive interface. */
      gap: 12px;
      /* Line 186: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      padding: 10px 14px;
      /* Line 187: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      color: rgba(217, 255, 255, 0.78);
      /* Line 188: Applies a palette color that distinguishes player, warning, telemetry, or interface text. */
      font-size: clamp(11px, 1.8vmin, 13px);
      /* Line 189: Sets typography to a monospace terminal style consistent with the game fiction. */
    }
    /* Line 190: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .meter {
    /* Line 192: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      flex: 1;
      /* Line 193: Contributes a focused visual or layout rule to the single-file responsive interface. */
      height: 8px;
      /* Line 194: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      border: 1px solid rgba(0, 255, 255, 0.32);
      /* Line 195: Draws a crisp vector-style boundary that matches the terminal instrumentation theme. */
      background: rgba(0, 255, 255, 0.05);
      /* Line 196: Applies procedural color, gradient, or panel treatment without using external image assets. */
      overflow: hidden;
      /* Line 197: Controls clipping behavior to prevent scrollbars or visual spill from disrupting play. */
    }
    /* Line 198: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    .meter > i {
    /* Line 200: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      display: block;
      /* Line 201: Contributes a focused visual or layout rule to the single-file responsive interface. */
      width: 100%;
      /* Line 202: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      height: 100%;
      /* Line 203: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      background: linear-gradient(90deg, var(--green), var(--amber), var(--red));
      /* Line 204: Defines a reusable custom property for the neon visual palette used across the interface. */
      box-shadow: 0 0 14px rgba(51, 255, 51, 0.5);
      /* Line 205: Adds neon glow treatment in CSS to reinforce the cyberpunk terminal aesthetic. */
      transform-origin: left center;
      /* Line 206: Uses a GPU-friendly transform for efficient visual state changes. */
      transform: scaleX(1);
      /* Line 207: Uses a GPU-friendly transform for efficient visual state changes. */
    }
    /* Line 208: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */

    @media (max-width: 660px) {
    /* Line 210: Starts a CSS selector block that scopes the following declarations to the matching interface region. */
      .shell { grid-template-rows: auto minmax(0, 1fr) auto; gap: 8px; padding: 8px; }
      /* Line 211: Defines spacing so the UI remains legible without shifting the canvas simulation area. */
      .hud { grid-template-columns: repeat(2, 1fr); min-height: 96px; }
      /* Line 212: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      .cell:nth-child(2) { border-right: 0; }
      /* Line 213: Draws a crisp vector-style boundary that matches the terminal instrumentation theme. */
      .cell:nth-child(-n + 2) { border-bottom: 1px solid rgba(0, 255, 255, 0.18); }
      /* Line 214: Draws a crisp vector-style boundary that matches the terminal instrumentation theme. */
      .footer { align-items: flex-start; flex-direction: column; }
      /* Line 215: Contributes a focused visual or layout rule to the single-file responsive interface. */
      .meter { width: 100%; flex: none; }
      /* Line 216: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
      .stage-wrap { width: min(96vw, 70vh); }
      /* Line 217: Constrains dimensions responsively so the playfield and UI remain stable across devices. */
    }
    /* Line 218: Closes the current CSS selector block so subsequent declarations do not leak into this rule. */
  </style>
  <!-- Line 219: Closes the embedded CSS block and returns parsing control to the HTML document. -->
</head>
<!-- Line 220: Ends the document metadata section before the visible game interface is declared. -->
<body>
<!-- Line 221: Starts the visible document body containing the HUD, canvas stage, overlay, and footer telemetry. -->
  <main class="shell">
  <!-- Line 222: Creates the top-level responsive shell that organizes the game into HUD, stage, and footer zones. -->
    <section class="hud" aria-label="Mission telemetry">
    <!-- Line 223: Defines the mission telemetry HUD exposed with an accessibility label. -->
      <div class="cell">
      <!-- Line 224: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
        <div class="label">LOOP</div>
        <!-- Line 225: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
        <div class="value" id="loopReadout">01 / 05</div>
        <!-- Line 226: Creates the loop counter readout that is updated by the runtime UI synchronization pass. -->
      </div>
      <!-- Line 227: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
      <div class="cell">
      <!-- Line 228: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
        <div class="label">TIME</div>
        <!-- Line 229: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
        <div class="value warn" id="timeReadout">10.00</div>
        <!-- Line 230: Creates the remaining-time readout used to communicate the ten-second loop pressure. -->
      </div>
      <!-- Line 231: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
      <div class="cell">
      <!-- Line 232: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
        <div class="label">DATA SHARDS</div>
        <!-- Line 233: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
        <div class="value" id="shardReadout">0 / 3</div>
        <!-- Line 234: Creates the collectible objective readout for the three data shards in the active loop. -->
      </div>
      <!-- Line 235: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
      <div class="cell">
      <!-- Line 236: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
        <div class="label">ECHOES</div>
        <!-- Line 237: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
        <div class="value cyan" id="echoReadout">0 ACTIVE</div>
        <!-- Line 238: Creates the active echo count readout so players can track accumulated timeline pressure. -->
      </div>
      <!-- Line 239: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
    </section>
    <!-- Line 240: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->

    <section class="stage-wrap" id="stageWrap" aria-label="RECURSION // ECHO game surface">
    <!-- Line 242: Defines the square playfield container that scales responsively while preserving world aspect ratio. -->
      <canvas id="gameCanvas" width="900" height="900"></canvas>
      <!-- Line 243: Creates the HTML5 Canvas surface where all procedural vector gameplay graphics are rendered. -->
      <div class="overlay is-active" id="overlay">
      <!-- Line 244: Declares the modal overlay used for the title, failure, and mission-success states. -->
        <div class="modal">
        <!-- Line 245: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
          <h1 id="overlayTitle">RECURSION // ECHO</h1>
          <!-- Line 246: Declares the modal overlay used for the title, failure, and mission-success states. -->
          <p id="overlayMessage">Collect 3 data shards in each 10 second loop. Every success creates an Echo Clone. Never collide with drones or your own history.</p>
          <!-- Line 247: Declares the modal overlay used for the title, failure, and mission-success states. -->
          <button type="button" id="restartButton">INITIALIZE RUN</button>
          <!-- Line 248: Creates the start/restart control bound to the game initialization routine. -->
        </div>
        <!-- Line 249: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
      </div>
      <!-- Line 250: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
    </section>
    <!-- Line 251: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->

    <section class="footer" aria-label="Controls and timer">
    <!-- Line 253: Defines the footer telemetry strip containing controls, timer meter, and status output. -->
      <span>WASD / ARROWS MOVE | POINTER DRAG SUPPORTED</span>
      <!-- Line 254: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
      <div class="meter" aria-hidden="true"><i id="timeMeter"></i></div>
      <!-- Line 255: Creates the visual time meter whose CSS transform is driven by remaining loop time. -->
      <span id="statusReadout">STATUS: STANDBY</span>
      <!-- Line 256: Creates the status readout used to expose runtime state changes to the player. -->
    </section>
    <!-- Line 257: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->
  </main>
  <!-- Line 258: Declares part of the static HTML structure that the runtime styles, updates, or renders into. -->

  <script>
  // Line 260: Begins the embedded JavaScript block, keeping all game systems in the deployable HTML file.
    (() => {
    // Line 261: Starts an immediately invoked function expression that isolates game variables from global scope.
      "use strict";
      // Line 262: Enables strict-mode JavaScript to catch unsafe assignments and improve runtime predictability.

      const canvas = document.getElementById("gameCanvas");
      // Line 264: Caches a DOM reference once so the game loop avoids repeated document queries.
      const ctx = canvas.getContext("2d", { alpha: false });
      // Line 265: Initializes the 2D Canvas rendering context with opaque output for efficient full-canvas redraws.
      const stageWrap = document.getElementById("stageWrap");
      // Line 266: Caches a DOM reference once so the game loop avoids repeated document queries.
      const loopReadout = document.getElementById("loopReadout");
      // Line 267: Creates the loop counter readout that is updated by the runtime UI synchronization pass.
      const timeReadout = document.getElementById("timeReadout");
      // Line 268: Creates the remaining-time readout used to communicate the ten-second loop pressure.
      const shardReadout = document.getElementById("shardReadout");
      // Line 269: Creates the collectible objective readout for the three data shards in the active loop.
      const echoReadout = document.getElementById("echoReadout");
      // Line 270: Creates the active echo count readout so players can track accumulated timeline pressure.
      const statusReadout = document.getElementById("statusReadout");
      // Line 271: Creates the status readout used to expose runtime state changes to the player.
      const timeMeter = document.getElementById("timeMeter");
      // Line 272: Creates the visual time meter whose CSS transform is driven by remaining loop time.
      const overlay = document.getElementById("overlay");
      // Line 273: Declares the modal overlay used for the title, failure, and mission-success states.
      const overlayTitle = document.getElementById("overlayTitle");
      // Line 274: Declares the modal overlay used for the title, failure, and mission-success states.
      const overlayMessage = document.getElementById("overlayMessage");
      // Line 275: Declares the modal overlay used for the title, failure, and mission-success states.
      const restartButton = document.getElementById("restartButton");
      // Line 276: Creates the start/restart control bound to the game initialization routine.

      const WORLD = 900;
      // Line 278: Defines the fixed simulation-space dimension used independently from responsive CSS scaling.
      const LOOP_DURATION = 10;
      // Line 279: Defines the exact ten-second temporal window used by every cascading loop.
      const MAX_LOOPS = 5;
      // Line 280: Defines the five-stage win condition for the recursive timeline stack.
      const TARGET_SHARDS = 3;
      // Line 281: Defines the collectible objective count required before a rewind can succeed.
      const PLAYER_RADIUS = 15;
      // Line 282: Defines a collision radius used by the Euclidean distance interaction system.
      const ECHO_RADIUS = 15;
      // Line 283: Defines a collision radius used by the Euclidean distance interaction system.
      const DRONE_RADIUS = 24;
      // Line 284: Defines a collision radius used by the Euclidean distance interaction system.
      const SHARD_RADIUS = 14;
      // Line 285: Defines a collision radius used by the Euclidean distance interaction system.
      const ORIGIN = Object.freeze({ x: WORLD * 0.5, y: WORLD - 76 });
      // Line 286: Defines the player respawn origin at the bottom-center of the simulation world.
      const PLAYER_SPEED = 365;
      // Line 287: Defines movement speed in world units per second for frame-rate independent motion.
      const POINTER_GAIN = 5.8;
      // Line 288: Defines pointer steering influence for mouse and touch control without changing keyboard physics.

      const keys = new Set();
      // Line 290: Allocates the active-input set used by keyboard event handlers and movement integration.
      const echoes = [];
      // Line 291: Allocates the historical playback stack that stores completed loop paths.
      const currentHistory = [];
      // Line 292: Allocates the rolling runtime sequence array that records current player coordinates every frame.
      const shards = [];
      // Line 293: Allocates the active collectible entity list for the current loop.
      const drones = [];
      // Line 294: Allocates the autonomous hazard entity list for the current loop.
      const sparks = [];
      // Line 295: Allocates transient visual particles created by successful shard pickups.

      let player;
      // Line 297: Declares mutable runtime state that changes as the simulation advances.
      let loopNumber = 1;
      // Line 298: Declares mutable runtime state that changes as the simulation advances.
      let loopTime = 0;
      // Line 299: Declares mutable runtime state that changes as the simulation advances.
      let lastStamp = 0;
      // Line 300: Declares mutable runtime state that changes as the simulation advances.
      let runState = "menu";
      // Line 301: Declares mutable runtime state that changes as the simulation advances.
      let pointer = { active: false, x: ORIGIN.x, y: ORIGIN.y };
      // Line 302: Defines the player respawn origin at the bottom-center of the simulation world.
      let rngState = 0x2f6e2b1;
      // Line 303: Declares mutable runtime state that changes as the simulation advances.

      function rand() {
      // Line 305: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        rngState = (rngState * 1664525 + 1013904223) >>> 0;
        // Line 306: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        return rngState / 4294967296;
        // Line 307: Returns control or computed data to the caller, keeping subsystem responsibilities explicit.
      }
      // Line 308: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function dist(a, b) {
      // Line 310: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        const dx = a.x - b.x;
        // Line 311: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        const dy = a.y - b.y;
        // Line 312: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        return Math.sqrt(dx * dx + dy * dy);
        // Line 313: Computes an exact Euclidean magnitude used for movement normalization or circular collision distance.
      }
      // Line 314: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function clamp(value, min, max) {
      // Line 316: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        return Math.max(min, Math.min(max, value));
        // Line 317: Returns control or computed data to the caller, keeping subsystem responsibilities explicit.
      }
      // Line 318: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function resetPlayer() {
      // Line 320: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        player = {
        // Line 321: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          x: ORIGIN.x,
          // Line 322: Defines the player respawn origin at the bottom-center of the simulation world.
          y: ORIGIN.y,
          // Line 323: Defines the player respawn origin at the bottom-center of the simulation world.
          vx: 0,
          // Line 324: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          vy: 0,
          // Line 325: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          radius: PLAYER_RADIUS,
          // Line 326: Defines a collision radius used by the Euclidean distance interaction system.
          alivePulse: 0
          // Line 327: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        };
        // Line 328: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        pointer.x = player.x;
        // Line 329: Updates pointer-control state for mouse and touch steering.
        pointer.y = player.y;
        // Line 330: Updates pointer-control state for mouse and touch steering.
      }
      // Line 331: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function seedDrones() {
      // Line 333: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        drones.length = 0;
        // Line 334: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        const configs = [
        // Line 335: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          { x: 168, y: 168, vx: 178, vy: 82, phase: 0.0 },
          // Line 336: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          { x: 720, y: 256, vx: -214, vy: 118, phase: 1.6 },
          // Line 337: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          { x: 274, y: 574, vx: 142, vy: -188, phase: 3.1 },
          // Line 338: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          { x: 650, y: 656, vx: -120, vy: -150, phase: 4.4 }
          // Line 339: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        ];
        // Line 340: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        for (let i = 0; i < configs.length; i++) {
        // Line 341: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          drones.push({
          // Line 342: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            x: configs[i].x,
            // Line 343: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            y: configs[i].y,
            // Line 344: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            vx: configs[i].vx,
            // Line 345: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            vy: configs[i].vy,
            // Line 346: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            phase: configs[i].phase,
            // Line 347: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            radius: DRONE_RADIUS
            // Line 348: Defines a collision radius used by the Euclidean distance interaction system.
          });
          // Line 349: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        }
        // Line 350: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      }
      // Line 351: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function spawnShards() {
      // Line 353: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        shards.length = 0;
        // Line 354: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        const candidates = [
        // Line 355: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          { x: 132, y: 132 }, { x: 450, y: 118 }, { x: 768, y: 146 },
          // Line 356: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          { x: 186, y: 344 }, { x: 454, y: 338 }, { x: 714, y: 342 },
          // Line 357: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          { x: 132, y: 600 }, { x: 436, y: 608 }, { x: 760, y: 582 },
          // Line 358: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          { x: 252, y: 752 }, { x: 640, y: 752 }
          // Line 359: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        ];
        // Line 360: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        for (let i = candidates.length - 1; i > 0; i--) {
        // Line 361: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          const j = Math.floor(rand() * (i + 1));
          // Line 362: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          const temp = candidates[i];
          // Line 363: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          candidates[i] = candidates[j];
          // Line 364: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          candidates[j] = temp;
          // Line 365: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        }
        // Line 366: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        let index = 0;
        // Line 367: Declares mutable runtime state that changes as the simulation advances.
        while (shards.length < TARGET_SHARDS && index < candidates.length) {
        // Line 368: Defines the collectible objective count required before a rewind can succeed.
          const c = candidates[index++];
          // Line 369: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          if (dist(c, ORIGIN) > 130) {
          // Line 370: Defines the player respawn origin at the bottom-center of the simulation world.
            shards.push({ x: c.x, y: c.y, radius: SHARD_RADIUS, collected: false, phase: rand() * Math.PI * 2 });
            // Line 371: Defines a collision radius used by the Euclidean distance interaction system.
          }
          // Line 372: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        }
        // Line 373: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      }
      // Line 374: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function startRun() {
      // Line 376: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        loopNumber = 1;
        // Line 377: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        loopTime = 0;
        // Line 378: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        echoes.length = 0;
        // Line 379: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        currentHistory.length = 0;
        // Line 380: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        sparks.length = 0;
        // Line 381: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        rngState = (Date.now() ^ 0xa7f3c91) >>> 0;
        // Line 382: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        resetPlayer();
        // Line 383: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        seedDrones();
        // Line 384: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        spawnShards();
        // Line 385: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        runState = "playing";
        // Line 386: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        overlay.classList.remove("is-active");
        // Line 387: Declares the modal overlay used for the title, failure, and mission-success states.
        statusReadout.textContent = "STATUS: LIVE";
        // Line 388: Creates the status readout used to expose runtime state changes to the player.
        lastStamp = performance.now();
        // Line 389: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        requestAnimationFrame(frame);
        // Line 390: Schedules simulation and rendering on the browser animation clock for smooth hardware-synced updates.
      }
      // Line 391: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function finishRun(title, message, success) {
      // Line 393: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        runState = success ? "victory" : "collapse";
        // Line 394: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        overlayTitle.textContent = title;
        // Line 395: Declares the modal overlay used for the title, failure, and mission-success states.
        overlayTitle.style.color = success ? "#33ff33" : "#ff3366";
        // Line 396: Declares the modal overlay used for the title, failure, and mission-success states.
        overlayMessage.textContent = message;
        // Line 397: Declares the modal overlay used for the title, failure, and mission-success states.
        restartButton.textContent = success ? "RUN AGAIN" : "RE-INITIALIZE TIMELINE";
        // Line 398: Creates the start/restart control bound to the game initialization routine.
        overlay.classList.add("is-active");
        // Line 399: Declares the modal overlay used for the title, failure, and mission-success states.
        statusReadout.textContent = success ? "STATUS: CLEAN EXIT" : "STATUS: TIMELINE COLLAPSE";
        // Line 400: Creates the status readout used to expose runtime state changes to the player.
      }
      // Line 401: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function advanceLoop() {
      // Line 403: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        echoes.push(currentHistory.map((p) => ({ x: p.x, y: p.y })));
        // Line 404: Commits a completed loop history into the echo playback stack for future paradox checks.
        currentHistory.length = 0;
        // Line 405: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        loopNumber += 1;
        // Line 406: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        loopTime = 0;
        // Line 407: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        resetPlayer();
        // Line 408: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        seedDrones();
        // Line 409: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        spawnShards();
        // Line 410: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        statusReadout.textContent = "STATUS: REWIND COMPLETE";
        // Line 411: Creates the status readout used to expose runtime state changes to the player.
      }
      // Line 412: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function emitSparks(x, y, color, count) {
      // Line 414: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        for (let i = 0; i < count; i++) {
        // Line 415: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          const angle = rand() * Math.PI * 2;
          // Line 416: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          const speed = 80 + rand() * 190;
          // Line 417: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          sparks.push({
          // Line 418: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            x,
            // Line 419: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            y,
            // Line 420: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            vx: Math.cos(angle) * speed,
            // Line 421: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            vy: Math.sin(angle) * speed,
            // Line 422: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            life: 0.42 + rand() * 0.32,
            // Line 423: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            maxLife: 0.74,
            // Line 424: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            color
            // Line 425: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          });
          // Line 426: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        }
        // Line 427: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      }
      // Line 428: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function updateInput(dt) {
      // Line 430: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        let ax = 0;
        // Line 431: Declares mutable runtime state that changes as the simulation advances.
        let ay = 0;
        // Line 432: Declares mutable runtime state that changes as the simulation advances.
        if (keys.has("arrowleft") || keys.has("a")) ax -= 1;
        // Line 433: Branches the simulation based on input, collision, objective, or state-machine conditions.
        if (keys.has("arrowright") || keys.has("d")) ax += 1;
        // Line 434: Branches the simulation based on input, collision, objective, or state-machine conditions.
        if (keys.has("arrowup") || keys.has("w")) ay -= 1;
        // Line 435: Branches the simulation based on input, collision, objective, or state-machine conditions.
        if (keys.has("arrowdown") || keys.has("s")) ay += 1;
        // Line 436: Branches the simulation based on input, collision, objective, or state-machine conditions.

        if (pointer.active) {
        // Line 438: Updates pointer-control state for mouse and touch steering.
          const dx = pointer.x - player.x;
          // Line 439: Updates pointer-control state for mouse and touch steering.
          const dy = pointer.y - player.y;
          // Line 440: Updates pointer-control state for mouse and touch steering.
          const mag = Math.sqrt(dx * dx + dy * dy);
          // Line 441: Computes an exact Euclidean magnitude used for movement normalization or circular collision distance.
          if (mag > 4) {
          // Line 442: Branches the simulation based on input, collision, objective, or state-machine conditions.
            ax += clamp(dx / 120, -1, 1) * POINTER_GAIN;
            // Line 443: Defines pointer steering influence for mouse and touch control without changing keyboard physics.
            ay += clamp(dy / 120, -1, 1) * POINTER_GAIN;
            // Line 444: Defines pointer steering influence for mouse and touch control without changing keyboard physics.
          }
          // Line 445: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        }
        // Line 446: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

        const len = Math.sqrt(ax * ax + ay * ay);
        // Line 448: Computes an exact Euclidean magnitude used for movement normalization or circular collision distance.
        if (len > 0) {
        // Line 449: Branches the simulation based on input, collision, objective, or state-machine conditions.
          ax /= len;
          // Line 450: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          ay /= len;
          // Line 451: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        }
        // Line 452: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

        const targetVx = ax * PLAYER_SPEED;
        // Line 454: Defines movement speed in world units per second for frame-rate independent motion.
        const targetVy = ay * PLAYER_SPEED;
        // Line 455: Defines movement speed in world units per second for frame-rate independent motion.
        const follow = 1 - Math.pow(0.0007, dt);
        // Line 456: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        player.vx += (targetVx - player.vx) * follow;
        // Line 457: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        player.vy += (targetVy - player.vy) * follow;
        // Line 458: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        player.x += player.vx * dt;
        // Line 459: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        player.y += player.vy * dt;
        // Line 460: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        player.x = clamp(player.x, player.radius, WORLD - player.radius);
        // Line 461: Clamps values to prevent entities, timers, or visual meters from exceeding valid bounds.
        player.y = clamp(player.y, player.radius, WORLD - player.radius);
        // Line 462: Clamps values to prevent entities, timers, or visual meters from exceeding valid bounds.
        player.alivePulse += dt;
        // Line 463: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      }
      // Line 464: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function updateDrones(dt) {
      // Line 466: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        for (const d of drones) {
        // Line 467: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          d.phase += dt * 3.4;
          // Line 468: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          d.x += d.vx * dt;
          // Line 469: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          d.y += d.vy * dt;
          // Line 470: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          if (d.x < d.radius || d.x > WORLD - d.radius) {
          // Line 471: Branches the simulation based on input, collision, objective, or state-machine conditions.
            d.x = clamp(d.x, d.radius, WORLD - d.radius);
            // Line 472: Clamps values to prevent entities, timers, or visual meters from exceeding valid bounds.
            d.vx *= -1;
            // Line 473: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          }
          // Line 474: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          if (d.y < d.radius || d.y > WORLD - d.radius) {
          // Line 475: Branches the simulation based on input, collision, objective, or state-machine conditions.
            d.y = clamp(d.y, d.radius, WORLD - d.radius);
            // Line 476: Clamps values to prevent entities, timers, or visual meters from exceeding valid bounds.
            d.vy *= -1;
            // Line 477: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          }
          // Line 478: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        }
        // Line 479: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      }
      // Line 480: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function updateSparks(dt) {
      // Line 482: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        for (let i = sparks.length - 1; i >= 0; i--) {
        // Line 483: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          const s = sparks[i];
          // Line 484: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          s.life -= dt;
          // Line 485: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          s.x += s.vx * dt;
          // Line 486: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          s.y += s.vy * dt;
          // Line 487: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          s.vx *= Math.pow(0.05, dt);
          // Line 488: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          s.vy *= Math.pow(0.05, dt);
          // Line 489: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          if (s.life <= 0) sparks.splice(i, 1);
          // Line 490: Branches the simulation based on input, collision, objective, or state-machine conditions.
        }
        // Line 491: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      }
      // Line 492: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function recordPlayerFrame() {
      // Line 494: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        currentHistory.push({
        // Line 495: Serializes the player coordinate transformation into the frame-by-frame playback sequence.
          x: player.x,
          // Line 496: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          y: player.y
          // Line 497: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        });
        // Line 498: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      }
      // Line 499: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function collisionChecks() {
      // Line 501: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        for (let i = 0; i < drones.length; i++) {
        // Line 502: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          if (dist(player, drones[i]) < player.radius + drones[i].radius) {
          // Line 503: Performs an O(N) Euclidean spatial check against gameplay entities or timeline echoes.
            finishRun("TIMELINE COLLAPSED", "Paradox exception: live cursor intersected security firewall drone #" + (i + 1) + ".", false);
            // Line 504: Transitions the state machine into victory or collapse with a specific diagnostic message.
            return;
            // Line 505: Returns control or computed data to the caller, keeping subsystem responsibilities explicit.
          }
          // Line 506: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        }
        // Line 507: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

        const frameIndex = currentHistory.length - 1;
        // Line 509: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        for (let i = 0; i < echoes.length; i++) {
        // Line 510: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          const history = echoes[i];
          // Line 511: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          const echo = history[Math.min(frameIndex, history.length - 1)];
          // Line 512: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          if (echo && dist(player, echo) < player.radius + ECHO_RADIUS) {
          // Line 513: Defines a collision radius used by the Euclidean distance interaction system.
            const playerInOriginGate = dist(player, ORIGIN) < 42;
            // Line 514: Defines the player respawn origin at the bottom-center of the simulation world.
            const echoInOriginGate = dist(echo, ORIGIN) < 42;
            // Line 515: Defines the player respawn origin at the bottom-center of the simulation world.
            if (playerInOriginGate && echoInOriginGate && loopTime < 0.85) continue;
            // Line 516: Branches the simulation based on input, collision, objective, or state-machine conditions.
            finishRun("TIMELINE COLLAPSED", "Paradox exception: live cursor crossed Echo Clone loop #" + (i + 1) + " at frame " + frameIndex + ".", false);
            // Line 517: Transitions the state machine into victory or collapse with a specific diagnostic message.
            return;
            // Line 518: Returns control or computed data to the caller, keeping subsystem responsibilities explicit.
          }
          // Line 519: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        }
        // Line 520: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

        for (const shard of shards) {
        // Line 522: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          if (!shard.collected && dist(player, shard) < player.radius + shard.radius) {
          // Line 523: Performs an O(N) Euclidean spatial check against gameplay entities or timeline echoes.
            shard.collected = true;
            // Line 524: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            emitSparks(shard.x, shard.y, "#ffec66", 18);
            // Line 525: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          }
          // Line 526: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        }
        // Line 527: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      }
      // Line 528: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function update(dt) {
      // Line 530: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        loopTime += dt;
        // Line 531: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        updateInput(dt);
        // Line 532: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        updateDrones(dt);
        // Line 533: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        updateSparks(dt);
        // Line 534: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        recordPlayerFrame();
        // Line 535: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        collisionChecks();
        // Line 536: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

        if (runState !== "playing") return;
        // Line 538: Branches the simulation based on input, collision, objective, or state-machine conditions.

        const collected = shards.filter((s) => s.collected).length;
        // Line 540: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        if (loopTime >= LOOP_DURATION) {
        // Line 541: Defines the exact ten-second temporal window used by every cascading loop.
          if (collected < TARGET_SHARDS) {
          // Line 542: Defines the collectible objective count required before a rewind can succeed.
            finishRun("TIMELINE COLLAPSED", "Paradox exception: loop expired with " + collected + " / " + TARGET_SHARDS + " data shards recovered.", false);
            // Line 543: Defines the collectible objective count required before a rewind can succeed.
          } else if (loopNumber >= MAX_LOOPS) {
          // Line 544: Defines the five-stage win condition for the recursive timeline stack.
            finishRun("SYSTEM HACKED", "Five recursive loops resolved. Echo stack stable. No paradox intersections detected.", true);
            // Line 545: Transitions the state machine into victory or collapse with a specific diagnostic message.
          } else {
          // Line 546: Branches the simulation based on input, collision, objective, or state-machine conditions.
            advanceLoop();
            // Line 547: Triggers the rewind transition after the active loop meets its shard objective.
          }
          // Line 548: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        }
        // Line 549: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      }
      // Line 550: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function drawBackground(time) {
      // Line 552: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        ctx.fillStyle = "#03040b";
        // Line 553: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.fillRect(0, 0, WORLD, WORLD);
        // Line 554: Issues a Canvas API command that contributes to the procedural vector render pass.

        ctx.save();
        // Line 556: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineWidth = 1;
        // Line 557: Issues a Canvas API command that contributes to the procedural vector render pass.
        for (let i = 0; i <= WORLD; i += 45) {
        // Line 558: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          const major = i % 180 === 0;
          // Line 559: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          ctx.strokeStyle = major ? "rgba(0,255,255,0.16)" : "rgba(0,255,255,0.055)";
          // Line 560: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.beginPath();
          // Line 561: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.moveTo(i, 0);
          // Line 562: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.lineTo(i, WORLD);
          // Line 563: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.stroke();
          // Line 564: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.beginPath();
          // Line 565: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.moveTo(0, i);
          // Line 566: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.lineTo(WORLD, i);
          // Line 567: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.stroke();
          // Line 568: Issues a Canvas API command that contributes to the procedural vector render pass.
        }
        // Line 569: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

        ctx.strokeStyle = "rgba(51,255,51,0.16)";
        // Line 571: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineWidth = 2;
        // Line 572: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.strokeRect(28, 28, WORLD - 56, WORLD - 56);
        // Line 573: Issues a Canvas API command that contributes to the procedural vector render pass.

        ctx.strokeStyle = "rgba(255,51,102,0.11)";
        // Line 575: Issues a Canvas API command that contributes to the procedural vector render pass.
        for (let y = 90; y < WORLD; y += 180) {
        // Line 576: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          ctx.beginPath();
          // Line 577: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.moveTo(0, y + Math.sin(time + y) * 8);
          // Line 578: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.lineTo(WORLD, y + Math.cos(time * 0.7 + y) * 8);
          // Line 579: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.stroke();
          // Line 580: Issues a Canvas API command that contributes to the procedural vector render pass.
        }
        // Line 581: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        ctx.restore();
        // Line 582: Issues a Canvas API command that contributes to the procedural vector render pass.
      }
      // Line 583: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function drawShard(s, time) {
      // Line 585: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        if (s.collected) return;
        // Line 586: Branches the simulation based on input, collision, objective, or state-machine conditions.
        const r = s.radius + Math.sin(time * 5 + s.phase) * 2;
        // Line 587: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        ctx.save();
        // Line 588: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.translate(s.x, s.y);
        // Line 589: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.rotate(time * 1.9 + s.phase);
        // Line 590: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.shadowBlur = 18;
        // Line 591: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.shadowColor = "#ffec66";
        // Line 592: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.strokeStyle = "#ffec66";
        // Line 593: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.fillStyle = "rgba(255,236,102,0.18)";
        // Line 594: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineWidth = 3;
        // Line 595: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.beginPath();
        // Line 596: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.moveTo(0, -r * 1.25);
        // Line 597: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineTo(r, 0);
        // Line 598: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineTo(0, r * 1.25);
        // Line 599: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineTo(-r, 0);
        // Line 600: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.closePath();
        // Line 601: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.fill();
        // Line 602: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.stroke();
        // Line 603: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.restore();
        // Line 604: Issues a Canvas API command that contributes to the procedural vector render pass.
      }
      // Line 605: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function drawDrone(d) {
      // Line 607: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        ctx.save();
        // Line 608: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.translate(d.x, d.y);
        // Line 609: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.shadowBlur = 22;
        // Line 610: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.shadowColor = "#ff3366";
        // Line 611: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.strokeStyle = "#ff3366";
        // Line 612: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineWidth = 4;
        // Line 613: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.beginPath();
        // Line 614: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.arc(0, 0, d.radius, 0, Math.PI * 2);
        // Line 615: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.stroke();
        // Line 616: Issues a Canvas API command that contributes to the procedural vector render pass.

        ctx.rotate(d.phase);
        // Line 618: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineWidth = 2;
        // Line 619: Issues a Canvas API command that contributes to the procedural vector render pass.
        for (let i = 0; i < 4; i++) {
        // Line 620: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          ctx.rotate(Math.PI / 2);
          // Line 621: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.beginPath();
          // Line 622: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.moveTo(d.radius * 0.55, 0);
          // Line 623: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.lineTo(d.radius * 1.45, 0);
          // Line 624: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.stroke();
          // Line 625: Issues a Canvas API command that contributes to the procedural vector render pass.
        }
        // Line 626: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        ctx.fillStyle = "rgba(255,51,102,0.18)";
        // Line 627: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.beginPath();
        // Line 628: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.arc(0, 0, d.radius * 0.45, 0, Math.PI * 2);
        // Line 629: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.fill();
        // Line 630: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.restore();
        // Line 631: Issues a Canvas API command that contributes to the procedural vector render pass.
      }
      // Line 632: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function drawEchoes(time) {
      // Line 634: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        const frameIndex = Math.max(0, currentHistory.length - 1);
        // Line 635: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        for (let i = 0; i < echoes.length; i++) {
        // Line 636: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          const history = echoes[i];
          // Line 637: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          const echo = history[Math.min(frameIndex, history.length - 1)];
          // Line 638: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          if (!echo) continue;
          // Line 639: Branches the simulation based on input, collision, objective, or state-machine conditions.

          ctx.save();
          // Line 641: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.shadowBlur = 18;
          // Line 642: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.shadowColor = "#00ffff";
          // Line 643: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.strokeStyle = "rgba(0,255,255,0.2)";
          // Line 644: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.lineWidth = 2;
          // Line 645: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.beginPath();
          // Line 646: Issues a Canvas API command that contributes to the procedural vector render pass.
          const start = Math.max(0, Math.min(frameIndex, history.length - 1) - 90);
          // Line 647: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          for (let p = start; p <= Math.min(frameIndex, history.length - 1); p += 2) {
          // Line 648: Runs a bounded iteration over entities, history samples, or procedural draw steps.
            const pos = history[p];
            // Line 649: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
            if (p === start) ctx.moveTo(pos.x, pos.y);
            // Line 650: Issues a Canvas API command that contributes to the procedural vector render pass.
            else ctx.lineTo(pos.x, pos.y);
            // Line 651: Issues a Canvas API command that contributes to the procedural vector render pass.
          }
          // Line 652: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          ctx.stroke();
          // Line 653: Issues a Canvas API command that contributes to the procedural vector render pass.

          ctx.globalAlpha = 0.52;
          // Line 655: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.fillStyle = "rgba(0,255,255,0.26)";
          // Line 656: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.strokeStyle = "#00ffff";
          // Line 657: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.lineWidth = 2;
          // Line 658: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.beginPath();
          // Line 659: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.arc(echo.x, echo.y, ECHO_RADIUS + Math.sin(time * 6 + i) * 1.5, 0, Math.PI * 2);
          // Line 660: Defines a collision radius used by the Euclidean distance interaction system.
          ctx.fill();
          // Line 661: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.stroke();
          // Line 662: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.restore();
          // Line 663: Issues a Canvas API command that contributes to the procedural vector render pass.
        }
        // Line 664: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      }
      // Line 665: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function drawPlayer(time) {
      // Line 667: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        ctx.save();
        // Line 668: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.translate(player.x, player.y);
        // Line 669: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.shadowBlur = 24;
        // Line 670: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.shadowColor = "#33ff33";
        // Line 671: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.strokeStyle = "#33ff33";
        // Line 672: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.fillStyle = "rgba(51,255,51,0.22)";
        // Line 673: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineWidth = 3;
        // Line 674: Issues a Canvas API command that contributes to the procedural vector render pass.
        const pulse = Math.sin(time * 10) * 2;
        // Line 675: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        ctx.beginPath();
        // Line 676: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.arc(0, 0, player.radius + pulse, 0, Math.PI * 2);
        // Line 677: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.fill();
        // Line 678: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.stroke();
        // Line 679: Issues a Canvas API command that contributes to the procedural vector render pass.

        ctx.lineWidth = 2;
        // Line 681: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.beginPath();
        // Line 682: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.moveTo(-24, 0);
        // Line 683: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineTo(-8, 0);
        // Line 684: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.moveTo(8, 0);
        // Line 685: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineTo(24, 0);
        // Line 686: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.moveTo(0, -24);
        // Line 687: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineTo(0, -8);
        // Line 688: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.moveTo(0, 8);
        // Line 689: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineTo(0, 24);
        // Line 690: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.stroke();
        // Line 691: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.restore();
        // Line 692: Issues a Canvas API command that contributes to the procedural vector render pass.
      }
      // Line 693: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function drawSparks() {
      // Line 695: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        ctx.save();
        // Line 696: Issues a Canvas API command that contributes to the procedural vector render pass.
        for (const s of sparks) {
        // Line 697: Runs a bounded iteration over entities, history samples, or procedural draw steps.
          const a = clamp(s.life / s.maxLife, 0, 1);
          // Line 698: Clamps values to prevent entities, timers, or visual meters from exceeding valid bounds.
          ctx.globalAlpha = a;
          // Line 699: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.shadowBlur = 14;
          // Line 700: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.shadowColor = s.color;
          // Line 701: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.fillStyle = s.color;
          // Line 702: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.beginPath();
          // Line 703: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.arc(s.x, s.y, 2 + a * 3, 0, Math.PI * 2);
          // Line 704: Issues a Canvas API command that contributes to the procedural vector render pass.
          ctx.fill();
          // Line 705: Issues a Canvas API command that contributes to the procedural vector render pass.
        }
        // Line 706: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        ctx.restore();
        // Line 707: Issues a Canvas API command that contributes to the procedural vector render pass.
      }
      // Line 708: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function draw() {
      // Line 710: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        const time = performance.now() / 1000;
        // Line 711: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        drawBackground(time);
        // Line 712: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        drawEchoes(time);
        // Line 713: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        for (const shard of shards) drawShard(shard, time);
        // Line 714: Runs a bounded iteration over entities, history samples, or procedural draw steps.
        for (const d of drones) drawDrone(d);
        // Line 715: Runs a bounded iteration over entities, history samples, or procedural draw steps.
        drawSparks();
        // Line 716: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        drawPlayer(time);
        // Line 717: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

        ctx.save();
        // Line 719: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.shadowBlur = 10;
        // Line 720: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.shadowColor = "#00ffff";
        // Line 721: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.strokeStyle = "rgba(0,255,255,0.36)";
        // Line 722: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.lineWidth = 2;
        // Line 723: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.beginPath();
        // Line 724: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.arc(ORIGIN.x, ORIGIN.y, 28, 0, Math.PI * 2);
        // Line 725: Defines the player respawn origin at the bottom-center of the simulation world.
        ctx.stroke();
        // Line 726: Issues a Canvas API command that contributes to the procedural vector render pass.
        ctx.restore();
        // Line 727: Issues a Canvas API command that contributes to the procedural vector render pass.
      }
      // Line 728: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function updateUi() {
      // Line 730: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        const remaining = Math.max(0, LOOP_DURATION - loopTime);
        // Line 731: Defines the exact ten-second temporal window used by every cascading loop.
        const collected = shards.filter((s) => s.collected).length;
        // Line 732: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        loopReadout.textContent = String(loopNumber).padStart(2, "0") + " / 05";
        // Line 733: Creates the loop counter readout that is updated by the runtime UI synchronization pass.
        timeReadout.textContent = remaining.toFixed(2);
        // Line 734: Creates the remaining-time readout used to communicate the ten-second loop pressure.
        shardReadout.textContent = collected + " / " + TARGET_SHARDS;
        // Line 735: Creates the collectible objective readout for the three data shards in the active loop.
        echoReadout.textContent = echoes.length + (echoes.length === 1 ? " ACTIVE" : " ACTIVE");
        // Line 736: Creates the active echo count readout so players can track accumulated timeline pressure.
        timeMeter.style.transform = "scaleX(" + clamp(remaining / LOOP_DURATION, 0, 1).toFixed(4) + ")";
        // Line 737: Creates the visual time meter whose CSS transform is driven by remaining loop time.
      }
      // Line 738: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function frame(stamp) {
      // Line 740: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        if (runState !== "playing") {
        // Line 741: Branches the simulation based on input, collision, objective, or state-machine conditions.
          draw();
          // Line 742: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          updateUi();
          // Line 743: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
          return;
          // Line 744: Returns control or computed data to the caller, keeping subsystem responsibilities explicit.
        }
        // Line 745: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        const dt = Math.min(0.05, Math.max(0, (stamp - lastStamp) / 1000));
        // Line 746: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        lastStamp = stamp;
        // Line 747: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        update(dt);
        // Line 748: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        draw();
        // Line 749: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        updateUi();
        // Line 750: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        if (runState === "playing") requestAnimationFrame(frame);
        // Line 751: Schedules simulation and rendering on the browser animation clock for smooth hardware-synced updates.
      }
      // Line 752: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      function pointerToWorld(event) {
      // Line 754: Declares a named subsystem function used by the game loop, renderer, input layer, or state machine.
        const rect = canvas.getBoundingClientRect();
        // Line 755: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        return {
        // Line 756: Returns control or computed data to the caller, keeping subsystem responsibilities explicit.
          x: clamp(((event.clientX - rect.left) / rect.width) * WORLD, 0, WORLD),
          // Line 757: Clamps values to prevent entities, timers, or visual meters from exceeding valid bounds.
          y: clamp(((event.clientY - rect.top) / rect.height) * WORLD, 0, WORLD)
          // Line 758: Clamps values to prevent entities, timers, or visual meters from exceeding valid bounds.
        };
        // Line 759: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      }
      // Line 760: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      window.addEventListener("keydown", (event) => {
      // Line 762: Registers an event listener that connects browser input to the game control system.
        const key = event.key.toLowerCase();
        // Line 763: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        if (["arrowup", "arrowdown", "arrowleft", "arrowright", "w", "a", "s", "d", " "].includes(key)) {
        // Line 764: Branches the simulation based on input, collision, objective, or state-machine conditions.
          event.preventDefault();
          // Line 765: Suppresses browser scrolling or page interaction so movement keys remain dedicated to gameplay.
        }
        // Line 766: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
        if (key === " " && runState !== "playing") startRun();
        // Line 767: Branches the simulation based on input, collision, objective, or state-machine conditions.
        keys.add(key);
        // Line 768: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      }, { passive: false });
      // Line 769: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      window.addEventListener("keyup", (event) => {
      // Line 771: Registers an event listener that connects browser input to the game control system.
        keys.delete(event.key.toLowerCase());
        // Line 772: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      });
      // Line 773: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      stageWrap.addEventListener("pointerdown", (event) => {
      // Line 775: Registers an event listener that connects browser input to the game control system.
        pointer.active = true;
        // Line 776: Updates pointer-control state for mouse and touch steering.
        const p = pointerToWorld(event);
        // Line 777: Updates pointer-control state for mouse and touch steering.
        pointer.x = p.x;
        // Line 778: Updates pointer-control state for mouse and touch steering.
        pointer.y = p.y;
        // Line 779: Updates pointer-control state for mouse and touch steering.
        stageWrap.setPointerCapture(event.pointerId);
        // Line 780: Updates pointer-control state for mouse and touch steering.
        if (runState !== "playing") startRun();
        // Line 781: Branches the simulation based on input, collision, objective, or state-machine conditions.
      });
      // Line 782: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      stageWrap.addEventListener("pointermove", (event) => {
      // Line 784: Registers an event listener that connects browser input to the game control system.
        if (!pointer.active) return;
        // Line 785: Updates pointer-control state for mouse and touch steering.
        const p = pointerToWorld(event);
        // Line 786: Updates pointer-control state for mouse and touch steering.
        pointer.x = p.x;
        // Line 787: Updates pointer-control state for mouse and touch steering.
        pointer.y = p.y;
        // Line 788: Updates pointer-control state for mouse and touch steering.
      });
      // Line 789: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      window.addEventListener("pointerup", () => {
      // Line 791: Registers an event listener that connects browser input to the game control system.
        pointer.active = false;
        // Line 792: Updates pointer-control state for mouse and touch steering.
      });
      // Line 793: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.

      restartButton.addEventListener("click", startRun);
      // Line 795: Creates the start/restart control bound to the game initialization routine.

      resetPlayer();
      // Line 797: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      seedDrones();
      // Line 798: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      spawnShards();
      // Line 799: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      draw();
      // Line 800: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
      updateUi();
      // Line 801: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
    })();
    // Line 802: Executes part of the self-contained JavaScript game system that drives simulation, state, or rendering.
  </script>
  <!-- Line 803: Closes the embedded JavaScript block after all systems, listeners, and boot logic are registered. -->
</body>
<!-- Line 804: Closes the visible document body after the game application has been declared. -->
</html>
<!-- Line 805: Closes the HTML document and completes the single-file application. -->

```
