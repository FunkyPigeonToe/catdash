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
let score = 0;
let fuel = 100;
let meters = 0;

let spawnTimer = 0;
let last = undefined;
let graceTimer = 0;

/* Speed ~10% faster than the slow build */
let roadSpeed = 226;
let maxSpeed  = 704;

const baseAccel = 0.6;
let slipTimer = 0, slipOffset = 0;

/* Secret shield cheat: double-press lane buttons (gives 2 tree ignores) */
let shieldCharges = 0;
const COMBO_WINDOW_MS = 180;
let lastBtn1Time = -1, lastBtn3Time = -1;
let shieldFlashTimer = 0;
function tryActivateShield(fromBtn){
  const now = performance.now();
  if (fromBtn === 1){
    if (now - lastBtn3Time <= COMBO_WINDOW_MS && shieldCharges === 0){
      shieldCharges = 2; shieldFlashTimer = 0.6;
    }
    lastBtn1Time = now;
  } else {
    if (now - lastBtn1Time <= COMBO_WINDOW_MS && shieldCharges === 0){
      shieldCharges = 2; shieldFlashTimer = 0.6;
    }
    lastBtn3Time = now;
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

/* ===== Background & rounded trees ===== */
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

/* Rounder tree (clustered canopy + trunk) */
function drawTree(x,y,w,h){
  withShadow('rgba(0,0,0,0.35)', 10, 4, ()=>{
    // trunk
    ctx.fillStyle = '#6d3f17';
    const trunkW = w*0.22, trunkH = h*0.45;
    ctx.fillRect(x - trunkW/2, y + h*0.1, trunkW, trunkH);

    // canopy blobs (rounded)
    const cx = x, cy = y - h*0.10;
    const rBig = h*0.30, rSide = h*0.22, rTop = h*0.18;
    ctx.fillStyle = '#2e7d32';
    ctx.beginPath(); ctx.arc(cx, cy, rBig, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(cx - w*0.28, cy + h*0.02, rSide, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(cx + w*0.28, cy + h*0.02, rSide, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(cx, cy - h*0.22, rTop, 0, Math.PI*2); ctx.fill();

    // subtle highlight rim
    ctx.strokeStyle = 'rgba(0,0,0,0.35)';
    ctx.lineWidth = 1.5;
    ctx.beginPath(); ctx.arc(cx, cy, rBig, 0, Math.PI*2); ctx.stroke();
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

/* ===== Land pickups: mice, birds, lizards ===== */
/* Drawing helpers for pickups – all a bit stylized & rounded */
function drawMouse(x, y, scale, golden){
  const t = performance.now()*0.006, wiggle = Math.sin(t + x*0.01)*1.2*scale;
  const body = golden ? '#ffd54f' : '#c7a17a';
  const ear   = golden ? '#ffe082' : '#d7b894';
  withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{
    // body (capsule)
    ctx.fillStyle = body;
    ctx.beginPath(); ctx.ellipse(x, y+wiggle, 12*scale, 7*scale, 0, 0, Math.PI*2); ctx.fill();
    // head
    ctx.beginPath(); ctx.ellipse(x+9*scale, y-1*scale+wiggle, 6*scale, 5*scale, 0, 0, Math.PI*2); ctx.fill();
    // ears
    ctx.fillStyle = ear;
    ctx.beginPath(); ctx.arc(x+12*scale, y-5*scale+wiggle, 2.8*scale, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(x+7.5*scale, y-6*scale+wiggle, 2.2*scale, 0, Math.PI*2); ctx.fill();
    // tail
    ctx.strokeStyle = golden ? '#ffe082' : '#b78963';
    ctx.lineWidth = 1.4*scale;
    ctx.beginPath(); ctx.moveTo(x-12*scale, y+2*scale+wiggle);
    ctx.quadraticCurveTo(x-18*scale, y+6*scale+wiggle, x-22*scale, y+3*scale+wiggle);
    ctx.stroke();
    // eye
    ctx.fillStyle = '#000';
    ctx.beginPath(); ctx.arc(x+11*scale, y-2*scale+wiggle, 1.4*scale, 0, Math.PI*2); ctx.fill();
  });
  if (golden){
    ctx.save();
    ctx.globalAlpha = 0.4 + 0.3*Math.sin(performance.now()*0.008);
    const g = ctx.createRadialGradient(x, y, 0, x, y, 22*scale);
    g.addColorStop(0,'rgba(255,215,0,0.9)'); g.addColorStop(1,'rgba(255,215,0,0)');
    ctx.fillStyle = g; ctx.beginPath(); ctx.arc(x, y, 22*scale, 0, Math.PI*2); ctx.fill();
    ctx.restore();
  }
}

function drawBird(x, y, scale, golden){
  const t = performance.now()*0.004, bob = Math.sin(t + x*0.02)*1.2*scale;
  const body = golden ? '#ffe066' : '#66a9ff';
  withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{
    // body
    ctx.fillStyle = body;
    ctx.beginPath(); ctx.ellipse(x, y+bob, 10*scale, 7*scale, 0, 0, Math.PI*2); ctx.fill();
    // head
    ctx.beginPath(); ctx.arc(x+7*scale, y-3*scale+bob, 4*scale, 0, Math.PI*2); ctx.fill();
    // wing
    ctx.fillStyle = golden ? '#ffd54f' : '#4f94f5';
    ctx.beginPath(); ctx.ellipse(x-3*scale, y+1*scale+bob, 6*scale, 4*scale, -0.7, 0, Math.PI*2); ctx.fill();
    // beak
    ctx.fillStyle = golden ? '#ffca28' : '#ffb300';
    ctx.beginPath(); ctx.moveTo(x+11*scale, y-3*scale+bob);
    ctx.lineTo(x+15*scale, y-1*scale+bob);
    ctx.lineTo(x+11*scale, y-1*scale+bob);
    ctx.closePath(); ctx.fill();
    // eye
    ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(x+6*scale, y-4*scale+bob, 1.2*scale, 0, Math.PI*2); ctx.fill();
    // tiny legs
    ctx.strokeStyle = golden ? '#ffb300' : '#cc7a00';
    ctx.lineWidth = 1.2*scale;
    ctx.beginPath();
    ctx.moveTo(x-2*scale, y+7*scale+bob); ctx.lineTo(x-2*scale, y+9.5*scale+bob);
    ctx.moveTo(x+1*scale, y+7*scale+bob); ctx.lineTo(x+1*scale, y+9.5*scale+bob);
    ctx.stroke();
  });
  if (golden){
    ctx.save();
    ctx.globalAlpha = 0.4 + 0.3*Math.sin(performance.now()*0.01);
    const g = ctx.createRadialGradient(x, y, 0, x, y, 20*scale);
    g.addColorStop(0,'rgba(255,215,0,0.9)'); g.addColorStop(1,'rgba(255,215,0,0)');
    ctx.fillStyle = g; ctx.beginPath(); ctx.arc(x, y, 20*scale, 0, Math.PI*2); ctx.fill();
    ctx.restore();
  }
}

function drawLizard(x, y, scale, golden){
  const t = performance.now()*0.006, sway = Math.sin(t + x*0.03)*1.5*scale;
  const body = golden ? '#ffd54f' : '#66cc66';
  const spot = golden ? '#ffe082' : '#4caf50';
  withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{
    // body
    ctx.fillStyle = body;
    ctx.beginPath(); ctx.ellipse(x, y+sway, 12*scale, 5.5*scale, 0, 0, Math.PI*2); ctx.fill();
    // head
    ctx.beginPath(); ctx.ellipse(x+9*scale, y-0.5*scale+sway, 5*scale, 4*scale, 0, 0, Math.PI*2); ctx.fill();
    // tail
    ctx.beginPath();
    ctx.moveTo(x-12*scale, y+sway);
    ctx.quadraticCurveTo(x-20*scale, y+4*scale+sway, x-24*scale, y+1*scale+sway);
    ctx.quadraticCurveTo(x-19*scale, y-2*scale+sway, x-12*scale, y+sway);
    ctx.fill();
    // legs (simple)
    ctx.fillStyle = spot;
    for (let i=-1;i<=1;i+=2){
      ctx.fillRect(x-2*scale, y+5*scale+sway, 3*scale*i, 2*scale);
      ctx.fillRect(x+6*scale, y+4*scale+sway, 3*scale*i, 2*scale);
    }
    // eye
    ctx.fillStyle = '#000';
    ctx.beginPath(); ctx.arc(x+11*scale, y-1*scale+sway, 1.2*scale, 0, Math.PI*2); ctx.fill();
  });
  if (golden){
    ctx.save();
    ctx.globalAlpha = 0.4 + 0.3*Math.sin(performance.now()*0.012);
    const g = ctx.createRadialGradient(x, y, 0, x, y, 22*scale);
    g.addColorStop(0,'rgba(255,215,0,0.9)'); g.addColorStop(1,'rgba(255,215,0,0)');
    ctx.fillStyle = g; ctx.beginPath(); ctx.arc(x, y, 22*scale, 0, Math.PI*2); ctx.fill();
    ctx.restore();
  }
}

function drawPickup(p){
  if (p.type === 'mouse')   drawMouse(p.x, p.y, p.scale, p.golden);
  else if (p.type === 'bird') drawBird(p.x, p.y, p.scale, p.golden);
  else drawLizard(p.x, p.y, p.scale, p.golden);
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

// keep pickups out of lanes that currently have enemies anywhere (same behavior as before)
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

  // Choose pickup type: mice & birds common, lizard rare
  const r = Math.random();
  const type = r < 0.45 ? 'mouse' : (r < 0.90 ? 'bird' : 'lizard');
  const golden = Math.random() < 0.25;
  let scale = 1;
  if (type === 'bird') scale = golden ? 1.25 : 1.05;
  if (type === 'mouse') scale = golden ? 1.2 : 1.0;
  if (type === 'lizard') scale = golden ? 1.25 : 1.1;

  // Hitboxes roughly matching visuals
  const baseW = type === 'bird' ? 30 : (type === 'mouse' ? 34 : 36);
  const baseH = type === 'bird' ? 18 : (type === 'mouse' ? 18 : 16);
  const w = baseW * scale;
  const h = baseH * scale;

  pickups.push({type, x: lx[lane], y: spawnY, w, h, scale, golden});
}

/* ===== Cat ===== */
const CAT_W = 20, CAT_H = 30;
function drawCat(x, y, w, h){
  withShadow('rgba(0,0,0,0.35)', 12, 5, ()=>{
    ctx.fillStyle = '#d2691e';
    ctx.beginPath(); ctx.ellipse(x, y, w/2, h/2, 0, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = '#a0522d';
    ctx.beginPath(); ctx.ellipse(x, y, w/2.5, h/2.5, 0, 0, Math.PI*2); ctx.fill();
    const headR = h*0.25, hx=x, hy=y - h*0.55;
    ctx.fillStyle = '#d2691e'; ctx.beginPath(); ctx.arc(hx,hy,headR,0,Math.PI*2); ctx.fill();
    ctx.fillStyle = '#a0522d'; ctx.beginPath(); ctx.arc(hx,hy,headR*0.75,0,Math.PI*2); ctx.fill();
    ctx.fillStyle = '#d2691e';
    ctx.beginPath(); ctx.moveTo(hx-headR*0.8,hy-headR*0.2); ctx.lineTo(hx-headR*0.3,hy-headR*1.1); ctx.lineTo(hx-headR*0.05,hy-headR*0.2); ctx.closePath(); ctx.fill();
    ctx.beginPath(); ctx.moveTo(hx+headR*0.8,hy-headR*0.2); ctx.lineTo(hx+headR*0.3,hy-headR*1.1); ctx.lineTo(hx+headR*0.05,hy-headR*0.2); ctx.closePath(); ctx.fill();
    ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(hx-headR*0.4, hy, headR*0.15, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(hx+headR*0.4, hy, headR*0.15, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = '#d2691e'; ctx.beginPath(); ctx.ellipse(x + w/2.2, y, w/6, h/3, 0, 0, Math.PI*2); ctx.fill();
  });
  strokeAround('rgba(0,0,0,0.4)', 1, ()=>{
    ctx.beginPath(); ctx.ellipse(x, y, w/2, h/2, 0, 0, Math.PI*2);
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
    p.vy += 80 * dt; // gravity-ish
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

  const collisionsActive = graceTimer <= 0;

  if (collisionsActive){
    for (let i=0; i<enemies.length; i++){
      const e = enemies[i];

      // slimmer hitbox for trees (and a touch shorter)
      let ew = e.w * (e.type === 'tree' ? 0.6 : 1);
      let eh = e.h * (e.type === 'tree' ? 0.8 : 1);

      if (Math.abs(e.x - px) < (ew + pw)/2 && Math.abs(e.y - py) < (eh + ph)/2){
        if (e.type === 'mud'){
          enemies.splice(i,1);
          fuel = Math.max(0, fuel - 10);
          score = Math.max(0, score - 2);
          slipTimer = 0.6;
        } else {
          if (shieldCharges > 0){
            enemies.splice(i,1);
            shieldCharges--;
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
      // rewards tuned per type; golden gives more
      if (p.type === 'lizard'){
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

  meters += (roadSpeed * dt) / 120;
  fuel -= dt * 2;
  if (fuel <= 0) endGame();

  if (shieldFlashTimer > 0) shieldFlashTimer = Math.max(0, shieldFlashTimer - dt);
}

function drawHUD(){
  ctx.fillStyle = '#fff'; ctx.font = '16px system-ui, sans-serif';
  ctx.fillText('Score: '  + score, 10, 22);
  ctx.fillText('Energy: ' + Math.round(fuel), 10, 42);
  ctx.fillText('Meters: ' + Math.round(meters), 10, 62);

  const txt = shieldCharges > 0 ? `Shield x${shieldCharges}` : '';
  if (txt){
    const w = ctx.measureText(txt).width + 12;
    const x = W - w - 10;
    if (shieldFlashTimer > 0){
      ctx.save();
      ctx.globalAlpha = 0.6 + 0.4 * Math.sin(performance.now()/80);
      ctx.fillStyle = '#ffd54f';
      ctx.fillRect(x - 4, 10, w + 8, 22);
      ctx.restore();
    }
    ctx.fillStyle = '#fff';
    ctx.fillText(txt, x, 28);
  }
}

/* Gentle vignette overlay */
function drawVignette(){
  const g = ctx.createRadialGradient(W/2, H*0.58, Math.min(W,H)*0.25, W/2, H*0.58, Math.max(W,H)*0.75);
  g.addColorStop(0, 'rgba(0,0,0,0)');
  g.addColorStop(1, 'rgba(0,0,0,0.35)');
  ctx.fillStyle = g;
  ctx.fillRect(0,0,W,H);
}

function draw(dt){
  drawBackgroundBase();
  drawHills(dt); // parallax under lanes
  enemies.forEach(e => { if (e.type==='tree') drawTree(e.x,e.y,e.w,e.h); else drawMud(e.x,e.y,e.w,e.h); });
  pickups.forEach(drawPickup);
  drawCat(lanesX()[currentLane] + slipOffset, PLAYER_Y(), CAT_W, CAT_H);
  drawParticles();
  drawHUD();
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
  shieldCharges = 0;
  shieldFlashTimer = 0;
  lastBtn1Time = lastBtn3Time = -1;
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

/* Lane buttons */
const btn1 = document.getElementById('btnLane1');
const btn3 = document.getElementById('btnLane3');

function nudgeLeft(){ if (!gameRunning){ resetGame(); gameRunning = true; } if (currentLane > 0) currentLane--; }
function nudgeRight(){ if (!gameRunning){ resetGame(); gameRunning = true; } if (currentLane < 2) currentLane++; }

['click','touchstart'].forEach(evt=>{
  btn1.addEventListener(evt, e=>{
    e.preventDefault();
    tryActivateShield(1);
    nudgeLeft();
  }, {passive: false});
  btn3.addEventListener(evt, e=>{
    e.preventDefault();
    tryActivateShield(3);
    nudgeRight();
  }, {passive: false});
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
