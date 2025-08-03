<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, watch } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
// CSS2DRenderer no longer needed - using Canvas-based sprites instead
import gsap from 'gsap';

interface DataItem {
  name: string;
  value: string;
  ranking: number;
  url_image: string;
}

interface DataProps {
  title: string;
  data: DataItem[];
}

interface CustomBackgrounds {
  sky: string;
  floor: string;
}

interface Props {
  data: DataProps;
  customBackgrounds: CustomBackgrounds;
  barTexture?: string;
}

const props = defineProps<Props>();

const emit = defineEmits<{
  'animation-status': [status: boolean]
}>();

// DOM refs
const chartContainer = ref<HTMLElement | null>(null);

// Reactive variables for UI controls
const isSpotlightMode = ref<boolean>(true); // Start in spotlight mode
const isAutoCamEnabled = ref<boolean>(false);
const currentRank = ref<number>(0);
const isRecording = ref<boolean>(false);
const recordingProgress = ref<number>(0); // Progress percentage for recording
const recordingQuality = ref<string>('medium'); // 'low', 'medium', 'high'
const isCameraSetupMode = ref<boolean>(true); // Start in camera setup mode
const isAnimationStarted = ref<boolean>(false); // Track if animation has started
const heightScaleFactor = ref<number>(1.0); // Scale factor for bar heights (0.1 to 2.0)
const barHeightIncrement = ref<number>(0.8); // Level kenaikan antar bar (0.5 to 5.0) - dikurangi dari 2.0 ke 0.8
const minimumBarHeight = ref<number>(2.0); // Tinggi minimum bar (1.0 to 5.0)

// Fog controls
const fogIntensity = ref<number>(0.003); // Dikurangi dari 0.008 ke 0.003
const fogType = ref<string>('exponential'); // 'linear' or 'exponential'
const showFogControls = ref<boolean>(false); // Toggle fog controls visibility

// Variables to store camera setup position and target
let savedCameraPosition: THREE.Vector3 | null = null;
let savedControlsTarget: THREE.Vector3 | null = null;

// Three.js variables
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
// labelRenderer no longer needed - using Canvas-based sprites
let controls: OrbitControls;
let bars: THREE.Mesh[] = [];
let labels: THREE.Mesh[] = []; // Changed from Sprite to Mesh (plane)
let flagSprites: THREE.Mesh[] = []; // Changed from Sprite to Mesh (plane)
const maxVisibleTextures = 10; // Sesuaikan dengan batas GPU (misal 10 untuk aman)
const fallbackMaterial = new THREE.MeshBasicMaterial({ color: 0x888888, transparent: true }); // Material tanpa tekstur
let floor: THREE.Mesh;
let animationInProgress = false;
let autoCamAnimation: gsap.core.Tween | null = null;

// Recording variables
let capturer: any = null;
let recordingStartTime: number = 0;
let recordingDuration: number = 0;
let recordingTimer: number | null = null;
let originalPixelRatio: number = 1;
let originalShadowMapSize: number = 2048;

// Global variables for separate textures
let labelTextures: THREE.CanvasTexture[] = [];
let flagTextures: THREE.Texture[] = []; // Changed from CanvasTexture to Texture for flag images

// Texture pools to reduce memory usage
let texturePool: Map<string, THREE.CanvasTexture> = new Map();
let flagTexturePool: Map<string, THREE.Texture> = new Map();

// Theme colors
const themes = {
  default: {
    background: 0x121212,
    barColors: {
      low: 0x33cc33,    // Green
      mid: 0xff9933,     // Orange
      high: 0xff3333     // Red
    },
    textColor: 'white',
    floorColor: 0x333333,
    gridColor: 0x444444,
    fogColor: 0x121212,
    fogDensity: 0.01
  },
  vibrant: {
    background: 0x000000,
    barColors: {
      low: 0x00ffcc,     // Teal
      mid: 0xff66cc,     // Pink
      high: 0xff3366      // Red-Pink
    },
    textColor: 'white',
    floorColor: 0x222222,
    gridColor: 0x333333,
    fogColor: 0x000000,
    fogDensity: 0.015
  },
  cinematic: {
    background: 0x0a0a0a,
    barColors: {
      low: 0x3366ff,     // Blue
      mid: 0xff9900,     // Orange
      high: 0xff0000      // Red
    },
    textColor: '#f0f0f0',
    floorColor: 0x1a1a1a,
    gridColor: 0x2a2a2a,
    fogColor: 0x0a0a0a,
    fogDensity: 0.02
  },
  sky: {
    background: 0x87CEEB, // Sky blue (though we'll use texture instead)
    barColors: {
      low: 0x4CAF50,     // Green
      mid: 0xFF9800,     // Orange
      high: 0xF44336     // Red
    },
    textColor: '#333333', // Darker text for better visibility against sky
    floorColor: 0x4CAF50, // Green for grass (though we'll use texture instead)
    gridColor: 0xAAAAAA,  // Subtle gray grid
    gridColor2: 0x666666, // Darker grid for larger grid
    fogColor: 0x87CEEB,   // Sky blue for fog
    fogDensity: 0.008     // Lower density for better visibility
  }
};

// Create 3D title text based on JSON data
const createTitleText = () => {
  if (!props.data || !props.data.title) return;
  
  // Remove existing title text if any
  const existingTitle = scene.getObjectByName('titleText');
  if (existingTitle && existingTitle instanceof THREE.Mesh) {
    scene.remove(existingTitle);
    if (existingTitle.geometry) existingTitle.geometry.dispose();
    if (existingTitle.material) {
      if (Array.isArray(existingTitle.material)) {
        existingTitle.material.forEach((mat: THREE.Material) => mat.dispose());
      } else {
        existingTitle.material.dispose();
      }
    }
  }
  
  // Create canvas for 3D text texture
  const canvas = document.createElement('canvas');
  const context = canvas.getContext('2d');
  
  if (!context) return;
  
  // Set canvas size for high quality text - increased for larger font
  const devicePixelRatio = Math.max(window.devicePixelRatio || 1, 2);
  canvas.width = 1600 * devicePixelRatio; // Increased width from 1400 to 1600
  canvas.height = 350 * devicePixelRatio; // Increased height
  canvas.style.width = '1600px'; // Update style width
  canvas.style.height = '350px';
  
  // Scale context for high DPI
  context.scale(devicePixelRatio, devicePixelRatio);
  context.imageSmoothingEnabled = true;
  context.imageSmoothingQuality = 'high';
  
  // Set text properties - increased font size
  const fontSize = 72; // Increased from 48 to 72
  context.font = `bold ${fontSize}px Arial, sans-serif`;
  context.textAlign = 'center';
  context.textBaseline = 'middle';
  
  // Clear background (transparent)
  context.clearRect(0, 0, 1600, 350); // Update clearRect width
  
  // Draw text with outline for better visibility
  const drawTextWithOutline = (text: string, x: number, y: number) => {
    // Draw thick black outline for better contrast
    context.strokeStyle = '#000';
    context.lineWidth = 10; // Increased from 6 to 10 for better visibility
    context.strokeText(text, x, y);
    
    // Draw additional white outline for extra contrast
    context.strokeStyle = '#fff';
    context.lineWidth = 18;
    context.strokeText(text, x, y);
    
    // Draw main text with gradient
    const textGradient = context.createLinearGradient(0, y - fontSize/2, 0, y + fontSize/2);
    textGradient.addColorStop(0, '#FFD700'); // Gold top
    textGradient.addColorStop(0.5, '#FFA500'); // Orange middle
    textGradient.addColorStop(1, '#FF6347'); // Red bottom
    
    context.fillStyle = textGradient;
    context.fillText(text, x, y);
    
    // Add inner shadow/highlight
    context.shadowColor = 'rgba(255, 255, 255, 0.3)';
    context.shadowBlur = 2;
    context.shadowOffsetX = 1;
    context.shadowOffsetY = 1;
    context.fillText(text, x, y);
    
    // Reset shadow
    context.shadowColor = 'transparent';
    context.shadowBlur = 0;
    context.shadowOffsetX = 0;
    context.shadowOffsetY = 0;
  };
  
  // Draw the title text - adjusted for larger canvas
  drawTextWithOutline(props.data.title, 800, 175); // Center of 1600x350 canvas (800 instead of 700)
  
  // Create texture from canvas
  const texture = new THREE.CanvasTexture(canvas);
  texture.minFilter = THREE.LinearFilter;
  texture.magFilter = THREE.LinearFilter;
  texture.generateMipmaps = false;
  texture.anisotropy = renderer.capabilities.getMaxAnisotropy();
  
  // Create plane geometry for the text - adjusted for larger canvas
  const textGeometry = new THREE.PlaneGeometry(45, 10); // Increase plane width from 40 to 45
  const textMaterial = new THREE.MeshBasicMaterial({
    map: texture,
    transparent: true,
    alphaTest: 0.1,
    side: THREE.DoubleSide
  });
  
  // Create mesh
  const titleMesh = new THREE.Mesh(textGeometry, textMaterial);
  titleMesh.name = 'titleText';
  
  // Position the title behind the chart bars
  const chartCenter = bars.length > 0 ? 0 : 0; // Center X position
  const titleDistance = 25; // Distance behind the bars
  const titleHeight = 8; // Height above ground
  
  titleMesh.position.set(chartCenter, titleHeight, -titleDistance);
  titleMesh.rotation.y = 0; // Face forward
  
  // Add to scene
  scene.add(titleMesh);
};

// Initialize Three.js scene
const initScene = () => {
  if (!chartContainer.value) return;

  // Create scene
  scene = new THREE.Scene();
  
  // Load sky background texture
  const skyTextureLoader = new THREE.TextureLoader();
  const skyTexture = skyTextureLoader.load(props.customBackgrounds.sky);
  scene.background = skyTexture;
  
  // Add enhanced fog for depth perception with sky-compatible color
  const skyFogColor = new THREE.Color(0x87CEEB); // Light blue color for sky
  scene.fog = new THREE.FogExp2(skyFogColor, fogIntensity.value); // Use reactive fog intensity

  // Create camera with cinematic field of view and fixed 16:9 aspect ratio
  const aspect = 16 / 9; // Fixed 16:9 aspect ratio for camera
  camera = new THREE.PerspectiveCamera(40, aspect, 0.1, 1000); // Narrower FOV for more cinematic look
  // Position camera for front-facing low-angle view (will be moved during animation)
  camera.position.set(0, 10, 50); // Centered X, higher Y for better view, much further back Z to see full chart
  camera.lookAt(0, 2, 0); // Look slightly higher above the ground

  // Create WebGL renderer with improved shadow settings
  renderer = new THREE.WebGLRenderer({ 
    antialias: true, 
    alpha: true,
    powerPreference: "high-performance", // Use high-performance GPU if available
    preserveDrawingBuffer: true, // Required for video recording
    // Add memory optimizations
    logarithmicDepthBuffer: false, // Disable for better performance
    precision: "mediump" // Use medium precision to save memory
  });
  
  // Set limits based on data length
  const dataLength = props.data?.data?.length || 0;

  // Optimize renderer settings for different dataset sizes
  if (dataLength > 50) {
    // For very large datasets (50+ items) - aggressive optimization
    renderer.setPixelRatio(0.75); // Reduce to lower pixel ratio
    renderer.shadowMap.enabled = false; // Disable shadows for performance
  } else if (dataLength > 30) {
    // For large datasets (30-50 items)
    renderer.setPixelRatio(1.0); // Reduce to base pixel ratio
    renderer.shadowMap.enabled = false;
    renderer.shadowMap.type = THREE.BasicShadowMap;
  } else if (dataLength > 20) {
    // For medium-large datasets (20-30 items)
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.2));
    renderer.shadowMap.type = THREE.BasicShadowMap;
  } else {
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  }
  
  // Force WebGL context optimization
  const gl = renderer.getContext();
  const maxTextures = gl.getParameter(gl.MAX_TEXTURE_IMAGE_UNITS);

  // Set strict texture limits for large datasets
  if (dataLength > 20) {
    renderer.capabilities.maxTextures = Math.min(maxTextures, 8); // Conservative limit for 20+
    // Force texture size reduction
    renderer.capabilities.maxTextureSize = 256; // Small textures
  } else if (dataLength > 15) {
    renderer.capabilities.maxTextures = Math.min(maxTextures, 12);
    // Force texture size reduction
    renderer.capabilities.maxTextureSize = 512;
  } else {
    // For small datasets, use conservative limit
    renderer.capabilities.maxTextures = Math.min(maxTextures, 14); // Conservative limit
    renderer.capabilities.maxTextureSize = 1024;
  }
  
  // Call texture monitoring to prevent issues
  setTimeout(() => {
    monitorTextureUsage();
  }, 100);
  
  // Helper function to create fallback flag texture
  const createFallbackFlagTexture = (item: any, index: number) => {
    const key = `fallback-${item.name}-${index}`;
    
    if (flagTexturePool.has(key)) {
      const texture = flagTexturePool.get(key)!;
      return texture;
    }
    
    const flagCanvas = document.createElement('canvas');
    flagCanvas.width = 256;
    flagCanvas.height = 64; // 4:1 ratio for flag
    const flagContext = flagCanvas.getContext('2d');
    
    if (flagContext) {
      flagContext.fillStyle = `hsl(${index * 30}, 80%, 60%)`;
      flagContext.fillRect(0, 0, flagCanvas.width, flagCanvas.height);
      flagContext.strokeStyle = 'white';
      flagContext.lineWidth = 4;
      flagContext.strokeRect(4, 4, flagCanvas.width - 8, flagCanvas.height - 8);
      flagContext.fillStyle = 'white';
      flagContext.font = 'bold 32px Arial';
      flagContext.textAlign = 'center';
      flagContext.textBaseline = 'middle';
      flagContext.fillText(item.name.charAt(0), flagCanvas.width/2, flagCanvas.height/2);
    }
    
    const flagTexture = new THREE.CanvasTexture(flagCanvas);
    flagTexture.minFilter = THREE.LinearFilter;
    flagTexture.magFilter = THREE.LinearFilter;
    flagTexture.generateMipmaps = false;
    flagTexturePool.set(key, flagTexture);
    return flagTexture;
  };
  
  // Calculate renderer size to maintain 16:9 aspect ratio within container
  const containerWidth = chartContainer.value.clientWidth;
  const containerHeight = chartContainer.value.clientHeight;
  
  // Set renderer size to maintain 16:9 aspect ratio
  renderer.setSize(containerWidth, containerWidth * (9/16));
  renderer.shadowMap.enabled = true;
  
  // Optimize renderer for recording performance
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1.0;
  
  // Center the renderer in the container
  renderer.domElement.style.display = 'block';
  renderer.domElement.style.margin = '0 auto';
  
  chartContainer.value.appendChild(renderer.domElement);
  
  // CSS2D renderer no longer needed - using Canvas-based sprites

  // Add controls with enhanced settings
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.rotateSpeed = 0.5;
  controls.enableZoom = true;
  controls.enablePan = true; // Enable panning for camera setup mode
  controls.minDistance = 10;
  controls.maxDistance = 100;
  controls.enabled = true; // Enable controls initially for camera setup mode

  // Enhanced lighting setup for outdoor sky visuals
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.7); // Brighter ambient for outdoor scene
  scene.add(ambientLight);

  // Main sunlight
  const sunLight = new THREE.DirectionalLight(0xFFD54F, 1.2); // Warm sunlight color
  sunLight.position.set(15, 30, 20); // Coming from higher up like the sun
  sunLight.castShadow = true;
  sunLight.shadow.mapSize.width = 2048;
  sunLight.shadow.mapSize.height = 2048;
  sunLight.shadow.camera.near = 0.5;
  sunLight.shadow.camera.far = 500;
  sunLight.shadow.bias = -0.0001;
  scene.add(sunLight);
  
  // Add fill light from opposite side for better illumination
  const fillLight = new THREE.DirectionalLight(0xE1F5FE, 0.4); // Slight blue tint for sky reflection
  fillLight.position.set(-10, 10, -10);
  scene.add(fillLight);
  
  // Add rim light for dramatic effect and edge highlighting
  const rimLight = new THREE.SpotLight(0xFFF8E1, 0.6); // Warm rim light
  rimLight.position.set(0, 20, -15);
  rimLight.angle = Math.PI / 4;
  rimLight.penumbra = 0.5;
  rimLight.decay = 2;
  scene.add(rimLight);

  // Create initial floor and grid with default size
  // This will be updated later when data is loaded
  const floorSize = 100;
  const floorGeometry = new THREE.PlaneGeometry(floorSize, floorSize);
  
  // Load custom floor texture
  const floorTextureLoader = new THREE.TextureLoader();
  const floorTexture = floorTextureLoader.load(props.customBackgrounds.floor);
  
  // Set texture repeat to cover large floor area
  floorTexture.wrapS = THREE.RepeatWrapping;
  floorTexture.wrapT = THREE.RepeatWrapping;
  floorTexture.repeat.set(10, 10);
  
  const floorMaterial = new THREE.MeshBasicMaterial({ 
    map: floorTexture,
    side: THREE.DoubleSide
  });
  
  floor = new THREE.Mesh(floorGeometry, floorMaterial);
  floor.rotation.x = Math.PI / 2;
  floor.position.y = -5;
  floor.receiveShadow = true;
  scene.add(floor);
  
  // Add initial grid
  const gridHelper = new THREE.GridHelper(floorSize, 50, 0xAAAAAA, 0x888888);
  gridHelper.position.y = -4.95;
  gridHelper.material.opacity = 0.3;
  gridHelper.material.transparent = true;
  scene.add(gridHelper);
  
  // Add a second, larger grid for more depth
  const largeGridHelper = new THREE.GridHelper(floorSize, 10, 0x666666, 0x666666);
  largeGridHelper.position.y = -4.9;
  largeGridHelper.material.opacity = 0.5;
  largeGridHelper.material.transparent = true;
  scene.add(largeGridHelper);

  // Create bars
  createBars();
  
  // Create 3D title text
  createTitleText();

  // Animation loop
  animate();
  
  // Don't start animation automatically
  // Wait for user to click Start Animation button

  // Handle window resize
  window.addEventListener('resize', onWindowResize);
  
  // No UI controls needed - they're in the template
};

// Function to update floor and grid size based on data length
  const updateFloorAndGridSize = () => {
    if (!props.data || !props.data.data || props.data.data.length === 0) return;
    
    // Calculate floor size based on data length
    const spacing = 2.5; // Same spacing as used for bars - dikurangi dari 4 ke 2.5
    const dataLength = props.data.data.length;
    // Calculate required width based on data length and spacing
    // Add extra padding (20 units) on each side for visual appeal
    const floorSize = Math.max(100, (dataLength * spacing) + 40);
    
    // Remove existing floor and grids
    scene.children.forEach(child => {
      if (child === floor || child instanceof THREE.GridHelper) {
        scene.remove(child);
      }
    });
    
    // Create new floor with updated size
    const floorGeometry = new THREE.PlaneGeometry(floorSize, floorSize);
    const floorTexture = new THREE.TextureLoader().load(props.customBackgrounds.floor);
    
    // Set texture repeat to cover large floor area
    floorTexture.wrapS = THREE.RepeatWrapping;
    floorTexture.wrapT = THREE.RepeatWrapping;
    // Adjust texture repeat based on floor size
    const repeatCount = Math.ceil(floorSize / 10);
    floorTexture.repeat.set(repeatCount, repeatCount);
    
    const floorMaterial = new THREE.MeshBasicMaterial({ 
      map: floorTexture,
      side: THREE.DoubleSide
    });
    
    floor = new THREE.Mesh(floorGeometry, floorMaterial);
    floor.rotation.x = Math.PI / 2;
    floor.position.y = -5;
    floor.receiveShadow = true;
    scene.add(floor);
    
    // Add enhanced grid with more detail - subtle grid that complements the floor texture
    const gridHelper = new THREE.GridHelper(floorSize, Math.floor(floorSize/2), 0xAAAAAA, 0x888888); // Subtle gray grid
    gridHelper.position.y = -4.95; // Very slightly above floor
    gridHelper.material.opacity = 0.3; // Make grid semi-transparent
    gridHelper.material.transparent = true; // Enable transparency
    scene.add(gridHelper);
    
    // Add a second, larger grid for more depth
    const largeGridHelper = new THREE.GridHelper(floorSize, Math.floor(floorSize/10), 0x666666, 0x666666); // Darker, larger grid
    largeGridHelper.position.y = -4.9; // Slightly above the first grid
    largeGridHelper.material.opacity = 0.5; // Make grid semi-transparent
    largeGridHelper.material.transparent = true; // Enable transparency
    scene.add(largeGridHelper);
  };

// Create 3D bars
const createBars = () => {
  if (!props.data || !props.data.data || props.data.data.length === 0) return;

  // Reset animation state
  currentRank.value = 0;
  animationInProgress = false;
  emit('animation-status', false);
    
  // Update floor and grid size based on data length
  updateFloorAndGridSize();
    
  // If in camera setup mode, create all bars but don't animate them
  // This allows the user to see the full chart while setting up the camera
  
  // Clear existing objects with proper cleanup
  bars.forEach(bar => {
    scene.remove(bar);
    // Dispose geometry and material
    if (bar.geometry) bar.geometry.dispose();
    if (bar.material) {
      if (Array.isArray(bar.material)) {
        bar.material.forEach((mat: THREE.Material) => mat.dispose());
      } else {
        bar.material.dispose();
      }
    }
  });
    
  labels.forEach(label => {
    scene.remove(label);
    // Dispose texture and material
    if (label.userData.texture) {
      label.userData.texture.dispose();
    }
    if (label.material) {
      if (Array.isArray(label.material)) {
        label.material.forEach((mat: THREE.Material) => mat.dispose());
      } else {
        label.material.dispose();
      }
    }
  });
    
  flagSprites.forEach(sprite => {
    scene.remove(sprite);
    // Dispose texture and material
    if (sprite.userData.texture) {
      sprite.userData.texture.dispose();
    }
    if (sprite.material) {
      if (Array.isArray(sprite.material)) {
        sprite.material.forEach((mat: THREE.Material) => mat.dispose());
      } else {
        sprite.material.dispose();
      }
    }
  });
    
  // Dispose separate textures
  if (labelTextures) {
    labelTextures.forEach(texture => texture.dispose());
  }
  if (flagTextures) {
    flagTextures.forEach(texture => texture.dispose());
  }

  // Clear texture pools
  texturePool.forEach(texture => texture.dispose());
  flagTexturePool.forEach(texture => texture.dispose());
  texturePool.clear();
  flagTexturePool.clear();
    
  // Also remove any spotlights that might be lingering
  scene.children.forEach(child => {
    if (child instanceof THREE.SpotLight) {
      scene.remove(child);
    }
  });
    
  bars = [];
  labels = [];
  flagSprites = [];

  // Sort data by ranking (descending) - from highest ranking to lowest ranking
  // This way ranking 1 (best) will be at the end and appear last with tallest bar
  const sortedData = [...props.data.data].sort((a, b) => b.ranking - a.ranking);

  // Calculate max ranking for scaling
  const maxRanking = Math.max(...sortedData.map(item => item.ranking));

  // Helper function to create fallback flag texture
  const createFallbackFlagTexture = (item: any, index: number) => {
    const key = `fallback-${item.name}-${index}`;
    
    if (flagTexturePool.has(key)) {
      const texture = flagTexturePool.get(key)!;
      return texture;
    }
    
    const flagCanvas = document.createElement('canvas');
    flagCanvas.width = 256;
    flagCanvas.height = 64; // 4:1 ratio for flag
    const flagContext = flagCanvas.getContext('2d');
    
    if (flagContext) {
      flagContext.fillStyle = `hsl(${index * 30}, 80%, 60%)`;
      flagContext.fillRect(0, 0, flagCanvas.width, flagCanvas.height);
      flagContext.strokeStyle = 'white';
      flagContext.lineWidth = 4;
      flagContext.strokeRect(4, 4, flagCanvas.width - 8, flagCanvas.height - 8);
      flagContext.fillStyle = 'white';
      flagContext.font = 'bold 32px Arial';
      flagContext.textAlign = 'center';
      flagContext.textBaseline = 'middle';
      flagContext.fillText(item.name.charAt(0), flagCanvas.width/2, flagCanvas.height/2);
    }
    
    const flagTexture = new THREE.CanvasTexture(flagCanvas);
    flagTexture.minFilter = THREE.LinearFilter;
    flagTexture.magFilter = THREE.LinearFilter;
    flagTexture.generateMipmaps = false;
    flagTexturePool.set(key, flagTexture);
    return flagTexture;
  };

  // Create label texture function with optimization for large datasets
  const createLabelTexture = (item: any, index: number, maxRanking: number) => {
    // Generate a unique key for this label
    const key = `label-${item.name}-${item.ranking}-${item.value}`;
    
    // Check if we already have this texture in the pool
    if (texturePool.has(key)) {
      const texture = texturePool.get(key)!;
      return texture;
    }
    
    // Sesuaikan kualitas texture berdasarkan jumlah data
    const dataLength = sortedData.length;
    let devicePixelRatio = Math.max(window.devicePixelRatio || 1, 2);
    let superSampleRatio = 3;
    
    // Kurangi kualitas untuk data banyak
    if (dataLength > 50) {
      devicePixelRatio = 1.0; // Minimal pixel ratio
      superSampleRatio = 1.0;  // No super sampling
    } else if (dataLength > 30) {
      devicePixelRatio = 1.5; // Kurangi pixel ratio
      superSampleRatio = 1.5;  // Kurangi super sampling
    } else if (dataLength > 20) {
      devicePixelRatio = 2;
      superSampleRatio = 2;
    }
    
    // Create label texture
    const labelCanvas = document.createElement('canvas');
    
    // Calculate text dimensions first to size canvas accordingly
    const tempCanvas = document.createElement('canvas');
    const tempContext = tempCanvas.getContext('2d');
    
    let maxTextWidth = 0;
    let totalTextHeight = 0;
    
    if (tempContext) {
       // Measure rank text - DIPERBESAR LAGI menjadi 20px untuk semua ranking
       tempContext.font = 'bold 20px Arial, sans-serif'; // Increased to 20px for all rankings
       const rankText = `#${item.ranking}`; // Use actual ranking
       maxTextWidth = Math.max(maxTextWidth, tempContext.measureText(rankText).width);
       totalTextHeight += 24; // Font size + spacing (increased)
       
       // Measure name text - DIPERBESAR LAGI menjadi 18px
       tempContext.font = 'bold 18px Arial, sans-serif'; // Increased to 18px
       maxTextWidth = Math.max(maxTextWidth, tempContext.measureText(item.name).width);
       totalTextHeight += 22; // Font size + spacing (increased)
       
       // Measure value text - DIPERBESAR LAGI menjadi 16px
       tempContext.font = 'bold 16px Arial, sans-serif'; // Increased to 16px
       const valueText = item.value.toString(); // Use full text instead of parsing
       maxTextWidth = Math.max(maxTextWidth, tempContext.measureText(valueText).width);
       totalTextHeight += 20; // Font size + spacing (increased)
     }
    
    // Calculate additional height based on value
    // Extract numeric value more intelligently
    const numericMatch = item.value.toString().match(/([0-9]+\.?[0-9]*)/); // Extract first number
    const numericValue = numericMatch ? parseFloat(numericMatch[1]) : 0;
    const maxValue = Math.max(...sortedData.map(d => {
      const match = d.value.toString().match(/([0-9]+\.?[0-9]*)/); 
      return match ? parseFloat(match[1]) : 0;
    }));
    const normalizedValue = maxValue > 0 ? numericValue / maxValue : 0;
    const additionalHeight = Math.round(normalizedValue * 100); // Increased scale factor to 100
    
    // Add 50% padding and use higher pixel ratio for super crisp text
    // devicePixelRatio and superSampleRatio are now defined at the top of the function
    const canvasWidth = Math.round(maxTextWidth * 1.4); // Increased from 1.2 to 1.5
    const canvasHeight = Math.round(totalTextHeight * 1) + additionalHeight; // Add value-based height
    
    labelCanvas.width = canvasWidth * devicePixelRatio * superSampleRatio;
    labelCanvas.height = canvasHeight * devicePixelRatio * superSampleRatio;
    labelCanvas.style.width = canvasWidth + 'px';
    labelCanvas.style.height = canvasHeight + 'px';
    
    const labelContext = labelCanvas.getContext('2d');
    if (labelContext) {
      labelContext.scale(devicePixelRatio * superSampleRatio, devicePixelRatio * superSampleRatio);
      labelContext.imageSmoothingEnabled = true;
      labelContext.imageSmoothingQuality = 'high';
    }
    
    if (labelContext) {
      labelContext.fillStyle = 'rgba(255, 255, 255, 0.9)';
      labelContext.fillRect(0, 0, canvasWidth, canvasHeight);
      labelContext.strokeStyle = '#333';
      labelContext.lineWidth = 1.5; // Slightly thinner border for sharper look
      labelContext.strokeRect(1, 1, canvasWidth - 2, canvasHeight - 2);
      
      labelContext.textAlign = 'center';
      labelContext.textBaseline = 'middle';
      
      const drawTextWithOutline = (text: string, x: number, y: number, fontSize: number, fillColor: string) => {
        labelContext.font = `bold ${fontSize}px Arial, sans-serif`;
        labelContext.lineWidth = 2; // Reduced outline width for sharper text
        labelContext.strokeStyle = '#ffffff';
        labelContext.fillStyle = fillColor;
        labelContext.strokeText(text, x, y);
        labelContext.fillText(text, x, y);
      };
      
      const rankText = `#${item.ranking}`; // Use actual ranking instead of calculated position
      
      // Font sizes yang lebih besar untuk semua ranking
      let rankColor;
      let rankFontSize = 28; // Increased to 28px for all rankings
      
      if (item.ranking === 1) {
        // Ranking #1 - Merah Maroon
        rankColor = '#800000'; // Maroon red
      } else if (item.ranking === 2) {
        // Ranking #2 - Orange Gelap
        rankColor = '#CC5500'; // Dark orange
      } else {
        // Ranking lainnya - berdasarkan ranking, bukan value
        const normalizedRanking = item.ranking / maxRanking;
        rankColor = normalizedRanking > 0.66 ? '#10324d' : '#000000';
      }
      
      const valueText = item.value.toString(); // Use full original text
      drawTextWithOutline(rankText, canvasWidth/2, canvasHeight/4, rankFontSize, rankColor); // 28px untuk ranking
      drawTextWithOutline(item.name, canvasWidth/2, canvasHeight/2, 24, '#000000'); // 24px untuk nama
      drawTextWithOutline(valueText, canvasWidth/2, 3*canvasHeight/4, 20, '#000000'); // 20px untuk value
    }
    
    const labelTexture = new THREE.CanvasTexture(labelCanvas);
    labelTexture.minFilter = THREE.LinearFilter;
    labelTexture.magFilter = THREE.NearestFilter; // Use nearest for crisp text at zoom
    labelTexture.generateMipmaps = false;
    labelTexture.anisotropy = renderer.capabilities.getMaxAnisotropy(); // Maximum anisotropic filtering
    labelTexture.flipY = true; // Fix flipped text orientation
    
    // Store in pool for reuse
    texturePool.set(key, labelTexture);
    
    return labelTexture;
  };

  // Function to get or create flag texture with optimization for large datasets
  const getOrCreateFlagTexture = (item: any, index: number) => {
    if (item.url_image) {
      // Check if we already have this texture in the pool
      if (flagTexturePool.has(item.url_image)) {
        const texture = flagTexturePool.get(item.url_image)!;
        return texture;
      }
      
      // Load image from URL with optimization for large datasets
      const textureLoader = new THREE.TextureLoader();
      const flagTexture = textureLoader.load(
        item.url_image,
        // onLoad callback
        (texture) => {
          // Optimize texture based on dataset size
          const dataLength = sortedData.length;
          if (dataLength > 50) {
            texture.minFilter = THREE.NearestFilter; // Use nearest for performance
            texture.magFilter = THREE.NearestFilter;
            texture.generateMipmaps = false;
            // Force texture to be smaller
            if (texture.image) {
              texture.image.width = Math.min(texture.image.width, 128);
              texture.image.height = Math.min(texture.image.height, 64);
            }
          } else if (dataLength > 30) {
            texture.minFilter = THREE.LinearFilter;
            texture.magFilter = THREE.LinearFilter;
            texture.generateMipmaps = false;
            // Force texture to be smaller
            if (texture.image) {
              texture.image.width = Math.min(texture.image.width, 256);
              texture.image.height = Math.min(texture.image.height, 128);
            }
          } else {
            texture.minFilter = THREE.LinearFilter;
            texture.magFilter = THREE.LinearFilter;
            texture.generateMipmaps = false;
          }
        },
        // onProgress callback
        undefined,
        // onError callback - fallback to generated texture
        () => {
          console.warn(`Failed to load image for ${item.name}, using fallback`);
          return createFallbackFlagTexture(item, index);
        }
      );
      
      // Store in pool for reuse
      flagTexturePool.set(item.url_image, flagTexture);
      return flagTexture;
    } else {
      // Fallback to generated texture
      return createFallbackFlagTexture(item, index);
    }
  };

  // Create separate textures for labels and flags with optimization
  const createSeparateTextures = () => {
    const labelTextures: THREE.CanvasTexture[] = [];
    const flagTextures: THREE.Texture[] = [];
    
    // For very large datasets, use texture atlasing or reduce quality
    const dataLength = sortedData.length;
    
    if (dataLength > 20) {
      // For 20+ items, create simplified textures to prevent texture limit issues
      sortedData.forEach((item, index) => {
        // Create simplified label texture
        const labelTexture = createLabelTexture(item, index, maxRanking);
        labelTextures.push(labelTexture);
        
        // Create simplified flag texture
        const flagTexture = createFallbackFlagTexture(item, index);
        flagTextures.push(flagTexture);
      });
    } else {
      // Normal texture creation for smaller datasets
      sortedData.forEach((item, index) => {
        // Create and store label texture
        const labelTexture = createLabelTexture(item, index, maxRanking);
        labelTextures.push(labelTexture);
        
        // Create and store flag texture
        const flagTexture = getOrCreateFlagTexture(item, index);
        flagTextures.push(flagTexture);
      });
    }
     
    return { labelTextures, flagTextures };
  };

  const textureData = createSeparateTextures();
  labelTextures = textureData.labelTextures;
  flagTextures = textureData.flagTextures;
    
  // Default bar colors (no theme needed)
  const defaultBarColors = {
    low: 0x4CAF50,    // Green
    mid: 0xFF9800,    // Orange  
    high: 0xF44336    // Red
  };
    
  // Create bars
  sortedData.forEach((item, index) => {
    // Calculate bar height based on ranking (lower ranking number = taller bar)
    // Ranking 1 = tallest, highest ranking number = shortest
    const maxRanking = Math.max(...sortedData.map(d => d.ranking));
    // Invert the ranking calculation so ranking 1 becomes the tallest
    const invertedRanking = maxRanking - item.ranking + 1;
    // Apply minimum height and increment between bars
    const baseHeight = minimumBarHeight.value + ((invertedRanking - 1) * barHeightIncrement.value);
    const height = baseHeight * heightScaleFactor.value; // Apply user-defined scale factor
    
    // Create bar geometry
    const geometry = new THREE.BoxGeometry(2, height, 2);
    
    // Create color based on ranking position only
    let barColor;
    
    // Special colors for top rankings (ranking 1 = best, ranking 2 = second best)
    if (item.ranking === 1) {
      // Ranking #1 (best ranking) - Merah Maroon
      barColor = new THREE.Color(0x800000); // Maroon red
    } else if (item.ranking === 2) {
      // Ranking #2 - Orange Gelap
      barColor = new THREE.Color(0xCC5500); // Dark orange
    } else {
      // For other rankings, use gradient based on inverted ranking position
      const normalizedRanking = invertedRanking / maxRanking; // 0 to 1
      
      // Create a color gradient
      const lowColor = new THREE.Color(defaultBarColors.low);   // Low ranking color
      const midColor = new THREE.Color(defaultBarColors.mid);   // Mid ranking color
      const highColor = new THREE.Color(defaultBarColors.high);  // High ranking color
      
      // Determine color based on ranking range
      if (normalizedRanking < 0.5) {
        // Interpolate between low and mid colors
        barColor = new THREE.Color().lerpColors(lowColor, midColor, normalizedRanking * 2);
      } else {
        // Interpolate between mid and high colors
        barColor = new THREE.Color().lerpColors(midColor, highColor, (normalizedRanking - 0.5) * 2);
      }
    }
    
    // Create material with optimized settings
    // Get shared texture if available
    const sharedTexture = getSharedBarTexture();
    
    // Optimize texture settings for better performance
    if (sharedTexture) {
      sharedTexture.minFilter = THREE.NearestFilter;
      sharedTexture.magFilter = THREE.NearestFilter;
      sharedTexture.anisotropy = 1; // Minimum value for better performance
    }
    let material = new THREE.MeshPhongMaterial({
      map: sharedTexture,
      color: barColor,
      transparent: false,
      opacity: 1.0,
      depthTest: true,
      depthWrite: true
    });
    
    // Create mesh
    const bar = new THREE.Mesh(geometry, material);
    bar.castShadow = true;
    bar.receiveShadow = true;
    
    // Position bar
    const spacing = 2.5; // Dikurangi dari 4 menjadi 2.5 untuk jarak lebih dekat
    const xPos = index * spacing - (sortedData.length - 1) * spacing / 2;
    bar.position.set(xPos, height / 2 - 5, 0); // Center Y position based on height
    
    // Set initial scale for animation
    bar.visible = false; // Hide initially
    
    // Add to scene and bars array
    scene.add(bar);
    bars.push(bar);
    
    // Create label plane using separate texture (menghadap depan, tidak mengikuti kamera)
    const labelGeometry = new THREE.PlaneGeometry(1.9, 1.9);
    const labelMaterial = new THREE.MeshBasicMaterial({
      map: labelTextures[index],
      transparent: true,
      alphaTest: 0.1,
      depthTest: true,  // Enable depth testing
      depthWrite: true, // Enable depth writing
      side: THREE.DoubleSide
    });
    const labelPlane = new THREE.Mesh(labelGeometry, labelMaterial);
    
    labelPlane.position.set(0, height/2 - 1, 1.1); // Position below the flag sprite
    labelPlane.rotation.y = 0; // Always face front (0 rotation)
    labelPlane.visible = false;
    
    bar.add(labelPlane);
    labels.push(labelPlane);
    
    // Create flag plane using separate texture (menghadap depan, tidak mengikuti kamera)
    const flagGeometry = new THREE.PlaneGeometry(1.8, 0.9);
    const flagMaterial = new THREE.MeshBasicMaterial({
      map: flagTextures[index],
      transparent: true,
      alphaTest: 0.1,
      depthTest: true,  // Enable depth testing
      depthWrite: true, // Enable depth writing
      side: THREE.DoubleSide
    });
    const flagPlane = new THREE.Mesh(flagGeometry, flagMaterial);
    
    flagPlane.position.set(0, height/2 + 0.5, 0.85);
    flagPlane.rotation.y = 0; // Always face front (0 rotation)
    flagPlane.visible = false;
    
    bar.add(flagPlane);
    flagSprites.push(flagPlane);
  });
  
  // Auto-adjust camera position based on maximum bar height
  const adjustCameraForBars = () => {
    if (bars.length === 0) return;
    
    // Calculate the maximum bar height in the scene including flag sprites
  const validBars = bars.filter(b => b && b.geometry && b.geometry.parameters);
  const maxBarHeight = validBars.length > 0 ? 
    Math.max(...validBars.map(bar => {
      const barHeight = (bar.geometry as THREE.BoxGeometry).parameters.height;
      const barTop = bar.position.y + barHeight / 2; // Top of the bar
      const flagHeight = 0.9; // Flag sprite scale height
      const flagOffset = 0.5; // Flag position offset above bar
      return barTop + flagOffset + flagHeight; // Flag sprite is the highest element
    })) : 0; // Default value jika tidak ada bar valid
    
    // Calculate chart width based on number of bars
    const spacing = 2.5; // Dikurangi dari 4 ke 2.5
    const chartWidth = (bars.length - 1) * spacing;
      
    // Calculate optimal camera distance to fit all bars
    // Use field of view and chart dimensions to calculate distance
    const fov = camera.fov * (Math.PI / 180); // Convert to radians
    
    // Untuk data banyak, kurangi margin dan jarak minimum
    const dataLength = bars.length;
    let heightMargin = maxBarHeight * 1.2; // Kurangi dari 1.5 ke 1.2
    let minDistance = 80; // Kurangi dari 120 ke 80
    
    if (dataLength > 50) {
      heightMargin = maxBarHeight * 0.8; // Very small margin for very large datasets
      minDistance = 40;
    } else if (dataLength > 20) {
      heightMargin = maxBarHeight * 1.0; // Margin lebih kecil untuk data banyak
      minDistance = 60;
    }
    
    const heightDistance = (maxBarHeight + heightMargin) / Math.tan(fov / 2); // Distance to fit height with margin
    const widthDistance = chartWidth / (2 * Math.tan(fov / 2) * camera.aspect); // Distance to fit width
    
    // Use the larger distance to ensure everything fits, with adjusted minimum distance
    const optimalDistance = Math.max(heightDistance, widthDistance, minDistance);
    
    // Calculate optimal camera height - keep perspective horizontal, don't go too high
    const optimalCameraHeight = Math.min(maxBarHeight * 0.4, 15); // Cap camera height at reasonable level
    const optimalLookAtHeight = maxBarHeight * 0.3; // Looking at 30% of max bar height
    
    // Only adjust if not in camera setup mode (to preserve user's manual setup)
    if (!isCameraSetupMode.value && !savedCameraPosition) {
      camera.position.set(0, optimalCameraHeight, optimalDistance);
      camera.lookAt(0, optimalLookAtHeight, 0);
      controls.target.set(0, optimalLookAtHeight, 0);
      controls.update();
    }
  };
  
  // Call auto-adjust function
  adjustCameraForBars();
  
  // If in camera setup mode, show all bars without animation
  if (isCameraSetupMode.value) {
    // Make all bars visible immediately
    bars.forEach((bar, index) => {
      bar.visible = true;
      
      // Show flags
      if (index < flagSprites.length) {
        flagSprites[index].visible = true;
      }
      
      // Show labels
      if (index < labels.length) {
        labels[index].visible = true;
      }
    });
    
    // Keep controls enabled for camera setup
    controls.enabled = true;
  }
  // If animation has started, proceed with normal animation modes
  else if (!isCameraSetupMode.value) {
    // If we have saved camera position and target from setup mode, restore them
    if (savedCameraPosition && savedControlsTarget) {
      // Set camera position and target to saved values
      camera.position.copy(savedCameraPosition);
      controls.target.copy(savedControlsTarget);
      camera.updateProjectionMatrix();
      controls.update();
    }
    
    if (isSpotlightMode.value) {
      // Start the spotlight animation with the current camera position
      startSpotlightAnimation();
    } else {
      // Show all bars at once for lineup view
      showLineupView();
    }
  }
};

// Spotlight animation - animate bars one by one with camera movement
const startSpotlightAnimation = () => {
  if (animationInProgress) return;
  animationInProgress = true;
  emit('animation-status', true);
  currentRank.value = 0; // Start from first rank (lowest value)
  
  // Auto-adjust camera for animation if no saved position
  if (!savedCameraPosition) {
    adjustCameraForBars();
  }
  
  // Process the first bar immediately without changing camera position
  processNextBar();
  
  // Enable controls for user interaction during animation
  controls.enabled = true;
};

// Process next bar in the spotlight sequence
const processNextBar = () => {
  if (currentRank.value >= bars.length) {
    // Animation complete, transition to lineup view
    transitionToLineupView();
    return;
  }
  
  const currentBar = bars[currentRank.value];
  // Check if currentBar is null
  if (!currentBar) {
    // Skip to next bar if current bar is null
    currentRank.value++;
    processNextBar();
    return;
  }
  
  const currentLabel = labels[currentRank.value];
  const currentFlag = flagSprites[currentRank.value];

  // Assign texture saat bar muncul
  if (currentLabel) {
    currentLabel.material.map = labelTextures[currentRank.value]; // Assign texture
    currentLabel.material.needsUpdate = true;
  }
  if (currentFlag) {
    currentFlag.material.map = flagTextures[currentRank.value];
    currentFlag.material.needsUpdate = true;
  }
function destroyBar(index) {
  const bar = bars[index];
  if (bar) {
    // Hapus label dan flag dari bar terlebih dahulu
    const label = labels[index];
    if (label) {
      bar.remove(label);
      if (label.geometry) label.geometry.dispose();
      if (label.material) {
        label.material.dispose();
        if (label.material.map) label.material.map.dispose();
      }
    }
    
    const flag = flagSprites[index];
    if (flag) {
      bar.remove(flag);
      if (flag.geometry) flag.geometry.dispose();
      if (flag.material) {
        flag.material.dispose();
        if (flag.material.map) flag.material.map.dispose();
      }
    }
    
    // Kemudian hapus bar dari scene
    scene.remove(bar);
    if (bar.geometry) bar.geometry.dispose();
    if (bar.material) {
      bar.material.dispose();
      if (bar.material.map) bar.material.map.dispose();
    }
  }
  
  // Hapus dari array
  bars[index] = null;
  labels[index] = null;
  flagSprites[index] = null;
}
  // Komentar atau hapus baris kode berikut untuk mencegah penghapusan bar yang terlewat
  // Destroy bar yang sudah terlewat (misal >5 dari current)
  // for (let i = 0; i < currentRank.value - 5; i++) {
  //   destroyBar(i);
  // }
  
  // Directly position camera to focus on canvas label area (no movement from below)
  if (savedCameraPosition && savedControlsTarget && currentBar && currentBar.geometry) {
    // Calculate target position for the current bar's label area
    const finalBarHeight = (currentBar.geometry as THREE.BoxGeometry).parameters.height;
    const finalBarY = finalBarHeight / 2 - 5;
    const finalLabelYPosition = finalBarY + (finalBarHeight / 2 - 0.5) - 1.5; // Turunkan fokus kamera sedikit
    const nextTargetPosition = new THREE.Vector3(
      currentBar.position.x,
      finalLabelYPosition,
      savedControlsTarget.z
    );
    
    // Calculate camera position with height adjustment for tall bars
    const cameraOffset = new THREE.Vector3().subVectors(savedCameraPosition, savedControlsTarget);
    
    // For first bar, use user's camera setup. For tall bars, adjust camera height
    let adjustedCameraY = savedCameraPosition.y;
    if (currentRank.value > 0) { // Skip adjustment for first bar
      // Calculate height adjustment based on bar height
      const maxBarHeight = Math.max(...bars.filter(b => b).map(b => (b.geometry as THREE.BoxGeometry).parameters.height));
      const heightRatio = finalBarHeight / maxBarHeight;
      
      // Only adjust for bars that are significantly tall (above 60% of max height)
      if (heightRatio > 0.6) {
        const heightAdjustment = (heightRatio - 0.6) * (finalBarHeight * 0.3); // Gradual height increase
        adjustedCameraY = savedCameraPosition.y + heightAdjustment;
      }
    }
    
    const nextCameraPosition = new THREE.Vector3(
      nextTargetPosition.x + cameraOffset.x,
      adjustedCameraY, // Use adjusted height for tall bars
      nextTargetPosition.z + cameraOffset.z
    );
    
    // Smoothly move camera to focus on label area (slow and gentle)
    gsap.to(controls.target, {
      x: nextTargetPosition.x,
      y: nextTargetPosition.y,
      z: nextTargetPosition.z,
      duration: 1.8, // Slower, gentler camera movement
      ease: 'power1.inOut', // Gentler easing
      onUpdate: () => {
        controls.update();
      }
    });
    
    gsap.to(camera.position, {
      x: nextCameraPosition.x,
      y: nextCameraPosition.y, // Include Y position for height adjustment
      z: nextCameraPosition.z,
      duration: 1.8, // Slower, gentler camera movement
      ease: 'power1.inOut', // Gentler easing
      onUpdate: () => {
        camera.lookAt(controls.target);
        camera.updateProjectionMatrix();
      },
      onComplete: () => {
        // After camera gently reaches label position, start bar animation
        startBarAnimation();
      }
    });
  } else {
    // If no saved camera position, start animation immediately
    startBarAnimation();
  }
  
  // Function to start the actual bar growth animation
  const startBarAnimation = () => {
    // Make current bar visible
    currentBar.visible = true;
  
    // Animate bar growth with simple growth effect - slower animation
    const originalHeight = currentBar.scale.y;
    currentBar.scale.y = 0.01; // Start with almost zero height
    gsap.to(currentBar.scale, {
      y: originalHeight,
      duration: 2.5, // Increased duration for much slower animation
      ease: 'power1.inOut', // Smoother easing for more gradual animation
      onUpdate: () => {
        // Update bar position to keep it grounded
        const newHeight = (currentBar.geometry as THREE.BoxGeometry).parameters.height * currentBar.scale.y;
        currentBar.position.y = newHeight / 2 - 5;
      }
    });
  
    // Show flag with increased delay to match slower animation
    if (currentFlag) {
      setTimeout(() => {
        currentFlag.visible = true;
      }, 1000); // Increased delay to match slower bar animation
    }
  
    // Show label with fade-in after bar animation
    if (currentLabel) {
      setTimeout(() => {
        currentLabel.visible = true;
      }, 1500); // Increased delay to match slower bar animation
    }
  
    // Add spotlight on the current bar
    const spotlight = new THREE.SpotLight(0xffffff, 2);
    spotlight.position.set(currentBar.position.x, 15, 10);
    spotlight.target = currentBar;
    spotlight.angle = Math.PI / 6;
    spotlight.penumbra = 0.2;
    spotlight.decay = 1.5;
    spotlight.distance = 40;
    spotlight.castShadow = true;
    scene.add(spotlight);
  
    // Camera stays focused on label canvas area - no movement during bar growth
    // Keep camera target fixed at the final label position while bar grows
    if (savedCameraPosition && savedControlsTarget) {
      // Calculate final label position and keep camera focused there
      const finalBarHeight = (currentBar.geometry as THREE.BoxGeometry).parameters.height;
      const finalBarY = finalBarHeight / 2 - 5;
      const finalLabelYPosition = finalBarY + (finalBarHeight / 2 - 0.5) - 1.5; // Turunkan fokus kamera sedikit
      
      // Keep camera target fixed at label canvas area - no following movement
      controls.target.y = finalLabelYPosition;
      controls.update();
    }
  }; // End of startBarAnimation function
  
  // Increment rank for next bar
  currentRank.value++;
  
  // Calculate delay based on current bar height for slower pacing
  const currentBarHeight = currentBar && currentBar.geometry && currentBar.geometry.parameters ? 
    (currentBar.geometry as THREE.BoxGeometry).parameters.height : 0;
  
  // Pastikan hanya bar dengan geometry valid yang digunakan dalam perhitungan
  const validBars = bars.filter(b => b && b.geometry && b.geometry.parameters);
  const maxBarHeight = validBars.length > 0 ? 
    Math.max(...validBars.map(b => (b.geometry as THREE.BoxGeometry).parameters.height)) : 1;
  
  const normalizedBarHeight = currentBarHeight / maxBarHeight;
  const baseDelay = 3500; // Increased base delay for slower transitions
  const maxDelay = 5500; // Increased maximum delay for slower flow
  const dynamicDelay = baseDelay + (normalizedBarHeight * (maxDelay - baseDelay));
  
  // Wait with dynamic delay before processing next bar
  setTimeout(() => {
    processNextBar();
  }, dynamicDelay); // Dynamic delay based on bar height
};

// Transition to lineup view after spotlight animation with cinematic camera movement
const transitionToLineupView = () => {
  // Auto-adjust camera for animation if no saved position
  if (!savedCameraPosition) {
    adjustCameraForBars();
  }
  
  // Make all bars visible with staggered animation
  bars.forEach((bar, index) => {
    if (bar) { // Add null check
      bar.visible = true;
      
      // Add a slight staggered delay for a wave-like effect
      setTimeout(() => {
        // Show flags with smooth fade
        if (index < flagSprites.length && flagSprites[index]) {
          flagSprites[index].visible = true;
        }
        
        // Show labels with smooth fade
        if (index < labels.length && labels[index]) {
          labels[index].visible = true;
        }
      }, index * 100); // Staggered delay based on index
    }
  });
  
  // After spotlight animation, wait 1 second then start preview mode directly
  setTimeout(() => {
    startPreviewMode();
  }, 1000); // Wait 1 second after spotlight animation completes
  
  // Enable controls for user interaction
  controls.enabled = true;
  animationInProgress = false;
  emit('animation-status', false);
  isSpotlightMode.value = false;
  
  // Start auto-rotation if enabled
  if (isAutoCamEnabled.value) {
    startAutoCam();
  }
};

// Start preview mode - pull camera back for wide overview
const startPreviewMode = () => {
  if (!savedCameraPosition || !savedControlsTarget) return;
  
  // Calculate preview camera position - moderately back and higher (zoom diperbesar dikit)
  const previewDistance = 60; // Moderately back (reduced from 80 to 60)
  const previewHeight = savedCameraPosition.y + 12; // Slightly higher up (reduced from 15 to 12)
  
  // Calculate direction from target to current camera
  const direction = new THREE.Vector3().subVectors(savedCameraPosition, savedControlsTarget).normalize();
  
  // New preview position
  const previewPosition = new THREE.Vector3(
    savedControlsTarget.x + direction.x * previewDistance,
    previewHeight,
    savedControlsTarget.z + direction.z * previewDistance
  );
  
  // Animate camera to preview position
  gsap.to(camera.position, {
    x: previewPosition.x,
    y: previewPosition.y,
    z: previewPosition.z,
    duration: 3.0, // Slow zoom out
    ease: 'power2.inOut',
    onUpdate: () => {
      camera.lookAt(controls.target);
      camera.updateProjectionMatrix();
    },
    onComplete: () => {
      // Start slow rotation around the chart
      startPreviewRotation();
    }
  });
};

// Start slow rotation for preview mode
const startPreviewRotation = () => {
  if (!savedControlsTarget) return;
  
  const radius = camera.position.distanceTo(savedControlsTarget);
  const centerX = savedControlsTarget.x;
  const centerZ = savedControlsTarget.z;
  const height = camera.position.y;
  
  // Slow rotation animation
  gsap.to({}, {
    duration: 20, // 20 seconds for full rotation
    repeat: -1,
    ease: 'none',
    onUpdate: function() {
      const progress = this.progress();
      const angle = progress * Math.PI * 2;
      
      camera.position.x = centerX + Math.cos(angle) * radius;
      camera.position.z = centerZ + Math.sin(angle) * radius;
      camera.position.y = height;
      
      camera.lookAt(savedControlsTarget);
      camera.updateProjectionMatrix();
      controls.update();
    }
  });
};

// Show all bars at once (lineup view) with smooth transitions
const showLineupView = () => {
  // Auto-adjust camera for animation if no saved position
  if (!savedCameraPosition) {
    adjustCameraForBars();
  }
  
  // Make all bars visible with staggered animation
  bars.forEach((bar, index) => {
    bar.visible = true;
    
    // Add a slight staggered delay for a wave-like effect
    setTimeout(() => {
      // Show flags with smooth fade
      if (index < flagSprites.length) {
        flagSprites[index].visible = true;
      }
      
      // Show labels with smooth fade
      if (index < labels.length) {
        labels[index].visible = true;
      }
    }, index * 80); // Staggered delay based on index, slightly faster than in transitionToLineupView
  });
  
  // If we have saved camera position and target from setup mode, restore them with smooth animation
  if (savedCameraPosition && savedControlsTarget) {
    // Animate camera position and target to saved values
    gsap.to(camera.position, {
      x: savedCameraPosition.x,
      y: savedCameraPosition.y,
      z: savedCameraPosition.z,
      duration: 2.2, // Slow, cinematic camera movement (extended 10%)
      ease: 'power1.inOut',
      onUpdate: () => camera.updateProjectionMatrix()
    });
    
    gsap.to(controls.target, {
      x: savedControlsTarget.x,
      y: savedControlsTarget.y,
      z: savedControlsTarget.z,
      duration: 2.2, // Match camera animation duration (extended 10%)
      ease: 'power1.inOut',
      onUpdate: () => controls.update()
    });
  }
  
  // Enable controls for user interaction
  controls.enabled = true;
  
  // Start auto-rotation if enabled
  if (isAutoCamEnabled.value) {
    startAutoCam();
  }
};

// Start auto camera rotation - slower and smoother
const startAutoCam = () => {
  if (autoCamAnimation) {
    autoCamAnimation.kill();
  }
  
  // Get the current camera distance from target
  const currentDistance = camera.position.distanceTo(controls.target);
  
  // Create a much slower and smoother orbit animation that maintains the same distance
  autoCamAnimation = gsap.to(camera.position, {
    duration: 40, // Much longer duration for slower rotation
    x: () => controls.target.x + currentDistance * Math.sin(Date.now() * 0.00005), // Slower rotation speed
    z: () => controls.target.z + currentDistance * Math.cos(Date.now() * 0.00005), // Slower rotation speed
    y: camera.position.y, // Keep the same height
    repeat: -1,
    ease: 'power1.inOut', // Smoother easing for more cinematic feel
    onUpdate: () => {
      camera.lookAt(controls.target);
      // Ensure smooth camera movement
      camera.updateProjectionMatrix();
      controls.update();
    }
  });
};

// Stop auto camera rotation
const stopAutoCam = () => {
  if (autoCamAnimation) {
    autoCamAnimation.kill();
    autoCamAnimation = null;
  }
};

// Update fog settings berdasarkan jumlah data
const updateFog = () => {
  if (!scene || !scene.fog) return;
  
  const skyFogColor = new THREE.Color(0x87CEEB);
  
  // Sesuaikan fog intensity berdasarkan jumlah bar
  const dataLength = props.data?.data?.length || 0;
  let adjustedFogIntensity = fogIntensity.value;
  
  // Untuk data yang banyak, kurangi fog intensity
  if (dataLength > 20) {
    adjustedFogIntensity = fogIntensity.value * 0.3; // Kurangi 70%
  } else if (dataLength > 10) {
    adjustedFogIntensity = fogIntensity.value * 0.6; // Kurangi 40%
  }
  
  if (fogType.value === 'linear') {
    // Linear fog: near and far distances
    const near = 10;
    const far = 150 + (dataLength * 5); // Perluas jarak fog untuk data banyak
    scene.fog = new THREE.Fog(skyFogColor, near, far);
  } else {
    // Exponential fog: density-based
    scene.fog = new THREE.FogExp2(skyFogColor, adjustedFogIntensity);
  }
};

// Toggle fog controls visibility
const toggleFogControls = () => {
  showFogControls.value = !showFogControls.value;
};

// Restart animation
const restartAnimation = () => {
  // Stop auto camera if running
  stopAutoCam();
  
  // Reset mode
  isSpotlightMode.value = true;
  
  // First, remove all bars from scene with proper cleanup
  bars.forEach(bar => {
    if (bar) {
      scene.remove(bar);
      // Dispose geometry and material
      if (bar.geometry) bar.geometry.dispose();
      if (bar.material) {
        if (Array.isArray(bar.material)) {
          bar.material.forEach(mat => mat.dispose());
        } else {
          bar.material.dispose();
        }
      }
    }
  });
  
  // Remove all existing labels from scene with proper cleanup
  labels.forEach(label => {
    if (label && label.parent) {
      label.parent.remove(label);
    }
    if (label) {
      // Dispose texture and material
      if (label.userData.texture) {
        label.userData.texture.dispose();
      }
      if (label.material) {
        if (Array.isArray(label.material)) {
          label.material.forEach((mat: THREE.Material) => mat.dispose());
        } else {
          label.material.dispose();
        }
      }
      label.visible = false;
    }
  });
  
  // Remove all flags with proper cleanup
  flagSprites.forEach(flag => {
    if (flag && flag.parent) {
      flag.parent.remove(flag);
    }
    if (flag) {
      // Dispose texture and material
      if (flag.userData && flag.userData.texture) {
        flag.userData.texture.dispose();
      }
      if (flag.material) {
        if (Array.isArray(flag.material)) {
          flag.material.forEach((mat: THREE.Material) => mat.dispose());
        } else {
          flag.material.dispose();
        }
      }
    }
  });
  
  // Dispose separate textures
  if (labelTextures) {
    labelTextures.forEach(texture => texture.dispose());
  }
  if (flagTextures) {
    flagTextures.forEach(texture => texture.dispose());
  }
  
  // Clear arrays
  bars = [];
  labels = [];
  flagSprites = [];
  
  // Canvas-based sprites don't need DOM cleanup like CSS2D
  
  // If we have saved camera position, smoothly transition back to starting position before restarting
  if (savedCameraPosition && savedControlsTarget) {
    // Calculate starting position for first bar (similar to processNextBar logic)
    const firstBarX = -((props.data.length - 1) * 12) / 2; // Position of first bar
    const startingTargetPosition = new THREE.Vector3(
      firstBarX,
      savedControlsTarget.y,
      savedControlsTarget.z
    );
    
    // Calculate camera position that maintains the same height and distance
    const cameraOffset = new THREE.Vector3().subVectors(savedCameraPosition, savedControlsTarget);
    const startingCameraPosition = new THREE.Vector3(
      startingTargetPosition.x + cameraOffset.x,
      savedCameraPosition.y,
      startingTargetPosition.z + cameraOffset.z
    );
    
    // Smoothly move camera to starting position
    gsap.to(controls.target, {
      x: startingTargetPosition.x,
      y: startingTargetPosition.y,
      z: startingTargetPosition.z,
      duration: 2.0, // Smooth transition duration
      ease: 'power2.inOut',
      onUpdate: () => controls.update()
    });
    
    gsap.to(camera.position, {
      x: startingCameraPosition.x,
      y: startingCameraPosition.y,
      z: startingCameraPosition.z,
      duration: 2.0, // Smooth transition duration
      ease: 'power2.inOut',
      onUpdate: () => {
        camera.lookAt(controls.target);
        camera.updateProjectionMatrix();
      },
      onComplete: () => {
        // After smooth transition, recreate bars and start animation
        createBars();
      }
    });
  } else {
    // If no saved camera position, recreate bars immediately
    createBars();
  }
};

// Handle window resize
const onWindowResize = () => {
  if (!chartContainer.value) return;
  
  const width = chartContainer.value.clientWidth;
  
  // Maintain fixed 16:9 aspect ratio for camera
  camera.aspect = 16 / 9;
  camera.updateProjectionMatrix();
  
  // Set renderer size to maintain 16:9 aspect ratio
  renderer.setSize(width, width * (9/16));
  
  // Re-adjust camera if bars exist and not in camera setup mode
  if (bars.length > 0 && !isCameraSetupMode.value && !savedCameraPosition) {
    adjustCameraForBars();
  }
  // labelRenderer no longer needed
};

// Setup frustum for visibility culling
const frustum = new THREE.Frustum();
const matrix = new THREE.Matrix4();

// Animation loop with optimized recording performance
let lastProgressUpdate = 0;
const animate = () => {
  requestAnimationFrame(animate);
  
  // Keep controls enabled for camera setup mode
  if (isCameraSetupMode.value) {
    controls.enabled = true;
  }
  
  // Reduce controls update frequency during recording for better performance
  if (!isRecording.value || Date.now() - lastProgressUpdate > 100) {
    controls.update();
  }
  
  // Update frustum for visibility culling
  matrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
  frustum.setFromProjectionMatrix(matrix);
  
  // Check each bar for visibility and manage textures
  bars.forEach((bar, index) => {
    // Skip if bar, label, or flagSprite is null (already destroyed)
    if (!bar || !bar.geometry) return;
  });
  
  renderer.render(scene, camera);
  // labelRenderer no longer needed - sprites render automatically
  
  // Capture frame if recording
  if (isRecording.value && capturer) {
    capturer.capture(renderer.domElement);
    
    // Update recording progress less frequently to reduce lag
    const currentTime = Date.now();
    if (currentTime - lastProgressUpdate > 200) { // Update progress every 200ms instead of every frame
      const elapsed = currentTime - recordingStartTime;
      recordingProgress.value = Math.min(100, Math.floor((elapsed / recordingDuration) * 100));
      lastProgressUpdate = currentTime;
      
      // Auto-stop recording when duration is reached
      if (elapsed >= recordingDuration) {
        stopRecording();
      }
    }
  }
  
  // If in camera setup mode, don't run animations
  // Just render the scene with controls enabled
  if (isCameraSetupMode.value) {
    // Keep controls enabled for camera positioning
    controls.enabled = true;
  }
};

// Start recording
const startRecording = () => {
  if (isRecording.value) return;
  
  // Get quality settings based on user selection
  const qualitySettings = {
    low: { framerate: 24, quality: 70, pixelRatio: 0.75, shadowMapSize: 512 },
    medium: { framerate: 30, quality: 85, pixelRatio: 1, shadowMapSize: 1024 },
    high: { framerate: 60, quality: 95, pixelRatio: Math.min(window.devicePixelRatio, 2), shadowMapSize: 2048 }
  };
  
  const settings = qualitySettings[recordingQuality.value as keyof typeof qualitySettings];
  
  // Store original settings
  originalPixelRatio = renderer.getPixelRatio();
  originalShadowMapSize = renderer.shadowMap.enabled ? 2048 : 0;
  
  // Apply quality-specific renderer settings
  renderer.setPixelRatio(settings.pixelRatio);
  
  // Apply quality-specific shadow map size
  if (renderer.shadowMap.enabled) {
    scene.traverse((child) => {
      if (child instanceof THREE.DirectionalLight && child.shadow) {
        child.shadow.mapSize.width = settings.shadowMapSize;
        child.shadow.mapSize.height = settings.shadowMapSize;
        child.shadow.map?.dispose();
        child.shadow.map = null;
      }
    });
  }
  
  // Initialize CCapture with quality-specific settings
  capturer = new CCapture({
    format: 'webm',
    framerate: settings.framerate,
    verbose: false,
    display: false, // Disable display overlay to reduce lag
    quality: settings.quality,
    name: 'cobastat-animation',
    workersPath: '/lib/', // Specify workers path for better performance
    timeLimit: 0, // No time limit
    autoSaveTime: 0 // Disable auto-save during recording
  });
  
  // Start recording
  capturer.start();
  isRecording.value = true;
  recordingStartTime = Date.now();
  recordingProgress.value = 0;
  
  // Set recording duration based on animation mode (extended 10%)
  // For spotlight mode, calculate approximate duration based on number of bars
  if (isSpotlightMode.value) {
    // Each bar takes about 3 seconds to animate + 1 second pause (extended 10%)
    recordingDuration = Math.round((bars.length * 4 + 3) * 1000 * 1.1); // Add 3 seconds for final transition, extended 10%
  } else {
    // For lineup mode, use a fixed duration (extended 10%)
    recordingDuration = 11000; // 11 seconds (10% longer)
  }
  
  // Restart animation to capture from beginning
  restartAnimation();
};

// Stop recording and save video
const stopRecording = () => {
  if (!isRecording.value || !capturer) return;
  
  // Stop recording
  capturer.stop();
  isRecording.value = false;
  recordingProgress.value = 100;
  
  // Restore original rendering quality
  renderer.setPixelRatio(originalPixelRatio);
  
  // Restore shadow map size
  if (originalShadowMapSize > 0) {
    scene.traverse((child) => {
      if (child instanceof THREE.DirectionalLight && child.shadow) {
        child.shadow.mapSize.width = originalShadowMapSize;
        child.shadow.mapSize.height = originalShadowMapSize;
        child.shadow.map?.dispose();
        child.shadow.map = null;
      }
    });
  }
  
  // Save the video
  capturer.save((blob: Blob) => {
    // Create download link
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'cobastat-animation.webm';
    document.body.appendChild(a);
    a.click();
    
    // Clean up
    setTimeout(() => {
      document.body.removeChild(a);
      window.URL.revokeObjectURL(url);
    }, 100);
  });
  
  // Reset capturer
  capturer = null;
};

// Watch for custom background changes
watch(() => props.customBackgrounds, (newBackgrounds) => {
  if (!scene) return;
  
  // Check if we already have this sky texture in the pool
  if (!flagTexturePool.has(newBackgrounds.sky)) {
    // Load sky background texture
    const skyTextureLoader = new THREE.TextureLoader();
    const skyTexture = skyTextureLoader.load(newBackgrounds.sky);
    flagTexturePool.set(newBackgrounds.sky, skyTexture);
    scene.background = skyTexture;
  } else {
    // Use existing texture from pool
    scene.background = flagTexturePool.get(newBackgrounds.sky)!;
  }
  
  // Update floor texture
  if (floor) {
    const floorMat = (floor.material as THREE.MeshBasicMaterial);
    
    // Check if we already have this floor texture in the pool
    if (!flagTexturePool.has(newBackgrounds.floor)) {
      const floorTextureLoader = new THREE.TextureLoader();
      const floorTexture = floorTextureLoader.load(newBackgrounds.floor);
      
      // Set texture repeat to cover large floor area
      floorTexture.wrapS = THREE.RepeatWrapping;
      floorTexture.wrapT = THREE.RepeatWrapping;
      floorTexture.repeat.set(10, 10);
      
      flagTexturePool.set(newBackgrounds.floor, floorTexture);
      floorMat.map = floorTexture;
    } else {
      // Use existing texture from pool
      floorMat.map = flagTexturePool.get(newBackgrounds.floor)!;
    }
    
    floorMat.needsUpdate = true;
  }
}, { deep: true });

// Watch for bar texture changes
watch(() => props.barTexture, (newTexture) => {
  if (!scene) return;

  // Jika data banyak, pakai warna solid saja
  if (bars.length > 50) {
    // For very large datasets, always use solid colors
    bars.forEach((bar, index) => {
      if (bar && bar.material) {
        const material = bar.material as THREE.MeshLambertMaterial;
        // Set warna sesuai ranking
        if (props.data && props.data.data) {
          const sortedData = [...props.data.data].sort((a, b) => b.ranking - a.ranking);
          const maxRanking = Math.max(...sortedData.map(d => d.ranking));
          const item = sortedData[index];
          let barColor;
          if (item.ranking === 1) {
            barColor = new THREE.Color(0x800000);
          } else if (item.ranking === 2) {
            barColor = new THREE.Color(0xCC5500);
          } else {
            const invertedRanking = maxRanking - item.ranking + 1;
            const normalizedRanking = invertedRanking / maxRanking;
            const defaultBarColors = { low: 0x4CAF50, mid: 0xFF9800, high: 0xF44336 };
            const lowColor = new THREE.Color(defaultBarColors.low);
            const midColor = new THREE.Color(defaultBarColors.mid);
            const highColor = new THREE.Color(defaultBarColors.high);
            if (normalizedRanking < 0.5) {
              barColor = new THREE.Color().lerpColors(lowColor, midColor, normalizedRanking * 2);
            } else {
              barColor = new THREE.Color().lerpColors(midColor, highColor, (normalizedRanking - 0.5) * 2);
            }
          }
          material.map = null;
          material.color.copy(barColor);
        }
        material.needsUpdate = true;
      }
    });
    return;
  } else if (bars.length > 16 || !newTexture) {
    // Jika data banyak, pakai warna solid saja
    bars.forEach((bar, index) => {
      if (bar && bar.material) {
        const material = bar.material as THREE.MeshLambertMaterial;
        // Set warna sesuai ranking
        if (props.data && props.data.data) {
          const sortedData = [...props.data.data].sort((a, b) => b.ranking - a.ranking);
          const maxRanking = Math.max(...sortedData.map(d => d.ranking));
          const item = sortedData[index];
          let barColor;
          if (item.ranking === 1) {
            barColor = new THREE.Color(0x800000);
          } else if (item.ranking === 2) {
            barColor = new THREE.Color(0xCC5500);
          } else {
            const invertedRanking = maxRanking - item.ranking + 1;
            const normalizedRanking = invertedRanking / maxRanking;
            const defaultBarColors = { low: 0x4CAF50, mid: 0xFF9800, high: 0xF44336 };
            const lowColor = new THREE.Color(defaultBarColors.low);
            const midColor = new THREE.Color(defaultBarColors.mid);
            const highColor = new THREE.Color(defaultBarColors.high);
            if (normalizedRanking < 0.5) {
              barColor = new THREE.Color().lerpColors(lowColor, midColor, normalizedRanking * 2);
            } else {
              barColor = new THREE.Color().lerpColors(midColor, highColor, (normalizedRanking - 0.5) * 2);
            }
          }
          material.map = null;
          material.color.copy(barColor);
        }
        material.needsUpdate = true;
      }
    });
    return;
  }

  // Jika data sedikit, pakai satu instance texture untuk semua bar
  let sharedTexture: THREE.Texture;
  if (!flagTexturePool.has(newTexture)) {
    const textureLoader = new THREE.TextureLoader();
    sharedTexture = textureLoader.load(newTexture);
    sharedTexture.wrapS = THREE.ClampToEdgeWrapping;
    sharedTexture.wrapT = THREE.RepeatWrapping;
    flagTexturePool.set(newTexture, sharedTexture);
  } else {
    sharedTexture = flagTexturePool.get(newTexture)!;
  }

  bars.forEach((bar) => {
    if (bar && bar.material) {
      const material = bar.material as THREE.MeshLambertMaterial;
      material.map = sharedTexture;
      material.color.setHex(0xffffff);
      material.needsUpdate = true;
    }
  });
});

// Watch for data changes
watch(() => props.data, () => {
  // Reset to spotlight mode for new data
  isSpotlightMode.value = true;
  
  // Stop any ongoing animations
  stopAutoCam();
  
  // Reset sharedBarTexture when data changes
  sharedBarTexture = null;
  
  // Recreate bars and title when data changes
  if (scene) {
    createBars();
    createTitleText();
  }
}, { deep: true });

// Watch for spotlight mode changes
watch(isSpotlightMode, (newMode) => {
  // Recreate bars when mode changes
  if (scene && !animationInProgress) {
    createBars();
  }
});

// Watch for auto camera mode changes
watch(isAutoCamEnabled, (enabled) => {
  if (enabled && !animationInProgress) {
    startAutoCam();
  } else {
    stopAutoCam();
  }
});

// Watch for fog changes
watch([fogIntensity, fogType], () => {
  updateFog();
}, { immediate: false });

// Start animation from camera setup mode
const startAnimation = () => {
  // Save current camera position and controls target before switching modes
  savedCameraPosition = camera.position.clone();
  savedControlsTarget = controls.target.clone();
  
  // Switch from camera setup mode to animation mode
  isCameraSetupMode.value = false;
  isAnimationStarted.value = true;
  
  // Clean up existing labels before restarting animation
  // First, remove all bars from scene
  bars.forEach(bar => {
    if (bar) {
      scene.remove(bar);
    }
  });
  
  // Remove all existing labels from scene and DOM
  labels.forEach(label => {
    if (label && label.parent) {
      label.parent.remove(label);
    }
    // Hide sprite labels
      if (label) {
        label.visible = false;
      }
  });
  
  // Remove all flags
  flagSprites.forEach(flag => {
    if (flag && flag.parent) {
      flag.parent.remove(flag);
    }
  });
  
  // Canvas-based sprites are automatically cleaned up with scene
  
  // Clear arrays
  bars = [];
  labels = [];
  flagSprites = [];
  
  // Restart animation with current camera position
  createBars();
  createTitleText();
};

// Reset to camera setup mode
const resetCameraSetup = () => {
  // Switch back to camera setup mode
  isCameraSetupMode.value = true;
  isAnimationStarted.value = false;
  
  // Reset saved camera position and target
  savedCameraPosition = null;
  savedControlsTarget = null;
  
  // Recreate bars and title in setup mode
  createBars();
  createTitleText();
  
  // Make sure controls are enabled
  controls.enabled = true;
};

// Update bar heights when scale factor changes
const updateBarHeights = () => {
  if (!scene || bars.length === 0) return;
  
  // Preserve current texture state before recreating bars
  const currentTextureState = {
    labelTextures: [...labelTextures],
    flagTextures: [...flagTextures],
    texturePool: new Map(texturePool),
    flagTexturePool: new Map(flagTexturePool)
  };
  
  // Recreate bars with new height scale
  createBars();
  
  // Restore texture state if textures were lost
  if (labelTextures.length === 0 && currentTextureState.labelTextures.length > 0) {
    labelTextures = currentTextureState.labelTextures;
    flagTextures = currentTextureState.flagTextures;
    texturePool = currentTextureState.texturePool;
    flagTexturePool = currentTextureState.flagTexturePool;
    
    // Reapply textures to bars
    bars.forEach((bar, index) => {
      if (index < labels.length && labels[index]) {
        const labelMaterial = labels[index].material as THREE.MeshBasicMaterial;
        if (labelMaterial && index < labelTextures.length) {
          labelMaterial.map = labelTextures[index];
          labelMaterial.needsUpdate = true;
        }
      }
      
      if (index < flagSprites.length && flagSprites[index]) {
        const flagMaterial = flagSprites[index].material as THREE.MeshBasicMaterial;
        if (flagMaterial && index < flagTextures.length) {
          flagMaterial.map = flagTextures[index];
          flagMaterial.needsUpdate = true;
        }
      }
    });
  }
};

// Add texture monitoring system to prevent 14-texture limit
const monitorTextureUsage = () => {
  if (!renderer) return;
  
  const gl = renderer.getContext();
  const maxTextures = gl.getParameter(gl.MAX_TEXTURE_IMAGE_UNITS);
  const currentTextureCount = labelTextures.length + flagTextures.length;
  
  // If approaching texture limit, force simplified textures
  if (currentTextureCount > maxTextures * 0.7) { // 70% of limit (more conservative)
    console.warn(`Texture usage high: ${currentTextureCount}/${maxTextures}. Switching to simplified textures.`);
    
    // Force recreation with simplified textures
    if (bars.length > 0) {
      createBars();
    }
  }
};

// Force texture cleanup to prevent 14-texture limit
const forceTextureCleanup = () => {
  // Dispose all existing textures
  labelTextures.forEach(texture => texture.dispose());
  flagTextures.forEach(texture => texture.dispose());
  texturePool.forEach(texture => texture.dispose());
  flagTexturePool.forEach(texture => texture.dispose());
  
  // Clear arrays and pools
  labelTextures = [];
  flagTextures = [];
  texturePool.clear();
  flagTexturePool.clear();
  
  // Force garbage collection hint
  if (window.gc) {
    window.gc();
  }
};

// Initialize on mount
onMounted(() => {
  initScene();
  
  // Cleanup on unmount
  return () => {
    window.removeEventListener('resize', onWindowResize);
    
    // Stop animations
    if (autoCamAnimation) {
      autoCamAnimation.kill();
    }
    
    cleanupResources();
    
    // Dispose of Three.js resources
    if (renderer) {
      renderer.dispose();
    }
    
    // Canvas-based sprites cleanup automatically with scene
    
    // Clear scene
    if (scene) {
      scene.clear();
    }
  };
});

// Helper: get shared bar texture (for <= 1 instance)
let sharedBarTexture: THREE.Texture | null = null;
function getSharedBarTexture() {
  if (!sharedBarTexture && props.barTexture) {
    const textureLoader = new THREE.TextureLoader();
    sharedBarTexture = textureLoader.load(props.barTexture);
    sharedBarTexture.wrapS = THREE.ClampToEdgeWrapping;
    sharedBarTexture.wrapT = THREE.RepeatWrapping;
  }
  return sharedBarTexture;
}

// Dynamic bar texture assignment: only N bars around currentRank get texture, others get solid color
function updateBarTexturesDynamic(currentIdx: number, windowSize: number = 7) {
  // windowSize: total bars with texture (e.g. 7 means 3 before, current, 3 after)
  const half = Math.floor(windowSize / 2);
  const barTexture = getSharedBarTexture();
  bars.forEach((bar, i) => {
    if (!bar || !bar.material) return;
    
    const mat = bar.material as THREE.MeshLambertMaterial;
    if (Math.abs(i - currentIdx) <= half && barTexture && props.barTexture) {
      // Assign texture only if not already assigned
      if (mat.map !== barTexture) {
        mat.map = barTexture;
        mat.color.setHex(0xffffff);
        mat.needsUpdate = true;
      }
    } else {
      // Remove texture, use solid color
      if (mat.map) {
        mat.map = null;
        // Optionally, use a color based on ranking or default
        mat.color.setHex(0xcccccc);
        mat.needsUpdate = true;
      }
    }
  });
}

// Call this after each bar animation and camera move
// Example: in processNextBar, after animating bar, call:
// updateBarTexturesDynamic(currentRank.value, 7);

// ... existing code ...

// In processNextBar, after bar animation:
const startBarAnimation = () => {
  // ... existing code ...
  // After bar animation logic:
  updateBarTexturesDynamic(currentRank.value, 7); // Only 7 bars (3 before, current, 3 after) use texture
};
// ... existing code ...

</script>

<template>
  <div class="chart-container-wrapper">
    <div class="chart-wrapper">
      <div ref="chartContainer" class="chart-container">
        <!-- Camera setup overlay inside chart container -->
        <div v-if="isCameraSetupMode" class="camera-setup-overlay">
          <div class="camera-setup-info">
            <div class="setup-instructions">
              <h3>Camera Setup Mode</h3>
              <p>Atur posisi kamera sesuai keinginan Anda:</p>
              <ul>
                <li>Klik dan geser untuk memutar kamera</li>
                <li>Scroll untuk zoom in/out</li>
                <li>Klik kanan dan geser untuk menggeser kamera</li>
              </ul>
            </div>
            <button @click="startAnimation" class="control-btn start-animation-btn">
              <span class="icon">▶️</span> Mulai Animasi
            </button>
          </div>
        </div>
      </div>
    </div>
    
    <!-- Controls outside the chart container -->
    <div class="controls-container">
      <div class="chart-controls">
        <button @click="restartAnimation" class="control-btn">
          <span class="icon">↺</span> Restart
        </button>
        <button @click="isAutoCamEnabled = !isAutoCamEnabled" class="control-btn">
          <span class="icon">🔄</span> {{ isAutoCamEnabled ? 'Stop' : 'Start' }} Auto Cam
        </button>
        <button @click="isSpotlightMode = !isSpotlightMode; restartAnimation()" class="control-btn">
          <span class="icon">🔦</span> {{ isSpotlightMode ? 'Lineup' : 'Spotlight' }} Mode
        </button>
        <button @click="resetCameraSetup" class="control-btn">
          <span class="icon">🔧</span> Atur Kamera
        </button>
        <button @click="toggleFogControls" class="control-btn">
          <span class="icon">🌫️</span> Fog Effects
        </button>
        <div class="height-control">
          <label class="height-label">Tinggi Bar:</label>
          <input 
            type="range" 
            v-model.number="heightScaleFactor" 
            min="0.1" 
            max="2.0" 
            step="0.1" 
            class="height-slider"
            @input="updateBarHeights"
          />
          <span class="height-value">{{ heightScaleFactor.toFixed(1) }}x</span>
        </div>
        <div class="height-control">
          <label class="height-label">Kenaikan:</label>
          <input 
            type="range" 
            v-model.number="barHeightIncrement" 
            min="0.5" 
            max="5.0" 
            step="0.5" 
            class="height-slider"
            @input="updateBarHeights"
          />
          <span class="height-value">{{ barHeightIncrement.toFixed(1) }}</span>
        </div>
        <div class="height-control">
          <label class="height-label">Min Height:</label>
          <input 
            type="range" 
            v-model.number="minimumBarHeight" 
            min="1.0" 
            max="5.0" 
            step="0.5" 
            class="height-slider"
            @input="updateBarHeights"
          />
          <span class="height-value">{{ minimumBarHeight.toFixed(1) }}</span>
        </div>
        <div class="recording-controls">
          <select v-model="recordingQuality" class="quality-selector" :disabled="isRecording">
            <option value="low">Kualitas Rendah (Performa Tinggi)</option>
            <option value="medium">Kualitas Sedang</option>
            <option value="high">Kualitas Tinggi</option>
          </select>
          <button @click="isRecording ? stopRecording() : startRecording()" class="control-btn" :class="{ 'recording': isRecording }">
            <span class="icon">⏺</span> {{ isRecording ? 'Stop Recording' : 'Record Video' }}
          </button>
        </div>
        <div v-if="isRecording" class="recording-progress">
          <div class="progress-bar">
            <div class="progress-fill" :style="{ width: recordingProgress + '%' }"></div>
          </div>
          <div class="progress-text">{{ recordingProgress }}%</div>
        </div>
      </div>
      
      <!-- Fog Controls Panel -->
      <div v-if="showFogControls" class="fog-controls-panel">
        <h3>🌫️ Fog Effects</h3>
        <div class="fog-control-group">
          <label>Intensitas Fog:</label>
          <input 
            type="range" 
            v-model.number="fogIntensity" 
            min="0" 
            max="0.05" 
            step="0.001" 
            class="fog-slider"
          />
          <span class="fog-value">{{ fogIntensity.toFixed(3) }}</span>
        </div>
        <div class="fog-control-group">
          <label>Jenis Fog:</label>
          <select v-model="fogType" class="fog-type-selector">
            <option value="exponential">Exponential (Realistis)</option>
            <option value="linear">Linear (Gradual)</option>
          </select>
        </div>
        <div class="fog-presets">
          <button @click="fogIntensity = 0.003" class="preset-btn">Ringan</button>
          <button @click="fogIntensity = 0.008" class="preset-btn">Sedang</button>
          <button @click="fogIntensity = 0.02" class="preset-btn">Tebal</button>
          <button @click="fogIntensity = 0" class="preset-btn">Tidak Ada</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.chart-container-wrapper {
  display: flex;
  flex-direction: column;
  width: 100%;
  height: 100%;
  gap: 15px;
  padding: 20px;
  box-sizing: border-box;
}

.chart-wrapper {
  position: relative;
  width: 100%;
  aspect-ratio: 16/9;
  overflow: hidden;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  margin: 0 auto;
  max-width: 1200px;
}

.chart-container {
  width: 100%;
  height: 100%;
  position: relative;
  overflow: hidden;
  background: #121212;
  border-radius: 8px;
}

.controls-container {
  display: flex;
  justify-content: center;
  width: 100%;
  padding: 20px 0;
}

.chart-controls {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 10px;
  padding: 15px 0;
  width: 100%;
  flex-wrap: wrap;
}

.recording-controls {
  display: flex;
  align-items: center;
  gap: 8px;
}

.quality-selector {
  background: rgba(0, 0, 0, 0.7);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 15px;
  padding: 6px 12px;
  font-size: 12px;
  cursor: pointer;
  backdrop-filter: blur(5px);
  transition: all 0.2s ease;
  min-width: 180px;
}

.quality-selector:hover:not(:disabled) {
  background: rgba(0, 0, 0, 0.9);
  border-color: rgba(255, 255, 255, 0.4);
}

.quality-selector:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.quality-selector option {
  background: #333;
  color: white;
}

.control-btn {
  background: rgba(0, 0, 0, 0.7);
  color: white;
  border: none;
  border-radius: 20px;
  padding: 8px 15px;
  font-size: 14px;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 5px;
  backdrop-filter: blur(5px);
  transition: all 0.2s ease;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
}

.control-btn:hover {
  background: rgba(0, 0, 0, 0.9);
  transform: scale(1.05);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}

.control-btn.recording {
  background: rgba(255, 0, 0, 0.7);
  animation: pulse 1.5s infinite;
}

.control-btn.recording:hover {
  background: rgba(255, 0, 0, 0.9);
}

.start-animation-btn {
  background: rgba(0, 128, 0, 0.8);
  font-size: 14px;
  padding: 8px 16px;
  margin-top: 10px;
}

.start-animation-btn:hover {
  background: rgba(0, 150, 0, 0.9);
}

.camera-setup-overlay {
  position: absolute;
  top: 20px;
  right: 20px;
  z-index: 1000;
  pointer-events: none;
}

.camera-setup-info {
  background: rgba(0, 0, 0, 0.8);
  padding: 12px;
  border-radius: 8px;
  max-width: 280px;
  text-align: center;
  pointer-events: auto;
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.setup-instructions {
  color: white;
  margin-bottom: 10px;
}

.setup-instructions h3 {
  margin: 0 0 8px 0;
  font-size: 16px;
  color: #4CAF50;
}

.setup-instructions p {
  margin: 0 0 6px 0;
  font-size: 12px;
}

.setup-instructions ul {
  text-align: left;
  margin: 0;
  padding-left: 16px;
  font-size: 11px;
}

.setup-instructions li {
  margin-bottom: 2px;
}

@keyframes pulse {
  0% {
    box-shadow: 0 0 0 0 rgba(255, 0, 0, 0.4);
  }
  70% {
    box-shadow: 0 0 0 10px rgba(255, 0, 0, 0);
  }
  100% {
    box-shadow: 0 0 0 0 rgba(255, 0, 0, 0);
  }
}

.recording-progress {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-left: 10px;
  margin-top: 10px;
  width: 100%;
  max-width: 300px;
}

.progress-bar {
  width: 100%;
  height: 10px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 5px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #ff0000, #ff6b6b);
  border-radius: 5px;
  transition: width 0.3s ease;
}

.progress-text {
  color: white;
  font-size: 12px;
  min-width: 40px;
}

.icon {
  font-size: 16px;
}

/* Custom styles for bar labels */
:deep(.bar-label) {
  transform: translateX(-50%) scale(1) perspective(500px); /* Add perspective to match 3D view */
  transform-origin: center;
  white-space: normal;
  box-shadow: none; /* Remove shadow for more realistic look */
  border: none; /* Remove border for more realistic look */
  overflow: hidden;
  text-overflow: ellipsis;
  transform-style: preserve-3d; /* Enhance 3D appearance */
}

/* Fog Controls Panel */
.fog-controls-panel {
  background: rgba(0, 0, 0, 0.9);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 12px;
  padding: 20px;
  margin-top: 15px;
  backdrop-filter: blur(10px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  max-width: 400px;
  margin-left: auto;
  margin-right: auto;
}

.fog-controls-panel h3 {
  color: white;
  margin: 0 0 15px 0;
  text-align: center;
  font-size: 16px;
}

.fog-control-group {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 15px;
  flex-wrap: wrap;
}

.fog-control-group label {
  color: white;
  font-size: 14px;
  min-width: 100px;
  font-weight: 500;
}

.fog-slider {
  flex: 1;
  min-width: 150px;
  height: 6px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.fog-slider::-webkit-slider-thumb {
  appearance: none;
  width: 18px;
  height: 18px;
  background: #4CAF50;
  border-radius: 50%;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
  transition: all 0.2s ease;
}

.fog-slider::-webkit-slider-thumb:hover {
  background: #66BB6A;
  transform: scale(1.1);
}

.fog-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  background: #4CAF50;
  border-radius: 50%;
  cursor: pointer;
  border: none;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}

.fog-value {
  color: #4CAF50;
  font-weight: bold;
  font-size: 12px;
  min-width: 50px;
  text-align: center;
}

.fog-type-selector {
  flex: 1;
  background: rgba(255, 255, 255, 0.1);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 6px;
  padding: 8px 12px;
  font-size: 14px;
  cursor: pointer;
  min-width: 180px;
}

.fog-type-selector:focus {
  outline: none;
  border-color: #4CAF50;
  box-shadow: 0 0 0 2px rgba(76, 175, 80, 0.2);
}

.fog-type-selector option {
  background: #333;
  color: white;
}

.fog-presets {
  display: flex;
  gap: 8px;
  justify-content: center;
  flex-wrap: wrap;
  margin-top: 10px;
}

.preset-btn {
  background: rgba(76, 175, 80, 0.2);
  color: white;
  border: 1px solid rgba(76, 175, 80, 0.4);
  border-radius: 6px;
  padding: 6px 12px;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.2s ease;
  min-width: 60px;
}

.preset-btn:hover {
  background: rgba(76, 175, 80, 0.4);
  border-color: rgba(76, 175, 80, 0.6);
  transform: translateY(-1px);
}

.preset-btn:active {
  transform: translateY(0);
}

/* Height Control Styles */
.height-control {
  display: flex;
  align-items: center;
  gap: 8px;
  background: rgba(0, 0, 0, 0.7);
  border-radius: 15px;
  padding: 8px 12px;
  backdrop-filter: blur(5px);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.height-label {
  color: white;
  font-size: 12px;
  font-weight: 500;
  white-space: nowrap;
}

.height-slider {
  width: 80px;
  height: 4px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 2px;
  outline: none;
  cursor: pointer;
}

.height-slider::-webkit-slider-thumb {
  appearance: none;
  width: 16px;
  height: 16px;
  background: #4CAF50;
  border-radius: 50%;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}

.height-slider::-moz-range-thumb {
  width: 16px;
  height: 16px;
  background: #4CAF50;
  border-radius: 50%;
  cursor: pointer;
  border: none;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}

.height-value {
  color: #4CAF50;
  font-size: 12px;
  font-weight: bold;
  min-width: 30px;
  text-align: center;
}
</style>