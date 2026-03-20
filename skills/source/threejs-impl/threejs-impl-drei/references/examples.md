# threejs-impl-drei — Examples

## Example 1: Product Viewer with Environment and Shadows

Complete product viewer with orbit controls, environment lighting, and contact shadows.

```jsx
import { Canvas } from '@react-three/fiber'
import { OrbitControls, Environment, ContactShadows, Center } from '@react-three/drei'
import { Suspense } from 'react'

function ProductViewer() {
  return (
    <Canvas shadows camera={{ position: [0, 2, 5], fov: 50 }}>
      <Suspense fallback={null}>
        <Center>
          <Product />
        </Center>
        <Environment preset="studio" background={false} />
      </Suspense>
      <ContactShadows
        position={[0, -1, 0]}
        opacity={0.5}
        scale={10}
        blur={2}
        far={4}
        resolution={512}
      />
      <OrbitControls
        makeDefault
        minDistance={2}
        maxDistance={10}
        maxPolarAngle={Math.PI / 2}
        enablePan={false}
      />
    </Canvas>
  )
}

function Product(props) {
  const { nodes, materials } = useGLTF('/shoe.glb')
  return (
    <group {...props}>
      <mesh
        geometry={nodes.Shoe.geometry}
        material={materials.Leather}
        castShadow
      />
    </group>
  )
}
useGLTF.preload('/shoe.glb')
```

---

## Example 2: Scroll-Driven Experience

A scrolling website with 3D elements that animate based on scroll position.

```jsx
import { Canvas, useFrame } from '@react-three/fiber'
import { ScrollControls, useScroll, Text, Float, Environment } from '@react-three/drei'
import { useRef } from 'react'

function ScrollExperience() {
  return (
    <Canvas camera={{ position: [0, 0, 5] }}>
      <Environment preset="sunset" />
      <ScrollControls pages={3} damping={0.1}>
        <ScrollContent />
      </ScrollControls>
    </Canvas>
  )
}

function ScrollContent() {
  const scroll = useScroll()
  const groupRef = useRef()

  useFrame(() => {
    const offset = scroll.offset
    groupRef.current.rotation.y = offset * Math.PI * 2
    groupRef.current.position.y = -offset * 10
  })

  return (
    <group ref={groupRef}>
      <Float speed={2} floatIntensity={0.5}>
        <Text
          fontSize={0.8}
          color="white"
          anchorX="center"
          anchorY="middle"
          position={[0, 0, 0]}
        >
          Scroll Down
        </Text>
      </Float>

      <mesh position={[0, -5, 0]}>
        <torusKnotGeometry args={[1, 0.3, 128, 16]} />
        <meshStandardMaterial color="hotpink" metalness={0.8} roughness={0.2} />
      </mesh>

      <mesh position={[0, -10, 0]}>
        <icosahedronGeometry args={[1.5, 4]} />
        <meshStandardMaterial color="cyan" metalness={0.9} roughness={0.1} />
      </mesh>
    </group>
  )
}
```

---

## Example 3: Glass Material with Transmission

Realistic glass objects using MeshTransmissionMaterial.

```jsx
import { Canvas } from '@react-three/fiber'
import {
  MeshTransmissionMaterial, Environment, OrbitControls,
  AccumulativeShadows, RandomizedLight, Center
} from '@react-three/drei'
import { Suspense } from 'react'

function GlassScene() {
  return (
    <Canvas shadows camera={{ position: [0, 1.5, 4], fov: 45 }}>
      <color attach="background" args={['#f0f0f0']} />
      <Suspense fallback={null}>
        <Center>
          <GlassSphere />
          <GlassCube position={[2, 0, 0]} />
        </Center>
        <Environment preset="city" />
      </Suspense>
      <AccumulativeShadows
        temporal
        frames={100}
        scale={10}
        position={[0, -1, 0]}
        opacity={0.8}
      >
        <RandomizedLight amount={8} radius={4} position={[5, 5, -10]} />
      </AccumulativeShadows>
      <OrbitControls makeDefault />
    </Canvas>
  )
}

function GlassSphere(props) {
  return (
    <mesh {...props}>
      <sphereGeometry args={[0.8, 64, 64]} />
      <MeshTransmissionMaterial
        transmission={1}
        thickness={0.5}
        roughness={0}
        chromaticAberration={0.1}
        ior={1.5}
        color="#ffffff"
        backside
        backsideThickness={0.3}
        resolution={1024}
        samples={16}
      />
    </mesh>
  )
}

function GlassCube({ position }) {
  return (
    <mesh position={position}>
      <boxGeometry args={[1, 1, 1]} />
      <MeshTransmissionMaterial
        transmission={0.95}
        thickness={0.8}
        roughness={0.05}
        chromaticAberration={0.15}
        distortion={0.1}
        temporalDistortion={0.1}
        ior={1.45}
        color="#aaddff"
        backside
      />
    </mesh>
  )
}
```

---

## Example 4: Instanced Rendering with Performance Monitor

Efficiently rendering thousands of objects with adaptive performance.

```jsx
import { Canvas } from '@react-three/fiber'
import {
  Instances, Instance, PerformanceMonitor,
  AdaptiveDpr, Environment, OrbitControls
} from '@react-three/drei'
import { useState, useMemo } from 'react'

function MassiveScene() {
  const [dpr, setDpr] = useState(1.5)

  return (
    <Canvas dpr={dpr} camera={{ position: [0, 10, 20], fov: 60 }}>
      <PerformanceMonitor
        onIncline={() => setDpr(2)}
        onDecline={() => setDpr(1)}
        flipflops={3}
        onFallback={() => setDpr(0.5)}
      />
      <AdaptiveDpr pixelated />
      <Environment preset="warehouse" />
      <Cubes count={5000} />
      <OrbitControls makeDefault />
    </Canvas>
  )
}

function Cubes({ count }) {
  const positions = useMemo(() => {
    return Array.from({ length: count }, () => [
      (Math.random() - 0.5) * 50,
      (Math.random() - 0.5) * 50,
      (Math.random() - 0.5) * 50,
    ])
  }, [count])

  return (
    <Instances limit={count} range={count}>
      <boxGeometry args={[0.5, 0.5, 0.5]} />
      <meshStandardMaterial roughness={0.3} metalness={0.8} />
      {positions.map((pos, i) => (
        <Instance
          key={i}
          position={pos}
          rotation={[Math.random() * Math.PI, Math.random() * Math.PI, 0]}
          color={`hsl(${(i / count) * 360}, 70%, 60%)`}
        />
      ))}
    </Instances>
  )
}
```

---

## Example 5: Annotated 3D Model with Html Labels

Loading a GLTF model with interactive HTML annotations positioned in 3D space.

```jsx
import { Canvas } from '@react-three/fiber'
import {
  useGLTF, Html, OrbitControls, Environment, Billboard, Text
} from '@react-three/drei'
import { Suspense, useState } from 'react'

function AnnotatedModel() {
  return (
    <Canvas camera={{ position: [0, 2, 5], fov: 50 }}>
      <Suspense fallback={null}>
        <Environment preset="apartment" />
        <CarModel />
      </Suspense>
      <OrbitControls makeDefault />
    </Canvas>
  )
}

function CarModel() {
  const { nodes, materials } = useGLTF('/car.glb')
  const [active, setActive] = useState(null)

  return (
    <group>
      <primitive object={nodes.Scene} />

      {/* Engine annotation */}
      <Annotation
        position={[0, 1.2, 1.5]}
        label="Engine"
        description="V8 Twin-Turbo, 450 HP"
        isActive={active === 'engine'}
        onClick={() => setActive(active === 'engine' ? null : 'engine')}
      />

      {/* Wheel annotation */}
      <Annotation
        position={[1.5, 0.4, 1.8]}
        label="Wheels"
        description="20-inch alloy, Michelin PS4S"
        isActive={active === 'wheels'}
        onClick={() => setActive(active === 'wheels' ? null : 'wheels')}
      />
    </group>
  )
}

function Annotation({ position, label, description, isActive, onClick }) {
  return (
    <group position={position}>
      {/* 3D marker */}
      <Billboard>
        <mesh onClick={onClick}>
          <circleGeometry args={[0.1, 32]} />
          <meshBasicMaterial color={isActive ? '#ff6600' : '#ffffff'} />
        </mesh>
      </Billboard>

      {/* HTML tooltip */}
      {isActive && (
        <Html
          transform
          distanceFactor={8}
          center
          occlude
          style={{
            background: 'rgba(0, 0, 0, 0.8)',
            color: 'white',
            padding: '12px 16px',
            borderRadius: '8px',
            fontSize: '14px',
            whiteSpace: 'nowrap',
            pointerEvents: 'none',
          }}
        >
          <strong>{label}</strong>
          <p style={{ margin: '4px 0 0', opacity: 0.8 }}>{description}</p>
        </Html>
      )}
    </group>
  )
}

useGLTF.preload('/car.glb')
```
