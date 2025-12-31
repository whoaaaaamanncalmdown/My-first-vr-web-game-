
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
        #debug {
            position: absolute;
            top: 10px;
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
    <div id="debug">Move joysticks...</div>
    
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script>
        let scene, camera, renderer, currentGun, zombies = [];
        let controller1, controller2, dolly;
        let score = 0, points = 500, currentRound = 1, zombiesThisRound = 5, zombiesKilled = 0;
        let ammo = 30, maxAmmo = 30;
        let hudCanvas, hudTexture, hudMesh;
        
        // Gun stats
        let guns = {
            pistol: { name: 'Pistol', damage: 1, fireRate: 300, maxAmmo: 30, packed: false },
            rifle: { name: 'Rifle', damage: 2, fireRate: 150, maxAmmo: 60, packed: false, cost: 1500 },
            shotgun: { name: 'Shotgun', damage: 3, fireRate: 500, maxAmmo: 24, packed: false, cost: 2000 }
        };
        
        let currentGunType = 'pistol';
        let canShoot = true;
        let wallGuns = [];
        let packAPunchMachine = null;
        
        const startButton = document.getElementById('startButton');
        startButton.addEventListener('click', init);
        
        function init() {
            startButton.style.display = 'none';
            
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x1a1a2e);
            scene.fog = new THREE.Fog(0x1a1a2e, 10, 50);
            
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            
            dolly = new THREE.Group();
            dolly.position.set(0, 0, 10);
            dolly.add(camera);
            scene.add(dolly);
            
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.xr.enabled = true;
            document.body.appendChild(renderer.domElement);
            
            const ambient = new THREE.AmbientLight(0xffffff, 0.3);
            scene.add(ambient);
            
            const light = new THREE.DirectionalLight(0xffffff, 0.8);
            light.position.set(10, 20, 10);
            scene.add(light);
            
            createMap();
            createGun();
            createInGameHUD();
            setupControllers();
            createWallGuns();
            createPackAPunch();
            startRound(1);
            
            const vrButton = createVRButton();
            document.body.appendChild(vrButton);
            
            renderer.setAnimationLoop(animate);
            window.addEventListener('resize', onWindowResize);
        }
        
        function createInGameHUD() {
            // Create canvas for HUD
            hudCanvas = document.createElement('canvas');
            hudCanvas.width = 512;
            hudCanvas.height = 256;
            
            // Create texture from canvas
            hudTexture = new THREE.CanvasTexture(hudCanvas);
            
            // Create HUD mesh
            const hudGeometry = new THREE.PlaneGeometry(1.5, 0.75);
            const hudMaterial = new THREE.MeshBasicMaterial({ 
                map: hudTexture, 
                transparent: true,
                opacity: 0.9
            });
            hudMesh = new THREE.Mesh(hudGeometry, hudMaterial);
            hudMesh.position.set(-0.6, 0.4, -1);
            camera.add(hudMesh);
            
            updateInGameHUD();
        }
        
        function updateInGameHUD() {
            const ctx = hudCanvas.getContext('2d');
            
            // Clear canvas
            ctx.clearRect(0, 0, hudCanvas.width, hudCanvas.height);
            
            // Background
            ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
            ctx.fillRect(0, 0, hudCanvas.width, hudCanvas.height);
            
            // Round
            ctx.fillStyle = '#ff4444';
            ctx.font = 'bold 40px Arial';
            ctx.fillText(`ROUND ${currentRound}`, 20, 50);
            
            // Points
            ctx.fillStyle = '#ffff00';
            ctx.font = 'bold 36px Arial';
            ctx.fillText(`${points} PTS`, 20, 100);
            
            // Ammo
            ctx.fillStyle = '#ffffff';
            ctx.font = 'bold 32px Arial';
            ctx.fillText(`${ammo}/${maxAmmo}`, 20, 145);
            
            // Gun name
            const gunData = guns[currentGunType];
            ctx.fillStyle = gunData.packed ? '#ff00ff' : '#00ff00';
            ctx.font = 'bold 24px Arial';
            const gunText = gunData.packed ? gunData.name + ' [PaP]' : gunData.name;
            ctx.fillText(gunText, 20, 180);
            
            // Zombies
            ctx.fillStyle = '#00ff00';
            ctx.font = '28px Arial';
            ctx.fillText(`Zombies: ${zombies.length}`, 20, 220);
            
            hudTexture.needsUpdate = true;
        }
        
        function createMap() {
            const floorGeo = new THREE.PlaneGeometry(50, 50);
            const floorMat = new THREE.MeshStandardMaterial({ color: 0x2d4a2b, roughness: 0.8 });
            const floor = new THREE.Mesh(floorGeo, floorMat);
            floor.rotation.x = -Math.PI / 2;
            floor.position.y = -1;
            scene.add(floor);
            
            const wallMat = new THREE.MeshStandardMaterial({ color: 0x4a4a4a });
            
            const wall1 = new THREE.Mesh(new THREE.BoxGeometry(50, 10, 1), wallMat);
            wall1.position.set(0, 4, -25);
            scene.add(wall1);
            
            const wall2 = new THREE.Mesh(new THREE.BoxGeometry(1, 10, 50), wallMat);
            wall2.position.set(-25, 4, 0);
            scene.add(wall2);
            
            const wall3 = new THREE.Mesh(new THREE.BoxGeometry(1, 10, 50), wallMat);
            wall3.position.set(25, 4, 0);
            scene.add(wall3);
            
            for (let i = 0; i < 8; i++) {
                const box = new THREE.Mesh(
                    new THREE.BoxGeometry(2, 3, 2),
                    new THREE.MeshStandardMaterial({ color: 0x663300 })
                );
                box.position.set(Math.random() * 30 - 15, 1.5, Math.random() * 30 - 15);
                scene.add(box);
            }
        }
        
        function createWallGuns() {
            // Rifle on wall
            const rifleStation = createGunStation(-20, 1.5, 5, 0xff6600, 'RIFLE\n1500 pts');
            rifleStation.userData.gunType = 'rifle';
            rifleStation.userData.cost = 1500;
            wallGuns.push(rifleStation);
            
            // Shotgun on wall
            const shotgunStation = createGunStation(20, 1.5, 5, 0xffaa00, 'SHOTGUN\n2000 pts');
            shotgunStation.userData.gunType = 'shotgun';
            shotgunStation.userData.cost = 2000;
            wallGuns.push(shotgunStation);
        }
        
        function createGunStation(x, y, z, color, label) {
            const group = new THREE.Group();
            
            const base = new THREE.Mesh(
                new THREE.BoxGeometry(1.5, 2, 0.2),
                new THREE.MeshStandardMaterial({ color: color })
            );
            group.add(base);
            
            const gunModel = new THREE.Mesh(
                new THREE.BoxGeometry(0.15, 0.15, 0.6),
                new THREE.MeshStandardMaterial({ color: 0x222222 })
            );
            gunModel.position.set(0, 0, 0.3);
            group.add(gunModel);
            
            group.position.set(x, y, z);
            scene.add(group);
            return group;
        }
        
        function createPackAPunch() {
            packAPunchMachine = new THREE.Group();
            
            const machine = new THREE.Mesh(
                new THREE.BoxGeometry(2, 3, 2),
                new THREE.MeshStandardMaterial({ 
                    color: 0x9900ff,
                    emissive: 0x9900ff,
                    emissiveIntensity: 0.3
                })
            );
            packAPunchMachine.add(machine);
            
            const light = new THREE.PointLight(0x9900ff, 2, 10);
            light.position.y = 2;
            packAPunchMachine.add(light);
            
            packAPunchMachine.position.set(0, 1.5, -20);
            packAPunchMachine.userData.cost = 5000;
            scene.add(packAPunchMachine);
        }
        
        function createGun() {
            currentGun = new THREE.Group();
            updateGunModel();
        }
        
        function updateGunModel() {
            while(currentGun.children.length > 0) {
                currentGun.remove(currentGun.children[0]);
            }
            
            const gunData = guns[currentGunType];
            const isPacked = gunData.packed;
            
            const bodyColor = isPacked ? 0xff00ff : 0x222222;
            const barrelColor = isPacked ? 0x00ffff : 0x333333;
            
            const body = new THREE.Mesh(
                new THREE.BoxGeometry(0.1, 0.1, 0.5),
                new THREE.MeshStandardMaterial({ 
                    color: bodyColor,
                    emissive: isPacked ? bodyColor : 0x000000,
                    emissiveIntensity: isPacked ? 0.5 : 0
                })
            );
            body.position.z = -0.25;
            currentGun.add(body);
            
            const barrel = new THREE.Mesh(
                new THREE.CylinderGeometry(0.02, 0.02, 0.3),
                new THREE.MeshStandardMaterial({ 
                    color: barrelColor,
                    emissive: isPacked ? barrelColor : 0x000000,
                    emissiveIntensity: isPacked ? 0.5 : 0
                })
            );
            barrel.rotation.x = Math.PI / 2;
            barrel.position.z = -0.55;
            currentGun.add(barrel);
            
            const handle = new THREE.Mesh(
                new THREE.BoxGeometry(0.08, 0.15, 0.08),
                new THREE.MeshStandardMaterial({ color: 0x1a1a1a })
            );
            handle.position.set(0, -0.1, -0.15);
            currentGun.add(handle);
            
            document.getElementById('gunName').textContent = 
                gunData.packed ? gunData.name + ' (PACKED)' : gunData.name;
            updateInGameHUD();
        }
        
        function setupControllers() {
            controller1 = renderer.xr.getController(0);
            controller2 = renderer.xr.getController(1);
            
            controller2.add(currentGun);
            dolly.add(controller2);
            dolly.add(controller1);
            
            controller2.addEventListener('selectstart', shoot);
            controller2.addEventListener('squeezestart', interact);
        }
        
        function interact() {
            const controllerPos = new THREE.Vector3();
            controller2.getWorldPosition(controllerPos);
            
            // Check wall guns
            wallGuns.forEach(station => {
                if (controllerPos.distanceTo(station.position) < 3) {
                    if (points >= station.userData.cost) {
                        points -= station.userData.cost;
                        currentGunType = station.userData.gunType;
                        const gunData = guns[currentGunType];
                        ammo = gunData.maxAmmo;
                        maxAmmo = gunData.maxAmmo;
                        updateGunModel();
                        updateHUD();
                    }
                }
            });
            
            // Check Pack-a-Punch
            if (packAPunchMachine && controllerPos.distanceTo(packAPunchMachine.position) < 3) {
                if (points >= 5000 && !guns[currentGunType].packed) {
                    points -= 5000;
                    guns[currentGunType].packed = true;
                    guns[currentGunType].damage *= 3;
                    guns[currentGunType].fireRate = Math.max(50, guns[currentGunType].fireRate / 2);
                    guns[currentGunType].maxAmmo *= 2;
                    maxAmmo = guns[currentGunType].maxAmmo;
                    ammo = maxAmmo;
                    updateGunModel();
                    updateHUD();
                }
            }
        }
        
        function shoot() {
            if (!canShoot || ammo <= 0) return;
            
            canShoot = false;
            ammo--;
            updateHUD();
            
            const gunData = guns[currentGunType];
            setTimeout(() => { canShoot = true; }, gunData.fireRate);
            
            const bullet = new THREE.Mesh(
                new THREE.SphereGeometry(0.05),
                new THREE.MeshBasicMaterial({ 
                    color: gunData.packed ? 0xff00ff : 0xffff00 
                })
            );
            
            const gunWorldPos = new THREE.Vector3();
            currentGun.getWorldPosition(gunWorldPos);
            bullet.position.copy(gunWorldPos);
            
            const direction = new THREE.Vector3(0, 0, -1);
            direction.applyQuaternion(currentGun.getWorldQuaternion(new THREE.Quaternion()));
            bullet.userData.velocity = direction.multiplyScalar(1.5);
            bullet.userData.life = 60;
            bullet.userData.damage = gunData.damage;
            
            scene.add(bullet);
            
            const flashColor = gunData.packed ? 0xff00ff : 0xffaa00;
            const flash = new THREE.PointLight(flashColor, 2, 3);
            flash.position.copy(gunWorldPos);
            scene.add(flash);
            setTimeout(() => scene.remove(flash), 50);
            
            // Add points for shooting
            points += 10;
            updateHUD();
        }
        
        function startRound(round) {
            currentRound = round;
            zombiesKilled = 0;
            zombiesThisRound = 5 + (round - 1) * 3;
            
            // Create round notice in 3D space
            showRoundNotice(round);
            
            for (let i = 0; i < zombiesThisRound; i++) {
                setTimeout(() => spawnZombie(), i * 2000);
            }
            
            updateInGameHUD();
        }
        
        function showRoundNotice(round) {
            const canvas = document.createElement('canvas');
            canvas.width = 512;
            canvas.height = 256;
            const ctx = canvas.getContext('2d');
            
            ctx.fillStyle = '#ff4444';
            ctx.font = 'bold 80px Arial';
            ctx.textAlign = 'center';
            ctx.fillText(`ROUND ${round}`, 256, 150);
            
            const texture = new THREE.CanvasTexture(canvas);
            const material = new THREE.MeshBasicMaterial({ 
                map: texture, 
                transparent: true,
                side: THREE.DoubleSide
            });
            const geometry = new THREE.PlaneGeometry(4, 2);
            const notice = new THREE.Mesh(geometry, material);
            
            notice.position.set(0, 2, -5);
            scene.add(notice);
            
            setTimeout(() => scene.remove(notice), 3000);
        }
        
        function spawnZombie() {
            const zombie = new THREE.Group();
            
            const body = new THREE.Mesh(
                new THREE.BoxGeometry(0.8, 1.8, 0.8),
                new THREE.MeshStandardMaterial({ color: 0x00ff00 })
            );
            zombie.add(body);
            
            const healthBarBg = new THREE.Mesh(
                new THREE.PlaneGeometry(1, 0.1),
                new THREE.MeshBasicMaterial({ color: 0x000000 })
            );
            healthBarBg.position.y = 1.2;
            zombie.add(healthBarBg);
            
            const healthBar = new THREE.Mesh(
                new THREE.PlaneGeometry(1, 0.08),
                new THREE.MeshBasicMaterial({ color: 0xff0000 })
            );
            healthBar.position.set(0, 1.2, 0.01);
            zombie.add(healthBar);
            
            const angle = Math.random() * Math.PI * 2;
            const distance = 20 + Math.random() * 10;
            zombie.position.set(Math.cos(angle) * distance, 0.9, Math.sin(angle) * distance);
            
            zombie.userData.health = 2 + currentRound;
            zombie.userData.maxHealth = 2 + currentRound;
            zombie.userData.speed = 0.015 + (currentRound * 0.003);
            zombie.userData.healthBar = healthBar;
            
            scene.add(zombie);
            zombies.push(zombie);
            updateInGameHUD();
        }
        
        function updateHUD() {
            updateInGameHUD();
        }
        
        function animate() {
            const session = renderer.xr.getSession();
            let debugText = 'VR Status: ';
            
            if (!session) {
                debugText += 'Not in VR';
                document.getElementById('debug').textContent = debugText;
            } else {
                debugText += 'In VR | Controllers: ' + session.inputSources.length + '\n';
                
                for (let source of session.inputSources) {
                    const hand = source.handedness;
                    
                    if (source.gamepad) {
                        const axes = source.gamepad.axes;
                        
                        if (hand === 'left') {
                            let x = 0, y = 0;
                            if (axes.length >= 4) {
                                x = axes[2];
                                y = axes[3];
                            } else if (axes.length >= 2) {
                                x = axes[0];
                                y = axes[1];
                            }
                            
                            if (Math.abs(x) > 0.15 || Math.abs(y) > 0.15) {
                                const speed = 0.15;
                                const direction = new THREE.Vector3();
                                camera.getWorldDirection(direction);
                                direction.y = 0;
                                direction.normalize();
                                
                                const right = new THREE.Vector3();
                                right.crossVectors(direction, camera.up);
                                right.normalize();
                                
                                dolly.position.add(direction.multiplyScalar(-y * speed));
                                dolly.position.add(right.multiplyScalar(x * speed));
                            }
                        }
                        
                        if (hand === 'right') {
                            let turnX = 0;
                            if (axes.length >= 4) {
                                turnX = axes[2];
                            } else if (axes.length >= 2) {
                                turnX = axes[0];
                            }
                            
                            if (Math.abs(turnX) > 0.2) {
                                const turnSpeed = 0.02;
                                dolly.rotation.y -= turnX * turnSpeed;
                            }
                        }
                    }
                }
            }
            
            zombies.forEach((zombie, index) => {
                const direction = new THREE.Vector3();
                direction.subVectors(dolly.position, zombie.position);
                direction.y = 0;
                direction.normalize();
                
                zombie.position.add(direction.multiplyScalar(zombie.userData.speed));
                zombie.lookAt(new THREE.Vector3(dolly.position.x, zombie.position.y, dolly.position.z));
                
                zombie.userData.healthBar.lookAt(camera.position);
            });
            
            scene.children.forEach(child => {
                if (child.userData.velocity) {
                    child.position.add(child.userData.velocity);
                    child.userData.life--;
                    
                    zombies.forEach((zombie, zIndex) => {
                        if (child.position.distanceTo(zombie.position) < 1) {
                            zombie.userData.health -= child.userData.damage;
                            scene.remove(child);
                            
                            const healthPercent = zombie.userData.health / zombie.userData.maxHealth;
                            zombie.userData.healthBar.scale.x = Math.max(0, healthPercent);
                            
                            if (zombie.userData.health <= 0) {
                                scene.remove(zombie);
                                zombies.splice(zIndex, 1);
                                zombiesKilled++;
                                score += 100;
                                points += 100;
                                updateHUD();
                                
                                if (zombiesKilled >= zombiesThisRound) {
                                    setTimeout(() => startRound(currentRound + 1), 3000);
                                }
                            }
                        }
                    });
                    
                    if (child.userData.life <= 0) {
                        scene.remove(child);
                    }
                }
            });
            
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
