# Anti-Patterns (Three.js Scene Graph r160+)

## 1. Using add() Instead of attach() for Reparenting

```javascript
// WRONG: add() preserves local transform -- object jumps to unexpected position
const mesh = new Mesh(geometry, material);
mesh.position.set(0, 0, 0);
groupA.add(mesh); // world position = groupA.position + (0,0,0)

groupB.add(mesh); // mesh.position is still (0,0,0) but now relative to groupB
// object visually JUMPS to groupB's origin

// CORRECT: attach() preserves world transform -- object stays in place
groupB.attach(mesh); // mesh.position is recalculated so world position does not change
```

**WHY**: `add()` keeps the local position/rotation/scale unchanged, meaning the world transform changes when the new parent has a different world transform. `attach()` recalculates local transform to maintain the existing world transform.

---

## 2. Modifying Children Array During Traversal

```javascript
// WRONG: modifying children during traverse causes skipped or double-visited objects
scene.traverse((object) => {
  if (object.userData.expired) {
    object.removeFromParent(); // mutates children array mid-iteration!
  }
});

// CORRECT: collect first, then modify
const toRemove = [];
scene.traverse((object) => {
  if (object.userData.expired) {
    toRemove.push(object);
  }
});
toRemove.forEach((obj) => obj.removeFromParent());
```

**WHY**: `traverse()` iterates the `children` array with a for-loop. Removing or adding elements during iteration shifts indices, causing objects to be skipped or visited twice.

---

## 3. Directly Modifying the children Array

```javascript
// WRONG: bypasses parent reference updates and event dispatch
group.children.push(mesh);          // mesh.parent is still null!
group.children.splice(0, 1);        // removed object.parent still points to group!
group.children = [];                 // orphans all children with stale parent refs

// CORRECT: ALWAYS use the hierarchy methods
group.add(mesh);                     // sets mesh.parent, fires 'added' event
group.remove(mesh);                  // clears mesh.parent, fires 'removed' event
group.clear();                       // properly removes all children
```

**WHY**: `add()` and `remove()` manage parent references, fire events, and handle auto-removal from previous parents. Direct array manipulation breaks all of these guarantees.

---

## 4. Reading Stale matrixWorld

```javascript
// WRONG: matrixWorld is stale immediately after changing position
mesh.position.set(10, 5, 0);
const worldPos = new Vector3();
mesh.getWorldPosition(worldPos); // may return old position if parent chain is stale

// ALSO WRONG: accessing matrixWorld.elements directly without update
mesh.position.set(10, 5, 0);
const x = mesh.matrixWorld.elements[12]; // stale value!

// CORRECT: force update before reading
mesh.position.set(10, 5, 0);
mesh.updateWorldMatrix(true, false); // update parents first, not children
const worldPos = new Vector3();
mesh.getWorldPosition(worldPos); // now correct
```

**WHY**: `matrixWorld` is only recalculated during `renderer.render()` or explicit `updateWorldMatrix()`/`updateMatrixWorld()` calls. Between frames, it contains the value from the last render.

**NOTE**: `getWorldPosition()`, `getWorldQuaternion()`, `getWorldScale()`, and `getWorldDirection()` call `updateWorldMatrix(true, false)` internally, so they are safe. But accessing `matrixWorld` properties directly is NOT safe without a manual update.

---

## 5. Forgetting to Dispose When Removing Objects

```javascript
// WRONG: remove() only detaches from scene graph -- GPU memory leaks
scene.remove(mesh);
// geometry and material are still in GPU memory!

// CORRECT: dispose geometry, material, and textures
scene.remove(mesh);
mesh.geometry.dispose();
if (Array.isArray(mesh.material)) {
  mesh.material.forEach((mat) => {
    Object.values(mat).forEach((value) => {
      if (value && value.isTexture) value.dispose();
    });
    mat.dispose();
  });
} else {
  Object.values(mesh.material).forEach((value) => {
    if (value && value.isTexture) value.dispose();
  });
  mesh.material.dispose();
}
```

**WHY**: Three.js separates scene graph management from GPU resource management. `remove()` and `clear()` only affect the parent-child hierarchy. Geometries, materials, and textures MUST be disposed explicitly to free GPU memory.

---

## 6. Using set() Instead of enable() on Layers

```javascript
// WRONG: set() disables ALL other layers
mesh.layers.set(2);     // mesh is now ONLY on layer 2, removed from layer 0!
// camera (on layer 0) can no longer see this mesh

// CORRECT: enable() adds a layer without removing existing ones
mesh.layers.enable(2);  // mesh is now on layers 0 AND 2
```

**WHY**: `layers.set(n)` replaces the entire bitmask with only bit `n`. `layers.enable(n)` performs a bitwise OR, adding the layer without affecting others. Use `set()` only when you want exclusive layer membership.

---

## 7. Iterating Children with for-loop for Removal

```javascript
// WRONG: forward iteration skips elements when removing
for (let i = 0; i < group.children.length; i++) {
  group.remove(group.children[i]); // children[1] becomes children[0], gets skipped
}

// ALSO WRONG: for-of with removal
for (const child of group.children) {
  group.remove(child); // mutates the array being iterated
}

// CORRECT: use clear() for removing all children
group.clear();

// CORRECT: if selectively removing, iterate backwards or collect first
for (let i = group.children.length - 1; i >= 0; i--) {
  if (group.children[i].userData.removable) {
    group.remove(group.children[i]);
  }
}
```

**WHY**: Forward iteration with removal causes index shifting. After removing index 0, the previous index 1 slides to index 0 and is never visited. Reverse iteration or `clear()` avoids this.

---

## 8. Calling lookAt() Before Setting Position

```javascript
// WRONG: lookAt computes rotation from current position
mesh.lookAt(targetPosition); // rotation computed from default (0,0,0)
mesh.position.set(10, 5, 0); // position changes but rotation is stale

// CORRECT: ALWAYS set position first, then lookAt
mesh.position.set(10, 5, 0);
mesh.lookAt(targetPosition); // rotation computed from (10,5,0)
```

**WHY**: `lookAt()` calculates the rotation needed to face the target from the object's current position. Setting position after `lookAt()` moves the object without updating the rotation.

---

## 9. Storing Three.js Objects in userData Without Cleanup

```javascript
// WRONG: userData references prevent garbage collection
mesh.userData.originalMaterial = mesh.material;
mesh.userData.helperMesh = new Mesh(geometry, helperMaterial);
// when mesh is removed and disposed, userData references keep material and helperMesh alive

// CORRECT: clean up userData references during disposal
function disposeMesh(mesh) {
  if (mesh.userData.helperMesh) {
    mesh.userData.helperMesh.geometry.dispose();
    mesh.userData.helperMesh.material.dispose();
    mesh.userData.helperMesh = null;
  }
  mesh.userData.originalMaterial = null;
  mesh.geometry.dispose();
  mesh.material.dispose();
  mesh.removeFromParent();
}
```

**WHY**: `userData` is a plain object that is NOT touched by `dispose()`. Any Three.js objects stored there (materials, textures, geometries, meshes) remain in memory until their references are explicitly cleared and their `dispose()` methods are called.

---

## 10. Assuming userData Survives Deep Clone

```javascript
// WRONG: expecting userData to be deeply cloned
mesh.userData.config = { layers: [1, 2, 3], nested: { value: true } };
const clone = mesh.clone();
clone.userData.config.layers.push(4);
console.log(mesh.userData.config.layers); // [1, 2, 3, 4] -- original is mutated!

// CORRECT: manually deep-clone userData if it contains nested objects
const clone = mesh.clone();
clone.userData = JSON.parse(JSON.stringify(mesh.userData));
// now modifying clone.userData does not affect the original
```

**WHY**: `Object3D.clone()` performs a shallow copy of `userData` via `Object.assign()`. Nested objects and arrays are shared by reference between the original and clone.

---

## 11. Setting matrixAutoUpdate = false Without Manual Updates

```javascript
// WRONG: disabling auto-update and then using position/rotation/scale
mesh.matrixAutoUpdate = false;
mesh.position.set(5, 0, 0); // position changed but matrix is NOT updated
// mesh renders at old position!

// CORRECT option A: manually update matrix after changes
mesh.matrixAutoUpdate = false;
mesh.position.set(5, 0, 0);
mesh.updateMatrix(); // recompute matrix from position/rotation/scale

// CORRECT option B: set matrix directly
mesh.matrixAutoUpdate = false;
mesh.matrix.makeTranslation(5, 0, 0);
mesh.matrixWorldNeedsUpdate = true;
```

**WHY**: When `matrixAutoUpdate` is `false`, the renderer skips `updateMatrix()` for that object. The `matrix` property is only updated when you call `updateMatrix()` explicitly or set it directly. This is a performance optimization for static objects, but using `position`/`rotation`/`scale` without calling `updateMatrix()` results in the object rendering at its last computed transform.
