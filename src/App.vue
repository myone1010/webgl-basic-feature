<template>
  <div></div>
</template>

<script setup>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { OBJLoader } from 'three/examples/jsm/loaders/OBJLoader.js'
import { FBXLoader } from 'three/examples/jsm/loaders/FBXLoader.js'
import { SVGLoader } from 'three/examples/jsm/loaders/SVGLoader.js'
import { GUI } from 'three/examples/jsm/libs/lil-gui.module.min.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { EXRLoader } from 'three/examples/jsm/loaders/EXRLoader.js'

// import TWEEN from '@tweenjs/tween.js';

let camera, scene, renderer, controls, cube
let object, group, objGroup, settings, shaderMaterial

// GUI Control Values
let cameraView = 0
let gWidth = 0.5
let gSpeed = 0.5
let gRotation = 0
let gTransparency = 0
let gDirection = 0
// let gColor = 'false'
let gGradient = 'false'

init()

function init() {
  camera = new THREE.PerspectiveCamera(35, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.z = 5
  // scene

  scene = new THREE.Scene()

  const ambientLight = new THREE.AmbientLight(0xffffff)
  scene.add(ambientLight)

  scene.add(camera)

  // Create GUI
  createPanel()

  // shaderMaterial
  shaderMaterial = new THREE.ShaderMaterial({
    uniforms: {
      rotation: { value: Math.PI / 2 },
      stripeCount: { value: 15.0 },
      opacity: { value: 1 },
      gradient: { value: false },
      frequency: { value: 157.5 },
      resolution: { value: new THREE.Vector2() }
    },
    vertexShader: `
        attribute vec2 uvs;
        varying vec2 vUv;
        void main() {
          vUv = uvs;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        uniform float frequency;
        uniform vec2 resolution;
        uniform float rotation;
        uniform bool gradient;
        varying vec2 vUv;
        #define PI 3.141592

        // mat2 rotate2d(float angle){
        //   return mat2(cos(angle),-sin(angle),
        //                 sin(angle), cos(angle));
        // }

        void mainImage(out vec4 fragColor, in vec2 fragCoord) {
          vec2 uv = fragCoord.xy/resolution.xy;
          // vec2 translatedUvs = uv.x;

          // vec2 targetPos = rotate2d(rotation) * translatedUvs;
          // vec2 adjustedUV = vec2(vUv.x, floor(vUv.y * frequency) / frequency);

          float stripes = vUv.x * frequency;
          float rounded = floor(stripes);

          float mask = 0.0;
          if(gradient){
              float gradientMask = abs(fract(stripes - 0.5));
              mask = smoothstep(-0.5, 1.2, gradientMask);
              fragColor = vec4(vec3(mask, mask, mask), 1.0);
          }
          else {
            if (mod(rounded, 2.0) == 0.0) {
              fragColor = vec4(1.0, 1.0, 1.0, 1.0); // White
            } else {
              fragColor = vec4(0.0, 0.0, 0.0, 1.0); // Black
            }
          }

        }

        void main() {
          vec2 fragCoord = gl_FragCoord.xy;
          mainImage(gl_FragColor, fragCoord);
        }
    `
  })

  // manager
  function loadModel() {
    shaderMaterial.uniforms.resolution.value.x = renderer.domElement.width
    shaderMaterial.uniforms.resolution.value.y = renderer.domElement.height

    object.traverse(function (child) {
      if (child instanceof THREE.Mesh) {
        child.material = shaderMaterial
        let maxU = -Infinity
        let maxV = -Infinity
        const radius = 2.5
        const height = 5
        const resolution = 628

        const vertices = []
        const uvs = []

        for (let i = 0; i <= resolution; i++) {
          const theta = (i / resolution) * 2 * Math.PI
          const x = radius * Math.cos(theta)
          const z = radius * Math.sin(theta)

          for (let j = 0; j <= height; j++) {
            const y = j / height

            vertices.push([x, y, z])
            uvs.push([theta / (2 * Math.PI), y])
          }
          child.geometry.attributes.uvs = new THREE.Float32BufferAttribute(uvs.flat(), 2)
          // const u = faceVertexUvs[i]
          // const v = faceVertexUvs[i + 1]

          // if (u > maxU) {
          //   maxU = u
          // }
          // if (v > maxV) {
          //   maxV = v
          // }
        }
        console.log('Max U:', maxU)
        console.log('Max V:', maxV)
        // console.log(child.geometry.attributes)
        child.geometry.attributes.uv.needsUpdate = true
      }
    })

    object.position.y = 0.05
    object.position.z = 5
    object.scale.setScalar(0.5)
    object.name = 'cylinder'

    objGroup = new THREE.Group()
    objGroup.add(object)

    scene.add(objGroup)

    render()
  }

  const manager = new THREE.LoadingManager(loadModel)

  // texture

  const texture = new THREE.TextureLoader().load('Wood_floor_004_COLOR.jpg')
  texture.colorSpace = THREE.SRGBColorSpace
  texture.wrapS = THREE.RepeatWrapping
  texture.wrapT = THREE.RepeatWrapping
  texture.repeat.set(3, 3)

  const floorMaterial = new THREE.MeshPhongMaterial({ map: texture, side: THREE.DoubleSide })

  // model

  function onProgress(xhr) {
    if (xhr.lengthComputable) {
      const percentComplete = (xhr.loaded / xhr.total) * 100
      console.log('model ' + percentComplete.toFixed(2) + '% downloaded')
    }
  }

  function onError() {}

  const loader = new OBJLoader(manager)
  loader.load(
    'Pipe.obj',
    function (obj) {
      object = obj
    },
    onProgress,
    onError
  )

  const headManager = new THREE.LoadingManager(render)
  const headLoader = new THREE.TextureLoader(headManager)
  const loaderEXR = new EXRLoader(manager)
  const normalmap = headLoader.load('human_head/smooth.jpg')
  const matcap = loaderEXR.load('human_head/040full.exr')

  new GLTFLoader(headManager).load('human_head/LeePerrySmith.glb', (gltf) => {
    const mesh = gltf.scene.children[0]
    mesh.position.y = 0.4
    mesh.position.z = 4.7
    mesh.material = new THREE.MeshMatcapMaterial({
      color: new THREE.Color().setHex(0xffffff).convertSRGBToLinear(),
      matcap: matcap,
      normalMap: normalmap
    })
    mesh.rotation.set(0, Math.PI, 0)
    mesh.scale.set(0.03, 0.03, 0.03)

    scene.add(mesh)
  })

  const floorLoader = new FBXLoader(manager)
  floorLoader.load('/Floor.FBX', function (obj) {
    obj.traverse((child) => {
      if (child.isMesh) {
        child.material = floorMaterial
        child.material.needsupdate = true
      }
    })
    obj.scale.set(0.02, 0.02, 0.02)
    scene.add(obj)
    obj.position.y = -4
  })

  // Create an SVGLoader instance
  const svgLoader = new SVGLoader()
  svgLoader.load('/yellow.svg', (data) => {
    const paths = data.paths

    group = new THREE.Group()
    for (let i = 0; i < paths.length; i++) {
      const path = paths[i]

      const svgMaterial = new THREE.MeshBasicMaterial({
        color: path.color,
        side: THREE.DoubleSide,
        depthWrite: false
      })

      const shapes = SVGLoader.createShapes(path)

      for (let j = 0; j < shapes.length; j++) {
        const shape = shapes[j]
        const svgGeometry = new THREE.ShapeGeometry(shape)
        const svgMesh = new THREE.Mesh(svgGeometry, svgMaterial)
        group.add(svgMesh)
      }
    }
    group.scale.set(0.0005, 0.0005, 0.0005)
    group.position.set(0, 0, 3.9)

    group.name = 'svg'
    scene.add(group)
  })

  // Create a cube
  const geometry = new THREE.BoxGeometry(0.05, 0.05, 0.05)
  const material = new THREE.MeshBasicMaterial({ color: 0x0000ff })
  cube = new THREE.Mesh(geometry, material)
  cube.name = 'cube'
  scene.add(cube)

  cube.position.x = 0
  cube.position.z = 3.8

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setPixelRatio(window.devicePixelRatio)
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setClearColor(0xffffff)
  document.body.appendChild(renderer.domElement)

  controls = new OrbitControls(camera, renderer.domElement)
  // const gui = new dat.GUI();
  controls.enableZoom = false
  controls.enablePan = false

  //
  window.addEventListener('resize', onWindowResize)
}

function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight
  const pixelRatio = window.devicePixelRatio

  camera.updateProjectionMatrix()

  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(pixelRatio)
}

function createPanel() {
  const panel = new GUI({ width: 310 })

  settings = {
    cameraView: 'First person View',
    width: 1.0,
    speed: 0.5,
    rotation: 0,
    transparency: 1,
    direction: 'Left',
    gradient: false,
    color: false
  }

  panel.add(settings, 'cameraView', ['First person View', 'Third person View']).onChange(() => {
    if (settings.cameraView == 'First person View') {
      cameraView = 0
    }
    if (settings.cameraView == 'Third person View') {
      cameraView = 1
    }
  })

  panel.add(settings, 'width', 0, 1, 0.01).onChange((value) => {
    gWidth = settings.width
    shaderMaterial.uniforms.frequency.value = value * 156
    console.log(gWidth)
  })
  panel.add(settings, 'speed', 0, 1).onChange(() => {
    gSpeed = settings.speed
  })
  panel.add(settings, 'rotation', 0, 2 * Math.PI).onChange(() => {
    gRotation = settings.rotation
    console.log(gRotation)
  })
  panel.add(settings, 'transparency', 0, 1).onChange((value) => {
    gTransparency = settings.transparency
    shaderMaterial.uniforms.opacity.value = value
    console.log(gTransparency)
  })
  panel.add(settings, 'direction', ['Up', 'Down', 'Right', 'Left']).onChange(() => {
    if (settings.direction === 'Up') {
      gDirection = 0
    }
    if (settings.direction === 'Down') {
      gDirection = 1
    }
    if (settings.direction === 'Right') {
      gDirection = 2
    }
    if (settings.direction === 'Left') {
      gDirection = 3
    }
  })
  panel.add(settings, 'gradient').name('gradient').onChange((value)=>{
    gGradient = value;
  })
  panel.add(settings, 'color').name('color')
}

function render() {
  renderer.render(scene, camera)
}

let lol = 0
let lol1 = 0
let lat = 0
let phi = 0
// eslint-disable-next-line no-unused-vars
let theta = 0
// eslint-disable-next-line no-unused-vars
let angle = 0

let clock = new THREE.Clock()
let x1 = 0
let z1 = 0

function animate() {
  requestAnimationFrame(animate)

  let dt = clock.getDelta()
  // let dt1 = clock.getDelta();

  angle += 0.005

  lol += 20 * dt * gSpeed

  lat = Math.max(-85, Math.min(85, lat))
  phi = THREE.MathUtils.degToRad(90 - lat)
  theta = THREE.MathUtils.degToRad(lol)

  let rad = 700 * Math.sin(phi)

  scene.traverse((child) => {
    const name = child.name
    if (name === 'cylinder') {
      shaderMaterial.uniforms.gradient.value = gGradient;
      // console.log(gGradient)
      child.rotation.y = lol / 100
      if (gDirection === 3) {
        child.rotation.y = lol / 100
      }
      if (gDirection === 2) {
        child.rotation.y = -lol / 100
      }
      // if(gDirection === 1) { child.rotation.x = - Math.PI/2 }
      // if(gDirection === 0) { child.rotation.x = Math.PI/2 }
      child.rotation.z = gRotation
    }
    if (name === 'cube') {
      child.rotation.y = lol / 100
    }
    if (name === 'svg') {
      document.addEventListener('keydown', (event) => {
        switch (event.keyCode) {
          case 68:
            lol1 += 0.01
            x1 = rad * Math.cos(THREE.MathUtils.degToRad(lol1) - Math.PI / 2)
            z1 = rad * Math.sin(THREE.MathUtils.degToRad(lol1) - Math.PI / 2)
            child.position.x = x1 / 620
            child.position.z = z1 / 620 + 5
            break
          case 65:
            lol1 -= 0.01
            x1 = rad * Math.cos(THREE.MathUtils.degToRad(lol1) - Math.PI / 2)
            z1 = rad * Math.sin(THREE.MathUtils.degToRad(lol1) - Math.PI / 2)
            child.position.x = x1 / 620
            child.position.z = z1 / 620 + 5
            break
          default:
            break
        }
      })
    }
  })

  if (cameraView === 0) {
    camera.position.set(0, 0, 5)
    camera.fov = 45
    camera.lookAt(0, 0, 0)
    camera.updateProjectionMatrix()
  }

  if (cameraView === 1) {
    camera.position.set(0, 8, 9)
    camera.fov = 25
    camera.rotation.set(0, 30, 0)
    camera.lookAt(cube.position)
    camera.updateProjectionMatrix()
  }
  renderer.render(scene, camera)
}

animate()
</script>

<style></style>
