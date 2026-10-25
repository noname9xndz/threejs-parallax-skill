---
name: threejs-parallax
description: Creates a 2.5D parallax effect using Three.js with layered transparent PNG assets on PlaneGeometry — pointer/touch/gyro-driven, scroll-driven, or camera-follow (web game backgrounds with infinite tiling). Use when adding parallax scenes, layered product imagery, interactive media placeholders, side-scroller backgrounds, or when the user mentions "parallax", "2.5D", "depth effect", or "layered images".
---

# Three.js 2.5D Parallax Effect

Renders stacked transparent PNGs on separate Three.js planes with pointer-driven parallax and scroll-driven layer motion. Background layers move least, foreground most. Layers can also drift on scroll (left/right/up) for added dynamism.

Two modes:
- **Media mode** (all sections up to "Cleanup"): a card/hero image reacting to pointer + scroll.
- **Game mode** (see "Camera-Follow Mode for Web Games"): side-scroller backgrounds that follow a moving camera and tile infinitely.

## Quick Start

1. Give the container element a unique `id`
2. Add a `<script type="module">` that builds the scene inside it
3. Define layers back-to-front with increasing `parallaxFactor` and `renderOrder`

## Import

```html
<script type="module">
import * as THREE from 'https://esm.sh/three@0.170.0';
```

Always pin an exact version. In a bundler project use `import * as THREE from 'three'` instead.

## Scene Setup

```javascript
const scene  = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(55, W / H, 0.1, 100);
camera.position.z = 5;

const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true });
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.setSize(W, H);
renderer.setClearColor(0x000000, 0);
renderer.outputColorSpace = THREE.SRGBColorSpace;
```

**Critical**: Always set `renderer.outputColorSpace = THREE.SRGBColorSpace` — without this, textures render washed out / brighter than the source PNGs.

## Texture Loading

```javascript
const loader  = new THREE.TextureLoader();
const loadTex = src => new Promise((res, rej) => loader.load(src, tex => {
  tex.colorSpace = THREE.SRGBColorSpace;
  res(tex);
}, undefined, rej));
```

**Critical**: Set `tex.colorSpace = THREE.SRGBColorSpace` on every texture to prevent gamma double-correction. Without both this AND `renderer.outputColorSpace`, colours will not match the original assets.

The loaded texture already knows its pixel size, so the aspect ratio can be computed instead of measured by hand:

```javascript
const aspectOf = tex => tex.image.width / tex.image.height;
// makeLayer(tex, SIZE, SIZE / aspectOf(tex), [X, Y, Z], FACTOR, ORDER)
```

## Layer Factory

```javascript
function makeLayer(tex, pw, ph, [bx, by, bz], parallaxAmt, order) {
  const geo  = new THREE.PlaneGeometry(pw, ph);
  const mat  = new THREE.MeshBasicMaterial({
    map: tex, transparent: true, depthWrite: false, depthTest: false,
  });
  const mesh = new THREE.Mesh(geo, mat);
  mesh.position.set(bx, by, bz);
  mesh.renderOrder      = order;
  mesh.userData.base    = [bx, by, bz];
  mesh.userData.pFactor = parallaxAmt;
  scene.add(mesh);
  return mesh;
}
```

### Parameters

| Param | Meaning |
|-------|---------|
| `pw` | Plane width in world units |
| `ph` | Plane height — always compute as `pw / aspectRatio` to preserve proportions |
| `[bx, by, bz]` | Base position (x = horizontal, y = vertical, z = depth) |
| `parallaxAmt` | How far the layer shifts per unit of mouse movement |
| `order` | `renderOrder` — lower draws first (back), higher draws last (front) |

After creating layers, assign scroll parallax factors via `userData`:

```javascript
layer.userData.scrollFX = 0;   // horizontal scroll drift (negative = left, positive = right)
layer.userData.scrollFY = 0;   // vertical scroll drift (positive = upward)
```

### Material rules
- `transparent: true` — required so PNG alpha works
- `depthWrite: false` + `depthTest: false` — prevents z-fighting between overlapping transparent planes
- `renderOrder` — explicitly controls draw order independent of z position

### Parallax factor guidelines

| Layer role | parallaxFactor | renderOrder |
|------------|---------------|-------------|
| Background | ~0.04–0.08 | 0 |
| Midground | ~0.15–0.25 | 1 |
| Foreground | ~0.30–0.45 | 2 |

More layers can be added by continuing the pattern. Larger `parallaxFactor` = more movement = feels closer.

## Layer Configuration

To define a layer, compute the aspect ratio from the asset's pixel dimensions:

```
aspectRatio = pixelWidth / pixelHeight
```

Then call:
```javascript
makeLayer(texture, SIZE, SIZE / aspectRatio, [X, Y, Z], PARALLAX_FACTOR, RENDER_ORDER)
```

### Example 1 — "AI Glasses" card (3 layers, assets spread diagonally)

```javascript
// bg.png:          896×680 → aspect 1.318
const bgLayer  = makeLayer(bgTex,  4.44, 4.44 / 1.318,  [ 0.28, -0.05, -1],   0.06, 0);

// MRB-display.png: 812×692 → aspect 1.174 — lower-right
const mrbLayer = makeLayer(mrbTex, 3.50, 3.50 / 1.174,  [ 0.90, -1.40, -0.5], 0.20, 1);

// RBM.png:         738×500 → aspect 1.476 — upper-left
const rbmLayer = makeLayer(rbmTex, 3.00, 3.00 / 1.476,  [-1.00,  1.20,  0],   0.38, 2);

// Scroll: foreground layers spread apart diagonally
bgLayer.userData.scrollFX  =  0.00; bgLayer.userData.scrollFY  =  0.06;
mrbLayer.userData.scrollFX =  0.45; mrbLayer.userData.scrollFY = -0.20;  // drifts right + down
rbmLayer.userData.scrollFX = -0.40; rbmLayer.userData.scrollFY =  0.30;  // drifts left + up
```

### Example 2 — "Quest Cash" card (4 layers, characters + floating object)

```javascript
// bg_2.png:      1320×990 → aspect 1.333
const bgLayer     = makeLayer(bgTex,     4.80, 4.80 / 1.333,  [ 0.00,  0.00, -1],   0.06, 0);

// person 1.png:   818×1186 → aspect 0.69 — LEFT character
const personLayer = makeLayer(personTex, 3.23, 3.23 / 0.69,   [-1.40,  0.60, -0.9], 0.20, 1);

// TAYA user.png:  854×1140 → aspect 0.749 — RIGHT character
const tayaLayer   = makeLayer(tayaTex,   2.64, 2.64 / 0.749,  [ 1.40, -0.05,  0.5], 0.20, 2);

// wallet.png:     866×868 → aspect 1.0 — floating foreground object
const walletLayer = makeLayer(walletTex, 2.40, 2.40 / 1.0,    [-0.05,  0.80,  0.0], 0.38, 3);

// Scroll: characters split left/right, wallet floats up
bgLayer.userData.scrollFX     =  0.00; bgLayer.userData.scrollFY     =  0.06;
personLayer.userData.scrollFX = -0.55; personLayer.userData.scrollFY =  0.00;  // drifts left
tayaLayer.userData.scrollFX   =  0.55; tayaLayer.userData.scrollFY   =  0.00;  // drifts right
walletLayer.userData.scrollFX =  0.00; walletLayer.userData.scrollFY =  0.36;  // floats up
```

## Pointer Tracking & Parallax Motion

Use `pointermove` (mouse, pen and touch) rather than `mousemove`, so the effect also works on tablets and phones.

```javascript
let tX = 0, tY = 0, cX = 0, cY = 0;

const onPointerMove = e => {
  tX =  (e.clientX / window.innerWidth  - 0.5) * 2;
  tY = -(e.clientY / window.innerHeight - 0.5) * 2;
};
window.addEventListener('pointermove', onPointerMove, { passive: true });
```

Optional — tilt on phones (iOS requires a user gesture and `DeviceOrientationEvent.requestPermission()` before events arrive):

```javascript
window.addEventListener('deviceorientation', e => {
  if (e.gamma == null) return;
  tX = THREE.MathUtils.clamp(e.gamma / 30, -1, 1);       // left/right tilt
  tY = THREE.MathUtils.clamp((e.beta - 45) / 30, -1, 1); // forward/back tilt
}, { passive: true });
```

```javascript
function animate() {
  requestAnimationFrame(animate);

  cX += (tX - cX) * 0.065;
  cY += (tY - cY) * 0.065;

  camera.rotation.y = cX * 0.035;
  camera.rotation.x = cY * 0.025;

  for (const layer of layers) {
    const [bx, by] = layer.userData.base;
    const f        = layer.userData.pFactor;
    layer.position.x = bx + cX * f;
    layer.position.y = by + cY * f;
  }

  renderer.render(scene, camera);
}
animate();
```

### Key values

| Value | Purpose | Tune for |
|-------|---------|----------|
| `0.065` | Lerp smoothing factor | Lower = more lag/smooth, higher = snappier |
| `0.035` | Camera Y rotation multiplier | Perspective lean amount (horizontal) |
| `0.025` | Camera X rotation multiplier | Perspective lean amount (vertical) |

## Performance & Accessibility (required)

The basic loop renders every frame forever, even when the section is off-screen. Always add these:

```javascript
// 1. Respect reduced-motion: keep the static composition, skip motion
const reduceMotion = matchMedia('(prefers-reduced-motion: reduce)');

// 2. Only animate while the container is visible
let visible = true, rafId = 0;
const io = new IntersectionObserver(([entry]) => {
  visible = entry.isIntersecting;
  if (visible && !rafId) rafId = requestAnimationFrame(animate);
});
io.observe(container);

function animate() {
  if (!visible) { rafId = 0; return; }       // stop the loop off-screen
  rafId = requestAnimationFrame(animate);
  if (reduceMotion.matches) { tX = tY = 0; scrollProg = 0; }
  // ... lerp + layer updates as above ...
  renderer.render(scene, camera);
}
```

- `renderer.setPixelRatio(Math.min(devicePixelRatio, 2))` — never render above 2× on high-DPI phones.
- Compress large PNGs (or use WebP/KTX2) — layer textures are usually the biggest cost.
- With many parallax cards on one page, share one renderer or limit to the cards on screen; browsers cap the number of live WebGL contexts (~16).

## Scroll-Driven Layer Motion

Adds subtle per-layer drift as the section scrolls through the viewport, making the page feel more dynamic. Each layer can move independently on X and/or Y based on scroll progress.

### Scroll Progress Tracking

Requires a reference to the media container element (`mediaEl`) to compute how far through the viewport the section has scrolled.

```javascript
let scrollProg = 0, smoothScrollProg = 0;

function updateScrollProg() {
  const rect = mediaEl.getBoundingClientRect();
  const vh   = window.innerHeight;
  const mid  = rect.top + rect.height * 0.5;
  scrollProg = (vh * 0.5 - mid) / (vh * 0.5 + rect.height * 0.5);
  scrollProg = Math.max(-1.4, Math.min(1.4, scrollProg));
}

window.addEventListener('scroll', updateScrollProg, { passive: true });
updateScrollProg();
```

`scrollProg` ranges from **−1** (section fully below viewport center) through **0** (centered) to **+1** (section above center). Clamped to ±1.4 to allow slight overshoot for a natural feel.

### Assigning Scroll Factors

After creating layers, set `scrollFX` (horizontal) and `scrollFY` (vertical) on each:

```javascript
bgLayer.userData.scrollFX     =  0.00; bgLayer.userData.scrollFY     =  0.06;
personLayer.userData.scrollFX = -0.55; personLayer.userData.scrollFY =  0.00;  // drifts left
tayaLayer.userData.scrollFX   =  0.55; tayaLayer.userData.scrollFY   =  0.00;  // drifts right
walletLayer.userData.scrollFX =  0.00; walletLayer.userData.scrollFY =  0.36;  // floats up
```

### Scroll factor guidelines

| Motion intent | scrollFX | scrollFY |
|---------------|----------|----------|
| Subtle background drift | 0 | ~0.04–0.08 |
| Character drifts left | −0.3 to −0.6 | 0 |
| Character drifts right | +0.3 to +0.6 | 0 |
| Object floats upward | 0 | +0.2 to +0.4 |
| No scroll motion | 0 | 0 |

Negative `scrollFX` moves the layer leftward as the section scrolls up; positive moves it rightward. Positive `scrollFY` moves the layer upward.

### Updated Animate Loop

The animate loop must lerp `smoothScrollProg` and add scroll offsets alongside mouse parallax:

```javascript
function animate() {
  requestAnimationFrame(animate);
  cX += (tX - cX) * 0.065;
  cY += (tY - cY) * 0.065;
  smoothScrollProg += (scrollProg - smoothScrollProg) * 0.06;

  camera.rotation.y = cX * 0.035;
  camera.rotation.x = cY * 0.025;

  for (const layer of layers) {
    const [bx, by] = layer.userData.base;
    const f  = layer.userData.pFactor;
    const sx = layer.userData.scrollFX || 0;
    const sy = layer.userData.scrollFY || 0;
    layer.position.x = bx + cX * f + sx * smoothScrollProg;
    layer.position.y = by + cY * f + sy * smoothScrollProg;
  }

  renderer.render(scene, camera);
}
animate();
```

### Scroll motion key values

| Value | Purpose | Tune for |
|-------|---------|----------|
| `0.06` | Scroll lerp smoothing | Lower = smoother/laggier, higher = tighter tracking |
| `±1.4` | scrollProg clamp | Allows slight overshoot beyond the centered range for softer edges |

## Container Integration

The parallax canvas must fill its container element. Use this pattern:

```javascript
(async () => {
  await new Promise(r => requestAnimationFrame(r)); // wait for layout

  const container = document.getElementById('YOUR_ID');
  if (!container) return;

  const canvas = document.createElement('canvas');
  canvas.style.cssText = 'position:absolute;inset:0;width:100%;height:100%;display:block;';
  container.appendChild(canvas);

  let W = container.offsetWidth;
  let H = container.offsetHeight;

  // ... scene setup using W, H ...

  const ro = new ResizeObserver(() => {
    W = container.offsetWidth;
    H = container.offsetHeight;
    camera.aspect = W / H;
    camera.updateProjectionMatrix();
    renderer.setSize(W, H);
  });
  ro.observe(container);
})();
```

### HTML structure

The container needs `position: relative` (or `absolute`) and a known size. Typical setup inside a project card:

```html
<div class="project-media" id="my-parallax-media">
  <div class="media-placeholder" id="my-parallax"></div>
</div>
```

With CSS:
```css
.project-media { position: relative; overflow: hidden; }
.media-placeholder { position: absolute; inset: 0; width: 100%; height: 100%; }
```

Set `background: transparent` on the media wrapper if the card background should show through the alpha channel.

## Cleanup

When the container is removed (SPA route change, React/Vue unmount), free GPU memory and listeners:

```javascript
function destroy() {
  cancelAnimationFrame(rafId);
  io.disconnect();
  ro.disconnect();
  window.removeEventListener('scroll', updateScrollProg);
  window.removeEventListener('pointermove', onPointerMove);
  for (const layer of layers) {
    layer.geometry.dispose();
    layer.material.map?.dispose();
    layer.material.dispose();
  }
  renderer.dispose();
  canvas.remove();
}
```

## Camera-Follow Mode for Web Games

For side-scrollers the parallax follows the game camera, not the pointer. Rule: `layerOffset = cameraPos × factor` (sky ~0–0.1, far hills ~0.2–0.3, mid ~0.5–0.7, gameplay 1.0, foreground 1.2–1.5).

### Perspective camera (2.5D)

Place planes at different `z`; moving `camera.position.x` gives parallax for free. Scale far planes so they still cover the view:

```javascript
function coverScale(z, planeH) {
  const dist = camera.position.z - z;
  const visH = 2 * Math.tan(THREE.MathUtils.degToRad(camera.fov / 2)) * dist;
  return visH / planeH;
}
```

With `depthTest: false`, draw order comes only from `renderOrder` — keep it consistent with `z`.

### Orthographic camera (pure 2D)

No natural parallax, so offset each layer every frame:

```javascript
for (const L of layers) {
  L.mesh.position.x = camera.position.x * (1 - L.factor);
  L.mesh.position.y = camera.position.y * (1 - L.factorY); // factorY < factor
}
```

### Infinite tiling via texture offset

One screen-sized plane per layer that rides with the camera; only the UVs scroll:

```javascript
tex.wrapS = THREE.RepeatWrapping;
// each frame
L.mesh.position.x = camera.position.x;
tex.offset.x = (camera.position.x * L.factor) / L.tileWorldWidth;
```

Assets must be seamlessly tileable horizontally. For autoscroll (clouds, vehicle levels) add `+= speed * dt` to `tex.offset.x`.

### Pixel art

```javascript
tex.magFilter = THREE.NearestFilter;
tex.minFilter = THREE.NearestFilter;
tex.generateMipmaps = false;
// snap to whole texels to avoid shimmering
L.mesh.position.x = Math.round(L.mesh.position.x * PPU) / PPU; // PPU = pixels per world unit
```

## Checklist for Adding a New Parallax Scene

1. Prepare transparent PNG assets (compressed; tileable if used in game mode)
2. Compute aspect ratio for each: `width / height` — or use `aspectOf(tex)` at runtime (macOS: `sips -g pixelWidth -g pixelHeight`; Windows PowerShell / Linux: `magick identify file.png`)
3. Add unique `id` to the container div
4. Add `<script type="module">` block following the pattern above
5. Define layers back-to-front with increasing `parallaxFactor` and `renderOrder`
6. Assign `scrollFX` / `scrollFY` on each layer for scroll-driven drift (set to `0` for no scroll motion)
7. Add `updateScrollProg()` listener and use `smoothScrollProg` in the animate loop
8. Set both `renderer.outputColorSpace` and each `tex.colorSpace` to `THREE.SRGBColorSpace`
9. Test pointer interaction (mouse and touch) — adjust `parallaxFactor` values and positions as needed
10. Test scroll interaction — scroll past the section and verify layers drift subtly in the intended directions
11. Add the IntersectionObserver pause and `prefers-reduced-motion` check; verify the loop stops when scrolled away (DevTools → Performance)
12. Wire up `destroy()` if the page can unmount the scene
