<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<title>Cat Dash</title>
<style>
  html, body {
    margin: 0;
    padding: 0;
    background: #333;
    color: #fff;
    font-family: system-ui, sans-serif;
    height: 100%;
    overflow: hidden;
  }
  canvas {
    display: block;
    margin: 0 auto;
    background: #6dbb4a;
    touch-action: none;
  }
  .controls {
    position: fixed;
    inset: 0;
    pointer-events: none;
    z-index: 10;
  }
  .laneBtn {
    position: absolute;
    width: 88px; height: 88px;
    border: none; border-radius: 20px;
    background: rgba(0,0,0,0.35); color: #fff;
    font-size: 28px;
    box-shadow: 0 6px 18px rgba(0,0,0,0.35);
    -webkit-tap-highlight-color: transparent;
    touch-action: manipulation;
    pointer-events: auto;
    transform: translate(-50%, -50%);
  }
  .laneBtn:active { transform: translate(-50%, -50%) scale(0.96); }

  @media (min-width: 900px) {
    .controls { display: none; }
  }
</style>
</head>
<body>
<canvas id="gameCanvas"></canvas>

<div id="controls" class="controls">
  <button id="btnLane1" class="laneBtn" aria-label="Move left">◀</button>
  <button id="btnLane3" class="laneBtn" aria-label="Move right">▶</button>
</div>

<script>
/* ===== Setup & HD Canvas ===== */
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d', { alpha: false, desynchronized: true });

let W = window.innerWidth;
let H = window.innerHeight;

function setHDCanvas(){
  const DPR = Math.max(1, Math.min(3, window.devicePixelRatio || 1));
  canvas.style.width  = W + 'px';
  canvas.style.height = H + 'px';
  canvas.width  = Math.floor(W * DPR);
  canvas.height = Math.floor(H * DPR);
  ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
  ctx.imageSmoothingEnabled = true;
  ctx.imageSmoothingQuality = 'high';
}
setHDCanvas();

function lanesX(){ return [W/4, W/2, (3*W)/4]; }

/* Shared vertical offset for both cat and buttons */
const CAT_AND_BUTTON_OFFSET = 50;

function positionButtons(){
  const btn1 = document.getElementById('btnLane1');
  const btn3 = document.getElementById('btnLane3');
  const lx = lanesX();

  const baseY = H - Math.min(120, H * 0.12);
  const y = Math.min(H - 10, baseY + CAT_AND_BUTTON_OFFSET);

  btn1.style.left = lx[0] + 'px';
  btn1.style.top  = y + 'px';
  btn3.style.left = lx[2] + 'px';
  btn3.style.top  = y + 'px';
}
positionButtons();

window.addEventListener('resize', () => {
  W = window.innerWidth;
  H = window.innerHeight;
  setHDCanvas();
  positionButtons();
});

/* ===== State ===== */
let gameRunning = false;
let currentLane = 1;
let enemies = [];
let pickups = [];
let particles = [];
let announcements = []; // {text, timer}
let score = 0;
let fuel = 100;
let meters = 0;

let spawnTimer = 0;
let last = undefined;
let graceTimer = 0;

/* Speed (unchanged) */
let roadSpeed = 226;
let maxSpeed  = 704;

const baseAccel = 0.6;
let slipTimer = 0, slipOffset = 0;

/* ===== CHEAT: hold both buttons ≥50ms to arm 2 tree ignores ===== */
let btn1Down = false, btn3Down = false;
let cheatArmTimerMs = 0;
let cheatCharges = 0;
let cheatRearmLock = false; // must fully release both buttons before re-arming
const CHEAT_HOLD_TIME_MS = 50;
let cheatToastTimer = 0;
let cheatToastText = '';

function updateCheat(dt){
  const bothDown = btn1Down && btn3Down;
  if (bothDown && !cheatRearmLock && cheatCharges === 0){
    cheatArmTimerMs += dt * 1000;
    if (cheatArmTimerMs >= CHEAT_HOLD_TIME_MS){
      cheatCharges = 2;
      cheatRearmLock = true;
      addAnnounce('Shield armed x2', 1.1);
    }
  } else {
    cheatArmTimerMs = 0;
  }
  if (!btn1Down && !btn3Down && cheatCharges === 0){
    cheatRearmLock = false;
  }
}

/* ===== Skins (Alt Characters) ===== */
const SKINS = [
  { name: 'Cat',     base:'#d2691e', patch:'#a0522d', tail:'#d2691e' , unlock:0 },
  { name: 'Fox',     base:'#e57300', patch:'#bf5f00', tail:'#ffb74d' , unlock:50 },
  { name: 'Rabbit',  base:'#cfd8dc', patch:'#90a4ae', tail:'#cfd8dc' , unlock:100 },
  { name: 'Squirrel',base:'#8d6e63', patch:'#6d4c41', tail:'#a1887f' , unlock:200 },
];
let currentSkinIndex = 0;

function isSkinUnlocked(i){ return scoreBest() >= SKINS[i].unlock; }
const BEST_KEY = 'cat_best_score';
function scoreBest(){ try{ return +localStorage.getItem(BEST_KEY) || 0; }catch{ return 0; } }
function saveBest(sc){ try{ const b = scoreBest(); if (sc > b) localStorage.setItem(BEST_KEY, sc|0); }catch{} }

function cycleSkin(){
  // only when not running
  if (gameRunning) return;
  let tries = 0;
  do {
    currentSkinIndex = (currentSkinIndex + 1) % SKINS.length;
    tries++;
    if (tries > SKINS.length) break;
  } while (!isSkinUnlocked(currentSkinIndex));
}

function drawSkinCharacter(colors, x, y, w, h){
  // same geometry as before, but swap colors
  const base = colors.base, patch = colors.patch;
  withShadow('rgba(0,0,0,0.35)', 12, 5, ()=>{
    ctx.fillStyle = base;
    ctx.beginPath(); ctx.ellipse(x, y, w/2, h/2, 0, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = patch;
    ctx.beginPath(); ctx.ellipse(x, y, w/2.5, h/2.5, 0, 0, Math.PI*2); ctx.fill();
    const headR = h*0.25, hx=x, hy=y - h*0.55;
    ctx.fillStyle = base; ctx.beginPath(); ctx.arc(hx,hy,headR,0,Math.PI*2); ctx.fill();
    ctx.fillStyle = patch; ctx.beginPath(); ctx.arc(hx,hy,headR*0.75,0,Math.PI*2); ctx.fill();
    ctx.fillStyle = base;
    ctx.beginPath(); ctx.moveTo(hx-headR*0.8,hy-headR*0.2); ctx.lineTo(hx-headR*0.3,hy-headR*1.1); ctx.lineTo(hx-headR*0.05,hy-headR*0.2); ctx.closePath(); ctx.fill();
    ctx.beginPath(); ctx.moveTo(hx+headR*0.8,hy-headR*0.2); ctx.lineTo(hx+headR*0.3,hy-headR*1.1); ctx.lineTo(hx+headR*0.05,hy-headR*0.2); ctx.closePath(); ctx.fill();
    ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(hx-headR*0.4, hy, headR*0.15, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(hx+headR*0.4, hy, headR*0.15, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = colors.tail; ctx.beginPath(); ctx.ellipse(x + w/2.2, y, w/6, h/3, 0, 0, Math.PI*2); ctx.fill();
  });
  strokeAround('rgba(0,0,0,0.4)', 1, ()=>{
    ctx.beginPath(); ctx.ellipse(x, y, w/2, h/2, 0, 0, Math.PI*2);
  });
}

/* ===== Leaderboard (unchanged core) ===== */
const BOARD_KEY = 'cat_leaderboard';
function loadBoard(){ try{ return JSON.parse(localStorage.getItem(BOARD_KEY)||'[]'); } catch { return []; } }
function saveBoard(b){ localStorage.setItem(BOARD_KEY, JSON.stringify(b)); }
function addScore(name, sc){
  const board = loadBoard();
  board.push({ name: (name||'CAT').slice(0,10), score: sc|0, date: new Date().toISOString().slice(0,10) });
  board.sort((a,b)=> b.score - a.score);
  saveBoard(board.slice(0,10));
}
function maybeRecordScore(sc){
  const board = loadBoard();
  const qualifies = (board.length < 10) || (sc > board[board.length-1].score);
  if (!qualifies || sc <= 0) return;
  let name = prompt('New high score. Enter name or initials', 'CAT');
  if (!name) name = 'CAT';
  addScore(name.trim().slice(0,10), sc);
}
function drawBoard(x, y){
  const board = loadBoard();
  ctx.fillStyle = '#fff';
  ctx.font = '18px system-ui, sans-serif';
  ctx.fillText('Leaderboard Top 10', x, y);
  ctx.font = '14px ui-monospace, SFMono-Regular, Menlo, monospace';
  if (board.length === 0){
    ctx.fillText('No scores yet. Be the first', x, y+22);
    return;
  }
  for (let i=0; i<board.length && i<10; i++){
    const e = board[i];
    const line = `${String(i+1).padStart(2,' ')}. ${e.name.padEnd(10,' ')}  ${String(e.score).padStart(4,' ')}  ${e.date}`;
    ctx.fillText(line, x, y + 22 + i*18);
  }
}

/* ===== Visual helpers ===== */
function withShadow(color = 'rgba(0,0,0,0.35)', blur = 8, offsetY = 3, drawFn){
  ctx.save();
  ctx.shadowColor = color;
  ctx.shadowBlur = blur;
  ctx.shadowOffsetX = 0;
  ctx.shadowOffsetY = offsetY;
  drawFn();
  ctx.restore();
}
function strokeAround(strokeStyle = 'rgba(0,0,0,0.35)', lineWidth = 2, drawPathFn){
  ctx.save();
  ctx.lineWidth = lineWidth;
  ctx.strokeStyle = strokeStyle;
  drawPathFn();
  ctx.stroke();
  ctx.restore();
}

/* ===== Parallax background (distant hills) ===== */
const hills = [];
function initHills(){
  hills.length = 0;
  const rows = 3;
  for (let r=0; r<rows; r++){
    const baseY = -H * (r*0.6);
    const amp = 20 + r*8;
    const color = `rgba(20,60,20,${0.08 + r*0.05})`;
    hills.push({ y: baseY, amp, color, speedMul: 0.25 + r*0.08, phase: Math.random()*Math.PI*2 });
  }
}
initHills();

function drawHills(dt){
  hills.forEach(h=>{
    h.y += roadSpeed * dt * h.speedMul;
    if (h.y > H + 60) h.y -= H * 2;

    ctx.fillStyle = h.color;
    ctx.beginPath();
    const cycles = 6;
    for (let i=0;i<=cycles;i++){
      const x = (i/cycles) * W;
      const y = h.y + Math.sin(h.phase + i*0.8) * h.amp;
      if (i===0) ctx.moveTo(x, y);
      else ctx.lineTo(x, y);
    }
    ctx.lineTo(W, H); ctx.lineTo(0, H); ctx.closePath();
    ctx.fill();
  });
}

/* ===== Background & NEW tree style ===== */
function drawBackgroundBase(){
  const g = ctx.createLinearGradient(0, 0, 0, H);
  g.addColorStop(0, '#64b24a'); g.addColorStop(1, '#4d9c3b');
  ctx.fillStyle = g; ctx.fillRect(0,0,W,H);

  ctx.fillStyle = 'rgba(40,90,40,0.10)';
  for (let y=0; y<H; y+=40){
    for (let x=((y/40)%2===0?0:20); x<W; x+=40){ ctx.fillRect(x,y,10,10); }
  }

  const trailW = W/6; const lx = lanesX();
  for (let i=0;i<3;i++){
    const rg = ctx.createLinearGradient(0, 0, 0, H);
    rg.addColorStop(0, '#8b684f'); rg.addColorStop(1, '#6f523f');
    ctx.fillStyle = rg; ctx.fillRect(lx[i]-trailW/2, 0, trailW, H);
    ctx.fillStyle = 'rgba(0,0,0,0.18)'; ctx.fillRect(lx[i]-1, 0, 2, H);
  }
}

/* New "soft pine" tree: layered rounded triangles + subtle rim */
function drawTree(x,y,w,h){
  withShadow('rgba(0,0,0,0.35)', 10, 4, ()=>{
    // trunk
    ctx.fillStyle = '#6d3f17';
    const trunkW = w*0.22, trunkH = h*0.30;
    ctx.fillRect(x - trunkW/2, y + h*0.20, trunkW, trunkH);

    // layered pine boughs
    const layers = 3;
    for (let i=0;i<layers;i++){
      const topY = y - h*0.45 + i*h*0.22;
      const halfW = w*(0.65 - i*0.16);
      const color = i===0 ? '#2e7d32' : (i===1 ? '#297a2f' : '#246b29');
      ctx.fillStyle = color;
      ctx.beginPath();
      ctx.moveTo(x, topY);
      ctx.quadraticCurveTo(x - halfW, topY + h*0.22, x - halfW*0.2, topY + h*0.28);
      ctx.lineTo(x + halfW*0.2, topY + h*0.28);
      ctx.quadraticCurveTo(x + halfW, topY + h*0.22, x, topY);
      ctx.closePath(); ctx.fill();
    }
  });
  // thin rim
  strokeAround('rgba(0,0,0,0.28)', 1.2, ()=>{
    ctx.beginPath();
    ctx.moveTo(x, y - h*0.45);
    ctx.lineTo(x - w*0.55, y - h*0.23);
    ctx.moveTo(x, y - h*0.45);
    ctx.lineTo(x + w*0.55, y - h*0.23);
  });
}

/* Other hazards */
function drawRock(x,y,w,h){
  withShadow('rgba(0,0,0,0.35)', 8, 3, ()=>{
    ctx.fillStyle = '#7f8c8d';
    ctx.beginPath();
    ctx.moveTo(x - w*0.45, y + h*0.15);
    ctx.lineTo(x - w*0.25, y - h*0.25);
    ctx.lineTo(x + w*0.15, y - h*0.28);
    ctx.lineTo(x + w*0.45, y + h*0.05);
    ctx.lineTo(x + w*0.25, y + h*0.30);
    ctx.lineTo(x - w*0.35, y + h*0.28);
    ctx.closePath(); ctx.fill();
  });
}
function drawBush(x,y,w,h){
  withShadow('rgba(0,0,0,0.3)', 8, 3, ()=>{
    const c = ['#2e7d32','#2a6f2d','#266629'];
    for (let i=0;i<3;i++){
      ctx.fillStyle = c[i];
      ctx.beginPath(); ctx.arc(x - w*0.3 + i*w*0.3, y, h*0.35 + i*1.5, 0, Math.PI*2); ctx.fill();
    }
  });
}
function drawLog(x,y,w,h){
  withShadow('rgba(0,0,0,0.35)', 10, 4, ()=>{
    ctx.fillStyle = '#8d6e63';
    ctx.beginPath(); ctx.ellipse(x, y, w*0.5, h*0.35, 0, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = '#5d4037';
    ctx.beginPath(); ctx.arc(x - w*0.45, y, h*0.28, 0, Math.PI*2); ctx.fill();
  });
}

/* ===== Land pickups: mouse, bird, lizard, golden chicken ===== */
function drawAdditiveGlow(x, y, radius, centerAlpha=0.9){
  ctx.save();
  ctx.globalCompositeOperation = 'lighter';
  const g = ctx.createRadialGradient(x, y, 0, x, y, radius);
  g.addColorStop(0, `rgba(255,215,0,${centerAlpha})`);
  g.addColorStop(1, 'rgba(255,215,0,0)');
  ctx.fillStyle = g;
  ctx.beginPath(); ctx.arc(x, y, radius, 0, Math.PI*2); ctx.fill();
  ctx.restore();
}

/* Stylized critters */
function drawMouse(x, y, scale, golden){
  const t = performance.now()*0.006, wiggle = Math.sin(t + x*0.01)*1.2*scale;
  const body = golden ? '#ffd54f' : '#c7a17a';
  const ear   = golden ? '#ffe082' : '#d7b894';
  withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{
    ctx.fillStyle = body;
    ctx.beginPath(); ctx.ellipse(x, y+wiggle, 12*scale, 7*scale, 0, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(x+9*scale, y-1*scale+wiggle, 6*scale, 5*scale, 0, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = ear;
    ctx.beginPath(); ctx.arc(x+12*scale, y-5*scale+wiggle, 2.8*scale, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(x+7.5*scale, y-6*scale+wiggle, 2.2*scale, 0, Math.PI*2); ctx.fill();
    ctx.strokeStyle = golden ? '#ffe082' : '#b78963';
    ctx.lineWidth = 1.4*scale;
    ctx.beginPath(); ctx.moveTo(x-12*scale, y+2*scale+wiggle);
    ctx.quadraticCurveTo(x-18*scale, y+6*scale+wiggle, x-22*scale, y+3*scale+wiggle);
    ctx.stroke();
    ctx.fillStyle = '#000';
    ctx.beginPath(); ctx.arc(x+11*scale, y-2*scale+wiggle, 1.4*scale, 0, Math.PI*2); ctx.fill();
  });
  if (golden) drawAdditiveGlow(x, y, 22*scale, 0.85);
}

function drawBird(x, y, scale, golden){
  const t = performance.now()*0.004, bob = Math.sin(t + x*0.02)*1.2*scale;
  const body = golden ? '#ffe066' : '#66a9ff';
  withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{
    ctx.fillStyle = body;
    ctx.beginPath(); ctx.ellipse(x, y+bob, 10*scale, 7*scale, 0, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(x+7*scale, y-3*scale+bob, 4*scale, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = golden ? '#ffd54f' : '#4f94f5';
    ctx.beginPath(); ctx.ellipse(x-3*scale, y+1*scale+bob, 6*scale, 4*scale, -0.7, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = golden ? '#ffca28' : '#ffb300';
    ctx.beginPath(); ctx.moveTo(x+11*scale, y-3*scale+bob);
    ctx.lineTo(x+15*scale, y-1*scale+bob);
    ctx.lineTo(x+11*scale, y-1*scale+bob);
    ctx.closePath(); ctx.fill();
    ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(x+6*scale, y-4*scale+bob, 1.2*scale, 0, Math.PI*2); ctx.fill();
    ctx.strokeStyle = golden ? '#ffb300' : '#cc7a00';
    ctx.lineWidth = 1.2*scale;
    ctx.beginPath();
    ctx.moveTo(x-2*scale, y+7*scale+bob); ctx.lineTo(x-2*scale, y+9.5*scale+bob);
    ctx.moveTo(x+1*scale, y+7*scale+bob); ctx.lineTo(x+1*scale, y+9.5*scale+bob);
    ctx.stroke();
  });
  if (golden) drawAdditiveGlow(x, y, 20*scale, 0.85);
}

function drawLizard(x, y, scale, golden){
  const t = performance.now()*0.006, sway = Math.sin(t + x*0.03)*1.5*scale;
  const body = golden ? '#ffd54f' : '#66cc66';
  const spot = golden ? '#ffe082' : '#4caf50';
  withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{
    ctx.fillStyle = body;
    ctx.beginPath(); ctx.ellipse(x, y+sway, 12*scale, 5.5*scale, 0, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(x+9*scale, y-0.5*scale+sway, 5*scale, 4*scale, 0, 0, Math.PI*2); ctx.fill();
    ctx.beginPath();
    ctx.moveTo(x-12*scale, y+sway);
    ctx.quadraticCurveTo(x-20*scale, y+4*scale+sway, x-24*scale, y+1*scale+sway);
    ctx.quadraticCurveTo(x-19*scale, y-2*scale+sway, x-12*scale, y+sway);
    ctx.fill();
    ctx.fillStyle = spot;
    for (let i=-1;i<=1;i+=2){
      ctx.fillRect(x-2*scale, y+5*scale+sway, 3*scale*i, 2*scale);
      ctx.fillRect(x+6*scale, y+4*scale+sway, 3*scale*i, 2*scale);
    }
    ctx.fillStyle = '#000';
    ctx.beginPath(); ctx.arc(x+11*scale, y-1*scale+sway, 1.2*scale, 0, Math.PI*2); ctx.fill();
  });
  if (golden) drawAdditiveGlow(x, y, 22*scale, 0.85);
}

/* Golden Chicken (always golden, +1 shield) */
function drawChicken(x, y, scale){
  const bob = Math.sin(performance.now()*0.004 + x*0.01) * 1.0 * scale;
  withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{
    ctx.fillStyle = '#ffe082';
    ctx.beginPath(); ctx.ellipse(x, y+bob, 12*scale, 9*scale, 0, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(x+9*scale, y-5*scale+bob, 5*scale, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = '#ffb300';
    ctx.beginPath(); ctx.moveTo(x+14*scale, y-5*scale+bob);
    ctx.lineTo(x+18*scale, y-4*scale+bob);
    ctx.lineTo(x+14*scale, y-2.5*scale+bob);
    ctx.closePath(); ctx.fill();
    ctx.fillStyle = '#e53935';
    ctx.beginPath(); ctx.arc(x+8*scale, y-9*scale+bob, 2.1*scale, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(x+10.8*scale, y-9.4*scale+bob, 1.8*scale, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = '#ffd54f';
    ctx.beginPath(); ctx.ellipse(x-3*scale, y-1*scale+bob, 7*scale, 5*scale, -0.7, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(x+8*scale, y-6*scale+bob, 1.4*scale, 0, Math.PI*2); ctx.fill();
    ctx.strokeStyle = '#ffb300'; ctx.lineWidth = 1.3*scale;
    ctx.beginPath();
    ctx.moveTo(x-2*scale, y+9*scale+bob); ctx.lineTo(x-2*scale, y+12*scale+bob);
    ctx.moveTo(x+1*scale, y+9*scale+bob); ctx.lineTo(x+1*scale, y+12*scale+bob);
    ctx.stroke();
  });
  drawAdditiveGlow(x, y, 24*scale, 0.9);
}

function drawPickup(p){
  if (p.type === 'mouse')      drawMouse(p.x, p.y, p.scale, p.golden);
  else if (p.type === 'bird')  drawBird(p.x, p.y, p.scale, p.golden);
  else if (p.type === 'lizard')drawLizard(p.x, p.y, p.scale, p.golden);
  else                         drawChicken(p.x, p.y, p.scale);
}

/* ===== Spawning and placement rules ===== */
const SPAWN_BUFFER_Y = 70;
function laneIsFree(x, y){
  return !enemies.some(e => e.x === x && Math.abs(e.y - y) < SPAWN_BUFFER_Y)
      && !pickups.some(p => p.x === x && Math.abs(p.y - y) < SPAWN_BUFFER_Y);
}
function pickFreeLane(spawnY){
  const lx = lanesX();
  const candidates = [0,1,2].filter(i => laneIsFree(lx[i], spawnY));
  if (!candidates.length) return null;
  return candidates[Math.floor(Math.random()*candidates.length)];
}
let lastPickupLane = null;

function spawnEnemy(){
  const lane = pickFreeLane(-60);
  if (lane == null) return;
  const lx = lanesX();
  // Mix in new hazards
  const r = Math.random();
  let type = 'tree';
  if (r < 0.25) type = 'rock';
  else if (r < 0.45) type = 'bush';
  else if (r < 0.60) type = 'log';
  // tree remains most frequent
  const base = { x: lx[lane], y: -50, w: 40, h: 70, type:'tree' };
  let e;
  if (type==='rock')   e = {type, x: lx[lane], y: -36, w: 44, h: 28};
  else if (type==='bush') e = {type, x: lx[lane], y: -42, w: 60, h: 30};
  else if (type==='log')  e = {type, x: lx[lane], y: -40, w: 66, h: 26};
  else e = base;
  enemies.push(e);
}

function laneHasAnyEnemy(laneIndex){
  const lx = lanesX(); const x = lx[laneIndex];
  return enemies.some(e => e.x === x);
}

// Coin trail (3 in a line)
function maybeSpawnTrail(){
  const chance = 0.22;
  if (Math.random() > chance) return false;
  const spawnY0 = -40;
  const lane = pickFreeLane(spawnY0);
  if (lane == null) return false;
  const lx = lanesX();
  const gap = 120;
  const typePick = ()=> {
    const r = Math.random();
    if (r < 0.40) return 'mouse';
    if (r < 0.80) return 'bird';
    if (r < 0.95) return 'lizard';
    return 'chicken';
  };
  for (let i=0;i<3;i++){
    const y = spawnY0 - i*gap;
    if (!laneIsFree(lx[lane], y)) return false;
  }
  for (let i=0;i<3;i++){
    const y = spawnY0 - i*gap;
    const type = typePick();
    let golden=false, scale=1;
    if (type==='chicken'){ golden=true; scale=1.35; }
    else if (type==='mouse'){ golden=Math.random()<0.25; scale= golden?1.2:1.0; }
    else if (type==='bird'){ golden=Math.random()<0.25; scale= golden?1.25:1.05; }
    else { golden=Math.random()<0.25; scale= golden?1.25:1.1; }
    const baseW = type==='bird' ? 30 : type==='mouse' ? 34 : type==='lizard' ? 36 : 38;
    const baseH = type==='bird' ? 18 : type==='mouse' ? 18 : type==='lizard' ? 16 : 22;
    pickups.push({type, x: lx[lane], y, w: baseW*scale, h: baseH*scale, scale, golden});
  }
  return true;
}

function spawnPickup(){
  if (maybeSpawnTrail()) return;

  const spawnY = -40;
  const sameYEnemy  = enemies.some(e => Math.abs(e.y - spawnY) < SPAWN_BUFFER_Y);
  const sameYPickup = pickups.some(p => Math.abs(p.y - spawnY) < SPAWN_BUFFER_Y);
  if (sameYEnemy || sameYPickup) return;

  const lx = lanesX();
  let candidateLanes = [0,1,2].filter(i => laneIsFree(lx[i], spawnY) && !laneHasAnyEnemy(i));
  if (!candidateLanes.length) return;

  let lane;
  const withoutLast = candidateLanes.filter(l => l !== lastPickupLane);
  lane = (withoutLast.length ? withoutLast : candidateLanes)[Math.floor(Math.random()* (withoutLast.length ? withoutLast.length : candidateLanes.length))];
  lastPickupLane = lane;

  const r = Math.random();
  let type, golden=false, scale=1;
  if (r < 0.40){ type='mouse';   golden = Math.random()<0.25; scale = golden?1.2:1.0; }
  else if (r < 0.80){ type='bird';   golden = Math.random()<0.25; scale = golden?1.25:1.05; }
  else if (r < 0.95){ type='lizard'; golden = Math.random()<0.25; scale = golden?1.25:1.1; }
  else { type='chicken'; golden=true; scale=1.35; }

  const baseW = type==='bird' ? 30 : type==='mouse' ? 34 : type==='lizard' ? 36 : 38;
  const baseH = type==='bird' ? 18 : type==='mouse' ? 18 : type==='lizard' ? 16 : 22;
  const w = baseW * scale;
  const h = baseH * scale;

  pickups.push({type, x: lx[lane], y: spawnY, w, h, scale, golden});
}

/* ===== Character ===== */
const CAT_W = 20, CAT_H = 30;
function drawCharacter(x, y, w, h){
  drawSkinCharacter(SKINS[currentSkinIndex], x, y, w, h);
}

/* ===== Particles & Announcements ===== */
function spawnSparkles(x, y, baseColor){
  const n = 12;
  for (let i=0;i<n;i++){
    const ang = (Math.PI*2) * (i/n) + Math.random()*0.4;
    const spd = 60 + Math.random()*110;
    particles.push({
      x, y,
      vx: Math.cos(ang)*spd,
      vy: Math.sin(ang)*spd - 40,
      life: 0.6 + Math.random()*0.4,
      age: 0,
      color: baseColor
    });
  }
}
function updateParticles(dt){
  for (let i=particles.length-1; i>=0; i--){
    const p = particles[i];
    p.age += dt;
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.vy += 80 * dt;
    if (p.age >= p.life) particles.splice(i,1);
  }
}
function drawParticles(){
  particles.forEach(p=>{
    const a = Math.max(0, 1 - p.age / p.life);
    ctx.save();
    ctx.globalAlpha = a;
    ctx.fillStyle = p.color;
    ctx.beginPath(); ctx.arc(p.x, p.y, 2 + 1.2*a, 0, Math.PI*2); ctx.fill();
    ctx.restore();
  });
}

function addAnnounce(text, time=1.0){
  announcements.push({text, timer: time});
}
function drawAnnouncements(){
  for (let i=announcements.length-1; i>=0; i--){
    const a = announcements[i];
    const life = a.timer;
    const alpha = Math.min(1, life/0.3); // fade in
    ctx.save();
    ctx.globalAlpha = alpha;
    ctx.font = '18px system-ui, sans-serif';
    const tw = ctx.measureText(a.text).width;
    const x = (W - tw)/2;
    const y = 44 + (announcements.length-1-i)*28;
    ctx.fillStyle = 'rgba(0,0,0,0.35)';
    ctx.fillRect(x - 12, y - 18, tw + 24, 28);
    ctx.fillStyle = '#ffd54f';
    ctx.fillText(a.text, x, y);
    ctx.restore();
  }
}

function updateAnnouncements(dt){
  for (let i=announcements.length-1; i>=0; i--){
    announcements[i].timer -= dt;
    if (announcements[i].timer <= 0) announcements.splice(i,1);
  }
}

/* ===== Combo tracking for announcer ===== */
let pickupStreak = 0;
let nextScoreMilestone = 50;

/* ===== Update & Draw ===== */
const PLAYER_Y = () => Math.min(H - 190, H * 0.64 + CAT_AND_BUTTON_OFFSET);

function update(dt){
  const accel = baseAccel * (1 - Math.min(1, roadSpeed / maxSpeed));
  roadSpeed = Math.min(maxSpeed, (roadSpeed + accel) * Math.pow(1.00000002, meters));
  if (slipTimer > 0){
    slipTimer = Math.max(0, slipTimer - dt);
    slipOffset = Math.sin(performance.now()/40) * 4;
  } else slipOffset = 0;

  spawnTimer += dt;
  if (spawnTimer > 0.6){
    spawnTimer = 0;
    if (Math.random() < 0.70) spawnEnemy(); else spawnPickup();
    if (Math.random() < 0.35) spawnPickup();
  }

  enemies.forEach(e => e.y += roadSpeed*dt);
  pickups.forEach(p => p.y += roadSpeed*dt);
  enemies = enemies.filter(e => e.y < H + 60);
  pickups = pickups.filter(p => p.y < H + 60);

  const px = lanesX()[currentLane] + slipOffset;
  const py = PLAYER_Y();
  const pw = CAT_W - 4;
  const ph = CAT_H - 2;

  updateCheat(dt);

  const collisionsActive = graceTimer <= 0;

  if (collisionsActive){
    for (let i=0; i<enemies.length; i++){
      const e = enemies[i];
      // per-type hitbox tweaks
      let ew=e.w, eh=e.h;
      if (e.type === 'tree'){ ew *= 0.58; eh *= 0.78; }
      if (e.type === 'rock'){ ew *= 0.75; eh *= 0.75; }
      if (e.type === 'bush'){ ew *= 0.70; eh *= 0.65; }
      if (e.type === 'log'){  ew *= 0.80; eh *= 0.70; }

      if (Math.abs(e.x - px) < (ew + pw)/2 && Math.abs(e.y - py) < (eh + ph)/2){
        pickupStreak = 0; // break combo
        if (e.type === 'mud'){ // legacy mud still possible
          enemies.splice(i,1);
          fuel = Math.max(0, fuel - 10);
          score = Math.max(0, score - 2);
          slipTimer = 0.6;
        } else {
          if (cheatCharges > 0){
            enemies.splice(i,1);
            cheatCharges--;
          } else endGame();
        }
        break;
      }
    }
  }

  for (let i=0; i<pickups.length; i++){
    const p = pickups[i];
    if (Math.abs(p.x - px) < (p.w + pw)/2 && Math.abs(p.y - py) < (p.h + ph)/2){
      pickups.splice(i,1);
      pickupStreak++;
      if (pickupStreak>=3) addAnnounce(`Combo x${pickupStreak}!`, 0.9);

      if (p.type === 'chicken'){
        fuel = Math.min(100, fuel + 25);
        score += 15;
        cheatCharges = Math.min(2, cheatCharges + 1);
        addAnnounce('Lucky Chicken!', 1.2);
        spawnSparkles(px, py, 'rgba(255,230,140,0.95)');
      } else if (p.type === 'lizard'){
        fuel = Math.min(100, fuel + (p.golden ? 22 : 12));
        score += (p.golden ? 12 : 5);
        spawnSparkles(px, py, 'rgba(180,255,120,0.95)');
      } else if (p.type === 'bird'){
        fuel = Math.min(100, fuel + (p.golden ? 20 : 10));
        score += (p.golden ? 10 : 3);
        spawnSparkles(px, py, 'rgba(180,210,255,0.95)');
      } else { // mouse
        fuel = Math.min(100, fuel + (p.golden ? 20 : 10));
        score += (p.golden ? 10 : 3);
        spawnSparkles(px, py, 'rgba(255,230,150,0.95)');
      }

      // score milestone announcer
      if (score >= nextScoreMilestone){
        addAnnounce(`Score ${nextScoreMilestone}!`, 1.0);
        nextScoreMilestone += 50;
      }
      break;
    }
  }

  updateParticles(dt);
  updateAnnouncements(dt);

  meters += (roadSpeed * dt) / 120;
  fuel -= dt * 2;
  if (fuel <= 0) endGame();
}

function drawHUD(){
  ctx.fillStyle = '#fff'; ctx.font = '16px system-ui, sans-serif';
  ctx.fillText('Score: '  + score, 10, 22);
  ctx.fillText('Energy: ' + Math.round(fuel), 10, 42);
  ctx.fillText('Meters: ' + Math.round(meters), 10, 62);

  const txt = cheatCharges > 0 ? `Shield x${cheatCharges}` : '';
  if (txt){
    const w = ctx.measureText(txt).width + 12;
    const x = W - w - 10;
    ctx.fillStyle = '#fff';
    ctx.fillText(txt, x, 28);
  }
}

function drawVignette(){
  const g = ctx.createRadialGradient(W/2, H*0.58, Math.min(W,H)*0.25, W/2, H*0.58, Math.max(W,H)*0.75);
  g.addColorStop(0, 'rgba(0,0,0,0)');
  g.addColorStop(1, 'rgba(0,0,0,0.35)');
  ctx.fillStyle = g;
  ctx.fillRect(0,0,W,H);
}

function draw(dt){
  drawBackgroundBase();
  drawHills(dt);
  enemies.forEach(e => {
    if (e.type==='tree') drawTree(e.x,e.y,e.w,e.h);
    else if (e.type==='rock') drawRock(e.x,e.y,e.w,e.h);
    else if (e.type==='bush') drawBush(e.x,e.y,e.w,e.h);
    else if (e.type==='log')  drawLog(e.x,e.y,e.w,e.h);
    else drawMud(e.x,e.y,e.w,e.h); // fallback
  });
  pickups.forEach(drawPickup);
  drawCharacter(lanesX()[currentLane] + slipOffset, PLAYER_Y(), CAT_W, CAT_H);
  drawParticles();
  drawHUD();
  drawAnnouncements();
  drawVignette();
}

function drawMenu(){
  drawBackgroundBase();
  drawHills(0);
  ctx.fillStyle = '#fff';

  ctx.font = '16px system-ui, sans-serif';
  const title = 'Cat Dash';
  ctx.fillText(title, (W - ctx.measureText(title).width)/2, 56);

  ctx.font = '14px system-ui, sans-serif';
  const sub = 'Tap, swipe, arrows — press C or tap top-right to change character';
  ctx.fillText(sub, (W - ctx.measureText(sub).width)/2, 78);

  const lines = ['Aim: Reach a high score', 'collect golden critters', 'avoid hazards'];
  lines.forEach((line,i)=> ctx.fillText(line, (W - ctx.measureText(line).width)/2, 98 + i*16));

  // Skin display with locks
  const y0 = 140;
  ctx.font = '14px system-ui, sans-serif';
  let x = 24;
  SKINS.forEach((s, i)=>{
    const unlocked = isSkinUnlocked(i);
    const label = unlocked ? s.name : `${s.name} 🔒${s.unlock}`;
    ctx.globalAlpha = (i===currentSkinIndex && unlocked) ? 1 : 0.7;
    ctx.fillText(label, x, y0);
    // tiny avatar
    drawSkinCharacter(s, x+40, y0+26, 16, 24);
    ctx.globalAlpha = 1;
    x += 120;
  });

  drawBoard(24, y0 + 64);
  drawVignette();
}

/* ===== Control ===== */
function endGame(){
  gameRunning = false;
  maybeRecordScore(score);
  saveBest(score);
}
function resetGame(){
  enemies = [];
  pickups = [];
  particles = [];
  announcements = [];
  score = 0;
  fuel = 100;
  meters = 0;
  roadSpeed = 226;
  currentLane = 1;
  spawnTimer = 0;
  graceTimer = 0.75;
  last = undefined;
  cheatArmTimerMs = 0;
  cheatCharges = 0;
  cheatRearmLock = false;
  pickupStreak = 0;
  nextScoreMilestone = 50;
}

function loop(ts){
  if (last === undefined) last = ts;
  let dt = (ts - last) / 1000;
  if (!Number.isFinite(dt) || dt < 0) dt = 0;
  dt = Math.min(dt, 0.05);
  last = ts;

  ctx.clearRect(0,0,W,H);

  if (gameRunning){
    if (graceTimer > 0) graceTimer = Math.max(0, graceTimer - dt);
    update(dt);
    draw(dt);
  } else {
    drawMenu();
  }
  requestAnimationFrame(loop);
}

/* Keyboard */
let keyLock = false;
document.addEventListener('keydown', e=>{
  if (!gameRunning){
    if (e.key.toLowerCase() === 'c'){ cycleSkin(); return; }
    // start on any other key
    resetGame(); gameRunning = true; return;
  }
  if (keyLock) return;
  if (e.key === 'ArrowLeft'){
    if (currentLane > 0) currentLane--;
    keyLock = true;
  } else if (e.key === 'ArrowRight'){
    if (currentLane < 2) currentLane++;
    keyLock = true;
  }
});
document.addEventListener('keyup', e=>{
  if (e.key === 'ArrowLeft' || e.key === 'ArrowRight') keyLock = false;
});

/* Touch on canvas */
let touchStartX = null;
canvas.addEventListener('touchstart', e=>{
  // top-right 100x100 area cycles skins when not running
  const t = e.touches[0];
  if (!gameRunning && t.clientX > W-100 && t.clientY < 120){ cycleSkin(); return; }

  if (!gameRunning){ resetGame(); gameRunning = true; }
  touchStartX = t.clientX;
}, {passive: true});
canvas.addEventListener('touchmove', e=>{
  if (touchStartX === null) return;
  const dx = e.touches[0].clientX - touchStartX;
  if (dx > 50 && currentLane < 2){ currentLane++; touchStartX = e.touches[0].clientX; }
  else if (dx < -50 && currentLane > 0){ currentLane--; touchStartX = e.touches[0].clientX; }
}, {passive: true});
canvas.addEventListener('touchend', ()=>{ touchStartX = null; });

/* Lane buttons — pointer events for hold state */
const btn1 = document.getElementById('btnLane1');
const btn3 = document.getElementById('btnLane3');

function nudgeLeft(){ if (!gameRunning){ resetGame(); gameRunning = true; } if (currentLane > 0) currentLane--; }
function nudgeRight(){ if (!gameRunning){ resetGame(); gameRunning = true; } if (currentLane < 2) currentLane++; }

function onPointerDownBtn1(e){ e.preventDefault(); btn1Down = true; nudgeLeft(); }
function onPointerUpBtn1(e){ e.preventDefault(); btn1Down = false; if (!btn3Down && cheatCharges===0) cheatArmTimerMs = 0; }
function onPointerDownBtn3(e){ e.preventDefault(); btn3Down = true; nudgeRight(); }
function onPointerUpBtn3(e){ e.preventDefault(); btn3Down = false; if (!btn1Down && cheatCharges===0) cheatArmTimerMs = 0; }

['pointerdown'].forEach(evt=>{
  btn1.addEventListener(evt, onPointerDownBtn1, {passive:false});
  btn3.addEventListener(evt, onPointerDownBtn3, {passive:false});
});
['pointerup','pointercancel','pointerout','pointerleave'].forEach(evt=>{
  btn1.addEventListener(evt, onPointerUpBtn1, {passive:false});
  btn3.addEventListener(evt, onPointerUpBtn3, {passive:false});
});

/* Click to start or cycle skin area */
canvas.addEventListener('click', e=>{
  if (!gameRunning){
    // allow clicking top-right to cycle skin
    if (e.clientX > W-100 && e.clientY < 120){ cycleSkin(); return; }
    resetGame(); gameRunning = true; return;
  }
  const x = e.clientX;
  const center = lanesX()[currentLane];
  if (x < center) nudgeLeft(); else if (x > center) nudgeRight();
});

/* Start */
requestAnimationFrame(loop);
</script>
</body>
</html>
