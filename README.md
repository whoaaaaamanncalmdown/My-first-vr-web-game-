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
const handVelLeft = new THREE.Vector3();
const handVelRight = new THREE.Vector3();
const bodyVelocity = new THREE.Vector3();
const targetVelocity = new THREE.Vector3();

/* TUNING (SMOOTH FEEL) */
const CONTACT_Y = 1.4;
const PUSH_FORCE = 2.0;
const MAX_SPEED = 0.25;
const HAND_SMOOTH = 0.25;  // higher = smoother hands
const BODY_SMOOTH = 0.18;  // higher = smoother body
const FRICTION = 0.985;    // closer to 1 = more glide
const DEADZONE = 0.002;    // minimum speed for movement to count

function gorillaStep(controller, prevPos, handVel) {
  const world = new THREE.Vector3();
  controller.getWorldPosition(world);

  const local = world.clone();
  rig.worldToLocal(local);

  const delta = local.clone().sub(prevPos);

  // Smooth hand velocity
  handVel.lerp(delta, HAND_SMOOTH);

  if (local.y < CONTACT_Y && handVel.lengthSq() > DEADZONE) {
    targetVelocity.sub(handVel.clone().multiplyScalar(PUSH_FORCE));
  }

  prevPos.copy(local);
}

/* =======================
   MAIN LOOP (FIXED ANIMATION LOOP)
======================= */
renderer.setAnimationLoop(() => {
  targetVelocity.set(0, 0, 0);

  // Update the hand velocities using the gorillaStep function
  gorillaStep(leftController, prevLeft, handVelLeft);
  gorillaStep(rightController, prevRight, handVelRight);

  // Smooth body velocity
  bodyVelocity.lerp(targetVelocity, BODY_SMOOTH);

  // Apply friction to slow down movement
  bodyVelocity.multiplyScalar(FRICTION);

  // Clamp the speed to MAX_SPEED
  bodyVelocity.clampLength(0, MAX_SPEED);

  // Update the rig's position based on the velocity
  rig.position.add(bodyVelocity);

  // Ensure the rig stays above the ground (clamp Y position)
  if (rig.position.y < CONTACT_Y) {
    rig.position.y = CONTACT_Y;
  }

  // Render the scene
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

