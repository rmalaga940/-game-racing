<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Monaco Racing Game Mejorado</title>
  <style>
    html, body {
      margin: 0; padding: 0; overflow: hidden; height: 100%; background: #202020;
      font-family: Arial, sans-serif;
      user-select: none;
    }
    canvas {
      display: block;
    }
    #controls, #speedometer, #musicControl {
      position: absolute;
      background: rgba(0,0,0,0.6);
      color: white;
      padding: 10px;
      border-radius: 6px;
      z-index: 10;
      user-select: none;
    }
    #controls {
      top: 10px; left: 10px;
      max-width: 220px;
    }
    #speedometer {
      top: 10px; right: 10px;
      font-size: 18px;
      font-weight: bold;
    }
    #musicControl {
      bottom: 10px; left: 10px;
      cursor: pointer;
      font-weight: bold;
    }
  </style>
</head>
<body>

  <div id="controls">
    <p><strong>Controles:</strong></p>
    <p>W - Acelerar | S - Frenar</p>
    <p>A/D - Girar Izquierda/Derecha</p>
    <p>Espacio - Freno de mano</p>
  </div>

  <div id="speedometer">Velocidad: 0 km/h</div>

  <div id="musicControl">🎵 Pausar Música</div>

  <audio id="bgm" loop autoplay>
    <source src="https://cdn.pixabay.com/download/audio/2023/05/24/audio_aaf55f4bb8.mp3?filename=night-drive-147506.mp3" type="audio/mpeg" />
  </audio>

  <script type="module">
    import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.152.2/build/three.module.js';
    import { GLTFLoader } from 'https://cdn.jsdelivr.net/npm/three@0.152.2/examples/jsm/loaders/GLTFLoader.js';

    // Escena y cámara
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x202020);

    const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(0, 6, 12);

    // Renderizador
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.shadowMap.enabled = true;
    document.body.appendChild(renderer.domElement);

    // Luces
    const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
    directionalLight.position.set(10, 20, 10);
    directionalLight.castShadow = true;
    directionalLight.shadow.mapSize.width = 2048;
    directionalLight.shadow.mapSize.height = 2048;
    directionalLight.shadow.camera.near = 0.5;
    directionalLight.shadow.camera.far = 50;
    scene.add(directionalLight);

    const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
    scene.add(ambientLight);

    // Suelo y pista
    const groundGeo = new THREE.PlaneGeometry(250, 250);
    const groundMat = new THREE.MeshStandardMaterial({ color: 0x222222 });
    const ground = new THREE.Mesh(groundGeo, groundMat);
    ground.rotation.x = -Math.PI / 2;
    ground.receiveShadow = true;
    scene.add(ground);

    // Pista con curvas (trayecto simple con líneas y curvas)
    const trackPoints = [
      new THREE.Vector3(-60, 0.01, -90),
      new THREE.Vector3(-60, 0.01, -30),
      new THREE.Vector3(-40, 0.01, -10),
      new THREE.Vector3(20, 0.01, -10),
      new THREE.Vector3(60, 0.01, 10),
      new THREE.Vector3(60, 0.01, 60),
      new THREE.Vector3(0, 0.01, 90),
      new THREE.Vector3(-40, 0.01, 80),
      new THREE.Vector3(-60, 0.01, 60),
      new THREE.Vector3(-60, 0.01, -90), // Cerramos pista
    ];

    const trackMaterial = new THREE.LineBasicMaterial({ color: 0xffffff, linewidth: 4 });
    const trackGeometry = new THREE.BufferGeometry().setFromPoints(trackPoints);
    const trackLine = new THREE.Line(trackGeometry, trackMaterial);
    scene.add(trackLine);

    // Bordes de pista para colisiones
    const borderMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000, opacity: 0.6, transparent: true });
    const borders = [];

    for (let i = 0; i < trackPoints.length - 1; i++) {
      const p1 = trackPoints[i];
      const p2 = trackPoints[i + 1];
      const mid = new THREE.Vector3().addVectors(p1, p2).multiplyScalar(0.5);
      const length = p1.distanceTo(p2);
      const angle = Math.atan2(p2.z - p1.z, p2.x - p1.x);

      const borderWidth = 5;
      const borderHeight = 1;

      // Borde izquierdo
      const leftBorder = new THREE.Mesh(new THREE.BoxGeometry(length, borderHeight, borderWidth), borderMaterial);
      leftBorder.position.copy(mid);
      leftBorder.position.x += Math.sin(angle) * (borderWidth / 2);
      leftBorder.position.z -= Math.cos(angle) * (borderWidth / 2);
      leftBorder.position.y = borderHeight / 2;
      leftBorder.rotation.y = -angle;
      leftBorder.receiveShadow = true;
      scene.add(leftBorder);
      borders.push(leftBorder);

      // Borde derecho
      const rightBorder = new THREE.Mesh(new THREE.BoxGeometry(length, borderHeight, borderWidth), borderMaterial);
      rightBorder.position.copy(mid);
      rightBorder.position.x -= Math.sin(angle) * (borderWidth / 2);
      rightBorder.position.z += Math.cos(angle) * (borderWidth / 2);
      rightBorder.position.y = borderHeight / 2;
      rightBorder.rotation.y = -angle;
      rightBorder.receiveShadow = true;
      scene.add(rightBorder);
      borders.push(rightBorder);
    }

    // Cargar modelo de auto F1
    let car;
    const loader = new GLTFLoader();
    loader.load(
      'https://raw.githubusercontent.com/KhronosGroup/glTF-Sample-Models/master/2.0/F1Car/glTF/F1Car.gltf',
      (gltf) => {
        car = gltf.scene;
        car.scale.set(0.5, 0.5, 0.5);
        car.position.set(-60, 0, -90);
        car.castShadow = true;
        scene.add(car);
      },
      undefined,
      (error) => {
        console.error('Error cargando modelo:', error);
      }
    );

    // Variables físicas y controles
    let speed = 0;
    let angle = 0;
    const keys = {};

    window.addEventListener('keydown', (e) => { keys[e.key.toLowerCase()] = true; });
    window.addEventListener('keyup', (e) => { keys[e.key.toLowerCase()] = false; });

    // Detección simple de colisiones con bounding boxes
    function checkCollision(object, obstacles) {
      if (!object) return false;
      object.updateMatrixWorld();
      const box = new THREE.Box3().setFromObject(object);
      for (let obs of obstacles) {
        const obsBox = new THREE.Box3().setFromObject(obs);
        if (box.intersectsBox(obsBox)) return true;
      }
      return false;
    }

    // Velocímetro
    const speedometer = document.getElementById('speedometer');

    // Control de música
    const musicControl = document.getElementById('musicControl');
    const bgm = document.getElementById('bgm');
    musicControl.onclick = () => {
      if (bgm.paused) {
        bgm.play();
        musicControl.textContent = '🎵 Pausar Música';
      } else {
        bgm.pause();
        musicControl.textContent = '🎵 Reproducir Música';
      }
    };

    // Cámara con seguimiento suave
    function updateCamera() {
      if (!car) return;
      const carPos = car.position;
      const offset = new THREE.Vector3(0, 5, 12);

      const rotatedOffset = offset.clone();
      rotatedOffset.applyAxisAngle(new THREE.Vector3(0, 1, 0), angle);

      const desiredPos = new THREE.Vector3().addVectors(carPos, rotatedOffset);

      camera.position.lerp(desiredPos, 0.1);
      camera.lookAt(carPos.x, carPos.y + 1, carPos.z);
    }

    // Animación y física
    function animate() {
      requestAnimationFrame(animate);
      if (!car) return;

      const acceleration = 0.03;
      const maxSpeed = 1.5;
      const friction = 0.96;
      const turnSpeed = 0.04;

      // Acelerar/frenar
      if (keys['w']) speed = Math.min(maxSpeed, speed + acceleration);
      else if (keys['s']) speed = Math.max(-maxSpeed / 2, speed - acceleration);
      else speed *= friction;

      // Freno de mano
      if (keys[' ']) speed *= 0.7;

      // Giro sólo si está en movimiento
      if (Math.abs(speed) > 0.02) {
        if (keys['a']) angle += turnSpeed * (speed / maxSpeed);
        if (keys['d']) angle -= turnSpeed * (speed / maxSpeed);
      }

      // Posición tentativa
      const nextX = car.position.x - Math.sin(angle) * speed;
      const nextZ = car.position.z - Math.cos(angle) * speed;

      // Guardar posición actual
      const currentPos = car.position.clone();

      // Mover auto
      car.position.x = nextX;
      car.position.z = nextZ;
      car.rotation.y = angle;

      // Colisión con bordes
      if (checkCollision(car, borders)) {
        car.position.copy(currentPos);
        speed *= -0.3;
      }

      // Actualizar velocímetro
      const kmh = Math.round(speed * 100);
      speedometer.textContent = `Velocidad: ${Math.abs(kmh)} km/h`;

      updateCamera();

      renderer.render(scene, camera);
    }

    animate();

    // Ajuste al redimensionar ventana
    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
  </script>
</body>
</html>
