# threejs-impl-audio — Anti-Patterns

> What NOT to do with Three.js audio. Each anti-pattern includes the mistake, why it fails, and the correct approach.

---

## AP-1: Playing Audio Without User Interaction

### Wrong

```js
const sound = new THREE.Audio( listener );
audioLoader.load( 'music.mp3', ( buffer ) => {
    sound.setBuffer( buffer );
    sound.play(); // FAILS silently — AudioContext is suspended
});
```

### Why It Fails

Modern browsers (Chrome, Firefox, Safari) enforce autoplay policies. The `AudioContext` starts in a `'suspended'` state and WILL NOT process audio until a user gesture (click, tap, keypress) activates it. Calling `play()` on a suspended context does nothing — no error is thrown, audio simply does not play.

### Correct

```js
audioLoader.load( 'music.mp3', ( buffer ) => {
    sound.setBuffer( buffer );
});

document.addEventListener( 'click', () => {
    if ( listener.context.state === 'suspended' ) {
        listener.context.resume();
    }
    if ( !sound.isPlaying ) {
        sound.play();
    }
}, { once: true } );
```

---

## AP-2: Forgetting to Attach AudioListener to Camera

### Wrong

```js
const listener = new THREE.AudioListener();
// listener is never added to the scene graph
const positionalSound = new THREE.PositionalAudio( listener );
mesh.add( positionalSound );
```

### Why It Fails

The `AudioListener` derives its world position and orientation from its parent in the scene graph. Without being attached to the camera (or any Object3D in the scene), the listener stays at the origin with no orientation updates. Spatial audio panning and distance calculations produce incorrect results — sounds do not respond to camera movement.

### Correct

```js
const listener = new THREE.AudioListener();
camera.add( listener ); // ALWAYS add to camera
```

---

## AP-3: Creating Multiple AudioListeners

### Wrong

```js
const listener1 = new THREE.AudioListener();
const listener2 = new THREE.AudioListener();
camera.add( listener1 );
camera.add( listener2 );

const music = new THREE.Audio( listener1 );
const sfx = new THREE.Audio( listener2 );
```

### Why It Fails

Each `AudioListener` creates its own `AudioContext`. The Web Audio API uses a single audio destination per context. Multiple listeners mean multiple independent audio graphs with separate timing, volume, and spatial processing. This wastes resources and produces unpredictable behavior — particularly for spatial audio where only ONE listener position makes physical sense.

### Correct

```js
const listener = new THREE.AudioListener();
camera.add( listener );

const music = new THREE.Audio( listener );     // shares same listener
const sfx = new THREE.Audio( listener );       // shares same listener
const spatial = new THREE.PositionalAudio( listener ); // shares same listener
```

---

## AP-4: Using autoplay Without Interaction Handler

### Wrong

```js
const sound = new THREE.Audio( listener );
sound.autoplay = true; // will be blocked by browser

audioLoader.load( 'music.mp3', ( buffer ) => {
    sound.setBuffer( buffer ); // autoplay triggers here — but AudioContext is suspended
});
```

### Why It Fails

Setting `autoplay = true` causes `play()` to be called automatically when `setBuffer()` is invoked. However, if the `AudioContext` is still suspended (no user interaction has occurred), the play request is silently ignored. The `autoplay` property does NOT bypass browser autoplay policies.

### Correct

```js
const sound = new THREE.Audio( listener );
// Do NOT use autoplay — explicitly play after user interaction

audioLoader.load( 'music.mp3', ( buffer ) => {
    sound.setBuffer( buffer );
    sound.setLoop( true );
});

document.addEventListener( 'click', () => {
    listener.context.resume().then( () => {
        sound.play();
    });
}, { once: true } );
```

---

## AP-5: Wrong Distance Model for the Use Case

### Wrong

```js
const sound = new THREE.PositionalAudio( listener );
sound.setDistanceModel( 'inverse' );
sound.setMaxDistance( 50 ); // maxDistance has NO effect with 'inverse' model
```

### Why It Fails

The `maxDistance` parameter is ONLY used by the `'linear'` distance model. For `'inverse'` and `'exponential'` models, sound attenuates based on `refDistance` and `rolloffFactor` but NEVER reaches zero — `maxDistance` is completely ignored. Developers who set `maxDistance` expecting a cutoff radius with the `'inverse'` model will hear sound at arbitrary distances.

### Correct

```js
// Option A: Use 'linear' if you need a hard cutoff
sound.setDistanceModel( 'linear' );
sound.setRefDistance( 1 );
sound.setMaxDistance( 50 );    // sound reaches zero at 50 units
sound.setRolloffFactor( 1 );

// Option B: Use 'inverse' with appropriate rolloffFactor for natural falloff
sound.setDistanceModel( 'inverse' );
sound.setRefDistance( 5 );     // full volume within 5 units
sound.setRolloffFactor( 2 );  // higher = faster falloff (but never zero)
```

---

## AP-6: Not Handling Audio Load Errors

### Wrong

```js
audioLoader.load( 'sound.ogg', ( buffer ) => {
    sound.setBuffer( buffer );
    sound.play();
});
// No error callback — if file is missing, fails silently
```

### Why It Fails

`AudioLoader.load()` fails silently when the file is missing, the URL is wrong, or the server returns an error. Without an `onError` callback, the application has no way to detect the failure, display a fallback, or inform the user. The `sound.buffer` remains `null`, and calling `play()` on a sound with no buffer throws a runtime error.

### Correct

```js
audioLoader.load(
    'sound.ogg',
    ( buffer ) => {
        sound.setBuffer( buffer );
    },
    undefined, // onProgress (optional)
    ( err ) => {
        console.error( 'Failed to load audio:', err );
        // Fallback: try alternative format, show UI message, etc.
    }
);
```

---

## AP-7: Calling play() on an Already Playing Sound

### Wrong

```js
document.addEventListener( 'click', () => {
    sound.play(); // Called on EVERY click — error on second click
});
```

### Why It Fails

Calling `play()` on an `Audio` instance that is already playing throws an error or causes the sound to restart abruptly. Three.js creates a new `AudioBufferSourceNode` on each `play()` call, and attempting to start a new source while the previous one is active produces undefined behavior.

### Correct

```js
document.addEventListener( 'click', () => {
    if ( !sound.isPlaying ) {
        sound.play();
    }
});
```

---

## AP-8: Using setMediaElementSource and Then Calling sound.play()

### Wrong

```js
const audioEl = document.createElement( 'audio' );
audioEl.src = 'podcast.mp3';

const sound = new THREE.Audio( listener );
sound.setMediaElementSource( audioEl );
sound.play(); // Does NOT work as expected
```

### Why It Fails

When using `setMediaElementSource()`, the Three.js `Audio` object wraps an HTML5 media element — it does NOT create an `AudioBufferSourceNode`. Playback MUST be controlled through the HTML element (`audioEl.play()`, `audioEl.pause()`), not through `sound.play()`. The `sound.play()` method attempts to create and start a buffer source node, which conflicts with the media element source.

### Correct

```js
const audioEl = document.createElement( 'audio' );
audioEl.src = 'podcast.mp3';
audioEl.crossOrigin = 'anonymous';

const sound = new THREE.Audio( listener );
sound.setMediaElementSource( audioEl );
sound.setVolume( 0.5 ); // volume control works through Three.js

// Control playback via the HTML element
document.addEventListener( 'click', () => {
    listener.context.resume();
    audioEl.play(); // use the HTML element's play()
}, { once: true } );
```
