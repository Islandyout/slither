import { useEffect, useRef, useState, useCallback } from "react";
import * as THREE from "three";

// ============================================================
// PWA — inject manifest + service worker + meta tags
// ============================================================
(function injectPWA() {
  if (document.getElementById("pwa-manifest")) return;

  // ── Generate PNG icon via Canvas (required by Chrome install criteria) ──
  function makePngDataUrl(size) {
    const c = document.createElement("canvas");
    c.width = c.height = size;
    const ctx = c.getContext("2d");
    // Background
    ctx.fillStyle = "#050a0f";
    ctx.fillRect(0, 0, size, size);
    // Rounded rect border
    ctx.strokeStyle = "#00ffcc";
    ctx.lineWidth = size * 0.04;
    const r = size * 0.14;
    ctx.beginPath();
    ctx.roundRect(size * 0.08, size * 0.08, size * 0.84, size * 0.84, r);
    ctx.stroke();
    // Snake emoji
    ctx.font = `${size * 0.55}px serif`;
    ctx.textAlign = "center";
    ctx.textBaseline = "middle";
    ctx.fillText("🐍", size / 2, size / 2);
    return c.toDataURL("image/png");
  }

  const icon192 = makePngDataUrl(192);
  const icon512 = makePngDataUrl(512);

  // Apple touch icon link
  const appleIcon = Object.assign(document.createElement("link"), {
    rel: "apple-touch-icon", href: icon192,
  });
  document.head.appendChild(appleIcon);

  // ── Manifest as blob URL with PNG data-URI icons ──────────
  const manifest = {
    name: "Slither.3D",
    short_name: "Slither3D",
    description: "Massive multiplayer 3D snake game",
    start_url: "./",
    scope: "./",
    display: "fullscreen",
    orientation: "landscape",
    background_color: "#050a0f",
    theme_color: "#00ffcc",
    icons: [
      { src: icon192, sizes: "192x192", type: "image/png", purpose: "any maskable" },
      { src: icon512, sizes: "512x512", type: "image/png", purpose: "any maskable" },
    ],
  };
  const manifestBlob = new Blob([JSON.stringify(manifest)], { type: "application/json" });
  const manifestUrl = URL.createObjectURL(manifestBlob);
  const manifestLink = Object.assign(document.createElement("link"), {
    id: "pwa-manifest", rel: "manifest", href: manifestUrl,
  });
  document.head.appendChild(manifestLink);

  // ── Meta tags ─────────────────────────────────────────────
  [
    { name: "theme-color",                        content: "#00ffcc" },
    { name: "apple-mobile-web-app-capable",       content: "yes" },
    { name: "apple-mobile-web-app-status-bar-style", content: "black-translucent" },
    { name: "apple-mobile-web-app-title",         content: "Slither.3D" },
    { name: "mobile-web-app-capable",             content: "yes" },
    { name: "application-name",                   content: "Slither.3D" },
  ].forEach(({ name, content }) => {
    const m = document.createElement("meta");
    m.name = name; m.content = content;
    document.head.appendChild(m);
  });

  // ── Service Worker via blob URL ───────────────────────────
  // Note: blob-URL SWs work in Chrome/Edge but not Safari.
  // For full cross-browser support, host a real sw.js file.
  const swCode = `
const CACHE = 'slither3d-v2';
const PRECACHE = ['./', './index.html'];
self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(PRECACHE)).then(() => self.skipWaiting()));
});
self.addEventListener('activate', e => {
  e.waitUntil(
    caches.keys().then(keys => Promise.all(
      keys.filter(k => k !== CACHE).map(k => caches.delete(k))
    )).then(() => self.clients.claim())
  );
});
self.addEventListener('fetch', e => {
  if (e.request.method !== 'GET') return;
  e.respondWith(
    caches.match(e.request).then(cached => {
      const fresh = fetch(e.request).then(res => {
        if (res && res.status === 200) {
          const clone = res.clone();
          caches.open(CACHE).then(c => c.put(e.request, clone));
        }
        return res;
      }).catch(() => cached);
      return cached || fresh;
    })
  );
});
  `;
  if ("serviceWorker" in navigator) {
    // Use real sw.js when running as top-level page (deployed), blob SW in iframe/dev
    if (window.self === window.top) {
      navigator.serviceWorker.register("./sw.js", { scope: "./" }).catch(() => {});
    } else {
      const swBlob = new Blob([swCode], { type: "application/javascript" });
      navigator.serviceWorker.register(URL.createObjectURL(swBlob), { scope: "./" }).catch(() => {});
    }
  }
})();

// ============================================================
// CONSTANTS
// ============================================================
const ARENA_SIZE = 2000;
const SEGMENT_RADIUS = 8;
const SEGMENT_SPACING = 12;
const PELLET_COUNT = 800;
const PELLET_RADIUS = 5;
const BASE_SPEED = 120;
const BOOST_SPEED = 240;
const TURN_SPEED = 2.8;
const MIN_SEGMENTS = 8;
const AI_COUNT_TARGET = 16;
const TICK_RATE = 15;
const HALF = ARENA_SIZE / 2;

// Spawn safety
const SPAWN_IMMUNITY_SECONDS = 4.0;
const SAFE_SPAWN_MIN_DISTANCE = 450;

// Power-up config
const POWERUP_COUNT = 12;
const POWERUP_TYPES = {
  speed:   { color: 0xffdd00, emissive: 0x886600, label: "⚡ SPEED BOOST",  duration: 6,  icon: "⚡" },
  ghost:   { color: 0x88aaff, emissive: 0x223388, label: "👻 GHOST MODE",   duration: 5,  icon: "👻" },
  magnet:  { color: 0xff44aa, emissive: 0x881133, label: "🧲 PELLET MAGNET", duration: 7, icon: "🧲" },
  freeze:  { color: 0x44ffff, emissive: 0x006666, label: "❄ FREEZE",        duration: 4,  icon: "❄️" },
};

// Smoother controls
const MOUSE_SMOOTH  = 0.14;   // lerp factor for raw mouse angle (lower = smoother)
const TURN_ACCEL    = 6.0;    // how fast turn rate builds up
const MAX_TURN_RATE = 3.2;    // radians/sec max
const JOY_DEAD_ZONE = 14;     // px dead zone radius

const FAKE_NAMES = [
  "KingVex","JamaicaBoss","NoScopeZed","LilKobra","ShadowMamba",
  "ViperX","CryptoSlith","NeonBite","GhostFang","BlazeSnek",
  "QuantumWorm","NightCrawlr","DrillFang","CobaltKing","RedViper",
  "ToxicTail","ZeroGravity","SlithPro","UltraFang","MegaWorm",
  "ByteSerpent","GlitchSnake","ChaosViper","StormCrawler","DarkMamba",
  "FlameKing","IceVenom","ThunderBolt","CyberFang","NeonViper",
  "SwiftSlith","DeathCoil","AcidBite","SteelSerpent","ChromeKing",
  "VoidCrawler","ElectricEel","SonicSnek","PlasmaFang","OmegaViper"
];
const CHAT_MESSAGES = [
  "gg","lol","lag?","bro stop teaming","nice cut","ez","??","rip",
  "noob","gitgud","wow","haha","this server trash","let's go!!!",
  "i was top 1 yesterday","anyone else lagging?","gg wp","insane",
  "bro i just died to a wall","stop following me","nice one!","lmaoo"
];
const REGIONS = ["NA-East #4","EU-West #2","NA-West #1","AS-South #3","EU-Central #5"];
const SNAKE_COLORS = [
  0xff4444, 0x44ff44, 0x4444ff, 0xffff44, 0xff44ff,
  0x44ffff, 0xff8844, 0x8844ff, 0x44ff88, 0xff4488,
  0x88ff44, 0x4488ff, 0xffaa00, 0x00ffaa, 0xaa00ff,
  0xff0088, 0x00ff88, 0x8800ff, 0xffa500, 0x00a5ff
];

// ============================================================
// PURE HELPERS
// ============================================================
function randBetween(a, b) { return a + Math.random() * (b - a); }
function randInt(a, b) { return Math.floor(a + Math.random() * (b - a + 1 - 1e-10)); }
function randFrom(arr) {
  if (!arr || arr.length === 0) return undefined;
  return arr[Math.floor(Math.random() * arr.length)];
}
function clamp(v, mn, mx) { return Math.max(mn, Math.min(mx, v)); }
function angleLerp(a, b, t) {
  let diff = b - a;
  while (diff > Math.PI) diff -= Math.PI * 2;
  while (diff < -Math.PI) diff += Math.PI * 2;
  return a + diff * t;
}
function distSq(ax, az, bx, bz) { return (ax - bx) ** 2 + (az - bz) ** 2; }
function uniqueName(used) {
  let n, tries = 0;
  do {
    const base = FAKE_NAMES[Math.floor(Math.random() * FAKE_NAMES.length)];
    n = base + (Math.random() < 0.4 ? randInt(10, 999) : "");
    tries++;
  } while (used.has(n) && tries < 40);
  return n;
}
function fmtTime(s) {
  const h = Math.floor(s / 3600).toString().padStart(2, "0");
  const m = Math.floor((s % 3600) / 60).toString().padStart(2, "0");
  const sc = (s % 60).toString().padStart(2, "0");
  return `${h}:${m}:${sc}`;
}

// ============================================================
// MAIN COMPONENT
// ============================================================
export default function App() {
  const mountRef = useRef(null);
  const [phase, setPhase] = useState("start");
  const [playerName, setPlayerName] = useState("");
  const [score, setScore] = useState(0);

  const bestScoreRef = useRef(parseInt(localStorage.getItem("slither_best") || "0"));
  const [bestScoreDisplay, setBestScoreDisplay] = useState(bestScoreRef.current);

  const [leaderboard, setLeaderboard] = useState([]);
  const [feedMessages, setFeedMessages] = useState([]);
  const [chatMessages, setChatMessages] = useState([]);
  const [ping, setPing] = useState(0);
  const [playersOnline, setPlayersOnline] = useState(0);
  const [region] = useState(() => randFrom(REGIONS) || "NA-East #4");
  const [uptime, setUptime] = useState(0);
  const [finalScore, setFinalScore] = useState(0);

  // Power-up HUD state
  const [activePowerups, setActivePowerups] = useState([]);
  const [powerupPopup, setPowerupPopup] = useState(null);

  // PWA install prompt
  const [installPrompt, setInstallPrompt] = useState(null);
  const [showInstall, setShowInstall] = useState(false);

  useEffect(() => {
    const handler = (e) => { e.preventDefault(); setInstallPrompt(e); setShowInstall(true); };
    window.addEventListener("beforeinstallprompt", handler);
    return () => window.removeEventListener("beforeinstallprompt", handler);
  }, []);

  const handleInstall = useCallback(() => {
    if (!installPrompt) return;
    installPrompt.prompt();
    installPrompt.userChoice.then(() => { setInstallPrompt(null); setShowInstall(false); });
  }, [installPrompt]);

  // Input refs
  const keysRef = useRef({});
  const mouseRef = useRef({ x: 0, y: 0 });
  const boostRef = useRef(false);
  const playerNameRef = useRef("");

  // Smooth mouse angle refs
  const rawTargetAngleRef = useRef(0);
  const smoothAngleRef = useRef(0);
  const turnRateRef = useRef(0);

  // Touch joystick
  const joystickTouchRef = useRef(null);
  const joystickStateRef = useRef({ active: false, angle: 0, distance: 0 });
  const joyBaseRef = useRef(null);
  const joyKnobRef = useRef(null);

  // Spawn shield state for HUD
  const [spawnShield, setSpawnShield] = useState(0);

  // ── FEED / CHAT ─────────────────────────────────────────────
  const addFeed = useCallback((msg) => {
    const id = Date.now() + Math.random();
    setFeedMessages(prev => [...prev.slice(-7), { id, msg }]);
    setTimeout(() => setFeedMessages(prev => prev.filter(m => m.id !== id)), 5000);
  }, []);

  const addChat = useCallback((msg) => {
    const id = Date.now() + Math.random();
    const name = FAKE_NAMES[Math.floor(Math.random() * FAKE_NAMES.length)];
    setChatMessages(prev => [...prev.slice(-5), { id, name, msg }]);
    setTimeout(() => setChatMessages(prev => prev.filter(m => m.id !== id)), 7000);
  }, []);

  // ── UI EFFECTS ───────────────────────────────────────────────
  useEffect(() => {
    if (phase !== "playing") return;
    setUptime(0);
    const t = setInterval(() => setUptime(u => u + 1), 1000);
    return () => clearInterval(t);
  }, [phase]);

  useEffect(() => {
    if (phase !== "playing") return;
    const base = randInt(32, 140);
    setPing(base);
    const t = setInterval(() => setPing(base + randInt(-8, 12)), 3000);
    return () => clearInterval(t);
  }, [phase]);

  useEffect(() => {
    if (phase !== "playing") return;
    const base = randInt(112, 156);
    setPlayersOnline(base);
    const t = setInterval(() => setPlayersOnline(base + randInt(-5, 8)), 7000);
    return () => clearInterval(t);
  }, [phase]);

  useEffect(() => {
    if (phase !== "playing") return;
    const t = setInterval(() => {
      if (Math.random() < 0.4) addChat(randFrom(CHAT_MESSAGES) || "gg");
    }, 4000);
    return () => clearInterval(t);
  }, [phase, addChat]);

  // ── GLOBAL INPUT ─────────────────────────────────────────────
  useEffect(() => {
    const kd = (e) => {
      keysRef.current[e.code] = true;
      if (["ShiftLeft","ShiftRight","Space"].includes(e.code)) {
        e.preventDefault();
        boostRef.current = true;
      }
    };
    const ku = (e) => {
      keysRef.current[e.code] = false;
      if (["ShiftLeft","ShiftRight","Space"].includes(e.code)) boostRef.current = false;
    };
    window.addEventListener("keydown", kd);
    window.addEventListener("keyup", ku);
    return () => { window.removeEventListener("keydown", kd); window.removeEventListener("keyup", ku); };
  }, []);

  useEffect(() => {
    const mm = (e) => { mouseRef.current = { x: e.clientX, y: e.clientY }; };
    window.addEventListener("mousemove", mm);
    return () => window.removeEventListener("mousemove", mm);
  }, []);

  // ── NAVIGATION ───────────────────────────────────────────────
  const handlePlay = useCallback(() => {
    playerNameRef.current = (playerName.trim() || "Guest" + randInt(1000, 9999)).slice(0, 16);
    setScore(0);
    setFeedMessages([]);
    setChatMessages([]);
    setActivePowerups([]);
    setPowerupPopup(null);
    setPhase("connecting");
    setTimeout(() => setPhase("playing"), randInt(1200, 2200));
  }, [playerName]);

  const handleRestart = useCallback(() => {
    setScore(0);
    setFeedMessages([]);
    setChatMessages([]);
    setActivePowerups([]);
    setPowerupPopup(null);
    setPhase("connecting");
    setTimeout(() => setPhase("playing"), randInt(1200, 2200));
  }, []);

  // ── TOUCH HANDLERS ───────────────────────────────────────────
  const handleTouchStart = useCallback((e) => {
    for (let i = 0; i < e.changedTouches.length; i++) {
      const t = e.changedTouches[i];
      if (!joystickTouchRef.current && t.clientX < window.innerWidth * 0.65) {
        joystickTouchRef.current = { id: t.identifier, startX: t.clientX, startY: t.clientY };
        joystickStateRef.current = { active: true, angle: 0, distance: 0 };
        if (joyBaseRef.current) {
          joyBaseRef.current.style.left = (t.clientX - 50) + "px";
          joyBaseRef.current.style.top  = (t.clientY - 50) + "px";
          joyBaseRef.current.style.bottom = "auto";
        }
      }
    }
  }, []);

  const handleTouchMove = useCallback((e) => {
    e.preventDefault();
    for (let i = 0; i < e.changedTouches.length; i++) {
      const t = e.changedTouches[i];
      if (joystickTouchRef.current && t.identifier === joystickTouchRef.current.id) {
        const dx = t.clientX - joystickTouchRef.current.startX;
        const dy = t.clientY - joystickTouchRef.current.startY;
        const dist = Math.sqrt(dx * dx + dy * dy);
        const angle = Math.atan2(dy, dx);
        joystickStateRef.current = { active: true, angle, distance: dist };
        if (joyKnobRef.current) {
          const c = Math.min(dist, 44);
          joyKnobRef.current.style.transform = `translate(${Math.cos(angle) * c}px,${Math.sin(angle) * c}px)`;
        }
      }
    }
  }, []);

  const handleTouchEnd = useCallback((e) => {
    for (let i = 0; i < e.changedTouches.length; i++) {
      const t = e.changedTouches[i];
      if (joystickTouchRef.current && t.identifier === joystickTouchRef.current.id) {
        joystickTouchRef.current = null;
        joystickStateRef.current = { active: false, angle: 0, distance: 0 };
        if (joyKnobRef.current) joyKnobRef.current.style.transform = "translate(0,0)";
      }
    }
  }, []);

  // ============================================================
  // THREE.JS GAME LOOP
  // ============================================================
  useEffect(() => {
    if (phase !== "playing") return;
    const container = mountRef.current;
    if (!container) return;

    setSpawnShield(SPAWN_IMMUNITY_SECONDS);

    // Player power-up state (mutable, synced to React state periodically)
    const playerPowerups = {};   // { speed: timeLeft, ghost: timeLeft, ... }
    let powerupHudTimer = 0;

    // ── RENDERER ─────────────────────────────────────────────
    // Use window dimensions — container.clientWidth can be 0 on mobile before layout
    const W = window.innerWidth, H = window.innerHeight;
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.setSize(W, H);
    renderer.domElement.style.display = "block";
    renderer.domElement.style.width = "100%";
    renderer.domElement.style.height = "100%";
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    container.appendChild(renderer.domElement);

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x050a0f);
    scene.fog = new THREE.FogExp2(0x050a0f, 0.0007);

    const camera = new THREE.PerspectiveCamera(65, W / H, 1, 5000);
    camera.position.set(0, 300, 400);
    camera.lookAt(0, 0, 0);

    // ── LIGHTS ───────────────────────────────────────────────
    scene.add(new THREE.AmbientLight(0x112233, 2.2));
    const dirLight = new THREE.DirectionalLight(0xffffff, 2.5);
    dirLight.position.set(300, 600, 300);
    dirLight.castShadow = true;
    dirLight.shadow.mapSize.set(2048, 2048);
    dirLight.shadow.camera.near = 1;
    dirLight.shadow.camera.far = 3000;
    dirLight.shadow.camera.left = dirLight.shadow.camera.bottom = -1200;
    dirLight.shadow.camera.right = dirLight.shadow.camera.top = 1200;
    scene.add(dirLight);
    const pointLight = new THREE.PointLight(0x00ffff, 2.5, 900);
    pointLight.position.set(0, 200, 0);
    scene.add(pointLight);

    // ── FLOOR ────────────────────────────────────────────────
    const floorGeo = new THREE.PlaneGeometry(ARENA_SIZE, ARENA_SIZE);
    const floorMat = new THREE.MeshStandardMaterial({ color: 0x050f15, roughness: 0.95, metalness: 0.05 });
    const floor = new THREE.Mesh(floorGeo, floorMat);
    floor.rotation.x = -Math.PI / 2;
    floor.receiveShadow = true;
    scene.add(floor);

    // Grid raised to y=2 to avoid z-fighting with the floor on mobile GPUs
    const gridHelper = new THREE.GridHelper(ARENA_SIZE, 80, 0x0a2030, 0x071520);
    gridHelper.position.y = 2;
    scene.add(gridHelper);

    // ── ARENA WALLS ──────────────────────────────────────────
    const wallMat = new THREE.MeshStandardMaterial({
      color: 0x00ffff, emissive: 0x003333, transparent: true, opacity: 0.25
    });
    const wallDefs = [
      { pos: [ HALF, 40,     0], ry: -Math.PI / 2 },
      { pos: [-HALF, 40,     0], ry:  Math.PI / 2 },
      { pos: [    0, 40,  HALF], ry:  Math.PI      },
      { pos: [    0, 40, -HALF], ry:  0            },
    ];
    const wallGeometries = [];
    wallDefs.forEach(({ pos, ry }) => {
      const wg = new THREE.PlaneGeometry(ARENA_SIZE, 80);
      wallGeometries.push(wg);
      const wm = new THREE.Mesh(wg, wallMat);
      wm.position.set(...pos);
      wm.rotation.y = ry;
      scene.add(wm);
    });

    // ── PELLETS ──────────────────────────────────────────────
    const PELLET_COLORS = [0xff5555,0x55ff55,0x5555ff,0xffff55,0xff55ff,0x55ffff,0xffaa00,0xaa00ff];
    const pellets = Array.from({ length: PELLET_COUNT }, (_, i) => ({
      x: randBetween(-HALF + 50, HALF - 50),
      z: randBetween(-HALF + 50, HALF - 50),
      value: randInt(1, 3),
      alive: i < Math.floor(PELLET_COUNT * 0.8),
      color: PELLET_COLORS[Math.floor(Math.random() * PELLET_COLORS.length)],
      pulse: Math.random() * Math.PI * 2,
    }));

    const pelletGeo = new THREE.SphereGeometry(PELLET_RADIUS, 8, 6);
    const pelletMat = new THREE.MeshStandardMaterial({ roughness: 0.3, metalness: 0.5 });
    const pelletMesh = new THREE.InstancedMesh(pelletGeo, pelletMat, PELLET_COUNT);
    pelletMesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage);
    scene.add(pelletMesh);

    const dummy = new THREE.Object3D();
    const pelletColor = new THREE.Color();

    function spawnPellet(x, z, value) {
      for (let i = 0; i < PELLET_COUNT; i++) {
        if (!pellets[i].alive) {
          pellets[i].x = x ?? randBetween(-HALF + 50, HALF - 50);
          pellets[i].z = z ?? randBetween(-HALF + 50, HALF - 50);
          pellets[i].value = value || randInt(1, 3);
          pellets[i].alive = true;
          pellets[i].color = PELLET_COLORS[Math.floor(Math.random() * PELLET_COLORS.length)];
          pellets[i].pulse = Math.random() * Math.PI * 2;
          return i;
        }
      }
      return -1;
    }

    function updatePelletInstances() {
      for (let i = 0; i < PELLET_COUNT; i++) {
        const p = pellets[i];
        if (!p.alive) {
          dummy.scale.setScalar(0);
          dummy.updateMatrix();
          pelletMesh.setMatrixAt(i, dummy.matrix);
          continue;
        }
        p.pulse += 0.05;
        dummy.position.set(p.x, PELLET_RADIUS + 2 * Math.sin(p.pulse), p.z);
        dummy.scale.setScalar(1 + 0.18 * Math.sin(p.pulse));
        dummy.updateMatrix();
        pelletMesh.setMatrixAt(i, dummy.matrix);
        pelletColor.set(p.color);
        pelletMesh.setColorAt(i, pelletColor);
      }
      pelletMesh.instanceMatrix.needsUpdate = true;
      if (pelletMesh.instanceColor) pelletMesh.instanceColor.needsUpdate = true;
    }

    // ── POWER-UPS ─────────────────────────────────────────────
    const puTypes = Object.keys(POWERUP_TYPES);
    const powerups = Array.from({ length: POWERUP_COUNT }, () => ({
      x: randBetween(-HALF + 100, HALF - 100),
      z: randBetween(-HALF + 100, HALF - 100),
      type: puTypes[Math.floor(Math.random() * puTypes.length)],
      alive: true,
      pulse: Math.random() * Math.PI * 2,
      respawnTimer: 0,
    }));

    const puGeo = new THREE.OctahedronGeometry(14, 0);
    const puMeshes = powerups.map((pu) => {
      const cfg = POWERUP_TYPES[pu.type];
      const mat = new THREE.MeshStandardMaterial({
        color: cfg.color, emissive: cfg.emissive, emissiveIntensity: 0.9,
        roughness: 0.2, metalness: 0.8,
      });
      const mesh = new THREE.Mesh(puGeo, mat);
      mesh.position.set(pu.x, 20, pu.z);
      mesh.castShadow = true;
      scene.add(mesh);
      return mesh;
    });

    function updatePowerupMeshes(dt, now) {
      for (let i = 0; i < POWERUP_COUNT; i++) {
        const pu = powerups[i];
        const mesh = puMeshes[i];

        if (!pu.alive) {
          pu.respawnTimer -= dt;
          if (pu.respawnTimer <= 0) {
            pu.alive = true;
            pu.x = randBetween(-HALF + 100, HALF - 100);
            pu.z = randBetween(-HALF + 100, HALF - 100);
            pu.type = puTypes[Math.floor(Math.random() * puTypes.length)];
            const cfg = POWERUP_TYPES[pu.type];
            mesh.material.color.setHex(cfg.color);
            mesh.material.emissive.setHex(cfg.emissive);
            mesh.visible = true;
          } else {
            mesh.visible = false;
            continue;
          }
        }

        pu.pulse += dt * 2.5;
        mesh.position.set(pu.x, 18 + 6 * Math.sin(pu.pulse), pu.z);
        mesh.rotation.y = pu.pulse * 0.8;
        mesh.rotation.x = pu.pulse * 0.4;
        mesh.visible = true;
      }
    }

    function checkPowerupCollect(snake) {
      if (!snake.isPlayer || !snake.alive || snake.segments.length === 0) return;
      const head = snake.segments[0];
      for (let i = 0; i < POWERUP_COUNT; i++) {
        const pu = powerups[i];
        if (!pu.alive) continue;
        if (distSq(head.x, head.z, pu.x, pu.z) < (snake.radius * 3.5) ** 2) {
          pu.alive = false;
          pu.respawnTimer = randBetween(12, 25);

          const cfg = POWERUP_TYPES[pu.type];
          playerPowerups[pu.type] = cfg.duration;

          // Show popup
          setPowerupPopup({ label: cfg.label, icon: cfg.icon, id: Date.now() });
          setTimeout(() => setPowerupPopup(null), 2500);
          addFeed(`${snake.name} grabbed ${cfg.icon} ${pu.type.toUpperCase()}!`);
        }
      }
    }

    // Tick down player powerups and sync HUD
    function tickPowerups(dt) {
      let changed = false;
      for (const key of Object.keys(playerPowerups)) {
        playerPowerups[key] -= dt;
        if (playerPowerups[key] <= 0) {
          delete playerPowerups[key];
          changed = true;
        }
      }
      powerupHudTimer += dt;
      if (powerupHudTimer >= 0.15 || changed) {
        powerupHudTimer = 0;
        setActivePowerups(Object.entries(playerPowerups).map(([type, t]) => ({
          type, timeLeft: Math.ceil(t), icon: POWERUP_TYPES[type].icon, color: POWERUP_TYPES[type].color,
        })));
      }
    }

    // ── SEGMENT POOL ──────────────────────────────────────────
    const segmentPool = [];

    function getSegment(color, radius) {
      let mesh;
      if (segmentPool.length > 0) {
        mesh = segmentPool.pop();
        mesh.material.color.setHex(color);
        mesh.material.emissive.setHex(color);
      } else {
        const geo = new THREE.SphereGeometry(1, 8, 6);
        const mat = new THREE.MeshStandardMaterial({
          color, emissive: color, emissiveIntensity: 0.35, roughness: 0.4, metalness: 0.6
        });
        mesh = new THREE.Mesh(geo, mat);
        mesh.castShadow = true;
      }
      mesh.scale.setScalar(radius);
      mesh.visible = true;
      scene.add(mesh);
      return mesh;
    }

    function releaseSegment(mesh) {
      if (!mesh) return;
      mesh.visible = false;
      scene.remove(mesh);
      segmentPool.push(mesh);
    }

    // ── SNAKE FACTORY ─────────────────────────────────────────
    function createSnake({ name, color, isPlayer, x = 0, z = 0, angle = 0 }) {
      const numStart = isPlayer ? 20 : randInt(10, 35);
      const segRadius = isPlayer ? SEGMENT_RADIUS : randBetween(5, 9);
      const segments = [];
      for (let i = 0; i < numStart; i++) {
        const r = segRadius * (i === 0 ? 1.35 : i < 3 ? 1.1 : 1.0);
        const mesh = getSegment(color, r);
        segments.push({
          x: x - Math.cos(angle) * i * SEGMENT_SPACING,
          z: z - Math.sin(angle) * i * SEGMENT_SPACING,
          mesh, radius: segRadius,
        });
      }
      return {
        name, color, isPlayer, segments,
        angle, boosting: false, alive: true,
        score: numStart, radius: segRadius,
        spawnImmunity: isPlayer ? SPAWN_IMMUNITY_SECONDS : 0,
        spawnFlickerT: 0,
        personality: isPlayer ? null : {
          aggression: Math.random(),
          greed: Math.random(),
          awareness: Math.random(),
          boostFrequency: Math.random(),
          riskTolerance: Math.random(),
          role: randFrom(["aggressive","passive","runner","cutter","circler"]) || "passive",
        },
        aiAngle: Math.random() * Math.PI * 2,
        aiTimer: 0,
      };
    }

    const usedNames = new Set();
    const pName = playerNameRef.current || "Guest" + randInt(1000, 9999);
    usedNames.add(pName);

    const spawnPoints = [
      { x: 500, z: 500 }, { x: -500, z: 500 },
      { x: 500, z: -500 }, { x: -500, z: -500 },
    ];
    const chosen = randFrom(spawnPoints) || { x: 500, z: 500 };
    const px = chosen.x + randBetween(-120, 120);
    const pz = chosen.z + randBetween(-120, 120);

    const player = createSnake({ name: pName, color: 0x00ffcc, isPlayer: true, x: px, z: pz, angle: Math.random() * Math.PI * 2 });
    const snakes = [player];

    for (let i = 0; i < AI_COUNT_TARGET; i++) {
      const n = uniqueName(usedNames); usedNames.add(n);
      let ax = 0, az = 0, tries = 0;
      do {
        ax = randBetween(-HALF + 150, HALF - 150);
        az = randBetween(-HALF + 150, HALF - 150);
        tries++;
      } while (distSq(ax, az, player.segments[0].x, player.segments[0].z) < SAFE_SPAWN_MIN_DISTANCE ** 2 && tries < 50);
      snakes.push(createSnake({ name: n, color: SNAKE_COLORS[i % SNAKE_COLORS.length], isPlayer: false, x: ax, z: az, angle: Math.random() * Math.PI * 2 }));
    }

    // ── SERVER TICK ───────────────────────────────────────────
    let serverTick = 0;
    function serverUpdate() {
      serverTick++;
      const dt = 1 / TICK_RATE;

      for (let si = 1; si < snakes.length; si++) {
        const s = snakes[si];
        if (!s.alive || s.segments.length === 0) continue;
        const p = s.personality;
        const head = s.segments[0];
        s.aiTimer -= dt;
        let desiredAngle = s.aiAngle;
        const playerHead = player.alive && player.segments.length > 0 ? player.segments[0] : null;
        const dToPlayer = playerHead ? Math.sqrt(distSq(head.x, head.z, playerHead.x, playerHead.z)) : Infinity;

        if (s.aiTimer <= 0) {
          s.aiTimer = randBetween(0.25, 1.0);
          const wallMargin = 130;
          let avoidWall = false;
          if (head.x < -HALF + wallMargin)      { desiredAngle = randBetween(-0.3, 0.3);           avoidWall = true; }
          else if (head.x > HALF - wallMargin)  { desiredAngle = Math.PI + randBetween(-0.3, 0.3); avoidWall = true; }
          if (!avoidWall) {
            if (head.z < -HALF + wallMargin)     { desiredAngle = Math.PI/2 + randBetween(-0.3, 0.3); avoidWall = true; }
            else if (head.z > HALF - wallMargin) { desiredAngle = -Math.PI/2 + randBetween(-0.3, 0.3); avoidWall = true; }
          }
          if (!avoidWall) {
            if (p.greed > 0.35) {
              let best = Infinity, bx = 0, bz = 0;
              const step = Math.max(1, Math.floor(PELLET_COUNT / 60));
              const start = Math.floor(Math.random() * step);
              for (let pi = start; pi < PELLET_COUNT; pi += step) {
                if (!pellets[pi].alive) continue;
                const d2 = distSq(head.x, head.z, pellets[pi].x, pellets[pi].z);
                if (d2 < best) { best = d2; bx = pellets[pi].x; bz = pellets[pi].z; }
              }
              if (best < (600 / (p.greed + 0.1)) ** 2) desiredAngle = Math.atan2(bz - head.z, bx - head.x);
            }
            if (playerHead && player.spawnImmunity <= 0) {
              if (p.role === "aggressive" && p.aggression > 0.5 && dToPlayer < 450)
                desiredAngle = Math.atan2(playerHead.z - head.z, playerHead.x - head.x);
              else if (p.role === "runner" && dToPlayer < 350)
                desiredAngle = Math.atan2(head.z - playerHead.z, head.x - playerHead.x);
              else if (p.role === "cutter" && dToPlayer < 600) {
                const tx = playerHead.x + Math.cos(player.angle) * 180;
                const tz = playerHead.z + Math.sin(player.angle) * 180;
                desiredAngle = Math.atan2(tz - head.z, tx - head.x);
              } else if (p.role === "circler" && dToPlayer < 500)
                desiredAngle = Math.atan2(playerHead.z - head.z, playerHead.x - head.x) + Math.PI * 0.45;
            }
            if (p.role === "passive" || dToPlayer > 400) {
              if (Math.random() < 0.25) desiredAngle += randBetween(-0.7, 0.7);
            }
          }
          s.aiAngle = desiredAngle;
          s.boosting = Math.random() < p.boostFrequency * 0.07 && s.segments.length > MIN_SEGMENTS + 5;
        }

        s.angle = angleLerp(s.angle, s.aiAngle, 0.1);
        const spd = (s.boosting ? BOOST_SPEED : BASE_SPEED) * dt;
        for (let i = s.segments.length - 1; i > 0; i--) {
          s.segments[i].x = s.segments[i - 1].x;
          s.segments[i].z = s.segments[i - 1].z;
        }
        s.segments[0].x = clamp(head.x + Math.cos(s.angle) * spd, -HALF + 10, HALF - 10);
        s.segments[0].z = clamp(head.z + Math.sin(s.angle) * spd, -HALF + 10, HALF - 10);

        if (s.boosting && s.segments.length > MIN_SEGMENTS && serverTick % 8 === 0) {
          const last = s.segments.pop();
          if (Math.random() < 0.5) spawnPellet(last.x, last.z, 1);
          releaseSegment(last.mesh);
        }
      }
    }

    const serverInterval = setInterval(serverUpdate, 1000 / TICK_RATE);

    // ── PELLET EATING ─────────────────────────────────────────
    function checkPelletEat(snake) {
      if (!snake.alive || snake.segments.length === 0) return;
      const head = snake.segments[0];
      const magnetActive = snake.isPlayer && playerPowerups["magnet"] > 0;
      const magnetRange = magnetActive ? (snake.radius * 9) ** 2 : 0;
      const eatR2 = (snake.radius * 2.8) ** 2;

      for (let i = 0; i < PELLET_COUNT; i++) {
        const p = pellets[i];
        if (!p.alive) continue;
        const d2 = distSq(head.x, head.z, p.x, p.z);

        // Magnet: pull pellets towards head
        if (magnetActive && d2 < magnetRange && d2 > eatR2) {
          const dx = head.x - p.x, dz = head.z - p.z;
          const dist = Math.sqrt(d2);
          p.x += (dx / dist) * 6;
          p.z += (dz / dist) * 6;
          continue;
        }

        if (d2 < eatR2) {
          p.alive = false;
          const last = snake.segments[snake.segments.length - 1];
          for (let g = 0; g < p.value; g++) {
            const mesh = getSegment(snake.color, snake.radius * 0.95);
            snake.segments.push({ x: last.x, z: last.z, mesh, radius: snake.radius });
          }
          snake.score += p.value;
          if (snake.isPlayer) setScore(snake.score);
          setTimeout(() => spawnPellet(null, null, randInt(1, 3)), 3000);
        }
      }
    }

    // ── COLLISION ─────────────────────────────────────────────
    function checkCollisions() {
      for (let si = 0; si < snakes.length; si++) {
        const s = snakes[si];
        if (!s.alive || s.segments.length === 0) continue;
        if (s.isPlayer && s.spawnImmunity > 0) continue;
        // Ghost mode: player can't die
        if (s.isPlayer && playerPowerups["ghost"] > 0) continue;

        const head = s.segments[0];
        for (let sj = 0; sj < snakes.length; sj++) {
          const other = snakes[sj];
          if (!other.alive || other.segments.length === 0) continue;

          // Player never dies from own body — pass through self entirely
          if (s.isPlayer && si === sj) continue;

          // AI self-collision still skips first 10 segments
          if (!s.isPlayer && si === sj) {
            if (s.segments.length < 12) continue;
          }

          const startSeg = (!s.isPlayer && si === sj) ? 10 : 0;
          const minD = s.radius + other.radius;
          const minD2 = minD * minD;
          for (let seg = startSeg; seg < other.segments.length; seg++) {
            if (distSq(head.x, head.z, other.segments[seg].x, other.segments[seg].z) < minD2) {
              killSnake(si, si !== sj ? sj : -1);
              break;
            }
          }
          if (!s.alive) break;
        }
      }
    }

    function killSnake(dyingIdx, killerIdx) {
      const s = snakes[dyingIdx];
      if (!s.alive) return;
      s.alive = false;
      const dropCount = Math.min(s.segments.length, 80);
      for (let i = 0; i < dropCount; i++) {
        const seg = s.segments[i];
        spawnPellet(seg.x + randBetween(-25, 25), seg.z + randBetween(-25, 25), 2);
      }
      s.segments.forEach(seg => releaseSegment(seg.mesh));
      s.segments = [];

      if (s.isPlayer) {
        const fs = s.score;
        setFinalScore(fs);
        if (fs > bestScoreRef.current) {
          bestScoreRef.current = fs;
          setBestScoreDisplay(fs);
          localStorage.setItem("slither_best", String(fs));
        }
        setPhase("dead");
        return;
      }

      const killer = killerIdx >= 0 ? snakes[killerIdx] : null;
      if (killer) addFeed(`${killer.name} eliminated ${s.name}`);

      const capturedIdx = dyingIdx;
      setTimeout(() => {
        if (!snakes[capturedIdx]) return;
        const n = uniqueName(usedNames); usedNames.add(n);
        const col = SNAKE_COLORS[Math.floor(Math.random() * SNAKE_COLORS.length)];
        const ns = createSnake({ name: n, color: col, isPlayer: false, x: randBetween(-HALF + 200, HALF - 200), z: randBetween(-HALF + 200, HALF - 200), angle: Math.random() * Math.PI * 2 });
        snakes[capturedIdx] = ns;
      }, randInt(3000, 8000));
    }

    const feedInterval = setInterval(() => {
      const alive = snakes.filter(x => x.alive && !x.isPlayer && x.segments.length > 60);
      if (alive.length === 0 || Math.random() > 0.4) return;
      const rnd = randFrom(alive);
      if (!rnd) return;
      const roll = Math.random();
      if (roll < 0.5) addFeed(`${rnd.name} is on a killing spree!`);
      else            addFeed(`${rnd.name} reached length ${rnd.segments.length}`);
    }, randInt(12000, 22000));

    // ── PLAYER INPUT — SMOOTHED ────────────────────────────────
    let cameraYaw = 0;
    let boostFrameCounter = 0;

    // Init smooth angle from player spawn angle
    smoothAngleRef.current = player.angle;
    rawTargetAngleRef.current = player.angle;
    turnRateRef.current = 0;

    function updatePlayerInput(dt) {
      if (!player.alive || player.segments.length === 0) return;

      const cw = container.clientWidth, ch = container.clientHeight;

      // ── Compute raw target angle ───────────────────────────
      const joy = joystickStateRef.current;
      let newRaw = rawTargetAngleRef.current;
      let hasInput = false;

      if (joy.active && joy.distance > JOY_DEAD_ZONE) {
        newRaw = joy.angle + cameraYaw;
        hasInput = true;
      } else {
        const mx = mouseRef.current.x - cw / 2;
        const my = mouseRef.current.y - ch / 2;
        if (Math.abs(mx) > 8 || Math.abs(my) > 8) {
          newRaw = Math.atan2(my, mx) + cameraYaw;
          hasInput = true;
        }
      }

      // Keyboard overrides — direct, snappy turn
      const keys = keysRef.current;
      if (keys["ArrowLeft"]  || keys["KeyA"]) { newRaw = player.angle - MAX_TURN_RATE * dt * 18; hasInput = true; }
      if (keys["ArrowRight"] || keys["KeyD"]) { newRaw = player.angle + MAX_TURN_RATE * dt * 18; hasInput = true; }

      rawTargetAngleRef.current = newRaw;

      // ── Exponential smooth towards raw target ──────────────
      // Build up turn rate gradually for analogue feel
      if (hasInput) {
        turnRateRef.current = Math.min(turnRateRef.current + TURN_ACCEL * dt, MAX_TURN_RATE);
      } else {
        turnRateRef.current = Math.max(0, turnRateRef.current - TURN_ACCEL * dt * 0.5);
      }

      // Lerp smooth angle toward raw target, scaled by turn rate
      const lerpT = clamp(MOUSE_SMOOTH + (turnRateRef.current / MAX_TURN_RATE) * 0.08, 0.04, 0.22);
      smoothAngleRef.current = angleLerp(smoothAngleRef.current, rawTargetAngleRef.current, lerpT);
      player.angle = smoothAngleRef.current;

      // Boost with speed powerup multiplier
      const isBoosting = boostRef.current || keys["ShiftLeft"] || keys["ShiftRight"] || keys["Space"];
      player.boosting = isBoosting && player.segments.length > MIN_SEGMENTS;

      const speedMult = playerPowerups["speed"] > 0 ? 1.55 : 1.0;
      const spd = ((player.boosting ? BOOST_SPEED : BASE_SPEED) * speedMult) * dt;

      const head = player.segments[0];
      for (let i = player.segments.length - 1; i > 0; i--) {
        player.segments[i].x = player.segments[i - 1].x;
        player.segments[i].z = player.segments[i - 1].z;
      }

      let nx = head.x + Math.cos(player.angle) * spd;
      let nz = head.z + Math.sin(player.angle) * spd;
      const WALL = HALF - 12;
      const BOUNCE_DAMP = 0.85; // slight speed loss on bounce

      // X walls — reflect X component of velocity
      if (nx > WALL) {
        nx = WALL;
        // angle was heading right (+x), reflect: negate x component
        // new angle = π - old angle
        player.angle = Math.PI - player.angle;
        smoothAngleRef.current = player.angle;
        rawTargetAngleRef.current = player.angle;
        spd *= BOUNCE_DAMP; // already used, just nudge back
      } else if (nx < -WALL) {
        nx = -WALL;
        player.angle = Math.PI - player.angle;
        smoothAngleRef.current = player.angle;
        rawTargetAngleRef.current = player.angle;
      }

      // Z walls — reflect Z component of velocity
      if (nz > WALL) {
        nz = WALL;
        player.angle = -player.angle;
        smoothAngleRef.current = player.angle;
        rawTargetAngleRef.current = player.angle;
      } else if (nz < -WALL) {
        nz = -WALL;
        player.angle = -player.angle;
        smoothAngleRef.current = player.angle;
        rawTargetAngleRef.current = player.angle;
      }

      player.segments[0].x = nx;
      player.segments[0].z = nz;

      if (player.boosting && player.segments.length > MIN_SEGMENTS) {
        boostFrameCounter++;
        if (boostFrameCounter % 8 === 0) {
          const last = player.segments.pop();
          spawnPellet(last.x, last.z, 1);
          releaseSegment(last.mesh);
        }
      }
    }

    // ── CAMERA ────────────────────────────────────────────────
    const camPos = new THREE.Vector3(px, 500, pz);
    let cameraZoom = 1;
    camera.up.set(0, 0, -1);

    const onWheel = (e) => { cameraZoom = clamp(cameraZoom + e.deltaY * 0.001, 0.4, 2.8); };
    window.addEventListener("wheel", onWheel, { passive: true });

    function updateCamera() {
      if (!player.alive || player.segments.length === 0) return;
      const head = player.segments[0];
      const camHeight = 420 * cameraZoom;
      camPos.x += (head.x - camPos.x) * 0.1;
      camPos.z += (head.z - camPos.z) * 0.1;
      camPos.y  = camHeight;
      camera.position.copy(camPos);
      camera.lookAt(camPos.x, 0, camPos.z);
      cameraYaw = 0;
    }

    // ── LEADERBOARD ───────────────────────────────────────────
    let lbTimer = 0;
    function updateLeaderboard(dt) {
      lbTimer += dt;
      if (lbTimer < 1.0) return;
      lbTimer = 0;
      const lb = snakes
        .filter(s => s.alive && s.segments.length > 0)
        .map(s => ({ name: s.name, length: s.segments.length, isPlayer: s.isPlayer }))
        .sort((a, b) => b.length - a.length)
        .slice(0, 10);
      setLeaderboard(lb);
    }

    // ── RENDER LOOP ───────────────────────────────────────────
    let lastTime = performance.now();
    let animId;
    let shieldHudTimer = 0;

    function loop(now) {
      animId = requestAnimationFrame(loop);
      const dt = Math.min((now - lastTime) / 1000, 0.05);
      lastTime = now;

      if (player.spawnImmunity > 0) {
        player.spawnImmunity = Math.max(0, player.spawnImmunity - dt);
        player.spawnFlickerT += dt;
      }

      shieldHudTimer += dt;
      if (shieldHudTimer >= 0.1) {
        shieldHudTimer = 0;
        setSpawnShield(player.spawnImmunity);
      }

      tickPowerups(dt);
      updatePlayerInput(dt);

      if (player.alive) {
        checkPelletEat(player);
        checkPowerupCollect(player);
      }
      for (let si = 1; si < snakes.length; si++) {
        if (snakes[si].alive) checkPelletEat(snakes[si]);
      }

      checkCollisions();
      updatePowerupMeshes(dt, now);

      // Ghost: make player snake translucent
      const ghostActive = playerPowerups["ghost"] > 0;
      for (let si = 0; si < snakes.length; si++) {
        const s = snakes[si];
        if (!s.alive) continue;
        for (let seg = 0; seg < s.segments.length; seg++) {
          const sg = s.segments[seg];
          if (!sg.mesh) continue;
          let visible = true;
          if (s.isPlayer && s.spawnImmunity > 0) {
            visible = Math.floor(s.spawnFlickerT * 10) % 2 === 0;
          }
          sg.mesh.visible = visible;
          if (visible) {
            sg.mesh.position.set(sg.x, sg.radius + 1 + (s.boosting ? 3 : 0), sg.z);
            sg.mesh.scale.setScalar(sg.radius * (seg === 0 ? 1 + 0.06 * Math.sin(now * 0.006) : seg < 3 ? 1.08 : 1.0));
            // Ghost effect: transparent blue tint
            if (s.isPlayer) {
              sg.mesh.material.transparent = ghostActive;
              sg.mesh.material.opacity = ghostActive ? 0.45 : 1.0;
              sg.mesh.material.color.setHex(ghostActive ? 0x88aaff : s.color);
              sg.mesh.material.emissive.setHex(ghostActive ? 0x2244aa : s.color);
            }
          }
        }
      }

      updatePelletInstances();
      updateCamera();
      updateLeaderboard(dt);

      pointLight.intensity = 1.8 + 0.6 * Math.sin(now * 0.002);
      renderer.render(scene, camera);
    }

    animId = requestAnimationFrame(loop);

    const onResize = () => {
      const w = window.innerWidth, h = window.innerHeight;
      camera.aspect = w / h;
      camera.updateProjectionMatrix();
      renderer.setSize(w, h);
    };
    window.addEventListener("resize", onResize);

    // ── CLEANUP ───────────────────────────────────────────────
    return () => {
      cancelAnimationFrame(animId);
      clearInterval(serverInterval);
      clearInterval(feedInterval);
      window.removeEventListener("resize", onResize);
      window.removeEventListener("wheel", onWheel);

      for (const s of snakes) {
        for (const seg of s.segments) releaseSegment(seg.mesh);
        s.segments = [];
      }
      for (const mesh of puMeshes) { mesh.geometry.dispose(); mesh.material.dispose(); scene.remove(mesh); }
      for (const mesh of segmentPool) { mesh.geometry.dispose(); mesh.material.dispose(); }

      floorGeo.dispose(); floorMat.dispose();
      pelletGeo.dispose(); pelletMat.dispose();
      puGeo.dispose();
      wallMat.dispose(); wallGeometries.forEach(g => g.dispose());
      renderer.dispose();
      if (container.contains(renderer.domElement)) container.removeChild(renderer.domElement);
    };
  }, [phase, addFeed]); // eslint-disable-line react-hooks/exhaustive-deps

  // ============================================================
  // RENDER
  // ============================================================
  return (
    <div style={{ width:"100vw", height:"100vh", overflow:"hidden", background:"#050a0f", fontFamily:"'Courier New',monospace", userSelect:"none" }}>

      {/* THREE.JS CANVAS MOUNT */}
      {phase === "playing" && (
        <div ref={mountRef} style={{ position:"absolute", top:0, left:0, width:"100vw", height:"100vh", overflow:"hidden" }}
          onTouchStart={handleTouchStart}
          onTouchMove={handleTouchMove}
          onTouchEnd={handleTouchEnd}
        />
      )}

      {/* ── START SCREEN ── */}
      {phase === "start" && (
        <div style={{ position:"absolute", inset:0, display:"flex", flexDirection:"column", alignItems:"center", justifyContent:"center", background:"radial-gradient(ellipse at center,#0a1a2a 0%,#050a0f 70%)" }}>
          <div style={{ textAlign:"center", padding:40, maxWidth:420, width:"90%" }}>
            <div style={{ fontSize:12, color:"#00ffcc", letterSpacing:8, marginBottom:8, textTransform:"uppercase" }}>● Online</div>
            <h1 style={{ fontSize:"clamp(44px,10vw,86px)", fontWeight:900, color:"#fff", margin:"0 0 4px", letterSpacing:-2, textShadow:"0 0 40px #00ffcc88" }}>
              SLITHER<span style={{ color:"#00ffcc" }}>.3D</span>
            </h1>
            <div style={{ fontSize:12, color:"#336655", marginBottom:28, letterSpacing:3 }}>MASSIVE MULTIPLAYER SNAKE</div>

            {/* Power-up legend */}
            <div style={{ display:"flex", justifyContent:"center", gap:16, marginBottom:28, flexWrap:"wrap" }}>
              {Object.entries(POWERUP_TYPES).map(([k, v]) => (
                <div key={k} style={{ display:"flex", alignItems:"center", gap:5, fontSize:10, color:"#667788" }}>
                  <span style={{ fontSize:16 }}>{v.icon}</span>
                  <span style={{ textTransform:"uppercase", letterSpacing:1 }}>{k}</span>
                </div>
              ))}
            </div>

            <input
              type="text" maxLength={16} placeholder="Enter your name..."
              value={playerName}
              onChange={e => setPlayerName(e.target.value)}
              onKeyDown={e => e.key === "Enter" && handlePlay()}
              style={{ width:"100%", padding:"14px 20px", marginBottom:14, boxSizing:"border-box", background:"rgba(0,255,204,0.06)", border:"1px solid rgba(0,255,204,0.3)", borderRadius:8, color:"#00ffcc", fontSize:16, outline:"none", letterSpacing:1 }}
            />
            <button onClick={handlePlay}
              style={{ width:"100%", padding:"18px 0", background:"linear-gradient(135deg,#00ffcc,#0088ff)", border:"none", borderRadius:8, color:"#000", fontSize:18, fontWeight:900, cursor:"pointer", letterSpacing:2, textTransform:"uppercase", boxShadow:"0 0 30px #00ffcc44" }}>
              PLAY NOW
            </button>

            {showInstall ? (
              <button onClick={handleInstall}
                style={{ width:"100%", padding:"13px 0", marginTop:10, background:"transparent", border:"1px solid rgba(0,255,204,0.3)", borderRadius:8, color:"#00ffcc", fontSize:13, cursor:"pointer", letterSpacing:2 }}>
                📲 INSTALL APP
              </button>
            ) : /iphone|ipad|ipod/i.test(navigator.userAgent) ? (
              <div style={{ marginTop:10, padding:"10px 14px", border:"1px solid rgba(0,255,204,0.15)", borderRadius:8, fontSize:11, color:"#336655", lineHeight:1.7, textAlign:"left" }}>
                📲 <span style={{ color:"#00ffcc" }}>Install on iOS:</span> tap the Share button then <span style={{ color:"#00ffcc" }}>"Add to Home Screen"</span>
              </div>
            ) : null}

            <div style={{ marginTop:22, display:"flex", justifyContent:"space-between", fontSize:11, color:"#336655" }}>
              <span>🌍 {region}</span>
              <span>Best: {bestScoreDisplay}</span>
              <span>⚡ {randInt(32,120)}ms</span>
            </div>
            <div style={{ marginTop:28, fontSize:11, color:"#1a3322", lineHeight:1.9 }}>
              <div>Mouse / WASD — steer &nbsp;|&nbsp; Shift / Space — boost</div>
              <div>Touch left → joystick &nbsp;|&nbsp; Touch right → boost button</div>
            </div>
          </div>
        </div>
      )}

      {/* ── CONNECTING SCREEN ── */}
      {phase === "connecting" && (
        <div style={{ position:"absolute", inset:0, display:"flex", flexDirection:"column", alignItems:"center", justifyContent:"center", background:"#050a0f" }}>
          <div style={{ textAlign:"center" }}>
            <div style={{ width:60, height:60, border:"3px solid #00ffcc22", borderTop:"3px solid #00ffcc", borderRadius:"50%", animation:"spin 0.9s linear infinite", margin:"0 auto 24px" }} />
            <div style={{ color:"#00ffcc", fontSize:18, letterSpacing:4 }}>CONNECTING...</div>
            <div style={{ color:"#336655", fontSize:12, marginTop:8 }}>{region}</div>
            <div style={{ color:"#1a3322", fontSize:11, marginTop:4 }}>Establishing secure connection...</div>
          </div>
          <style>{`@keyframes spin{to{transform:rotate(360deg)}}`}</style>
        </div>
      )}

      {/* ── GAME OVER SCREEN ── */}
      {phase === "dead" && (
        <div style={{ position:"absolute", inset:0, display:"flex", flexDirection:"column", alignItems:"center", justifyContent:"center", background:"rgba(5,10,15,0.93)" }}>
          <div style={{ textAlign:"center", padding:40, maxWidth:360, width:"90%" }}>
            <div style={{ fontSize:12, color:"#ff4444", letterSpacing:6, marginBottom:10 }}>YOU DIED</div>
            <h2 style={{ fontSize:68, fontWeight:900, color:"#fff", margin:"0 0 4px", textShadow:"0 0 30px #ff444488" }}>{finalScore}</h2>
            <div style={{ color:"#665533", fontSize:12, marginBottom:8 }}>SCORE</div>
            {finalScore > 0 && finalScore >= bestScoreDisplay && (
              <div style={{ color:"#ffcc00", fontSize:13, letterSpacing:2, marginBottom:14 }}>🏆 NEW BEST!</div>
            )}
            <div style={{ color:"#336655", fontSize:12, marginBottom:30 }}>Best: {Math.max(bestScoreDisplay, finalScore)}</div>
            <button onClick={handleRestart}
              style={{ width:"100%", padding:"18px 0", background:"linear-gradient(135deg,#ff4444,#ff8800)", border:"none", borderRadius:8, color:"#fff", fontSize:18, fontWeight:900, cursor:"pointer", letterSpacing:2, textTransform:"uppercase", boxShadow:"0 0 28px #ff444444" }}>
              PLAY AGAIN
            </button>
            <button onClick={() => setPhase("start")}
              style={{ width:"100%", padding:"14px 0", marginTop:10, background:"transparent", border:"1px solid #1a3322", borderRadius:8, color:"#336655", fontSize:14, cursor:"pointer", letterSpacing:2 }}>
              MENU
            </button>
          </div>
        </div>
      )}

      {/* ── HUD (playing only) ── */}
      {phase === "playing" && <>

        {/* Score — top center */}
        <div style={{ position:"absolute", top:14, left:"50%", transform:"translateX(-50%)", textAlign:"center", pointerEvents:"none" }}>
          <div style={{ fontSize:38, fontWeight:900, color:"#fff", textShadow:"0 0 20px #00ffcc99", lineHeight:1 }}>{score}</div>
          <div style={{ fontSize:10, color:"#336655", letterSpacing:3 }}>SCORE</div>
          <div style={{ fontSize:10, color:"#1a3322" }}>Best: {Math.max(bestScoreDisplay, score)}</div>
          {spawnShield > 0.05 && (
            <div style={{ marginTop:8, fontSize:12, color:"#00ffcc", letterSpacing:2, textShadow:"0 0 18px #00ffcc88" }}>
              🛡 SPAWN SHIELD {Math.ceil(spawnShield)}s
            </div>
          )}
        </div>

        {/* Power-up status bar */}
        {activePowerups.length > 0 && (
          <div style={{ position:"absolute", top:100, left:"50%", transform:"translateX(-50%)", display:"flex", gap:8, pointerEvents:"none" }}>
            {activePowerups.map(pu => (
              <div key={pu.type} style={{
                display:"flex", flexDirection:"column", alignItems:"center",
                background:"rgba(5,10,15,0.85)", border:`1px solid #${POWERUP_TYPES[pu.type].color.toString(16).padStart(6,"0")}55`,
                borderRadius:8, padding:"4px 10px", minWidth:52,
              }}>
                <span style={{ fontSize:18 }}>{pu.icon}</span>
                <span style={{ fontSize:10, color:"#aabbcc", letterSpacing:1 }}>{pu.timeLeft}s</span>
              </div>
            ))}
          </div>
        )}

        {/* Power-up popup */}
        {powerupPopup && (
          <div key={powerupPopup.id} style={{
            position:"absolute", top:"38%", left:"50%", transform:"translateX(-50%)",
            background:"rgba(5,10,15,0.92)", border:"1px solid rgba(0,255,204,0.3)",
            borderRadius:12, padding:"12px 28px", textAlign:"center", pointerEvents:"none",
            animation:"puPop 0.4s ease",
          }}>
            <div style={{ fontSize:36 }}>{powerupPopup.icon}</div>
            <div style={{ fontSize:13, color:"#00ffcc", letterSpacing:3, fontWeight:700 }}>{powerupPopup.label}</div>
          </div>
        )}

        {/* Server info + leaderboard */}
        <div style={{ position:"absolute", top:12, right:12, display:"flex", flexDirection:"column", gap:8, alignItems:"flex-end", pointerEvents:"none" }}>
          <div style={{ textAlign:"right", fontSize:10, color:"#1a3322", lineHeight:1.9 }}>
            <div style={{ color:"#00ffcc", fontSize:11 }}>● {playersOnline} online</div>
            <div>{region}</div>
            <div>Ping:&nbsp;
              <span style={{ color: ping < 60 ? "#00ff88" : ping < 100 ? "#ffcc00" : "#ff4444" }}>{ping}ms</span>
            </div>
            <div>Uptime: {fmtTime(uptime)}</div>
          </div>
          <div style={{ background:"rgba(5,10,15,0.82)", border:"1px solid #0a2030", borderRadius:8, padding:"8px 12px", minWidth:170 }}>
            <div style={{ fontSize:9, color:"#336655", letterSpacing:3, marginBottom:7, textTransform:"uppercase" }}>Leaderboard</div>
            {leaderboard.map((e, i) => (
              <div key={i} style={{ display:"flex", alignItems:"center", fontSize:11, marginBottom:3, color: e.isPlayer ? "#00ffcc" : "#9aabbb" }}>
                <span style={{ color: i === 0 ? "#ffcc00" : "#334455", width:16, flexShrink:0 }}>{i + 1}.</span>
                <span style={{ flex:1, overflow:"hidden", textOverflow:"ellipsis", whiteSpace:"nowrap", marginLeft:4 }}>{e.name}</span>
                <span style={{ marginLeft:8, color:"#334455", flexShrink:0 }}>{e.length}</span>
              </div>
            ))}
          </div>
        </div>

        {/* Activity feed */}
        <div style={{ position:"absolute", bottom:96, left:12, pointerEvents:"none", maxWidth:280 }}>
          {feedMessages.map(m => (
            <div key={m.id} style={{ fontSize:11, color:"#55aa88", background:"rgba(5,10,15,0.72)", padding:"3px 9px", borderRadius:4, marginBottom:3, borderLeft:"2px solid #0a3020", animation:"fadeIn 0.3s ease" }}>
              {m.msg}
            </div>
          ))}
        </div>

        {/* Chat */}
        <div style={{ position:"absolute", bottom:210, left:12, pointerEvents:"none", maxWidth:260 }}>
          {chatMessages.map(m => (
            <div key={m.id} style={{ fontSize:11, color:"#445566", background:"rgba(5,10,15,0.6)", padding:"3px 8px", borderRadius:4, marginBottom:2 }}>
              <span style={{ color:"#00aa88" }}>{m.name}:</span> {m.msg}
            </div>
          ))}
        </div>

        {/* Boost button */}
        <div style={{ position:"absolute", bottom:44, right:28, pointerEvents:"auto" }}>
          <button
            onTouchStart={(e) => { e.stopPropagation(); boostRef.current = true; }}
            onTouchEnd={(e)   => { e.stopPropagation(); boostRef.current = false; }}
            onMouseDown={() => { boostRef.current = true; }}
            onMouseUp={()   => { boostRef.current = false; }}
            onMouseLeave={() => { boostRef.current = false; }}
            style={{ width:74, height:74, borderRadius:"50%", background:"radial-gradient(circle,#ff8800,#ff3300)", border:"3px solid rgba(255,136,0,0.6)", color:"#fff", fontSize:11, fontWeight:900, cursor:"pointer", boxShadow:"0 0 22px #ff440066", display:"flex", alignItems:"center", justifyContent:"center", flexDirection:"column", WebkitTapHighlightColor:"transparent" }}>
            <div style={{ fontSize:22 }}>⚡</div>
            <div style={{ fontSize:9, letterSpacing:1 }}>BOOST</div>
          </button>
        </div>

        {/* Virtual joystick base */}
        <div ref={joyBaseRef}
          style={{ position:"absolute", bottom:80, left:40, width:100, height:100, borderRadius:"50%", background:"rgba(0,255,204,0.07)", border:"2px solid rgba(0,255,204,0.18)", display:"flex", alignItems:"center", justifyContent:"center", pointerEvents:"none" }}>
          <div ref={joyKnobRef}
            style={{ width:42, height:42, borderRadius:"50%", background:"rgba(0,255,204,0.28)", border:"2px solid rgba(0,255,204,0.55)", willChange:"transform" }} />
        </div>

        {/* Controls hint */}
        <div style={{ position:"absolute", bottom:10, left:"50%", transform:"translateX(-50%)", fontSize:9, color:"#1a3322", letterSpacing:2, pointerEvents:"none", whiteSpace:"nowrap" }}>
          MOUSE / WASD — steer &nbsp;|&nbsp; SHIFT / SPACE — boost &nbsp;|&nbsp; Collect ⚡👻🧲❄️ power-ups!
        </div>
      </>}

      <style>{`
        *{box-sizing:border-box}
        input::placeholder{color:#336655}
        @keyframes fadeIn{from{opacity:0;transform:translateX(-8px)}to{opacity:1;transform:translateX(0)}}
        @keyframes puPop{from{opacity:0;transform:translateX(-50%) scale(0.7)}to{opacity:1;transform:translateX(-50%) scale(1)}}
      `}</style>
    </div>
  );
}
