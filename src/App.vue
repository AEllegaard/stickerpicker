<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import './assets/style.css';

// import deco-ressourcer
import hatSrc from './assets/hat.webp';

const canvasHost = ref(null);

// UI state
const inMiroRef = ref(false);
const buttonLabel = ref('Save sticker locally');

// handles
let P5, ml5, pInstance = null, faceMesh = null, handpose = null;
let faces = [];
let hands = [];

// deco-objekter på lærredet
let decos = [];

// fallback loop stopper
let stopManualFaceDetect = null;
let stopManualHandDetect = null;

// face konstanter
const FACE_EXPAND = 1.3;
const faceOutline = [10,338,297,332,284,251,389,356,447,366,401,288,397,365,379,378,400,377,152,148,176,149,150,136,172,58,132,93,234,127,162,103,67];

// hånd konstanter (brug alle punkter + konveks hylster)
const HAND_EXPAND = 1.15; // let op pustning af hylster

// tegne-ressourcer
let g = {
  video: null,
  pg: null,
  maskG: null,
  maskedImg: null,
};

const VID_W = 330, VID_H = 240;

// palette-ikoner (nede i panelet)
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
    this.p = p;
    this.img = img;
    this.x = x; this.y = y;
    this.w = w; this.h = h;
    this.dragging = false;
    this._dx = 0; this._dy = 0;
  }
  contains(mx, my) {
    return Math.abs(mx - this.x) <= this.w/2 && Math.abs(my - this.y) <= this.h/2;
  }
  //drag handling
  startDrag(mx, my) {
    this.dragging = true;
    this._dx = mx - this.x;
    this._dy = my - this.y;
  }
  drag(mx, my) {
    if (!this.dragging) return;
    this.x = mx - this._dx;
    this.y = my - this._dy;
  }
  stopDrag() { this.dragging = false; }
//peger decor mod centrum, og flipper på midten
  angleToOrigin() {
    if (this.x > VID_W/2 + 20){
      return Math.atan2(this.y, this.x); // peger mod (0,0)

    } if (this.x < VID_W/2 - 20){
      return Math.atan2(-this.y, this.x); // peger mod (0,0) spejlet  
    }
    return 0;
  }
  draw() {
    //tegner decor på ansigtet. 
    const p = this.p;
    p.push();
    p.translate(this.x, this.y);
    p.rotate(this.angleToOrigin());
    p.imageMode(p.CENTER);
    p.image(this.img, 0, 0, this.w, this.h);
    p.pop();
  }
}

const makeSketch = (p) => {
  p.assets = {};

  p.preload = () => {
    p.assets.hat = p.loadImage(hatSrc);
  };

  p.setup = () => {
    p.pixelDensity(1);
    p.angleMode(p.RADIANS);
    p.createCanvas(330, 390);

    // opretter kamera og skjuler det 
    g.video = p.createCapture({ video: { facingMode: 'user', width: VID_W, height: VID_H }, audio: false });
    g.video.size(VID_W, VID_H);
    g.video.hide();
    Object.assign(g.video.elt.style, { position: 'absolute', left: '-10000px', width: '0px', height: '0px', opacity: '0' });
    g.video.elt.setAttribute('playsinline', 'true');

    // offscreen buffers
    g.pg    = p.createGraphics(VID_W, VID_H);
    g.maskG = p.createGraphics(VID_W, VID_H);

    // ml5 initialisering
    (async () => {
      await waitForVideo(g.video.elt);

      try {
        const tf = ml5?.tf;
        if (tf?.setBackend) {
          try { await tf.setBackend('webgl'); await tf.ready(); }
          catch { await tf.setBackend('wasm'); await tf.ready(); }
        }
      } catch {}

      // FaceMesh
      faceMesh = await ml5.faceMesh({
        maxFaces: 1,
        refineLandmarks: false,
        flipped: true
      });
      await waitForModel(faceMesh, 15000);

      try {
        faceMesh.detectStart(g.video.elt, (results) => { faces = results || []; });
        console.log('FaceMesh detectStart kører');
      } catch (e) {
        console.warn('Face detectStart fejlede, skifter til manuelt loop', e);
        startManualFaceDetectLoop();
      }

      // Hands
      try {
        handpose = await ml5.handpose({
          flipped: true,
          maxHands: 2,
        });
      } catch (e) {
        console.warn('ml5.handpose kunne ikke initialiseres', e);
      }

      if (handpose) {
        try {
          // nogle versioner har detectStart
          handpose.detectStart?.(g.video.elt, (res) => { hands = normalizeHands(res); });
          if (!handpose.detectStart) throw new Error('ingen detectStart på handpose');
          console.log('Handpose detectStart kører');
        } catch (e) {
          console.warn('Hand detectStart fejlede, skifter til manuelt loop', e);
          startManualHandDetectLoop();
        }
      }
    })().catch(console.error);
  };

  p.draw = () => {
    p.background(220);

    // ryd buffers
    g.pg.clear();
    g.maskG.clear();
    g.maskG.noStroke();
    g.maskG.fill(255);

    // spejl video ind i pg
    if (g.video) {
      g.pg.push();
      g.pg.translate(VID_W, 0);
      g.pg.scale(-1, 1);
      g.pg.image(g.video, 0, 0, VID_W, VID_H);
      g.pg.pop();
    }

    // dekorationspanel
    drawPalette(p);

    // byg maske: ansigt + hænder
    let anyMask = false;

    if (faces.length > 0) {
      anyMask = true;
      const fPts = faceExpandedOutlinePoints(faces[0]);
      drawPolygon(g.maskG, fPts);
    }

    if (hands.length > 0) {
      for (const h of hands) {
        const hull = handExpandedHullPoints(h);
        if (hull && hull.length >= 3) {
          anyMask = true;
          drawPolygon(g.maskG, hull);
        }
      }
    }

    if (anyMask) {
      const masked = g.pg.get();
      masked.mask(g.maskG.get());
      g.maskedImg = masked;
      p.image(g.maskedImg, 0, 0, VID_W, VID_H);

      // visuelle outlines
      if (faces.length > 0) drawPolyline(p, faceExpandedOutlinePoints(faces[0]));
      if (hands.length > 0) {
        for (const h of hands) {
          const hull = handExpandedHullPoints(h);
          if (hull && hull.length >= 3) drawPolyline(p, hull);
        }
      }
    } else {
      g.maskedImg = null;
    }

    // tegn alle decos ovenpå
    for (const d of decos) d.draw();
  };

  // klik i paletten: opret nyt deco
  p.mousePressed = () => {
    const hit = hitPalette(p.mouseX, p.mouseY);
    if (hit) {
      const pos = faceCenterOrFallback();
      const img = p.assets[hit.key];

      //find ansigts bredden
      const fb = currentFaceBounds();

      //basis på 70% af ansigts bredden
      let base = fb ? Math.max(fb.w, fb.h) * 1 : 100;

      //hold den inden for grænserne
      base = Math.max(80, Math.min(base, 220));

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

  //dobbeltklik fjerner øverste deco under musen
  p.doubleClicked = () => {
    for (let i = decos.length - 1; i >= 0; i--) {
      if (decos[i].contains(p.mouseX, p.mouseY)) {
        decos.splice(i, 1);
        break;
      }
    }
    return false; // forhindre evt. default
  };

  function drawPalette(p) {
    p.noStroke();
    p.fill(205);
    p.rect(0, 250, 330, 140);

    p.fill(30);
    p.textSize(12);
    p.text('Decor', 10, 265);

    for (const item of palette) {
      const img = p.assets[item.key];
      if (!img) continue;
      p.push();
      p.imageMode(p.CENTER);
      p.image(img, item.x, item.y, item.w, item.h);
      p.pop();

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
      const { cx, cy } = currentFaceBounds();
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
      if (videoEl && videoEl.readyState >= 2) {
        videoEl.play?.().catch(()=>{});
        return resolve();
      }
      if (performance.now() - t0 > timeoutMs) return reject(new Error('video timeout'));
      requestAnimationFrame(tick);
    };
    tick();
  });
}

function waitForModel(fm, timeoutMs = 15000) {
  return new Promise((resolve, reject) => {
    const t0 = performance.now();
    const tick = () => {
      const ready =
        !!fm &&
        (
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

function startManualFaceDetectLoop() {
  let alive = true;
  const step = async () => {
    if (!alive) return;
    try {
      const res = await faceMesh.detect(g.video.elt);
      faces = Array.isArray(res) ? res : (res?.faces || []);
    } catch {}
    requestAnimationFrame(step);
  };
  step();
  stopManualFaceDetect = () => { alive = false; };
}

function startManualHandDetectLoop() {
  if (!handpose) return;
  let alive = true;
  const step = async () => {
    if (!alive) return;
    try {
      const res = await handpose.detect(g.video.elt);
      hands = normalizeHands(res);
    } catch {}
    requestAnimationFrame(step);
  };
  step();
  stopManualHandDetect = () => { alive = false; };
}

function normalizeHands(res) {
  // ml5.handpose kan returnere forskelligt format på tværs af versioner
  if (!res) return [];
  if (Array.isArray(res)) return res; // allerede en liste
  if (res.hands) return res.hands; // evt. wrapper
  if (res.predictions) return res.predictions; // ældre tfjs handpose
  return [];
}

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
  return { 
    x: minX,
    y: minY,
    w: maxX - minX,
    h: maxY - minY,
    cx: (minX + maxX) / 2,
    cy: (minY + maxY) / 2
  };
}

function faceExpandedOutlinePoints(face) {
  // beregn centrum
  let cx = 0, cy = 0;
  for (let i = 0; i < faceOutline.length; i++) {
    const kp = face.keypoints[faceOutline[i]];
    cx += kp.x; cy += kp.y;
  }
  cx /= faceOutline.length; cy /= faceOutline.length;
  const pts = [];
  for (let i = 0; i < faceOutline.length; i++) {
    const kp = face.keypoints[faceOutline[i]];
    const vx = kp.x - cx, vy = kp.y - cy;
    pts.push({ x: cx + vx * FACE_EXPAND, y: cy + vy * FACE_EXPAND });
  }
  return pts;
}

function handExpandedHullPoints(hand) {
  // hånd punkter kan hedde keypoints eller landmarks
  const pts = (hand.keypoints || hand.landmarks || []).map(pt => ({ x: pt.x ?? pt[0], y: pt.y ?? pt[1] }));
  if (pts.length < 3) return null;
  const hull = convexHull(pts);
  // pust ud fra centroid
  let cx = 0, cy = 0;
  for (const p of hull) { cx += p.x; cy += p.y; }
  cx /= hull.length; cy /= hull.length;
  const out = hull.map(p => ({ x: cx + (p.x - cx) * HAND_EXPAND, y: cy + (p.y - cy) * HAND_EXPAND }));
  return out;
}

// Monotonic chain konveks hylster
function convexHull(points) {
  const pts = points.slice().sort((a,b)=> a.x===b.x ? a.y-b.y : a.x-b.x);
  if (pts.length <= 1) return pts;
  const cross = (o,a,b)=> (a.x-o.x)*(b.y-o.y) - (a.y-o.y)*(b.x-o.x);
  const lower = [];
  for (const p of pts) {
    while (lower.length >= 2 && cross(lower[lower.length-2], lower[lower.length-1], p) <= 0) lower.pop();
    lower.push(p);
  }
  const upper = [];
  for (let i = pts.length-1; i>=0; i--) {
    const p = pts[i];
    while (upper.length >= 2 && cross(upper[upper.length-2], upper[upper.length-1], p) <= 0) upper.pop();
    upper.push(p);
  }
  upper.pop(); lower.pop();
  return lower.concat(upper);
}

function drawPolygon(gfx, pts) {
  gfx.beginShape();
  for (const p of pts) gfx.vertex(p.x, p.y);
  gfx.endShape(gfx._renderer?.p?.CLOSE ?? 'close');
}

function drawPolyline(p, pts) {
  p.push();
  p.noFill();
  p.stroke(255);
  p.strokeWeight(10);
  p.beginShape();
  for (const v of pts) p.vertex(v.x, v.y);
  p.endShape(p.CLOSE);
  p.pop();
}

async function buildStickerPNG(p) {
  // saml alle punkter der definerer omkredsen af sticker
  const shapePts = [];
  if (faces.length > 0) shapePts.push(...faceExpandedOutlinePoints(faces[0]));
  for (const h of hands) {
    const hull = handExpandedHullPoints(h);
    if (hull) shapePts.push(...hull);
  }
  if (shapePts.length < 3) return null;

  // bbox
  let minX = p.width, maxX = 0, minY = p.height, maxY = 0;
  for (const pt of shapePts) {
    if (pt.x < minX) minX = pt.x;
    if (pt.x > maxX) maxX = pt.x;
    if (pt.y < minY) minY = pt.y;
    if (pt.y > maxY) maxY = pt.y;
  }
  const faceW = maxX - minX;
  const faceH = maxY - minY;
  const size = (Math.max(faceW, faceH) * 2) | 0;
  const centerX = (minX + maxX) / 2;
  const centerY = (minY + maxY) / 2;

  // byg sticker på offscreen square
  const square = p.createGraphics(size, size);
  square.pixelDensity(1);
  square.clear();

  const offsetX = size / 2 - centerX;
  const offsetY = size / 2 - centerY;

  // rekonstruer masken lokalt
  const tmpMask = p.createGraphics(VID_W, VID_H);
  tmpMask.clear();
  tmpMask.noStroke(); tmpMask.fill(255);
  if (faces.length > 0) drawPolygon(tmpMask, faceExpandedOutlinePoints(faces[0]));
  for (const h of hands) {
    const hull = handExpandedHullPoints(h);
    if (hull) drawPolygon(tmpMask, hull);
  }

  const videoCopy = g.pg.get();
  videoCopy.mask(tmpMask.get());

  square.image(videoCopy, offsetX, offsetY);

  // tegne outlines
  if (faces.length > 0) drawPolylineLocal(square, faceExpandedOutlinePoints(faces[0]), centerX, centerY, size);
  for (const h of hands) {
    const hull = handExpandedHullPoints(h);
    if (hull) drawPolylineLocal(square, hull, centerX, centerY, size);
  }

  // læg alle decos ovenpå (samme rotation)
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

  const dataUrl = square.canvas.toDataURL('image/png');
  return dataUrl;
}

function drawPolylineLocal(square, pts, cx, cy, size) {
  square.push();
  square.noFill();
  square.stroke(255);
  square.strokeWeight(10);
  square.beginShape();
  for (const v of pts) {
    const px = v.x - cx + size / 2;
    const py = v.y - cy + size / 2;
    square.vertex(px, py);
  }
  square.endShape(square.CLOSE);
  square.pop();
}

async function pasteSticker() {
  if (!g.pg) return;
  const p = pInstance;
  const dataUrl = await buildStickerPNG(p);
  if (!dataUrl) {
    console.log('Intet motiv fundet');
    return;
  }

  try {
    await window.miro?.board?.getInfo();

    const viewport = await window.miro.board.viewport.get();
    const { x, y, width, height } = viewport;
    const cx = x + width / 2;
    const cy = y + height / 2;

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
    a.href = dataUrl;
    a.download = 'sticker.png';
    a.click();
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
  try { handpose?.detectStop?.(); } catch {}
  try { stopManualFaceDetect?.(); } catch {}
  try { stopManualHandDetect?.(); } catch {}
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
