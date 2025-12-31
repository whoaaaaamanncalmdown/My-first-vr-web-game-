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
scene.background = new THREE.Color(0x87ceeb); // Day sky

const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 1.6, 3);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.xr.enabled = true;
document.body.appendChild(renderer.domElement);
document.body.appendChild(VRButton.createButton(renderer));

/* LIGHTING */
scene.add(new THREE.HemisphereLight(0xffffff, 0x888888, 1.2));
const sun = new THREE.DirectionalLight(0xffffff, 0.8);
sun.position.set(5, 10, 7);
scene.add(sun);

/* FLOOR */
const floor = new THREE.Mesh(
  new THREE.PlaneGeometry(50, 50),
  new THREE.MeshStandardMaterial({ color: 0x55aa55 })
);
floor.rotation.x = -Math.PI / 2;
scene.add(floor);

/* CUBES */
const cubes = [];
for (let i = 0; i < 15; i++) {
  const cube = new THREE.Mesh(
    new THREE.BoxGeometry(),
    new THREE.MeshStandardMaterial({ color: 0xff8844 })
  );
  cube.position.set(
    Math.random() * 10 - 5,
    Math.random() * 3 + 1,
    Math.random() * -10
  );
  scene.add(cube);
  cubes.push(cube);
}

/* XR RIG */
const player = new THREE.Group();
player.add(camera);
scene.add(player);

/* CONTROLLERS */
const leftController = renderer.xr.getController(0);
const rightController = renderer.xr.getController(1);
scene.add(leftController);
scene.add(rightController);

/* HANDS */
function hand(color) {
  return new THREE.Mesh(
    new THREE.BoxGeometry(0.08, 0.08, 0.15),
    new THREE.MeshStandardMaterial({ color })
  );
}
leftController.add(hand(0x2222ff));
rightController.add(hand(0xff2222));

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
rightController.add(gun);

/* JOYSTICK MOVEMENT */
const speed = 0.05;

renderer.setAnimationLoop(() => {
  const session = renderer.xr.getSession();
  if (session) {
    for (const source of session.inputSources) {
      if (!source.gamepad || source.handedness !== "left") continue;

      const [xAxis, yAxis] = source.gamepad.axes;

      // Direction based on headset
      const dir = new THREE.Vector3();
      camera.getWorldDirection(dir);
      dir.y = 0;
      dir.normalize();

      const right = new THREE.Vector3();
      right.crossVectors(dir, camera.up).normalize();

      player.position.addScaledVector(dir, -yAxis * speed);
      player.position.addScaledVector(right, xAxis * speed);
    }
  }

  cubes.forEach(c => {
    c.rotation.x += 0.01;
    c.rotation.y += 0.01;
  });

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
