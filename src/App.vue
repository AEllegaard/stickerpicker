<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import './assets/style.css';

const canvasHost = ref(null);

// UI state
const inMiroRef = ref(false);
const buttonLabel = ref('Save sticker locally');

// handles
let P5, ml5, pInstance = null, faceMesh = null;
let faces = [];

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

// robust Miro-detektion: prøv et SDK-kald
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

const makeSketch = (p) => {
  p.setup = () => {
    p.pixelDensity(1);

    const cnv = p.createCanvas(330, 390);
    cnv.parent(canvasHost.value);

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
      // vent på video
      await waitForVideo(g.video.elt);

      // vælg tf backend
      try {
        const tf = ml5?.tf;
        if (tf?.setBackend) {
          try { await tf.setBackend('webgl'); await tf.ready(); }
          catch { await tf.setBackend('wasm'); await tf.ready(); }
        }
      } catch {}

      // facemesh
      faceMesh = await ml5.faceMesh({
        maxFaces: 1,
        refineLandmarks: false,
        flipped: true
      });

      // vent på intern model
      await waitForModel(faceMesh, 15000);

      // stream-detection
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
    // previewen må gerne have baggrund, men PNG’en vi bygger er separat og gennemsigtig
    p.background(220);

    // ryd buffers
    g.pg.clear();
    g.maskG.clear();
    g.maskG.noStroke();
    g.maskG.fill(255);

    // spejl video ind i pg, så pixels matcher flipped keypoints
    if (g.video) {
      g.pg.push();
      g.pg.translate(VID_W, 0);
      g.pg.scale( -1, 1 );
      g.pg.image(g.video, 0, 0, VID_W, VID_H);
      g.pg.pop();
    }

    //decor
      p.noStroke();
      p.fill(205);
      p.rect(0,250, 330, 150)

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

      // frisk kopi hver frame, da mask() er destruktiv
      const masked = g.pg.get();
      masked.mask(g.maskG.get());
      g.maskedImg = masked;

      // tegn det udklippede ansigt i preview
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
  };
};

// hjælpere
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

// fallback: manuelt detection-loop
function startManualDetectLoop() {
  let alive = true;
  const step = async () => {
    if (!alive) return;
    try {
      const res = await faceMesh.detect(g.video.elt);
      faces = Array.isArray(res) ? res : (res?.faces || []);
    } catch {
      // prøv igen næste frame
    }
    requestAnimationFrame(step);
  };
  step();
  stopManualDetect = () => { alive = false; };
}

async function buildStickerPNG(p) {
  // kræver g.maskedImg og faces[0]
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

  // byg endelig sticker i offscreen square med gennemsigtig baggrund
  const square = p.createGraphics(size, size);
  square.pixelDensity(1);
  square.clear(); // gennemsigtig baggrund

  const offsetX = size / 2 - centerX;
  const offsetY = size / 2 - centerY;
  square.image(g.maskedImg, offsetX, offsetY); // maskedImg har alfa uden for masken

  // outline ovenpå
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

  // data URL med alfa
  const dataUrl = square.canvas.toDataURL('image/png');
  return dataUrl;
}

async function pasteSticker() {
  if (!g.maskedImg || faces.length === 0) {
    console.log('Ingen face detekteret');
    return;
  }

  const p = pInstance;
  const dataUrl = await buildStickerPNG(p); // "data:image/png;base64,...."

  try {
    // er vi i Miro og har board access?
    await window.miro?.board?.getInfo();

    // 👉 v2 kræver `url:` – data-URL er OK; blob: er IKKE OK
  // hent brugerens aktuelle viewport
const viewport = await window.miro.board.viewport.get();
const { x, y, width, height } = viewport;

// beregn centrum
const cx = x + width / 2;
const cy = y + height / 2;

// indsæt billedet dér
const created = await window.miro.board.createImage({
  url: dataUrl,
  x: cx,
  y: cy,
  title: 'sticker.png',
});

// zoom pænt ind på det nye element
await window.miro.board.viewport.zoomTo(created);


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

  pInstance = new P5(makeSketch);
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
