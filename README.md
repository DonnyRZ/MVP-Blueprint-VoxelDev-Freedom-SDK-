# MVP-Blueprint-VoxelDev-Freedom-SDK-

**I. Core Product: JavaScript/TypeScript SDK**

The goal here is to create a library that developers install (`npm install your-sdk`) and use within their own Node.js/TypeScript/JavaScript projects to build the core logic and structure of their voxel game. It leverages Three.js under the hood for rendering but provides a higher-level, game-focused API.

---

**A. Engine Core**

*   **Purpose:** To provide the basic setup, initialization, and the main heartbeat (game loop) for the game.
*   **Details:**
    *   **Initialization Function (`createGame`):**
        *   This would be the primary entry point for a developer using your SDK.
        *   It likely takes a configuration object, potentially including a reference to an HTML `<canvas>` element for rendering.
        *   It sets up the Three.js renderer, scene, camera basics, and hooks into the game loop.
        *   *Example Usage:*
            ```typescript
            import { createGame } from 'your-sdk';

            const canvasElement = document.getElementById('game-canvas');
            const game = createGame({
                canvas: canvasElement,
                // other initial config? e.g., physics settings?
            });

            // Load world, setup player, etc., using the 'game' object
            game.world.loadMap('path/to/map.json');
            // ...
            game.start(); // Start the game loop
            ```
    *   **Game Loop Management:**
        *   Provides a standard `requestAnimationFrame` loop.
        *   Manages timing (`deltaTime`).
        *   Orchestrates the order of operations each frame: process inputs -> update physics (multiple fixed steps if needed) -> update entity logic -> update animations -> render.
    *   **Event Emitter System:**
        *   The `game` object (or core sub-modules like `world`, `playerManager`) should implement a simple event emitter pattern (`on`, `off`, `emit`).
        *   This allows different parts of the SDK and the user's code to communicate without tight coupling.
        *   *Example Events:* `game.world.on('loaded', () => {...})`, `game.playerManager.on('playerJoined', (player) => {...})`.

---

**B. World System**

*   **Purpose:** To manage the game's environment, primarily the terrain blocks and their properties.
*   **Details:**
    *   **World Data Loading:**
        *   Needs a function like `world.loadMap(mapDataOrUrl)` which accepts a predefined JSON structure.
        *   This JSON should define:
            *   `blockTypes`: An array defining each unique block (ID, name, texture path(s), potentially basic physics properties like friction if not handled by colliders).
            *   `blocks`: A dictionary mapping "x,y,z" string coordinates to a block type ID.
        *   *Example map.json snippet:*
            ```json
            {
              "blockTypes": [
                { "id": 1, "name": "Dirt", "textureUri": "textures/dirt.png" },
                { "id": 2, "name": "Grass", "textureUri": "textures/grass_top.png", "textures": { "top": "grass_top.png", "side": "grass_side.png", "bottom": "dirt.png"} } // Example for different faces
              ],
              "blocks": {
                "0,0,0": 1,
                "1,0,0": 1,
                "0,1,0": 2
              }
            }
            ```
    *   **Block Type Management:**
        *   Internal registry to store the definitions loaded from the map or registered via code (`world.registerBlockType(...)`).
    *   **Efficient Voxel Rendering:**
        *   Crucially uses `THREE.InstancedMesh` per block type to render the entire world efficiently.
        *   Needs logic to update instances when blocks change (via `setBlock`).
        *   Handles texture mapping (potentially using a texture atlas generated from loaded block textures for optimization).
    *   **API:**
        *   `world.getBlock(x, y, z)`: Returns the block type ID or null at given coordinates.
        *   `world.setBlock(x, y, z, blockTypeId)`: Updates the block at coordinates, triggering rendering updates and physics updates (collision shape changes). Setting ID 0 might mean "air".

---

**C. Entity System**

*   **Purpose:** To manage dynamic objects in the world (players, NPCs, items, etc.).
*   **Details:**
    *   **Base `Entity` Class:**
        *   Has properties like `id`, `position` (Vector3), `rotation` (Quaternion).
        *   Maybe a `tick(deltaTime)` method for custom entity logic.
        *   Methods for adding/removing from the world (`entity.spawn(world, position?)`, `entity.despawn()`).
    *   **Visual Representation:**
        *   Constructor takes options like `modelUri` (path to a simple GLTF) or perhaps `blockDefinition` (for block-like entities, defining size and texture like a world block).
        *   Manages the corresponding `THREE.Object3D` (Mesh or Group).
    *   **Transform Controls:** Wraps Three.js transform operations.
        *   `entity.setPosition(x, y, z)` / `entity.setPosition(vector3)`.
        *   `entity.setRotation(quaternion)` / `entity.setRotationFromEuler(x, y, z)`.
        *   `entity.getPosition()`: Returns current position.
        *   `entity.getRotation()`: Returns current rotation.

---

**D. Core Voxel Physics (Robust Basics)**

*   **Purpose:** To handle movement, gravity, and collision for entities within the voxel world reliably.
*   **Details:**
    *   **Physics Engine Integration:** Integrate a library like Rapier.js or refine your own. This engine runs in parallel with the main game loop, potentially using fixed timesteps.
    *   **World Collision Shapes:** The physics engine needs static collision shapes representing the voxel grid loaded from `world.blocks`. This is often the most performance-critical part. (Rapier has heightfield or voxel shapes).
    *   **Character Controller Primitive:**
        *   A specialized physics object (often a Capsule shape in the physics engine) linked to a player/NPC entity.
        *   Handles interactions with the static world geometry (sliding along walls, stopping at obstacles).
        *   Includes configuration for step height, slope limits (maybe post-MVP).
        *   **API:**
            *   `characterController.move(directionVector)`: Tries to move the character, respecting collisions.
            *   `characterController.jump(jumpForce)`: Applies vertical impulse.
            *   `characterController.getPosition()`: Gets the physics-simulated position.
            *   `characterController.isOnGround()`: Returns boolean status.
    *   **Entity Linking:** The `Entity`'s visual position/rotation needs to be updated *from* the physics simulation results each frame/tick.
    *   **Collision Events:** The physics engine should emit events when colliders intersect. The SDK needs to translate these into user-friendly events tied to the `Entity` objects involved.
        *   *Example:* `entity.on('collision', (otherEntityOrBlockData) => { ... });`

---

**E. Animation Control (Basic)**

*   **Purpose:** Allow playing pre-defined animations embedded in GLTF models.
*   **Details:**
    *   Uses Three.js's AnimationMixer internally.
    *   **API:**
        *   `entity.playAnimation('run', true)`: Plays the animation named "run", loops it.
        *   `entity.playAnimation('attack', false)`: Plays "attack" once.
        *   `entity.stopAnimation('run')`: Stops the "run" animation.
        *   `entity.stopAllAnimations()`: Stops everything.

---

**F. Input System**

*   **Purpose:** Provide a unified way to handle different input methods (keyboard, touch).
*   **Details:**
    *   **Abstraction Layer:** Defines abstract actions (e.g., 'moveForward', 'jump', 'fire') and axes ('moveX', 'moveY', 'lookX', 'lookY').
    *   **Default Mappings:**
        *   Keyboard: W -> 'moveForward', Space -> 'jump', Mouse Movement -> 'lookX'/'lookY'.
        *   Touch: Predefined screen areas (e.g., left half for joystick, right half for camera look) mapped to axes. Define specific smaller areas for buttons mapped to actions ('jump', 'fire').
    *   **API:**
        *   `input.isActionPressed('jump')`: Returns true/false.
        *   `input.isActionJustPressed('jump')`: Returns true only on the frame it was first pressed.
        *   `input.getAxis('moveX')`: Returns a value (e.g., -1 to 1).
        *   The system listens to browser events (keydown, pointermove, touchstart, etc.) and updates the internal state.

---

**G. Camera System**

*   **Purpose:** Manage the viewpoint for the player.
*   **Details:**
    *   Manages the main `THREE.PerspectiveCamera`.
    *   **Modes:**
        *   `camera.setMode('firstPerson')`: Locks camera rotation to an entity, offsets position slightly forward/up.
        *   `camera.setMode('thirdPerson', { distance: 5, height: 2 })`: Places camera behind and slightly above an entity.
    *   **Targeting:** `camera.followEntity(entity)`. The camera updates its position based on the target entity's position each frame.

---

**H. Build System (Web Output)**

*   **Purpose:** Make it easy for developers to go from their SDK-based code to a runnable web game.
*   **Details:**
    *   Likely involves providing a pre-configured build tool setup (e.g., Vite).
    *   Developer runs a command like `npm run build`.
    *   Output is a `/dist` folder containing:
        *   `index.html` (with a `<canvas>` element).
        *   Bundled `game.js` (user's code + SDK code).
        *   Copied `assets` folder (textures, models, maps).
    *   This `/dist` folder can then be served by any static web server.

---

This detailed breakdown covers the *what* and *why* of each component in the core SDK for the MVP. The examples show how a developer might interact with it. Remember, implementation details (like choosing the exact physics library or build tool) are still open, but this defines the necessary capabilities and APIs for the user.
