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
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

let W = window.innerWidth;
let H = window.innerHeight;
canvas.width = W;
canvas.height = H;

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
  canvas.width = W;
  canvas.height = H;
  positionButtons();
});

let gameRunning = false;
let currentLane = 1;
let enemies = [];
let pickups = [];
let score = 0;
let fuel = 100;
let meters = 0;

let spawnTimer = 0;
let last = 0;
let roadSpeed = 320;
let maxSpeed   = 1000;
const baseAccel = 0.6;
let slipTimer = 0, slipOffset = 0;

// Leaderboard
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

// Artwork
function drawBackground(){
  ctx.fillStyle = '#5e9d45'; ctx.fillRect(0,0,W,H);
  ctx.fillStyle = 'rgba(40,90,40,0.15)';
  for (let y=0; y<H; y+=40){
    for (let x=((y/40)%2===0?0:20); x<W; x+=40){ ctx.fillRect(x,y,10,10); }
  }
  const trailW = W/6;
  const lx = lanesX();
  for (let i=0;i<3;i++){
    ctx.fillStyle = '#7a5b45';
    ctx.fillRect(lx[i]-trailW/2, 0, trailW, H);
    ctx.fillStyle = 'rgba(0,0,0,0.18)';
    ctx.fillRect(lx[i]-1, 0, 2, H);
  }
}
function drawTree(x,y,w,h){
  ctx.fillStyle = '#6d3f17'; ctx.fillRect(x - w*0.12, y + h*0.1, w*0.24, h*0.45);
  ctx.fillStyle = '#1b5e20'; ctx.beginPath(); ctx.moveTo(x, y - h*0.45); ctx.lineTo(x - w*0.65, y + h*0.25); ctx.lineTo(x + w*0.65, y + h*0.25); ctx.closePath(); ctx.fill();
  ctx.fillStyle = '#2e7d32'; ctx.beginPath(); ctx.moveTo(x, y - h*0.25); ctx.lineTo(x - w*0.5, y + h*0.4); ctx.lineTo(x + w*0.5, y + h*0.4); ctx.closePath(); ctx.fill();
}
function drawMud(x, y, w, h){
  ctx.fillStyle = '#5b3a29';
  ctx.beginPath(); ctx.ellipse(x, y, w*0.5, h*0.5, 0, 0, Math.PI*2); ctx.fill();
  ctx.fillStyle = '#3e2723';
  ctx.beginPath(); ctx.ellipse(x - w*0.2, y - h*0.1, w*0.15, h*0.15, 0, 0, Math.PI*2); ctx.fill();
}

// Shared fish helper so shapes always match
function drawFishShape(x, y, scale, bodyColor, withGlow){
  const tail = 10 * scale;
  const tailH = 6 * scale;
  const bodyR = 6 * scale;
  const eyeR  = 1.5 * scale;

  if (withGlow){
    const t = performance.now() * 0.005;
    const pulse = 0.5 + 0.5 * Math.sin(t);
    ctx.save();
    ctx.globalAlpha = 0.35 + 0.45 * pulse;
    const g = ctx.createRadialGradient(x + bodyR, y, 0, x + bodyR, y, 20 * scale + 6 * pulse);
    g.addColorStop(0, 'rgba(255,215,0,0.95)');
    g.addColorStop(1, 'rgba(255,215,0,0)');
    ctx.fillStyle = g;
    ctx.beginPath();
    ctx.arc(x + bodyR, y, 20 * scale + 6 * pulse, 0, Math.PI * 2);
    ctx.fill();
    ctx.restore();
  }

  ctx.fillStyle = bodyColor;
  ctx.beginPath();
  ctx.moveTo(x, y);
  ctx.lineTo(x - tail, y - tailH);
  ctx.lineTo(x - tail, y + tailH);
  ctx.closePath();
  ctx.fill();

  ctx.beginPath();
  ctx.arc(x + bodyR, y, bodyR, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#000';
  ctx.beginPath();
  ctx.arc(x + bodyR + 2 * scale, y - 1 * scale, eyeR, 0, Math.PI * 2);
  ctx.fill();
}

function drawFish(x,y){ drawFishShape(x, y, 1, 'orange', false); }
function drawGoldenFish(x,y){ drawFishShape(x, y, 2, 'gold', true); }

function drawCat(x, y, w, h){
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
}

// Spawning and placement rules
/* Tighter vertical gap for more action */
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
let lastFishLane = null;

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

// lane wide check so fish never mix with trees or mud
function laneHasAnyEnemy(laneIndex){
  const lx = lanesX();
  const laneX = lx[laneIndex];
  return enemies.some(e => e.x === laneX);
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
  const withoutLast = candidateLanes.filter(l => l !== lastFishLane);
  if (withoutLast.length) lane = withoutLast[Math.floor(Math.random()*withoutLast.length)];
  else lane = candidateLanes[Math.floor(Math.random()*candidateLanes.length)];
  lastFishLane = lane;

  const goldenChance = 0.25;
  const golden = Math.random() < goldenChance;

  const scale  = golden ? 2 : 1;
  const w = 30 * scale;
  const h = 16 * scale;

  pickups.push({x: lx[lane], y: spawnY, w, h, golden});
}

// Sizes and player position
const CAT_W = 20, CAT_H = 30;
/* Lower the cat by the same shared offset, with a safety cap */
const PLAYER_Y = () => Math.min(H - 190, H * 0.64 + CAT_AND_BUTTON_OFFSET);

// Update and draw
function update(dt){
  const accel = baseAccel * (1 - Math.min(1, roadSpeed / maxSpeed));
  roadSpeed = Math.min(maxSpeed, (roadSpeed + accel) * Math.pow(1.00000002, meters));
  if (slipTimer > 0){
    slipTimer = Math.max(0, slipTimer - dt);
    slipOffset = Math.sin(performance.now()/40) * 4;
  } else slipOffset = 0;

  /* Faster cadence for spawns */
  spawnTimer += dt;
  if (spawnTimer > 0.6){
    spawnTimer = 0;
    if (Math.random() < 0.82) spawnEnemy(); else spawnPickup();
    if (Math.random() < 0.20) spawnPickup();
  }

  enemies.forEach(e => e.y += roadSpeed*dt);
  pickups.forEach(p => p.y += roadSpeed*dt);
  enemies = enemies.filter(e => e.y < H + 60);
  pickups = pickups.filter(p => p.y < H + 60);

  const px = lanesX()[currentLane] + slipOffset, py = PLAYER_Y(), pw = CAT_W-4, ph = CAT_H-2;
  for (let i=0; i<enemies.length; i++){
    const e = enemies[i];
    if (Math.abs(e.x-px) < (e.w+pw)/2 && Math.abs(e.y-py) < (e.h+ph)/2){
      if (e.type === 'mud'){
        enemies.splice(i,1);
        fuel = Math.max(0, fuel - 10);
        score = Math.max(0, score - 2);
        slipTimer = 0.6;
      } else endGame();
      break;
    }
  }
  for (let i=0; i<pickups.length; i++){
    const p = pickups[i];
    if (Math.abs(p.x-px) < (p.w+pw)/2 && Math.abs(p.y-py) < (p.h+ph)/2){
      pickups.splice(i,1);
      if (p.golden){
        fuel = Math.min(100, fuel + 20);
        score += 10;
      } else {
        fuel = Math.min(100, fuel + 10);
        score += 3;
      }
      break;
    }
  }
  meters += (roadSpeed * dt) / 120;
  fuel -= dt*2;
  if (fuel <= 0) endGame();
}
function drawHUD(){
  ctx.fillStyle = '#fff'; ctx.font = '16px system-ui, sans-serif';
  ctx.fillText('Score: '  + score, 10, 22);
  ctx.fillText('Energy: ' + Math.round(fuel), 10, 42);
  ctx.fillText('Meters: ' + Math.round(meters), 10, 62);
}
function draw(){
  drawBackground();
  enemies.forEach(e => { if (e.type==='tree') drawTree(e.x,e.y,e.w,e.h); else drawMud(e.x,e.y,e.w,e.h); });
  pickups.forEach(p => p.golden ? drawGoldenFish(p.x,p.y) : drawFish(p.x,p.y));
  drawCat(lanesX()[currentLane] + slipOffset, PLAYER_Y(), CAT_W, CAT_H);
  drawHUD();
}
function drawMenu(){
  drawBackground();
  ctx.fillStyle = '#fff';

  /* Smaller title block to free play space */
  ctx.font = '16px system-ui, sans-serif';
  const title = 'Cat Dash';
  ctx.fillText(title, (W - ctx.measureText(title).width)/2, 56);

  ctx.font = '14px system-ui, sans-serif';
  const sub = 'Tap, swipe, or use arrows to start';
  ctx.fillText(sub, (W - ctx.measureText(sub).width)/2, 78);

  const lines = ['Aim: Reach a high score', 'by collecting as many', 'fish as you can'];
  lines.forEach((line,i)=> ctx.fillText(line, (W - ctx.measureText(line).width)/2, 98 + i*16));

  drawBoard(24, 160);
}

// Control
function endGame(){
  gameRunning = false;
  maybeRecordScore(score);
}
function resetGame(){
  enemies = [];
  pickups = [];
  score = 0;
  fuel = 100;
  meters = 0;
  roadSpeed = 320;
  currentLane = 1;
}
function loop(ts){
  const dt = (ts - last) / 1000;
  last = ts || 0;
  ctx.clearRect(0,0,W,H);
  if (gameRunning){ update(dt); draw(); } else { drawMenu(); }
  requestAnimationFrame(loop);
}

// Keyboard
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

// Touch on canvas
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

// Lane buttons
const btn1 = document.getElementById('btnLane1');
const btn3 = document.getElementById('btnLane3');

function nudgeLeft(){ if (!gameRunning){ resetGame(); gameRunning = true; } if (currentLane > 0) currentLane--; }
function nudgeRight(){ if (!gameRunning){ resetGame(); gameRunning = true; } if (currentLane < 2) currentLane++; }

['click','touchstart'].forEach(evt=>{
  btn1.addEventListener(evt, e=>{ e.preventDefault(); nudgeLeft(); }, {passive: false});
  btn3.addEventListener(evt, e=>{ e.preventDefault(); nudgeRight(); }, {passive: false});
});

// Tap anywhere
canvas.addEventListener('click', e=>{
  if (!gameRunning){ resetGame(); gameRunning = true; return; }
  const x = e.clientX;
  const center = lanesX()[currentLane];
  if (x < center) nudgeLeft(); else if (x > center) nudgeRight();
});

// Start
requestAnimationFrame(loop);
</script>
</body>
</html>
