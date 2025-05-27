<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Gravity's Edge - Improved</title>
  <style>
    html, body { margin: 0; overflow: hidden; background: #87ceeb; }
    canvas { display: block; }
    #instructions {
      position: absolute;
      color: white;
      font-family: sans-serif;
      text-align: center;
      top: 0;
      width: 100%;
      padding: 1em;
      background: rgba(0,0,0,0.7);
      z-index: 1;
    }
    #crosshair {
      position: absolute;
      left: 50%;
      top: 50%;
      width: 4px;
      height: 4px;
      margin-left: -2px;
      margin-top: -2px;
      background: white;
      border-radius: 50%;
      z-index: 2;
    }
    #throwBar {
      position: absolute;
      bottom: 52%;
      left: 50%;
      width: 100px;
      height: 6px;
      margin-left: -50px;
      background: rgba(255, 255, 255, 0.1);
      border: 1px solid white;
      display: none;
      z-index: 2;
    }
    #throwBarFill {
      height: 100%;
      width: 0%;
      background: linear-gradient(to right, lime, yellow, red);
      transition: width 0.05s;
    }
    #prompt {
      position: absolute;
      bottom: 20%;
      width: 100%;
      text-align: center;
      color: white;
      font-family: sans-serif;
      font-size: 18px;
      display: none;
      z-index: 2;
    }
  </style>
</head>
<body>
  <div id="instructions">Click to play | WASD = move | Shift = run | Space = jump | E = grab/release crate</div>
  <div id="crosshair"></div>
  <div id="throwBar"><div id="throwBarFill"></div></div>
  <div id="prompt">Press E to grab/release</div>
  <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>
  <script type="module">
    import * as CANNON from 'https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/dist/cannon-es.js';

    let scene, camera, renderer, world;
    let playerBody, playerMesh, pitchObject;
    let crateBodies = [], crateMeshes = [], heldCrate = null;
    let keys = {}, yaw = 0, pitch = 0;
    let grabbing = false, throwPower = 0, chargingThrow = false;
    let controlsEnabled = false;
    let canJump = true, jumpQueued = false;
    let highlightedCrate = null;

    init();

    function init() {
      scene = new THREE.Scene();
      scene.background = new THREE.Color('#87ceeb');

      const skyGeo = new THREE.SphereGeometry(500, 32, 32);
      const skyMat = new THREE.MeshBasicMaterial({ color: '#87ceeb', side: THREE.BackSide });
      scene.add(new THREE.Mesh(skyGeo, skyMat));

      camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.1, 1000);
      renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(innerWidth, innerHeight);
      document.body.appendChild(renderer.domElement);

      scene.add(new THREE.AmbientLight(0xffffff, 0.5));
      const sun = new THREE.DirectionalLight(0xffffff, 1);
      sun.position.set(100, 100, 100);
      scene.add(sun);

      world = new CANNON.World({ gravity: new CANNON.Vec3(0, -9.82, 0) });
      const groundShape = new CANNON.Plane();
      const groundBody = new CANNON.Body({ mass: 0 });
      groundBody.addShape(groundShape);
      groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
      world.addBody(groundBody);

      const floor = new THREE.Mesh(
        new THREE.PlaneGeometry(100, 100),
        new THREE.MeshLambertMaterial({ color: 0x228b22 })
      );
      floor.rotation.x = -Math.PI / 2;
      scene.add(floor);

      playerBody = new CANNON.Body({ mass: 5, shape: new CANNON.Sphere(0.5), position: new CANNON.Vec3(0, 2, 0), linearDamping: 0.9 });
      world.addBody(playerBody);

      playerMesh = new THREE.Object3D();
      pitchObject = new THREE.Object3D();
      pitchObject.add(camera);
      playerMesh.add(pitchObject);
      scene.add(playerMesh);

      for (let i = 0; i < 3; i++) {
        const body = new CANNON.Body({ mass: 1, shape: new CANNON.Box(new CANNON.Vec3(0.5, 0.5, 0.5)), position: new CANNON.Vec3(i * 2 - 2, 3, 0) });
        const mesh = new THREE.Mesh(new THREE.BoxGeometry(1, 1, 1), new THREE.MeshStandardMaterial({ color: 0xff9900 }));
        crateBodies.push(body);
        crateMeshes.push(mesh);
        world.addBody(body);
        scene.add(mesh);
      }

      document.addEventListener('keydown', e => {
        keys[e.key.toLowerCase()] = true;
        if (e.key === 'e') toggleGrab();
        if (e.code === 'Space') jumpQueued = true;
      });

      document.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);
      document.addEventListener('click', () => renderer.domElement.requestPointerLock());

      document.addEventListener('pointerlockchange', () => {
        controlsEnabled = document.pointerLockElement === renderer.domElement;
        document.getElementById('instructions').style.display = controlsEnabled ? 'none' : 'block';
      });

      document.addEventListener('mousedown', () => { if (grabbing) chargingThrow = true; });
      document.addEventListener('mouseup', () => {
        if (grabbing && heldCrate) {
          const shootDir = new THREE.Vector3();
          camera.getWorldDirection(shootDir);
          shootDir.normalize();
          const power = 10 + 20 * throwPower;
          heldCrate.velocity.set(shootDir.x * power, shootDir.y * power + 2, shootDir.z * power);
          heldCrate.angularVelocity.set((Math.random() - 0.5) * 5, (Math.random() - 0.5) * 5, (Math.random() - 0.5) * 5);
          grabbing = false;
          heldCrate = null;
          throwPower = 0;
          chargingThrow = false;
          document.getElementById('throwBar').style.display = 'none';
        }
      });

      document.addEventListener('mousemove', e => {
        if (!controlsEnabled) return;
        yaw -= e.movementX * 0.002;
        pitch -= e.movementY * 0.002;
        pitch = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, pitch));
        playerMesh.rotation.y = yaw;
        pitchObject.rotation.x = pitch;
      });

      animate();
    }

    function toggleGrab() {
      if (grabbing && heldCrate) {
        heldCrate = null;
        grabbing = false;
        document.getElementById('throwBar').style.display = 'none';
        return;
      }
      if (highlightedCrate !== null) {
        heldCrate = crateBodies[highlightedCrate];
        grabbing = true;
      }
    }

    function animate() {
      if (grabbing && heldCrate) {
        const camPos = new THREE.Vector3();
        camera.getWorldPosition(camPos);
        const forward = new THREE.Vector3();
        camera.getWorldDirection(forward);
        forward.normalize().multiplyScalar(1.5);
        heldCrate.position.set(camPos.x + forward.x, camPos.y + forward.y, camPos.z + forward.z);
        const targetQuat = new THREE.Quaternion();
        camera.getWorldQuaternion(targetQuat);
        heldCrate.quaternion.copy(targetQuat);
        heldCrate.angularVelocity.set(0, 0, 0);
        heldCrate.velocity.set(0, 0, 0);
        if (chargingThrow) {
          throwPower = Math.min(1, throwPower + 0.02);
          document.getElementById('throwBarFill').style.width = (throwPower * 100) + '%';
        }
        document.getElementById('throwBar').style.display = 'block';
      }

      requestAnimationFrame(animate);

      const speed = keys['shift'] ? 9 : 5;
      const forward = new THREE.Vector3(0, 0, -1).applyEuler(playerMesh.rotation);
      const right = new THREE.Vector3(1, 0, 0).applyEuler(playerMesh.rotation);
      let move = new CANNON.Vec3();
      if (keys['w']) move = move.vadd(new CANNON.Vec3(forward.x, 0, forward.z));
      if (keys['s']) move = move.vsub(new CANNON.Vec3(forward.x, 0, forward.z));
      if (keys['a']) move = move.vsub(new CANNON.Vec3(right.x, 0, right.z));
      if (keys['d']) move = move.vadd(new CANNON.Vec3(right.x, 0, right.z));
      move.normalize();
      move.scale(speed, move);
      playerBody.velocity.x = move.x;
      playerBody.velocity.z = move.z;

      if (jumpQueued && canJump && Math.abs(playerBody.velocity.y) < 0.05) {
        playerBody.velocity.y = 6;
        canJump = false;
      }
      jumpQueued = false;

      world.step(1 / 60);
      playerMesh.position.copy(playerBody.position);
      if (Math.abs(playerBody.velocity.y) < 0.1) canJump = true;

      crateMeshes.forEach((m, i) => {
        m.position.copy(crateBodies[i].position);
        m.quaternion.copy(crateBodies[i].quaternion);
      });

      let bestCrate = null, bestDot = 0;
      const camDir = new THREE.Vector3();
      camera.getWorldDirection(camDir);
      const camPos = new THREE.Vector3();
      camera.getWorldPosition(camPos);

      crateBodies.forEach((crate, i) => {
        const toCrate = new THREE.Vector3().subVectors(crate.position, camPos);
        const distance = toCrate.length();
        if (distance < 2) {
          toCrate.normalize();
          const dot = camDir.dot(toCrate);
          if (dot > 0.95 && dot > bestDot) {
            bestCrate = i;
            bestDot = dot;
          }
        }
      });

      highlightedCrate = bestCrate;
      crateMeshes.forEach((m, i) => m.material.color.set(i === highlightedCrate ? 0xffff00 : 0xff9900));
      document.getElementById('prompt').style.display = (highlightedCrate !== null && !grabbing) ? 'block' : 'none';

      renderer.render(scene, camera);
    }
  </script>
</body>
</html>
