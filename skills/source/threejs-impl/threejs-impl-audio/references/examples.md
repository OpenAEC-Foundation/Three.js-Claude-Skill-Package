# threejs-impl-audio — Examples

> Working code examples for Three.js audio (r160+). All examples use ES module imports.

---

## Example 1: Background Music with Autoplay Policy

Non-positional audio that loops background music, correctly handling the browser autoplay policy.

```js
import * as THREE from 'three';

// Scene setup (assumes scene, camera, renderer exist)
const listener = new THREE.AudioListener();
camera.add( listener );

const backgroundMusic = new THREE.Audio( listener );
const audioLoader = new THREE.AudioLoader();

audioLoader.load( 'assets/music.mp3', ( buffer ) => {
    backgroundMusic.setBuffer( buffer );
    backgroundMusic.setLoop( true );
    backgroundMusic.setVolume( 0.3 );
});

// MANDATORY: Resume AudioContext on user interaction
const startButton = document.getElementById( 'start' );
startButton.addEventListener( 'click', () => {
    if ( listener.context.state === 'suspended' ) {
        listener.context.resume();
    }
    if ( !backgroundMusic.isPlaying ) {
        backgroundMusic.play();
    }
    startButton.style.display = 'none';
}, { once: true } );
```

---

## Example 2: 3D Positional Audio on a Mesh

A sound source attached to a mesh. Volume and panning change as the camera moves.

```js
import * as THREE from 'three';

const listener = new THREE.AudioListener();
camera.add( listener );

const audioLoader = new THREE.AudioLoader();

// Create a visible object with a sound
const geometry = new THREE.SphereGeometry( 1, 32, 32 );
const material = new THREE.MeshStandardMaterial( { color: 0xff6600 } );
const speaker = new THREE.Mesh( geometry, material );
speaker.position.set( 10, 2, 0 );
scene.add( speaker );

// Attach positional audio to the mesh
const engineSound = new THREE.PositionalAudio( listener );

audioLoader.load( 'assets/engine.ogg', ( buffer ) => {
    engineSound.setBuffer( buffer );
    engineSound.setRefDistance( 5 );
    engineSound.setMaxDistance( 100 );
    engineSound.setRolloffFactor( 1 );
    engineSound.setDistanceModel( 'inverse' );
    engineSound.setLoop( true );
    engineSound.setVolume( 0.8 );
});

speaker.add( engineSound ); // sound position = mesh position

// Resume context on interaction
document.addEventListener( 'click', () => {
    if ( listener.context.state === 'suspended' ) {
        listener.context.resume();
    }
    if ( !engineSound.isPlaying ) {
        engineSound.play();
    }
}, { once: true } );
```

---

## Example 3: Audio Visualization with AudioAnalyser

Drive visual elements from real-time audio frequency data.

```js
import * as THREE from 'three';

const listener = new THREE.AudioListener();
camera.add( listener );

const sound = new THREE.Audio( listener );
const audioLoader = new THREE.AudioLoader();
const analyser = new THREE.AudioAnalyser( sound, 256 );

audioLoader.load( 'assets/beat.mp3', ( buffer ) => {
    sound.setBuffer( buffer );
    sound.setLoop( true );
});

// Create bar visualization (128 bars = fftSize / 2)
const bars = [];
const barCount = 128;
const barGeometry = new THREE.BoxGeometry( 0.1, 1, 0.1 );
const barMaterial = new THREE.MeshStandardMaterial( { color: 0x00ff88 } );

for ( let i = 0; i < barCount; i++ ) {
    const bar = new THREE.Mesh( barGeometry, barMaterial );
    bar.position.x = ( i - barCount / 2 ) * 0.15;
    scene.add( bar );
    bars.push( bar );
}

// Start audio on user interaction
document.addEventListener( 'click', () => {
    if ( listener.context.state === 'suspended' ) {
        listener.context.resume();
    }
    if ( !sound.isPlaying ) {
        sound.play();
    }
}, { once: true } );

// Animation loop — update bars from frequency data
function animate() {
    requestAnimationFrame( animate );

    const data = analyser.getFrequencyData();

    for ( let i = 0; i < barCount; i++ ) {
        const value = data[ i ] / 255;       // normalize to [0, 1]
        bars[ i ].scale.y = 0.1 + value * 5; // minimum height + scaled
        bars[ i ].position.y = bars[ i ].scale.y / 2; // keep bottom at y=0
    }

    renderer.render( scene, camera );
}
animate();
```

---

## Example 4: Multiple Sound Sources with Shared Listener

A scene with background music and multiple positional sounds.

```js
import * as THREE from 'three';

const listener = new THREE.AudioListener();
camera.add( listener );

const audioLoader = new THREE.AudioLoader();

// Background music (non-positional)
const bgMusic = new THREE.Audio( listener );
audioLoader.load( 'assets/ambient.mp3', ( buffer ) => {
    bgMusic.setBuffer( buffer );
    bgMusic.setLoop( true );
    bgMusic.setVolume( 0.2 );
});

// Waterfall sound (positional)
const waterfallSound = new THREE.PositionalAudio( listener );
audioLoader.load( 'assets/waterfall.ogg', ( buffer ) => {
    waterfallSound.setBuffer( buffer );
    waterfallSound.setRefDistance( 10 );
    waterfallSound.setRolloffFactor( 1.5 );
    waterfallSound.setDistanceModel( 'inverse' );
    waterfallSound.setLoop( true );
    waterfallSound.setVolume( 0.7 );
});
waterfallMesh.add( waterfallSound );

// Bird sound (positional, directional cone)
const birdSound = new THREE.PositionalAudio( listener );
audioLoader.load( 'assets/birdsong.ogg', ( buffer ) => {
    birdSound.setBuffer( buffer );
    birdSound.setRefDistance( 5 );
    birdSound.setRolloffFactor( 2 );
    birdSound.setDistanceModel( 'exponential' );
    birdSound.setDirectionalCone( 120, 230, 0.2 );
    birdSound.setLoop( true );
    birdSound.setVolume( 0.5 );
});
birdMesh.add( birdSound );

// Single interaction handler for all sounds
document.addEventListener( 'click', () => {
    if ( listener.context.state === 'suspended' ) {
        listener.context.resume();
    }
    if ( !bgMusic.isPlaying ) bgMusic.play();
    if ( !waterfallSound.isPlaying ) waterfallSound.play();
    if ( !birdSound.isPlaying ) birdSound.play();
}, { once: true } );
```

---

## Example 5: HTML5 Media Element Source (Streaming)

Use an HTML5 `<audio>` element for large files that should stream rather than fully preload.

```js
import * as THREE from 'three';

const listener = new THREE.AudioListener();
camera.add( listener );

// Create HTML5 audio element
const audioElement = document.createElement( 'audio' );
audioElement.src = 'assets/long-podcast.mp3';
audioElement.crossOrigin = 'anonymous'; // REQUIRED for cross-origin audio

// Wrap in Three.js Audio
const sound = new THREE.Audio( listener );
sound.setMediaElementSource( audioElement );
sound.setVolume( 0.6 );

// Play on user interaction
document.addEventListener( 'click', () => {
    if ( listener.context.state === 'suspended' ) {
        listener.context.resume();
    }
    audioElement.play(); // use the HTML element's play() for media element sources
}, { once: true } );
```

**Key difference:** When using `setMediaElementSource()`, control playback via the HTML element (`audioElement.play()`, `audioElement.pause()`), NOT via `sound.play()`. The Three.js `Audio` object acts as a pass-through for volume and filters only.
