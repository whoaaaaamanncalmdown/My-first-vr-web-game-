
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <title>BO2 VR Prototype</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />

    <!-- Three.js (module version) -->
    <script type="module">
        import * as THREE from "https://unpkg.com/three@0.160.0/build/three.module.js";
        import { VRButton } from "https://unpkg.com/three@0.160.0/examples/jsm/webxr/VRButton.js";

        let scene, camera, renderer, clock;
        let playerRig;
        let leftController, rightController;
        let leftHandMesh, rightHandMesh, pistolMesh;

        // Temp vectors reused each frame
        const forwardVec = new THREE.Vector3();
        const sideVec = new THREE.Vector3();

        init();
        animate();

        function init() {
            // Scene
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x000000);

            // Camera + player rig (so we move the rig, not the camera directly)
            camera = new THREE.PerspectiveCamera(
                70,
                window.innerWidth / window.innerHeight,
                0.1,
                200
            );

            playerRig = new THREE.Group();
            playerRig.add(camera);
            scene.add(playerRig);

            // Lights
            const hemiLight = new THREE.HemisphereLight(0xffffff, 0x444444, 0.8);
            hemiLight.position.set(0, 50, 0);
            scene.add(hemiLight);

            const dirLight = new THREE.DirectionalLight(0xffffff, 0.6);
            dirLight.position.set(10, 20, 10);
            scene.add(dirLight);

            // Renderer
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setPixelRatio(window.devicePixelRatio);
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.xr.enabled = true;
            document.body.style.margin = "0";
            document.body.style.overflow = "hidden";
            document.body.appendChild(renderer.domElement);

            // Custom "Play in VR" button using VRButton under the hood
            const vrDomButton = VRButton.createButton(renderer);
            vrDomButton.style.display = "none"; // hide default
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

            customBtn.onclick = () => {
                // Trigger the hidden VRButton
                vrDomButton.click();
            };

            document.body.appendChild(customBtn);

            // Clock
            clock = new THREE.Clock();

            // Build the map
            createMap();

            // VR controllers / hands / pistol
            setupControllers();

            // Handle window resize
            window.addEventListener("resize", onWindowResize);
        }

        function createMap() {
            // Floor (big square arena)
            const floorSize = 40;
            const floorGeo = new THREE.PlaneGeometry(floorSize, floorSize);
            const floorMat = new THREE.MeshStandardMaterial({
                color: 0x222222,
                metalness: 0.1,
                roughness: 0.8
            });
            const floor = new THREE.Mesh(floorGeo, floorMat);
            floor.rotation.x = -Math.PI / 2;
            floor.receiveShadow = true;
            scene.add(floor);

            // Perimeter walls (simple BO2-ish arena)
            const wallHeight = 3;
            const wallThickness = 0.5;
            const wallColor = 0x333333;
            const wallMat = new THREE.MeshStandardMaterial({
                color: wallColor,
                metalness: 0.2,
                roughness: 0.7
            });

            // Front wall
            addWall(0, wallHeight / 2, -floorSize / 2, floorSize, wallHeight, wallThickness, wallMat);
            // Back wall
            addWall(0, wallHeight / 2, floorSize / 2, floorSize, wallHeight, wallThickness, wallMat);
            // Left wall
            addWall(-floorSize / 2, wallHeight / 2, 0, wallThickness, wallHeight, floorSize, wallMat);
            // Right wall
            addWall(floorSize / 2, wallHeight / 2, 0, wallThickness, wallHeight, floorSize, wallMat);

            // Cover objects in the middle (like boxes / crates)
            const coverMat = new THREE.MeshStandardMaterial({
                color: 0x555555,
                metalness: 0.3,
                roughness: 0.6
            });

            const crateGeo = new THREE.BoxGeometry(1.2, 1.2, 1.2);

            const cratePositions = [
                [ -4, 0.6, -4 ],
                [  4, 0.6, -4 ],
                [ -4, 0.6,  4 ],
                [  4, 0.6,  4 ],
                [  0, 0.9,  0 ]
            ];

            cratePositions.forEach(([x, y, z]) => {
                const crate = new THREE.Mesh(crateGeo, coverMat);
                crate.position.set(x, y, z);
                crate.castShadow = true;
                crate.receiveShadow = true;
                scene.add(crate);
            });

            // Slight fog for mood
            scene.fog = new THREE.Fog(0x000000, 20, 60);
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
            // LEFT CONTROLLER (movement)
            leftController = renderer.xr.getController(0);
            leftController.addEventListener("connected", (event) => {
                // Simple hand mesh for left
                leftHandMesh = createHandMesh(0x00ffcc);
                leftController.add(leftHandMesh);
            });
            leftController.addEventListener("disconnected", () => {
                if (leftHandMesh) leftController.remove(leftHandMesh);
                leftHandMesh = null;
            });
            scene.add(leftController);

            // RIGHT CONTROLLER (pistol)
            rightController = renderer.xr.getController(1);
            rightController.addEventListener("connected", (event) => {
                // Right hand
                rightHandMesh = createHandMesh(0xffcc00);
                rightController.add(rightHandMesh);

                // Pistol mesh attached to right hand
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
                metalness: 0.0,
                roughness: 0.7
            });
            const mesh = new THREE.Mesh(geo, mat);
            mesh.position.set(0, -0.03, -0.05);
            return mesh;
        }

        function createPistolMesh() {
            const group = new THREE.Group();

            // Handle
            const handleGeo = new THREE.BoxGeometry(0.03, 0.08, 0.02);
            const handleMat = new THREE.MeshStandardMaterial({
                color: 0x111111,
                metalness: 0.5,
                roughness: 0.4
            });
            const handle = new THREE.Mesh(handleGeo, handleMat);
            handle.position.set(0, -0.04, 0);
            group.add(handle);

            // Barrel
            const barrelGeo = new THREE.BoxGeometry(0.02, 0.02, 0.18);
            const barrelMat = new THREE.MeshStandardMaterial({
                color: 0x222222,
                metalness: 0.7,
                roughness: 0.3
            });
            const barrel = new THREE.Mesh(barrelGeo, barrelMat);
            barrel.position.set(0, 0.0, -0.09);
            group.add(barrel);

            // Slide
            const slideGeo = new THREE.BoxGeometry(0.03, 0.03, 0.14);
            const slideMat = new THREE.MeshStandardMaterial({
                color: 0x555555,
                metalness: 0.8,
                roughness: 0.3
            });
            const slide = new THREE.Mesh(slideGeo, slideMat);
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

            // Movement with left joystick
            handleLeftJoystickMovement(delta);

            renderer.render(scene, camera);
        }

        function handleLeftJoystickMovement(delta) {
            const session = renderer.xr.getSession();
            if (!session) return;

            const speed = 3.0; // meters per second

            for (const source of session.inputSources) {
                if (!source.gamepad) continue;
                if (source.handedness !== "left") continue;

                const axes = source.gamepad.axes;
                if (axes.length < 2) continue;

                const xAxis = axes[0]; // left/right
                const yAxis = axes[1]; // forward/back (usually -forward, +back)

                // Only move if stick is actually being used
                const deadzone = 0.15;
                if (Math.abs(xAxis) < deadzone && Math.abs(yAxis) < deadzone) {
                    continue;
                }

                // Forward direction based on camera
                camera.getWorldDirection(forwardVec);
                forwardVec.y = 0;
                forwardVec.normalize();

                // Side direction
                sideVec.crossVectors(camera.up, forwardVec).normalize();

                // Move rig (not camera directly)
                const moveAmount = speed * delta;
                playerRig.position.addScaledVector(forwardVec, -yAxis * moveAmount);
                playerRig.position.addScaledVector(sideVec, -xAxis * moveAmount);
            }
        }
    </script>
</head>
<body></body>
</html>
