
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>VR Zombie Shooter</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            font-family: Arial, sans-serif;
        }
        #startButton {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            padding: 20px 40px;
            font-size: 24px;
            background: #ff4444;
            color: white;
            border: none;
            border-radius: 10px;
            cursor: pointer;
            z-index: 100;
        }
        #startButton:hover {
            background: #ff6666;
        }
        #info {
            position: absolute;
            top: 10px;
            left: 10px;
            color: white;
            background: rgba(0,0,0,0.7);
            padding: 10px;
            border-radius: 5px;
            font-size: 14px;
            max-width: 300px;
        }
        #debug {
            position: absolute;
            top: 60px;
            left: 10px;
            color: white;
            background: rgba(0,0,0,0.5);
            padding: 10px;
            border-radius: 5px;
            font-size: 11px;
            font-family: monospace;
            max-width: 400px;
            line-height: 1.4;
        }
    </style>
</head>
<body>
    <button id="startButton">PLAY VR ZOMBIE SHOOTER</button>
    <div id="info">Score: <span id="score">0</span> | Zombies: <span id="zombieCount">0</span></div>
    <div id="debug">Joystick: waiting...</div>
    
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script>
        let scene, camera, renderer, gun, zombies = [], score = 0;
        let controller1, controller2, dolly;
        let moveVector = new THREE.Vector2();
        
        const startButton = document.getElementById('startButton');
        
        startButton.addEventListener('click', init);
        
        function init() {
            startButton.style.display = 'none';
            
            // Scene setup
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x1a1a2e);
            scene.fog = new THREE.Fog(0x1a1a2e, 10, 50);
            
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            
            // Dolly for movement
            dolly = new THREE.Group();
            dolly.position.set(0, 0, 10);
            dolly.add(camera);
            scene.add(dolly);
            
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.xr.enabled = true;
            document.body.appendChild(renderer.domElement);
            
            // Lighting
            const ambient = new THREE.AmbientLight(0xffffff, 0.3);
            scene.add(ambient);
            
            const light = new THREE.DirectionalLight(0xffffff, 0.8);
            light.position.set(10, 20, 10);
            scene.add(light);
            
            // Create map (floor and walls)
            createMap();
            
            // Create gun
            createGun();
            
            // Setup VR controllers
            setupControllers();
            
            // Spawn initial zombies
            for (let i = 0; i < 5; i++) {
                spawnZombie();
            }
            
            // Create VR button
            const vrButton = createVRButton();
            document.body.appendChild(vrButton);
            
            renderer.xr.addEventListener('sessionstart', () => {
                console.log('VR Session started');
            });
            
            renderer.setAnimationLoop(animate);
            
            window.addEventListener('resize', onWindowResize);
        }
        
        function createMap() {
            // Floor
            const floorGeo = new THREE.PlaneGeometry(50, 50);
            const floorMat = new THREE.MeshStandardMaterial({ 
                color: 0x2d4a2b,
                roughness: 0.8
            });
            const floor = new THREE.Mesh(floorGeo, floorMat);
            floor.rotation.x = -Math.PI / 2;
            floor.position.y = -1;
            scene.add(floor);
            
            // Walls
            const wallMat = new THREE.MeshStandardMaterial({ color: 0x4a4a4a });
            
            // Back wall
            const wall1 = new THREE.Mesh(new THREE.BoxGeometry(50, 10, 1), wallMat);
            wall1.position.set(0, 4, -25);
            scene.add(wall1);
            
            // Left wall
            const wall2 = new THREE.Mesh(new THREE.BoxGeometry(1, 10, 50), wallMat);
            wall2.position.set(-25, 4, 0);
            scene.add(wall2);
            
            // Right wall
            const wall3 = new THREE.Mesh(new THREE.BoxGeometry(1, 10, 50), wallMat);
            wall3.position.set(25, 4, 0);
            scene.add(wall3);
            
            // Add some obstacles
            for (let i = 0; i < 8; i++) {
                const box = new THREE.Mesh(
                    new THREE.BoxGeometry(2, 3, 2),
                    new THREE.MeshStandardMaterial({ color: 0x663300 })
                );
                box.position.set(
                    Math.random() * 30 - 15,
                    1.5,
                    Math.random() * 30 - 15
                );
                scene.add(box);
            }
        }
        
        function createGun() {
            gun = new THREE.Group();
            
            // Gun body
            const body = new THREE.Mesh(
                new THREE.BoxGeometry(0.1, 0.1, 0.5),
                new THREE.MeshStandardMaterial({ color: 0x222222 })
            );
            body.position.z = -0.25;
            gun.add(body);
            
            // Barrel
            const barrel = new THREE.Mesh(
                new THREE.CylinderGeometry(0.02, 0.02, 0.3),
                new THREE.MeshStandardMaterial({ color: 0x333333 })
            );
            barrel.rotation.x = Math.PI / 2;
            barrel.position.z = -0.55;
            gun.add(barrel);
            
            // Handle
            const handle = new THREE.Mesh(
                new THREE.BoxGeometry(0.08, 0.15, 0.08),
                new THREE.MeshStandardMaterial({ color: 0x1a1a1a })
            );
            handle.position.set(0, -0.1, -0.15);
            gun.add(handle);
        }
        
        function setupControllers() {
            controller1 = renderer.xr.getController(0);
            controller2 = renderer.xr.getController(1);
            
            // Attach gun to right controller
            controller2.add(gun);
            dolly.add(controller2);
            dolly.add(controller1);
            
            // Trigger shooting
            controller2.addEventListener('selectstart', shoot);
        }
        
        function shoot() {
            // Create bullet
            const bullet = new THREE.Mesh(
                new THREE.SphereGeometry(0.05),
                new THREE.MeshBasicMaterial({ color: 0xffff00 })
            );
            
            const gunWorldPos = new THREE.Vector3();
            gun.getWorldPosition(gunWorldPos);
            bullet.position.copy(gunWorldPos);
            
            const direction = new THREE.Vector3(0, 0, -1);
            direction.applyQuaternion(gun.getWorldQuaternion(new THREE.Quaternion()));
            bullet.userData.velocity = direction.multiplyScalar(1.5);
            bullet.userData.life = 60;
            
            scene.add(bullet);
            
            // Muzzle flash
            const flash = new THREE.PointLight(0xffaa00, 2, 3);
            flash.position.copy(gunWorldPos);
            scene.add(flash);
            setTimeout(() => scene.remove(flash), 50);
        }
        
        function spawnZombie() {
            const zombie = new THREE.Mesh(
                new THREE.BoxGeometry(0.8, 1.8, 0.8),
                new THREE.MeshStandardMaterial({ color: 0x00ff00 })
            );
            
            const angle = Math.random() * Math.PI * 2;
            const distance = 20 + Math.random() * 10;
            zombie.position.set(
                Math.cos(angle) * distance,
                0.9,
                Math.sin(angle) * distance
            );
            
            zombie.userData.health = 3;
            zombie.userData.speed = 0.02 + Math.random() * 0.02;
            
            scene.add(zombie);
            zombies.push(zombie);
            
            document.getElementById('zombieCount').textContent = zombies.length;
        }
        
        function animate() {
            // Get current session and read joystick input
            const session = renderer.xr.getSession();
            
            let debugText = 'VR Status: ';
            
            if (!session) {
                debugText += 'Not in VR mode';
                document.getElementById('debug').textContent = debugText;
            } else {
                debugText += 'In VR | Controllers: ' + session.inputSources.length + '\n';
                
                let leftFound = false;
                let rightFound = false;
                
                // Process all input sources
                for (let source of session.inputSources) {
                    const hand = source.handedness;
                    debugText += `${hand} hand: `;
                    
                    if (source.gamepad) {
                        const axes = source.gamepad.axes;
                        const buttons = source.gamepad.buttons;
                        debugText += `${axes.length} axes, ${buttons.length} btns\n`;
                        debugText += `  Axes: ${axes.map((v, i) => `${i}:${v.toFixed(2)}`).join(' ')}\n`;
                        
                        // LEFT CONTROLLER - MOVEMENT
                        if (hand === 'left') {
                            leftFound = true;
                            let x = 0, y = 0;
                            
                            // Try all possible axis configurations
                            if (axes.length >= 4) {
                                x = axes[2];
                                y = axes[3];
                            } else if (axes.length >= 2) {
                                x = axes[0];
                                y = axes[1];
                            }
                            
                            debugText += `  Move: X=${x.toFixed(2)} Y=${y.toFixed(2)}\n`;
                            
                            // Check if joystick is being moved
                            if (Math.abs(x) > 0.15 || Math.abs(y) > 0.15) {
                                const speed = 0.15;
                                
                                // Get camera direction
                                const direction = new THREE.Vector3();
                                camera.getWorldDirection(direction);
                                direction.y = 0;
                                direction.normalize();
                                
                                // Get right vector
                                const right = new THREE.Vector3();
                                right.crossVectors(direction, camera.up);
                                right.normalize();
                                
                                // Move dolly
                                dolly.position.add(direction.multiplyScalar(-y * speed));
                                dolly.position.add(right.multiplyScalar(x * speed));
                                
                                debugText += `  MOVING! Pos: ${dolly.position.x.toFixed(1)},${dolly.position.z.toFixed(1)}\n`;
                            }
                        }
                        
                        // RIGHT CONTROLLER - TURNING
                        if (hand === 'right') {
                            rightFound = true;
                            let turnX = 0;
                            
                            // Try all possible axis configurations
                            if (axes.length >= 4) {
                                turnX = axes[2];
                            } else if (axes.length >= 2) {
                                turnX = axes[0];
                            }
                            
                            debugText += `  Turn: X=${turnX.toFixed(2)}\n`;
                            
                            // Smooth turn when right joystick is moved left/right
                            if (Math.abs(turnX) > 0.2) {
                                const turnSpeed = 0.02;
                                dolly.rotation.y -= turnX * turnSpeed;
                                debugText += `  TURNING! Rot: ${(dolly.rotation.y * 180 / Math.PI).toFixed(1)}°\n`;
                            }
                        }
                    } else {
                        debugText += 'No gamepad\n';
                    }
                }
                
                if (!leftFound) debugText += 'LEFT CONTROLLER NOT FOUND!\n';
                if (!rightFound) debugText += 'RIGHT CONTROLLER NOT FOUND!\n';
                
                document.getElementById('debug').textContent = debugText;
            }
            
            // Update zombies
            zombies.forEach((zombie, index) => {
                const direction = new THREE.Vector3();
                direction.subVectors(dolly.position, zombie.position);
                direction.y = 0;
                direction.normalize();
                
                zombie.position.add(direction.multiplyScalar(zombie.userData.speed));
                zombie.lookAt(dolly.position);
            });
            
            // Update bullets
            scene.children.forEach(child => {
                if (child.userData.velocity) {
                    child.position.add(child.userData.velocity);
                    child.userData.life--;
                    
                    // Check collision with zombies
                    zombies.forEach((zombie, zIndex) => {
                        if (child.position.distanceTo(zombie.position) < 1) {
                            zombie.userData.health--;
                            scene.remove(child);
                            
                            if (zombie.userData.health <= 0) {
                                scene.remove(zombie);
                                zombies.splice(zIndex, 1);
                                score += 100;
                                document.getElementById('score').textContent = score;
                                document.getElementById('zombieCount').textContent = zombies.length;
                                
                                // Spawn new zombie
                                if (Math.random() < 0.7) spawnZombie();
                            }
                        }
                    });
                    
                    if (child.userData.life <= 0) {
                        scene.remove(child);
                    }
                }
            });
            
            // Spawn more zombies if needed
            if (zombies.length < 3) {
                spawnZombie();
            }
            
            renderer.render(scene, camera);
        }
        
        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }
        
        function createVRButton() {
            const button = document.createElement('button');
            button.style.position = 'absolute';
            button.style.bottom = '20px';
            button.style.left = '50%';
            button.style.transform = 'translateX(-50%)';
            button.style.padding = '12px 24px';
            button.style.border = 'none';
            button.style.borderRadius = '4px';
            button.style.background = '#1183d6';
            button.style.color = '#fff';
            button.style.font = 'normal 13px sans-serif';
            button.style.cursor = 'pointer';
            button.textContent = 'ENTER VR';
            
            button.onclick = function() {
                if (navigator.xr) {
                    navigator.xr.isSessionSupported('immersive-vr').then((supported) => {
                        if (supported) {
                            navigator.xr.requestSession('immersive-vr', {
                                optionalFeatures: ['local-floor', 'bounded-floor']
                            }).then((session) => {
                                renderer.xr.setSession(session);
                            });
                        } else {
                            button.textContent = 'VR NOT SUPPORTED';
                        }
                    });
                } else {
                    button.textContent = 'VR NOT AVAILABLE';
                }
            };
            
            return button;
        }
    </script>
</body>
</html>
