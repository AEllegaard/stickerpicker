<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import './assets/style.css';

// kun hatten
import hatSrc from './assets/hat.webp';

const canvasHost = ref(null);

// UI state
const inMiroRef = ref(false);
const buttonLabel = ref('Save sticker locally');

// handles
let P5, ml5, pInstance = null, faceMesh = null;
let faces = [];

// deco-objekter på lærredet
let decos = [];

// fallback loop stopper
let stopManualDetect = null;

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
  angleToOrigin() {
    return Math.atan2(-this.y, this.x); // peg mod (0,0)
  }
  draw() {
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

    // kamera
    g.video = p.createCapture({ video: { facingMode: 'user', width: VID_W, height: VID_H }, audio: false });
    g.video.size(VID_W, VID_H);
    g.video.hide();
    Object.assign(g.video.elt.style, { position: 'absolute', left: '-10000px', width: '0px', height: '0px', opacity: '0' });
    g.video.elt.setAttribute('playsinline', 'true');

    // offscreen buffers
    g.pg    = p.createGraphics(VID_W, VID_H);
    g.maskG = p.createGraphics(VID_W, VID_H);

    // ml5 init
    (async () => {
      await waitForVideo(g.video.elt);

      try {
        const tf = ml5?.tf;
        if (tf?.setBackend) {
          try { await tf.setBackend('webgl'); await tf.ready(); }
          catch { await tf.setBackend('wasm'); await tf.ready(); }
        }
      } catch {}

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
        console.warn('detectStart fejlede, skifter til manuelt loop', e);
        startManualDetectLoop();
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

    if (faces.length > 0) {
      const face = faces[0];

      // centrum
      let cx = 0, cy = 0;
      for (let i = 0; i < faceOutline.length; i++) {
        const kp = face.keypoints[faceOutline[i]];
        cx += kp.x; cy += kp.y;
      }
      cx /= faceOutline.length; cy /= faceOutline.length;

      // maske
      g.maskG.beginShape();
      for (let i = 0; i < faceOutline.length; i++) {
        const kp = face.keypoints[faceOutline[i]];
        const vx = kp.x - cx, vy = kp.y - cy;
        g.maskG.vertex(cx + vx * FACE_EXPAND, cy + vy * FACE_EXPAND);
      }
      g.maskG.endShape(p.CLOSE);

      const masked = g.pg.get();
      masked.mask(g.maskG.get());
      g.maskedImg = masked;

      // ansigt
      p.image(g.maskedImg, 0, 0, VID_W, VID_H);

      // outline
      p.push();
      p.noFill();
      p.stroke(255);
      p.strokeWeight(10);
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

  // NYT: dobbeltklik fjerner øverste deco under musen
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

function startManualDetectLoop() {
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
  stopManualDetect = () => { alive = false; };
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


async function buildStickerPNG(p) {
  const face = faces[0];

  // bbox omkring face
  let minX = p.width, maxX = 0, minY = p.height, maxY = 0;
  for (let i = 0; i < faceOutline.length; i++) {
    const kp = face.keypoints[faceOutline[i]];
    if (kp.x < minX) minX = kp.x;
    if (kp.x > maxX) maxX = kp.x;
    if (kp.y < minY) minY = kp.y;
    if (kp.y > maxY) maxY = kp.y;
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
  square.image(g.maskedImg, offsetX, offsetY);

  // outline
  let cx = 0, cy = 0;
  for (let i = 0; i < faceOutline.length; i++) {
    const kp = face.keypoints[faceOutline[i]];
    cx += kp.x; cy += kp.y;
  }
  cx /= faceOutline.length; cy /= faceOutline.length;

  square.push();
  square.noFill();
  square.stroke(255);
  square.strokeWeight(10);
  square.beginShape();
  for (let i = 0; i < faceOutline.length; i++) {
    const kp = face.keypoints[faceOutline[i]];
    const vx = kp.x - cx, vy = kp.y - cy;
    const ex = cx + vx * FACE_EXPAND;
    const ey = cy + vy * FACE_EXPAND;
    const px = ex - centerX + size / 2;
    const py = ey - centerY + size / 2;
    square.vertex(px, py);
  }
  square.endShape(p.CLOSE);
  square.pop();

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

async function pasteSticker() {
  if (!g.maskedImg || faces.length === 0) {
    console.log('Ingen face detekteret');
    return;
  }

  const p = pInstance;
  const dataUrl = await buildStickerPNG(p);

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
  try { stopManualDetect?.(); } catch {}
  try { pInstance?.remove?.(); } catch {}
});
</script>

<template>
  <div id="root">
    <div class="grid wrapper">
      <div class="cs1 ce12">
        <h1>Sticker Picker</h1>
      </div>

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
