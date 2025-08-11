<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<title>Cat Dash — Mobile</title>
<style>
  body { margin:0; background:#333; color:#fff; font-family:sans-serif; text-align:center; }
  canvas { display:block; margin:0 auto; background:#6dbb4a; touch-action: none; }
</style>
</head>
<body>
<canvas id="gameCanvas" width="400" height="600"></canvas>
<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;

// ===== State =====
let gameRunning = false;          // playing vs menu
let lanesX = [W/4, W/2, 3*W/4];
let currentLane = 1;
let enemies = [], pickups = [];
let score = 0, fuel = 100, meters = 0;
let spawnTimer = 0;
let last = 0;

// Speed & progression
let roadSpeed = 320;
let maxSpeed = 1000;
const baseAccel = 0.6;

// Tap/Swipe handling (fix accidental restarts)
let touchStartX = null, touchStartY = null, touchStartTime = 0;
let didSwipe = false;                   // set true once a swipe is recognized
let lastEndTime = 0;                    // debounce after game over
const SWIPE_THRESHOLD = 50;             // px
const TAP_MAX_MOVE = 12;                // px
const RESTART_COOLDOWN_MS = 400;        // ms

// ===== Art =====
function drawBackground(){
  ctx.fillStyle = '#5e9d45'; ctx.fillRect(0,0,W,H);
  const trailWidth = W/6;
  for(let i=0;i<3;i++){
    ctx.fillStyle = '#8b5a2b';
    ctx.fillRect(lanesX[i] - trailWidth/2, 0, trailWidth, H);
    ctx.fillStyle = 'rgba(0,0,0,0.12)';
    ctx.fillRect(lanesX[i]-1, 0, 2, H);
  }
}
function drawTree(x,y,w,h){
  // trunk
  ctx.fillStyle = '#6d3f17';
  ctx.fillRect(x - w*0.12, y + h*0.1, w*0.24, h*0.45);
  // canopy (darker green for contrast)
  ctx.fillStyle = '#1b5e20';
  ctx.beginPath(); ctx.moveTo(x, y - h*0.45); ctx.lineTo(x - w*0.65, y + h*0.25); ctx.lineTo(x + w*0.65, y + h*0.25); ctx.closePath(); ctx.fill();
  // mid canopy
  ctx.fillStyle = '#2e7d32';
  ctx.beginPath(); ctx.moveTo(x, y - h*0.25); ctx.lineTo(x - w*0.5, y + h*0.4); ctx.lineTo(x + w*0.5, y + h*0.4); ctx.closePath(); ctx.fill();
}
function drawPuddle(x,y,w,h){
  ctx.fillStyle = '#3aa3ff';
  ctx.beginPath(); ctx.ellipse(x, y, w*0.5, h*0.5, 0, 0, Math.PI*2); ctx.fill();
  ctx.globalAlpha=0.25; ctx.fillStyle='#fff'; ctx.beginPath(); ctx.ellipse(x+w*0.15, y-h*0.1, w*0.18, h*0.14, 0, 0, Math.PI*2); ctx.fill(); ctx.globalAlpha=1;
}
function drawFish(x,y){
  // bright orange fish for visibility
  ctx.fillStyle = 'orange';
  ctx.beginPath(); ctx.moveTo(x, y); ctx.lineTo(x-10, y-6); ctx.lineTo(x-10, y+6); ctx.closePath(); ctx.fill();
  ctx.beginPath(); ctx.arc(x+6, y, 6, 0, Math.PI*2); ctx.fill();
  ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(x+8, y-1, 1.5, 0, Math.PI*2); ctx.fill();
}
function drawCat(x, y, w, h){
  ctx.fillStyle = '#f7c67f'; ctx.strokeStyle = '#000'; ctx.lineWidth = 2;
  const r = Math.min(w,h)*0.25;
  // capsule body
  ctx.beginPath();
  ctx.moveTo(x - w/2 + r, y - h/2);
  ctx.lineTo(x + w/2 - r, y - h/2);
  ctx.quadraticCurveTo(x + w/2, y - h/2, x + w/2, y - h/2 + r);
  ctx.lineTo(x + w/2, y + h/2 - r);
  ctx.quadraticCurveTo(x + w/2, y + h/2, x + w/2 - r, y + h/2);
  ctx.lineTo(x - w/2 + r, y + h/2);
  ctx.quadraticCurveTo(x - w/2, y + h/2, x - w/2, y + h/2 - r);
  ctx.lineTo(x - w/2, y - h/2 + r);
  ctx.quadraticCurveTo(x - w/2, y - h/2, x - w/2 + r, y - h/2);
  ctx.closePath(); ctx.fill(); ctx.stroke();
  // head
  const headR = h*0.28, hx=x, hy=y-h*0.4;
  ctx.beginPath(); ctx.arc(hx,hy,headR,0,Math.PI*2); ctx.fill(); ctx.stroke();
  // ears
  ctx.beginPath(); ctx.moveTo(hx-headR*0.8,hy-headR*0.2); ctx.lineTo(hx-headR*0.3,hy-headR*1.1); ctx.lineTo(hx-headR*0.05,hy-headR*0.2); ctx.closePath(); ctx.fill(); ctx.stroke();
  ctx.beginPath(); ctx.moveTo(hx+headR*0.8,hy-headR*0.2); ctx.lineTo(hx+headR*0.3,hy-headR*1.1); ctx.lineTo(hx+headR*0.05,hy-headR*0.2); ctx.closePath(); ctx.fill(); ctx.stroke();
  // eyes + nose
  ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(hx-headR*0.4, hy-headR*0.1, headR*0.15, 0, Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.arc(hx+headR*0.4, hy-headR*0.1, headR*0.15, 0, Math.PI*2); ctx.fill();
  ctx.fillStyle = '#ff99aa'; ctx.beginPath(); ctx.arc(hx, hy+headR*0.1, headR*0.1, 0, Math.PI*2); ctx.fill();
}

// ===== Spawning =====
function spawnEnemy(){
  const lane = Math.floor(Math.random()*lanesX.length);
  const type = Math.random()<0.55 ? 'tree' : 'puddle';
  enemies.push(type==='tree' ? {type, x: lanesX[lane], y: -50, w: 40, h: 70}
                             : {type, x: lanesX[lane], y: -40, w: 56, h: 24});
}
function spawnPickup(){
  const lane = Math.floor(Math.random()*lanesX.length);
  pickups.push({x: lanesX[lane], y: -40, w: 30, h: 16});
}

// ===== Game Loop =====
function update(dt){
  // speed ramp
  roadSpeed = Math.min(maxSpeed, (roadSpeed + baseAccel) * Math.pow(1.00000002, meters));

  // spawns
  spawnTimer += dt;
  if (spawnTimer > 1){ spawnTimer = 0; (Math.random() < 0.78 ? spawnEnemy : spawnPickup)(); }

  // movement
  enemies.forEach(e=> e.y += roadSpeed*dt);
  pickups.forEach(p=> p.y += roadSpeed*dt);
  enemies = enemies.filter(e=> e.y < H+60);
  pickups = pickups.filter(p=> p.y < H+60);

  // collisions
  const px = lanesX[currentLane], py = H - 75, pw = 36, ph = 58;
  for (let i=0;i<enemies.length;i++){
    const e = enemies[i];
    if (Math.abs(e.x-px) < (e.w+pw)/2 && Math.abs(e.y-py) < (e.h+ph)/2){
      if (e.type==='puddle'){
        enemies.splice(i,1); fuel = Math.max(0, fuel - 10); score = Math.max(0, score - 2);
      } else { endGame(); }
      break;
    }
  }
  for (let i=0;i<pickups.length;i++){
    const p = pickups[i];
    if (Math.abs(p.x-px) < (p.w+pw)/2 && Math.abs(p.y-py) < (p.h+ph)/2){ pickups.splice(i,1); fuel = Math.min(100, fuel + 10); score += 3; break; }
  }

  // meters: 1 cat length (~60px) = 0.5 m ⇒ px / 120
  meters += (roadSpeed * dt) / 120;
  fuel -= dt*2; if (fuel <= 0) endGame();
}

function drawHUD(){
  ctx.fillStyle = '#fff'; ctx.font = '16px sans-serif';
  ctx.fillText('Score: '+score, 10, 20);
  ctx.fillText('Fish: '+Math.round(fuel), 10, 40);
  ctx.fillText('Meters: '+Math.round(meters), 10, 60);
}

function draw(){
  drawBackground();
  enemies.forEach(e=>{ if (e.type==='tree') drawTree(e.x,e.y,e.w,e.h); else drawPuddle(e.x,e.y,e.w,e.h); });
  pickups.forEach(p=> drawFish(p.x,p.y));
  drawCat(lanesX[currentLane], H-60, 40, 60);
  drawHUD();
}

function drawMenu(){
  drawBackground();
  ctx.fillStyle = '#fff'; ctx.font = '28px sans-serif';
  const title = 'Cat Dash';
  ctx.fillText(title, W/2 - ctx.measureText(title).width/2, 80);
  ctx.font = '18px sans-serif';
  ctx.fillText('Tap or swipe to start', W/2 - 90, 120);
}

function loop(ts){
  if(!last) last = ts; const dt = (ts - last)/1000; last = ts;
  if (gameRunning){ update(dt); draw(); }
  else { drawMenu(); }
  requestAnimationFrame(loop);
}

function startGame(){
  enemies.length = 0; pickups.length = 0;
  score = 0; fuel = 100; meters = 0; roadSpeed = 320;
  gameRunning = true; last = 0; spawnTimer = 0;
  // Seed a few obstacles so trees appear immediately
  for (let i=0;i<3;i++){ spawnEnemy(); enemies[i].y -= i*120; }
}
function endGame(){
  gameRunning = false;
  lastEndTime = performance.now();
}

// ===== Touch controls (debounced start) =====
canvas.addEventListener('touchstart', e=>{
  e.preventDefault();
  const t = e.touches[0];
  touchStartX = t.clientX; touchStartY = t.clientY; touchStartTime = performance.now();
  didSwipe = false;
}, {passive:false});

canvas.addEventListener('touchmove', e=>{
  e.preventDefault();
  if (touchStartX === null) return;
  const dx = e.touches[0].clientX - touchStartX;
  const dy = e.touches[0].clientY - touchStartY;
  if (Math.abs(dx) > Math.abs(dy) && Math.abs(dx) > SWIPE_THRESHOLD){
    // horizontal swipe → lane change
    if (dx > 0 && currentLane < 2) currentLane++;
    else if (dx < 0 && currentLane > 0) currentLane--;
    didSwipe = true;
    // lock until next touch to avoid multi-lane jumps
    touchStartX = null; touchStartY = null;
  }
}, {passive:false});

canvas.addEventListener('touchend', e=>{
  e.preventDefault();
  const now = performance.now();
  // Only start from menu on a true tap (small move) and after cooldown
  if (!gameRunning){
    const movedTooMuch = (touchStartX!==null && touchStartY!==null) ? false : didSwipe; // if swipe detected, don't treat as tap
    if (!movedTooMuch && (now - touchStartTime < 500) && (now - lastEndTime > RESTART_COOLDOWN_MS)){
      startGame();
    }
  }
  touchStartX = null; touchStartY = null; didSwipe = false;
}, {passive:false});

requestAnimationFrame(loop);
</script>
</body>
</html>
