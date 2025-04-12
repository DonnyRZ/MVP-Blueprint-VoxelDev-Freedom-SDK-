Okay, here is the code for the new `Fireball.js` file and the necessary modifications to the existing files (`Player.js`, `main.js`, `Ghast.js`, `GhastMovement.js`, `physics.js`) to implement the Ghast attack behavior and player health/respawn system.

---

**1. New File: `Fireball.js`**

```javascript
// src/Fireball.js
import * as THREE from 'three';

export class Fireball {
    constructor(scene, startPosition, direction, speed, damage, lifetime, owner) {
        this.scene = scene;
        this.speed = speed;
        this.damage = damage;
        this.lifetimeTimer = lifetime;
        this.owner = owner; // Reference to the Ghast that fired it
        this.toBeRemoved = false;

        // --- Geometry & Material ---
        const radius = 0.5; // Size of the fireball
        const segments = 8;
        const geometry = new THREE.SphereGeometry(radius, segments, segments);
        const material = new THREE.MeshStandardMaterial({
            color: 0xff4500, // Orange-red
            emissive: 0xcc2200, // Glowing effect
            emissiveIntensity: 1.5,
            roughness: 0.6,
            metalness: 0.1,
        });

        // --- Mesh ---
        this.mesh = new THREE.Mesh(geometry, material);
        this.mesh.position.copy(startPosition);
        this.mesh.castShadow = true; // Optional: Fireballs casting shadows
        this.mesh.name = "GhastFireball";
        this.scene.add(this.mesh);

        // --- Velocity ---
        this.velocity = direction.clone().normalize().multiplyScalar(this.speed);
    }

    update(deltaTime) {
        // Move the fireball
        this.mesh.position.addScaledVector(this.velocity, deltaTime);

        // Update lifetime
        this.lifetimeTimer -= deltaTime;
        if (this.lifetimeTimer <= 0) {
            this.toBeRemoved = true;
        }
    }

    // Call this when the fireball should be removed from the game
    destroy() {
        if (this.mesh.parent) {
            this.scene.remove(this.mesh);
        }
        if (this.mesh.geometry) {
            this.mesh.geometry.dispose();
        }
        if (this.mesh.material) {
            // Check if material is an array (though unlikely for simple sphere)
            if (Array.isArray(this.mesh.material)) {
                this.mesh.material.forEach(m => m.dispose());
            } else {
                this.mesh.material.dispose();
            }
        }
        // console.log("Fireball destroyed"); // Optional debug log
    }
}
```

---

**2. Modified File: `Player.js`**

```javascript
// src/player.js

import * as THREE from 'three';
import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';
import { Human } from './human.js';

export class Player {
    // Added activeGhasts to the constructor signature
    constructor(camera, domElement, scene, activeLaserBeams, physics, world, audioManager, activeGhasts) {
        this.camera = camera;
        this.domElement = domElement;
        this.scene = scene;
        this.activeLaserBeams = activeLaserBeams;
        this.physics = physics;
        this.world = world; // Store initial world reference
        this.audioManager = audioManager;
        this.activeGhasts = activeGhasts; // Store the passed array

        // --- Player Physics & State ---
        this.position = new THREE.Vector3(0, 20, 0); // Initial position (will be overwritten by spawn logic)
        this.visualPosition = this.position.clone();
        this.velocity = new THREE.Vector3();
        this.height = 1.75;
        this.radius = 0.5; // Overall radius for model scale
        this.cylinderRadius = 0.475; // Physics cylinder radius
        this.cylinderHeight = 2.35; // Physics cylinder height
        this.gravityEnabled = true;
        this.onGround = false;
        this.isFirstPerson = false;

        // --- Health & Status ---
        this.maxHealth = 120; // <<< NEW
        this.health = this.maxHealth; // <<< NEW
        this.isDead = false; // <<< NEW

        // --- Movement Input Flags ---
        this.moveForward = false;
        this.moveBackward = false;
        this.moveLeft = false;
        this.moveRight = false;
        this.moveUp = false; // For no-gravity mode
        this.moveDown = false; // For no-gravity mode

        // --- Jumping State ---
        this.jumpSpeed = 10.0; // <<< NEW (Define jump speed here)
        this.coyoteTimer = 0;
        this.coyoteTimeDuration = 0.1;
        this.jumpBufferTimer = 0;
        this.jumpBufferDuration = 0.15;

        // --- Combat State ---
        this.laserCooldownDuration = 0.2;
        this.laserCooldownTimer = 0;
        this.canShoot = true;
        this.laserDamageAmount = 10;

        // --- Controls & View ---
        this.controls = new PointerLockControls(this.camera, this.domElement);
        this.camera.position.copy(this.position); // Set initial camera position
        this.isFrontViewRequested = false;
        this.wasFirstPersonBeforeFrontView = false;
        this.previousCameraQuaternion = new THREE.Quaternion();

        // --- Human Model & Inventory ---
        this.human = new Human(this.scene, this.position, this.radius, this.height);
        this.isOneTogglePressed = false; // For saber toggle
        this.isTwoTogglePressed = false; // For laser gun toggle

        // --- Event Listeners ---
        document.addEventListener('keydown', this.onKeyDown.bind(this), false);
        document.addEventListener('keyup', this.onKeyUp.bind(this), false);
        document.addEventListener('mousedown', this.onMouseDown.bind(this), false);

        this.controls.addEventListener('lock', () => {
            this.isFirstPerson = true;
        });
        this.controls.addEventListener('unlock', () => {
            this.isFirstPerson = false;
        });
    }

    // --- NEW: Method to handle taking damage ---
    takeDamage(amount) {
        if (this.isDead) return; // Already dead, do nothing

        this.health -= amount;
        console.log(`Player took ${amount} damage. Health: ${this.health}/${this.maxHealth}`);

        // TODO: Add visual/audio feedback (e.g., screen flash, sound)

        if (this.health <= 0) {
            this.health = 0;
            this.isDead = true;
            console.log("Player has died!");
            // Death handling (world switch) is managed in main.js
        }
    }

    // --- NEW: Method to reset player state after respawn ---
    resetState(spawnPosition) {
        console.log("Resetting player state...");
        this.health = this.maxHealth;
        this.isDead = false;

        // Reset physics state
        this.position.copy(spawnPosition);
        this.visualPosition.copy(spawnPosition);
        this.velocity.set(0, 0, 0);
        this.onGround = false; // Let physics re-evaluate ground state

        // Reset timers
        this.coyoteTimer = 0;
        this.jumpBufferTimer = 0;
        this.laserCooldownTimer = 0;
        this.canShoot = true;

        // Reset movement flags
        this.moveForward = false;
        this.moveBackward = false;
        this.moveLeft = false;
        this.moveRight = false;
        this.moveUp = false;
        this.moveDown = false;

        // Reset view state (optional, might want to keep front view if toggled)
        // this.isFrontViewRequested = false;
        // this.isFirstPerson = false; // Will be set by PointerLockControls

        // Reset weapon state (optional)
        // this.human.unequipSaber();
        // this.human.unequipLaserGun();

        // Update human model position
        if (this.human) {
            this.human.updatePosition(this.visualPosition);
        }

        // Ensure controls are unlocked initially after respawn
        if (this.controls.isLocked) {
            this.controls.unlock();
        }
    }


    onMouseDown(event) {
        // Check for left mouse button (button === 0)
        if (event.button === 0 && this.human.isLaserGunEquipped && this.controls.isLocked && this.canShoot) {
            this.shootLaser();
            this.canShoot = false; // Prevent immediate re-firing
            this.laserCooldownTimer = this.laserCooldownDuration; // Start cooldown
        }
    }

    shootLaser() {
        // Play laser shooting sound
        this.audioManager.playSound('laser');

        // Laser origin from barrel tip
        const laserOrigin = new THREE.Vector3();
        // Ensure the barrel tip's world matrix is up-to-date
        this.human.laserGun.barrelTip.updateWorldMatrix(true, false);
        this.human.laserGun.barrelTip.getWorldPosition(laserOrigin);

        // Laser direction towards crosshair (center of view)
        const laserDirection = new THREE.Vector3();
        this.camera.getWorldDirection(laserDirection);

        // Set up raycaster
        const raycaster = new THREE.Raycaster(laserOrigin, laserDirection);
        // Ensure creatures/ghasts meshes are added to the scene to be intersected
        const objectsToIntersect = this.scene.children.filter(child => child !== this.human.mesh); // Exclude player model
        const intersects = raycaster.intersectObjects(objectsToIntersect, true); // Check recursively

        let laserEndPoint;
        let hitTarget = null; // Will hold the Martian or Ghast object if hit
        let targetType = null; // 'martian' or 'ghast'

        if (intersects.length > 0) {
            const hitObject = intersects[0].object;
            // console.log('Raycast hit an object:', hitObject.name, hitObject); // Debugging

            // --- START: Modified Hit Detection Logic ---
            let current = hitObject;
            while (current && !hitTarget) { // Loop until a target is found or we reach the scene root

                // Check Martians (using physics.creatures)
                // Ensure physics and creatures array exist and are iterable
                if (this.physics && Array.isArray(this.physics.creatures)) {
                    const potentialMartian = this.physics.creatures.find(creature => creature && creature.mesh === current);
                    if (potentialMartian) {
                        if (!potentialMartian.isDead) { // Check if already dead
                            hitTarget = potentialMartian;
                            targetType = 'martian';
                            // console.log("MATCHED LIVE MARTIAN:", hitTarget); // Debug
                            break; // Found a live Martian, stop searching
                        } else {
                            // console.log("Matched dead martian, ignoring."); // Debug
                            // Don't break, continue checking parents in case a live one is behind
                        }
                    }
                }

                // Check Ghasts (using this.activeGhasts)
                // Ensure activeGhasts array exists and is iterable
                if (!hitTarget && Array.isArray(this.activeGhasts)) {
                     const potentialGhast = this.activeGhasts.find(ghast => ghast && ghast.mesh === current);
                     if (potentialGhast) {
                         if (!potentialGhast.isDead) { // Check if already dead
                            hitTarget = potentialGhast;
                            targetType = 'ghast';
                            // console.log("MATCHED LIVE GHAST:", hitTarget); // Debug
                            break; // Found a live Ghast, stop searching
                         } else {
                            // console.log("Matched dead ghast, ignoring."); // Debug
                            // Don't break, continue checking parents
                         }
                     }
                }

                current = current.parent; // Move up the hierarchy
            }
            // --- END: Modified Hit Detection Logic ---


            if (hitTarget && targetType) { // Ensure we found a valid, live target
                console.log(`Hit a live ${targetType}!`);
                hitTarget.takeDamage(this.laserDamageAmount);
                // Play impact sound (maybe use different sounds later?)
                this.audioManager.playSound('impact_martian'); // Reusing Martian sound for now
            } else {
                // console.log('Hit geometry or an unknown/dead object.');
            }

            laserEndPoint = intersects[0].point; // Endpoint is the hit point

        } else {
            // console.log('No hits detected.'); // Keep for debug
            const maxRange = 100; // Max laser range if nothing is hit
            laserEndPoint = laserOrigin.clone().add(laserDirection.clone().multiplyScalar(maxRange));
        }

        // --- Laser Beam Visualization (No changes needed here) ---
        const length = laserOrigin.distanceTo(laserEndPoint);
        if (length < 0.01) return; // Avoid creating zero-length beams if hit point is too close

        const innerRadius = 0.03;
        const outerRadius = 0.1;
        const innerColor = 0xff00ff; // Magenta core
        const outerColor = 0x880088; // Darker magenta glow
        const segments = 8; // Fewer segments for performance

        // Inner laser beam
        const innerGeometry = new THREE.CylinderGeometry(innerRadius, innerRadius, length, segments);
        const innerMaterial = new THREE.MeshBasicMaterial({
            color: innerColor,
            blending: THREE.AdditiveBlending,
            transparent: true,
            opacity: 0.8, // Slightly less intense core
            depthWrite: false
        });
        const innerLaserBeamMesh = new THREE.Mesh(innerGeometry, innerMaterial);

        // Outer laser beam (glow)
        const outerGeometry = new THREE.CylinderGeometry(outerRadius, outerRadius, length, segments);
        const outerMaterial = new THREE.MeshBasicMaterial({
            color: outerColor,
            blending: THREE.AdditiveBlending,
            transparent: true,
            opacity: 0.4, // Fainter glow
            depthWrite: false
        });
        const outerLaserBeamMesh = new THREE.Mesh(outerGeometry, outerMaterial);

        // Orient and position the beam
        const direction = new THREE.Vector3().subVectors(laserEndPoint, laserOrigin); // Get direction vector first
        direction.normalize(); // Normalize it
        const quat = new THREE.Quaternion().setFromUnitVectors(new THREE.Vector3(0, 1, 0), direction); // Align Y axis with direction
        const midPoint = new THREE.Vector3().lerpVectors(laserOrigin, laserEndPoint, 0.5); // Position at midpoint

        innerLaserBeamMesh.quaternion.copy(quat);
        innerLaserBeamMesh.position.copy(midPoint);
        outerLaserBeamMesh.quaternion.copy(quat);
        outerLaserBeamMesh.position.copy(midPoint);

        // Add to scene and manage lifecycle
        this.scene.add(innerLaserBeamMesh);
        this.scene.add(outerLaserBeamMesh);
        const currentTime = performance.now();
        const beamDuration = 80; // Make beam visible slightly longer (milliseconds)
        this.activeLaserBeams.push({ mesh: innerLaserBeamMesh, despawnTime: currentTime + beamDuration });
        this.activeLaserBeams.push({ mesh: outerLaserBeamMesh, despawnTime: currentTime + beamDuration });
        // --- End Laser Beam Visualization ---
    }

    onKeyDown(event) {
        switch (event.code) {
            case 'Tab':
                event.preventDefault();
                this.isFrontViewRequested = !this.isFrontViewRequested;
                if (this.isFrontViewRequested) {
                    this.previousCameraQuaternion.copy(this.camera.quaternion);
                    this.wasFirstPersonBeforeFrontView = this.isFirstPerson;
                    this.isFirstPerson = false;
                    this.controls.unlock();
                } else {
                    this.camera.quaternion.copy(this.previousCameraQuaternion);
                    this.isFirstPerson = this.wasFirstPersonBeforeFrontView;
                    if (this.isFirstPerson) {
                        this.controls.lock();
                    }
                }
                break;
            // --- Movement keys ---
            // Use KeyW for forward, KeyS for backward (more standard)
            case 'KeyW': this.moveForward = true; break;
            case 'KeyS': this.moveBackward = true; break;
            case 'KeyA': this.moveLeft = true; break;
            case 'KeyD': this.moveRight = true; break;
            // --- End Movement Keys ---

            case 'KeyE': this.moveUp = true; break; // Keep for no-gravity mode
            case 'KeyQ': this.moveDown = true; break; // Keep for no-gravity mode

            case 'Space':
                // Use jump buffer for more responsive jumps
                this.jumpBufferTimer = this.jumpBufferDuration;
                // Actual jump logic is now handled in physics.update based on buffer and onGround/coyote time
                break;
            case 'KeyV':
                this.gravityEnabled = !this.gravityEnabled;
                console.log("Gravity Enabled:", this.gravityEnabled);
                if (!this.gravityEnabled) {
                    this.velocity.y = 0; // Stop vertical movement if gravity turned off
                }
                break;
            case 'Digit1':
                if (!this.isOneTogglePressed) {
                    this.isOneTogglePressed = true;
                    if (this.human.isSaberEquipped) {
                        this.human.unequipSaber();
                    } else {
                        this.human.equipSaber(); // This automatically unequips the laser gun
                    }
                    // Lock controls only if entering first-person weapon view
                    if ((this.human.isSaberEquipped || this.human.isLaserGunEquipped) && !this.isFrontViewRequested) {
                         if (!this.controls.isLocked) this.controls.lock();
                    }
                }
                break;
            case 'Digit2':
                if (!this.isTwoTogglePressed) {
                    this.isTwoTogglePressed = true;
                    if (this.human.isLaserGunEquipped) {
                        this.human.unequipLaserGun();
                    } else {
                        this.human.equipLaserGun(); // This automatically unequips the saber
                    }
                     // Lock controls only if entering first-person weapon view
                    if ((this.human.isSaberEquipped || this.human.isLaserGunEquipped) && !this.isFrontViewRequested) {
                         if (!this.controls.isLocked) this.controls.lock();
                    }
                }
                break;
        }
    }

    onKeyUp(event) {
        switch (event.code) {
             // --- Movement keys ---
            case 'KeyW': this.moveForward = false; break;
            case 'KeyS': this.moveBackward = false; break;
            case 'KeyA': this.moveLeft = false; break;
            case 'KeyD': this.moveRight = false; break;
             // --- End Movement Keys ---

            case 'KeyE': this.moveUp = false; break;
            case 'KeyQ': this.moveDown = false; break;

            case 'Digit1': this.isOneTogglePressed = false; break;
            case 'Digit2': this.isTwoTogglePressed = false; break;
        }
    }

    applyInputs(dt) {
        const speed = 5.0; // Player speed
        const direction = new THREE.Vector3(); // Create direction vector inside

        // Calculate movement direction based on camera view (if locked) or model orientation (if front view)
        if (this.controls.isLocked && !this.isFrontViewRequested) {
            direction.set(0, 0, 0);
            if (this.moveForward) direction.z = -1;
            if (this.moveBackward) direction.z = 1;
            if (this.moveLeft) direction.x = -1;
            if (this.moveRight) direction.x = 1;

            direction.normalize(); // Ensure consistent speed diagonally
            direction.applyQuaternion(this.camera.quaternion); // Apply camera rotation

        } else if (this.isFrontViewRequested) {
             direction.set(0, 0, 0);
             if (this.moveForward) direction.z = -1; // Moving "forward" relative to model
             if (this.moveBackward) direction.z = 1; // Moving "backward" relative to model
             if (this.moveLeft) direction.x = -1;
             if (this.moveRight) direction.x = 1;

             direction.normalize();
             direction.applyQuaternion(this.human.mesh.quaternion); // Apply model rotation
        } else {
            // Controls unlocked, not front view - no horizontal movement from keys
            direction.set(0, 0, 0);
        }

        // Apply calculated velocity
        this.velocity.x = direction.x * speed;
        this.velocity.z = direction.z * speed;

        // Apply vertical velocity for no-gravity mode
        if (!this.gravityEnabled) {
            if (this.moveUp) this.velocity.y = speed;
            else if (this.moveDown) this.velocity.y = -speed;
            else this.velocity.y = 0;
        }

        // Update position based on velocity (gravity is applied separately in physics.update)
        // Note: We only add the X and Z components here. Y is handled by gravity + jumps.
        this.position.x += this.velocity.x * dt;
        this.position.z += this.velocity.z * dt;
        // Y position is updated by adding velocity.y * dt AFTER gravity has been applied
        this.position.y += this.velocity.y * dt;


        // Update human model rotation (only when controls are locked/first person)
        if (this.controls.isLocked && !this.isFrontViewRequested) {
            const cameraEuler = new THREE.Euler().setFromQuaternion(this.camera.quaternion, 'YXZ');
            const yaw = cameraEuler.y;
            const pitch = cameraEuler.x;
            // Apply pitch adjustment only when weapon equipped and in first person
            const applyPitchToArm = (this.human.isSaberEquipped || this.human.isLaserGunEquipped);
            this.human.updateRotation(yaw, pitch, applyPitchToArm);
        } else if (this.isFrontViewRequested) {
            // In front view, model rotation might be controlled differently if needed
            // For now, let its rotation be set by the direction logic above
             const targetRotationY = Math.atan2(this.velocity.x, this.velocity.z);
             // Smoothly rotate model? Or snap? Snap for now.
             this.human.mesh.rotation.y = targetRotationY + Math.PI; // Add PI if model faces -Z
             this.human.updateRotation(this.human.mesh.rotation.y - Math.PI, 0, false); // Update internal rotation state without pitch
        }


        // Update laser cooldown timer
        if (this.laserCooldownTimer > 0) {
            this.laserCooldownTimer -= dt;
            if (this.laserCooldownTimer <= 0) {
                this.canShoot = true;
                this.laserCooldownTimer = 0; // Ensure it doesn't go negative
            }
        }
    }
}
```

---

**3. Modified File: `main.js`**

```javascript
// src/main.js
// Major adjustments for game states and world switching
// FIX: Corrected click listener to allow pointer lock in Mars Base
// ADDED: Portal Menu integration
// ADDED: Martian Hunt countdown timer
// UPDATED: Player spawn logic for both worlds
// FIX: Prevent Portal Menu from immediately reopening after closing
// ADDED: Player pushback when closing portal menu
// ADDED: Ghast creature integration with territorial movement
// FIX: Removed Ghast spawn height clamping based on world.params.height
// MODIFIED: Implemented Martian mesh removal on death (Option A)
// ADDED: Passing activeGhasts list to Player
// ADDED: Fireball management and player death handling

import * as THREE from 'three';
import { MathUtils } from 'three/src/math/MathUtils.js';
import { MarsBaseWorld } from './MarsBaseWorld.js';
import { MartianHuntWorld } from './MartianHuntWorld.js';
import Stats from 'three/examples/jsm/libs/stats.module.js';
import { LightingSystem } from './lighting.js';
import { Player } from './player.js';
import { Physics } from './physics.js';
import { Martian } from './martian.js';
import { Ghast } from './Ghast.js';
import { Fireball } from './Fireball.js'; // <<<--- IMPORT Fireball
import { WorldBoundary } from './world_boundary.js';
import { Menu } from './menu.js';
import { PortalMenu } from './PortalMenu.js';
import { POV } from './camera.js';
import { AudioManager } from './AudioManager.js';

// --- Global Variables ---
let scene, camera, renderer, stats, clock;
let lightingSystem, physics, player, pov, menu, crosshair, audioManager;
let portalMenu;
let timerDisplayElement, timerValueElement;
let world = null;
let worldBoundary = null;
let gameState = 'MARS_BASE';
let activeLaserBeams = [];
let activeGhasts = []; // Initialize Ghast list here
let activeFireballs = []; // <<<--- NEW: Array for active fireballs
let ghastTerritories = [];
let isTransitioning = false;
let huntTimerRemaining = 0;
const huntDuration = 300.0; // 5 minutes (300 seconds)
let justClosedPortalMenu = false;

// --- Initialization ---
async function init() {
    // Basic Three.js Setup
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    document.body.appendChild(renderer.domElement);

    scene = new THREE.Scene();
    camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    clock = new THREE.Clock();

    stats = new Stats();
    document.body.appendChild(stats.dom);

    // Core Systems
    lightingSystem = new LightingSystem(scene);
    audioManager = new AudioManager();
    audioManager.loadSound('laser', 'public/sfx/laser.MP3'); // Ensure paths are correct
    audioManager.loadSound('impact_martian', 'public/sfx/impact_martian.MP3'); // Ensure paths are correct
    // TODO: Add fireball shoot/impact sounds later
    // audioManager.loadSound('fireball_shoot', 'public/sfx/fireball_shoot.wav');
    // audioManager.loadSound('fireball_impact', 'public/sfx/fireball_impact.wav');

    // Load Initial World (Mars Base)
    physics = new Physics(scene, [], activeFireballs); // <<< Pass activeFireballs to Physics
    await loadMarsBase(); // This will set world

    // Initialize Player - PASS activeGhasts (initially empty)
    player = new Player(camera, renderer.domElement, scene, activeLaserBeams, physics, world, audioManager, activeGhasts);
    setMarsBasePlayerPosition(); // Set initial position after player exists

    // POV Setup
    crosshair = document.getElementById('crosshair');
    pov = new POV(camera, player, crosshair);

    // Menus
    menu = new Menu();
    portalMenu = new PortalMenu(switchToMartianHunt, closePortalMenu);

    // Timer Display
    timerDisplayElement = document.getElementById('timer-display');
    timerValueElement = document.getElementById('timer-value');
    if (!timerDisplayElement || !timerValueElement) {
        console.error("Timer display elements not found in DOM!");
    } else {
        timerDisplayElement.classList.add('hidden');
    }

    // Event Listeners
    setupEventListeners();

    // Start Animation Loop
    renderer.setAnimationLoop(animate);

    console.log("Initialization Complete. Game State:", gameState);
}

// --- World Loading Functions ---

function cleanupCurrentWorld() {
    console.log("Cleaning up current world...");
    if (world && typeof world.disposeMeshes === 'function') {
        world.disposeMeshes();
    }
    // --- Cleanup Fireballs --- <<< NEW
    if (activeFireballs.length > 0) {
        console.log(`Removing ${activeFireballs.length} Fireball meshes from scene.`);
        activeFireballs.forEach(fireball => {
            if (fireball && typeof fireball.destroy === 'function') {
                fireball.destroy(); // Use the fireball's own cleanup method
            }
        });
        activeFireballs = []; // Clear the array
    }
    // --- End Fireball Cleanup ---

    if (physics) {
        // Cleanup Martians
        if (physics.creatures && physics.creatures.length > 0) {
            console.log(`Removing ${physics.creatures.length} Martian creature meshes from scene.`);
            physics.creatures.forEach(creature => {
                if (creature && creature.mesh && creature.mesh.parent) {
                    scene.remove(creature.mesh);
                    // Dispose geometry/materials if unique per instance
                    // Assuming Martian materials/geo are unique:
                     if (creature.mesh.geometry) creature.mesh.geometry.dispose();
                     if (creature.mesh.material) {
                        if (Array.isArray(creature.mesh.material)) {
                             creature.mesh.material.forEach(m => m.dispose());
                        } else {
                             creature.mesh.material.dispose();
                        }
                     }
                }
            });
        }
        physics.creatures = []; // Clear the list
        physics.barriers = []; // Clear barriers
    }

    // Cleanup Ghasts
    if (activeGhasts.length > 0) {
        console.log(`Removing ${activeGhasts.length} Ghast meshes from scene.`);
        activeGhasts.forEach(ghast => {
            // Ghast's die() method should handle removal and disposal,
            // but we ensure removal here in case die() wasn't called or finished.
            if (ghast && ghast.mesh && ghast.mesh.parent) {
                scene.remove(ghast.mesh);
                // Trigger disposal if not already dead (or handle disposal here)
                if (!ghast.isDead) {
                    // Optional: Force disposal if needed, though die() should handle it
                    // if (ghast.particleGeometry) ghast.particleGeometry.dispose();
                    // ... dispose other ghast resources ...
                }
            }
        });
        activeGhasts = []; // Clear the list
    }
    ghastTerritories = []; // Clear territories

    if (timerDisplayElement) {
        timerDisplayElement.classList.add('hidden');
    }
    world = null;
    worldBoundary = null;
    console.log("World cleanup complete.");
}

async function loadMarsBase() {
    cleanupCurrentWorld(); // Clears physics.creatures, activeGhasts, and activeFireballs
    gameState = 'MARS_BASE';
    console.log("Loading Mars Base...");

    world = new MarsBaseWorld(scene);
    await world.generate();

    worldBoundary = new WorldBoundary(world.params);
    if (physics) {
        physics.barriers = worldBoundary.barriers;
        physics.creatures = []; // Ensure creatures list is empty for base
        physics.activeFireballs = activeFireballs; // Update physics reference (will be empty)
    } else {
        console.warn("Physics system not available when setting Mars Base barriers.");
    }

    // Update player's world and ghast list reference
    if (player) {
        const spawnPos = getMarsBasePlayerSpawnPosition(); // Get spawn position
        player.world = world; // Update world reference *before* resetting state
        player.activeGhasts = activeGhasts; // Update reference (will be empty)
        player.resetState(spawnPos); // Reset player state with correct spawn pos
    }

    // Unlock controls if they were locked
    if (player && player.controls && player.controls.isLocked) {
        player.controls.unlock();
    }
    isTransitioning = false;
    console.log("Mars Base Loaded.");
}

// --- NEW Helper function to calculate spawn position ---
function getMarsBasePlayerSpawnPosition() {
    if (!world || world.minFlatX === undefined || world.worldCenterOffset === undefined) {
        console.error("Cannot get Mars Base player spawn position: World data missing.");
        return new THREE.Vector3(0, 5, 0); // Fallback position
    }
    const centerOffset = world.worldCenterOffset;
    const spawnOffset = 5; // Distance outside flat area
    const flatCenterZGrid = Math.floor((world.minFlatZ + world.maxFlatZ) / 2);
    const spawnZ = flatCenterZGrid - centerOffset; // World Z coordinate
    const targetGridX = world.minFlatX - spawnOffset; // Grid X coordinate
    const spawnX = targetGridX - centerOffset; // World X coordinate

    // Find ground height at spawn location
    let groundY = 1; // Default ground level in base
    let foundGround = false;
    if (world && world.params && world.params.height) {
        for (let y = world.params.height - 1; y >= 0; y--) {
            const block = world.getBlock(targetGridX, y, flatCenterZGrid); // Use grid coords for getBlock
            if (block && block.id !== 'air') {
                groundY = y + 1.0; // Position above the ground block
                foundGround = true;
                break;
            }
        }
    }
    if (!foundGround) console.warn(`Could not find ground at Mars Base spawn (${targetGridX}, ${flatCenterZGrid}). Using default Y=1.`);

    const spawnY = groundY + 0.1; // Start slightly above ground
    return new THREE.Vector3(spawnX, spawnY, spawnZ);
}

// --- Modified: No longer sets player position directly ---
function setMarsBasePlayerPosition() {
    if (!player) return;
    const spawnPos = getMarsBasePlayerSpawnPosition();
    player.position.copy(spawnPos);
    player.velocity.set(0, 0, 0);
    player.visualPosition.copy(player.position);
    player.onGround = false; // Assume not on ground initially, physics will check
    if (player.human) player.human.updatePosition(player.visualPosition);
    console.log(`Initial player position set OUTSIDE Mars Base flat area at world coords: (${spawnPos.x.toFixed(2)}, ${spawnPos.y.toFixed(2)}, ${spawnPos.z.toFixed(2)})`);
}


async function switchToMartianHunt() {
    if (isTransitioning) return;
    isTransitioning = true;
    console.log("Transitioning to Martian Hunt...");

    cleanupCurrentWorld(); // Clears physics.creatures, activeGhasts, and activeFireballs
    gameState = 'MARTIAN_HUNT';

    world = new MartianHuntWorld(scene);
    await world.generate();

    worldBoundary = new WorldBoundary(world.params);
    if (physics) {
        physics.barriers = worldBoundary.barriers;
        physics.creatures = []; // Ensure creatures list starts empty
        physics.activeFireballs = activeFireballs; // Update physics reference (will be empty)
    } else {
        console.warn("Physics system not available when setting Martian Hunt barriers.");
    }

    // --- Player Position for Hunt ---
    const targetWorldX = 0; // Center of the hunt world
    const targetWorldZ = 0;
    let groundY = 5; // Default height if ground check fails
    if (world && world.worldCenterOffset !== undefined && world.params && world.params.height) {
        const centerOffset = world.worldCenterOffset;
        const gridX = Math.floor(targetWorldX + centerOffset);
        const gridZ = Math.floor(targetWorldZ + centerOffset);
        let foundGround = false;
        // Scan down from a reasonable height
        for (let y = Math.min(world.params.height - 1, 15); y >= 0; y--) {
            const block = world.getBlock(gridX, y, gridZ);
            if (block && block.id !== 'air') {
                groundY = y + 1.0; // Position player feet slightly above ground block
                foundGround = true;
                break;
            }
        }
        if (!foundGround) console.warn(`Could not find ground at player spawn (${gridX}, ${gridZ}). Using default Y.`);
    } else {
        console.error("Cannot calculate ground height for Hunt spawn: World data missing.");
    }
    const spawnClearance = 0.1; // Start just above ground
    const huntSpawnY = groundY + spawnClearance;
    const huntSpawnPos = new THREE.Vector3(targetWorldX, huntSpawnY, targetWorldZ);

    // Update Player State (using resetState for consistency)
    player.world = world; // Update world reference *before* resetting state
    player.resetState(huntSpawnPos); // Reset health, set position/velocity for hunt world
    // activeGhasts is currently empty, will be updated after spawnGhasts

    console.log(`Player position set for Martian Hunt at world coords: (${huntSpawnPos.x}, ${huntSpawnPos.y.toFixed(2)}, ${huntSpawnPos.z})`);

    // Spawn Creatures AFTER setting player position
    spawnMartians(); // Populates physics.creatures
    spawnGhasts();   // Populates global activeGhasts

    // --- IMPORTANT: Update player's ghast list reference AFTER spawning ---
    player.activeGhasts = activeGhasts;

    // Start Timer
    huntTimerRemaining = huntDuration;
    if (timerDisplayElement && timerValueElement) {
        timerValueElement.textContent = huntTimerRemaining.toFixed(1);
        timerDisplayElement.classList.remove('hidden');
    }

    // Attempt to lock controls
    if (player && player.controls && !menu.isOpen() && !portalMenu.isOpen() && !player.isFrontViewRequested) {
        console.log("Attempting to lock controls after Martian Hunt load.");
        // Delay lock slightly to ensure DOM/browser is ready
        setTimeout(() => {
             if (!menu.isOpen() && !portalMenu.isOpen() && !player.isFrontViewRequested) {
                player.controls.lock();
             }
        }, 100); // 100ms delay
    } else {
        console.log("Conditions not met to lock controls after Martian Hunt load.");
    }

    isTransitioning = false;
    console.log("Martian Hunt Loaded.");
}


function closePortalMenu() {
    console.log("Portal menu closed, returning to base.");
    portalMenu.hide();
    justClosedPortalMenu = true; // Flag to prevent immediate reopening

    // Push player back logic (ensure world and portalCenterWorld exist)
    if (player && world && world.portalCenterWorld) {
        try {
            const portalCenter = world.portalCenterWorld;
            const playerPos = player.position;
            const pushbackDistance = 4.0; // How far to push back

            // Calculate direction away from portal center (horizontal only)
            const direction = new THREE.Vector3().subVectors(playerPos, portalCenter);
            direction.y = 0; // Ignore vertical difference
            if (direction.lengthSq() > 0.001) { // Avoid normalizing zero vector
                direction.normalize();
            } else {
                direction.set(1, 0, 0); // Default pushback direction if player is exactly at center
            }

            // Calculate new position
            const pushbackVector = direction.multiplyScalar(pushbackDistance);
            const newPosition = portalCenter.clone().add(pushbackVector);
            newPosition.y = playerPos.y; // Keep player's current height

            // Apply new position and reset velocity
            player.position.copy(newPosition);
            player.visualPosition.copy(newPosition); // Sync visual immediately
            player.velocity.set(0, 0, 0); // Stop movement
            if (player.human && typeof player.human.updatePosition === 'function') {
                player.human.updatePosition(player.visualPosition);
            }
            console.log(`Player pushed back from portal to: ${newPosition.x.toFixed(2)}, ${newPosition.y.toFixed(2)}, ${newPosition.z.toFixed(2)}`);

            // Attempt to re-lock controls if appropriate
            if (player.controls && !menu.isOpen() && !player.isFrontViewRequested) {
                 setTimeout(() => { // Delay lock slightly
                      if (!menu.isOpen() && !portalMenu.isOpen() && !player.isFrontViewRequested) {
                         player.controls.lock();
                      }
                 }, 100);
            }
        } catch (error) {
            console.error("Error during player pushback:", error);
            // Fallback: just try to lock controls if possible
            if (player && player.controls && !menu.isOpen() && !player.isFrontViewRequested) {
                 setTimeout(() => {
                      if (!menu.isOpen() && !portalMenu.isOpen() && !player.isFrontViewRequested) {
                         player.controls.lock();
                      }
                 }, 100);
            }
        }
    } else {
        console.warn("Cannot push player back: Player or World/Portal data missing.");
        // Still try to lock controls if appropriate
        if (player && player.controls && !menu.isOpen() && !player.isFrontViewRequested) {
             setTimeout(() => {
                  if (!menu.isOpen() && !portalMenu.isOpen() && !player.isFrontViewRequested) {
                     player.controls.lock();
                  }
             }, 100);
        }
    }
}

// --- Creature Spawning ---

function spawnMartians() {
    if (gameState !== 'MARTIAN_HUNT' || !world || !physics || !world.params) {
        console.warn("Cannot spawn Martians: Invalid state, world, physics or params.");
        return;
    }
    console.log("Spawning Martians...");
    physics.creatures = []; // Clear previous martians
    const numMartians = 20;
    const worldWidth = world.params.width;
    const centerOffset = world.worldCenterOffset;
    const spawnMargin = 10; // Keep away from edges

    for (let i = 0; i < numMartians; i++) {
        // Calculate random world coordinates within margins
        const randomX = MathUtils.randFloat(
            -centerOffset + spawnMargin,
            worldWidth - 1 - centerOffset - spawnMargin
        );
        const randomZ = MathUtils.randFloat(
            -centerOffset + spawnMargin,
            worldWidth - 1 - centerOffset - spawnMargin
        );

        // Convert world coords to grid coords for ground check
        const gridX = Math.floor(randomX + centerOffset);
        const gridZ = Math.floor(randomZ + centerOffset);

        // Find ground height at spawn location
        let groundY = 5; // Default height
        let foundGround = false;
        for (let y = world.params.height - 1; y >= 0; y--) {
            const block = world.getBlock(gridX, y, gridZ);
            if (block && block.id !== 'air') {
                groundY = y + 1.0; // Position above ground block
                foundGround = true;
                break;
            }
        }
        // if (!foundGround) console.warn(`Martian spawn ${i} couldn't find ground at ${gridX}, ${gridZ}. Using default Y.`);

        const startY = groundY + 0.5; // Start slightly above ground
        const startPosition = new THREE.Vector3(randomX, startY, randomZ);

        try {
            const martianInstance = new Martian(scene, startPosition, world);
            physics.addCreature(martianInstance); // Add to physics system
        } catch (error) {
            console.error("Error creating Martian:", error);
        }
    }
    console.log(`Spawned ${physics.creatures.length} Martians.`);
}

function spawnGhasts() {
    if (gameState !== 'MARTIAN_HUNT' || !world || !scene || world.worldCenterOffset === undefined || !world.params) {
        console.warn("Cannot spawn Ghasts: Invalid state or missing world/scene data.");
        return;
    }
    console.log("Spawning Ghasts and defining territories...");
    activeGhasts = []; // Clear previous ghasts
    ghastTerritories = [];
    const numGhasts = 3;
    const worldWidth = world.params.width;
    const centerOffset = world.worldCenterOffset;
    const worldMinCoord = -centerOffset; // e.g., -44.5 for width 90
    const worldMaxCoord = worldWidth - 1 - centerOffset; // e.g., 89 - 44.5 = 44.5
    const spawnMargin = 5; // Keep away from territory edges
    const territoryMargin = 0.5; // Gap between territories

    const totalPlayableWidth = worldMaxCoord - worldMinCoord;
    if (totalPlayableWidth <= 0 || numGhasts <= 0) {
        console.error("Cannot divide world for Ghast territories: Invalid dimensions or count.");
        return;
    }
    // Calculate width per territory, accounting for margins between them
    const territoryWidth = Math.max(1, (totalPlayableWidth - (numGhasts - 1) * territoryMargin) / numGhasts);

    // Define territories
    for (let i = 0; i < numGhasts; i++) {
        const minX = worldMinCoord + i * (territoryWidth + territoryMargin);
        const maxX = minX + territoryWidth;
        // Territories span the full Z depth
        const territory = { minX, maxX, minZ: worldMinCoord, maxZ: worldMaxCoord };
        ghastTerritories.push(territory);
        console.log(`Territory ${i}: X[${minX.toFixed(1)}, ${maxX.toFixed(1)}], Z[${worldMinCoord.toFixed(1)}, ${worldMaxCoord.toFixed(1)}]`);
    }

    // Spawn Ghasts within their territories
    for (let i = 0; i < numGhasts; i++) {
        const territory = ghastTerritories[i];
        // Spawn within territory, respecting spawn margin
        const randomX = MathUtils.randFloat(territory.minX + spawnMargin, territory.maxX - spawnMargin);
        const randomZ = MathUtils.randFloat(territory.minZ + spawnMargin, territory.maxZ - spawnMargin);

        // Convert world coords to grid coords for ground check
        const gridX = Math.floor(randomX + centerOffset);
        const gridZ = Math.floor(randomZ + centerOffset);

        // Find ground height below spawn point
        let groundY = 5; // Default height
        let foundGround = false;
        // Scan down from a reasonable height (e.g., 20 blocks up)
        for (let y = Math.min(world.params.height - 1, 20); y >= 0; y--) {
            const block = world.getBlock(gridX, y, gridZ);
            if (block && block.id !== 'air') {
                groundY = y + 1.0;
                foundGround = true;
                break;
            }
        }
        // if (!foundGround) console.warn(`Ghast spawn ${i} couldn't find ground at ${gridX}, ${gridZ}. Using default Y.`);

        // Determine spawn height (within altitude limits defined in GhastMovement)
        const altitudeMin = 5; // <<<--- INCREASED MIN ALTITUDE
        const altitudeMax = 20;
        const preferredMidY = groundY + (8 + 15) / 2; // Aim for middle of preferred range (higher)
        const startY = MathUtils.clamp(preferredMidY, groundY + altitudeMin, groundY + altitudeMax);

        const startPosition = new THREE.Vector3(randomX, startY, randomZ);

        try {
            // Create Ghast instance, passing its territory
            const ghastInstance = new Ghast(scene, startPosition, world, territory);
            activeGhasts.push(ghastInstance); // Add to the global list
            console.log(`Spawned Ghast ${i} in territory ${i} at ${startPosition.x.toFixed(1)}, ${startPosition.y.toFixed(1)}, ${startPosition.z.toFixed(1)}`);
        } catch (error) {
            console.error("Error creating Ghast:", error);
        }
    }
    console.log(`Spawned ${activeGhasts.length} Ghasts.`);
}


// --- Event Listeners Setup ---
function setupEventListeners() {
    window.addEventListener('resize', () => {
        if (camera && renderer) {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }
    });

    // Use keydown for continuous actions (like movement) and single presses
    window.addEventListener('keydown', (event) => {
        // Let player class handle its own key inputs via its event listeners
        // player.onKeyDown(event); // Player should handle its own listeners

        // Global keys (like pause menu)
        if (event.key === 'm' || event.key === 'M') {
            // Only allow toggling pause menu if portal menu isn't open
            if (portalMenu && !portalMenu.isOpen()) {
                menu.toggle();
                if (menu.isOpen()) {
                    if (player && player.controls && player.controls.isLocked) {
                        player.controls.unlock(); // Release pointer lock when menu opens
                    }
                } else {
                    // Attempt to re-lock pointer ONLY if appropriate
                    // (not in front view, not transitioning, etc.)
                    if (player && player.controls && !player.controls.isLocked && !portalMenu.isOpen() && !player.isFrontViewRequested && !isTransitioning) {
                         setTimeout(() => { // Delay lock slightly
                             if (!menu.isOpen() && !portalMenu.isOpen() && !player.isFrontViewRequested) {
                                player.controls.lock();
                             }
                         }, 100);
                    }
                }
            }
        }
        // Test sounds (remove later)
        // else if (event.key === 't') {
        //     audioManager.playSound('laser');
        // } else if (event.key === 'y') {
        //     audioManager.playSound('impact_martian');
        // }
    });

    // Keyup is handled by player class internally

    // Mousedown is handled by player class internally

    // Click to lock pointer (and resume audio)
    document.addEventListener('click', () => {
        // Only lock if menus are closed and not in front-view mode
        if (!menu.isOpen() && !portalMenu.isOpen() && player && player.controls && !player.controls.isLocked && !player.isFrontViewRequested) {
            // Resume audio context if suspended
            if (audioManager.context.state === 'suspended') {
                audioManager.context.resume().then(() => {
                    console.log("AudioContext resumed on click.");
                    player.controls.lock(); // Lock pointer after resuming audio
                }).catch(e => {
                    console.error("Error resuming AudioContext:", e);
                    player.controls.lock(); // Still try to lock pointer even if audio fails
                });
            } else {
                player.controls.lock(); // Lock pointer directly if audio is running
            }
        }
    });

    // Handle PointerLockControls events
    if (player && player.controls) {
        player.controls.addEventListener('lock', () => {
            // Resume audio just in case it was suspended externally
            if (audioManager.context.state === 'suspended') {
                audioManager.context.resume();
            }
            console.log("PointerLock acquired.");
            menu.hide(); // Ensure menus are hidden
            portalMenu.hide();
            if (crosshair && (player.human.isSaberEquipped || player.human.isLaserGunEquipped)) {
                 crosshair.classList.remove('hidden'); // Show crosshair only if weapon equipped
            }
        });

        player.controls.addEventListener('unlock', () => {
            console.log("PointerLock lost.");
            if (crosshair) crosshair.classList.add('hidden'); // Always hide crosshair when unlocked
            // Open pause menu ONLY if the unlock wasn't intentional
            // (i.e., not because portal/pause menu was opened, not during transition, not front view)
            if (!menu.isOpen() && !portalMenu.isOpen() && !isTransitioning && !player.isFrontViewRequested) {
                console.log("PointerLock lost unexpectedly, showing pause menu.");
                menu.show();
            }
        });
    } else {
        console.warn("Player or controls not initialized when setting up event listeners.");
    }
}


// --- Animation Loop ---
async function animate() { // <<<--- Make animate async
    // Request next frame immediately
    // renderer.setAnimationLoop(animate); // This is often set once in init()

    // Basic checks for essential components before proceeding
    if (!clock || !renderer || !scene || !camera || !stats || !physics || !player || !world || !menu || !portalMenu || !timerDisplayElement || !timerValueElement) {
        console.warn("Animation loop skipped: Essential component missing.");
        return; // Don't proceed if basics aren't ready
    }

    // Prevent updates during world transitions
    if (isTransitioning) {
        stats.update(); // Still update stats
        renderer.render(scene, camera); // Still render
        return;
    }

    // Calculate delta time, clamp to prevent large jumps
    let deltaTime = clock.getDelta();
    if (deltaTime > 0.1) { // Max frame time to prevent physics issues
        // console.warn(`DeltaTime clamped from ${deltaTime.toFixed(4)} to 0.1`);
        deltaTime = 0.1;
    }

    // --- Game State Logic ---

    // Portal Interaction (Mars Base Only)
    if (gameState === 'MARS_BASE' && world instanceof MarsBaseWorld && !menu.isOpen()) {
        if (player && player.position && world.portalCenterWorld) {
            const distanceSq = player.position.distanceToSquared(world.portalCenterWorld);
            // Check if player is close enough and portal menu isn't already open or just closed
            if (distanceSq < world.portalTriggerRadiusSq && !portalMenu.isOpen() && !justClosedPortalMenu) {
                const title = "MEAT COLLECTION: HUNTING MARS ANIMAL";
                const description = "Mission objective: Gather vital food supplies. Alien wildlife detected. Proceed with caution. Are you ready to begin the hunt?";
                portalMenu.show(title, description);
                // Stop player movement when menu opens
                if (player.velocity) {
                    player.velocity.x = 0;
                    player.velocity.z = 0;
                }
            }
        }
        // Reset the 'just closed' flag after one frame
        if (justClosedPortalMenu) {
            justClosedPortalMenu = false;
        }
    }

    // --- Player Death Check --- <<< NEW
    if (player.isDead && !isTransitioning) {
        console.log("Player death detected in animate loop. Loading Mars Base...");
        isTransitioning = true; // Prevent further updates/checks during load
        await loadMarsBase(); // Wait for the base to load
        // player.resetState() is now called within loadMarsBase after world is ready
        isTransitioning = false; // Allow updates again
        // No return here, let the loop finish this frame, transition check will catch next frame
    }


    // Update systems only if menus are closed AND player is not dead
    if (!menu.isOpen() && !portalMenu.isOpen() && !player.isDead) {

        // Martian Hunt Timer Update
        if (gameState === 'MARTIAN_HUNT') {
            huntTimerRemaining -= deltaTime;
            if (huntTimerRemaining <= 0) {
                huntTimerRemaining = 0;
                console.log("Hunt timer expired! Returning to base.");
                isTransitioning = true; // Set transition flag
                await loadMarsBase(); // Wait for base to load <<< Use await
                isTransitioning = false; // Allow updates again
                // No return here, let the loop finish this frame, transition check will catch next frame
            }
            // Update timer display (check elements exist)
            if (timerValueElement) {
                timerValueElement.textContent = huntTimerRemaining.toFixed(1);
            }
        }

        // Physics Update (Handles Player and Martian movement/collisions)
        // Also handles Fireball-Player collisions now
        if (physics && world) {
            physics.update(deltaTime, player, world); // Physics now handles fireball hits
        }

        // Lighting Update
        if (lightingSystem) {
            lightingSystem.update(deltaTime);
        }

        // Player Visual Smoothing (Lerp visual position towards physics position)
        if (player && player.visualPosition && player.position && player.human) {
            const alpha = 0.2; // Smoothing factor (adjust for desired smoothness)
            player.visualPosition.lerp(player.position, alpha);
            player.human.updatePosition(player.visualPosition);
            // Player rotation is handled within player.applyInputs or POV update
        }

        // --- Martian Update & Removal Logic ---
        if (gameState === 'MARTIAN_HUNT' && physics && physics.creatures) {
            const currentCreatures = physics.creatures;
            const liveCreatures = []; // Build a new list of live creatures

            for (let i = 0; i < currentCreatures.length; i++) {
                const creature = currentCreatures[i];
                if (!creature) continue; // Safety check

                if (creature.isDead) {
                    // If dead, remove its mesh if it hasn't been already
                    if (creature.mesh && creature.mesh.parent === scene) {
                        // console.log("Removing dead Martian mesh from scene in animate loop.");
                        scene.remove(creature.mesh);
                        // Dispose geometry & materials (assuming unique per Martian)
                        if (creature.mesh.geometry) creature.mesh.geometry.dispose();
                        if (creature.mesh.material) {
                            if (Array.isArray(creature.mesh.material)) {
                                creature.mesh.material.forEach(m => m.dispose());
                            } else {
                                creature.mesh.material.dispose();
                            }
                        }
                    }
                    // Do not add to liveCreatures list
                } else {
                    // Process living creatures
                    // Smooth visual position
                    if (creature.mesh && creature.visualPosition && creature.position) {
                        const alpha = 0.15; // Smoothing factor for creatures
                        creature.visualPosition.lerp(creature.position, alpha);
                        creature.mesh.position.copy(creature.visualPosition);
                        // Rotation is handled within creature.updateLogic -> updateRotation
                    }
                    liveCreatures.push(creature); // Keep in the list
                }
            }
            // Update the physics creatures array to only contain live ones
            physics.creatures = liveCreatures;
        }
        // --- End Martian Update ---

        // --- Ghast Update & Removal Logic ---
        if (gameState === 'MARTIAN_HUNT' && activeGhasts) {
            const currentGhasts = activeGhasts;
            const liveGhasts = []; // Build a new list of live ghasts

            for (let i = 0; i < currentGhasts.length; i++) {
                const ghast = currentGhasts[i];
                if (!ghast) continue; // Safety check

                if (ghast.isDead) {
                    // Ghast's die() method handles fade out and removal.
                    // We just need to stop tracking it once it's fully gone.
                } else {
                    // Update logic for live Ghasts - PASS PLAYER REFERENCE <<< MODIFIED
                    ghast.updateLogic(deltaTime, player, activeFireballs); // Pass player and fireballs
                    // Smooth visual position
                    if (ghast.mesh && ghast.visualPosition && ghast.position) {
                        const alpha = 0.1; // Smoothing factor for Ghasts
                        // Lerp mesh position directly towards physics position
                        ghast.mesh.position.lerp(ghast.position, alpha);
                        // Ghast rotation is handled by its movement controller
                    }
                    liveGhasts.push(ghast); // Keep in the list
                }
            }
            // Update the global activeGhasts array
            activeGhasts = liveGhasts;
            // Update player's reference as well (important!)
            if (player) {
                player.activeGhasts = activeGhasts;
            }
        }
        // --- End Ghast Update ---


        // --- Fireball Update & Cleanup --- <<< NEW
        const liveFireballs = [];
        for (let i = 0; i < activeFireballs.length; i++) {
            const fireball = activeFireballs[i];
            if (!fireball) continue;

            fireball.update(deltaTime);

            if (fireball.toBeRemoved) {
                fireball.destroy(); // Remove mesh, dispose resources
            } else {
                liveFireballs.push(fireball); // Keep it
            }
        }
        activeFireballs = liveFireballs; // Update the main array
        // --- End Fireball Update ---


        // POV Update (Camera position/rotation based on player state)
        if (pov) {
            pov.update();
        }

        // Active Laser Beam Cleanup
        const currentTime = performance.now();
        for (let i = activeLaserBeams.length - 1; i >= 0; i--) {
            const beam = activeLaserBeams[i];
            if (currentTime >= beam.despawnTime) {
                if (beam.mesh) {
                    scene.remove(beam.mesh);
                    if (beam.mesh.geometry) beam.mesh.geometry.dispose();
                    if (beam.mesh.material) beam.mesh.material.dispose();
                }
                activeLaserBeams.splice(i, 1); // Remove from array
            }
        }

    } else {
        // Menus are open or player is dead - stop player physics movement
        if (player && player.velocity) {
            player.velocity.x = 0;
            player.velocity.z = 0;
            // Keep vertical velocity if gravity is on, otherwise zero it
            if (!player.gravityEnabled) {
                player.velocity.y = 0;
            }
        }
    }

    // Update performance stats
    if (stats) stats.update();

    // Render the scene
    if (renderer && scene && camera) {
        renderer.render(scene, camera);
    }
}

// --- Start the Application ---
init().catch(error => {
    console.error("Initialization failed:", error);
    // Display error message to the user
    document.body.innerHTML = `<div style="color: red; padding: 20px; font-family: sans-serif; text-align: center;"><h2>Initialization Failed</h2><p>Could not start the application. Please check the browser console (F12) for more details.</p><pre style="text-align: left; background: #333; color: #eee; padding: 10px; border-radius: 5px; max-height: 300px; overflow-y: auto; white-space: pre-wrap; word-wrap: break-word;">${error.stack || error}</pre></div>`;
});

// --- END OF FILE main.js ---
```

---

**4. Modified File: `Ghast.js`**

```javascript
// --- START OF FILE Ghast.js ---

import * as THREE from 'three';
// Import MathUtils for seeded random (simple noise approximation)
import { MathUtils } from 'three/src/math/MathUtils.js';
import { GhastMovement } from './GhastMovement.js'; // <<<--- IMPORT GhastMovement
import { Fireball } from './Fireball.js'; // <<<--- IMPORT Fireball

export class Ghast {
    // Add territoryBounds parameter to constructor
    constructor(scene, position, world, territoryBounds) { // <<<--- ADD territoryBounds
        if (!scene || !position || !world) { console.error("Ghast constructor missing required arguments:", { scene, position, world }); throw new Error("Ghast constructor requires scene, position, and world."); }
        // <<<--- ADD Check for territoryBounds
        if (!territoryBounds || territoryBounds.minX === undefined) {
             console.error("Ghast constructor missing valid territoryBounds:", territoryBounds);
             throw new Error("Ghast constructor requires valid territoryBounds object.");
        }

        this.scene = scene;
        this.world = world;
        this.position = position.clone();
        this.visualPosition = this.position.clone();
        this.health = 1000; // Ghasts are tough
        this.isDead = false;
        this.clock = new THREE.Clock();

        // --- Attack Parameters --- <<< NEW
        this.attackRange = 35.0; // How close player needs to be to trigger attack
        this.attackRangeSq = this.attackRange * this.attackRange; // Use squared distance for checks
        this.fireballCooldown = 2.5; // Seconds between shots
        this.fireballSpeed = 18.0; // Units per second
        this.fireballLifetime = 4.0; // Seconds before fireball disappears
        this.fireballDamage = 30; // Damage dealt to player
        this.attackCooldownTimer = Math.random() * this.fireballCooldown; // Start with random cooldown
        this.currentTarget = null; // Store player reference when targeted
        this.isAttackingOrCoolingDown = false; // Flag for movement controller

        // Animation state helpers
        this.twitchTimer = 0;
        this.twitchDuration = 0.08 + Math.random() * 0.1;
        this.timeUntilTwitch = 0.5 + Math.random() * 1.0;
        this.twitchOffset = new THREE.Vector3();
        this.twitchRotOffset = new THREE.Euler();

        this.pulsatingMaterials = []; // Holds { material: obj, baseValue: num, property: 'opacity'/'emissiveIntensity'}
        this.particleSources = [];
        this.particles = null;
        this.particleGeometry = null;
        this.particleLifetimes = [];
        this.particleVelocities = [];
        this.particlesToSpawn = 0;

        // Fireball system reference (visual only, actual projectiles managed elsewhere)
        this.fireballGroup = null;
        this.bodyYOffset = 0; // Will be calculated in buildCreature

        // --- Instantiate Movement Controller --- <<<--- NEW
        this.movementController = new GhastMovement(this, world, territoryBounds);

        this.mesh = new THREE.Group();
        this.mesh.name = "GhastCreature_Scary";

        // --- Collision Cylinder Properties --- // <<<--- NEW
        this.cylinderRadius = 8; // Example radius (slightly larger than half body size)
        this.cylinderHeight = 20; // Example height (encompassing body + some tentacles/horns)
        this.showCollisionCylinder = false; // Set to true for debugging visibility
        this.collisionCylinderMesh = null; // To hold the cylinder mesh reference

        this.buildCreature(); // Builds visual mesh AND collision cylinder mesh
        const desiredScale = 0.6;
        this.mesh.scale.set(desiredScale, desiredScale, desiredScale);
        this.mesh.position.copy(this.position); // Set initial mesh position
        this.scene.add(this.mesh);
        console.log("Aggressive Ghast spawned at:", this.position, "with scale:", desiredScale);
        console.log(`Ghast physics cylinder: Radius=${this.cylinderRadius}, Height=${this.cylinderHeight}`);
    }

    buildCreature() {
        // --- SCARIER Configuration ---
        const BASE_COLOR = 0x1a1a1a; const NOISE_COLORS_BODY = [0x0a0a0a, 0x111111, 0x151515]; const NOISE_LIGHT_BODY = 0x252525; const NOISE_LARGE_BODY = 0x303030; const MOUTH_BG_COLOR = 0x000000; const HORN_COLOR = 0x151515; const HORN_NOISE_VARIANTS = [0x050505, 0x0a0a0a, 0x0f0f0f]; const HORN_NOISE_LIGHT = 0x1f1f1f; const TENTACLE_COLOR_BASE = 0x222222; const TENTACLE_NOISE_VARIANTS = [0x121212, 0x181818, 0x1e1e1e]; const TENTACLE_NOISE_LIGHT = 0x333333; const TENTACLE_TIP_COLOR = 0x0a0a0a; const EMISSIVE_COLOR = 0xffffff; const SIGIL_EMISSIVE_INTENSITY_BASE = 1.5; const HORN_SIGIL_EMISSIVE_INTENSITY_BASE = 1.0; const BUMP_SCALE = 0.02; const TEXTURE_RESOLUTION = 256; const sf = TEXTURE_RESOLUTION / 16; const BODY_SIZE = 9; const TENTACLE_WIDTH = 1.8; const TENTACLE_HEIGHT = 7;
        const TENTACLE_SPACING = 2.5; const HORN_BASE_WIDTH = 2.2; const HORN_BASE_DEPTH = 2.2; const HORN_SEGMENT_HEIGHT = 0.6; const HORN_SEGMENTS = 12; const HORN_SPACING_X = 1.8; const HORN_TAPER = 0.91; const HORN_CURVE_ANGLE = Math.PI / 45; const HORN_TWIST_ANGLE = Math.PI / 96; const PARTICLE_COUNT = 250; const PARTICLE_SIZE = 0.10; const PARTICLE_LIFETIME = 1.1; const PARTICLE_INITIAL_VELOCITY = 1.4; const PARTICLE_DRAG = 0.97;
        const RED_GLOW_COLOR_HEX = 0xff0000; const FACE_CORE_BASE_COLOR_HEX = 0x330000; const HALO_SCALE = 1.35; const HALO_OPACITY = 0.55; const FACE_CORE_EMISSIVE_INTENSITY = 1.0;
        const FIREBALL_COUNT = 25; const FIREBALL_CORE_SIZE = 0.30; const FIREBALL_HALO_SIZE = 0.55; const FIREBALL_HALO_OPACITY = 0.7; const FIREBALL_COLOR = 0xff6600; /* Added back for material */ const FIREBALL_CORE_EMISSIVE_INTENSITY = 1.0; const FIREBALL_ORBIT_RADIUS_MIN = BODY_SIZE * 0.7; const FIREBALL_ORBIT_RADIUS_MAX = BODY_SIZE * 1.1; const FIREBALL_ORBIT_SPEED_MIN = 0.8; const FIREBALL_ORBIT_SPEED_MAX = 1.4; const FIREBALL_VERTICAL_SPREAD = BODY_SIZE * 0.5; const FIREBALL_NOISE_AMOUNT = 0.8; const FIREBALL_NOISE_SPEED = 0.7; const FIREBALL_SYSTEM_ROTATION_SPEED = 0.3;

        // --- Advanced Texture Generation (FIXED PARAMETERS AGAIN) ---
        function createHyperGhastTextures(
            width, height, bgColor, // <<< Correct parameters
            noiseParams = { countFine: 0, countLarge: 0, colorsFine: [], colorLightFine: 0xffffff, colorLarge: 0xffffff },
            markings = [], eyes = [], sigils = [],
            gradientStrength = 0.05
        ) {
             const canvas = document.createElement('canvas'); canvas.width = width; canvas.height = height; const ctx = canvas.getContext('2d'); ctx.imageSmoothingEnabled = false; const emissiveCanvas = document.createElement('canvas'); emissiveCanvas.width = width; emissiveCanvas.height = height; const emissiveCtx = emissiveCanvas.getContext('2d'); emissiveCtx.imageSmoothingEnabled = false; emissiveCtx.fillStyle = '#000000'; emissiveCtx.fillRect(0, 0, width, height); const grad = ctx.createLinearGradient(0, 0, 0, height); const baseColorHex = `#${bgColor.toString(16).padStart(6, '0')}`; const gradColorTop = new THREE.Color(bgColor).lerp(new THREE.Color(0xffffff), gradientStrength).getHexString(); const gradColorBottom = new THREE.Color(bgColor).lerp(new THREE.Color(0x000000), gradientStrength).getHexString(); grad.addColorStop(0, `#${gradColorTop}`); grad.addColorStop(1, `#${gradColorBottom}`); ctx.fillStyle = grad; ctx.fillRect(0, 0, width, height); if (noiseParams.countLarge > 0) { const largeDotSize = 8 * sf; ctx.fillStyle = `#${noiseParams.colorLarge.toString(16).padStart(6, '0')}1A`; for (let i = 0; i < noiseParams.countLarge; i++) { const x = Math.random() * (width - largeDotSize); const y = Math.random() * (height - largeDotSize); ctx.fillRect(x, y, largeDotSize, largeDotSize); } } if (noiseParams.countFine > 0 && noiseParams.colorsFine.length > 0) { const dotSizeW = 1 * sf; const dotSizeH = 1 * sf; const allNoiseColors = [...noiseParams.colorsFine, noiseParams.colorLightFine]; for (let i = 0; i < noiseParams.countFine * 4; i++) { const x = Math.random() * (width - dotSizeW); const y = Math.random() * (height - dotSizeH); const colorIndex = Math.floor(Math.random() * allNoiseColors.length); ctx.fillStyle = `#${allNoiseColors[colorIndex].toString(16).padStart(6, '0')}`; ctx.fillRect(x, y, dotSizeW, dotSizeH); } } const mouthBg = markings.find(m => m.isMouthBg); if (mouthBg) { ctx.fillStyle = `#${mouthBg.color.toString(16).padStart(6, '0')}`; ctx.fillRect(mouthBg.x, mouthBg.y, mouthBg.w, mouthBg.h); } markings.forEach(m => { if (!m.isMouthBg) { ctx.fillStyle = `#${m.color.toString(16).padStart(6, '0')}`; ctx.fillRect(m.x, m.y, m.w, m.h); } }); sigils.forEach(sigilPart => { ctx.fillStyle = `#${EMISSIVE_COLOR.toString(16).padStart(6, '0')}`; ctx.fillRect(sigilPart.x, sigilPart.y, sigilPart.w, sigilPart.h); emissiveCtx.fillStyle = '#ffffff'; emissiveCtx.fillRect(sigilPart.x, sigilPart.y, sigilPart.w, sigilPart.h); }); const texture = new THREE.CanvasTexture(canvas); texture.magFilter = THREE.NearestFilter; texture.minFilter = THREE.LinearMipMapLinearFilter; texture.needsUpdate = true; texture.colorSpace = THREE.SRGBColorSpace; const emissiveTexture = new THREE.CanvasTexture(emissiveCanvas); emissiveTexture.magFilter = THREE.NearestFilter; emissiveTexture.minFilter = THREE.NearestFilter; emissiveTexture.colorSpace = THREE.SRGBColorSpace; return { texture, emissiveTexture };
        }

        // --- Sigil Generation ---
        function createArcSigil(cx, cy, radius, startAngle, endAngle, segments, thickness) { /* ... (logic unchanged) ... */ const parts = []; const angleStep = (endAngle - startAngle) / segments; for (let i = 0; i <= segments; i++) { const angle = startAngle + i * angleStep; const x = cx + Math.cos(angle) * radius; const y = cy + Math.sin(angle) * radius; parts.push({ x: x - (thickness/2)*sf, y: y - (thickness/2)*sf, w: thickness*sf, h: thickness*sf }); } return parts; }
        function createLineSigil(x1, y1, x2, y2, segments, thickness) { /* ... (logic unchanged) ... */ const parts = []; const dx = (x2 - x1) / segments; const dy = (y2 - y1) / segments; for (let i = 0; i <= segments; i++) { const x = x1 + i * dx; const y = y1 + i * dy; parts.push({ x: x - (thickness/2)*sf, y: y - (thickness/2)*sf, w: thickness*sf, h: thickness*sf }); } return parts; }

        // --- Define Texture Data ---
        const bodyNoiseParams = { countFine: 800, countLarge: 70, colorsFine: NOISE_COLORS_BODY, colorLightFine: NOISE_LIGHT_BODY, colorLarge: NOISE_LARGE_BODY }; const mouthW = 7 * sf; const mouthH = 3 * sf; const mouthX = (16 * sf - mouthW) / 2; const mouthY = 10 * sf; const frontFaceMarkings = [ { x: mouthX, y: mouthY, w: mouthW, h: mouthH, color: MOUTH_BG_COLOR, isMouthBg: true }, ]; const frontSigilData = [ ...createArcSigil(3*sf, 4*sf, 2*sf, Math.PI*1.1, Math.PI*1.9, 10, 1), ...createLineSigil(13*sf, 13*sf, 15*sf, 15*sf, 5, 1)]; const sideSigilData = [ ...createLineSigil(11*sf, 10*sf, 14*sf, 4*sf, 12, 1), ...createArcSigil(4*sf, 4*sf, 1.5*sf, 0, Math.PI*0.8, 8, 1)]; const topSigilData = [ ...createArcSigil(8*sf, 8*sf, 5*sf, Math.PI*0.2, Math.PI*0.8, 15, 1), ...createLineSigil(2*sf, 2*sf, 14*sf, 14*sf, 20, 1)]; const hornNoiseParams = { countFine: 150, countLarge: 20, colorsFine: HORN_NOISE_VARIANTS, colorLightFine: HORN_NOISE_LIGHT, colorLarge: HORN_COLOR }; const hornSigilData = [ ...createArcSigil(2*sf, 8*sf, 1.5*sf, Math.PI*0.5, Math.PI*1.5, 8, 0.5) ]; const tentacleNoiseParams = { countFine: 200, countLarge: 25, colorsFine: TENTACLE_NOISE_VARIANTS, colorLightFine: TENTACLE_NOISE_LIGHT, colorLarge: TENTACLE_COLOR_BASE };

        // --- Create Textures ---
        const frontTextures = createHyperGhastTextures(TEXTURE_RESOLUTION, TEXTURE_RESOLUTION, BASE_COLOR, bodyNoiseParams, frontFaceMarkings, [], frontSigilData); const sideTextures = createHyperGhastTextures(TEXTURE_RESOLUTION, TEXTURE_RESOLUTION, BASE_COLOR, bodyNoiseParams, [], [], sideSigilData); const topTextures = createHyperGhastTextures(TEXTURE_RESOLUTION, TEXTURE_RESOLUTION, BASE_COLOR, bodyNoiseParams, [], [], topSigilData); const bottomTextures = createHyperGhastTextures(TEXTURE_RESOLUTION, TEXTURE_RESOLUTION, BASE_COLOR, { ...bodyNoiseParams, countFine: bodyNoiseParams.countFine - 100 }, [], [], [], 0.02); const backTextures = createHyperGhastTextures(TEXTURE_RESOLUTION, TEXTURE_RESOLUTION, BASE_COLOR, bodyNoiseParams, [], [], sideSigilData); const hornTextures = createHyperGhastTextures(TEXTURE_RESOLUTION, TEXTURE_RESOLUTION, HORN_COLOR, hornNoiseParams, [], [], hornSigilData, 0); const tentacleTextureData = createHyperGhastTextures(TEXTURE_RESOLUTION / 2, TEXTURE_RESOLUTION / 2, TENTACLE_COLOR_BASE, tentacleNoiseParams,[],[],[],0);

        // --- Materials ---
        const physicalMaterialProps = { roughness: 0.75, metalness: 0.0, clearcoat: 0.2, clearcoatRoughness: 0.3, transmission: 0.0, bumpScale: BUMP_SCALE }; function createGhastPhysicalMaterial(textures, intensity) { return new THREE.MeshPhysicalMaterial({ map: textures.texture, emissiveMap: textures.emissiveTexture, emissive: EMISSIVE_COLOR, emissiveIntensity: intensity, bumpMap: textures.emissiveTexture, ...physicalMaterialProps }); } const frontMaterial = createGhastPhysicalMaterial(frontTextures, SIGIL_EMISSIVE_INTENSITY_BASE); const sigilIntensity = SIGIL_EMISSIVE_INTENSITY_BASE; const bodyMaterials = [ createGhastPhysicalMaterial(sideTextures, sigilIntensity), createGhastPhysicalMaterial(sideTextures, sigilIntensity), createGhastPhysicalMaterial(topTextures, sigilIntensity), createGhastPhysicalMaterial(bottomTextures, 0), frontMaterial, createGhastPhysicalMaterial(backTextures, sigilIntensity),]; const hornMaterial = new THREE.MeshStandardMaterial({ map: hornTextures.texture, emissiveMap: hornTextures.emissiveTexture, emissive: EMISSIVE_COLOR, emissiveIntensity: HORN_SIGIL_EMISSIVE_INTENSITY_BASE, bumpMap: hornTextures.emissiveTexture, bumpScale: BUMP_SCALE * 1.5, roughness: 0.65, metalness: 0.1 }); const tentacleMaterial = new THREE.MeshStandardMaterial({ map: tentacleTextureData.texture, roughness: 0.85, metalness: 0.0 }); const tentacleTipMaterial = new THREE.MeshStandardMaterial({ color: TENTACLE_TIP_COLOR, roughness: 0.5, metalness: 0.1 });
        const geometricFaceCoreMaterial = new THREE.MeshStandardMaterial({ color: FACE_CORE_BASE_COLOR_HEX, emissive: RED_GLOW_COLOR_HEX, emissiveIntensity: FACE_CORE_EMISSIVE_INTENSITY, roughness: 0.8, metalness: 0.0 });
        const geometricFaceHaloMaterial = new THREE.MeshBasicMaterial({ color: RED_GLOW_COLOR_HEX, transparent: true, opacity: HALO_OPACITY, blending: THREE.AdditiveBlending, depthWrite: false, side: THREE.FrontSide });
        // --- Fireball Materials (Layered) ---
        const fireballCoreMaterial = new THREE.MeshStandardMaterial({ color: FACE_CORE_BASE_COLOR_HEX, emissive: RED_GLOW_COLOR_HEX, emissiveIntensity: FIREBALL_CORE_EMISSIVE_INTENSITY, roughness: 0.9, metalness: 0.0 });
        const fireballHaloMaterial = new THREE.MeshBasicMaterial({ color: RED_GLOW_COLOR_HEX, transparent: true, opacity: FIREBALL_HALO_OPACITY, blending: THREE.AdditiveBlending, depthWrite: false });

        // Store pulsating materials
        this.pulsatingMaterials = [ { material: geometricFaceHaloMaterial, baseValue: HALO_OPACITY, property: 'opacity' } ]; // Only face halo pulsates now

        // --- Geometry ---
        const bodyGeo = new THREE.BoxGeometry(BODY_SIZE, BODY_SIZE, BODY_SIZE);
        const tentacleBaseGeo = new THREE.BoxGeometry(TENTACLE_WIDTH, TENTACLE_HEIGHT, TENTACLE_WIDTH);
        const tentacleTipGeo = new THREE.ConeGeometry(TENTACLE_WIDTH * 0.6, TENTACLE_WIDTH * 1.2, 4);
        const fireballCoreGeo = new THREE.SphereGeometry(FIREBALL_CORE_SIZE, 6, 4);
        const fireballHaloGeo = new THREE.SphereGeometry(FIREBALL_HALO_SIZE, 8, 6);

        // --- Function to build a SCARIER curved Horn ---
        function createCurvedHorn(material) { const hornGroup = new THREE.Group(); hornGroup.name = "HornGroup"; let currentWidth = HORN_BASE_WIDTH; let currentDepth = HORN_BASE_DEPTH; let currentY = 0; for (let i = 0; i < HORN_SEGMENTS; i++) { const segmentGeo = new THREE.BoxGeometry(currentWidth, HORN_SEGMENT_HEIGHT, currentDepth); const segmentMesh = new THREE.Mesh(segmentGeo, material); segmentMesh.position.y = currentY + HORN_SEGMENT_HEIGHT / 2; const xRot = HORN_CURVE_ANGLE * (i + 1) * 1.4 + (Math.random() - 0.5) * 0.1; const zRot = HORN_CURVE_ANGLE * (i + 1) * 0.7 + HORN_TWIST_ANGLE * (i + Math.random()*2) + (Math.random() - 0.5) * 0.15; segmentMesh.rotation.x = xRot; segmentMesh.rotation.z = zRot; segmentMesh.rotation.y = (Math.random() - 0.5) * 0.1; segmentMesh.position.y -= Math.sin(xRot) * HORN_SEGMENT_HEIGHT * 0.5; segmentMesh.position.z += Math.sin(xRot) * HORN_SEGMENT_HEIGHT * 0.5; currentWidth *= HORN_TAPER; currentDepth *= HORN_TAPER; currentY += Math.cos(xRot) * HORN_SEGMENT_HEIGHT; segmentMesh.castShadow = true; hornGroup.add(segmentMesh); } return hornGroup; }

        // --- Create Creature Parts ---
        this.bodyYOffset = TENTACLE_HEIGHT / 2 + 1; // Calculate and store bodyYOffset
        const bodyYOffset = this.bodyYOffset;

        const tentacleGroup = new THREE.Group(); tentacleGroup.name = "TentacleGroup"; this.mesh.add(tentacleGroup);
        const body = new THREE.Mesh(bodyGeo, bodyMaterials); body.position.y = bodyYOffset; body.name = "GhastBody"; body.castShadow = true; body.receiveShadow = true; this.mesh.add(body);

        // --- Geometric Face Features (LAYERED) ---
        const faceZOffset = BODY_SIZE / 2 + 0.01; const haloZOffset = BODY_SIZE / 2 + 0.02; const eyeWidth = 2.2; const eyeHeight = 0.8; const eyeDepth = 0.6; const eyeYPos = bodyYOffset + 1.5; const eyeXOffset = 2.5; const eyeAngle = Math.PI / 12; const eyeGeoCore = new THREE.BoxGeometry(eyeWidth, eyeHeight, eyeDepth); const eyeGeoHalo = new THREE.BoxGeometry(eyeWidth * HALO_SCALE, eyeHeight * HALO_SCALE, eyeDepth * HALO_SCALE); const leftEyeCore = new THREE.Mesh(eyeGeoCore, geometricFaceCoreMaterial); leftEyeCore.position.set(-eyeXOffset, eyeYPos, faceZOffset); leftEyeCore.rotation.z = eyeAngle; leftEyeCore.castShadow = true; this.mesh.add(leftEyeCore); const leftEyeHalo = new THREE.Mesh(eyeGeoHalo, geometricFaceHaloMaterial); leftEyeHalo.position.set(-eyeXOffset, eyeYPos, haloZOffset); leftEyeHalo.rotation.z = eyeAngle; this.mesh.add(leftEyeHalo); const rightEyeCore = new THREE.Mesh(eyeGeoCore, geometricFaceCoreMaterial); rightEyeCore.position.set(eyeXOffset, eyeYPos, faceZOffset); rightEyeCore.rotation.z = -eyeAngle; rightEyeCore.castShadow = true; this.mesh.add(rightEyeCore); const rightEyeHalo = new THREE.Mesh(eyeGeoHalo, geometricFaceHaloMaterial); rightEyeHalo.position.set(eyeXOffset, eyeYPos, haloZOffset); rightEyeHalo.rotation.z = -eyeAngle; this.mesh.add(rightEyeHalo); const fangWidth = 0.4; const fangHeight = 1.6; const fangDepth = 0.4; const fangYPos = bodyYOffset - 2.2; const numFangs = 7; const fangSpacing = 0.7; for (let i = 0; i < numFangs; i++) { const currentFangHeight = fangHeight * (0.7 + Math.random() * 0.6); const currentFangGeoCore = new THREE.BoxGeometry(fangWidth, currentFangHeight, fangDepth); const currentFangGeoHalo = new THREE.BoxGeometry(fangWidth * HALO_SCALE * 0.9, currentFangHeight * HALO_SCALE, fangDepth * HALO_SCALE * 0.9); const xPos = (i - (numFangs - 1) / 2) * fangSpacing; const yPos = fangYPos - (currentFangHeight - fangHeight) / 2; const fangCore = new THREE.Mesh(currentFangGeoCore, geometricFaceCoreMaterial); fangCore.position.set(xPos, yPos, faceZOffset); fangCore.castShadow = true; this.mesh.add(fangCore); const fangHalo = new THREE.Mesh(currentFangGeoHalo, geometricFaceHaloMaterial); fangHalo.position.set(xPos, yPos, haloZOffset); this.mesh.add(fangHalo); }

        // --- Horns (Using Scarier Function) ---
        const hornLeftGroup = createCurvedHorn(hornMaterial); hornLeftGroup.position.set( -HORN_SPACING_X, body.position.y + BODY_SIZE / 2 - 0.4, -BODY_SIZE / 2 + HORN_BASE_DEPTH * 0.6); hornLeftGroup.rotation.y = -Math.PI / 16 - 0.1; hornLeftGroup.rotation.z = Math.PI / 48; this.mesh.add(hornLeftGroup); const hornRightGroup = createCurvedHorn(hornMaterial); hornRightGroup.position.set( HORN_SPACING_X, body.position.y + BODY_SIZE / 2 - 0.4, -BODY_SIZE / 2 + HORN_BASE_DEPTH * 0.6); hornRightGroup.children.forEach(segment => { segment.rotation.z *= -1; }); hornRightGroup.rotation.y = Math.PI / 16 + 0.1; hornRightGroup.rotation.z = -Math.PI / 48; this.mesh.add(hornRightGroup);

        // --- Tentacles (With Tips) ---
        const tentaclePositions = []; for (let i = -1; i <= 1; i++) { for (let j = -1; j <= 1; j++) { tentaclePositions.push({ x: i * TENTACLE_SPACING, z: j * TENTACLE_SPACING }); } } const varyHeight = true; const minTentacleHeightFactor = 0.8; tentaclePositions.forEach((pos, index) => { let currentTentacleHeight = TENTACLE_HEIGHT; let currentTentacleGeo = tentacleBaseGeo.clone(); if (varyHeight) { currentTentacleHeight = TENTACLE_HEIGHT * (minTentacleHeightFactor + Math.random() * (1.0 - minTentacleHeightFactor)); currentTentacleGeo = new THREE.BoxGeometry(TENTACLE_WIDTH, currentTentacleHeight, TENTACLE_WIDTH); } const tentacle = new THREE.Mesh(currentTentacleGeo, tentacleMaterial); const baseX = pos.x; const baseZ = pos.z; const baseY = body.position.y - BODY_SIZE / 2 - currentTentacleHeight / 2; tentacle.position.set(baseX, baseY, baseZ); tentacle.castShadow = true; tentacle.userData = { originalX: baseX, originalY: baseY, originalZ: baseZ, index: index, swaySpeedMult: 0.8 + Math.random() * 0.4, swayAmountMult: 0.8 + Math.random() * 0.4, swayPhaseOffset: Math.random() * Math.PI * 2 }; const tip = new THREE.Mesh(tentacleTipGeo, tentacleTipMaterial); tip.position.y = -currentTentacleHeight / 2 - (TENTACLE_WIDTH * 1.2) / 2 + 0.1; tip.rotation.x = Math.PI; tentacle.add(tip); tentacleGroup.add(tentacle); });

        // --- White Particle System Setup ---
        this.particleGeometry = new THREE.BufferGeometry(); const particlePositions = new Float32Array(PARTICLE_COUNT * 3); const particleAlphas = new Float32Array(PARTICLE_COUNT); this.particleLifetimes = new Float32Array(PARTICLE_COUNT); this.particleVelocities = []; const particleColor = new THREE.Color(EMISSIVE_COLOR); for (let i = 0; i < PARTICLE_COUNT; i++) { particlePositions[i * 3] = 0; particlePositions[i * 3 + 1] = -100; particlePositions[i * 3 + 2] = 0; particleAlphas[i] = 0.0; this.particleLifetimes[i] = 0.0; this.particleVelocities[i] = new THREE.Vector3(); } this.particleGeometry.setAttribute('position', new THREE.BufferAttribute(particlePositions, 3)); this.particleGeometry.setAttribute('alpha', new THREE.BufferAttribute(particleAlphas, 1)); const pCanvas = document.createElement('canvas'); pCanvas.width=16; pCanvas.height=16; const pCtx = pCanvas.getContext('2d'); pCtx.fillStyle='#fff'; pCtx.beginPath(); pCtx.arc(8,8,7,0,Math.PI*2); pCtx.fill(); const particleTexture = new THREE.CanvasTexture(pCanvas); const particleMaterial = new THREE.PointsMaterial({ size: PARTICLE_SIZE, map: particleTexture, color: particleColor, blending: THREE.AdditiveBlending, transparent: true, opacity: 1.0, depthWrite: false, sizeAttenuation: true }); this.particles = new THREE.Points(this.particleGeometry, particleMaterial); this.particles.name = "GhastParticles"; this.mesh.add(this.particles); const approxHornTotalHeight = HORN_SEGMENT_HEIGHT * HORN_SEGMENTS; this.particleSources = [ new THREE.Vector3(0, bodyYOffset + 1, BODY_SIZE / 2 + 0.5), new THREE.Vector3(-HORN_SPACING_X, bodyYOffset + BODY_SIZE / 2 + approxHornTotalHeight * 0.6, -BODY_SIZE/2 + HORN_BASE_DEPTH * 0.6), new THREE.Vector3( HORN_SPACING_X, bodyYOffset + BODY_SIZE / 2 + approxHornTotalHeight * 0.6, -BODY_SIZE/2 + HORN_BASE_DEPTH * 0.6), ];

        // --- Fireball Particle System Setup (Layered) ---
        this.fireballGroup = new THREE.Group(); this.fireballGroup.name = "FireballSwarm"; const fireballOrbitCenter = new THREE.Vector3(0, bodyYOffset, 0); // Use calculated bodyYOffset
        for (let i = 0; i < FIREBALL_COUNT; i++) { const fireballContainer = new THREE.Group(); const fireballCore = new THREE.Mesh(fireballCoreGeo, fireballCoreMaterial); fireballContainer.add(fireballCore); const fireballHalo = new THREE.Mesh(fireballHaloGeo, fireballHaloMaterial); fireballContainer.add(fireballHalo); const radius = MathUtils.randFloat(FIREBALL_ORBIT_RADIUS_MIN, FIREBALL_ORBIT_RADIUS_MAX); const phi = Math.acos(-1 + (2 * Math.random())); const theta = Math.random() * Math.PI * 2; fireballContainer.position.setFromSphericalCoords(radius, phi, theta); fireballContainer.position.add(fireballOrbitCenter); fireballContainer.userData = { noiseSeedX: Math.random() * 100, noiseSeedY: Math.random() * 100, noiseSeedZ: Math.random() * 100, initialRelativePosition: fireballContainer.position.clone().sub(fireballOrbitCenter) }; this.fireballGroup.add(fireballContainer); } this.mesh.add(this.fireballGroup);

        // --- Build Collision Cylinder Visualization --- // <<<--- NEW
        const cylinderGeo = new THREE.CylinderGeometry(
            this.cylinderRadius, // radiusTop
            this.cylinderRadius, // radiusBottom
            this.cylinderHeight, // height
            16, // radialSegments
            1,  // heightSegments
            true // openEnded (optional, slightly less geometry)
        );
        const cylinderMat = new THREE.MeshBasicMaterial({
            color: 0x00ff00, // Bright green for visibility
            wireframe: true,
            transparent: true,
            opacity: 0.3,
            depthWrite: false // Allows seeing things through it
        });
        this.collisionCylinderMesh = new THREE.Mesh(cylinderGeo, cylinderMat);
        this.collisionCylinderMesh.name = "GhastCollisionCylinder";
        // Position the cylinder mesh at the center of the main mesh group.
        // The main mesh group's position is controlled by this.position (physics center).
        this.collisionCylinderMesh.position.set(0, 0, 0);
        // Set initial visibility based on the flag
        this.collisionCylinderMesh.visible = this.showCollisionCylinder;
        // Add it to the main mesh group
        this.mesh.add(this.collisionCylinderMesh);
        // --- End Collision Cylinder --- //
    }

    // <<<--- MODIFIED: Added player and activeFireballs parameters --- >>>
    updateLogic(deltaTime, player, activeFireballs) {
        if (this.isDead) return; // Don't update logic if dead

        const time = this.clock.getElapsedTime();

        // --- Update Attack Cooldown --- <<< NEW
        if (this.attackCooldownTimer > 0) {
            this.attackCooldownTimer -= deltaTime;
        }

        // --- Player Detection & Attack Logic --- <<< NEW
        this.isAttackingOrCoolingDown = false; // Reset flag each frame
        this.currentTarget = null; // Reset target

        if (player && !player.isDead) { // Check if player exists and is not dead
            const distanceSq = this.position.distanceToSquared(player.position);
            const territory = this.movementController.territory;

            // Check if player is within territory bounds
            const playerInTerritory =
                player.position.x >= territory.minX && player.position.x <= territory.maxX &&
                player.position.z >= territory.minZ && player.position.z <= territory.maxZ;

            if (playerInTerritory && distanceSq < this.attackRangeSq) {
                this.currentTarget = player; // Set player as target
                this.isAttackingOrCoolingDown = true; // Set flag for movement controller

                // --- Fireball Attack ---
                if (this.attackCooldownTimer <= 0) {
                    this.fireFireball(player, activeFireballs);
                    this.attackCooldownTimer = this.fireballCooldown; // Reset cooldown
                }
            }
        }
        // --- End Player Detection & Attack Logic ---

        // --- Update Movement Controller ---
        if (this.movementController) {
            // Pass the player reference for potential targeting adjustments
            this.movementController.update(deltaTime, player);
        }
        // Note: The movement controller now handles setting `this.position`.
        // The mesh position `this.mesh.position` should be lerped towards `this.position`
        // in the main animate loop for smoothness.
        // Rotation (`this.mesh.quaternion`) is handled by movement controller.


        // --- Animation Constants (Defined within updateLogic) ---
        const TENTACLE_SWAY_SPEED = 1.9; // <<< More sway
        const TENTACLE_SWAY_AMOUNT = 0.5;
        const TENTACLE_ROT_AMOUNT = 0.07;
        const TENTACLE_JITTER_AMOUNT = 0.15; // <<< More jitter
        const BODY_BREATHE_SPEED = 0.9; const BODY_BREATHE_AMOUNT = 0.015;
        const GLOW_PULSATION_SPEED = 1.8; // <<< Faster pulse
        const GLOW_PULSATION_AMOUNT = 0.6; // <<< Wider pulse range
        const PARTICLE_SPAWN_RATE = 150; const PARTICLE_LIFETIME = 1.1; const PARTICLE_INITIAL_VELOCITY = 1.4; const PARTICLE_DRAG = 0.97; const PARTICLE_COUNT = 250;
        const HALO_OPACITY = 0.55; // Base opacity for face halo pulse
        const FIREBALL_HALO_OPACITY = 0.7; // Base opacity for fireball halo
        const TWITCH_FREQUENCY_MIN = 0.3; // <<< Much More frequent twitch
        const TWITCH_FREQUENCY_MAX = 0.9;
        const TWITCH_MOVE_AMOUNT = 0.35; // <<< More intense twitch movement
        const TWITCH_ROT_AMOUNT = Math.PI / 48; // <<< More intense twitch rotation
        const FIREBALL_NOISE_AMOUNT = 1.2; // <<< More chaotic fireball noise
        const FIREBALL_NOISE_SPEED = 0.9; // <<< Faster fireball noise
        const FIREBALL_SYSTEM_ROTATION_SPEED = 0.4; // <<< Faster fireball rotation
        const BODY_SIZE = 9; // Define BODY_SIZE here
        const FIREBALL_VERTICAL_SPREAD = BODY_SIZE * 0.5;

        // --- Update Creature Internal Animations (Relative to Mesh Position) ---
        const creatureGroup = this.mesh; const body = creatureGroup.getObjectByName("GhastBody"); const tentacleGroup = creatureGroup.getObjectByName("TentacleGroup"); const childGroups = creatureGroup.children.filter(c => c.isGroup && c !== tentacleGroup && c !== this.particles && c !== body && !c.geometry && c !== this.fireballGroup && c !== this.collisionCylinderMesh); // Exclude cylinder
        const hornLeftGroup = childGroups.find(g => g.position.x < 0); const hornRightGroup = childGroups.find(g => g.position.x > 0);

        // --- Twitch Animation (Applies offset to internal parts, not the main mesh position) ---
        this.timeUntilTwitch -= deltaTime; if (this.timeUntilTwitch <= 0 && this.twitchTimer <= 0) { this.twitchTimer = this.twitchDuration; this.twitchOffset.set( (Math.random() - 0.5) * TWITCH_MOVE_AMOUNT, (Math.random() - 0.5) * TWITCH_MOVE_AMOUNT * 0.5, (Math.random() - 0.5) * TWITCH_MOVE_AMOUNT ); this.twitchRotOffset.set( (Math.random() - 0.5) * TWITCH_ROT_AMOUNT, (Math.random() - 0.5) * TWITCH_ROT_AMOUNT, (Math.random() - 0.5) * TWITCH_ROT_AMOUNT ); } let currentTwitchFactor = 0; if (this.twitchTimer > 0) { this.twitchTimer -= deltaTime; currentTwitchFactor = Math.sin((1.0 - this.twitchTimer / this.twitchDuration) * Math.PI); if (this.twitchTimer <= 0) { this.twitchTimer = 0; this.timeUntilTwitch = TWITCH_FREQUENCY_MIN + Math.random() * (TWITCH_FREQUENCY_MAX - TWITCH_FREQUENCY_MIN); this.twitchOffset.set(0,0,0); this.twitchRotOffset.set(0,0,0); currentTwitchFactor = 0; } }

        // --- Body Breathing (Scaling the body mesh itself) ---
        if (body) {
            const breatheFactor = Math.sin(time * BODY_BREATHE_SPEED) * BODY_BREATHE_AMOUNT;
            body.scale.set(1 + breatheFactor, 1 + breatheFactor, 1 + breatheFactor);
            // Apply twitch offset relative to body center if desired
            body.position.y = this.bodyYOffset + this.twitchOffset.y * currentTwitchFactor;
            // We don't apply main mesh rotation here, movement controller does that
        }

        // Apply twitch rotation offset to the main mesh (movement controller handles base rotation)
        creatureGroup.rotation.x += this.twitchRotOffset.x * currentTwitchFactor;
        creatureGroup.rotation.z += this.twitchRotOffset.z * currentTwitchFactor;
        // Note: We don't directly add twitchRotOffset.y here, as the movement controller manages Y rotation (look direction)

        // Horns follow body tilt (which is now part of twitchRotOffset applied to the group)
        // Horn rotation adjustment might need tweaking depending on desired effect
        if (hornLeftGroup) { hornLeftGroup.rotation.x = creatureGroup.rotation.x * 0.3; hornLeftGroup.rotation.z = creatureGroup.rotation.z * 0.3 + Math.PI / 48; } if (hornRightGroup) { hornRightGroup.rotation.x = creatureGroup.rotation.x * 0.3; hornRightGroup.rotation.z = creatureGroup.rotation.z * 0.3 - Math.PI / 48; }

        // Tentacle Animation (with Jitter) - No changes needed here, it's relative
        if (tentacleGroup) { tentacleGroup.children.forEach(tentacle => { const { originalX, originalY, originalZ, index, swaySpeedMult, swayAmountMult, swayPhaseOffset } = tentacle.userData; const swayOffset = time * TENTACLE_SWAY_SPEED * swaySpeedMult + swayPhaseOffset + index * 0.3; const currentSwayAmount = TENTACLE_SWAY_AMOUNT * swayAmountMult; const jitterX = (Math.random() - 0.5) * TENTACLE_JITTER_AMOUNT; const jitterZ = (Math.random() - 0.5) * TENTACLE_JITTER_AMOUNT; tentacle.position.x = originalX + Math.sin(swayOffset) * currentSwayAmount + jitterX; tentacle.position.z = originalZ + Math.cos(swayOffset * 0.8) * currentSwayAmount * 0.7 + jitterZ; tentacle.position.y = originalY; tentacle.rotation.y = Math.sin(swayOffset * 0.5) * TENTACLE_ROT_AMOUNT + jitterX * 0.1; tentacle.rotation.x = Math.cos(swayOffset * 0.6) * TENTACLE_ROT_AMOUNT * 0.5 + jitterZ * 0.1; }); }

        // Pulsating Glow Animation - No changes needed
        const pulseFactor = (Math.sin(time * GLOW_PULSATION_SPEED) + 1) / 2;
        this.pulsatingMaterials.forEach((pulseData) => {
             const { material, baseValue, property } = pulseData;
             if (material && typeof baseValue === 'number' && property) {
                  material[property] = baseValue * (1.0 - GLOW_PULSATION_AMOUNT + pulseFactor * GLOW_PULSATION_AMOUNT * 2);
              }
        });

        // Fireball Particle Update - No changes needed here, relative to mesh
        if (this.fireballGroup) {
            const orbitCenter = new THREE.Vector3(0, this.bodyYOffset, 0); // Use stored bodyYOffset
            this.fireballGroup.children.forEach(fireballContainer => { // Iterate through containers
                 const data = fireballContainer.userData;
                 const noiseTime = time * FIREBALL_NOISE_SPEED;
                 // Calculate noise offset
                 const noiseX = (MathUtils.seededRandom(data.noiseSeedX + noiseTime) - 0.5) * FIREBALL_NOISE_AMOUNT;
                 const noiseY = (MathUtils.seededRandom(data.noiseSeedY + noiseTime) - 0.5) * FIREBALL_NOISE_AMOUNT;
                 const noiseZ = (MathUtils.seededRandom(data.noiseSeedZ + noiseTime) - 0.5) * FIREBALL_NOISE_AMOUNT;
                 // Apply noise relative to initial position stored in userData
                 fireballContainer.position.copy(data.initialRelativePosition)
                                            .add(new THREE.Vector3(noiseX, noiseY, noiseZ));
            });
            // Rotate the entire group around the body center (Y axis)
            this.fireballGroup.position.copy(orbitCenter); // Ensure group is centered
            this.fireballGroup.rotation.y += FIREBALL_SYSTEM_ROTATION_SPEED * deltaTime;
             this.fireballGroup.rotation.x += FIREBALL_SYSTEM_ROTATION_SPEED * 0.4 * deltaTime; // Add more axis wobble
            this.fireballGroup.rotation.z -= FIREBALL_SYSTEM_ROTATION_SPEED * 0.2 * deltaTime;
             this.fireballGroup.position.sub(orbitCenter); // Offset back after rotation
        }

        // White Particle Animation & Spawning - No changes needed, uses matrixWorld
        if (this.particles && this.particleGeometry) { this.particlesToSpawn += PARTICLE_SPAWN_RATE * deltaTime; const posAttribute = this.particleGeometry.attributes.position; const alphaAttribute = this.particleGeometry.attributes.alpha; let spawnCount = Math.floor(this.particlesToSpawn); this.particlesToSpawn -= spawnCount; let nextAvailableParticleIndex = 0; const _tempVec = new THREE.Vector3(); for (let i = 0; i < PARTICLE_COUNT; i++) { if (this.particleLifetimes[i] > 0) { this.particleLifetimes[i] -= deltaTime; if (this.particleLifetimes[i] <= 0) { alphaAttribute.setX(i, 0); this.particleLifetimes[i] = 0; } else { this.particleVelocities[i].multiplyScalar(PARTICLE_DRAG); this.particleVelocities[i].y += 0.5 * deltaTime; posAttribute.setXYZ( i, posAttribute.getX(i) + this.particleVelocities[i].x * deltaTime, posAttribute.getY(i) + this.particleVelocities[i].y * deltaTime, posAttribute.getZ(i) + this.particleVelocities[i].z * deltaTime ); alphaAttribute.setX(i, Math.max(0, (this.particleLifetimes[i] / PARTICLE_LIFETIME))); } } else if (spawnCount > 0 && nextAvailableParticleIndex <= i) { const particleIndex = i; spawnCount--; nextAvailableParticleIndex = i + 1; const sourceIndex = Math.floor(Math.random() * this.particleSources.length); const sourcePosLocal = this.particleSources[sourceIndex].clone(); sourcePosLocal.add(new THREE.Vector3( (Math.random() - 0.5) * 1.8, (Math.random() - 0.5) * 1.8, (Math.random() - 0.5) * 1.8 )); const sourcePosWorld = _tempVec.copy(sourcePosLocal).applyMatrix4(this.mesh.matrixWorld); posAttribute.setXYZ(particleIndex, sourcePosWorld.x, sourcePosWorld.y, sourcePosWorld.z); alphaAttribute.setX(particleIndex, 1.0); this.particleLifetimes[particleIndex] = PARTICLE_LIFETIME * (0.7 + Math.random() * 0.6); const velocity = new THREE.Vector3( (Math.random() - 0.5), Math.random() * 0.6 + 0.1, (Math.random() - 0.5) ); velocity.normalize().multiplyScalar(PARTICLE_INITIAL_VELOCITY * (0.6 + Math.random() * 0.8)); this.particleVelocities[particleIndex].copy(velocity); } } posAttribute.needsUpdate = true; alphaAttribute.needsUpdate = true; }

        // Update visual position for potential lerping in main loop
        this.visualPosition.copy(this.position); // <<<--- Ensure visualPosition tracks physics position

        // --- Update Collision Cylinder Visibility --- // <<<--- NEW
        if (this.collisionCylinderMesh) {
            this.collisionCylinderMesh.visible = this.showCollisionCylinder;
        }
    }

    // --- NEW: Method to fire a fireball ---
    fireFireball(player, activeFireballs) {
        if (!player || this.isDead) return;

        // Calculate start position (e.g., slightly in front of the body center)
        const body = this.mesh.getObjectByName("GhastBody");
        const startOffset = new THREE.Vector3(0, 0, 1); // Offset in local Z (forward)
        if (body) {
            startOffset.applyQuaternion(body.quaternion); // Rotate offset by body rotation
        }
        startOffset.multiplyScalar(this.cylinderRadius * 0.5); // Scale offset
        const startPos = this.position.clone().add(startOffset);
        startPos.y += this.bodyYOffset * 0.5; // Adjust vertical start point slightly

        // Calculate direction towards the player
        const direction = new THREE.Vector3().subVectors(player.position, startPos).normalize();

        // Create the fireball
        const fireball = new Fireball(
            this.scene,
            startPos,
            direction,
            this.fireballSpeed,
            this.fireballDamage,
            this.fireballLifetime,
            this // Pass self as owner
        );

        // Add to the global list managed by main.js
        activeFireballs.push(fireball);

        // TODO: Play fireball shoot sound via AudioManager
        // this.audioManager.playSound('fireball_shoot');

        console.log("Ghast fired fireball!"); // Debug log
    }

    // Placeholder methods
    takeDamage(amount) { /* ... (logic unchanged) ... */ if (this.isDead) return; this.health -= amount; console.log(`Ghast took ${amount} damage, health: ${this.health}`); if (this.health <= 0) { this.die(); } }
    die() { /* ... (logic unchanged) ... */ if (this.isDead) return; console.log("Scary Ghast died!"); this.isDead = true; let fadeDuration = 1.0; let fadeTimer = 0; this.mesh.traverse(child => { if (child.material) { if(Array.isArray(child.material)){ child.material.forEach(m => {m.transparent = true; m.depthWrite=false;}); } else { child.material.transparent = true; child.material.depthWrite = false; } } }); const fadeInterval = setInterval(() => { fadeTimer += 0.05; const opacity = Math.max(0, 1.0 - fadeTimer / fadeDuration); this.mesh.traverse(child => { if (child.material) { if(Array.isArray(child.material)){ child.material.forEach(m => m.opacity = opacity); } else { child.material.opacity = opacity; } } }); if (opacity <= 0) { clearInterval(fadeInterval); if (this.mesh.parent) this.mesh.parent.remove(this.mesh); /* Dispose materials/geometry here if needed */ if (this.collisionCylinderMesh && this.collisionCylinderMesh.geometry) this.collisionCylinderMesh.geometry.dispose(); if (this.collisionCylinderMesh && this.collisionCylinderMesh.material) this.collisionCylinderMesh.material.dispose(); if (this.particleGeometry) this.particleGeometry.dispose(); /* Add disposal for other unique materials/geometries */ } }, 50); } // <<<--- Added removal from scene and basic disposal on fade complete
}

// --- END OF FILE Ghast.js ---
```

---

**5. Modified File: `GhastMovement.js`**

```javascript
// src/GhastMovement.js
import * as THREE from 'three';
import { MathUtils } from 'three/src/math/MathUtils.js'; // For random number generation

export class GhastMovement {
    constructor(ghast, world, territory) {
        if (!ghast || !world || !territory || territory.minX === undefined) {
            console.error("GhastMovement constructor missing required arguments or invalid territory:", { ghast, world, territory });
            throw new Error("GhastMovement requires ghast, world, and valid territory objects.");
        }
        this.ghast = ghast;
        this.world = world;
        this.territory = territory; // { minX, maxX, minZ, maxZ } in world coordinates

        // --- Movement Parameters ---
        this.speed = 3.5; // Average world units per second
        this.turnSpeed = 0.8; // Radians per second for smooth turning
        this.targetReachedThresholdSq = 1.5 * 1.5; // Squared distance to consider target reached
        this.newTargetIntervalMin = 4.0; // Seconds minimum before picking new target (when idle)
        this.newTargetIntervalMax = 9.0; // Seconds maximum (when idle)
        this.altitudeMin = 5; // <<<--- INCREASED Min blocks above ground
        this.altitudeMax = 20; // Max blocks above ground
        this.preferredAltitudeMin = 8; // <<<--- INCREASED Preferred range
        this.preferredAltitudeMax = 15;
        this.preferredAltitudeProbability = 0.8; // 80% chance (when idle)

        // --- Attack Movement Parameters --- <<< NEW
        this.minAttackDistance = 15.0; // Try to stay further than this
        this.maxAttackDistance = 25.0; // Try to stay closer than this
        this.hoverSpeedMultiplier = 0.5; // Slower speed when hovering/adjusting distance

        // --- State Variables ---
        this.currentTargetPosition = null;
        this.timeSinceTarget = Math.random() * this.newTargetIntervalMax; // Start with random time

        // --- Reusable Objects for Calculation ---
        this._tempTargetVec = new THREE.Vector3();
        this._tempDirectionVec = new THREE.Vector3();
        this._currentQuat = new THREE.Quaternion();
        this._targetQuat = new THREE.Quaternion();
        this._upVec = new THREE.Vector3(0, 1, 0);
        this._lookHelper = new THREE.Object3D();
        this._tempPlayerDirection = new THREE.Vector3(); // For attack movement
    }

    // <<<--- MODIFIED: Added player parameter --- >>>
    update(deltaTime, player) {
        this.timeSinceTarget += deltaTime;

        // --- Decide Movement Mode: Idle/Wander vs Attack/Maintain Distance ---
        let isAttacking = this.ghast.isAttackingOrCoolingDown && player && !player.isDead;
        let currentSpeed = this.speed;

        if (isAttacking) {
            // --- Attack Movement Logic ---
            this._calculateAttackTargetPosition(player);
            currentSpeed *= this.hoverSpeedMultiplier; // Move slower when attacking/hovering
            this.timeSinceTarget = 0; // Prevent idle target switching while attacking

        } else {
            // --- Idle/Wander Movement Logic ---
            let needsNewTarget = false;
            if (!this.currentTargetPosition) {
                needsNewTarget = true;
            } else {
                const distanceSq = this.ghast.position.distanceToSquared(this.currentTargetPosition);
                if (distanceSq < this.targetReachedThresholdSq) {
                    needsNewTarget = true;
                } else if (this.timeSinceTarget > this.newTargetIntervalMax) {
                    needsNewTarget = true;
                } else if (this.timeSinceTarget > this.newTargetIntervalMin && Math.random() < 0.1 * deltaTime) {
                    needsNewTarget = true; // Small chance to switch target early
                }
            }

            if (needsNewTarget) {
                this._calculateNewWanderTargetPosition();
                this.timeSinceTarget = 0; // Reset timer
            }
        }


        // --- Move Towards Current Target (Applies to both modes) ---
        if (this.currentTargetPosition) {
            // Calculate direction (normalized)
            this._tempDirectionVec.subVectors(this.currentTargetPosition, this.ghast.position);
            const distanceToTarget = this._tempDirectionVec.length();

            // --- Smooth Rotation (Revised) ---
            if (distanceToTarget > 0.1) { // Only rotate if actually moving
                // Normalize the direction vector
                this._tempDirectionVec.normalize();

                // Use helper object to look along the direction
                this._lookHelper.position.set(0,0,0); // Reset helper position
                this._lookHelper.lookAt(this._tempDirectionVec); // Make helper look along the direction
                this._targetQuat.copy(this._lookHelper.quaternion); // Get the target orientation

                // Get current mesh quaternion
                this._currentQuat.copy(this.ghast.mesh.quaternion);

                // Interpolate using Slerp
                this._currentQuat.slerp(this._targetQuat, this.turnSpeed * deltaTime);

                // Apply the interpolated rotation
                this.ghast.mesh.quaternion.copy(this._currentQuat);

            } // End rotation logic


            // Calculate move distance for this frame AFTER rotation logic
            let moveDistance = currentSpeed * deltaTime; // Use potentially modified speed
            moveDistance = Math.min(moveDistance, distanceToTarget); // Don't overshoot

            // Calculate potential next position using the CURRENT direction (before normalization above)
            this._tempDirectionVec.subVectors(this.currentTargetPosition, this.ghast.position); // Re-calculate (or store before normalize)
            this._tempTargetVec.copy(this.ghast.position).addScaledVector(this._tempDirectionVec.normalize(), moveDistance);


            // --- Boundary Enforcement ---
            // Clamp X and Z to territory
            this._tempTargetVec.x = MathUtils.clamp(this._tempTargetVec.x, this.territory.minX, this.territory.maxX);
            this._tempTargetVec.z = MathUtils.clamp(this._tempTargetVec.z, this.territory.minZ, this.territory.maxZ);

            // Clamp Y based on ground height *at the potential new XZ*
            const groundY = this._getGroundHeight(this._tempTargetVec.x, this._tempTargetVec.z);
            const minY = groundY + this.altitudeMin; // Use updated min altitude
            const maxY = groundY + this.altitudeMax;
            this._tempTargetVec.y = MathUtils.clamp(this._tempTargetVec.y, minY, maxY);

            // --- Apply Position Update ---
            this.ghast.position.copy(this._tempTargetVec);

        }
    }

    // --- NEW: Calculate target position when attacking/maintaining distance ---
    _calculateAttackTargetPosition(player) {
        if (!player) return; // Should not happen if called correctly

        const ghastPos = this.ghast.position;
        const playerPos = player.position;
        const idealDistance = (this.minAttackDistance + this.maxAttackDistance) / 2;

        // Direction from Ghast to Player
        this._tempPlayerDirection.subVectors(playerPos, ghastPos);
        const currentDistance = this._tempPlayerDirection.length();
        this._tempPlayerDirection.normalize(); // Normalize AFTER getting length

        let targetX, targetZ;
        const hoverOffset = 2.0; // Small distance for hover adjustment

        if (currentDistance < this.minAttackDistance) {
            // Too close, move away from player
            targetX = ghastPos.x - this._tempPlayerDirection.x * hoverOffset;
            targetZ = ghastPos.z - this._tempPlayerDirection.z * hoverOffset;
        } else if (currentDistance > this.maxAttackDistance) {
            // Too far, move towards player
            targetX = ghastPos.x + this._tempPlayerDirection.x * hoverOffset;
            targetZ = ghastPos.z + this._tempPlayerDirection.z * hoverOffset;
        } else {
            // Within ideal range, hover slightly (optional: add slight random strafe)
            targetX = ghastPos.x + (Math.random() - 0.5) * 0.5; // Small random jitter
            targetZ = ghastPos.z + (Math.random() - 0.5) * 0.5;
        }

        // Determine target altitude (maintain preferred altitude)
        const groundY = this._getGroundHeight(targetX, targetZ);
        let targetAltitude = MathUtils.randFloat(this.preferredAltitudeMin, this.preferredAltitudeMax);
        let targetY = groundY + targetAltitude;
        targetY = MathUtils.clamp(targetY, groundY + this.altitudeMin, groundY + this.altitudeMax);

        // Set the target position
        if (!this.currentTargetPosition) {
            this.currentTargetPosition = new THREE.Vector3();
        }
        this.currentTargetPosition.set(targetX, targetY, targetZ);
    }


    // --- Renamed: Calculate target position when wandering ---
    _calculateNewWanderTargetPosition() {
        // Random X/Z within territory
        const targetX = MathUtils.randFloat(this.territory.minX, this.territory.maxX);
        const targetZ = MathUtils.randFloat(this.territory.minZ, this.territory.maxZ);

        // Find ground height below target X/Z
        const groundY = this._getGroundHeight(targetX, targetZ);

        // Determine target altitude based on probability
        let targetAltitude;
        if (Math.random() < this.preferredAltitudeProbability) {
            targetAltitude = MathUtils.randFloat(this.preferredAltitudeMin, this.preferredAltitudeMax);
        } else {
            // Choose randomly from the remaining lower or upper ranges
            if (Math.random() < 0.5) { // Lower range
                targetAltitude = MathUtils.randFloat(this.altitudeMin, this.preferredAltitudeMin);
            } else { // Upper range
                targetAltitude = MathUtils.randFloat(this.preferredAltitudeMax, this.altitudeMax);
            }
        }

        // Calculate final target Y and clamp
        let targetY = groundY + targetAltitude;
        targetY = MathUtils.clamp(targetY, groundY + this.altitudeMin, groundY + this.altitudeMax);

        // Set the target position
        if (!this.currentTargetPosition) {
            this.currentTargetPosition = new THREE.Vector3();
        }
        this.currentTargetPosition.set(targetX, targetY, targetZ);
    }

    _getGroundHeight(worldX, worldZ) {
        if (!this.world || !this.world.getBlock || this.world.worldCenterOffset === undefined || !this.world.params) {
            // console.warn("GhastMovement: World object invalid for ground check.");
            return 5; // Fallback height
        }
        const gridX = Math.floor(worldX + this.world.worldCenterOffset);
        const gridZ = Math.floor(worldZ + this.world.worldCenterOffset);
        const worldHeight = this.world.params.height;

        // Scan down from a reasonable height (current altitude + buffer)
        const startScanY = Math.min(worldHeight - 1, Math.floor(this.ghast.position.y + 5));
        for (let y = startScanY; y >= 0; y--) {
            const block = this.world.getBlock(gridX, y, gridZ);
            // Check if block is not null/undefined AND is not 'air'
            if (block && block.id !== 'air') {
                return y + 1.0; // Return the Y level *above* the ground block
            }
        }
        return 0; // If no ground found (e.g., outside world vertical bounds somehow), return 0
    }
}
```

---

**6. Modified File: `physics.js`**

```javascript
// --- START OF FILE physics.js ---
import * as THREE from 'three';
import { updatePlayer } from './player_physics.js';
import { updateCreature } from './martian_physics.js';

// Temporary vectors to avoid allocations in loops (existing optimization)
const _tempVec3_1 = new THREE.Vector3();
const _tempVec3_2 = new THREE.Vector3();
const _aabbCenter = new THREE.Vector3();
const _fireballCheckPos = new THREE.Vector3(); // <<< NEW: For fireball collision check
const _playerCheckPos = new THREE.Vector3(); // <<< NEW: For fireball collision check

export class Physics {
    // <<< MODIFIED: Added activeFireballs parameter --- >>>
    constructor(scene, barriers, activeFireballs) {
        if (!scene) {
            console.error("Physics constructor requires a scene object.");
            // Potentially throw an error or handle gracefully
        }
        this.scene = scene;
        this.barriers = barriers || []; // Ensure barriers is an array
        this.gravity = new THREE.Vector3(0, -30.8, 0);
        this.timeStep = 1 / 60; // Using a reasonably high frequency fixed step
        this.accumulatedTime = 0;
        this.creatures = [];
        this.activeFireballs = activeFireballs || []; // <<< Store reference to fireballs array
        this.collisionEpsilon = 0.001; // Small value to prevent floating point issues and jitter
        this.maxSubSteps = 10; // Limit physics steps per frame to prevent spiral of death

        // Reusable objects for ground checking
        this._groundCheckRaycaster = new THREE.Raycaster();
        this._groundCheckRayOrigin = new THREE.Vector3();
        this._groundCheckDownVector = new THREE.Vector3(0, -1, 0);
    }

    addCreature(creature) {
        if (creature) {
            this.creatures.push(creature);
        } else {
            console.warn("Attempted to add null/undefined creature to physics.");
        }
    }

    update(deltaTime, player, world) {
        // Basic checks for essential components
        if (!player || !world || !world.params || !world.instancedMeshes) {
            console.error("Physics update missing player or valid world object. Skipping update.");
            return;
        }

        this.accumulatedTime += deltaTime;
        let steps = 0;
        const wasOnGround = player.onGround; // Store initial onGround state for Coyote Time

        // Fixed timestep loop
        while (this.accumulatedTime >= this.timeStep && steps < this.maxSubSteps) {
            this.accumulatedTime -= this.timeStep;
            steps++;

            // Decrement Coyote Time and Jump Buffer timers (ensure player exists)
            if (player) {
                 if (player.coyoteTimer > 0) {
                    player.coyoteTimer -= this.timeStep;
                    if (player.coyoteTimer < 0) player.coyoteTimer = 0;
                 }
                 if (player.jumpBufferTimer > 0) {
                    player.jumpBufferTimer -= this.timeStep;
                    if (player.jumpBufferTimer < 0) player.jumpBufferTimer = 0;
                 }
            }

            // Update player and creatures (applies forces/velocity, moves position)
            // Ensure update functions exist before calling
            if (player && typeof updatePlayer === 'function') {
                updatePlayer(this, player, world, this.timeStep);
            }
            this.creatures.forEach(creature => {
                if (creature && typeof updateCreature === 'function') {
                     updateCreature(this, creature, player, world, this.timeStep);
                }
            });

            // Collision Detection and Resolution Phase
            if (player) {
                this.handleCollisions(player, world);
            }
            this.creatures.forEach(creature => {
                if (creature) { // Check creature exists before handling collisions
                    this.handleCollisions(creature, world);
                }
            });
        }

        // Warning if physics can't keep up
        if (steps >= this.maxSubSteps) {
            console.warn(`Physics step limit (${this.maxSubSteps}) reached. Resetting accumulated time.`);
            this.accumulatedTime = 0; // Prevent spiral
        }

        // --- Fireball Collision Check (after physics steps) --- <<< NEW
        this.handleFireballCollisions(player);
        // --- End Fireball Collision Check ---

        // Final ground check after all physics updates
        // Prepare meshes to exclude (player model, creature models)
        const meshesToExclude = [];
        if (player && player.human && player.human.mesh) {
             meshesToExclude.push(player.human.mesh);
        }
        this.creatures.forEach(c => {
            if (c && c.mesh) meshesToExclude.push(c.mesh);
        });


        // Perform ground check for player
        if (player) {
            // Only check ground if entity might be near it (velocity.y <= 0.1) or just left ground
            if (player.velocity.y <= 0.1 || wasOnGround) {
                this.checkGround(player, world, meshesToExclude);
            } else {
                player.onGround = false; // Entity is moving upwards, assume not on ground
            }
        }

        // Perform ground check for creatures
        this.creatures.forEach(creature => {
            if (creature) { // Check creature exists
                if (creature.velocity.y <= 0.1) { // Only check if potentially near ground
                    this.checkGround(creature, world, meshesToExclude);
                } else {
                    creature.onGround = false;
                }
            }
        });

        // Manage Coyote Time: Start timer if player just left the ground
        if (player && wasOnGround && !player.onGround) {
            player.coyoteTimer = player.coyoteTimeDuration;
        }

        // Check Jump Buffer: Execute jump if buffered and now on ground
        if (player && player.jumpBufferTimer > 0 && (player.onGround || player.coyoteTimer > 0)) { // <<< MODIFIED: Allow jump during coyote time
            // Assuming player has jump velocity defined elsewhere or default here
            player.velocity.y = player.jumpSpeed; // Use player's jumpSpeed
            player.onGround = false;
            player.jumpBufferTimer = 0; // Consume buffer
            player.coyoteTimer = 0; // Consume coyote time if used
        }
    }

    // Encapsulates collision handling per entity
    handleCollisions(entity, world) {
        if (!entity || !entity.position || !world) return; // Basic checks

        const entityAABB = this.computeCylinderAABB(entity); // Broadphase AABB for candidate selection
        const candidateBlocks = this.getCandidateBlocks(entityAABB, world);
        const collisions = this.detectCollisions(entity, candidateBlocks, world);
        this.resolveCollisions(entity, collisions);
    }

    computeCylinderAABB(entity) {
        // Computes an AABB that completely encloses the physics cylinder
        const halfHeight = entity.cylinderHeight / 2;
        const radius = entity.cylinderRadius;
        const position = entity.position;

        const min = _tempVec3_1.set( // Reuse temporary vector
            position.x - radius,
            position.y - halfHeight,
            position.z - radius
        );
        const max = _tempVec3_2.set( // Reuse temporary vector
            position.x + radius,
            position.y + halfHeight,
            position.z + radius
        );

        // Return a new Box3 (or could reuse a Box3 instance if careful)
        return new THREE.Box3(min.clone(), max.clone());
    }

    getCandidateBlocks(entityAABB, world) {
        // Find world grid cells overlapping the entity's AABB
        // Use the pre-calculated offset from the world
        const centerOffset = world.worldCenterOffset;
        const buffer = 0.1; // Small buffer to catch edge cases

        // Convert entity AABB world coordinates to potential grid indices
        // Add 0.5 because block indices refer to the block *containing* that point
        const minX = entityAABB.min.x + centerOffset - buffer;
        const maxX = entityAABB.max.x + centerOffset + buffer;
        const minY = entityAABB.min.y - buffer; // Y grid index starts at 0 for world Y=0
        const maxY = entityAABB.max.y + buffer;
        const minZ = entityAABB.min.z + centerOffset - buffer;
        const maxZ = entityAABB.max.z + centerOffset + buffer;

        // Clamp indices to world bounds
        const i_min = Math.max(0, Math.floor(minX));
        const i_max = Math.min(world.params.width - 1, Math.floor(maxX));
        const j_min = Math.max(0, Math.floor(minY));
        const j_max = Math.min(world.params.height - 1, Math.floor(maxY));
        const k_min = Math.max(0, Math.floor(minZ));
        const k_max = Math.min(world.params.width - 1, Math.floor(maxZ));

        const candidates = [];
        for (let i = i_min; i <= i_max; i++) {
            for (let j = j_min; j <= j_max; j++) {
                for (let k = k_min; k <= k_max; k++) {
                    // Use world.getBlock method which handles bounds checks internally
                    const block = world.getBlock(i, j, k);
                    if (block && block.id !== 'air') {
                        candidates.push({ i, j, k }); // Store grid coordinates
                    }
                }
            }
        }
        return candidates;
    }

    detectCollisions(entity, candidateBlocks, world) {
        const collisions = [];
        const cylinderPosition = entity.position;
        const cylinderRadius = entity.cylinderRadius;
        const cylinderHeight = entity.cylinderHeight;

        // Check against candidate blocks
        candidateBlocks.forEach(blockCoords => {
            const blockAABB = this.getBlockAABB(blockCoords, world);
            const collision = this.checkCylinderAABBIntersectionMTV(
                cylinderPosition,
                cylinderRadius,
                cylinderHeight,
                blockAABB
            );
            if (collision.intersects) {
                collisions.push({
                    type: 'block',
                    blockCoords: blockCoords, // Keep grid coords for potential use
                    penetrationDepth: collision.penetrationDepth,
                    normal: collision.normal,
                    aabb: blockAABB // Keep AABB for resolution reference
                });
            }
        });

        // Check against world boundary barriers
        this.barriers.forEach(barrierAABB => {
            const collision = this.checkCylinderAABBIntersectionMTV(
                cylinderPosition,
                cylinderRadius,
                cylinderHeight,
                barrierAABB
            );
            if (collision.intersects) {
                collisions.push({
                    type: 'barrier',
                    penetrationDepth: collision.penetrationDepth,
                    normal: collision.normal,
                    aabb: barrierAABB
                });
            }
        });

        return collisions;
    }

    checkCylinderAABBIntersectionMTV(cylinderPosition, cylinderRadius, cylinderHeight, aabb) {
         // Basic AABB vs Cylinder check (simplified SAT-like)

         const halfHeight = cylinderHeight / 2;
         const cylinderBottom = cylinderPosition.y - halfHeight;
         const cylinderTop = cylinderPosition.y + halfHeight;

         // --- Y-axis check (vertical separation) ---
         if (cylinderTop <= aabb.min.y || cylinderBottom >= aabb.max.y) {
             return { intersects: false }; // No overlap vertically
         }

         // --- XZ plane check (horizontal separation) ---
         // Find closest point on AABB rectangle in XZ plane to cylinder center
         const closestX = Math.max(aabb.min.x, Math.min(cylinderPosition.x, aabb.max.x));
         const closestZ = Math.max(aabb.min.z, Math.min(cylinderPosition.z, aabb.max.z));

         // Calculate squared distance from cylinder center (in XZ) to this closest point
         const dx = cylinderPosition.x - closestX;
         const dz = cylinderPosition.z - closestZ;
         const distanceSqXZ = dx * dx + dz * dz;

         // If distance^2 is greater than radius^2, they might not intersect,
         // unless the cylinder center is *inside* the AABB footprint.
         if (distanceSqXZ >= cylinderRadius * cylinderRadius) {
              // Check if cylinder center is within the XZ bounds of the AABB
              const centerInsideFootprint = cylinderPosition.x >= aabb.min.x && cylinderPosition.x <= aabb.max.x &&
                                             cylinderPosition.z >= aabb.min.z && cylinderPosition.z <= aabb.max.z;
             // If center is outside footprint AND distance > radius, no intersection
             if (!centerInsideFootprint) {
                  return { intersects: false };
             }
             // If center is inside footprint, but distance check failed (shouldn't happen here?),
             // it means the closest point was one of the AABB corners, but cylinder still overlaps vertically.
             // Proceed to overlap calculation below.
         }

         // --- Calculate Penetration (MTV - Minimum Translation Vector) ---
         // If we reach here, there's an overlap on both Y and XZ plane projections.

         aabb.getCenter(_aabbCenter);
         const aabbHalfWidth = (aabb.max.x - aabb.min.x) / 2;
         const aabbHalfHeight = (aabb.max.y - aabb.min.y) / 2;
         const aabbHalfDepth = (aabb.max.z - aabb.min.z) / 2;

         // Difference vector between centers
         const diffX = cylinderPosition.x - _aabbCenter.x;
         const diffY = cylinderPosition.y - _aabbCenter.y;
         const diffZ = cylinderPosition.z - _aabbCenter.z;

         // Calculate overlaps on each axis (treating cylinder as a box for simplicity here)
         // This is an approximation but works reasonably well for resolution.
         const overlapX = (cylinderRadius + aabbHalfWidth) - Math.abs(diffX);
         const overlapY = (halfHeight + aabbHalfHeight) - Math.abs(diffY);
         const overlapZ = (cylinderRadius + aabbHalfDepth) - Math.abs(diffZ);

         // If any overlap is non-positive, something is wrong (or they are just touching)
         // Add a small epsilon check here.
         if (overlapX <= this.collisionEpsilon || overlapY <= this.collisionEpsilon || overlapZ <= this.collisionEpsilon) {
             // Consider them non-intersecting if overlap is negligible
             // Might still need a small push if exactly touching is problematic
              return { intersects: false }; // Treat touching as non-intersecting for resolution
         }

         // Find the axis with the minimum overlap
         let minOverlap = overlapX;
         let normal = _tempVec3_1.set(Math.sign(diffX), 0, 0); // Reuse temp vector

         if (overlapY < minOverlap) {
             minOverlap = overlapY;
             normal.set(0, Math.sign(diffY), 0);
         }
         if (overlapZ < minOverlap) {
             minOverlap = overlapZ;
             normal.set(0, 0, Math.sign(diffZ));
         }

         // Ensure normal is valid (avoid zero vector if centers coincide)
          if (normal.lengthSq() < 0.0001) {
              // Arbitrarily choose X if centers are identical
               normal.set(1, 0, 0);
               minOverlap = overlapX; // Use the calculated overlapX
           }

         return {
             intersects: true,
             penetrationDepth: minOverlap,
             normal: normal.clone() // Return a clone, as _tempVec3_1 will be reused
         };
     }

    resolveCollisions(entity, collisions) {
        if (collisions.length === 0) {
            return;
        }

        // Optional: Sort collisions by penetration depth (deepest first)
        // collisions.sort((a, b) => b.penetrationDepth - a.penetrationDepth);

        collisions.forEach(collision => {
            const { normal, penetrationDepth } = collision;

            // Ignore negligible penetrations
            if (penetrationDepth < (this.collisionEpsilon / 10)) {
                return;
            }

            // --- Position Correction ---
            // Push the entity out along the collision normal by the penetration depth
            // Add a small epsilon bias to prevent re-colliding immediately
            const adjustment = _tempVec3_2.copy(normal).multiplyScalar(penetrationDepth + this.collisionEpsilon);
            entity.position.add(adjustment);

            // If the entity has a visual model (like player.human), update its position too
            // This check might need refinement depending on your entity structure
            if (entity.human && typeof entity.human.updatePosition === 'function') {
                // Warning: Directly setting visual position here can cause jitter.
                // It's often better to let the main loop lerp visualPosition towards physics position.
                // entity.human.updatePosition(entity.position);
            } else if (entity.mesh) {
                 // For creatures, ensure their mesh position reflects the physics position eventually
                 // (handled by lerping in main loop's animate function)
            }


            // --- Velocity Correction ---
            // Calculate the component of velocity along the normal
            const velocityAlongNormal = entity.velocity.dot(normal);

            // If velocity is heading into the collision surface
            if (velocityAlongNormal < 0) {
                // Apply restitution (bounciness) - 0.0 means no bounce
                const restitution = 0.0;
                // Calculate the impulse needed to stop movement into the surface (and optionally bounce back)
                const velocityAdjustment = normal.clone().multiplyScalar(-velocityAlongNormal * (1 + restitution));
                entity.velocity.add(velocityAdjustment);

                // --- Ground Check Update ---
                // If the collision normal is mostly pointing upwards, consider the entity grounded
                 // Use a threshold slightly less than 1 to account for slight slopes
                if (normal.y > 0.7) { // Threshold for considering it "ground"
                    // Check if entity was moving downwards or stationary vertically before collision resolution
                    // This helps prevent sticking to ceilings
                    if (velocityAlongNormal <= 0.01) { // Check original velocity component before adjustment
                       entity.onGround = true;
                       // If grounded, potentially reset coyote timer or handle jump buffer consumption here if needed immediately
                    }
                }
            }
        });
    }

    // Calculates the world-space AABB for a block at given grid coordinates
    getBlockAABB(blockCoords, world) {
        // Use the pre-calculated offset from the world object
        const centerOffset = world.worldCenterOffset;

        // Calculate the world coordinates of the block's CENTER
        // Based on BaseWorld.generateMeshes: setPosition(x - offset, y, z - offset)
        // where x, y, z are grid coordinates. This sets the center of the 1x1x1 BoxGeometry.
        const blockCenterX = blockCoords.i - centerOffset;
        const blockCenterY = blockCoords.j; // Y grid coordinate is used directly for center Y
        const blockCenterZ = blockCoords.k - centerOffset;

        // The AABB is centered at (blockCenterX, blockCenterY, blockCenterZ)
        // with half-extents of 0.5 in each direction (for a 1x1x1 block).
        return new THREE.Box3(
            _tempVec3_1.set(blockCenterX - 0.5, blockCenterY - 0.5, blockCenterZ - 0.5), // Reuse temp vector for min
            _tempVec3_2.set(blockCenterX + 0.5, blockCenterY + 0.5, blockCenterZ + 0.5)  // Reuse temp vector for max
        ).clone(); // Return a clone
    }


    checkGround(entity, world, meshesToExclude) {
         if (!entity || !entity.position || !world || !world.instancedMeshes) return; // Safety check

         const halfHeight = entity.cylinderHeight / 2;
         // Raycast origin: Center of the cylinder's bottom face
         this._groundCheckRayOrigin.copy(entity.position);
         this._groundCheckRayOrigin.y -= halfHeight; // Move to bottom center

         // Raycaster setup (reusing objects)
         this._groundCheckRaycaster.set(this._groundCheckRayOrigin, this._groundCheckDownVector);

         // Target objects: Only the instanced meshes representing the world geometry
         const collidableObjects = world.instancedMeshes;

         // Tune ray parameters:
         // `far` distance: How far down to check for ground. Needs to be slightly more than the physics step distance + epsilon.
         // 0.25 is likely sufficient unless entities move very fast vertically per physics step.
         this._groundCheckRaycaster.far = 0.25; // Increased slightly from 0.05 for more reliability

         // Perform the raycast
         const intersects = this._groundCheckRaycaster.intersectObjects(collidableObjects, false); // Non-recursive check

         // Filter out excluded meshes (though raycasting instanced meshes might not return individual instances this way easily)
         // For instanced mesh, intersects often returns the whole mesh. A more robust check might involve checking
         // the instanceId if available and comparing it to excluded entity IDs, but that's complex.
         // The primary check is the distance.

         let hitGround = false;
         if (intersects.length > 0) {
             // Check the distance to the closest intersection
             // A small tolerance is needed to account for floating point errors and penetration.
             const groundTolerance = 0.1; // Allow slightly more distance due to physics steps/penetration
             if (intersects[0].distance <= groundTolerance) {
                 hitGround = true;
             }
         }

         // Update entity's onGround status
         entity.onGround = hitGround;

         // If the entity is determined to be on the ground and still has downward velocity, zero it out.
         if (entity.onGround && entity.velocity.y < 0) {
             entity.velocity.y = 0;
         }
     }

     // --- NEW: Method to handle fireball collisions with the player ---
     handleFireballCollisions(player) {
        if (!player || player.isDead || !this.activeFireballs || this.activeFireballs.length === 0) {
            return; // No player, player dead, or no fireballs to check
        }

        const playerHalfHeight = player.cylinderHeight / 2;
        const playerBottom = player.position.y - playerHalfHeight;
        const playerTop = player.position.y + playerHalfHeight;
        _playerCheckPos.copy(player.position); // Use temporary vector for player position

        // Iterate backwards for safe removal
        for (let i = this.activeFireballs.length - 1; i >= 0; i--) {
            const fireball = this.activeFireballs[i];
            if (!fireball || fireball.toBeRemoved || !fireball.mesh) continue; // Skip if invalid or already marked

            _fireballCheckPos.copy(fireball.mesh.position); // Use temporary vector for fireball position
            const fireballRadius = fireball.mesh.geometry.parameters.radius || 0.5; // Get radius from geometry

            // --- Simple Sphere vs Cylinder Check ---

            // 1. Vertical Overlap Check
            const fireballBottom = _fireballCheckPos.y - fireballRadius;
            const fireballTop = _fireballCheckPos.y + fireballRadius;
            if (fireballTop < playerBottom || fireballBottom > playerTop) {
                continue; // No vertical overlap
            }

            // 2. Horizontal Distance Check (XZ plane)
            const dx = _playerCheckPos.x - _fireballCheckPos.x;
            const dz = _playerCheckPos.z - _fireballCheckPos.z;
            const distanceSqXZ = dx * dx + dz * dz;
            const combinedRadius = player.cylinderRadius + fireballRadius;

            if (distanceSqXZ <= combinedRadius * combinedRadius) {
                // --- Collision Detected! ---
                console.log("Fireball hit player!");
                player.takeDamage(fireball.damage); // Apply damage

                // Mark fireball for removal and remove immediately from physics check
                fireball.toBeRemoved = true;
                this.activeFireballs.splice(i, 1); // Remove from the array being checked by physics

                // TODO: Play fireball impact sound
                // player.audioManager.playSound('fireball_impact');

                // Optional: Apply knockback to player (add impulse away from fireball)
                // const knockbackStrength = 5.0;
                // const knockbackDir = _tempVec3_1.set(dx, 0, dz).normalize();
                // player.velocity.addScaledVector(knockbackDir, knockbackStrength);
                // player.onGround = false; // Knockback might lift player

            }
        }
    }

}
// --- END OF FILE physics.js ---
```

---

**Summary of Changes:**

1.  **`Fireball.js`:** New file defining the fireball projectile.
2.  **`Player.js`:**
    *   Added `maxHealth`, `health`, `isDead` properties.
    *   Added `jumpSpeed` property.
    *   Added `takeDamage()` method.
    *   Added `resetState()` method to handle respawning.
    *   Modified jump logic in `onKeyDown` to use the jump buffer.
3.  **`main.js`:**
    *   Imported `Fireball`.
    *   Added `activeFireballs` array.
    *   Passed `activeFireballs` to `Physics` constructor.
    *   Added fireball update and cleanup loop in `animate()`.
    *   Added player death check in `animate()` which calls `loadMarsBase()` and `player.resetState()`.
    *   Modified `loadMarsBase` and `switchToMartianHunt` to use `player.resetState()` with the correct spawn position.
    *   Created `getMarsBasePlayerSpawnPosition` helper.
    *   Passed `player` reference to `ghast.updateLogic()`.
    *   Ensured `async/await` is used correctly for world loading on death/timer expiry.
    *   Added cleanup for `activeFireballs` in `cleanupCurrentWorld`.
4.  **`Ghast.js`:**
    *   Imported `Fireball`.
    *   Added attack parameters (`attackRange`, `fireballCooldown`, etc.).
    *   Added state variables (`attackCooldownTimer`, `currentTarget`, `isAttackingOrCoolingDown`).
    *   Modified `updateLogic` to accept `player` and `activeFireballs`, implement detection, territory check, cooldown, and call `fireFireball`.
    *   Added `fireFireball()` method to create and manage `Fireball` instances.
    *   Increased default `altitudeMin` in `spawnGhasts` (in `main.js`) to keep them further off the ground.
5.  **`GhastMovement.js`:**
    *   Added attack movement parameters (`minAttackDistance`, `maxAttackDistance`, `hoverSpeedMultiplier`).
    *   Modified `update` to accept `player`, check `ghast.isAttackingOrCoolingDown`, and call either `_calculateAttackTargetPosition` or `_calculateNewWanderTargetPosition`.
    *   Added `_calculateAttackTargetPosition` to handle movement relative to the player (maintain distance).
    *   Renamed `_calculateNewTargetPosition` to `_calculateNewWanderTargetPosition`.
    *   Increased default `altitudeMin` and preferred altitude range.
6.  **`physics.js`:**
    *   Added `activeFireballs` property and accepted it in the constructor.
    *   Added `handleFireballCollisions()` method to check for hits against the player.
    *   Called `handleFireballCollisions()` within the main `update()` loop after physics substeps.
    *   Modified player jump logic in `update` to check `coyoteTimer > 0` as well as `onGround`.

Remember to place the new `Fireball.js` file in your `src` directory alongside the other JavaScript files. Test thoroughly after applying these changes!
