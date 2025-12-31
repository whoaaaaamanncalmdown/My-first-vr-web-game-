
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>BO2 VR Prototype - Single File</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <!-- Three.js and WebXR utilities from CDN -->
  <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/webxr/VRButton.js"></script>
</head>
<body style="margin:0; overflow:hidden; background:#000;">
<script>
  // Basic THREE.js setup
  let scene, camera, renderer, clock;

  init();
  animate();

  function init() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x101010);

    // Camera (used as the reference for player position)
    camera = new THREE.PerspectiveCamera(
      70,
      window.innerWidth / window.innerHeight,
      0.1,
      100
    );

    // Lights
    const hemiLight = new THREE.HemisphereLight(0xffffff, 0x444444, 0.8);
    hemiLight.position.set(0, 20, 0);
    scene.add(hemiLight);

    const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
    dirLight.position.set(5, 10, 2);
    scene.add(dirLight);

    // Renderer
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.xr.enabled = true;
    document.body.appendChild(renderer.domElement);

    // VR button
    document.body.appendChild(THREE.VRButton.createButton(renderer));

    clock = new THREE.Clock();

    // Create map
    createMap();

    // Set up VR controllers / hands / pistol
    setupControllers();

    window.addEventListener("resize", onWindowResize, false);
  }

  function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }

  // Simple map: floor + a few walls
  function createMap() {
    // Floor
    const floorGeo = new THREE.PlaneGeometry(20, 20);
    const floorMat = new THREE.MeshStandardMaterial({
      color: 0x303030,
      roughness: 1
    });
    const floor = new THREE.Mesh(floorGeo, floorMat);
    floor.rotation.x = -Math.PI / 2;
    floor.receiveShadow = true;
    scene.add(floor);

    // Helper function to place walls
    function addWall(x, z, width, height, depth) {
      const wallGeo = new THREE.BoxGeometry(width, height, depth);
      const wallMat = new THREE.MeshStandardMaterial({ color: 0x555555 });
      const wall = new THREE.Mesh(wallGeo, wallMat);
      wall.position.set(x, height / 2, z);
      wall.castShadow = true;
      wall.receiveShadow = true;
      scene.add(wall);
    }

    // A few simple walls to walk around
    addWall(0, -5, 8, 2.5, 0.4);   // front wall
    addWall(-5, 0, 0.4, 2.5, 8);   // left wall
    addWall(5, 0, 0.4, 2.5, 8);    // right wall
    addWall(0, 5, 8, 2.5, 0.4);    // back wall

    // Some boxes in the middle
    function addBox(x, z) {
      const boxGeo = new THREE.BoxGeometry(1, 1, 1);
      const boxMat = new THREE.MeshStandardMaterial({ color: 0x777777 });
      const box = new THREE.Mesh(boxGeo, boxMat);
      box.position.set(x, 0.5, z);
      scene.add(box);
    }

    addBox(-2, -2);
    addBox(2, -1);
    addBox(0, 2);
  }

  // Controllers / hands / pistol
  let leftController, rightController;
  let leftHand, rightHand, gun;

  function setupControllers() {
    // Controller 0 and 1
    leftController = renderer.xr.getController(0);
    rightController = renderer.xr.getController(1);

    scene.add(leftController);
    scene.add(rightController);

    // Hand geometry (simple boxes)
    const handGeo = new THREE.BoxGeometry(0.08, 0.08, 0.2);
    const leftMat = new THREE.MeshStandardMaterial({ color: 0x00aaff });
    const rightMat = new THREE.MeshStandardMaterial({ color: 0xffaa00 });

    leftHand = new THREE.Mesh(handGeo, leftMat);
    rightHand = new THREE.Mesh(handGeo, rightMat);

    // Attach them to controllers
    leftController.add(leftHand);
    rightController.add(rightHand);

    leftHand.position.set(0, 0, 0);
    rightHand.position.set(0, 0, 0);

    // Simple pistol attached to right hand
    const gunGeo = new THREE.BoxGeometry(0.25, 0.12, 0.4);
    const gunMat = new THREE.MeshStandardMaterial({ color: 0x222222 });
    gun = new THREE.Mesh(gunGeo, gunMat);

    // Attach the gun to the right hand
    rightHand.add(gun);
    gun.position.set(0, -0.05, -0.25);
  }

  // Main loop
  function animate() {
    renderer.setAnimationLoop(render);
  }

  function render() {
    const delta = clock.getDelta();
    handleMovement(delta);
    renderer.render(scene, camera);
  }

  // Left joystick movement using WebXR gamepad axes
  function handleMovement(delta) {
    const session = renderer.xr.getSession();
    if (!session) return;

    const speed = 2.0; // meters per second
    const inputSources = session.inputSources;

    for (const source of inputSources) {
      if (!source.gamepad) continue;

      const gp = source.gamepad;
      const axes = gp.axes || [];

      // Many devices use axes[2], axes[3] for stick 1, some use [0],[1]
      let xAxis = 0;
      let yAxis = 0;

      // Try to detect which pair has the strongest input
      const pairs = [
        [0, 1],
        [2, 3]
      ];

      let bestMag = 0;
      let bestPair = [0, 1];

      for (const p of pairs) {
        const ax = axes[p[0]] || 0;
        const ay = axes[p[1]] || 0;
        const mag = Math.sqrt(ax * ax + ay * ay);
        if (mag > bestMag) {
          bestMag = mag;
          bestPair = p;
        }
      }

      xAxis = axes[bestPair[0]] || 0;
      yAxis = axes[bestPair[1]] || 0;

      // Use left-handed controller for movement
      if (source.handedness === "left") {
        // Movement relative to headset facing direction
        const xrCamera = renderer.xr.getCamera(camera);
        const dir = new THREE.Vector3();
        xrCamera.getWorldDirection(dir);
        dir.y = 0;
        dir.normalize();

        // Right vector
        const right = new THREE.Vector3();
        right.crossVectors(dir, new THREE.Vector3(0, 1, 0)).normalize();

        const moveForward = dir.multiplyScalar(-yAxis * speed * delta);
        const moveRight = right.multiplyScalar(xAxis * speed * delta);

        const move = moveForward.add(moveRight);

        // Move the "parent" camera position
        camera.position.add(move);
      }
    }
  }
</script>
</body>
</html>
