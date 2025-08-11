<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<title>Cat Dash</title>
<style>
  :root { color-scheme: dark; }
  html, body { height: 100%; margin: 0; background:#222; font-family: system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial; color:#fff; }
  /* Reserve space for iPhone bars */
  body { padding-bottom: env(safe-area-inset-bottom); padding-top: env(safe-area-inset-top); }
  #wrap { position: fixed; inset: 0; display: grid; place-items: center; }
  canvas { background:#5e9d45; display:block; border-radius:16px; box-shadow:0 10px 30px rgba(0,0,0,.35); touch-action:none; }
</style>
</head>
<body>
<div id="wrap">
  <canvas id="game" width="400" height="600"></canvas>
</div>

<script>
const cvs = document.getElementById('game');
const ctx = cvs.getContext('2d');
const W = cvs.width, H = cvs.height;

/* Fit canvas to screen while preserving 400x600 aspect */
function fit(){
  const s = Math.min(window.innerWidth / W, window.innerHeight / H);
  cvs.style.width  = (W * s) + 'px';
  cvs.style.height = (H * s) + 'px';
}
addEventListener('resize', fit, {passive:true}); fit();

/* --- Game state --- */
let gameRunning = false;
const lanesX = [W/4, W/2, 3*W/4];
let currentLane = 1;
let enemies = [], pickups = [];
let score = 0, fuel = 100, meters = 0;
let spawnTimer = 0, last = 0;

/* Speed */
let roadSpeed = 320, maxSpeed = 1000; const baseAccel = 0.6;

/* Player position (raised so it’s clear of bottom bar) */
const PLAYER_Y = H - 110;   // was H - 60
const CAT_W = 40, CAT_H = 60;

/* Touch: start only on true tap; swipes change lanes */
let touchStartX = null, touchStartY = null, touchStartTime = 0;
let didSwipe = false; let lastEndTime = 0;
const SWIPE_THRESHOLD = 50, TAP_MAX_MS = 500, RESTART_COOLDOWN_MS = 400;

/* --- Drawing helpers --- */
function drawBackground(){
  ctx.fillStyle = '#5e9d45'; ctx.fillRect(0,0,W,H);
  // trails
  const trailW = W/6;
  for (let i=0;i<3;i++){
    ctx.fillStyle = '#7a5b45';
    ctx.fillRect(lanesX[i]-trailW/2, 0, trailW, H);
    ctx.fillStyle = 'rgba(0,0,0,.18)';
    ctx.fillRect(lanesX[i]-1, 0, 2, H);
  }
}
function drawTree(x,y,w,h){
  ctx.fillStyle = '#6d3f17'; ctx.fillRect(x - w*0.12, y + h*0.1, w*0.24, h*0.45);
  ctx.fillStyle = '#1b5e20'; ctx.beginPath(); ctx.moveTo(x, y - h*0.45); ctx.lineTo(x - w*0.65, y + h*0.25); ctx.lineTo(x + w*0.65, y + h*0.25); ctx.closePath(); ctx.fill();
  ctx.fillStyle = '#2e7d32'; ctx.beginPath(); ctx.moveTo(x, y - h*0.25); ctx.lineTo(x - w*0.5, y + h*0.4); ctx.lineTo(x + w*0.5, y + h*0.4); ctx.closePath(); ctx.fill();
}
function drawPuddle(x,y,w,h){
  ctx.fillStyle = '#3aa3ff'; ctx.beginPath(); ctx.ellipse(x,y,w*0.5,h*0.5,0,0,Math.PI*2); ctx.fill();
  ctx.globalAlpha=.25; ctx.fillStyle='#fff'; ctx.beginPath(); ctx.ellipse(x+w*0.15,y-h*0.1,w*0.18,h*0.14,0,0,Math.PI*2); ctx.fill(); ctx.globalAlpha=1;
}
function drawFish(x,y){
  ctx.fillStyle = 'orange';
  ctx.beginPath(); ctx.moveTo(x, y); ctx.lineTo(x-10, y-6); ctx.lineTo(x-10, y+6); ctx.closePath(); ctx.fill();
  ctx.beginPath(); ctx.arc(x+6, y, 6, 0, Math.PI*2); ctx.fill();
  ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(x+8, y-1, 1.5, 0, Math.PI*2); ctx.fill();
}
function drawCat(x, y, w, h){
  ctx.fillStyle = '#f7c67f'; ctx.strokeStyle = '#000'; ctx.lineWidth = 2;
  const r = Math.min(w,h)*0.25;
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
  const headR = h*0.28, hx=x, hy=y-h*0.4;
  ctx.beginPath(); ctx.arc(hx,hy,headR,0,Math.PI*2); ctx.fill(); ctx.stroke();
  ctx.beginPath(); ctx.moveTo(hx-headR*0.8,hy-headR*0.2); ctx.lineTo(hx-headR*0.3,hy-headR*1.1); ctx.lineTo(hx-headR*0.05,hy-headR*0.2); ctx.closePath(); ctx.fill(); ctx.stroke();
  ctx.beginPath(); ctx.moveTo(hx+headR*0.8,hy-headR*0.2); ctx.lineTo(hx+headR*0.3,hy-headR*1.1); ctx.lineTo(hx+headR*0.05,hy-headR*0.2); ctx.closePath(); ctx.fill(); ctx.stroke();
  ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(hx-headR*0.4, hy-headR*0.1, headR*0.15, 0, Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.arc(hx+headR*0.4, hy-headR*0.1, headR*0.15, 0, Math.PI*2); ctx.fill();
  ctx.fillStyle = '#ff99aa'; ctx.beginPath(); ctx.arc(hx, hy+headR*0.1, headR*0.1, 0, Math.PI*2); ctx.fill();
}

/* --- Spawning --- */
function spawnEnemy(){
  const lane = (Math.random()*3|0);
  const type = Math.random()<0.55 ? 'tree' : 'puddle';
  enemies.push(type==='tree' ? {type, x: lanesX[lane], y: -50, w: 40, h: 70}
                             : {type, x: lanesX[lane], y: -40, w: 56, h: 24});
}
function spawnPickup(){
  const lane = (Math.random()*3|0);
  pickups.push({x: lanesX[lane], y: -40, w: 30, h: 16});
}

/* --- Loop --- */
function update(dt){
  roadSpeed = Math.min(maxSpeed, (roadSpeed + baseAccel) * Math.pow(1.00000002, meters));

  spawnTimer += dt;
  if (spawnTimer > 1){ spawnTimer = 0; (Math.random() < 0.78 ? spawnEnemy : spawnPickup)(); }

  enemies.forEach(e=> e.y += roadSpeed*dt);
  pickups.forEach(p=> p.y += roadSpeed*dt);
  enemies = enemies.filter(e=> e.y < H+60);
  pickups = pickups.filter(p=> p.y < H+60);

  const px = lanesX[currentLane], py = PLAYER_Y, pw = CAT_W-4, ph = CAT_H-2;

  // obstacles
  for (let i=0;i<enemies.length;i++){
    const e = enemies[i];
    if (Math.abs(e.x-px) < (e.w+pw)/2 && Math.abs(e.y-py) < (e.h+ph)/2){
      if (e.type==='puddle'){
        enemies.splice(i,1); fuel = Math.max(0, fuel - 10); score = Math.max(0, score - 2);
      } else {
        endGame();
      }
      break;
    }
  }
  // pickups
  for (let i=0;i<pickups.length;i++){
    const p = pickups[i];
    if (Math.abs(p.x-px) < (p.w+pw)/2 && Math.abs(p.y-py) < (p.h+ph)/2){
      pickups.splice(i,1); fuel = Math.min(100, fuel + 10); score += 3; break;
    }
  }

  // distance: 60px ≈ 0.5 m → px / 120
  meters += (roadSpeed * dt) / 120;
  fuel -= dt*2; if (fuel <= 0) endGame();
}
function drawHUD(){
  ctx.fillStyle = '#fff'; ctx.font = '16px system-ui, sans-serif';
  ctx.fillText('Score: '+score, 10, 22);
  ctx.fillText('Fish: '+Math.round(fuel), 10, 42);
  ctx.fillText('Meters: '+Math.round(meters), 10, 62);
}
function draw(){
  drawBackground();
  enemies.forEach(e=> (e.type==='tree' ? drawTree(e.x,e.y,e.w,e.h) : drawPuddle(e.x,e.y,e.w,e.h)));
  pickups.forEach(p=> drawFish(p.x,p.y));
  drawCat(lanesX[currentLane], PLAYER_Y, CAT_W, CAT_H);
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
}
function loop(ts){
  if(!last) last = ts; const dt = Math.min((ts - last)/1000, 0.05); last = ts;
  if (gameRunning){ update(dt); draw(); } else { drawMenu(); }
  requestAnimationFrame(loop);
}

/* --- Start/End --- */
function startGame(){
  enemies.length = 0; pickups.length = 0;
  score = 0; fuel = 100; meters = 0; roadSpeed = 320;
  gameRunning = true; last = 0; spawnTimer = 0;
  // seed a couple of obstacles so something appears quickly
  for (let i=0;i<3;i++){ spawnEnemy(); enemies[i].y -= i*120; }
}
function endGame(){ gameRunning = false; lastEndTime = performance.now(); }

/* --- Touch: start on true tap; swipe to switch lanes --- */
canvas.addEventListener('touchstart', e=>{
  e.preventDefault();
  const t = e.touches[0];
  touchStartX = t.clientX; touchStartY = t.clientY; touchStartTime = performance.now();
  didSwipe = false;
},{passive:false});

canvas.addEventListener('touchmove', e=>{
  e.preventDefault();
  if (touchStartX == null) return;
  const t = e.touches[0];
  const dx = t.clientX - touchStartX;
  const dy = t.clientY - touchStartY;
  if (Math.abs(dx) > Math.abs(dy) && Math.abs(dx) > SWIPE_THRESHOLD){
    if (dx > 0 && currentLane < 2) currentLane++;
    else if (dx < 0 && currentLane > 0) currentLane--;
    didSwipe = true; touchStartX = null; touchStartY = null;
  }
},{passive:false});

canvas.addEventListener('touchend', e=>{
  e.preventDefault();
  const now = performance.now();
  const isTap = !didSwipe && (now - touchStartTime) < TAP_MAX_MS;
  if (!gameRunning && isTap && (now - lastEndTime > RESTART_COOLDOWN_MS)) startGame();
  touchStartX = null; touchStartY = null; didSwipe = false;
},{passive:false});

/* Boot */
requestAnimationFrame(loop);
</script>
</body>
</html>
