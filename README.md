<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Web Gorilla VR</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<style>
  body { margin: 0; overflow: hidden; }
  #score {
    position: absolute;
    top: 10px;
    left: 10px;
    color: white;
    font-family: Arial;
    font-size: 20px;
    z-index: 10;
  }
</style>
</head>
<body>
<div id="score">Score: 0</div>

<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.161.0/build/three.module.js';
import { VRButton } from 'https://cdn.jsdelivr.net/npm/three@0.161.0/examples/jsm/webxr/VRButton.js';

let score = 0;
const scoreDiv = document.getElementById("score");

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87ceeb);

/* CAMERA + PLAYER */
const camera = new THREE.PerspectiveCamera(70, window.innerWidth/window.innerHeight, 0.1, 1000);
camera.position.set(0, 1.6, 0);

const player = new THREE.Group();
player.add(camera);
scene.add(player);

/* RENDERER */
const renderer = new THREE.WebGLRenderer({ antialias:true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.xr.enabled = true;
document.body.appendChild(renderer.domElement);
document.body.appendChild(VRButton.createButton(renderer));

/* LIGHT */
scene.add(new THREE.HemisphereLight(0xffffff, 0x888888, 1.2));
const sun = new THREE.DirectionalLight(0xffffff, 0.8);
sun.position.set(5,10,7);
scene.add(sun);

/* FLOOR */
const floor = new THREE.Mesh(
  new THREE.PlaneGeometry(100,100),
  new THREE.MeshStandardMaterial({ color: 0x55aa55 })
);
floor.rotation.x = -Math.PI/2;
scene.add(floor);

/* CUBES (TARGETS) */
const cubes = [];
for (let i=0;i<20;i++){
  const cube = new THREE.Mesh(
    new THREE.BoxGeometry(),
    new THREE.MeshStandardMaterial({ color: 0xff4444 })
  );
  cube.position.set(
    Math.random()*10-5,
    Math.random()*3+1,
    Math.random()*-10
  );
  scene.add(cube);
  cubes.push(cube);
}

/* CONTROLLERS */
const leftController = renderer.xr.getController(0);
const rightController = renderer.xr.getController(1);
scene.add(leftController, rightController);

/* HANDS */
function makeHand(color){
  return new THREE.Mesh(
    new THREE.SphereGeometry(0.08),
    new THREE.MeshStandardMaterial({ color })
  );
}
const leftHand = makeHand(0x2222ff);
const rightHand = makeHand(0xff2222);
leftController.add(leftHand);
rightController.add(rightHand);

/* GORILLA LOCOMOTION */
let lastLeftPos = new THREE.Vector3();
let lastRightPos = new THREE.Vector3();
const velocity = new THREE.Vector3();

function applyGorilla(hand, lastPos){
  const worldPos = new THREE.Vector3();
  hand.getWorldPosition(worldPos);

  if (worldPos.y < 1.1) { // touching ground
    const delta = worldPos.clone().sub(lastPos);
    velocity.sub(delta.multiplyScalar(1.2));
  }
  lastPos.copy(worldPos);
}

/* GUN */
const gun = new THREE.Mesh(
  new THREE.BoxGeometry(0.04,0.04,0.25),
  new THREE.MeshStandardMaterial({ color: 0x333333 })
);
gun.position.set(0,0,-0.15);
rightController.add(gun);

/* SHOOTING */
const raycaster = new THREE.Raycaster();

rightController.addEventListener("selectstart", () => {
  const dir = new THREE.Vector3(0,0,-1).applyQuaternion(rightController.quaternion);
  const origin = new THREE.Vector3();
  rightController.getWorldPosition(origin);

  raycaster.set(origin, dir);
  const hits = raycaster.intersectObjects(cubes);

  if (hits.length > 0) {
    const hit = hits[0].object;
    scene.remove(hit);
    cubes.splice(cubes.indexOf(hit),1);
    score++;
    scoreDiv.textContent = "Score: " + score;
  }
});

/* LOOP */
renderer.setAnimationLoop(() => {
  applyGorilla(leftController, lastLeftPos);
  applyGorilla(rightController, lastRightPos);

  velocity.multiplyScalar(0.92);
  player.position.add(velocity);

  renderer.render(scene, camera);
});

/* RESIZE */
window.addEventListener("resize", ()=>{
  camera.aspect = window.innerWidth/window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
</script>
</body>
</html>
