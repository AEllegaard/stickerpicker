<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import './assets/style.css';

// import deco-ressourcer
//hats
import BeanieSrc from './assets/png/Beanie.png';
import BeretSrc from './assets/png/Beret.png';
import BuckethatSrc from './assets/png/Buckethat.png';
import CapSrc from './assets/png/Cap.png';
import CowboyhatSrc from './assets/png/Cowboyhat.png';
import LadyhatSrc from './assets/png/Ladyhat.png';
import PartyhatoneSrc from './assets/png/Partyhatone.png';
import PartyhattwoSrc from './assets/png/Partyhattwo.png';
import PartyhatthreeSrc from './assets/png/Partyhatthree.png';
import PropelSrc from './assets/png/Propel.png';
import TallhatSrc from './assets/png/Tallhat.png';

//Glasses
import CatsunniesSrc from './assets/png/Catsunnies.png';
import FlowersunniesSrc from './assets/png/flowersunnies.png';
import HeartsunniesSrc from './assets/png/HeartSunnies.png';
import PixelsunniesSrc from './assets/png/Pixelsunnies.png';
import SquaresunniesSrc from './assets/png/Squaresunnies.png';
import StarSunniesSrc from './assets/png/Starsunnies.png';

//Mouths
import BiglipsSrc from './assets/png/Biglips.png';
import BluetoungeoutSrc from './assets/png/Bluetoungeout.png';
import GreenmouthSrc from './assets/png/Greenmouth.png';
import HappylaydSrc from './assets/png/Happylady.png';
import NoteethSrc from './assets/png/Noteeth.png';
import OpensmileSrc from './assets/png/Opensmile.png';
import PinkmouthSrc from './assets/png/Pinkmouth.png';
import PinktoungeoutSrc from './assets/png/Pinktoungeout.png';
import UnderbiteSrc from './assets/png/Underbite.png';


const canvasHost = ref(null);

// UI state
const inMiroRef = ref(false);
const buttonLabel = ref('Save sticker locally');

const loading = ref(true); // loading state for face mesh

// handles
let P5, ml5, pInstance = null, faceMesh = null;
// let handposeModel = null, tf = null, handDetector = null;
let faces = [];
// let hands = [];

// deco-objekter på lærredet
let decos = [];

// fallback loop stopper
let stopManualFaceDetect = null;
// let stopManualHandDetect = null;

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

// kategorier og deres assets
const categories = [
  { id: 'hats', label: 'Hats' },
  { id: 'glasses', label: 'Glasses' },
  { id: 'mouths', label: 'Mouths' }
];

let selectedCategory = 'hats'; // state for selected category

// palette-ikoner (nede i panelet)
const palette = [
  //hats
  { category: 'hats', key: 'Beanie', src: BeanieSrc, x: 55, y: 320, w: 60, h: 60 },
  { category: 'hats', key: 'Beret', src: BeretSrc, x: 115, y: 320, w: 60, h: 60 },
  { category: 'hats', key: 'Buckethat', src: BuckethatSrc, x: 175, y: 320, w: 60, h: 60 },
  { category: 'hats', key: 'Cap', src: CapSrc, x: 235, y: 320, w: 60, h: 60 },
  { category: 'hats', key: 'Cowboyhat', src: CowboyhatSrc, x: 295, y: 320, w: 60, h: 60 },
  { category: 'hats', key: 'Ladyhat', src: LadyhatSrc, x: 55, y: 380, w: 60, h: 60 },
  { category: 'hats', key: 'Partyhatone', src: PartyhatoneSrc, x: 115, y: 380, w: 60, h: 60 },
  { category: 'hats', key: 'Partyhattwo', src: PartyhattwoSrc, x: 175, y: 380, w: 60, h: 60 },
  { category: 'hats', key: 'Partyhatthree', src: PartyhatthreeSrc, x: 235, y: 380, w: 60, h: 60 },
  { category: 'hats', key: 'Propel', src: PropelSrc, x: 295, y: 380, w: 60, h: 60 },
  { category: 'hats', key: 'Tallhat', src: TallhatSrc, x: 55, y: 440, w: 60, h: 60 },

  //Glasses
  { category: 'glasses', key: 'Catsunnies', src: CatsunniesSrc, x: 115, y: 320, w: 60, h: 60 },
  { category: 'glasses', key: 'Flowersunnies', src: FlowersunniesSrc, x: 175, y: 320, w: 60, h: 60 },
  { category: 'glasses', key: 'Heartsunnies', src: HeartsunniesSrc, x: 235, y: 320, w: 60, h: 60 },
  { category: 'glasses', key: 'Pixelsunnies', src: PixelsunniesSrc, x: 295, y: 320, w: 60, h: 60 },
  { category: 'glasses', key: 'Squaresunnies', src: SquaresunniesSrc, x: 55, y: 380, w: 60, h: 60 },
  { category: 'glasses', key: 'Starsunnies', src: StarSunniesSrc, x: 115, y: 380, w: 60, h: 60 },

  //Mouths
  { category: 'mouths', key: 'Biglips', src: BiglipsSrc, x: 175, y: 380, w: 60, h: 60 },
  { category: 'mouths', key: 'Bluetoungeout', src: BluetoungeoutSrc, x: 235, y: 380, w: 60, h: 60 },
  { category: 'mouths', key: 'Greenmouth', src: GreenmouthSrc, x: 295, y: 380, w: 60, h: 60 },
  { category: 'mouths', key: 'Happylady', src: HappylaydSrc, x: 55, y: 440, w: 60, h: 60 },
  { category: 'mouths', key: 'Noteeth', src: NoteethSrc, x: 115, y: 440, w: 60, h: 60 },
  { category: 'mouths', key: 'Opensmile', src: OpensmileSrc, x: 175, y: 440, w: 60, h: 60 },
  { category: 'mouths', key: 'Pinkmouth', src: PinkmouthSrc, x: 235, y: 440, w: 60, h: 60 },
  { category: 'mouths', key: 'Pinktoungeout', src: PinktoungeoutSrc, x: 295, y: 440, w: 60, h: 60 },
  { category: 'mouths', key: 'Underbite', src: UnderbiteSrc, x: 55, y: 500, w: 60, h: 60 }
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
  // peger decor mod centrum, og flipper på midten
  angleToOrigin() {
    if (this.x > VID_W/2 + 20){
      return Math.atan2(this.y, this.x);
    } if (this.x < VID_W/2 - 20){
      return Math.atan2(-this.y, this.x);
    }
    return 0;
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
    for (const item of palette) {
      p.assets[item.key] = p.loadImage(item.src);
    }
  };

  p.setup = () => {
    p.pixelDensity(1);
    p.angleMode(p.RADIANS);
    p.createCanvas(330, 490);

    // opretter kamera og skjuler det
    g.video = p.createCapture({ video: { facingMode: 'user', width: VID_W, height: VID_H }, audio: false });
    g.video.size(VID_W, VID_H);
    g.video.hide();
    Object.assign(g.video.elt.style, { position: 'absolute', left: '-10000px', width: '0px', height: '0px', opacity: '0' });
    g.video.elt.setAttribute('playsinline', 'true');

    // offscreen buffers
    g.pg    = p.createGraphics(VID_W, VID_H);
    g.maskG = p.createGraphics(VID_W, VID_H);

    // initialisering
    (async () => {
      await waitForVideo(g.video.elt);

      // tf backend (deles af ml5 og TFJS)
      try {
        const _tf = ml5?.tf || tf;
        if (_tf?.setBackend) {
          try { await _tf.setBackend('webgl'); await _tf.ready(); }
          catch { await _tf.setBackend('wasm'); await _tf.ready(); }
        }
      } catch {}

      // FaceMesh via ml5
      faceMesh = await ml5.faceMesh({
        maxFaces: 1,
        refineLandmarks: false,
        flipped: true
      });
      await waitForModel(faceMesh, 15000);

      try {
        faceMesh.detectStart(g.video.elt, (results) => { faces = results || []; });
        console.log('FaceMesh detectStart kører');
        loading.value = false; // face mesh is running, hide loading
      } catch (e) {
        console.warn('Face detectStart fejlede, skifter til manuelt loop', e);
        startManualFaceDetectLoop();
        loading.value = false; // even fallback, hide loading
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

    // byg masken: ansigt + hænder
    let anyMask = false;

    // ansigt
    if (faces.length > 0) {
      anyMask = true;
      const fPts = faceExpandedOutlinePoints(faces[0]);
      drawPolygon(g.maskG, fPts);
    }

    if (anyMask) {
      // anvend masken
      const masked = g.pg.get();
      masked.mask(g.maskG.get());
      g.maskedImg = masked;
      p.image(g.maskedImg, 0, 0, VID_W, VID_H);

      // samlet kontur udledt af masken
      const outline = extractUnifiedOutlineFromMask(g.maskG, 128);
      if (outline && outline.length > 1) {
        drawPolyline(p, outline);
      }
    } else {
      g.maskedImg = null;
    }

    // tegn alle decos ovenpå
    for (const d of decos) d.draw();
  };

  // klik i paletten: opret nyt deco
  p.mousePressed = () => {
    // Check if clicking on category button
    const catClick = getCategoryButtonAt(p.mouseX, p.mouseY);
    if (catClick) {
      selectedCategory = catClick;
      return;
    }
    
    const hit = hitPalette(p.mouseX, p.mouseY);
    if (hit) {
      const pos = faceCenterOrFallback();
      const img = p.assets[hit.key];

      const fb = currentFaceBounds();
      let base = fb ? Math.max(fb.w, fb.h) * 1 : 100;
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

  // dobbeltklik fjerner øverste deco under musen
  p.doubleClicked = () => {
    for (let i = decos.length - 1; i >= 0; i--) {
      if (decos[i].contains(p.mouseX, p.mouseY)) {
        decos.splice(i, 1);
        break;
      }
    }
    return false;
  };

  function drawPalette(p) {
    p.noStroke();
    p.fill(205);
    p.rect(0, 250, 330, 240);

    // Draw category buttons at top
    const btnWidth = 100;
    const btnHeight = 30;
    const btnY = 260;
    
    for (let i = 0; i < categories.length; i++) {
      const cat = categories[i];
      const btnX = 15 + i * 110;
      const isSelected = cat.id === selectedCategory;
      
      p.noStroke();
      p.fill(isSelected ? 100 : 170);
      p.rect(btnX, btnY, btnWidth, btnHeight, 5);
      
      p.fill(255);
      p.textSize(12);
      p.textStyle(p.BOLD);
      p.textAlign(p.CENTER, p.CENTER);
      p.text(cat.label, btnX + btnWidth/2, btnY + btnHeight/2);
      p.textAlign(p.LEFT);
    }

    // Draw assets for selected category
    const filteredItems = palette.filter(item => item.category === selectedCategory);
    
    for (const item of filteredItems) {
      const img = p.assets[item.key];
      if (!img) continue;
      
      p.push();
      p.imageMode(p.CENTER);
      p.image(img, item.x, item.y, item.w, item.h);
      p.pop();

      if (dist2(p.mouseX, p.mouseY, item.x, item.y) < (Math.max(item.w, item.h)/2)**2) {
        p.noFill(); p.stroke(0); p.strokeWeight(2);
        p.rect(item.x - item.w/2 - 4, item.y - item.h/2 - 4, item.w + 8, item.h + 8);
      }
    }
  }
  
  function getCategoryButtonAt(mx, my) {
    const btnWidth = 100;
    const btnHeight = 30;
    const btnY = 260;
    
    for (let i = 0; i < categories.length; i++) {
      const btnX = 15 + i * 110;
      if (mx >= btnX && mx <= btnX + btnWidth && my >= btnY && my <= btnY + btnHeight) {
        return categories[i].id;
      }
    }
    return null;
  }

  function hitPalette(mx, my) {
    for (const item of palette) {
      if (item.category !== selectedCategory) continue;
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

// ========= HÆNDERS MASKETEGNING =========

// afrundede fingersegmenter
function drawThickSegment(gfx, a, b, w) {
  gfx.push();
  gfx.stroke(255);
  gfx.strokeWeight(w);
  gfx.strokeCap(gfx.ROUND);
  gfx.noFill();
  gfx.line(a.x, a.y, b.x, b.y);
  gfx.pop();
}

function drawJointDot(gfx, p, w) {
  gfx.push();
  gfx.noStroke();
  gfx.fill(255);
  gfx.circle(p.x, p.y, w);
  gfx.pop();
}

function drawFilledPoly(gfx, pts) {
  gfx.push();
  gfx.noStroke();
  gfx.fill(255);
  gfx.beginShape();
  for (const p of pts) gfx.vertex(p.x, p.y);
  gfx.endShape(gfx.CLOSE);
  gfx.pop();
}

// Chaikin smoothing til blødere palmekanter (mild)
function smoothPolyChaikin(pts, iters = 1) {
  let poly = pts.slice();
  for (let t = 0; t < iters; t++) {
    const out = [];
    for (let i = 0; i < poly.length; i++) {
      const a = poly[i];
      const b = poly[(i + 1) % poly.length];
      out.push({ x: 0.75 * a.x + 0.25 * b.x, y: 0.75 * a.y + 0.25 * b.y });
      out.push({ x: 0.25 * a.x + 0.75 * b.x, y: 0.25 * a.y + 0.75 * b.y });
    }
    poly = out;
  }
  return poly;
}

function midPoint(a,b){ return { x:(a.x+b.x)/2, y:(a.y+b.y)/2 }; }
function clamp(v, lo, hi){ return Math.max(lo, Math.min(hi, v)); }

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

function drawPolylineLocalFromPts(square, pts) {
  square.push();
  square.noFill();
  square.stroke(255);
  square.strokeWeight(6); // thinner stroke for smoother outline
  square.strokeJoin(square.ROUND);
  square.strokeCap(square.ROUND);
  square.beginShape();
  for (const v of pts) square.vertex(v.x, v.y);
  square.endShape(square.CLOSE);
  square.pop();
}

// kontur via boundary tracing
function extractUnifiedOutlineFromMask(maskG, threshold = 128) {
  const w = maskG.width, h = maskG.height;
  maskG.loadPixels();
  const pix = maskG.pixels;

  const alphaAt = (x,y) => {
    if (x < 0 || y < 0 || x >= w || y >= h) return 0;
    return pix[4*(y*w + x) + 3] || 0;
  };

  let sx = -1, sy = -1;
  for (let y = 0; y < h && sy === -1; y++) {
    for (let x = 0; x < w; x++) {
      if (alphaAt(x,y) >= threshold) { sx = x; sy = y; break; }
    }
  }
  if (sy === -1) return [];

  const nbs = [
    {dx: 1, dy: 0}, {dx: 1, dy: 1}, {dx: 0, dy: 1}, {dx: -1, dy: 1},
    {dx: -1, dy: 0}, {dx: -1, dy: -1}, {dx: 0, dy: -1}, {dx: 1, dy: -1}
  ];

  let bx = sx, by = sy;
  let cx = sx - 1, cy = sy;

  const outline = [];
  outline.push({x: bx + 0.5, y: by + 0.5});

  let guard = 0;
  while (guard++ < w*h*8) {
    let startK = 0;
    for (let k = 0; k < 8; k++) {
      if (bx + nbs[k].dx === cx && by + nbs[k].dy === cy) { startK = k; break; }
    }
    let found = false;
    for (let i = 1; i <= 8; i++) {
      const k = (startK + i) % 8;
      const nx = bx + nbs[k].dx;
      const ny = by + nbs[k].dy;
      if (alphaAt(nx, ny) >= threshold) {
        cx = bx + nbs[(k + 7) % 8].dx;
        cy = by + nbs[(k + 7) % 8].dy;
        bx = nx; by = ny;
        outline.push({x: bx + 0.5, y: by + 0.5});
        found = true;
        break;
      }
    }
    if (!found) break;
    if (bx === sx && by === sy && cx === sx - 1 && cy === sy) break;
  }

  // mindre epsilon så outline bliver glattere
  return simplifyRDP(outline, 0.05);
}

function simplifyRDP(points, epsilon) {
  if (points.length < 3) return points.slice();
  const first = points[0], last = points[points.length - 1];

  let index = -1, distMax = 0;
  for (let i = 1; i < points.length - 1; i++) {
    const d = perpDistance(points[i], first, last);
    if (d > distMax) { index = i; distMax = d; }
  }
  if (distMax > epsilon) {
    const left = simplifyRDP(points.slice(0, index + 1), epsilon);
    const right = simplifyRDP(points.slice(index), epsilon);
    return left.slice(0, -1).concat(right);
  } else {
    return [first, last];
  }
}

function perpDistance(p, a, b) {
  const x = p.x, y = p.y;
  const x1 = a.x, y1 = a.y, x2 = b.x, y2 = b.y;
  const dx = x2 - x1, dy = y2 - y1;
  if (dx === 0 && dy === 0) return Math.hypot(x - x1, y - y1);
  const t = ((x - x1)*dx + (y - y1)*dy) / (dx*dx + dy*dy);
  const px = x1 + t*dx, py = y1 + t*dy;
  return Math.hypot(x - px, y - py);
}

// afstande mellem polygon og rektangel
function minDistancePolyToRect(poly, rect){
  if (polyIntersectsRect(poly, rect)) return 0;
  let best = Infinity;

  for (let i = 0; i < poly.length; i++){
    const a = poly[i], b = poly[(i+1)%poly.length];
    best = Math.min(best, segmentToRectDistance(a, b, rect));
  }
  const corners = [
    {x: rect.x, y: rect.y},
    {x: rect.x + rect.w, y: rect.y},
    {x: rect.x + rect.w, y: rect.y + rect.h},
    {x: rect.x, y: rect.y + rect.h},
  ];
  for (const c of corners){
    best = Math.min(best, pointToPolyDistance(c, poly));
  }
  return best;
}

function polyIntersectsRect(poly, rect){
  for (const p of poly){
    if (p.x >= rect.x && p.x <= rect.x + rect.w && p.y >= rect.y && p.y <= rect.y + rect.h) return true;
  }
  const corners = [
    {x: rect.x, y: rect.y},
    {x: rect.x + rect.w, y: rect.y},
    {x: rect.x + rect.w, y: rect.y + rect.h},
    {x: rect.x, y: rect.y + rect.h},
  ];
  for (const c of corners){
    if (pointInPoly(c, poly)) return true;
  }
  for (let i = 0; i < poly.length; i++){
    const a = poly[i], b = poly[(i+1)%poly.length];
    if (segmentIntersectsRect(a, b, rect)) return true;
  }
  return false;
}

function pointInPoly(pt, poly){
  let inside = false;
  for (let i = 0, j = poly.length - 1; i < poly.length; j = i++){
    const xi = poly[i].x, yi = poly[i].y, xj = poly[j].x, yj = poly[j].y;
    const intersect = ((yi > pt.y) !== (yj > pt.y)) && (pt.x < (xj - xi) * (pt.y - yi) / ((yj - yi) || 1e-9) + xi);
    if (intersect) inside = !inside;
  }
  return inside;
}

function segmentIntersectsRect(a, b, rect){
  const edges = [
    [{x: rect.x, y: rect.y}, {x: rect.x + rect.w, y: rect.y}],
    [{x: rect.x + rect.w, y: rect.y}, {x: rect.x + rect.w, y: rect.y + rect.h}],
    [{x: rect.x + rect.w, y: rect.y + rect.h}, {x: rect.x, y: rect.y + rect.h}],
    [{x: rect.x, y: rect.y + rect.h}, {x: rect.x, y: rect.y}],
  ];
  for (const [p1, p2] of edges){
    if (segmentsIntersect(a, b, p1, p2)) return true;
  }
  return false;
}

function segmentsIntersect(p1, p2, p3, p4){
  const d = (p4.y - p3.y)*(p2.x - p1.x) - (p4.x - p3.x)*(p2.y - p1.y);
  if (Math.abs(d) < 1e-9) return false;
  const ua = ((p4.x - p3.x)*(p1.y - p3.y) - (p4.y - p3.y)*(p1.x - p3.x)) / d;
  const ub = ((p2.x - p1.x)*(p1.y - p3.y) - (p2.y - p1.y)*(p1.x - p3.x)) / d;
  return ua >= 0 && ua <= 1 && ub >= 0 && ub <= 1;
}

function segmentToRectDistance(a, b, rect){
  const edges = [
    [{x: rect.x, y: rect.y}, {x: rect.x + rect.w, y: rect.y}],
    [{x: rect.x + rect.w, y: rect.y}, {x: rect.x + rect.w, y: rect.y + rect.h}],
    [{x: rect.x + rect.w, y: rect.y + rect.h}, {x: rect.x, y: rect.y + rect.h}],
    [{x: rect.x, y: rect.y + rect.h}, {x: rect.x, y: rect.y}],
  ];
  let best = Infinity;
  if (segmentIntersectsRect(a, b, rect)) return 0;
  for (const [p1, p2] of edges){
    best = Math.min(best, segmentToSegmentDistance(a, b, p1, p2));
  }
  return best;
}

function segmentToSegmentDistance(a, b, c, d){
  if (segmentsIntersect(a,b,c,d)) return 0;
  return Math.min(pointToSegmentDistance(a,c,d), pointToSegmentDistance(b,c,d), pointToSegmentDistance(c,a,b), pointToSegmentDistance(d,a,b));
}

function pointToSegmentDistance(p, a, b){
  const vx = b.x - a.x, vy = b.y - a.y;
  const wx = p.x - a.x, wy = p.y - a.y;
  const c1 = vx*wx + vy*wy;
  if (c1 <= 0) return Math.hypot(p.x - a.x, p.y - a.y);
  const c2 = vx*vx + vy*vy;
  if (c2 <= c1) return Math.hypot(p.x - b.x, p.y - b.y);
  const t = c1 / c2;
  const px = a.x + t*vx, py = a.y + t*vy;
  return Math.hypot(p.x - px, p.y - py);
}

function pointToPolyDistance(p, poly){
  let best = Infinity;
  for (let i = 0; i < poly.length; i++){
    const a = poly[i], b = poly[(i+1)%poly.length];
    best = Math.min(best, pointToSegmentDistance(p, a, b));
  }
  return best;
}

// bygger PNG med samlet kontur
async function buildStickerPNG(p) {
  // masken opbygges som i draw
  const tmpMask = p.createGraphics(VID_W, VID_H);
  tmpMask.clear();
  tmpMask.noStroke(); tmpMask.fill(255);

  if (faces.length > 0) {
    drawPolygon(tmpMask, faceExpandedOutlinePoints(faces[0]));
  }

  // Add decos to mask so outline includes them
  for (const d of decos) {
    tmpMask.push();
    tmpMask.translate(d.x, d.y);
    tmpMask.rotate(d.angleToOrigin());
    tmpMask.imageMode(tmpMask.CENTER);
    tmpMask.image(d.img, 0, 0, d.w, d.h);
    tmpMask.pop();
  }

  // lav bbox ud fra face + håndpunkter
  const shapePts = [];
  if (faces.length > 0) shapePts.push(...faceExpandedOutlinePoints(faces[0]));
  if (shapePts.length < 3) return null;

  let minX = p.width, maxX = 0, minY = p.height, maxY = 0;
  for (const pt of shapePts) {
    if (pt.x < minX) minX = pt.x;
    if (pt.x > maxX) maxX = pt.x;
    if (pt.y < minY) minY = pt.y;
    if (pt.y > maxY) maxY = pt.y;
  }
  
  // Also include decos in bbox
  for (const d of decos) {
    const x1 = d.x - d.w/2, x2 = d.x + d.w/2;
    const y1 = d.y - d.h/2, y2 = d.y + d.h/2;
    if (x1 < minX) minX = x1;
    if (x2 > maxX) maxX = x2;
    if (y1 < minY) minY = y1;
    if (y2 > maxY) maxY = y2;
  }
  
  const faceW = maxX - minX;
  const faceH = maxY - minY;
  const size = (Math.max(faceW, faceH) * 2.5) | 0; // increased for higher quality
  const centerX = (minX + maxX) / 2;
  const centerY = (minY + maxY) / 2;

  const square = p.createGraphics(size, size);
  square.pixelDensity(2); // 2x resolution for better quality
  square.clear();

  const offsetX = size / 2 - centerX;
  const offsetY = size / 2 - centerY;

  const videoCopy = g.pg.get();
  videoCopy.mask(tmpMask.get());
  square.image(videoCopy, offsetX, offsetY);

  for (const d of decos) {
   // Brug samme rotationslogik som preview
    let angle = 0;
    const decoX = d.x;
    const decoY = d.y;
    if (decoX > VID_W/2 + 20) {
      angle = Math.atan2(decoY, decoX);
    } else if (decoX < VID_W/2 - 20) {
      angle = Math.atan2(-decoY, decoX);
    } else {
      angle = 0;
    }
    const dx = decoX - centerX + size/2;
    const dy = decoY - centerY + size/2;

    square.push();
    square.translate(dx, dy);
    square.rotate(angle);
    square.imageMode(square.CENTER);
    square.image(d.img, 0, 0, d.w, d.h);
    square.pop();
  }

  // Draw unified outline around face and decos
  const outline = extractUnifiedOutlineFromMask(tmpMask, 128);
  if (outline && outline.length > 1) {
    const localOutline = outline.map(v => ({
      x: v.x - centerX + size/2,
      y: v.y - centerY + size/2
    }));
    drawPolylineLocalFromPts(square, localOutline);
  }

  const dataUrl = square.canvas.toDataURL('image/png');
  return dataUrl;
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
  ml5 = ml5Mod.default || ml5Mod;

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
  try { stopManualFaceDetect?.(); } catch {}
  try { pInstance?.remove?.(); } catch {}
});
</script>

<template>
  <div id="root">
    <div v-if="loading" class="loading-overlay">
      <div class="spinner"></div>
      <div style="margin-top:16px; font-size:18px; color:#444; text-align: center;">
        Loading face mesh… <br>
        <span style="font-size:12px">(Remember to allow camera access)</span>
      </div>
    </div>
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
.loading-overlay {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(255,255,255,0.85);
  z-index: 1000;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
.spinner {
  width: 48px;
  height: 48px;
  border: 6px solid #ccc;
  border-top: 6px solid #333;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}
@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
</style>
