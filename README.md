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

const HAND_SMOOTH = 0.25;   // higher = smoother hands
const BODY_SMOOTH = 0.18;  // higher = smoother body
const FRICTION = 0.985;    // closer to 1 = more glide
const DEADZONE = 0.002;

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

