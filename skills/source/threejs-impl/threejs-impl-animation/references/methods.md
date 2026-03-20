# threejs-impl-animation — Method Reference

> Complete API signatures for the Three.js Animation System (r160+).

---

## AnimationMixer

### Constructor

```ts
new AnimationMixer(rootObject: Object3D): AnimationMixer
```

### Properties

```ts
mixer.time: number           // Global mixer time in seconds (default: 0)
mixer.timeScale: number      // Global speed multiplier (default: 1; 0 pauses all)
```

### Methods

```ts
mixer.clipAction(
  clip: AnimationClip,
  optionalRoot?: Object3D,
  blendMode?: number
): AnimationAction
// Returns (or creates and caches) an AnimationAction for the given clip.
// blendMode: THREE.NormalAnimationBlendMode | THREE.AdditiveAnimationBlendMode

mixer.existingAction(
  clip: AnimationClip,
  optionalRoot?: Object3D
): AnimationAction | null
// Returns a previously created action or null.

mixer.update(deltaTimeInSeconds: number): AnimationMixer
// Advances the mixer. MUST call every frame with clock.getDelta().

mixer.setTime(timeInSeconds: number): AnimationMixer
// Sets the global mixer time and forces an update of all active actions.

mixer.stopAllAction(): AnimationMixer
// Deactivates all currently scheduled actions.

mixer.getRoot(): Object3D
// Returns the root object passed to the constructor.

mixer.uncacheAction(clip: AnimationClip, optionalRoot?: Object3D): void
// Deallocates the cached action for the given clip.

mixer.uncacheClip(clip: AnimationClip): void
// Deallocates all cached data for the given clip.

mixer.uncacheRoot(root: Object3D): void
// Deallocates all cached data for the given root object.
```

### Events

```ts
mixer.addEventListener('finished', (event: {
  action: AnimationAction,
  direction: number,
  type: 'finished'
}) => void)
// Fires when an action with LoopOnce and clampWhenFinished=true completes.

mixer.addEventListener('loop', (event: {
  action: AnimationAction,
  loopDelta: number,
  type: 'loop'
}) => void)
// Fires when an action completes a loop iteration.
```

---

## AnimationAction

### Constructor (Internal -- NEVER call directly)

```ts
new AnimationAction(
  mixer: AnimationMixer,
  clip: AnimationClip,
  localRoot?: Object3D,
  blendMode?: number
): AnimationAction
```

ALWAYS create via `mixer.clipAction(clip)`.

### Properties

```ts
action.blendMode: number          // NormalAnimationBlendMode | AdditiveAnimationBlendMode
action.clampWhenFinished: boolean  // default: false
action.enabled: boolean            // default: true
action.loop: number                // LoopOnce | LoopRepeat | LoopPingPong (default: LoopRepeat)
action.paused: boolean             // default: false
action.repetitions: number         // default: Infinity
action.time: number                // local playback time in seconds (default: 0)
action.timeScale: number           // speed multiplier (default: 1; 0=pause, negative=reverse)
action.weight: number              // blend influence [0, 1] (default: 1)
action.zeroSlopeAtEnd: boolean     // default: true
action.zeroSlopeAtStart: boolean   // default: true
```

### Playback Methods

```ts
action.play(): AnimationAction
// Activates the action. Call ONCE to start playback.

action.stop(): AnimationAction
// Stops playback, resets time to 0, and deactivates the action.

action.reset(): AnimationAction
// Resets time=0, enabled=true, paused=false, timeScale=1, weight=1.
// Cancels any scheduled fading and warping. Does NOT deactivate.

action.startAt(time: number): AnimationAction
// Delays the start to the specified mixer time.
```

### Fading Methods

```ts
action.fadeIn(durationInSeconds: number): AnimationAction
// Fades weight from 0 to 1 over the specified duration.

action.fadeOut(durationInSeconds: number): AnimationAction
// Fades weight from current value to 0 over the specified duration.

action.crossFadeFrom(
  fadeOutAction: AnimationAction,
  durationInSeconds: number,
  warpBoolean: boolean
): AnimationAction
// Crossfades from fadeOutAction into this action.
// If warp=true, also warps timeScale for smooth speed transition.

action.crossFadeTo(
  fadeInAction: AnimationAction,
  durationInSeconds: number,
  warpBoolean: boolean
): AnimationAction
// Crossfades from this action to fadeInAction.

action.stopFading(): AnimationAction
// Cancels any currently active fade.
```

### Speed and Timing Methods

```ts
action.halt(durationInSeconds: number): AnimationAction
// Decelerates timeScale to 0 over the specified duration.

action.warp(
  startTimeScale: number,
  endTimeScale: number,
  durationInSeconds: number
): AnimationAction
// Smoothly transitions playback speed from start to end over duration.

action.stopWarping(): AnimationAction
// Cancels any currently active warp.

action.setDuration(durationInSeconds: number): AnimationAction
// Sets timeScale so one loop takes exactly durationInSeconds.

action.setEffectiveTimeScale(timeScale: number): AnimationAction
// Sets effective time scale (considers paused state).

action.setEffectiveWeight(weight: number): AnimationAction
// Sets effective weight (considers enabled state).

action.setLoop(mode: number, repetitions: number): AnimationAction
// Sets loop mode and repetition count.

action.syncWith(otherAction: AnimationAction): AnimationAction
// Synchronizes this action's time with another action.
```

### Query Methods

```ts
action.isRunning(): boolean
// true only when action is actively playing (not paused, weight > 0, speed != 0).

action.isScheduled(): boolean
// true if .play() was called and action has not been stopped.

action.getClip(): AnimationClip
action.getMixer(): AnimationMixer
action.getRoot(): Object3D
action.getEffectiveTimeScale(): number
action.getEffectiveWeight(): number
```

---

## AnimationClip

### Constructor

```ts
new AnimationClip(
  name?: string,          // default: ''
  duration?: number,      // default: -1 (auto-calculate)
  tracks?: KeyframeTrack[],
  blendMode?: number
): AnimationClip
```

### Properties

```ts
clip.blendMode: number
clip.duration: number          // length in seconds
clip.name: string              // identifier
clip.tracks: KeyframeTrack[]   // keyframe data
clip.userData: Object           // custom metadata
clip.uuid: string              // readonly unique ID
```

### Instance Methods

```ts
clip.clone(): AnimationClip
clip.optimize(): AnimationClip       // removes redundant sequential keys
clip.resetDuration(): AnimationClip  // updates duration to longest track
clip.toJSON(): Object
clip.trim(): AnimationClip           // crops all tracks to clip duration
clip.validate(): boolean             // returns true if all tracks are valid
```

### Static Methods

```ts
AnimationClip.findByName(
  objectOrClipArray: Object3D | AnimationClip[],
  name: string
): AnimationClip
// Finds a clip by name in an array or on an object's .animations property.

AnimationClip.CreateFromMorphTargetSequence(
  name: string,
  morphTargetSequence: MorphTarget[],
  fps: number,
  noLoop: boolean
): AnimationClip

AnimationClip.CreateClipsFromMorphTargetSequences(
  morphTargets: MorphTarget[],
  fps: number,
  noLoop: boolean
): AnimationClip[]

AnimationClip.parse(json: Object): AnimationClip
AnimationClip.toJSON(clip: AnimationClip): Object
```

---

## KeyframeTrack

### Base Constructor

```ts
new KeyframeTrack(
  name: string,                    // PropertyBinding path (e.g., "mesh.position")
  times: Float32Array | number[],  // keyframe times in seconds
  values: Float32Array | number[], // keyframe values
  interpolation?: number           // InterpolateDiscrete | InterpolateLinear | InterpolateSmooth
): KeyframeTrack
```

### Specialized Track Constructors

```ts
new VectorKeyframeTrack(name, times, values, interpolation?)
new QuaternionKeyframeTrack(name, times, values, interpolation?)
new NumberKeyframeTrack(name, times, values, interpolation?)
new BooleanKeyframeTrack(name, times, values)        // discrete only
new ColorKeyframeTrack(name, times, values, interpolation?)
new StringKeyframeTrack(name, times, values)          // discrete only
```

### Track Methods

```ts
track.optimize(): KeyframeTrack    // removes redundant sequential keys
track.scale(timeScale: number): KeyframeTrack
track.shift(timeOffset: number): KeyframeTrack
track.trim(startTime: number, endTime: number): KeyframeTrack
track.clone(): KeyframeTrack
track.validate(): boolean
```

---

## Clock

### Constructor

```ts
new Clock(autoStart?: boolean): Clock
// autoStart defaults to true.
```

### Properties

```ts
clock.autoStart: boolean
clock.elapsedTime: number     // total accumulated time in seconds
clock.oldTime: number         // last method call timestamp
clock.running: boolean        // default: true
clock.startTime: number       // when start() was last called
```

### Methods

```ts
clock.getDelta(): number        // seconds since last getDelta() call
clock.getElapsedTime(): number  // total elapsed time in seconds
clock.start(): void             // starts the clock
clock.stop(): void              // halts without resetting
```

---

## Constants

```ts
// Loop modes
THREE.LoopOnce        // plays once and stops
THREE.LoopRepeat      // loops, restarting from beginning (default)
THREE.LoopPingPong    // alternates forward/backward

// Blend modes
THREE.NormalAnimationBlendMode    // standard blending (default)
THREE.AdditiveAnimationBlendMode  // additive layering

// Interpolation modes
THREE.InterpolateDiscrete  // step function
THREE.InterpolateLinear    // linear (default)
THREE.InterpolateSmooth    // cubic spline
```
