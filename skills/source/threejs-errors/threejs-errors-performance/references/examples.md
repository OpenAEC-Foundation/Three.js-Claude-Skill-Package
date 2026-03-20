# threejs-errors-performance — Examples

## Example 1: Complete Scene Cleanup on Unmount

A React-style cleanup pattern that disposes ALL GPU resources when a 3D view is destroyed.

```javascript
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

function createScene(container) {
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);

  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(75, container.clientWidth / container.clientHeight, 0.1, 1000);
  camera.position.z = 5;

  const controls = new OrbitControls(camera, renderer.domElement);
  let animationId;

  function animate() {
    animationId = requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
  }
  animate();

  // CLEANUP FUNCTION — call on unmount
  function dispose() {
    cancelAnimationFrame(animationId);

    // Dispose all scene objects
    scene.traverse((object) => {
      if (object.geometry) object.geometry.dispose();
      if (object.material) {
        if (Array.isArray(object.material)) {
          object.material.forEach(disposeMaterial);
        } else {
          disposeMaterial(object.material);
        }
      }
    });
    scene.clear();

    // Dispose controls (removes DOM event listeners)
    controls.dispose();

    // Dispose renderer (destroys WebGL context)
    renderer.dispose();
    renderer.domElement.remove();
  }

  return { scene, camera, renderer, dispose };
}

function disposeMaterial(material) {
  const textureProps = [
    'map', 'lightMap', 'bumpMap', 'normalMap', 'specularMap',
    'envMap', 'alphaMap', 'aoMap', 'displacementMap',
    'emissiveMap', 'gradientMap', 'metalnessMap', 'roughnessMap'
  ];
  for (const prop of textureProps) {
    if (material[prop]) material[prop].dispose();
  }
  material.dispose();
}
```

---

## Example 2: InstancedMesh for a Forest of Trees

Renders 10,000 trees with a single draw call using InstancedMesh.

```javascript
import * as THREE from 'three';

function createForest(scene, treeGeometry, treeMaterial) {
  const count = 10000;
  const forest = new THREE.InstancedMesh(treeGeometry, treeMaterial, count);
  const dummy = new THREE.Object3D();
  const color = new THREE.Color();

  for (let i = 0; i < count; i++) {
    // Random position in a 500x500 area
    dummy.position.set(
      Math.random() * 500 - 250,
      0,
      Math.random() * 500 - 250
    );
    // Random Y rotation
    dummy.rotation.y = Math.random() * Math.PI * 2;
    // Random scale variation
    const s = 0.8 + Math.random() * 0.4;
    dummy.scale.set(s, s + Math.random() * 0.3, s);
    dummy.updateMatrix();
    forest.setMatrixAt(i, dummy.matrix);

    // Per-instance color variation
    color.setHSL(0.25 + Math.random() * 0.1, 0.6, 0.3 + Math.random() * 0.2);
    forest.setColorAt(i, color);
  }

  // CRITICAL: mark buffers for upload
  forest.instanceMatrix.needsUpdate = true;
  forest.instanceColor.needsUpdate = true;

  // Compute bounding sphere for frustum culling
  forest.computeBoundingSphere();

  scene.add(forest);
  return forest;
}
```

---

## Example 3: LOD System for Architectural Model

Switches between high, medium, and low detail based on camera distance.

```javascript
import * as THREE from 'three';

function createLODBuilding(highGeo, medGeo, lowGeo, material) {
  const lod = new THREE.LOD();

  const highMesh = new THREE.Mesh(highGeo, material);
  const medMesh = new THREE.Mesh(medGeo, material);
  const lowMesh = new THREE.Mesh(lowGeo, material);

  lod.addLevel(highMesh, 0);     // 0-50 units: full detail
  lod.addLevel(medMesh, 50);     // 50-200 units: medium detail
  lod.addLevel(lowMesh, 200);    // 200+ units: low detail

  return lod;
}

// In animation loop — ALWAYS call update
function animate() {
  requestAnimationFrame(animate);
  scene.traverse((child) => {
    if (child.isLOD) child.update(camera);
  });
  renderer.render(scene, camera);
}
```

---

## Example 4: Performance Monitor Dashboard

Real-time overlay showing draw calls, triangles, and memory usage.

```javascript
import * as THREE from 'three';
import Stats from 'three/addons/libs/stats.module.js';

function createPerformanceMonitor(renderer) {
  // FPS counter
  const stats = new Stats();
  stats.showPanel(0);
  document.body.appendChild(stats.dom);

  // Custom info panel
  const infoDiv = document.createElement('div');
  infoDiv.style.cssText =
    'position:fixed;top:0;right:0;padding:8px;background:rgba(0,0,0,0.7);' +
    'color:#0f0;font:12px monospace;z-index:10000;white-space:pre;';
  document.body.appendChild(infoDiv);

  function update() {
    const { render, memory } = renderer.info;
    infoDiv.textContent =
      `Draw calls:  ${render.calls}\n` +
      `Triangles:   ${render.triangles.toLocaleString()}\n` +
      `Geometries:  ${memory.geometries}\n` +
      `Textures:    ${memory.textures}\n` +
      `Programs:    ${renderer.info.programs?.length ?? 0}`;
  }

  function dispose() {
    stats.dom.remove();
    infoDiv.remove();
  }

  return { stats, update, dispose };
}

// Usage in animation loop
const monitor = createPerformanceMonitor(renderer);

function animate() {
  monitor.stats.begin();
  renderer.render(scene, camera);
  monitor.stats.end();
  monitor.update();
  requestAnimationFrame(animate);
}
```

---

## Example 5: Object Pool for Particle-Like Effects

Reuses mesh objects instead of creating/destroying them every frame.

```javascript
import * as THREE from 'three';

class ProjectilePool {
  constructor(scene, count) {
    this.scene = scene;
    this.geometry = new THREE.SphereGeometry(0.1, 8, 8);
    this.material = new THREE.MeshBasicMaterial({ color: 0xff4400 });
    this.pool = [];
    this.active = new Set();

    for (let i = 0; i < count; i++) {
      const mesh = new THREE.Mesh(this.geometry, this.material);
      mesh.visible = false;
      mesh.userData.velocity = new THREE.Vector3();
      mesh.userData.life = 0;
      scene.add(mesh);
      this.pool.push(mesh);
    }
  }

  spawn(position, velocity) {
    const mesh = this.pool.find((m) => !m.visible);
    if (!mesh) return null; // pool exhausted

    mesh.position.copy(position);
    mesh.userData.velocity.copy(velocity);
    mesh.userData.life = 2.0; // seconds
    mesh.visible = true;
    this.active.add(mesh);
    return mesh;
  }

  update(deltaTime) {
    for (const mesh of this.active) {
      mesh.position.addScaledVector(mesh.userData.velocity, deltaTime);
      mesh.userData.life -= deltaTime;
      if (mesh.userData.life <= 0) {
        mesh.visible = false;
        this.active.delete(mesh);
      }
    }
  }

  dispose() {
    this.geometry.dispose();
    this.material.dispose();
    this.pool.forEach((m) => this.scene.remove(m));
    this.pool.length = 0;
    this.active.clear();
  }
}
```
