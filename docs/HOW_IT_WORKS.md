# How this project actually works

This is the "under the hood" doc — a walkthrough of how the DC motor model was built, how the animation and controls work, and how the AR mode was implemented (including the bugs that had to get fixed along the way). It's written for anyone who opens the code and wonders "wait, how does *this* part work?"

The whole thing is **one HTML file**. No build step, no npm install, no framework — just Three.js loaded from a CDN, and a big inline `<script>` that does everything. That was a deliberate choice: it makes the project trivially deployable (drag one file onto Netlify) and easy to read top-to-bottom.

---

## 1. The 3D scene, in plain terms

Three.js needs three basic things to show anything: a **scene** (a container for objects), a **camera** (a viewpoint), and a **renderer** (something that draws the scene from the camera's point of view onto a `<canvas>`).

```js
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(42, window.innerWidth/window.innerHeight, 0.05, 100);
const renderer = new THREE.WebGLRenderer({ antialias:true, alpha:true });
```

`alpha:true` matters a lot later — it's what lets the AR camera feed show through instead of a solid background.

The camera doesn't move on rails; it orbits around the model. That's done manually with **spherical coordinates** (radius, theta, phi) rather than pulling in Three's `OrbitControls` add-on — dragging the mouse/finger adjusts `theta`/`phi`, scrolling adjusts `radius`, and every frame the camera's actual x/y/z position gets recomputed from those three numbers and pointed back at the origin. This keeps the whole file dependency-free.

---

## 2. Building the motor itself

Nothing in the model is an imported 3D file (no `.glb`/`.obj`) — every part is built from Three.js's primitive geometries and grouped together:

- **Housing** — a half-open cylinder (`THREE.CylinderGeometry` with a partial angle) acting as the outer stator casing.
- **N pole / S pole** — two curved shell segments (also partial cylinders) colored red and blue, built by a shared `makePole()` helper so both poles are constructed identically just mirrored.
- **Shaft** — a thin cylinder running through the center.
- **Armature coil** — a rectangular loop made from two "sides" (`sideA`, `sideB`, the current-carrying conductors) plus connecting bars, all parented to a `rotorGroup` so they can spin together as one unit.
- **Commutator / brushes** — small cylinder segments (`commSegA`, `commSegB`) that swap which one is "live" as the coil rotates, plus static brush contacts.
- **DC source** — a simple box-shaped battery model with a `+`/`–` face.

Everything that should *not* spin (the housing, poles, DC source, brushes) sits in one group; everything that *should* spin (the coil, shaft, commutator segments) sits inside `rotorGroup`. That separation is what makes the animation loop trivial — spinning the motor is just rotating one group, not repositioning dozens of individual meshes.

```js
const motorGroup = new THREE.Group();
const rotorGroup = new THREE.Group();
motorGroup.add(staticParts..., rotorGroup);
```

---

## 3. Making it move: the animation loop

Three.js calls one function every frame via `renderer.setAnimationLoop(animate)`. Inside `animate()`:

```js
angle += dt * speed * 0.6;
rotorGroup.rotation.y = angle;
```

`angle` just keeps accumulating over time (scaled by the `speed` slider), and gets applied directly as the rotor's Y-rotation. Pausing the animation is as simple as skipping that line when `playing` is false — the rest of the scene still renders, just frozen.

### Commutator switching (the "half" logic)

A real DC motor's commutator flips which side of the coil is connected to the positive terminal every half-rotation — that's *why* the coil keeps turning the same direction instead of oscillating back and forth. This is modeled with one function:

```js
function currentHalf(){
  return Math.floor(angle / Math.PI) % 2;
}
```

`angle` is in radians, so `Math.PI` is half a turn. Every half-turn, `currentHalf()` flips between `0` and `1`, and that value drives which side of the coil is drawn red (current flowing "in") vs blue ("current flowing out"), and which commutator segment is highlighted:

```js
const colA = half===0 ? COL_RED : COL_BLUE;
const colB = half===0 ? COL_BLUE : COL_RED;
sideA.material.color.setHex(colA);
sideB.material.color.setHex(colB);
```

### Force vectors

The force arrows aren't just decorative — they're recomputed every frame from an actual cross product, mirroring the real F = I**L** × **B** relationship (simplified here to direction, not magnitude):

```js
wForce.set(0, omega, 0).cross(rVecA).normalize();
forceArrowA.position.copy(worldA);
forceArrowA.setDirection(wForce);
```

`rVecA` is the vector from the rotor's pivot to the current world position of that side of the coil, and crossing it with the rotation axis gives the direction the force arrow should point — so as the coil spins, the arrows visibly rotate with it and flip when the commutator switches, instead of being a static pre-baked animation.

### The circuit schematic

The little schematic panel (battery + coil loop) isn't an image — it's redrawn every frame onto a `<canvas>` with plain 2D drawing calls (`drawSchematic(half===0)`), recoloring the same way the 3D coil does, so the abstract circuit diagram and the 3D model always agree about which side is "hot" at any instant.

---

## 4. Cross-section, Explode, and Labels

**Cross-section** uses a Three.js **clipping plane** — a plane you assign to material(s), and Three.js simply doesn't render any geometry on the far side of it:

```js
const clipPlane = new THREE.Plane(new THREE.Vector3(0,0,1), 0);
material.clippingPlanes = [clipPlane]; // when the toggle is on
```

Toggling it off just sets `clippingPlanes = []` back to an empty array — the geometry itself never changes, only what's allowed to render.

**Explode** doesn't touch geometry either — it just nudges each part's `.position` outward from the center, scaled by a 0–1 slider value:

```js
poleN.position.x = -explode * 1.4;
poleS.position.x =  explode * 1.4;
housing.position.y = explode * 1.6;
```

At `explode = 0` everything is back at its normal assembled position.

**Labels** (SHAFT, N POLE, etc.) are actually regular HTML `<div>` elements sitting in a layer *on top of* the canvas, not 3D text. Every frame, each label's 3D anchor point gets projected into 2D screen coordinates using the camera's projection matrix:

```js
projVec.copy(labelPos3D).applyMatrix4(motorGroup.matrixWorld);
projVec.project(camera);           // now in normalized device coords, -1..1
const x = (projVec.x*0.5+0.5) * window.innerWidth;
const y = (-projVec.y*0.5+0.5) * window.innerHeight;
```

That's the standard trick for "3D-anchored HTML overlays" — cheaper than actual 3D text meshes, and it's why the labels always sit exactly on their part even as the model rotates.

---

## 5. How the interactive controls are wired up

Nothing fancy here — every slider/toggle is a plain HTML input with a `change`/`input` listener that flips a JS variable, which the render loop then reads every frame:

```js
document.getElementById('toggleField').addEventListener('change', function(e){
  showField = e.target.checked;
  fieldGroup.visible = showField;
});
```

This pattern repeats for Field lines, Force vectors, Labels, Cross-section, and Explode — one listener, one variable, one `if` check inside `animate()`.

---

## 6. Making it responsive (mobile layout)

The desktop layout has an always-open controls panel on the right and a title card top-left. On a phone that's way too much competing for a small screen, so a `@media (max-width: 640px)` block:

- shrinks the title card and hides its longer hint text,
- moves the AR button to the corner instead of dead-center (so it can't collide with the title),
- turns the controls panel into a **collapsible bottom sheet** — collapsed to just a "Controls ▲" strip by default, sliding up on tap via a CSS `transform: translateY(...)` toggle,
- turns labels off by default (there just isn't room for six floating text tags on a phone screen at once).

None of this needed a second codebase — it's the same HTML/JS, just re-laid-out under a media query, with one small JS check (`window.matchMedia('(max-width: 640px)')`) to set mobile-appropriate defaults on load.

---

## 7. How the AR mode actually works

This was the hardest part, and also where most of the real debugging happened, so it's worth walking through in order.

### Step 1 — checking if AR is even possible

```js
navigator.xr.isSessionSupported('immersive-ar').then(supported => { ... });
```

This has to run before anything else, because WebXR support is inconsistent: Android Chrome (with ARCore) supports it; **iOS Safari does not implement WebXR at all**. The app detects this and shows an honest message ("AR needs Chrome on Android") instead of a button that silently does nothing.

WebXR also flatly refuses to run outside a **secure context** — `https://` only. That's why local testing with a plain `http://` dev server never shows the AR button; it only works once deployed (or tunneled through something like `localtunnel`/`ngrok`).

### Step 2 — starting the session

```js
navigator.xr.requestSession('immersive-ar', {
  requiredFeatures: ['hit-test'],
  optionalFeatures: ['local-floor', 'dom-overlay'],
  domOverlay: { root: document.body }
});
```

Two features matter here:
- **`hit-test`** — lets the app ask "what real-world surface is the camera currently pointed at?" every frame.
- **`dom-overlay`** — without this, none of the regular HTML UI (the Exit button, the instruction text) would be visible on top of the live camera feed during the session. It's what lets ordinary `<div>`/`<button>` elements render over passthrough video.

### Step 3 — finding a surface and placing the model

Every frame during the session, the code asks WebXR for hit-test results against the center of the screen:

```js
const results = frame.getHitTestResults(hitTestSource);
if (results.length) {
  reticle.matrix.fromArray(results[0].getPose(refSpace).transform.matrix);
}
```

A small ring mesh (the "reticle") tracks that pose live, so as you move your phone around, the ring visibly slides across the floor/table wherever the camera is pointed. The current version auto-places the motor the *first* time a surface is found (no tap required) — then lets you tap anywhere afterward to move it to a new spot, using the same `reticle.matrix.decompose(...)` trick to copy the detected position/rotation onto the motor.

### Step 4 — the bug that took the longest to find

Early on, AR would open the camera fine, but the model would just... never appear. The cause turned out to be subtle: a fallback placement function was reading `camera.position` / `camera.quaternion` — the same `camera` object used for the desktop orbit view — assuming it would reflect the phone's live AR pose.

It doesn't. Three.js does **not** mutate the camera object you pass in during an XR session; internally it computes the real device pose and uses it only for rendering, without writing it back onto your own camera variable. So that fallback was placing the model relative to wherever the *desktop* camera had last been pointed — completely unrelated to where the phone was actually facing in the real world. The fix was to read the pose the correct way, directly from the current XR frame:

```js
const viewerPose = frame.getViewerPose(refSpace);
const pos = viewerPose.transform.position;
const quat = viewerPose.transform.orientation;
```

That's the only reliable source of truth for "where is the device really pointed, right now" during a WebXR session — a good lesson in not assuming a library mutates objects just because it's convenient if it did.

### Step 5 — graceful fallback

Not every device honors `hit-test` the same way. If a session request with `requiredFeatures: ['hit-test']` gets rejected, the code catches that and silently retries the session *without* requiring it, falling back to just placing the model a fixed distance in front of the viewer instead of surface-anchored. Better a slightly-less-precise placement than no AR at all.

---

## 8. Deployment

The finished file was deployed via **Netlify Drop** — literally dragging the folder onto Netlify's upload page — which gives a public `https://` URL instantly, satisfying WebXR's secure-context requirement with zero server configuration.

---

## TL;DR architecture summary

```
index.html
 ├── <style>            responsive layout, cover both desktop + mobile via media queries
 ├── <body> markup       canvas host, title card, controls panel, labels layer, info panel
 └── <script>
      ├── scene/camera/renderer setup
      ├── geometry construction (housing, poles, coil, shaft, commutator, battery)
      ├── UI event listeners (sliders/toggles → plain JS variables)
      ├── label projection (3D → 2D HTML overlay)
      ├── circuit schematic (2D canvas, redrawn per frame)
      ├── AR session management (capability check → request session → hit-test → place)
      └── animate() — the one function that ties it all together, called every frame
```
