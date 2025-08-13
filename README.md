<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<title>Cat Dash — Mobile</title>
<style>
  :root { color-scheme: dark; }
  html, body { margin: 0; height: 100%; background:#222; color:#fff; font-family: system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial; }
  #wrap { position: fixed; inset: 0; display: grid; place-items: center; padding-top: env(safe-area-inset-top); padding-bottom: env(safe-area-inset-bottom); }
  canvas { background:#5e9d45; display:block; border-radius:16px; box-shadow:0 10px 30px rgba(0,0,0,.35); touch-action:none; }
</style>
</head>
<body>
<div id="wrap">
  <canvas id="game" width="400" height="600"></canvas>
</div>
<script>
/* ================= Canvas fit ================= */
const cvs = document.getElementById('game');
const ctx = cvs.getContext('2d');
const W = cvs.width, H = cvs.height;
function fit(){
  const s = Math.min(window.innerWidth / W, window.innerHeight / H);
  cvs.style.width  = (W*s) + 'px';
  cvs.style.height = (H*s) + 'px';
}
addEventListener('resize', fit, {passive:true});
fit();

/* ================= Game state ================= */
let gameRunning = false;
const lanesX = [W/4, W/2, 3*W/4];
let currentLane = 1;
let enemies = [], pickups = [];
let score = 0, fuel = 100, meters = 0;
let spawnTimer = 0, last = 0;

// Speed curve
let roadSpeed = 320, maxSpeed = 1000; const baseAccel = 0.6;

// Slip wobble when hitting mud (visual only)
let slipTimer = 0, slipOffset = 0;

/* ================= Leaderboard (localStorage) ================= */
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
  const qualifies = board.length < 10 || sc > board[board.length-1].score;
  if (!qualifies || sc <= 0) return;
  let name = prompt('New high score! Enter name/initials (3–10):','CAT');
  if (!name) name = 'CAT';
  addScore(name.trim().slice(0,10), sc);
}
function drawBoard(x, y){
  const board = loadBoard();
  ctx.fillStyle = '#fff';
  ctx.font = '18px system-ui, sans-serif';
  ctx.fillText('Leaderboard (Top 10)', x, y);
  ctx.font = '14px ui-monospace, SFMono-Regular, Menlo, monospace';
  if (board.length === 0){ ctx.fillText('No scores yet — be the first!', x, y+22); return; }
  for (let i=0; i<board.length && i<10; i++){
    const e = board[i];
    const line = `${String(i+1).padStart(2,' ')}. ${e.name.padEnd(10,' ')}  ${String(e.score).padStart(4,' ')}  ${e.date}`;
    ctx.fillText(line, x, y + 22 + i*18);
  }
}

/* ================= Art ================= */
function drawBackground(){
  // grass texture
  ctx.fillStyle = '#5e9d45'; ctx.fillRect(0,0,W,H);
  ctx.fillStyle = 'rgba(40,90,40,0.15)';
  for (let y=0; y<H; y+=40){
    for (let x=((y/40)%2===0?0:20); x<W; x+=40){ ctx.fillRect(x,y,10,10); }
  }
  // three mud trails
  const trailW = W/6;
  for (let i=0;i<3;i++){
    ctx.fillStyle = '#7a5b45';
    ctx.fillRect(lanesX[i]-trailW/2, 0, trailW, H);
    ctx.fillStyle = 'rgba(0,0,0,0.18)';
    ctx.fillRect(lanesX[i]-1, 0, 2, H);
  }
}
function drawTree(x,y,w,h){
  ctx.fillStyle = '#6d3f17'; ctx.fillRect(x - w*0.12, y + h*0.1, w*0.24, h*0.45);
  ctx.fillStyle = '#1b5e20'; ctx.beginPath(); ctx.moveTo(x, y - h*0.45); ctx.lineTo(x - w*0.65, y + h*0.25); ctx.lineTo(x + w*0.65, y + h*0.25); ctx.closePath(); ctx.fill();
  ctx.fillStyle = '#2e7d32'; ctx.beginPath(); ctx.moveTo(x, y - h*0.25); ctx.lineTo(x - w*0.5, y + h*0.4); ctx.lineTo(x + w*0.5, y + h*0.4); ctx.closePath(); ctx.fill();
}
function drawMud(x, y, w, h){
  // Mud patch (penalty)
  ctx.fillStyle = '#5b3a29';
  ctx.beginPath(); ctx.ellipse(x, y, w*0.5, h*0.5, 0, 0, Math.PI*2); ctx.fill();
  ctx.fillStyle = '#3e2723';
  ctx.beginPath(); ctx.ellipse(x - w*0.2, y - h*0.1, w*0.15, h*0.15, 0, 0, Math.PI*2); ctx.fill();
}
function drawFish(x,y){
  // bright orange fish for visibility (food=energy)
  ctx.fillStyle = 'orange';
  ctx.beginPath(); ctx.moveTo(x, y); ctx.lineTo(x-10, y-6); ctx.lineTo(x-10, y+6); ctx.closePath(); ctx.fill();
  ctx.beginPath(); ctx.arc(x+6, y, 6, 0, Math.PI*2); ctx.fill();
  ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(x+8, y-1, 1.5, 0, Math.PI*2); ctx.fill();
}
function drawGoldenFish(x,y){
  // rare golden fish
  ctx.fillStyle = 'gold';
  ctx.beginPath(); ctx.moveTo(x, y); ctx.lineTo(x-12, y-7); ctx.lineTo(x-12, y+7); ctx.closePath(); ctx.fill();
  ctx.beginPath(); ctx.arc(x+7, y, 7, 0, Math.PI*2); ctx.fill();
  ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(x+9, y-1, 1.6, 0, Math.PI*2); ctx.fill();
}
function drawCat(x, y, w, h){
  // rounder, two-tone cat with tail
  // body base
  ctx.fillStyle = '#d2691e';
  ctx.beginPath(); ctx.ellipse(x, y, w/2, h/2, 0, 0, Math.PI*2); ctx.fill();
  // inner patch
  ctx.fillStyle = '#a0522d';
  ctx.beginPath(); ctx.ellipse(x, y, w/2.5, h/2.5, 0, 0, Math.PI*2); ctx.fill();
  // head
  const headR = h*0.25, hx=x, hy=y - h*0.55;
  ctx.fillStyle = '#d2691e'; ctx.beginPath(); ctx.arc(hx,hy,headR,0,Math.PI*2); ctx.fill();
  ctx.fillStyle = '#a0522d'; ctx.beginPath(); ctx.arc(hx,hy,headR*0.75,0,Math.PI*2); ctx.fill();
  // ears
  ctx.fillStyle = '#d2691e';
  ctx.beginPath(); ctx.moveTo(hx-headR*0.8,hy-headR*0.2); ctx.lineTo(hx-headR*0.3,hy-headR*1.1); ctx.lineTo(hx-headR*0.05,hy-headR*0.2); ctx.closePath(); ctx.fill();
  ctx.beginPath(); ctx.moveTo(hx+headR*0.8,hy-headR*0.2); ctx.lineTo(hx+headR*0.3,hy-headR*1.1); ctx.lineTo(hx+headR*0.05,hy-headR*0.2); ctx.closePath(); ctx.fill();
  // eyes
  ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(hx-headR*0.4, hy, headR*0.15, 0, Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.arc(hx+headR*0.4, hy, headR*0.15, 0, Math.PI*2); ctx.fill();
  // tail (to the right)
  ctx.fillStyle = '#d2691e'; ctx.beginPath(); ctx.ellipse(x + w/2.2, y, w/6, h/3, 0, 0, Math.PI*2); ctx.fill();
}

/* ================= Spawning ================= */
const SPAWN_BUFFER_Y = 120; // avoid spawn overlap
function laneIsFree(x, y){
  return !enemies.some(e => Math.abs(e.x - x) < 1 && Math.abs(e.y - y) < SPAWN_BUFFER_Y)
      && !pickups.some(p => Math.abs(p.x - x) < 1 && Math.abs(p.y - y) < SPAWN_BUFFER_Y);
}
function pickFreeLane(spawnY){
  const candidates = [0,1,2].filter(l => laneIsFree(lanesX[l], spawnY));
  if (candidates.length === 0) return null;
  return candidates[Math.floor(Math.random()*candidates.length)];
}
function spawnEnemy(){
  const lane = pickFreeLane(-60);
  if (lane == null) return;
  const type = Math.random() < 0.55 ? 'tree' : 'mud';
  enemies.push(type==='tree' ? {type, x: lanesX[lane], y: -50, w: 40, h: 70}
                             : {type, x: lanesX[lane], y: -40, w: 56, h: 24});
}
function spawnPickup(){
  const lane = pickFreeLane(-50);
  if (lane == null) return;
  // ~1 in 15 chance of golden fish
  const golden = Math.random() < (1/15);
  pickups.push({x: lanesX[lane], y: -40, w: 30, h: 16, golden});
}

/* ================= Update & Draw ================= */
const PLAYER_Y = H - 110; // raised to avoid bottom bar
const CAT_W = 40, CAT_H = 60;

function update(dt){
  // speed ramp
  const accel = baseAccel * (1 - Math.min(1, roadSpeed / maxSpeed));
  roadSpeed = Math.min(maxSpeed, (roadSpeed + accel) * Math.pow(1.00000002, meters));

  // slip wobble timer
  if (slipTimer > 0){ slipTimer = Math.max(0, slipTimer - dt); slipOffset = Math.sin(performance.now()/40)*4; } else { slipOffset = 0; }

  // spawns  — increase pickups ~15% overall
  spawnTimer += dt;
  if (spawnTimer > 0.95){
    spawnTimer = 0;
    // Previously ~22% pickups; now ~25% (+~15% relative)
    if (Math.random() < 0.75) spawnEnemy(); else spawnPickup();
    // small extra chance of an additional pickup for more fish variety
    if (Math.random() < 0.10) spawnPickup();
  }

  // movement
  enemies.forEach(e=> e.y += roadSpeed*dt);
  pickups.forEach(p=> p.y += roadSpeed*dt);
  enemies = enemies.filter(e=> e.y < H+60);
  pickups = pickups.filter(p=> p.y < H+60);

  // collisions
  const px = lanesX[currentLane] + slipOffset, py = PLAYER_Y, pw = CAT_W-4, ph = CAT_H-2;
  for (let i=0;i<enemies.length;i++){
    const e = enemies[i];
    if (Math.abs(e.x-px) < (e.w+pw)/2 && Math.abs(e.y-py) < (e.h+ph)/2){
      if (e.type==='mud'){
        enemies.splice(i,1);
        fuel = Math.max(0, fuel - 10); // lose energy
        score = Math.max(0, score - 2); // small score penalty
        slipTimer = 0.6; // brief wobble; no global slow-down
      } else { endGame(); }
      break;
    }
  }
  for (let i=0;i<pickups.length;i++){
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

  // distance: 60px ≈ 0.5 m → px / 120
  meters += (roadSpeed * dt) / 120;
  fuel -= dt*2; if (fuel <= 0) endGame();
}

function drawHUD(){
  ctx.fillStyle = '#fff'; ctx.font = '16px system-ui, sans-serif';
  ctx.fillText('Score: '+score, 10, 22);
  ctx.fillText('Energy: '+Math.round(fuel), 10, 42);
  ctx.fillText('Meters: '+Math.round(meters), 10, 62);
}

function draw(){
  drawBackground();
  enemies.forEach(e=>{ if (e.type==='tree') drawTree(e.x,e.y,e.w,e.h); else drawMud(e.x,e.y,e.w,e.h); });
  pickups.forEach(p=> p.golden ? drawGoldenFish(p.x,p.y) : drawFish(p.x,p.y));
  drawCat(lanesX[currentLane] + slipOffset, PLAYER_Y, CAT_W, CAT_H);
  drawHUD();
}

function drawMenu(){
  drawBackground();
  ctx.fillStyle = '#fff'; ctx.font = '28px system-ui, sans-serif';
  const title = 'Cat Dash'; const w = ctx.measureText(title).width;
  ctx.fillText(title, (W - w)/2, 80);
  ctx.font = '18px system-ui, sans-serif';
  const sub = 'Tap or swipe to start';
  ctx.fillText(sub, (W - ctx.measureText(sub).width)/2, 120);
  ctx.font = '16px system-ui, sans-serif';
  const lines = ['Aim: Reach a high score', 'by collecting as many', 'fish as you can'];
  lines.forEach((line,i)=> ctx.fillText(line, (W - ctx.measureText(line).width)/2, 150 + i*18));
  // show leaderboard on menu
  drawBoard(24, 210);
}

function loop(ts){
  if(!last) last = ts; const dt = Math.min((ts - last)/1000, 0.05); last = ts;
  if (gameRunning){ update(dt); draw(); } else { drawMenu(); }
  requestAnimationFrame(loop);
}

function startGame(){
  enemies.length = 0; pickups.length = 0;
  score = 0; fuel = 100; meters = 0; roadSpeed = 320; slipTimer = 0;
  gameRunning = true; last = 0; spawnTimer = 0;
  // seed a couple of obstacles quickly
  for (let i=0;i<3;i++){ spawnEnemy(); enemies[i].y -= i*120; }
}
function endGame(){ gameRunning = false; lastEndTime = performance.now(); maybeRecordScore(score); }

/* ================= Touch controls ================= */
let touchStartX = null, touchStartY = null, touchStartTime = 0;
let didSwipe = false; let lastEndTime = 0;
const SWIPE_THRESHOLD = 50, TAP_MAX_MS = 500, RESTART_COOLDOWN_MS = 400;

cvs.addEventListener('touchstart', e=>{
  e.preventDefault();
  const t = e.touches[0];
  touchStartX = t.clientX; touchStartY = t.clientY; touchStartTime = performance.now();
  didSwipe = false;
}, {passive:false});

cvs.addEventListener('touchmove', e=>{
  e.preventDefault();
  if (touchStartX == null) return;
  const t = e.touches[0]; const dx = t.clientX - touchStartX; const dy = t.clientY - touchStartY;
  if (Math.abs(dx) > Math.abs(dy) && Math.abs(dx) > SWIPE_THRESHOLD){
    if (dx > 0 && currentLane < 2) currentLane++;
    else if (dx < 0 && currentLane > 0) currentLane--;
    didSwipe = true; touchStartX = null; touchStartY = null; // lock until next touch
  }
}, {passive:false});

cvs.addEventListener('touchend', e=>{
  e.preventDefault();
  const now = performance.now();
  const isTap = !didSwipe && (now - touchStartTime) < TAP_MAX_MS;
  if (!gameRunning && isTap && (now - lastEndTime > RESTART_COOLDOWN_MS)) startGame();
  touchStartX = null; touchStartY = null; didSwipe = false;
}, {passive:false});

// boot
requestAnimationFrame(loop);
</script>
</body>
</html>
