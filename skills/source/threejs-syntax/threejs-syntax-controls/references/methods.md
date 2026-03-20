# threejs-syntax-controls -- Methods Reference

## OrbitControls

**Import:** `import { OrbitControls } from 'three/addons/controls/OrbitControls.js'`

**Constructor:**
```typescript
new OrbitControls(camera: THREE.Camera, domElement: HTMLElement)
```

### Properties (Complete)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `autoRotate` | `boolean` | `false` | Auto-rotate around target |
| `autoRotateSpeed` | `number` | `2.0` | Rotation speed (degrees/sec at 60fps) |
| `cursor` | `string` | `'auto'` | CSS cursor style |
| `dampingFactor` | `number` | `0.05` | Inertia factor (0-1), requires `enableDamping` |
| `domElement` | `HTMLElement` | -- | Event listener target (read-only after construct) |
| `enabled` | `boolean` | `true` | Enable/disable all interaction |
| `enableDamping` | `boolean` | `false` | Smooth inertial movement |
| `enablePan` | `boolean` | `true` | Allow panning |
| `enableRotate` | `boolean` | `true` | Allow rotation |
| `enableZoom` | `boolean` | `true` | Allow zooming |
| `keys` | `{ LEFT: string, UP: string, RIGHT: string, BOTTOM: string }` | Arrow key codes | Keyboard pan keys |
| `maxAzimuthAngle` | `number` | `Infinity` | Max horizontal angle (radians) |
| `maxDistance` | `number` | `Infinity` | Max dolly distance (PerspectiveCamera only) |
| `maxPolarAngle` | `number` | `Math.PI` | Max vertical angle (radians) |
| `maxTargetRadius` | `number` | `Infinity` | Max distance target can move from origin |
| `maxZoom` | `number` | `Infinity` | Max zoom factor (OrthographicCamera only) |
| `minAzimuthAngle` | `number` | `-Infinity` | Min horizontal angle (radians) |
| `minDistance` | `number` | `0` | Min dolly distance (PerspectiveCamera only) |
| `minPolarAngle` | `number` | `0` | Min vertical angle (radians) |
| `minTargetRadius` | `number` | `0` | Min distance target can move from origin |
| `minZoom` | `number` | `0` | Min zoom factor (OrthographicCamera only) |
| `mouseButtons` | `{ LEFT: MOUSE, MIDDLE: MOUSE, RIGHT: MOUSE }` | `{ LEFT: ROTATE, MIDDLE: DOLLY, RIGHT: PAN }` | Mouse button mapping |
| `panSpeed` | `number` | `1.0` | Pan speed multiplier |
| `rotateSpeed` | `number` | `1.0` | Rotation speed multiplier |
| `screenSpacePanning` | `boolean` | `true` | `true`: pan in screen plane. `false`: pan in horizontal plane |
| `target` | `THREE.Vector3` | `(0, 0, 0)` | Orbit focus point |
| `touches` | `{ ONE: TOUCH, TWO: TOUCH }` | `{ ONE: ROTATE, TWO: DOLLY_PAN }` | Touch gesture mapping |
| `zoomSpeed` | `number` | `1.0` | Zoom speed multiplier |
| `zoomToCursor` | `boolean` | `false` | Zoom towards cursor position instead of target |

### Methods (Complete)

```typescript
update(deltaTime?: number): boolean
```
Update controls. Returns `true` if camera position changed. MUST call every frame when `enableDamping` or `autoRotate` is `true`. The optional `deltaTime` parameter enables framerate-independent damping.

```typescript
dispose(): void
```
Remove ALL event listeners from `domElement`. ALWAYS call on cleanup.

```typescript
saveState(): void
```
Save the current camera `position`, `target`, and `zoom` as the reset state.

```typescript
reset(): void
```
Reset camera to the last `saveState()` values. If `saveState()` was never called, resets to construction-time values.

```typescript
getDistance(): number
```
Returns the current distance from the camera to the target.

```typescript
getPolarAngle(): number
```
Returns the current vertical angle in radians (0 = top, Math.PI = bottom).

```typescript
getAzimuthalAngle(): number
```
Returns the current horizontal angle in radians.

```typescript
listenToKeyEvents(domElement: HTMLElement): void
```
Enable keyboard-based panning using arrow keys. Pass `document` or a focusable element.

```typescript
stopListenToKeyEvents(): void
```
Disable keyboard panning.

### Events

| Event | Properties | Trigger |
|-------|-----------|---------|
| `change` | none | Camera position or target changed |
| `start` | none | User interaction began |
| `end` | none | User interaction ended |

---

## MapControls

**Import:** `import { MapControls } from 'three/addons/controls/MapControls.js'`

**Constructor:**
```typescript
new MapControls(camera: THREE.Camera, domElement: HTMLElement)
```

Subclass of OrbitControls. ALL properties and methods are inherited. The ONLY differences:

| Property | OrbitControls Default | MapControls Default |
|----------|----------------------|---------------------|
| `mouseButtons.LEFT` | `ROTATE` | `PAN` |
| `mouseButtons.RIGHT` | `PAN` | `ROTATE` |
| `screenSpacePanning` | `true` | `true` |

---

## FlyControls

**Import:** `import { FlyControls } from 'three/addons/controls/FlyControls.js'`

**Constructor:**
```typescript
new FlyControls(camera: THREE.Camera, domElement: HTMLElement)
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `movementSpeed` | `number` | `1.0` | Translation speed |
| `rollSpeed` | `number` | `0.005` | Roll rotation speed |
| `dragToLook` | `boolean` | `false` | `true`: mouse drag to rotate. `false`: mouse always rotates |
| `autoForward` | `boolean` | `false` | Automatic forward movement |

### Methods

```typescript
update(delta: number): void
```
Update camera position/rotation. **ALWAYS** pass `delta` from `clock.getDelta()`. NEVER omit this argument.

```typescript
dispose(): void
```
Remove all event listeners.

### Keyboard Controls

| Key | Action |
|-----|--------|
| W | Move forward |
| S | Move backward |
| A | Move left |
| D | Move right |
| R | Move up |
| F | Move down |
| Q | Roll left |
| E | Roll right |

---

## PointerLockControls

**Import:** `import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js'`

**Constructor:**
```typescript
new PointerLockControls(camera: THREE.Camera, domElement: HTMLElement)
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `isLocked` | `boolean` | `false` (read-only) | Whether pointer is currently locked |
| `maxPolarAngle` | `number` | `Math.PI` | Max vertical look angle (radians) |
| `minPolarAngle` | `number` | `0` | Min vertical look angle (radians) |
| `pointerSpeed` | `number` | `1.0` | Mouse sensitivity multiplier |

### Methods

```typescript
lock(): void
```
Request pointer lock from the browser. **MUST** be called from a user gesture (click handler). Browsers reject unprompted lock requests.

```typescript
unlock(): void
```
Exit pointer lock.

```typescript
connect(): void
```
Attach pointer lock event listeners to `domElement`.

```typescript
disconnect(): void
```
Remove pointer lock event listeners.

```typescript
dispose(): void
```
Full cleanup. Calls `disconnect()` internally.

```typescript
getObject(): THREE.Camera
```
Returns the controlled camera.

```typescript
getDirection(target: THREE.Vector3): THREE.Vector3
```
Write the camera's look direction into `target` and return it.

```typescript
moveForward(distance: number): void
```
Move camera forward along the XZ plane (y stays constant).

```typescript
moveRight(distance: number): void
```
Move camera sideways along the XZ plane.

### Events

| Event | Trigger |
|-------|---------|
| `change` | Camera orientation changed |
| `lock` | Pointer lock acquired |
| `unlock` | Pointer lock released |

---

## TransformControls

**Import:** `import { TransformControls } from 'three/addons/controls/TransformControls.js'`

**Constructor:**
```typescript
new TransformControls(camera: THREE.Camera, domElement: HTMLElement)
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `axis` | `string \| null` | `null` | Constrain to axis: `'X'`, `'Y'`, `'Z'`, `'XY'`, `'YZ'`, `'XZ'` |
| `dragging` | `boolean` | `false` (read-only) | Whether user is currently dragging |
| `enabled` | `boolean` | `true` | Enable/disable interaction |
| `mode` | `string` | `'translate'` | `'translate'`, `'rotate'`, or `'scale'` |
| `object` | `THREE.Object3D` | `undefined` | Currently attached object |
| `showX` | `boolean` | `true` | Show X axis gizmo |
| `showY` | `boolean` | `true` | Show Y axis gizmo |
| `showZ` | `boolean` | `true` | Show Z axis gizmo |
| `size` | `number` | `1` | Gizmo visual scale |
| `space` | `string` | `'world'` | `'world'` or `'local'` coordinate space |
| `translationSnap` | `number \| null` | `null` | Snap to grid (world units) |
| `rotationSnap` | `number \| null` | `null` | Snap rotation (radians) |
| `scaleSnap` | `number \| null` | `null` | Snap scale increment |

### Methods

```typescript
attach(object: THREE.Object3D): TransformControls
```
Attach the gizmo to an object. Returns `this` for chaining.

```typescript
detach(): TransformControls
```
Remove the gizmo from the current object. Returns `this`.

```typescript
setMode(mode: string): void
```
Set the transform mode: `'translate'`, `'rotate'`, or `'scale'`.

```typescript
setSpace(space: string): void
```
Set coordinate space: `'world'` or `'local'`. NOTE: `'local'` has no effect in `'scale'` mode -- scale ALWAYS operates in local space.

```typescript
setSize(size: number): void
```
Set the gizmo visual scale.

```typescript
setTranslationSnap(snap: number | null): void
```
Set position snapping. Pass `null` to disable.

```typescript
setRotationSnap(snap: number | null): void
```
Set rotation snapping. Pass `null` to disable.

```typescript
setScaleSnap(snap: number | null): void
```
Set scale snapping. Pass `null` to disable.

```typescript
getRaycaster(): THREE.Raycaster
```
Returns the internal raycaster for custom intersection logic.

```typescript
getMode(): string
```
Returns the current mode.

```typescript
dispose(): void
```
Remove all event listeners and clean up.

### Events

| Event | Properties | Trigger |
|-------|-----------|---------|
| `change` | none | Gizmo visual state changed |
| `dragging-changed` | `event.value: boolean` | Drag started (`true`) or ended (`false`) |
| `objectChange` | none | Attached object's transform was modified by the gizmo |
| `mouseDown` | none | Pointer pressed on gizmo handle |
| `mouseUp` | none | Pointer released from gizmo handle |

---

## ArcballControls

**Import:** `import { ArcballControls } from 'three/addons/controls/ArcballControls.js'`

**Constructor:**
```typescript
new ArcballControls(camera: THREE.Camera, domElement: HTMLElement, scene?: THREE.Scene)
```

Advanced trackball rotation with no polar constraint. Supports animation states and cursor-based rotation scaling. The optional `scene` parameter enables focus animations.

Key methods: `update()`, `dispose()`, `setGizmosVisible(visible)`, `reset()`.

---

## TrackballControls

**Import:** `import { TrackballControls } from 'three/addons/controls/TrackballControls.js'`

**Constructor:**
```typescript
new TrackballControls(camera: THREE.Camera, domElement: HTMLElement)
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `rotateSpeed` | `number` | `1.0` | Rotation speed |
| `zoomSpeed` | `number` | `1.2` | Zoom speed |
| `panSpeed` | `number` | `0.3` | Pan speed |
| `noRotate` | `boolean` | `false` | Disable rotation |
| `noZoom` | `boolean` | `false` | Disable zooming |
| `noPan` | `boolean` | `false` | Disable panning |
| `staticMoving` | `boolean` | `false` | No inertia when `true` |
| `dynamicDampingFactor` | `number` | `0.2` | Inertia factor |

Key methods: `update()`, `dispose()`, `reset()`.

---

## DragControls

**Import:** `import { DragControls } from 'three/addons/controls/DragControls.js'`

**Constructor:**
```typescript
new DragControls(objects: THREE.Object3D[], camera: THREE.Camera, domElement: HTMLElement)
```

### Events

| Event | Trigger |
|-------|---------|
| `dragstart` | Drag started on an object |
| `drag` | Object is being dragged |
| `dragend` | Drag ended |
| `hoveron` | Pointer entered an object |
| `hoveroff` | Pointer left an object |

Key methods: `activate()`, `deactivate()`, `dispose()`, `getObjects()`, `getRaycaster()`.
