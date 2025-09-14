FILE MANAGER



import { useEffect, useRef, useState } from 'react';
import * as THREE from 'three';
import useFileSystem from '../hooks/useFileSystem';

const EnhancedNeuralFileSystem = () => {
  const mountRef = useRef(null);
  const sceneRef = useRef(null);
  const rendererRef = useRef(null);
  const cameraRef = useRef(null);
  const headRef = useRef(null);
  const neuralRaysRef = useRef([]);
  const thumbnailsRef = useRef([]);
  const thumbnailGroupRef = useRef(null);
  const mouseRef = useRef(new THREE.Vector2());
  const raycasterRef = useRef(new THREE.Raycaster());
  const [selectedFile, setSelectedFile] = useState(null);
  const [hoveredFile, setHoveredFile] = useState(null);
  
  const { files, loading, error } = useFileSystem();

  useEffect(() => {
    if (!mountRef.current) return;

    // Scene setup
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x000011);
    sceneRef.current = scene;

    // Camera setup
    const camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
      0.1,
      1000
    );
    camera.position.z = 8;
    cameraRef.current = camera;

    // Renderer setup
    const renderer = new THREE.WebGLRenderer({ 
      antialias: false, // Disable for mobile performance
      powerPreference: "high-performance",
      alpha: true
    });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5)); // Limit pixel ratio for mobile
    mountRef.current.appendChild(renderer.domElement);
    rendererRef.current = renderer;

    // Create thumbnail group for better organization
    const thumbnailGroup = new THREE.Group();
    scene.add(thumbnailGroup);
    thumbnailGroupRef.current = thumbnailGroup;

    // Create scene elements
    createEnhancedStarfield(scene);
    createEnhancedWireframeHead(scene);
    createDynamicNeuralRays(scene);

    // Animation loop
    const animate = () => {
      requestAnimationFrame(animate);
      
      // Gentle head rotation
      if (headRef.current) {
        headRef.current.rotation.y += 0.002;
        headRef.current.rotation.x = Math.sin(Date.now() * 0.0005) * 0.1;
      }

      // Animate dynamic neural rays
      animateDynamicRays();

      // Animate enhanced thumbnails
      animateEnhancedThumbnails();

      renderer.render(scene, camera);
    };

    animate();
// Event handlers
    const handleResize = () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    };

    const handleMouseMove = (event) => {
      let clientX, clientY;
      if (event.touches && event.touches.length > 0) {
        clientX = event.touches[0].clientX;
        clientY = event.touches[0].clientY;
      } else {
        clientX = event.clientX;
        clientY = event.clientY;
      }

      mouseRef.current.x = (clientX / window.innerWidth) * 2 - 1;
      mouseRef.current.y = -(clientY / window.innerHeight) * 2 + 1;
      
      // Smooth camera movement
      const targetX = mouseRef.current.x * 0.3;
      const targetY = mouseRef.current.y * 0.2;
      camera.position.x += (targetX - camera.position.x) * 0.05;
      camera.position.y += (targetY - camera.position.y) * 0.05;
      camera.lookAt(0, 0, 0);

      // Check for hover interactions
      raycasterRef.current.setFromCamera(mouseRef.current, camera);
      const thumbnailMeshes = thumbnailsRef.current.map(t => t.mesh);
      const intersects = raycasterRef.current.intersectObjects(thumbnailMeshes);
      
      // Reset all thumbnails
      thumbnailsRef.current.forEach(thumbnail => {
        thumbnail.mesh.scale.set(1, 1, 1);
        thumbnail.mesh.material.emissive.setHex(0x000000);
      });
      
      // Highlight hovered thumbnail
      if (intersects.length > 0) {
        const hoveredThumbnail = intersects[0].object;
        hoveredThumbnail.scale.set(1.3, 1.3, 1.3);
        hoveredThumbnail.material.emissive.setHex(0x222222);
        setHoveredFile(hoveredThumbnail.userData.file);
        
        // Intensify rays towards hovered thumbnail
        intensifyRaysToTarget(hoveredThumbnail.position);
      } else {
        setHoveredFile(null);
      }
    };

    const handleMouseClick = (event) => {
      let clientX, clientY;
      if (event.touches && event.touches.length > 0) {
        clientX = event.touches[0].clientX;
        clientY = event.touches[0].clientY;
      } else {
        clientX = event.clientX;
        clientY = event.clientY;
      }

      mouseRef.current.x = (clientX / window.innerWidth) * 2 - 1;
      mouseRef.current.y = -(clientY / window.innerHeight) * 2 + 1;
      
      raycasterRef.current.setFromCamera(mouseRef.current, camera);
      const thumbnailMeshes = thumbnailsRef.current.map(t => t.mesh);
      const intersects = raycasterRef.current.intersectObjects(thumbnailMeshes);
      
      if (intersects.length > 0) {
        const clickedThumbnail = intersects[0].object;
        const fileData = clickedThumbnail.userData.file;
        
        // Enhanced selection effect
        createSelectionRipple(clickedThumbnail.position);
        setSelectedFile(fileData);
        console.log("Selected file:", fileData);
      }
    };

    window.addEventListener("resize", handleResize);
    window.addEventListener("mousemove", handleMouseMove);
    window.addEventListener("click", handleMouseClick);
    window.addEventListener("touchmove", handleMouseMove, { passive: false });
    window.addEventListener("touchstart", handleMouseClick, { passive: false });

    // Cleanup function
    return () => {
      window.removeEventListener("resize", handleResize);
      window.removeEventListener("mousemove", handleMouseMove);
      window.removeEventListener("click", handleMouseClick);
      window.removeEventListener("touchmove", handleMouseMove);
      window.removeEventListener("touchstart", handleMouseClick);
      
      // Thorough cleanup
      scene.traverse((object) => {
        if (object.geometry) object.geometry.dispose();
        if (object.material) {
          if (Array.isArray(object.material)) {
            object.material.forEach(material => material.dispose());
          } else {
            object.material.dispose();
          }
        }
      });
      
      if (mountRef.current && renderer.domElement) {
        mountRef.current.removeChild(renderer.domElement);
      }
      renderer.dispose();
    };
  }, []);

  const createEnhancedStarfield = (scene) => {
    const starsGeometry = new THREE.BufferGeometry();
    const starsMaterial = new THREE.PointsMaterial({
      color: 0xffffff,
      size: 1.5,
      sizeAttenuation: false,
      transparent: true,
      opacity: 0.8
    });

    const starsVertices = [];
    const starsColors = [];
    
    for (let i = 0; i < 800; i++) { // Reduced from 1500 for mobile
      const x = (Math.random() - 0.5) * 300;
      const y = (Math.random() - 0.5) * 300;
      const z = (Math.random() - 0.5) * 300;
      starsVertices.push(x, y, z);
      
      // Add subtle color variation
      const color = new THREE.Color();
      color.setHSL(0.6 + Math.random() * 0.2, 0.3, 0.8 + Math.random() * 0.2);
      starsColors.push(color.r, color.g, color.b);
    }

    starsGeometry.setAttribute('position', new THREE.Float32BufferAttribute(starsVertices, 3));
    starsGeometry.setAttribute('color', new THREE.Float32BufferAttribute(starsColors, 3));
    
    starsMaterial.vertexColors = true;
    const starField = new THREE.Points(starsGeometry, starsMaterial);
    scene.add(starField);
  };

  const createEnhancedWireframeHead = (scene) => {
    const headGeometry = new THREE.SphereGeometry(2.2, 20, 20);
    
    const headMaterial = new THREE.MeshBasicMaterial({
      color: 0x00aaff,
      wireframe: true,
      transparent: true,
      opacity: 0.9
    });

    const head = new THREE.Mesh(headGeometry, headMaterial);
    
    // Add subtle inner glow
    const innerGlowGeometry = new THREE.SphereGeometry(2.1, 16, 16);
    const innerGlowMaterial = new THREE.MeshBasicMaterial({
      color: 0x0066cc,
      transparent: true,
      opacity: 0.1,
      side: THREE.BackSide
    });
    const innerGlow = new THREE.Mesh(innerGlowGeometry, innerGlowMaterial);
    
    const headGroup = new THREE.Group();
    headGroup.add(head);
    headGroup.add(innerGlow);
    
    scene.add(headGroup);
    headRef.current = headGroup;
  };

  const createDynamicNeuralRays = (scene) => {
    const rayCount = 10; // Reduced from 15 for mobile
    const rays = [];

    for (let i = 0; i < rayCount; i++) {
      // Create more sophisticated ray geometry
      const rayGeometry = new THREE.BufferGeometry();
      const rayMaterial = new THREE.LineBasicMaterial({
        color: new THREE.Color().setHSL(0.55 + Math.random() * 0.1, 0.8, 0.6),
        transparent: true,
        opacity: 0.6
      });

      const angle = (i / rayCount) * Math.PI * 2;
      const startPoint = new THREE.Vector3(0, 0, 0);
      const endPoint = new THREE.Vector3(
        Math.cos(angle) * 5,
        Math.sin(angle) * 5,
        (Math.random() - 0.5) * 3
      );

      rayGeometry.setFromPoints([startPoint, endPoint]);
      const ray = new THREE.Line(rayGeometry, rayMaterial);
      
      rays.push({
        line: ray,
        originalEnd: endPoint.clone(),
        phase: i * 0.4,
        baseOpacity: 0.3 + Math.random() * 0.3,
        pulseSpeed: 0.02 + Math.random() * 0.02
      });
      
      scene.add(ray);
    }

    neuralRaysRef.current = rays;
  };

  const createEnhancedThumbnails = (scene) => {
    // Clear existing thumbnails
    if (thumbnailGroupRef.current) {
      thumbnailGroupRef.current.clear();
    }
    thumbnailsRef.current = [];

    if (!files || files.length === 0) return;

    const thumbnails = [];
    const maxThumbnails = Math.min(files.length, 6); // Reduced from 10 for mobile

    for (let i = 0; i < maxThumbnails; i++) {
      const file = files[i];
      
      // Enhanced geometry based on file type
      let geometry;
      switch (file.file_type) {
        case 'image':
          geometry = new THREE.PlaneGeometry(0.6, 0.4, 2, 2);
          break;
        case 'video':
          geometry = new THREE.BoxGeometry(0.5, 0.3, 0.2);
          break;
        case 'document':
          geometry = new THREE.PlaneGeometry(0.4, 0.6, 1, 1);
          break;
        case 'folder':
          geometry = new THREE.BoxGeometry(0.6, 0.4, 0.4);
          break;
        default:
          geometry = new THREE.OctahedronGeometry(0.3);
      }
      
      // Enhanced materials with better colors
      let color;
      switch (file.file_type) {
        case 'image':
          color = 0x00ffaa; // Bright cyan-green
          break;
        case 'video':
          color = 0xff0088; // Bright magenta
          break;
        case 'document':
          color = 0xffaa00; // Bright orange
          break;
        case 'folder':
          color = 0x8800ff; // Bright purple
          break;
        default:
          color = 0xffffff; // White
      }

      const material = new THREE.MeshBasicMaterial({
        color: color,
        transparent: true,
        opacity: 0.8,
        wireframe: true
      });

      const thumbnail = new THREE.Mesh(geometry, material);
      
      // Position in a more dynamic arrangement
      const radius = 6 + Math.sin(i * 0.5) * 1;
      const angle = (i / maxThumbnails) * Math.PI * 2;
      const height = Math.sin(i * 0.8) * 2;
      
      thumbnail.position.x = Math.cos(angle) * radius;
      thumbnail.position.y = Math.sin(angle) * radius;
      thumbnail.position.z = height;
      
      // Store file data
      thumbnail.userData = { file: file };
      
      thumbnails.push({
        mesh: thumbnail,
        file: file,
        angle: angle,
        radius: radius,
        baseHeight: height,
        rotationSpeed: 0.005 + Math.random() * 0.01
      });
      
      thumbnailGroupRef.current.add(thumbnail);
    }

    thumbnailsRef.current = thumbnails;
  };

  const animateDynamicRays = () => {
    neuralRaysRef.current.forEach((ray) => {
      ray.phase += ray.pulseSpeed;
      const intensity = (Math.sin(ray.phase) + 1) * 0.5;
      ray.line.material.opacity = ray.baseOpacity + intensity * 0.4;
      
      // Dynamic ray length pulsing
      const pulseLength = 0.8 + intensity * 0.4;
      const currentEnd = ray.originalEnd.clone().multiplyScalar(pulseLength);
      const positions = ray.line.geometry.attributes.position.array;
      positions[3] = currentEnd.x;
      positions[4] = currentEnd.y;
      positions[5] = currentEnd.z;
      ray.line.geometry.attributes.position.needsUpdate = true;
    });
  };

  const animateEnhancedThumbnails = () => {
    const time = Date.now() * 0.0003;
    
    thumbnailsRef.current.forEach((thumbnail, index) => {
      // Enhanced rotation
      thumbnail.mesh.rotation.x += thumbnail.rotationSpeed;
      thumbnail.mesh.rotation.y += thumbnail.rotationSpeed * 0.7;
      thumbnail.mesh.rotation.z += thumbnail.rotationSpeed * 0.3;
      
      // Dynamic orbital movement with vertical oscillation
      const newAngle = thumbnail.angle + time;
      const dynamicRadius = thumbnail.radius + Math.sin(time * 2 + index) * 0.5;
      const dynamicHeight = thumbnail.baseHeight + Math.cos(time * 1.5 + index) * 0.8;
      
      thumbnail.mesh.position.x = Math.cos(newAngle) * dynamicRadius;
      thumbnail.mesh.position.y = Math.sin(newAngle) * dynamicRadius;
      thumbnail.mesh.position.z = dynamicHeight;
    });
  };

  const intensifyRaysToTarget = (targetPosition) => {
    neuralRaysRef.current.forEach((ray) => {
      const distance = ray.originalEnd.distanceTo(targetPosition);
      if (distance < 3) {
        ray.line.material.opacity = Math.min(1.0, ray.line.material.opacity + 0.4);
        ray.line.material.color.setHSL(0.8, 1, 0.8);
      }
    });
  };

  const createSelectionRipple = (position) => {
    const rippleGeometry = new THREE.RingGeometry(0.1, 0.2, 16);
    const rippleMaterial = new THREE.MeshBasicMaterial({
      color: 0xffffff,
      transparent: true,
      opacity: 1.0,
      side: THREE.DoubleSide
    });
    
    const ripple = new THREE.Mesh(rippleGeometry, rippleMaterial);
    ripple.position.copy(position);
    ripple.lookAt(cameraRef.current.position);
    sceneRef.current.add(ripple);
    
    let scale = 1;
    let opacity = 1;
    
    const animateRipple = () => {
      scale += 0.15;
      opacity -= 0.03;
      
      ripple.scale.set(scale, scale, scale);
      ripple.material.opacity = opacity;
      
      if (opacity > 0) {
        requestAnimationFrame(animateRipple);
      } else {
        sceneRef.current.remove(ripple);
        ripple.geometry.dispose();
        ripple.material.dispose();
      }
    };
    
    animateRipple();
  };

  // Recreate thumbnails when files change
  useEffect(() => {
    if (sceneRef.current && files.length > 0) {
      createEnhancedThumbnails(sceneRef.current);
    }
  }, [files]);

  return (
    <div className="relative w-full h-screen">
      <div ref={mountRef} className="w-full h-screen" />
      
      {/* Enhanced UI overlay */}
      {selectedFile && (
        <div className="absolute top-4 left-4 bg-black/90 text-cyan-400 p-6 rounded-lg border border-cyan-400/50 backdrop-blur-sm">
          <h3 className="text-xl font-bold mb-2">{selectedFile.name}</h3>
          <div className="space-y-1 text-sm">
            <p><span className="text-cyan-300">Type:</span> {selectedFile.file_type}</p>
            <p><span className="text-cyan-300">Size:</span> {Math.round(selectedFile.size / 1024)} KB</p>
            <p><span className="text-cyan-300">Modified:</span> {new Date(selectedFile.modified * 1000).toLocaleDateString()}</p>
          </div>
        </div>
      )}
      
      {hoveredFile && !selectedFile && (
        <div className="absolute top-4 left-4 bg-black/70 text-cyan-400 p-3 rounded border border-cyan-400/30">
          <p className="font-semibold">{hoveredFile.name}</p>
          <p className="text-xs opacity-70">{hoveredFile.file_type}</p>
        </div>
      )}
      
      {loading && (
        <div className="absolute top-4 right-4 text-cyan-400 animate-pulse">
          <div className="flex items-center space-x-2">
            <div className="w-2 h-2 bg-cyan-400 rounded-full animate-bounce"></div>
            <span>Loading files...</span>
          </div>
        </div>
      )}
      
      {/* Instructions */}
      <div className="absolute bottom-4 left-4 text-cyan-400/60 text-sm">
        <p>Touch to explore • Tap to select files</p>
      </div>
    </div>
  );
};



