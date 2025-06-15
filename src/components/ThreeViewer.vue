<template>
    <canvas ref="canvas" class="webgl"></canvas>
</template>

<script setup>
import { onMounted, ref } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

const canvas = ref(null);

onMounted(() => {
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0xeeeeee);

    // Camera — zoomed in closer
    const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(0, 0.25, 1.5);

    const renderer = new THREE.WebGLRenderer({ canvas: canvas.value, antialias: true, alpha: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);

    // ✅ Enable Shadows
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.autoRotate = true;
    controls.autoRotateSpeed = 2;
    controls.enableDamping = true;
    controls.enableZoom = true;
    controls.minPolarAngle = Math.PI / 2.5;
    controls.maxPolarAngle = Math.PI / 2.5;

    // Lighting
    scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 0.7));

    const dirLight = new THREE.DirectionalLight(0xffffff, 2);
    dirLight.position.set(5, 10, 7.5);
    dirLight.castShadow = true;
    dirLight.shadow.mapSize.width = 1024;
    dirLight.shadow.mapSize.height = 1024;
    dirLight.shadow.camera.near = 0.5;
    dirLight.shadow.camera.far = 50;
    scene.add(dirLight);

    scene.add(new THREE.AmbientLight(0xffffff, 0.5));

    // Environment map
    const envMap = new THREE.CubeTextureLoader().load([
        '/env/px.jpg', '/env/nx.jpg',
        '/env/py.jpg', '/env/ny.jpg',
        '/env/pz.jpg', '/env/nz.jpg'
    ]);
    scene.environment = envMap;

    // Procedural Marble Texture
    function generateMarbleTexture(size = 256, veins = 30) {
        const canvas = document.createElement('canvas');
        canvas.width = canvas.height = size;
        const ctx = canvas.getContext('2d');

        ctx.fillStyle = '#222';
        ctx.fillRect(0, 0, size, size);

        for (let i = 0; i < veins; i++) {
            ctx.beginPath();
            ctx.strokeStyle = `rgba(255, 255, 255, ${Math.random() * 0.4 + 0.2})`;
            ctx.lineWidth = Math.random() * 2 + 1;
            const startX = Math.random() * size;
            const startY = Math.random() * size;
            const endX = startX + (Math.random() - 0.5) * size;
            const endY = startY + (Math.random() - 0.5) * size;
            ctx.moveTo(startX, startY);
            ctx.lineTo(endX, endY);
            ctx.stroke();
        }

        const texture = new THREE.CanvasTexture(canvas);
        texture.wrapS = texture.wrapT = THREE.RepeatWrapping;
        texture.repeat.set(4, 4);
        return texture;
    }

    const marbleTexture = generateMarbleTexture();
    const bumpTexture = generateMarbleTexture();
    bumpTexture.needsUpdate = true;

    // ✅ Ground plane for shadows
    const groundGeo = new THREE.PlaneGeometry(10, 10);
    const groundMat = new THREE.ShadowMaterial({ opacity: 0.3 });
    const ground = new THREE.Mesh(groundGeo, groundMat);
    ground.rotation.x = -Math.PI / 2;
    ground.position.y = -0.5;
    ground.receiveShadow = true;
    scene.add(ground);

    // Load model
    const loader = new GLTFLoader();
    loader.load('/models/model1.glb', (gltf) => {
        const model = gltf.scene;

        // Scale & center
        const box = new THREE.Box3().setFromObject(model);
        const size = new THREE.Vector3();
        box.getSize(size);
        const scaleFactor = 2 / size.length();
        model.scale.setScalar(scaleFactor);
        const center = box.getCenter(new THREE.Vector3());


        model.position.sub(center);
        model.position.y += -0.25;

        // Fix all materials & enable shadows
        model.traverse((child) => {
            if (!child.isMesh) return;

            child.castShadow = true;
            child.receiveShadow = true;

            const mat = child.material;
            if (!mat || !mat.isMeshStandardMaterial) {
                child.material = new THREE.MeshStandardMaterial({
                    color: 0xffffff,
                    metalness: 0.2,
                    roughness: 0.3,
                    envMap,
                    envMapIntensity: 1.0
                });
            } else {
                mat.envMap = envMap;
                mat.envMapIntensity = 0.8;
                mat.roughness = mat.roughness ?? 0.3;
                mat.metalness = mat.metalness ?? 0.2;
                mat.needsUpdate = true;
            }
        });

        // Marble sphere
        const sphere = model.getObjectByName('Sphere');
        if (sphere && sphere.isMesh) {
            sphere.material = new THREE.MeshStandardMaterial({
                map: marbleTexture,
                bumpMap: bumpTexture,
                bumpScale: 0.05,
                roughness: 0.4,
                metalness: 0.2,
                envMap,
                envMapIntensity: 1.0
            });
            sphere.castShadow = true;
            sphere.receiveShadow = true;
        }

        scene.add(model);
    }, undefined, (err) => {
        console.error('Failed to load model:', err);
    });

    // Animate
    const animate = () => {
        requestAnimationFrame(animate);
        controls.update();
        renderer.render(scene, camera);
    };
    animate();

    // Resize
    window.addEventListener('resize', () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    });
});
</script>

<style scoped>
.webgl {
    width: 100vw;
    height: 100vh;
    display: block;
    outline: none;
}
</style>
