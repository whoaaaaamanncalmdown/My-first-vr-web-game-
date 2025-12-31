
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>BO2 VR Prototype</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <script type="module">
    import * as THREE from "https://unpkg.com/three@0.160.0/build/three.module.js";
    import { VRButton } from "https://unpkg.com/three@0.160.0/examples/jsm/webxr/VRButton.js";

    let scene, camera, renderer, clock;
    let playerRig;
    let leftController, rightController;
    let leftHandMesh, rightHandMesh, pistolMesh;

    const forwardVec = new THREE.Vector3();
    const sideVec = new THREE.Vector3();

    init();
    animate();

    function init() {
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x000000);

      camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 200);
      playerRig = new THREE.Group();
      playerRig.add(camera);
      scene.add(playerRig);

      const hemiLight = new THREE.HemisphereLight(0xffffff, 0x444444, 0.8);
      hemiLight.position.set(0, 50, 0);
      scene.add(hemiLight);

      const dirLight = new THREE.DirectionalLight(0xffffff, 0.6);
      dirLight.position.set(10, 20, 10);
      scene.add(dirLight);

      renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.xr.enabled = true;
      document.body.style.margin = "0";
      document.body.style.overflow = "hidden";
      document.body.appendChild(renderer.domElement);

      const vrDomButton = VRButton.createButton(renderer);
      vrDomButton.style.display = "none";
      document.body.appendChild(vrDomButton);

      const customBtn = document.createElement("button");
      customBtn.textContent = "Play in VR";
      customBtn.style.position = "absolute";
      customBtn.style.bottom = "20px";
      customBtn.style.left = "50%";
      customBtn.style.transform = "translateX(-50%)";
      customBtn.style.padding = "12px 24px";
      customBtn.style.fontSize = "18px";
      customBtn.style.borderRadius = "8px";
      customBtn.style.border = "none";
      customBtn.style.cursor = "pointer";
      customBtn.style.background = "#00bcd4";
      customBtn.style.color = "#000";
      customBtn.style.fontFamily = "system-ui, sans-serif";
      customBtn.style.zIndex = "10";
      customBtn.onclick = () => vrDomButton.click();
      document.body.appendChild(customBtn);

      clock = new THREE.Clock();

      createMap();
      setupControllers();

      window.addEventListener("resize", onWindowResize);
    }

    function createMap() {
      const floorSize = 40;
      const floorGeo = new THREE.PlaneGeometry(floorSize, floorSize);
      const floorMat = new THREE.MeshStandardMaterial({ color: 0x222222 });
      const floor = new THREE.Mesh(floorGeo, floorMat);
      floor.rotation.x = -Math.PI / 2;
      scene.add(floor);

      const wallMat = new THREE.MeshStandardMaterial({ color: 0x333333 });
      addWall(0, 1.5, -floorSize / 2, floorSize, 3, 0.5, wallMat);
      addWall(0, 1.5, floorSize / 2, floorSize, 3, 0.5, wallMat);
      addWall(-floorSize / 2, 1.5, 0, 0.5, 3, floorSize, wallMat);
      addWall(floorSize / 2, 1.5, 0, 0.5, 3, floorSize, wallMat);

      const crateMat = new THREE.MeshStandardMaterial({ color: 0x555555 });
      const crateGeo = new THREE.BoxGeometry(1.2, 1.2, 1.2);
      const positions = [ [-4, 0.6, -4], [4, 0.6, -4], [-4, 0.6, 4], [4, 0.6, 4], [0, 0.9, 0] ];
      positions.forEach(([x, y, z]) => {
        const crate = new THREE.Mesh(crateGeo, crateMat);
        crate.position.set(x, y, z);
        scene.add(crate);
      });

      scene.fog = new THREE.Fog(0x000000, 20, 60);
    }

    function addWall(x, y, z, w, h, d, mat) {
      const geo = new THREE.BoxGeometry(w, h, d);
      const wall = new THREE.Mesh(geo, mat);
      wall.position.set(x, y, z);
      scene.add(wall);
    }

    function setupControllers() {
      leftController = renderer.xr.getController(0);
      leftController.addEventListener("connected", () => {
        leftHandMesh = createHandMesh(0x00ffcc);
        leftController.add(leftHandMesh);
      });
      leftController.addEventListener("disconnected", () => {
        if (leftHandMesh) leftController.remove(leftHandMesh);
        leftHandMesh = null;
      });
      scene.add(leftController);

      rightController = renderer.xr.getController(1);
      rightController.addEventListener("connected", () => {
        rightHandMesh = createHandMesh(0xffcc00);
        rightController.add(rightHandMesh);
        pistolMesh = createPistolMesh();
        pistolMesh.position.set(0, -0.03, -0.15);
        pistolMesh.rotation.x = -Math.PI / 2;
        rightController.add(pistolMesh);
      });
      rightController.addEventListener("disconnected", () => {
        if (rightHandMesh) rightController.remove(rightHandMesh);
        if (pistolMesh) rightController.remove(pistolMesh);
        rightHandMesh = null;
        pistolMesh = null;
      });
      scene.add(rightController);
    }

    function createHandMesh(color) {
      const geo = new THREE.BoxGeometry(0.06, 0.1, 0.16);
      const mat = new THREE.MeshStandardMaterial({ color });
      const mesh = new THREE.Mesh(geo, mat);
      mesh.position.set(0, -0.03, -0.05);
      return mesh;
    }

    function createPistolMesh() {
      const group = new THREE.Group();
      const handle = new THREE.Mesh(new THREE.BoxGeometry(0.03, 0.08, 0.02), new THREE.MeshStandardMaterial({ color: 0x111111 }));
      handle.position.set(0, -0.04, 0);
      group.add(handle);
      const barrel = new THREE.Mesh(new THREE.BoxGeometry(0.02, 0.02, 0.18), new THREE.MeshStandardMaterial({ color: 0x222222 }));
      barrel.position.set(0, 0.0, -0.09);
      group.add(barrel);
      const slide = new THREE.Mesh(new THREE.BoxGeometry(0.03, 0.03, 0.14), new THREE.MeshStandardMaterial({ color: 0x555555 }));
      slide.position.set(0, 0.01, -0.08);
      group.add(slide);
      return group;
    }

    function onWindowResize() {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    }

    function animate() {
      renderer.setAnimationLoop(renderLoop);
    }

    function renderLoop() {
      const delta = clock.getDelta();
      handleLeftJoystickMovement(delta);
      renderer.render(scene, camera);
    }

    function handleLeftJoystickMovement(delta) {
      const session = renderer.xr.getSession();
      if (!session) return;

      const speed = 3.0;

      for (const source of session.inputSources) {
        if (!source.gamepad || source.handedness !== "left") continue;
        const axes = source.gamepad.axes;
        if (axes.length < 2) continue;

        const xAxis = axes[0];
        const yAxis = axes[1];
        const deadzone = 0.15;
        if (Math.abs(xAxis) < deadzone && Math.abs(yAxis) < deadzone) continue;

        camera.getWorldDirection(forwardVec);
        forwardVec.y = 0;
        forwardVec.normalize();
        sideVec.crossVectors(camera.up, forwardVec).normalize();

        const moveAmount = speed * delta;
