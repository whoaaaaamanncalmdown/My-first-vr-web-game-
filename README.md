
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Web Gorilla VR</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<style>
  body { margin: 0; overflow: hidden; background: black; }
  #ui {
    position: absolute;
    top: 10px;
    left: 10px;
    color: white;
    font-family: Arial, sans-serif;
    font-size: 20px;
    z-index: 10;
  }
</style>
</head>
<body>
<div id="ui">Score: <span id="score">0</span></div>

<script type="module">
import * as THREE from "https://cdn.jsdelivr.net/npm/three@0.161.0/build/three.module.js";
import { VRButton } from "https://cdn.jsdelivr.net/npm/three@0.161.0/examples/jsm/webxr/VRButton.js";

/* =======================
   BASIC SETUP
======================= */
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87ceeb);

const camera = new THREE.PerspectiveCamera(70, innerWidth / innerHeight, 0.1, 1000);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.xr.enabled = true;
document.body.appendChild(renderer.domElement);
document.body.appendChild(VRButton.createButton(renderer));

/* =======================
   XR RIG (IMPORTANT)
======================= */
const rig = new THREE.Group();
scene.add(rig);
rig.add(camera);

/* =======================
   LIGHTING
======================= */
scene.add(new THREE.HemisphereLight(0xffffff, 0x888888, 1.2));
const sun = new THREE.DirectionalLight(0xffffff, 0.8);
sun.position.set(5, 10, 7);
scene.add(sun);

/* =======================
   FLOOR
======================= */
const floor = new THREE.Mesh(
  new THREE.PlaneGeometry(200, 200),
  new THREE.MeshStandardMaterial({ color: 0x55aa55 })
);
floor.rotation.x = -Math.PI / 2;
scene.add(floor);

/* =======================
   TARGET CUBES
======================= */
const cubes = [];
for (let i = 0; i < 20; i++) {
  const cube = new THREE.Mesh(
    new THREE.BoxGeometry(),
    new THREE.MeshStandardMaterial({ color: 0xff3333 })
  );
  cube.position.set(
    Math.random() * 12 - 6,
    Math.random() * 3 + 1,
    Math.random() * -12
  );
  scene.add(cube);
  cubes.push(cube);
}

/* =======================
   CONTROLLERS
======================= */
const leftController = renderer.xr.getController(0);
const rightController = renderer.xr.getController(1);
rig.add(leftController);
rig.add(rightController);

/* =======================
   HAND VISUALS
======================= */
function makeHand(color) {
  return new THREE.Mesh(
    new THREE.SphereGeometry(0.08, 16, 16),
    new THREE.MeshStandardMaterial({ color })
  );
}
leftController.add(makeHand(0x2222ff));
rightController.add(makeHand(0xff2222));

/* =======================
   FAKE GUN
======================= */
const gun = new THREE.Mesh(
  new THREE.BoxGeometry(0.04, 0.04, 0.25),
  new THREE.MeshStandardMaterial({ color: 0x333333 })
);
gun.position.set(0, 0, -0.15);
rightController.add(gun);

/* =======================
   GORILLA LOCOMOTION (FIXED)
======================= */
const prevLeft = new THREE.Vector3();
const prevRight = new THREE.Vector3();
const velocity = new THREE.Vector3();

const MAX_SPEED = 0.15;
const DAMPING = 0.88;
const PUSH_FORCE = 1.4;
const GROUND_Y = 1.2;

function gorillaStep(controller, prevPos) {
  const world = new THREE.Vector3();
  controller.getWorldPosition(world);

  const local = world.clone();
  rig.worldToLocal(local);

  if (local.y < GROUND_Y) {
    const delta = local.clone().sub(prevPos);
    velocity.sub(delta.multiplyScalar(PUSH_FORCE));
  }

  prevPos.copy(local);
}

/* =======================
   SHOOTING + SCORE
======================= */
let score = 0;
const scoreText = document.getElementById("score");
const raycaster = new THREE.Raycaster();

rightController.addEventListener("selectstart", () => {
  const origin = new THREE.Vector3();
  rightController.getWorldPosition(origin);

  const dir = new THREE.Vector3(0, 0, -1)
    .applyQuaternion(rightController.quaternion)
    .normalize();

  raycaster.set(origin, dir);
  const hits = raycaster.intersectObjects(cubes);

  if (hits.length > 0) {
    const hit = hits[0].object;
    scene.remove(hit);
    cubes.splice(cubes.indexOf(hit), 1);
    score++;
    scoreText.textContent = score;
  }
});

/* =======================
   MAIN LOOP
======================= */
renderer.setAnimationLoop(() => {
  gorillaStep(leftController, prevLeft);
  gorillaStep(rightController, prevRight);

  velocity.multiplyScalar(DAMPING);
  velocity.clampLength(0, MAX_SPEED);
  rig.position.add(velocity);

  renderer.render(scene, camera);
});

/* =======================
   RESIZE
======================= */
addEventListener("resize", () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
});
</script>
</body>
</html>
