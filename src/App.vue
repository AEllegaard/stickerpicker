<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import './assets/style.css';

// importér dine deco assets via Vite
import hatSrc from './assets/hat.webp';

const canvasHost = ref(null);

// UI state
const inMiroRef = ref(false);
const buttonLabel = ref('Save sticker locally');

// handles
let P5, ml5, pInstance = null, faceMesh = null, handModel = null;
let faces = [];
let hands = [];

// MediaPipe fallback til hænder
let handLandmarker = null;
let handLoopStop = null;

// deco-objekter på lærredet
let decos = [];

// stop-funktioner til fallback loops
let stopManualFaceDetect = null;
let stopManualHandDetect = null;

// face konstanter
const FACE_EXPAND = 1.3;
const faceOutline = [10,338,297,332,284,251,389,356,447,366,401,288,397,365,379,378,400,377,152,148,176,149,150,136,172,58,132,93,234,127,162,103,67];

// tegne-ressourcer
let g = {
  video: null,
  pg: null,
  maskG: null,
  maskedImg: null,
};

const VID_W = 330, VID_H = 240;

// palette
const palette = [
  { key: 'hat', src: hatSrc, x: 55, y: 320, w: 60, h: 60 }
];

// robust Miro-detektion
async function probeMiro() {
  try {
    if (typeof window !== 'undefined' && window.miro && window.miro.board) {
      await window.miro.board.getInfo();
      return true;
    }
    return false;
  } catch {
    return false;
  }
}

// simpelt deco-objekt
class Deco {
  constructor(p, img, x, y, w, h) {
    this.p = p; this.img = img;
    this.x = x; this.y = y;
    this.w = w; this.h = h;
    this.dragging = false;
    this._dx = 0; this._dy = 0;
  }
  contains(mx, my) {
    return Math.abs(mx - this.x) <= this.w/2 && Math.abs(my - this.y) <= this.h/2;
  }
  startDrag(mx, my) { this.dragging = true; this._dx = mx - this.x; this._dy = my - this.y; }
  drag(mx, my) { if (this.dragging) { this.x = mx - this._dx; this.y = my - this._dy; } }
  stopDrag() { this.dragging = false; }
  angleToOrigin() { return Math.atan2(this.y, this.x); }
  draw() {
    const p = this.p;
    if (!this.img) return;
    p.push();
    p.translate(this.x, this.y);
    p.rotate(this.angleToOrigin());
    p.imageMode(p.CENTER);
    p.image(this.img, 0, 0, this.w, this.h);
    p.pop();
  }
}

const makeSketch = (p) => {
  p.assets = { hat: null };

  p.setup = () => {
    p.pixelDensity(1);
    p.angleMode(p.RADIANS);
    p.createCanvas(330, 390);

    // load hat
    p.loadImage(hatSrc, img => { p.assets.hat = img; });

    // kamera
    g.video = p.createCapture({ video: { facingMode: 'user', width: VID_W, height: VID_H }, audio: false });
    g.video.size(VID_W, VID_H);
    g.video.hide();
    Object.assign(g.video.elt.style, { position: 'absolute', left: '-10000px', width: '0px', height: '0px', opacity: '0' });
    g.video.elt.setAttribute('playsinline', 'true');

    // offscreen buffers
    g.pg    = p.createGraphics(VID_W, VID_H);
    g.maskG = p.createGraphics(VID_W, VID_H);

    // async init
    (async () => {
      await waitForVideo(g.video.elt);

      // TF backend
      try {
        const tf = ml5?.tf;
        if (tf?.setBackend) {
          try { await tf.setBackend('webgl'); await tf.ready(); }
          catch { await tf.setBackend('wasm'); await tf.ready(); }
        }
      } catch {}

      // FaceMesh
      faceMesh = await ml5.faceMesh({ maxFaces: 1, refineLandmarks: false, flipped: true });
      await waitForFaceModel(faceMesh, 15000);
      try {
        faceMesh.detectStart(g.video.elt, (results) => { faces = results || []; });
        console.log('FaceMesh detectStart kører');
      } catch (e) {
        console.warn('Face detectStart fejlede, manuelt loop', e);
        startManualFaceLoop();
      }

      // Hænder: prøv ml5.handpose, ellers fallback til MediaPipe
      try {
        if (ml5.handpose) {
          handModel = await ml5.handpose({ flipped: true });
          await waitForHandModel(handModel, 15000);
          try {
            handModel.detectStart(g.video.elt, (results) => { hands = results || []; });
            console.log('Handpose detectStart kører (ml5)');
          } catch (e) {
            console.warn('ml5 hand detectStart fejlede, manuelt loop', e);
            startManualHandLoop();
          }
        } else {
          console.warn('ml5.handpose mangler — starter MediaPipe fallback');
          await initHandsFallback(g.video.elt, VID_W, VID_H);
          console.log('HandLandmarker detect kører (MediaPipe)');
        }
      } catch (e) {
        console.warn('Kunne ikke starte hånddetektion. Fortsætter uden hænder.', e);
      }
    })().catch(console.error);
  };

  p.draw = () => {
    p.background(220);

    // clear buffers
    g.pg.clear();
    g.maskG.clear();
    g.maskG.noStroke();
    g.maskG.fill(255);

    // spejl video til offscreen
    if (g.video) {
      g.pg.push();
      g.pg.translate(VID_W, 0);
      g.pg.scale(-1, 1);
      g.pg.image(g.video, 0, 0, VID_W, VID_H);
      g.pg.pop();
    }

    // palette panel
    drawPalette(p);

    // mask: ansigt + hænder
    if (faces.length > 0) {
      const face = faces[0];

      // ansigtscentrum
      let cx = 0, cy = 0;
      for (let i = 0; i < faceOutline.length; i++) {
        const kp = face.keypoints[faceOutline[i]];
        cx += kp.x; cy += kp.y;
      }
      cx /= faceOutline.length; cy /= faceOutline.length;

      // ansigtsform
      g.maskG.beginShape();
      for (let i = 0; i < faceOutline.length; i++) {
        const kp = face.keypoints[faceOutline[i]];
        const vx = kp.x - cx, vy = kp.y - cy;
        g.maskG.vertex(cx + vx * FACE_EXPAND, cy + vy * FACE_EXPAND);
      }
      g.maskG.endShape(p.CLOSE);

      // hænder som elipser oveni masken
      const handBounds = currentAllHandBounds();
      for (const hb of handBounds) {
        const mw = hb.w * 1.25;
        const mh = hb.h * 1.25;
        g.maskG.push();
        g.maskG.ellipseMode(g.maskG.CENTER);
        g.maskG.noStroke();
        g.maskG.fill(255);
        g.maskG.ellipse(hb.cx, hb.cy, mw, mh);
        g.maskG.pop();
      }

      // anvend masken
      const masked = g.pg.get();
      masked.mask(g.maskG.get());
      g.maskedImg = masked;

      // vis ansigt + hænder
      p.image(g.maskedImg, 0, 0, VID_W, VID_H);

      // face-outline som pynt
      p.push();
      p.noFill(); p.stroke(255); p.strokeWeight(10);
      p.beginShape();
      for (let i = 0; i < faceOutline.length; i++) {
        const kp = face.keypoints[faceOutline[i]];
        const vx = kp.x - cx, vy = kp.y - cy;
        p.vertex(cx + vx * FACE_EXPAND, cy + vy * FACE_EXPAND);
      }
      p.endShape(p.CLOSE);
      p.pop();
    } else {
      g.maskedImg = null;
    }

    // decor: tegnes ovenpå og påvirkes ikke af masken
    for (const d of decos) d.draw();
  };

  // klik i paletten: opret nyt deco
  p.mousePressed = () => {
    const hit = hitPalette(p.mouseX, p.mouseY);
    if (hit) {
      const pos = faceCenterOrFallback();
      const img = p.assets[hit.key];

      const fb = currentFaceBounds();
      let base = fb ? Math.max(fb.w, fb.h) * 1 : 100;
      base = Math.max(80, Math.min(base, 240));

      decos.push(new Deco(p, img, pos.x, pos.y, base, base));
      return;
    }
    // grib eksisterende deco (øverst først)
    for (let i = decos.length - 1; i >= 0; i--) {
      const d = decos[i];
      if (d.contains(p.mouseX, p.mouseY)) {
        d.startDrag(p.mouseX, p.mouseY);
        const [grab] = decos.splice(i, 1);
        decos.push(grab);
        return;
      }
    }
  };

  p.mouseDragged = () => { for (const d of decos) d.drag(p.mouseX, p.mouseY); };
  p.mouseReleased = () => { for (const d of decos) d.stopDrag(); };

  // dobbeltklik: fjern øverste deco under musen
  p.doubleClicked = () => {
    for (let i = decos.length - 1; i >= 0; i--) {
      if (decos[i].contains(p.mouseX, p.mouseY)) { decos.splice(i, 1); break; }
    }
    return false;
  };

  function drawPalette(p) {
    p.noStroke(); p.fill(205); p.rect(0, 250, 330, 140);
    p.fill(30); p.textSize(12); p.text('Decor', 10, 265);
    for (const item of palette) {
      const img = p.assets[item.key];
      if (!img) continue;
      p.push(); p.imageMode(p.CENTER); p.image(img, item.x, item.y, item.w, item.h); p.pop();
      if (dist2(p.mouseX, p.mouseY, item.x, item.y) < (Math.max(item.w, item.h)/2)**2) {
        p.noFill(); p.stroke(0); p.strokeWeight(1);
        p.rect(item.x - item.w/2 - 4, item.y - item.h/2 - 4, item.w + 8, item.h + 8);
      }
    }
  }

  function hitPalette(mx, my) {
    for (const item of palette) {
      const dx = mx - item.x, dy = my - item.y;
      if (dx*dx + dy*dy <= (Math.max(item.w, item.h)/2)**2) return item;
    }
    return null;
  }

  function faceCenterOrFallback() {
    if (faces.length > 0) {
      let cx = 0, cy = 0;
      for (let i = 0; i < faceOutline.length; i++) {
        const kp = faces[0].keypoints[faceOutline[i]];
        cx += kp.x; cy += kp.y;
      }
      cx /= faceOutline.length; cy /= faceOutline.length;
      return { x: cx, y: cy };
    }
    return { x: VID_W/2, y: VID_H/2 };
  }
};

// hjælpere
function dist2(x1,y1,x2,y2){ const dx=x1-x2, dy=y1-y2; return dx*dx+dy*dy; }

function waitForVideo(videoEl, timeoutMs = 10000) {
  return new Promise((resolve, reject) => {
    const t0 = performance.now();
    const tick = () => {
      if (videoEl && videoEl.readyState >= 2) { videoEl.play?.().catch(()=>{}); return resolve(); }
      if (performance.now() - t0 > timeoutMs) return reject(new Error('video timeout'));
      requestAnimationFrame(tick);
    };
    tick();
  });
}

function waitForFaceModel(fm, timeoutMs = 15000) {
  return new Promise((resolve, reject) => {
    const t0 = performance.now();
    const tick = () => {
      const ready = !!fm && (
        (fm.model && typeof fm.model.estimateFaces === 'function') ||
        (fm.landmarkDetector && typeof fm.landmarkDetector.estimateFaces === 'function') ||
        (fm.detector && typeof fm.detector.estimateFaces === 'function')
      );
      if (ready) return resolve();
      if (performance.now() - t0 > timeoutMs) return reject(new Error('FaceMesh model load timeout'));
      requestAnimationFrame(tick);
    };
    tick();
  });
}

function waitForHandModel(hm, timeoutMs = 15000){
  return new Promise((resolve, reject) => {
    const t0 = performance.now();
    const tick = () => {
      const ready = !!hm && (
        (hm.model && typeof hm.model.estimateHands === 'function') ||
        (hm.detector && typeof hm.detector.estimateHands === 'function')
      );
      if (ready) return resolve();
      if (performance.now() - t0 > timeoutMs) return reject(new Error('Handpose model load timeout'));
      requestAnimationFrame(tick);
    };
    tick();
  });
}

// manuelt face-loop
function startManualFaceLoop() {
  let alive = true;
  const step = async () => {
    if (!alive) return;
    try {
      const res = await faceMesh.detect(g.video.elt);
      faces = Array.isArray(res) ? res : (res?.faces || []);
    } catch {}
    requestAnimationFrame(step);
  };
  requestAnimationFrame(step);
  stopManualFaceDetect = () => { alive = false; };
}

// manuelt hand-loop
function startManualHandLoop() {
  let alive = true;
  const step = async () => {
    if (!alive) return;
    try {
      const res = await handModel.detect(g.video.elt);
      hands = Array.isArray(res) ? res : (res?.hands || res?.[0]?.hands || []);
    } catch {}
    requestAnimationFrame(step);
  };
  requestAnimationFrame(step);
  stopManualHandDetect = () => { alive = false; };
}

// MediaPipe fallback init og loop
async function initHandsFallback(videoEl, vidW, vidH) {
  const { FilesetResolver, HandLandmarker } = await import('@mediapipe/tasks-vision');

  const vision = await FilesetResolver.forVisionTasks(
    'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.14/wasm'
  );

  handLandmarker = await HandLandmarker.createFromOptions(vision, {
    baseOptions: {
      modelAssetPath: 'https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task'
    },
    numHands: 2,
    runningMode: 'VIDEO'
  });

  let alive = true;
  const step = () => {
    if (!alive || !handLandmarker || !videoEl) return;
    try {
      const res = handLandmarker.detectForVideo(videoEl, performance.now());
      hands = (res?.landmarks || []).map(pts => ({
        keypoints: pts.map(pt => ({ x: pt.x * vidW, y: pt.y * vidH }))
      }));
    } catch {}
    requestAnimationFrame(step);
  };
  requestAnimationFrame(step);
  handLoopStop = () => { alive = false; };
}

// ansigts-bounds
function currentFaceBounds(){
  if (faces.length === 0) return null;
  const face = faces[0];
  let minX = Infinity, maxX = -Infinity, minY = Infinity, maxY = -Infinity;
  for (let i = 0; i < faceOutline.length; i++) {
    const kp = face.keypoints[faceOutline[i]];
    if (kp.x < minX) minX = kp.x;
    if (kp.x > maxX) maxX = kp.x;
    if (kp.y < minY) minY = kp.y;
    if (kp.y > maxY) maxY = kp.y;
  }
  return { x: minX, y: minY, w: maxX - minX, h: maxY - minY, cx: (minX + maxX)/2, cy: (minY + maxY)/2 };
}

// alle hånd-bounds
function currentAllHandBounds() {
  const out = [];
  for (const h of (hands || [])) {
    const pts = h?.keypoints || h?.landmarks || [];
    if (!pts || pts.length === 0) continue;
    let minX = Infinity, maxX = -Infinity, minY = Infinity, maxY = -Infinity;
    for (const kp of pts) {
      const x = kp.x ?? kp[0], y = kp.y ?? kp[1];
      if (x < minX) minX = x;
      if (x > maxX) maxX = x;
      if (y < minY) minY = y;
      if (y > maxY) maxY = y;
    }
    out.push({ x: minX, y: minY, w: maxX - minX, h: maxY - minY, cx: (minX + maxX)/2, cy: (minY + maxY)/2 });
  }
  return out;
}

async function buildStickerPNG(p) {
  if (!g.maskedImg || faces.length === 0) return null;

  const fb = currentFaceBounds();
  const size  = (Math.max(fb.w, fb.h) * 2) | 0;
  const centerX = fb.cx, centerY = fb.cy;

  const square = p.createGraphics(size, size);
  square.pixelDensity(1);
  square.clear();

  const offsetX = size / 2 - centerX;
  const offsetY = size / 2 - centerY;
  square.image(g.maskedImg, offsetX, offsetY);

  // face-outline ovenpå
  square.push();
  square.noFill();
  square.stroke(255);
  square.strokeWeight(10);
  square.beginShape();
  for (let i = 0; i < faceOutline.length; i++) {
    const kp = faces[0].keypoints[faceOutline[i]];
    const vx = kp.x - fb.cx, vy = kp.y - fb.cy;
    const ex = fb.cx + vx * FACE_EXPAND;
    const ey = fb.cy + vy * FACE_EXPAND;
    const px = ex - centerX + size / 2;
    const py = ey - centerY + size / 2;
    square.vertex(px, py);
  }
  square.endShape(square.CLOSE);
  square.pop();

  // læg decos ovenpå
  for (const d of decos) {
    const angle = Math.atan2(d.y, d.x);
    const dx = d.x - centerX + size/2;
    const dy = d.y - centerY + size/2;

    square.push();
    square.translate(dx, dy);
    square.rotate(angle);
    square.imageMode(square.CENTER);
    square.image(d.img, 0, 0, d.w, d.h);
    square.pop();
  }

  return square.canvas.toDataURL('image/png');
}

async function pasteSticker() {
  if (!g.maskedImg || faces.length === 0) {
    console.log('Ingen face detekteret');
    return;
  }
  const p = pInstance;
  const dataUrl = await buildStickerPNG(p);
  if (!dataUrl) return;

  try {
    await window.miro?.board?.getInfo();
    const viewport = await window.miro.board.viewport.get();
    const { x, y, width, height } = viewport;
    const cx = x + width / 2, cy = y + height / 2;

    const created = await window.miro.board.createImage({
      url: dataUrl,
      x: cx,
      y: cy,
      title: 'sticker.png',
    });
    await window.miro.board.viewport.zoomTo(created);
  } catch (e) {
    console.warn('Ikke i Miro eller createImage fejlede. Gemmer lokalt.', e);
    const a = document.createElement('a');
    a.href = dataUrl; a.download = 'sticker.png'; a.click();
  }
}

onMounted(async () => {
  const p5Mod = await import('p5');
  P5 = p5Mod.default;
  const ml5Mod = await import('ml5');
  ml5 = ml5Mod.default;

  inMiroRef.value = await probeMiro();
  buttonLabel.value = inMiroRef.value ? 'Paste sticker on the board' : 'Save sticker locally';

  pInstance = new P5(makeSketch, canvasHost.value);
  window.__p5instance = pInstance;

  if (!inMiroRef.value) {
    console.warn('Kører uden for Miro. SDK-fejl kan ignoreres lokalt.');
  }
});

onBeforeUnmount(() => {
  try { faceMesh?.detectStop?.(); } catch {}
  try { handModel?.detectStop?.(); } catch {}
  try { stopManualFaceDetect?.(); } catch {}
  try { stopManualHandDetect?.(); } catch {}
  try { handLoopStop?.(); } catch {}
  try { handLandmarker?.close?.(); } catch {}
  try { pInstance?.remove?.(); } catch {}
});
</script>

<template>
  <div id="root">
    <div class="grid wrapper">
      <div class="canvas cs1 ce12">
        <div ref="canvasHost" style="min-height: 400px;"></div>
      </div>

      <div class="butn cs1 ce12" style="display:flex; gap:12px; align-items:center;">
        <button class="button button-primary" @click="pasteSticker">
          {{ buttonLabel }}
        </button>
        <span v-if="!inMiroRef" style="font-size:12px; opacity:.7;">(kører uden for Miro)</span>
      </div>
    </div>
  </div>
</template>

<style scoped>
.canvas { background: #ececec; border: 1px solid #eee; border-radius: 8px; padding: 8px; }
</style>
