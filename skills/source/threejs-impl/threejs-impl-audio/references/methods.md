# threejs-impl-audio — Method Reference

> Complete API signatures for Three.js audio classes (r160+).

---

## AudioListener

Extends `Object3D`. The scene's virtual ear — wraps the Web Audio API `AudioContext`.

### Constructor

```ts
new AudioListener()
```

Creates a new listener with its own `AudioContext`.

### Properties

| Property | Type | Access | Description |
|----------|------|--------|-------------|
| `.context` | `AudioContext` | readonly | The native Web Audio API context |
| `.gain` | `GainNode` | readonly | Master volume control node |
| `.filter` | `AudioNode \| null` | read/write | Optional global audio filter |
| `.timeDelta` | `number` | readonly | Time delta for audio operations |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `.getMasterVolume()` | `number` | Returns current master volume |
| `.setMasterVolume( value: number )` | `AudioListener` | Sets master volume; range `[0, 1]` |
| `.getFilter()` | `AudioNode` | Returns current filter node |
| `.setFilter( filter: AudioNode )` | `AudioListener` | Applies a global audio filter |
| `.removeFilter()` | `AudioListener` | Removes the current filter |
| `.getInput()` | `GainNode` | Returns the listener's input gain node |

---

## Audio

Extends `Object3D`. Non-positional audio source — volume is constant regardless of listener position.

### Constructor

```ts
new Audio( listener: AudioListener )
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listener` | `AudioListener` | YES | The scene's audio listener |

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.buffer` | `AudioBuffer \| null` | `null` | Loaded audio data (readonly after set) |
| `.context` | `AudioContext` | — | Web Audio context (readonly) |
| `.gain` | `GainNode` | — | Volume control node (readonly) |
| `.isPlaying` | `boolean` | `false` | Current playback state (readonly) |
| `.source` | `AudioBufferSourceNode \| null` | `null` | Current source node (readonly) |
| `.autoplay` | `boolean` | `false` | Auto-play when buffer is set |
| `.loop` | `boolean` | `false` | Loop playback |
| `.loopStart` | `number` | `0` | Loop region start (seconds) |
| `.loopEnd` | `number` | `0` | Loop region end (seconds); `0` = end of buffer |
| `.offset` | `number` | `0` | Playback start offset (seconds) |
| `.playbackRate` | `number` | `1` | Speed multiplier |
| `.detune` | `number` | `0` | Pitch shift in cents (100 cents = 1 semitone) |
| `.duration` | `number \| undefined` | — | Buffer duration (readonly, available after setBuffer) |
| `.filters` | `AudioNode[]` | `[]` | Applied audio filter chain |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `.play( delay?: number )` | `Audio` | Start playback; optional delay in seconds |
| `.pause()` | `Audio` | Pause playback (resume with `.play()`) |
| `.stop()` | `Audio` | Stop and reset to beginning |
| `.setBuffer( buffer: AudioBuffer )` | `Audio` | Set audio data from AudioLoader |
| `.setMediaElementSource( el: HTMLMediaElement )` | `Audio` | Use HTML5 `<audio>` or `<video>` element as source |
| `.setMediaStreamSource( stream: MediaStream )` | `Audio` | Use live media stream (microphone) as source |
| `.setNodeSource( node: AudioScheduledSourceNode )` | `Audio` | Use custom Web Audio source node |
| `.setVolume( value: number )` | `Audio` | Set volume; range `[0, 1]` |
| `.getVolume()` | `number` | Get current volume |
| `.setPlaybackRate( value: number )` | `Audio` | Set playback speed multiplier |
| `.getPlaybackRate()` | `number` | Get current playback speed |
| `.setDetune( value: number )` | `Audio` | Set pitch shift in cents |
| `.getDetune()` | `number` | Get current pitch shift |
| `.setLoop( value: boolean )` | `Audio` | Enable or disable looping |
| `.getLoop()` | `boolean` | Get current loop state |
| `.setLoopStart( value: number )` | `Audio` | Set loop start point (seconds) |
| `.setLoopEnd( value: number )` | `Audio` | Set loop end point (seconds) |
| `.setFilters( filters: AudioNode[] )` | `Audio` | Apply a chain of audio filters |
| `.getFilters()` | `AudioNode[]` | Get the current filter chain |
| `.setFilter( filter: AudioNode )` | `Audio` | Set a single filter (shorthand) |
| `.getFilter()` | `AudioNode` | Get the first filter |
| `.getOutput()` | `GainNode` | Get the output gain node |
| `.connect()` | `Audio` | Connect to audio destination |
| `.disconnect()` | `Audio` | Disconnect from audio destination |

---

## PositionalAudio

Extends `Audio`. 3D spatial audio — volume and panning depend on listener distance and orientation.

### Constructor

```ts
new PositionalAudio( listener: AudioListener )
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listener` | `AudioListener` | YES | The scene's audio listener |

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `.panner` | `PannerNode` | The Web Audio PannerNode controlling 3D spatialization (readonly) |

Inherits ALL properties from `Audio`.

### Methods (Own)

| Method | Returns | Description |
|--------|---------|-------------|
| `.getDistanceModel()` | `string` | Returns `'linear'`, `'inverse'`, or `'exponential'` |
| `.setDistanceModel( model: string )` | `PositionalAudio` | Set distance attenuation algorithm |
| `.getRefDistance()` | `number` | Get reference distance |
| `.setRefDistance( value: number )` | `PositionalAudio` | Set reference distance (where volume = 100%) |
| `.getMaxDistance()` | `number` | Get maximum distance |
| `.setMaxDistance( value: number )` | `PositionalAudio` | Set max distance (only affects `'linear'` model) |
| `.getRolloffFactor()` | `number` | Get rolloff factor |
| `.setRolloffFactor( value: number )` | `PositionalAudio` | Set rate of volume decrease with distance |
| `.setDirectionalCone( coneInnerAngle: number, coneOuterAngle: number, coneOuterGain: number )` | `PositionalAudio` | Define directional audio cone |
| `.getOutput()` | `PannerNode` | Returns the PannerNode (overrides Audio.getOutput) |

### Distance Model Formulas

| Model | Formula | Notes |
|-------|---------|-------|
| `'inverse'` | `refDistance / (refDistance + rolloffFactor * (distance - refDistance))` | Default; NEVER reaches zero |
| `'linear'` | `1 - rolloffFactor * (distance - refDistance) / (maxDistance - refDistance)` | Reaches zero at maxDistance |
| `'exponential'` | `(distance / refDistance) ^ -rolloffFactor` | Steep falloff |

---

## AudioLoader

Extends `Loader`. Loads audio files into `AudioBuffer` objects.

### Constructor

```ts
new AudioLoader( manager?: LoadingManager )
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `manager` | `LoadingManager` | NO | Optional loading manager |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `.load( url: string, onLoad?: ( buffer: AudioBuffer ) => void, onProgress?: ( event: ProgressEvent ) => void, onError?: ( err: Error ) => void )` | `void` | Load audio file asynchronously |
| `.loadAsync( url: string, onProgress?: ( event: ProgressEvent ) => void )` | `Promise<AudioBuffer>` | Promise-based load |

---

## AudioAnalyser

Wraps `AnalyserNode` for real-time frequency analysis.

### Constructor

```ts
new AudioAnalyser( audio: Audio, fftSize?: number )
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `audio` | `Audio` | — | The Audio or PositionalAudio to analyze (REQUIRED) |
| `fftSize` | `number` | `2048` | FFT window size; MUST be power of 2 |

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `.analyser` | `AnalyserNode` | The underlying Web Audio AnalyserNode |
| `.data` | `Uint8Array` | Reusable buffer for frequency data |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `.getFrequencyData()` | `Uint8Array` | Frequency domain data (0-255 per bin); length = `fftSize / 2` |
| `.getAverageFrequency()` | `number` | Arithmetic mean of all frequency bins |
