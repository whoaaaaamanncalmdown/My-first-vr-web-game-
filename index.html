
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <title>BO2 VR Prototype (Dolly Locomotion)</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background: #000;
            font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
        }
        #playVrButton {
            position: absolute;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            padding: 14px 28px;
            font-size: 18px;
            border-radius: 10px;
            border: none;
            cursor: pointer;
            background: #00bcd4;
            color: #000;
            z-index: 10;
        }
        #playVrButton:hover {
            background: #14d4ec;
        }
    </style>

    <script type="module">
        import * as THREE from "https://unpkg.com/three@0.160.0/build/three.module.js";
        import { VRButton } from "https://unpkg.com/three@0.160.0/examples/jsm/webxr/VRButton.js";

        let scene, camera, renderer, clock;
        let dolly;                   // like your zombie shooter
        let leftController, rightController;
        let leftHandMesh, rightHandMesh, pistolMesh;

        const forwardVec = new THREE.Vector3();
        const sideVec = new THREE.Vector3();

        init();
        animate();

        function init() {
            // Scene
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x000000);
            scene.fog = new THREE.Fog(0x000000, 20, 60);

            // Camera
            camera = new THREE.PerspectiveCamera(
                75,
                window.innerWidth / window.innerHeight,
                0.1,
                200
            );

            // DOLLY (player rig) – this is what we move
            dolly = new THREE.Group();
            dolly.position.set(0, 1.6, 10); // start a bit into the map
            dolly.add(camera);
            scene.add(dolly);

            // Lights
            const hemiLight = new THREE.HemisphereLight(0xffffff, 0x444444, 0.8);
            hemiLight.position.set(0, 50, 0);
            scene.add(hemiLight);

            const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
            dirLight.position.set(10, 20, 10);
            dirLight.castShadow = true;
            scene.add(dirLight);

            // Renderer
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setPixelRatio(window.devicePixelRatio);
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.xr.enabled = true;
            document.body.appendChild(renderer.domElement);

            // Native VR button (hidden)
            const vrButton = VRButton.createButton(renderer);
            vrButton.style.display = "none";
            document.body.appendChild(vrButton);

            // Custom "Play in VR" button
            const playBtn = document.createElement("button");
            playBtn.id = "playVrButton";
            playBtn.textContent = "Play in VR";
            playBtn.onclick = () => vrButton.click();
            document.body.appendChild(playBtn);

            // Clock
            clock = new THREE.Clock();

            // Map / arena
            createMap();

            // Controllers + hands + pistol
            setupControllers();

            // Resize
            window.addEventListener("resize", onWindowResize);
        }

        function createMap() {
            const floorSize = 40;

            // Floor
            const floorGeo = new THREE.PlaneGeometry(floorSize, floorSize);
            const floorMat = new THREE.MeshStandardMaterial({
                color: 0x222222,
                roughness: 0.85,
                metalness: 0.1
            });
            const floor = new THREE.Mesh(floorGeo, floorMat);
            floor.rotation.x = -Math.PI / 2;
            floor.receiveShadow = true;
            scene.add(floor);

            // Walls (simple BO2-ish arena box)
            const wallMat = new THREE.MeshStandardMaterial({
                color: 0x333333,
                roughness: 0.7,
                metalness: 0.2
            });

            addWall(0, 1.5, -floorSize / 2, floorSize, 3, 0.5, wallMat);  // front
            addWall(0, 1.5,  floorSize / 2, floorSize, 3, 0.5, wallMat);  // back
            addWall(-floorSize / 2, 1.5, 0, 0.5, 3, floorSize, wallMat);  // left
            addWall( floorSize / 2, 1.5, 0, 0.5, 3, floorSize, wallMat);  // right

            // Cover boxes
            const crateMat = new THREE.MeshStandardMaterial({
                color: 0x555555,
                roughness: 0.6,
                metalness: 0.3
            });
            const crateGeo = new THREE.BoxGeometry(1.2, 1.2, 1.2);
            const crates = [
                [-4, 0.6, -4],
                [ 4, 0.6, -4],
                [-4, 0.6,  4],
                [ 4, 0.6,  4],
                [ 0, 0.9,  0]
            ];
            crates.forEach(([x, y, z]) => {
                const crate = new THREE.Mesh(crateGeo, crateMat);
                crate.position.set(x, y, z);
                crate.castShadow = true;
                crate.receiveShadow = true;
                scene.add(crate);
            });
        }

        function addWall(x, y, z, w, h, d, mat) {
            const geo = new THREE.BoxGeometry(w, h, d);
            const wall = new THREE.Mesh(geo, mat);
            wall.position.set(x, y, z);
            wall.castShadow = true;
            wall.receiveShadow = true;
            scene.add(wall);
        }

        function setupControllers() {
            // LEFT CONTROLLER (movement + left hand)
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

            // RIGHT CONTROLLER (right hand + pistol)
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
            const mat = new THREE.MeshStandardMaterial({
                color,
                roughness: 0.7,
                metalness: 0.0
            });
            const mesh = new THREE.Mesh(geo, mat);
            mesh.position.set(0, -0.03, -0.05);
            return mesh;
        }

        function createPistolMesh() {
            const group = new THREE.Group();

            const handle = new THREE.Mesh(
                new THREE.BoxGeometry(0.03, 0.08, 0.02),
                new THREE.MeshStandardMaterial({
                    color: 0x111111,
                    metalness: 0.5,
                    roughness: 0.4
                })
            );
            handle.position.set(0, -0.04, 0);
            group.add(handle);

            const barrel = new THREE.Mesh(
                new THREE.BoxGeometry(0.02, 0.02, 0.18),
                new THREE.MeshStandardMaterial({
                    color: 0x222222,
                    metalness: 0.7,
                    roughness: 0.3
                })
            );
            barrel.position.set(0, 0.0, -0.09);
            group.add(barrel);

            const slide = new THREE.Mesh(
                new THREE.BoxGeometry(0.03, 0.03, 0.14),
                new THREE.MeshStandardMaterial({
                    color: 0x555555,
                    metalness: 0.8,
                    roughness: 0.3
                })
            );
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
            handleLocomotion(delta);
            renderer.render(scene, camera);
        }

        // Get joystick axes for a controller, preferring axes[2,3] then falling back
        function getJoystickAxes(source) {
            if (!source.gamepad) return { x: 0, y: 0 };

            const axes = source.gamepad.axes || [];
            let x = 0;
            let y = 0;

            // Prefer 2 & 3 (common for Quest)
            if (axes.length >= 4) {
                x = axes[2] ?? 0;
                y = axes[3] ?? 0;
            }

            // If those are basically zero, try 0 & 1
            if (Math.abs(x) < 0.01 && Math.abs(y) < 0.01 && axes.length >= 2) {
                x = axes[0] ?? 0;
                y = axes[1] ?? 0;
            }

            return { x, y };
        }

        function handleLocomotion(delta) {
            const session = renderer.xr.getSession();
            if (!session || !dolly) return;

            const speed = 4.0;
            const deadzone = 0.15;

            for (const source of session.inputSources) {
                if (!source.gamepad) continue;
                if (source.handedness !== "left") continue; // left stick only

                const { x, y } = getJoystickAxes(source);

                if (Math.abs(x) < deadzone && Math.abs(y) < deadzone) continue;

                // Head-directed movement
                const xrCamera = renderer.xr.getCamera(camera);
                xrCamera.getWorldDirection(forwardVec);
                forwardVec.y = 0;
                forwardVec.normalize();

                sideVec.crossVectors(xrCamera.up, forwardVec).normalize();

                const move = speed * delta;

                // Move dolly, like your zombie shooter
                dolly.position.addScaledVector(forwardVec, -y * move);
                dolly.position.addScaledVector(sideVec, -x * move);

                // Only use first left controller found
                break;
            }
        }
    </script>
</head>
<body></body>
</html>
