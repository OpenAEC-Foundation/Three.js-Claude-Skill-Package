# threejs-impl-shadows -- Methods Reference

## LightShadow (Base Class)

All shadow-capable lights (DirectionalLight, SpotLight, PointLight) expose a `.shadow` property inheriting from `LightShadow`.

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.autoUpdate` | `boolean` | `true` | If `true`, shadow map re-renders every frame |
| `.bias` | `number` | `0` | Depth offset to reduce shadow acne; typical range `-0.005` to `0.001` |
| `.normalBias` | `number` | `0` | Offset along surface normal; reduces acne on curved surfaces |
| `.mapSize` | `Vector2` | `(512, 512)` | Shadow map resolution; MUST be power-of-two |
| `.radius` | `number` | `1` | Blur radius for VSMShadowMap; IGNORED by PCF and PCFSoft |
| `.blurSamples` | `number` | `8` | Number of blur samples; ONLY used by VSMShadowMap |
| `.intensity` | `number` | `1` | Shadow darkness; `0` = invisible, `1` = fully opaque |
| `.map` | `WebGLRenderTarget \| null` | `null` | Auto-generated depth texture; readonly |
| `.camera` | `Camera` | varies | Virtual camera used to render the shadow map |
| `.needsUpdate` | `boolean` | `false` | Set `true` to force re-render when `autoUpdate` is `false` |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `.clone()` | `(): LightShadow` | Returns a deep copy |
| `.copy(source)` | `(source: LightShadow): this` | Copies properties from source |
| `.dispose()` | `(): void` | Releases GPU resources (shadow map texture) |
| `.getFrustum()` | `(): Frustum` | Returns the shadow camera frustum |
| `.updateMatrices(light)` | `(light: Light): void` | Recalculates shadow camera matrices |
| `.toJSON()` | `(): Object` | Serializes to JSON |

---

## DirectionalLightShadow

Extends `LightShadow`. Uses an `OrthographicCamera`.

### Shadow Camera Properties (OrthographicCamera)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.camera.left` | `number` | `-5` | Left frustum boundary |
| `.camera.right` | `number` | `5` | Right frustum boundary |
| `.camera.top` | `number` | `5` | Top frustum boundary |
| `.camera.bottom` | `number` | `-5` | Bottom frustum boundary |
| `.camera.near` | `number` | `0.5` | Near clipping plane |
| `.camera.far` | `number` | `500` | Far clipping plane |

### Type Check

| Property | Value |
|----------|-------|
| `.isDirectionalLightShadow` | `true` (readonly) |

---

## SpotLightShadow

Extends `LightShadow`. Uses a `PerspectiveCamera` auto-configured from the SpotLight.

### Additional Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.focus` | `number` | `1` | Shadow camera FOV relative to spotlight FOV; range `[0, 1]` |
| `.aspect` | `number` | `1` | Shadow texture aspect ratio |

### Type Check

| Property | Value |
|----------|-------|
| `.isSpotLightShadow` | `true` (readonly) |

**Note**: The shadow camera perspective is auto-calculated from `spotLight.angle` and `spotLight.distance`. Manual frustum configuration is RARELY needed.

---

## PointLightShadow

Extends `LightShadow`. Uses 6 `PerspectiveCamera` instances (cubemap faces).

### Type Check

| Property | Value |
|----------|-------|
| `.isPointLightShadow` | `true` (readonly) |

**Note**: The shadow camera `.near` and `.far` can be configured. All 6 face cameras share the same near/far. FOV is fixed at 90 degrees per face.

---

## Shadow Map Type Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `THREE.BasicShadowMap` | `0` | No filtering; hard aliased edges |
| `THREE.PCFShadowMap` | `1` | Percentage-Closer Filtering; fixed kernel |
| `THREE.PCFSoftShadowMap` | `2` | Variable kernel PCF; softer edges |
| `THREE.VSMShadowMap` | `3` | Variance Shadow Maps; Gaussian blur |

Set via: `renderer.shadowMap.type = THREE.PCFSoftShadowMap;`

**ALWAYS** set the shadow map type BEFORE creating any shadow maps. Changing the type at runtime requires disposing all existing shadow maps.

---

## WebGLRenderer Shadow Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `renderer.shadowMap.enabled` | `boolean` | `false` | Master shadow toggle |
| `renderer.shadowMap.autoUpdate` | `boolean` | `true` | Auto re-render shadows each frame |
| `renderer.shadowMap.needsUpdate` | `boolean` | `false` | Force single shadow update |
| `renderer.shadowMap.type` | `number` | `THREE.PCFShadowMap` | Shadow map filtering algorithm |

---

## Object3D Shadow Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.castShadow` | `boolean` | `false` | Object casts shadows onto other objects |
| `.receiveShadow` | `boolean` | `false` | Object receives shadows from other objects |
| `.customDepthMaterial` | `Material \| null` | `null` | Custom material for shadow depth pass |
| `.customDistanceMaterial` | `Material \| null` | `null` | Custom material for PointLight shadow distance pass |

---

## CameraHelper

Visualizes any camera's frustum, including shadow cameras.

### Constructor

```js
new THREE.CameraHelper(camera: Camera)
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `.camera` | `Camera` | The camera being visualized |
| `.pointMap` | `Object` | Maps frustum point names to indices |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `.update()` | `(): void` | Refreshes the helper geometry; call after camera changes |
| `.dispose()` | `(): void` | Releases resources |

### Usage for Shadow Debugging

```js
const helper = new THREE.CameraHelper(light.shadow.camera);
scene.add(helper);

// ALWAYS call update after modifying shadow camera properties
light.shadow.camera.left = -30;
light.shadow.camera.updateProjectionMatrix();
helper.update();
```
