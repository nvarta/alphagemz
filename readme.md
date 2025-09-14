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



