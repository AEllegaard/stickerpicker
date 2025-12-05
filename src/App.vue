<script setup>
import { onMounted, onBeforeUnmount, ref, reactive, computed } from 'vue';
import './assets/style.css';

// import deco-ressourcer
//hats
import BeanieSrc from './assets/svg/Beanie.svg';
import BeretSrc from './assets/svg/Beret.svg';
import BuckethatSrc from './assets/svg/Buckethat.svg';
import CapSrc from './assets/svg/Cap.svg';
import CowboyhatSrc from './assets/svg/Cowboyhat.svg';
import LadyhatSrc from './assets/svg/Ladyhat.svg';
import PartyhatoneSrc from './assets/svg/Partyhatone.svg';
import PartyhattwoSrc from './assets/svg/Partyhattwo.svg';
import PartyhatthreeSrc from './assets/svg/PartyhatThree.svg';
import PropelSrc from './assets/svg/Propel.svg';
import TallhatSrc from './assets/svg/Tallhat.svg';

//Glasses
import CatsunniesSrc from './assets/svg/Catsunnies.svg';
import FlowersunniesSrc from './assets/svg/Flowersunnies.svg';
import HeartsunniesSrc from './assets/svg/Heartsunnies.svg';
import PixelsunniesSrc from './assets/svg/Pixelsunnies.svg';
import SquaresunniesSrc from './assets/svg/Squaresunnies.svg';
import StarSunniesSrc from './assets/svg/Starsunnies.svg';

//Mouths
import BiglipsSrc from './assets/svg/Biglips.svg';
import BluetoungeoutSrc from './assets/svg/Bluetoungeout.svg';
import GreenmouthSrc from './assets/svg/Greenmouth.svg';
import HappylaydSrc from './assets/svg/Happylady.svg';
import NoteethSrc from './assets/svg/Noteeth.svg';
import OpensmileSrc from './assets/svg/Opensmile.svg';
import PinkmouthSrc from './assets/svg/Pinkmouth.svg';
import PinktoungeoutSrc from './assets/svg/Pinktoungeout.svg';
import UnderbiteSrc from './assets/svg/Underbite.svg';


const canvasHost = ref(null);

// UI state
const inMiroRef = ref(false);
const buttonLabel = ref('Save sticker locally');
const isFaceCaptured = ref(false);

const loading = ref(true); // loading state for face mesh

// handles
let P5, ml5, pInstance = null, faceMesh = null;
// let handposeModel = null, tf = null, handDetector = null;
let faces = [];
// let hands = [];

// deco-objekter på lærredet
let decos = [];
let capturedFrame = null; // for frozen photo
let capturedOutline = null; // for frozen outline
// thumbnail positions for palette hit detection
let thumbPositions = new Map();
// pagination state per category

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
  { id: 'glasses', label: 'Glasses' },
  { id: 'mouths', label: 'Mouth' },
  { id: 'hats', label: 'Head wear' },
  { id: 'hands', label: 'Hands' }
];

const assetsByCategory = computed(() => {
  const grouped = {};
  for (const cat of categories) {
    grouped[cat.id] = palette.filter(item => item.category === cat.id);
  }
  return grouped;
});


// palette-ikoner (nede i panelet)
const palette = [
  //hats and bats
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

// hjælpefunktion til at konvertere hex-farve til HSL og returnere CSS filter
function getColorFilterFromHex(hex) {
  // Farve skift udkommenteret
  return 'none';
}

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
  constructor(p, img, x, y, w, h, color = null) {
    this.p = p;
    this.img = img;
    this.x = x; this.y = y;
    this.w = w; this.h = h;
    this.dragging = false;
    this._dx = 0; this._dy = 0;
  // this.color = color; // farve skift udkommenteret
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
  // p.drawingContext.filter = getColorFilterFromHex(this.color); // farve skift udkommenteret
    p.image(this.img, 0, 0, this.w, this.h);
    p.drawingContext.filter = 'none';
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
  p.createCanvas(420, 240); // Increased width for better visibility

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
        maxFaces: 2,
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
    p.background(236, 236, 236);

    // ryd buffers
    g.pg.clear();
    g.maskG.clear();
    g.maskG.noStroke();
    g.maskG.fill(255);

    // spejl video ind i pg (eller brug captured frame hvis face er frozen)
    if (g.video && !isFaceCaptured.value) {
      g.pg.push();
      g.pg.translate(VID_W, 0);
      g.pg.scale(-1, 1);
      g.pg.image(g.video, 0, 0, VID_W, VID_H);
      g.pg.pop();
    } else if (capturedFrame && isFaceCaptured.value) {
      // vis den frosne frame
      g.pg.image(capturedFrame, 0, 0, VID_W, VID_H);
    }

  // No asset panel or palette rendering in canvas

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
      let outline = extractUnifiedOutlineFromMask(g.maskG, 128);
      
      // hvis face er frozen, brug den gemte outline
      if (isFaceCaptured.value && capturedOutline) {
        outline = capturedOutline;
      }
      
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
    // Only handle canvas interactions (dragging, removing decos)
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
    // Responsive panel layout based on canvas width
    const panelPad = 6;
    const panelX = panelPad;
    const panelY = 254;
    const panelW = Math.max(220, p.width - panelPad * 2);
    const panelH = 240;

    p.noStroke();
    p.fill(250);
    p.stroke(220);
    p.strokeWeight(1);
    p.rect(panelX, panelY, panelW, panelH, 8);

    // header and randomize button (positions computed from panel)
    const headerX = panelX + 12;
    const headerY = panelY + 16;
    p.noStroke();
    p.fill(30);
    p.textSize(13);
    p.textStyle(p.BOLD);
    p.textAlign(p.LEFT, p.CENTER);
    p.text('Assets', headerX, headerY);

    const randomBtnW = Math.min(140, Math.max(96, Math.floor(panelW * 0.35)));
    const randomBtnH = 30;
    const randomBtnX = panelX + panelW - randomBtnW - 12;
    const randomBtnY = panelY + 10;
    const isHoveringRandomize = p.mouseX >= randomBtnX && p.mouseX <= randomBtnX + randomBtnW &&
                                p.mouseY >= randomBtnY && p.mouseY <= randomBtnY + randomBtnH;
    p.fill(isHoveringRandomize ? 90 : 110);
    p.rect(randomBtnX, randomBtnY, randomBtnW, randomBtnH, 6);
    p.fill(255);
    p.textSize(12);
    p.textAlign(p.CENTER, p.CENTER);
    p.text('Randomize colors', randomBtnX + randomBtnW/2, randomBtnY + randomBtnH/2);

    // category buttons row
    const catY = panelY + 46;
    const catPad = 12;
    const catBtnW = Math.floor((panelW - catPad * 2 - (categories.length - 1) * 8) / categories.length);
    const catBtnH = 28;
    for (let i = 0; i < categories.length; i++) {
      const btnX = panelX + catPad + i * (catBtnW + 8);
      const isSelected = categories[i].id === selectedCategory;
      p.noStroke();
      p.fill(isSelected ? 100 : 235);
      p.rect(btnX, catY, catBtnW, catBtnH, 6);
      p.fill(isSelected ? 255 : 50);
      p.textSize(11);
      p.textStyle(p.BOLD);
      p.textAlign(p.CENTER, p.CENTER);
      p.text(categories[i].label, btnX + catBtnW/2, catY + catBtnH/2);
    }
    p.textAlign(p.LEFT);

    // content area: compute thumbnail grid responsively
    const contentX = panelX + 12;
    const contentY = catY + catBtnH + 12;
    const contentW = panelW - 24;
    const contentH = panelH - (contentY - panelY) - 12;
    const filteredItems = palette.filter(item => item.category === selectedCategory);
    thumbPositions.clear();

    // determine thumbnail size and columns based on available width
    const maxThumb = 64;
    const minThumb = 44;
    const gap = 12;
    let cols = Math.floor((contentW + gap) / (minThumb + gap));
    cols = Math.max(1, Math.min(cols, 4));
    const thumbW = Math.max(minThumb, Math.min(maxThumb, Math.floor((contentW - (cols - 1) * gap) / cols)));
    const thumbH = thumbW;

    // compute rows that fit in content area
    const rows = Math.max(1, Math.floor((contentH + gap) / (thumbH + gap)));
    const itemsPerPage = rows * cols;
    const totalPages = Math.max(1, Math.ceil(filteredItems.length / itemsPerPage));
    let page = pageIndex[selectedCategory] || 0;
    if (page >= totalPages) page = totalPages - 1;
    pageIndex[selectedCategory] = page;

    const start = page * itemsPerPage;
    const end = Math.min(filteredItems.length, start + itemsPerPage);

    for (let idx = start; idx < end; idx++) {
      const localIdx = idx - start;
      const item = filteredItems[idx];
      const col = localIdx % cols;
      const row = Math.floor(localIdx / cols);
      const x = contentX + col * (thumbW + gap) + thumbW/2;
      const y = contentY + row * (thumbH + gap) + thumbH/2;
      thumbPositions.set(item.key, { x, y, w: thumbW, h: thumbH, item, page });

      const img = p.assets[item.key];
      if (!img) continue;
      p.push();
      p.imageMode(p.CENTER);
      p.noStroke();
      p.fill(245);
      p.rect(x, y, thumbW + 8, thumbH + 8, 8);
      p.image(img, x, y, thumbW, thumbH);
      p.pop();

      if (dist2(p.mouseX, p.mouseY, x, y) < (Math.max(thumbW, thumbH)/2)**2) {
        p.noFill(); p.stroke(0); p.strokeWeight(2);
        p.rect(x - thumbW/2 - 4, y - thumbH/2 - 4, thumbW + 8, thumbH + 8);
      }
    }

    // draw pagination controls
    const ctrlW = 28, ctrlH = 24;
    const ctrlX = panelX + panelW - 12 - ctrlW * 2 - 8;
    const ctrlY = panelY + panelH - 12 - ctrlH;
    // prev
    p.fill(page > 0 ? 120 : 200);
    p.noStroke();
    p.rect(ctrlX, ctrlY, ctrlW, ctrlH, 6);
    p.fill(255);
    p.textAlign(p.CENTER, p.CENTER);
    p.text('<', ctrlX + ctrlW/2, ctrlY + ctrlH/2);
    // page indicator
    p.noStroke();
    p.fill(80);
    p.textSize(11);
    p.text(`${page+1}/${totalPages}`, ctrlX + ctrlW + 6 + 20, ctrlY + ctrlH/2);
    // next
    p.fill(page < totalPages - 1 ? 120 : 200);
    p.rect(ctrlX + ctrlW + 48, ctrlY, ctrlW, ctrlH, 6);
    p.fill(255);
    p.text('>', ctrlX + ctrlW + 48 + ctrlW/2, ctrlY + ctrlH/2);
  }

  // draw pagination controls (if needed)
  function drawPagination(p) {
    const panelPad = 6;
    const panelX = panelPad;
    const panelY = 254;
    const panelW = Math.max(220, p.width - panelPad * 2);
    const contentW = panelW - 24;
    const filteredItems = palette.filter(item => item.category === selectedCategory);
    const gap = 12;
    const minThumb = 44;
    let cols = Math.floor((contentW + gap) / (minThumb + gap));
    cols = Math.max(1, Math.min(cols, 4));
    const rows = Math.ceil(filteredItems.length / cols);
    const itemsPerPage = cols * Math.max(1, Math.floor((200) / (minThumb + gap)));
    const totalPages = Math.max(1, Math.ceil(filteredItems.length / itemsPerPage));
    const page = pageIndex[selectedCategory] || 0;

    // small controls at bottom-right of panel
    const ctrlW = 28, ctrlH = 24;
    const ctrlX = panelX + panelW - 12 - ctrlW * 2 - 8;
    const ctrlY = panelY + panelH - 12 - ctrlH;

    // prev
    p.fill(page > 0 ? 120 : 200);
    p.noStroke();
    p.rect(ctrlX, ctrlY, ctrlW, ctrlH, 6);
    p.fill(255);
    p.textAlign(p.CENTER, p.CENTER);
    p.text('<', ctrlX + ctrlW/2, ctrlY + ctrlH/2);

    // page indicator
    p.noStroke();
    p.fill(80);
    p.textSize(11);
    p.text(`${page+1}/${totalPages}`, ctrlX + ctrlW + 6 + 20, ctrlY + ctrlH/2);

    // next
    p.fill(page < totalPages - 1 ? 120 : 200);
    p.rect(ctrlX + ctrlW + 48, ctrlY, ctrlW, ctrlH, 6);
    p.fill(255);
    p.text('>', ctrlX + ctrlW + 48 + ctrlW/2, ctrlY + ctrlH/2);
  }
  
  function getCategoryButtonAt(mx, my) {
    // compute category button positions dynamically (same logic as drawPalette)
    const panelPad = 6;
    const panelX = panelPad;
    const panelY = 254;
    const panelW = Math.max(220, pInstance.width - panelPad * 2);
    const catY = panelY + 46;
    const catPad = 12;
    const catBtnW = Math.floor((panelW - catPad * 2 - (categories.length - 1) * 8) / categories.length);
    const catBtnH = 28;

    for (let i = 0; i < categories.length; i++) {
      const btnX = panelX + catPad + i * (catBtnW + 8);
      if (mx >= btnX && mx <= btnX + catBtnW && my >= catY && my <= catY + catBtnH) {
        return categories[i].id;
      }
    }
    return null;
  }

  function isClickingRandomizeButton(mx, my) {
    const panelPad = 6;
    const panelX = panelPad;
    const panelY = 254;
    const panelW = Math.max(220, pInstance.width - panelPad * 2);
    const randomBtnW = Math.min(140, Math.max(96, Math.floor(panelW * 0.35)));
    const randomBtnH = 30;
    const randomBtnX = panelX + panelW - randomBtnW - 12;
    const randomBtnY = panelY + 10;
    return mx >= randomBtnX && mx <= randomBtnX + randomBtnW && my >= randomBtnY && my <= randomBtnY + randomBtnH;
  }

  function isClickingPrevPage(mx, my) {
    const panelPad = 6;
    const panelX = panelPad;
    const panelY = 254;
    const panelW = Math.max(220, pInstance.width - panelPad * 2);
    const panelH = 240;
    const ctrlW = 28, ctrlH = 24;
    const ctrlX = panelX + panelW - 12 - ctrlW * 2 - 8;
    const ctrlY = panelY + panelH - 12 - ctrlH;
    return mx >= ctrlX && mx <= ctrlX + ctrlW && my >= ctrlY && my <= ctrlY + ctrlH;
  }

  function isClickingNextPage(mx, my) {
    const panelPad = 6;
    const panelX = panelPad;
    const panelW = Math.max(220, pInstance.width - panelPad * 2);
    const panelH = 240;
    const ctrlW = 28, ctrlH = 24;
    const ctrlX = panelX + panelW - 12 - ctrlW * 2 - 8;
    const ctrlY = panelY + panelH - 12 - ctrlH;
    const nextX = ctrlX + ctrlW + 48;
    return mx >= nextX && mx <= nextX + ctrlW && my >= ctrlY && my <= ctrlY + ctrlH;
  }

  function randomizeDecoColors() {
    // farve skift udkommenteret
    // for (const deco of decos) {
    //   deco.color = COLORS[Math.floor(Math.random() * COLORS.length)];
    // }
  }

  function hitPalette(mx, my) {
    // use computed thumbnail positions for hit testing
    for (const [key, rect] of thumbPositions) {
      const { x, y, w, h, item } = rect;
      if (item.category !== selectedCategory) continue;
      if (mx >= x - w/2 && mx <= x + w/2 && my >= y - h/2 && my <= y + h/2) return item;
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

function addAsset(item) {
  if (!pInstance || !pInstance.assets || !pInstance.assets[item.key]) return;
  const img = pInstance.assets[item.key];
  let x = VID_W/2, y = VID_H/2;
  let size = Math.max(item.w, item.h, 160); // default size at least 160px
  if (faces.length > 0 && faces[0].boundingBox) {
    const fb = faces[0].boundingBox;
    x = fb.cx;
    if (item.category === 'hats') y = fb.cy - fb.h * 0.6;
    else if (item.category === 'glasses') y = fb.cy - fb.h * 0.12;
    else if (item.category === 'mouths') y = fb.cy + fb.h * 0.28;
    size = Math.max(fb.w, fb.h) * (item.category === 'glasses' || item.category === 'mouths' ? 0.7 : 1.1);
    size = Math.max(160, Math.min(size, 260));
  }
  // Always use default color (null disables randomization)
  decos.push(new Deco(pInstance, img, x, y, size, size, null));
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
  // masken opbygges som i draw - BARE ansigt, ikke decos
  const tmpMask = p.createGraphics(VID_W, VID_H);
  tmpMask.clear();
  tmpMask.noStroke(); tmpMask.fill(255);

  if (faces.length > 0) {
    drawPolygon(tmpMask, faceExpandedOutlinePoints(faces[0]));
  }

  // lav bbox ud fra face kun
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
  
  const faceW = maxX - minX;
  const faceH = maxY - minY;
  const padding = Math.max(faceW, faceH) * 0.05; // minimal 5% padding
  const sizeNoScale = Math.max(faceW, faceH) + padding * 2;
  const size = (sizeNoScale * 2) | 0; // scale up 2x for pixelDensity quality
  const centerX = (minX + maxX) / 2;
  const centerY = (minY + maxY) / 2;

  const square = p.createGraphics(size, size);
  square.pixelDensity(2); // 2x resolution for better quality
  square.clear();

  const offsetX = size / 2 - centerX;
  const offsetY = size / 2 - centerY;

  // Draw white background shape (expanded face outline) first - under everything
  const facePts = faceExpandedOutlinePoints(faces[0]);
  const expandScale = 1.15; // forstørrelse af rammen
  const faceCenterX = (minX + maxX) / 2;
  const faceCenterY = (minY + maxY) / 2;
  
  square.fill(255); // hvid fyld
  square.noStroke();
  square.beginShape();
  for (const pt of facePts) {
    const expanded = {
      x: faceCenterX + (pt.x - faceCenterX) * expandScale,
      y: faceCenterY + (pt.y - faceCenterY) * expandScale
    };
    const localX = expanded.x - centerX + size/2;
    const localY = expanded.y - centerY + size/2;
    square.vertex(localX, localY);
  }
  square.endShape(square.CLOSE);

  const videoCopy = g.pg.get();
  videoCopy.mask(tmpMask.get());
  square.image(videoCopy, offsetX, offsetY);

  // Draw decos on top
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
    
    // Apply color filter
    const colorFilter = getColorFilterFromHex(d.color);
    square.drawingContext.filter = colorFilter;
    square.image(d.img, 0, 0, d.w, d.h);
    square.drawingContext.filter = 'none';
    
    square.pop();
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
    
    // Reset after successful paste
    isFaceCaptured.value = false;
    capturedFrame = null;
    capturedOutline = null;
    decos = [];
    // Restart face tracking
    try { faceMesh?.detectStart?.(g.video.elt, (results) => { faces = results || []; }); } catch (e) { console.warn('faceMesh.detectStart fejlede', e); }
  } catch (e) {
    console.warn('Ikke i Miro eller createImage fejlede. Gemmer lokalt.', e);
    const a = document.createElement('a');
    a.href = dataUrl;
    a.download = 'sticker.png';
    a.click();
    
    // Reset after successful save
    isFaceCaptured.value = false;
    capturedFrame = null;
    capturedOutline = null;
    decos = [];
    // Restart face tracking
    try { faceMesh?.detectStart?.(g.video.elt, (results) => { faces = results || []; }); } catch (e2) { console.warn('faceMesh.detectStart fejlede', e2); }
  }
}

function takeFaceSnapshot() {
  if (faces.length === 0) {
    console.warn('Ingen ansigt fundet');
    return;
  }
  // Gem den aktuelle video frame
  capturedFrame = g.pg.get();
  // Gem også den aktuelle outline
  g.maskG.clear();
  g.maskG.noStroke();
  g.maskG.fill(255);
  if (faces.length > 0) {
    const fPts = faceExpandedOutlinePoints(faces[0]);
    drawPolygon(g.maskG, fPts);
  }
  capturedOutline = extractUnifiedOutlineFromMask(g.maskG, 128);
  // Gem det aktuelle ansigt så vi kan bruge det i buildStickerPNG
  const capturedFace = faces[0];
  // Stop live face tracking by stopping FaceMesh detection
  try { faceMesh?.detectStop?.(); } catch (e) { console.warn('faceMesh.detectStop fejlede', e); }
  faces = [];
  if (capturedFace) {
    faces = [capturedFace]; // Behold kun det captured ansigt
  }
  isFaceCaptured.value = true;
}

function resetSticker() {
  isFaceCaptured.value = false;
  capturedFrame = null;
  capturedOutline = null;
  decos = [];
  // Restart face tracking
  try { faceMesh?.detectStart?.(g.video.elt, (results) => { faces = results || []; }); } catch (e) { console.warn('faceMesh.detectStart fejlede', e); }
}

function undoLastAsset() {
  if (decos.length > 0) {
    decos.pop();
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
    <div style="position: fixed; top: 0; right: 0; z-index: 100; padding: 14px -5px;">
      <button class="btn-action" @click="resetSticker" style="padding: 0; border-radius: 50%; width:44px; height:44px; display:flex; align-items:center; justify-content:center; background: none; box-shadow: none; border: none;">
        <img src="./assets/redo.svg" alt="Reset" width="20" height="20" style="display:block;" />
      </button>
    </div>
    <div v-if="loading" class="loading-overlay">
      <div class="spinner"></div>
      <div style="margin-top:16px; font-size:18px; color:#444; text-align: center;">
        Loading face mesh… <br>
        <span style="font-size:12px">(Remember to allow camera access)</span>
      </div>
    </div>
    <div class="canvas" style="margin-bottom: 2px;">
      <div ref="canvasHost" style="min-height: 240px;"></div>
      <div class="button-group" style="display:flex; gap: 12px; align-items: flex-end; justify-content: flex-end;">
        <div style="display: flex; flex-direction: column; gap: 12px;">
          <button class="btn-action effectBtn" style="padding:0; background: none;" @click="decreaseAssetSize">
            <img src="./assets/minus.svg" alt="Decrease" width="22" height="22" style="display:inline; vertical-align:middle;" />
          </button>
          <button class="btn-action effectBtn" style="padding:0; background: none;" @click="increaseAssetSize">
            <img src="./assets/plus.svg" alt="Increase" width="22" height="22" style="display:inline; vertical-align:middle;" />
          </button>
        </div>
        <div class="gruppe" style="display: flex; flex-direction: row; gap: 8px;">
          <button class="btn-action" style="padding:0; background: none;" @click="undoLastAsset">
            <img src="./assets/undo.svg" alt="Undo" width="22" height="22" style="display:inline; vertical-align:middle;" />
          </button>
          <button class="btn-action" @click="takeFaceSnapshot" v-if="!isFaceCaptured">
            Take photo
          </button>
          <button class="btn-action" @click="pasteSticker" v-if="isFaceCaptured">
            Save to board
          </button>
        </div>
      </div>
    </div>

    <div class="asset-panel">
      <div class="asset-scroll">
        <div v-for="cat in categories" :key="cat.id" class="category-section">
          <h3 class="category-title">{{ cat.label }}</h3>
          <div class="asset-grid">
            <button v-for="item in assetsByCategory[cat.id]" :key="item.key" class="asset-thumb" @click="addAsset(item)">
              <img :src="item.src" :alt="item.key" v-if="item.src" />
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
#root {
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
  background: #ececec;
}

.canvas { 
  background: #ececec; 
  border: none;
  padding: 8px;
  margin-bottom: 8px;
  flex-shrink: 0;
  position: relative;
}

.button-group {
  position: absolute;
  right: 10px;
  top: 70%;
  transform: translateY(-50%);
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.btn-action {
  background: #000;
  color: #fff;
  border: none;
  border-radius: 24px;
  padding: 8px 10px;
  font-size: 10px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
  white-space: nowrap;
}

.btn-action resets{
  background-color: none;
}

.btn-action:hover {
  background: #333;
  transform: scale(1.05);
}

.asset-panel {
  background: #fcfcfc;
  border: #333 1px solid;
  border-radius: 24px 24px 0 0 ;
  padding: 16px;
  flex: 1;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  display: flex;
  flex-direction: column;
}

.asset-scroll {
  flex: 1;
  overflow-y: auto;
  overflow-x: hidden;
  padding-right: 2px;
}

.asset-scroll::-webkit-scrollbar {
  width: 4px;
}

.asset-scroll::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 10px;
}

.asset-scroll::-webkit-scrollbar-thumb {
  background: #ccc;
  border-radius: 10px;
}

.asset-scroll::-webkit-scrollbar-thumb:hover {
  background: #999;
}

.category-section {
  margin-bottom: 32px;
}

.category-title {
  font-size: 18px;
  font-weight: 700;
  margin: 0 0 16px 0;
  color: #333;
}

.asset-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 8px;
}

.asset-thumb {
  aspect-ratio: 1;
  min-width: 50x;
  min-height: 50px;
  background-color: #fcfcfc;
  border: none;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s ease;
  padding: 8px;
}

.asset-thumb:hover {
  transform: scale(1.25);
}

.asset-thumb img {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

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

.button-primary{
  background-color: black;
  border-radius: 50px;
}

.effectBtn{
  background-color: black;
  border-radius: 50px;
}
</style>
