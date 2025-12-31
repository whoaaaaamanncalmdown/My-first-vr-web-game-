<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Browser VR Game</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<style>
  body { margin: 0; overflow: hidden; }
</style>
</head>
<body>

<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.161.0/build/three.module.js';
import { VRButton } from 'https://cdn.jsdelivr.net/npm/three@0.161.0/examples/jsm/webxr/VRButton.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87ceeb);

const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 1.6, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.xr.enabled = true;
document.body.appendChild(renderer.domElement);
document.body.appendChild(VRButton.createButton(renderer));

/* LIGHT */
scene.add(new THREE.HemisphereLight(0xffffff, 0x888888, 1.2));
const sun = new THREE.DirectionalLight(0xffffff, 0.8);
sun.position.set(5, 10, 7);
scene.add(sun);

/* FLOOR */
const floor = new THREE.Mesh(
  new THREE.PlaneGeometry(100, 100),
  new THREE.MeshStandardMaterial({ color: 0x55aa55 })
);
floor.rotation.x = -Math.PI / 2;
scene.add(floor);

/* PLAYER RIG */
const player = new THREE.Group();
player.add(camera);
scene.add(player);

/* CONTROLLERS */
const controller1 = renderer.xr.getController(0);
const controller2 = renderer.xr.getController(1);
scene.add(controller1, controller2);

/* HANDS */
function makeHand(color) {
  return new THREE.Mesh(
    new THREE.BoxGeometry(0.08, 0.08, 0.15),
    new THREE.MeshStandardMaterial({ color })
  );
}
controller1.add(makeHand(0x2222ff));
controller2.add(makeHand(0xff2222));

/* FAKE GUN */
const gun = new THREE.Group();
const handle = new THREE.Mesh(
  new THREE.BoxGeometry(0.05, 0.12, 0.05),
  new THREE.MeshStandardMaterial({ color: 0x222222 })
);
handle.position.y = -0.06;

const barrel = new THREE.Mesh(
  new THREE.BoxGeometry(0.04, 0.04, 0.25),
  new THREE.MeshStandardMaterial({ color: 0x444444 })
);
barrel.position.z = -0.15;

gun.add(handle, barrel);
gun.rotation.x = -Math.PI / 2;
gun.position.set(0, -0.02, -0.08);
controller2.add(gun);

/* JOYSTICK MOVEMENT (FIXED) */
const speed = 0.06;

renderer.setAnimationLoop(() => {
  const session = renderer.xr.getSession();
  if (session) {
    for (const source of session.inputSources) {
      if (!source.gamepad) continue;

      const axes = source.gamepad.axes;
      if (axes.length < 2) continue;

      const x = axes[0];
      const y = axes[1];

      if (Math.abs(x) < 0.1 && Math.abs(y) < 0.1) continue;

      const forward = new THREE.Vector3();
      camera.getWorldDirection(forward);
      forward.y = 0;
      forward.normalize();

      const right = new THREE.Vector3();
      right.crossVectors(forward, camera.up).normalize();

      player.position.addScaledVector(forward, -y * speed);
      player.position.addScaledVector(right, x * speed);
    }
  }

  renderer.render(scene, camera);
});

window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
</script>
</body>
</html>
