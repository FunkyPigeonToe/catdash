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
    background: #6dbb4a; /* static, no animated background */
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
let score = 0;
let fuel = 100;
let meters = 0;

let spawnTimer = 0;
let last = undefined;
let graceTimer = 0;

/* Speed */
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
let cheatToastTimer = 0;    // UI toast for "+1" feedback
let cheatToastText = '';

function updateCheat(dt){
  const bothDown = btn1Down && btn3Down;
  if (bothDown && !cheatRearmLock && cheatCharges === 0){
    cheatArmTimerMs += dt * 1000;
    if (cheatArmTimerMs >= CHEAT_HOLD_TIME_MS){
      cheatCharges = 2;           // arm
      cheatRearmLock = true;      // prevent immediate rearm
      cheatToastText = 'Shield armed x2';
      cheatToastTimer = 1.0;
    }
  } else {
    cheatArmTimerMs = 0;
  }
  // When both buttons are released, allow re-arm next time (only if no current charges)
  if (!btn1Down && !btn3Down && cheatCharges === 0){
    cheatRearmLock = false;
  }
}

/* ===== Leaderboard ===== */
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

/* ===== Static Background (no motion) ===== */
function drawBackgroundBase(){
  // soft grass gradient (static)
  const g = ctx.createLinearGradient(0, 0, 0, H);
  g.addColorStop(0, '#64b24a'); g.addColorStop(1, '#4d9c3b');
  ctx.fillStyle = g; ctx.fillRect(0,0,W,H);

  // subtle grass texture
  ctx.fillStyle = 'rgba(40,90,40,0.10)';
  for (let y=0; y<H; y+=40){
    for (let x=((y/40)%2===0?0:20); x<W; x+=40){ ctx.fillRect(x,y,10,10); }
  }

  // dirt tracks for each lane
  const trailW = W/6; const lx = lanesX();
  for (let i=0;i<3;i++){
    const rg = ctx.createLinearGradient(0, 0, 0, H);
    rg.addColorStop(0, '#8b684f'); rg.addColorStop(1, '#6f523f');
    ctx.fillStyle = rg; ctx.fillRect(lx[i]-trailW/2, 0, trailW, H);
    ctx.fillStyle = 'rgba(0,0,0,0.18)'; ctx.fillRect(lx[i]-1, 0, 2, H);
  }
}

/* ===== New Cartoony Tree ===== */
function drawTree(x,y,w,h){
  withShadow('rgba(0,0,0,0.35)', 10, 4, ()=>{
    // rounded trunk
    const trunkW = w*0.28, trunkH = h*0.48;
    const trunkX = x - trunkW/2, trunkY = y + h*0.12;
    ctx.fillStyle = '#6d3f17';
    ctx.beginPath();
    const r = trunkW*0.35;
    // rounded-rect trunk
    ctx.moveTo(trunkX + r, trunkY);
    ctx.lineTo(trunkX + trunkW - r, trunkY);
    ctx.quadraticCurveTo(trunkX + trunkW, trunkY, trunkX + trunkW, trunkY + r);
    ctx.lineTo(trunkX + trunkW, trunkY + trunkH - r);
    ctx.quadraticCurveTo(trunkX + trunkW, trunkY + trunkH, trunkX + trunkW - r, trunkY + trunkH);
    ctx.lineTo(trunkX + r, trunkY + trunkH);
    ctx.quadraticCurveTo(trunkX, trunkY + trunkH, trunkX, trunkY + trunkH - r);
    ctx.lineTo(trunkX, trunkY + r);
    ctx.quadraticCurveTo(trunkX, trunkY, trunkX + r, trunkY);
    ctx.closePath();
    ctx.fill();

    // puffy canopy (3 layers) with highlight
    const cx = x, cy = y - h*0.06;
    const cMain = '#2e7d32';
    const cMid  = '#2f8c34';
    const cLight= '#399c3a';

    // back blobs
    ctx.fillStyle = cMid;
    ctx.beginPath(); ctx.arc(cx - w*0.28, cy + h*0.02, h*0.25, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(cx + w*0.28, cy + h*0.02, h*0.25, 0, Math.PI*2); ctx.fill();

    // main blob
    ctx.fillStyle = cMain;
    ctx.beginPath(); ctx.arc(cx, cy, h*0.32, 0, Math.PI*2); ctx.fill();

    // top highlight blob
    ctx.fillStyle = cLight;
    ctx.beginPath(); ctx.arc(cx, cy - h*0.22, h*0.18, 0, Math.PI*2); ctx.fill();

    // subtle outline around main canopy
    ctx.strokeStyle = 'rgba(0,0,0,0.35)';
    ctx.lineWidth = 1.5;
    ctx.beginPath(); ctx.arc(cx, cy, h*0.32, 0, Math.PI*2); ctx.stroke();
  });
}

/* Mud */
function drawMud(x, y, w, h){
  withShadow('rgba(0,0,0,0.3)', 8, 3, ()=>{
    const g = ctx.createRadialGradient(x, y, 2, x, y, Math.max(w,h));
    g.addColorStop(0, '#6a4a3a'); g.addColorStop(1, '#3e2723');
    ctx.fillStyle = g; ctx.beginPath(); ctx.ellipse(x, y, w*0.5, h*0.5, 0, 0, Math.PI*2); ctx.fill();
  });
  strokeAround('rgba(0,0,0,0.35)', 1.2, ()=>{
    ctx.beginPath(); ctx.ellipse(x, y, w*0.5, h*0.5, 0, 0, Math.PI*2);
  });
}

/* ===== Pickups (mouse, bird, lizard, chicken) ===== */
/* Additive, fixed-radius glow to avoid screen brightness pulsing */
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

/* Improved Lizard */
function drawLizard(x, y, scale, golden){
  const t = performance.now()*0.006;
  const sway = Math.sin(t + x*0.03)*1.5*scale;
  const body = golden ? '#ffd54f' : '#5cb85c';
  const belly = golden ? '#ffe082' : '#4cae4c';

  withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{
    // body
    ctx.fillStyle = body;
    ctx.beginPath();
    ctx.ellipse(x, y+sway, 16*scale, 6*scale, 0, 0, Math.PI*2);
    ctx.fill();

    // tail
    ctx.beginPath();
    ctx.moveTo(x-16*scale, y+sway);
    ctx.quadraticCurveTo(x-26*scale, y+3*scale+sway, x-30*scale, y+sway);
    ctx.lineTo(x-24*scale, y-2*scale+sway);
    ctx.closePath();
    ctx.fill();

    // head
    ctx.beginPath();
    ctx.ellipse(x+14*scale, y-1*scale+sway, 6*scale, 5*scale, 0, 0, Math.PI*2);
    ctx.fill();

    // belly stripe
    ctx.fillStyle = belly;
    ctx.fillRect(x-6*scale, y-2*scale+sway, 12*scale, 4*scale);

    // legs
    ctx.strokeStyle = body;
    ctx.lineWidth = 2*scale;
    ctx.beginPath();
    ctx.moveTo(x-4*scale, y+5*scale+sway); ctx.lineTo(x-8*scale, y+9*scale+sway);
    ctx.moveTo(x+4*scale, y+5*scale+sway); ctx.lineTo(x+8*scale, y+9*scale+sway);
    ctx.moveTo(x-4*scale, y-5*scale+sway); ctx.lineTo(x-8*scale, y-9*scale+sway);
    ctx.moveTo(x+4*scale, y-5*scale+sway); ctx.lineTo(x+8*scale, y-9*scale+sway);
    ctx.stroke();

    // eyes
    ctx.fillStyle = '#000';
    ctx.beginPath(); ctx.arc(x+17*scale, y-2*scale+sway, 1.4*scale, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(x+12*scale, y-2*scale+sway, 1.4*scale, 0, Math.PI*2); ctx.fill();
  });

  if (golden) drawAdditiveGlow(x, y, 24*scale, 0.85);
}

/* Golden Chicken (always golden, rare, big reward, +1 shield) */
function drawChicken(x, y, scale){
  const bob = Math.sin(performance.now()*0.004 + x*0.01) * 1.0 * scale;
  withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{
    // body
    ctx.fillStyle = '#ffe082';
    ctx.beginPath(); ctx.ellipse(x, y+bob, 12*scale, 9*scale, 0, 0, Math.PI*2); ctx.fill();
    // head
    ctx.beginPath(); ctx.arc(x+9*scale, y-5*scale+bob, 5*scale, 0, Math.PI*2); ctx.fill();
    // beak
    ctx.fillStyle = '#ffb300';
    ctx.beginPath(); ctx.moveTo(x+14*scale, y-5*scale+bob);
    ctx.lineTo(x+18*scale, y-4*scale+bob);
    ctx.lineTo(x+14*scale, y-2.5*scale+bob);
    ctx.closePath(); ctx.fill();
    // comb
    ctx.fillStyle = '#e53935';
    ctx.beginPath(); ctx.arc(x+8*scale, y-9*scale+bob, 2.1*scale, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(x+10.8*scale, y-9.4*scale+bob, 1.8*scale, 0, Math.PI*2); ctx.fill();
    // wing
    ctx.fillStyle = '#ffd54f';
    ctx.beginPath(); ctx.ellipse(x-3*scale, y-1*scale+bob, 7*scale, 5*scale, -0.7, 0, Math.PI*2); ctx.fill();
    // eye
    ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(x+8*scale, y-6*scale+bob, 1.4*scale, 0, Math.PI*2); ctx.fill();
    // legs
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
  else                         drawChicken(p.x, p.y, p.scale); // golden chicken
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
  const type = Math.random() < 0.55 ? 'tree' : 'mud';
  enemies.push(
    type==='tree'
      ? {type, x: lx[lane], y: -50, w: 40, h: 70}
      : {type, x: lx[lane], y: -40, w: 56, h: 24}
  );
}
function laneHasAnyEnemy(laneIndex){
  const lx = lanesX(); const x = lx[laneIndex];
  return enemies.some(e => e.x === x);
}

function spawnPickup(){
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

  // Choose pickup type: mice & birds common, lizard less common, golden chicken rare
  const r = Math.random();
  let type, golden=false, scale=1;
  if (r < 0.40){ type='mouse';   golden = Math.random()<0.25; scale = golden?1.2:1.0; }
  else if (r < 0.80){ type='bird';   golden = Math.random()<0.25; scale = golden?1.25:1.05; }
  else if (r < 0.95){ type='lizard'; golden = Math.random()<0.25; scale = golden?1.25:1.1; }
  else { type='chicken'; golden=true; scale=1.35; } // always golden

  // Hitboxes roughly matching visuals
  const baseW = type==='bird' ? 30 : type==='mouse' ? 34 : type==='lizard' ? 36 : 38;
  const baseH = type==='bird' ? 18 : type==='mouse' ? 18 : type==='lizard' ? 16 : 22;
  const w = baseW * scale;
  const h = baseH * scale;

  pickups.push({type, x: lx[lane], y: spawnY, w, h, scale, golden});
}

/* ===== Cat (more cat-like) ===== */
const CAT_W = 20, CAT_H = 30;
function drawCat(x, y, w, h){
  withShadow('rgba(0,0,0,0.35)', 12, 5, ()=>{
    // body
    ctx.fillStyle = '#d2691e';
    ctx.beginPath();
    ctx.ellipse(x, y, w/1.6, h/1.15, 0, 0, Math.PI*2);
    ctx.fill();

    // head
    const headR = h*0.36;
    const hx = x, hy = y - h*0.78;
    ctx.beginPath(); ctx.arc(hx,hy,headR,0,Math.PI*2); ctx.fill();

    // ears
    ctx.beginPath();
    ctx.moveTo(hx-headR*0.6,hy-headR*0.15);
    ctx.lineTo(hx-headR*0.25,hy-headR*1.0);
    ctx.lineTo(hx-0,hy-headR*0.15);
    ctx.closePath(); ctx.fill();
    ctx.beginPath();
    ctx.moveTo(hx+headR*0.6,hy-headR*0.15);
    ctx.lineTo(hx+headR*0.25,hy-headR*1.0);
    ctx.lineTo(hx+0,hy-headR*0.15);
    ctx.closePath(); ctx.fill();

    // belly patch
    ctx.fillStyle = '#a0522d';
    ctx.beginPath();
    ctx.ellipse(x, y+2, w/2.6, h/2.6, 0, 0, Math.PI*2);
    ctx.fill();

    // eyes
    ctx.fillStyle = '#000';
    ctx.beginPath(); ctx.arc(hx-headR*0.35, hy, headR*0.15, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(hx+headR*0.35, hy, headR*0.15, 0, Math.PI*2); ctx.fill();

    // whiskers
    ctx.strokeStyle = '#fff';
    ctx.lineWidth = 1.2;
    ctx.beginPath();
    ctx.moveTo(hx- headR*0.55, hy);   ctx.lineTo(hx- headR*1.1, hy-2);
    ctx.moveTo(hx- headR*0.55, hy+4); ctx.lineTo(hx- headR*1.1, hy+6);
    ctx.moveTo(hx- headR*0.55, hy-4); ctx.lineTo(hx- headR*1.1, hy-6);
    ctx.moveTo(hx+ headR*0.55, hy);   ctx.lineTo(hx+ headR*1.1, hy-2);
    ctx.moveTo(hx+ headR*0.55, hy+4); ctx.lineTo(hx+ headR*1.1, hy+6);
    ctx.moveTo(hx+ headR*0.55, hy-4); ctx.lineTo(hx+ headR*1.1, hy-6);
    ctx.stroke();

    // tail
    ctx.beginPath();
    ctx.moveTo(x+w/2.2, y);
    ctx.quadraticCurveTo(x+w/1.5, y-h/2, x+w/2.5, y-h);
    ctx.lineWidth = 6;
    ctx.strokeStyle = '#d2691e';
    ctx.stroke();
  });

  strokeAround('rgba(0,0,0,0.4)', 1, ()=>{
    ctx.beginPath(); ctx.ellipse(x, y, w/1.6, h/1.15, 0, 0, Math.PI*2);
  });
}

/* ===== Particles (sparkle burst on pickup) ===== */
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
      let ew = e.w, eh = e.h;
      if (e.type === 'tree'){ ew *= 0.6; eh *= 0.8; }

      if (Math.abs(e.x - px) < (ew + pw)/2 && Math.abs(e.y - py) < (eh + ph)/2){
        if (e.type === 'mud'){
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
      if (p.type === 'chicken'){
        // Big reward + Shield +1 (capped at 2)
        fuel = Math.min(100, fuel + 25);
        score += 15;
        cheatCharges = Math.min(2, cheatCharges + 1);
        cheatToastText = 'Shield +1';
        cheatToastTimer = 1.2;
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
      break;
    }
  }

  updateParticles(dt);

  // toast timer
  if (cheatToastTimer > 0) cheatToastTimer = Math.max(0, cheatToastTimer - dt);

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

  // toast (center-top)
  if (cheatToastTimer > 0){
    const a = Math.min(1, cheatToastTimer / 0.3); // quick fade in
    ctx.save();
    ctx.globalAlpha = a;
    ctx.font = '18px system-ui, sans-serif';
    const t = cheatToastText || '';
    const tw = ctx.measureText(t).width;
    const tx = (W - tw)/2;
    const ty = 44;
    // soft panel
    ctx.fillStyle = 'rgba(0,0,0,0.35)';
    ctx.fillRect(tx - 12, ty - 18, tw + 24, 28);
    ctx.fillStyle = '#ffd54f';
    ctx.fillText(t, tx, ty);
    ctx.restore();
  }
}

/* Static vignette */
function drawVignette(){
  const g = ctx.createRadialGradient(W/2, H*0.58, Math.min(W,H)*0.25, W/2, H*0.58, Math.max(W,H)*0.75);
  g.addColorStop(0, 'rgba(0,0,0,0)');
  g.addColorStop(1, 'rgba(0,0,0,0.35)');
  ctx.fillStyle = g;
  ctx.fillRect(0,0,W,H);
}

function draw(){
  drawBackgroundBase();
  // (No drawHills – removed moving background)
  enemies.forEach(e => { if (e.type==='tree') drawTree(e.x,e.y,e.w,e.h); else drawMud(e.x,e.y,e.w,e.h); });
  pickups.forEach(drawPickup);
  drawCat(lanesX()[currentLane] + slipOffset, PLAYER_Y(), CAT_W, CAT_H);
  drawParticles();
  drawHUD();
  drawVignette();
}

function drawMenu(){
  drawBackgroundBase();
  ctx.fillStyle = '#fff';

  ctx.font = '16px system-ui, sans-serif';
  const title = 'Cat Dash';
  ctx.fillText(title, (W - ctx.measureText(title).width)/2, 56);

  ctx.font = '14px system-ui, sans-serif';
  const sub = 'Tap, swipe, or use arrows to start';
  ctx.fillText(sub, (W - ctx.measureText(sub).width)/2, 78);

  const lines = ['Aim: Reach a high score', 'by collecting as many', 'golden critters as you can'];
  lines.forEach((line,i)=> ctx.fillText(line, (W - ctx.measureText(line).width)/2, 98 + i*16));

  drawBoard(24, 160);
  drawVignette();
}

/* ===== Control ===== */
function endGame(){
  gameRunning = false;
  maybeRecordScore(score);
}
function resetGame(){
  enemies = [];
  pickups = [];
  particles = [];
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
  cheatToastTimer = 0;
  cheatToastText = '';
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
    draw();
  } else {
    drawMenu();
  }
  requestAnimationFrame(loop);
}

/* Keyboard */
let keyLock = false;
document.addEventListener('keydown', e=>{
  if (!gameRunning){ resetGame(); gameRunning = true; }
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
  if (!gameRunning){ resetGame(); gameRunning = true; }
  touchStartX = e.touches[0].clientX;
}, {passive: true});
canvas.addEventListener('touchmove', e=>{
  if (touchStartX === null) return;
  const dx = e.touches[0].clientX - touchStartX;
  if (dx > 50 && currentLane < 2){ currentLane++; touchStartX = e.touches[0].clientX; }
  else if (dx < -50 && currentLane > 0){ currentLane--; touchStartX = e.touches[0].clientX; }
}, {passive: true});
canvas.addEventListener('touchend', ()=>{ touchStartX = null; });

/* Lane buttons — use pointer events so we can track hold state */
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

/* Tap anywhere */
canvas.addEventListener('click', e=>{
  if (!gameRunning){ resetGame(); gameRunning = true; return; }
  const x = e.clientX;
  const center = lanesX()[currentLane];
  if (x < center) nudgeLeft(); else if (x > center) nudgeRight();
});

/* Start */
requestAnimationFrame(loop);
</script>
</body>
</html>
