
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>BO2 VR Prototype</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/webxr/VRButton.js"></script>
</head>
<body style="margin:0; overflow:hidden; background:#000;">
<script>
  let scene, camera, renderer, clock;
  let leftController, rightController;
  let leftHand, rightHand, gun;

  init();
  animate();

  function init() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x101010);

    camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 100);

    const hemiLight = new THREE.HemisphereLight(0xffffff, 0x444444, 0.8);
    hemiLight.position.set(0, 20, 0);
    scene.add(hemiLight);

    const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
    dirLight.position.set(5, 10, 2);
    scene.add(dirLight);

    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.xr.enabled = true;
    document.body.appendChild(renderer.domElement);

    // ✅ Add the "Enter VR" button
    document.body.appendChild(THREE.VRButton.createButton(renderer));

    clock = new THREE.Clock();

    createMap();
    setupControllers();

    window.addEventListener("resize", onWindowResize, false);
  }

  function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }

  function createMap() {
    const floor = new THREE.Mesh(
      new THREE.PlaneGeometry(20, 20),
      new THREE.MeshStandardMaterial({ color: 0x303030, roughness: 1 })
    );
    floor.rotation.x = -Math.PI / 2;
    scene.add(floor);

    function addWall(x, z, w, h, d) {
      const wall = new THREE.Mesh(
        new THREE.BoxGeometry(w, h, d),
        new THREE.MeshStandardMaterial({ color: 0x555555 })
      );
      wall.position.set(x, h / 2, z);
      scene.add(wall);
    }

    addWall(0, -5, 8, 2.5, 0.4);
    addWall(-5, 0, 0.4, 2.5, 8);
    addWall(5, 0, 0.4, 2.5, 8);
    addWall(0, 5, 8, 2.5, 0.4);

    function addBox(x, z) {
      const box = new THREE.Mesh(
        new THREE.BoxGeometry(1, 1, 1),
        new THREE.MeshStandardMaterial({ color: 0x777777 })
      );
      box.position.set(x, 0.5, z);
      scene.add(box);
    }

    addBox(-2, -2);
    addBox(2, -1);
    addBox(0, 2);
  }

  function setupControllers() {
    leftController = renderer.xr.getController(0);
    rightController = renderer.xr.getController(1);
    scene.add(leftController);
    scene.add(rightController);

    const handGeo = new THREE.BoxGeometry(0.08, 0.08, 0.2);
    const leftMat = new THREE.MeshStandardMaterial({ color: 0x00aaff });
    const rightMat = new THREE.MeshStandardMaterial({ color: 0xffaa00 });

    leftHand = new THREE.Mesh(handGeo, leftMat);
    rightHand = new THREE.Mesh(handGeo, rightMat);

    leftController.add(leftHand);
    rightController.add(rightHand);

    leftHand.position.set(0, 0, 0);
    rightHand.position.set(0, 0, 0);

    const gunGeo = new THREE.BoxGeometry(0.25, 0.12, 0.4);
    const gunMat = new THREE.MeshStandardMaterial({ color: 0x222222 });
    gun = new THREE.Mesh(gunGeo, gunMat);
    rightHand.add(gun);
    gun.position.set(0, -0.05, -0.25);
  }

  function animate() {
    renderer.setAnimationLoop(render);
  }

  function render() {
    const delta = clock.getDelta();
    handleMovement(delta);
    renderer.render(scene, camera);
  }

  function handleMovement(delta) {
    const session = renderer.xr.getSession();
    if (!session) return;

    const speed = 2.0;
    const inputSources = session.inputSources;

    for (const source of inputSources) {
      if (!source.gamepad) continue;
      const gp = source.gamepad;
      const axes = gp.axes || [];

      let xAxis = 0;
      let yAxis = 0;
      const pairs = [[0, 1], [2, 3]];
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

      if (source.handedness === "left") {
        const xrCamera = renderer.xr.getCamera(camera);
        const dir = new THREE.Vector3();
        xrCamera.getWorldDirection(dir);
        dir.y = 0;
        dir.normalize();

        const right = new THREE.Vector3();
        right.crossVectors(dir, new THREE.Vector3(0, 1, 0)).normalize();

        const moveForward = dir.multiplyScalar(-yAxis * speed * delta);
        const moveRight = right.multiplyScalar(xAxis * speed * delta);
        const move = moveForward.add(moveRight);

        camera.position.add(move);
      }
    }
  }
</script>
</body>
</html>
