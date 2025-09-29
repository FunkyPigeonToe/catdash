<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<title></title>
<style>
  :root{
    --pink:#ff4081;
    --pink-hi:#ff6fa3;
    --indigo:#5865F2;
    --bg:#12161c;
    --panel: rgba(0,0,0,0.35);
  }
  html, body {
    margin: 0; padding: 0; background: var(--bg); color: #fff;
    font-family: system-ui, -apple-system, Segoe UI, Roboto, "Helvetica Neue", Arial, "Noto Sans", sans-serif;
    height: 100dvh; overflow: hidden;
  }
  canvas { display: block; margin: 0 auto; background: #6dbb4a; touch-action: none;
    backface-visibility: hidden; -webkit-backface-visibility: hidden; transform: translateZ(0); will-change: transform; }

  /* --- Front & Menu Buttons (centered, neat spacing) --- */
  .stack { position: fixed; display: none; z-index: 9999; left: 50%; transform: translateX(-50%); pointer-events: auto; width: min(92vw, 520px); }
  #frontStack { top: 62%; }
  #menuStack { top: 78%; }
  .stack .buttons { display: flex; flex-direction: column; gap: 12px; align-items: center; }
  .btn { border: 0; border-radius: 14px; padding: 14px 22px; font-size: 18px; color: #fff; box-shadow: 0 8px 22px rgba(0,0,0,0.35); -webkit-tap-highlight-color: transparent; }
  .btn:active { transform: translateY(1px) scale(0.98); }
  .btn-primary { background: linear-gradient(180deg, var(--pink-hi) 0%, var(--pink) 100%); font-weight: 700; min-width: 240px; }
  .btn-indigo  { background: var(--indigo); min-width: 240px; }
  #fsNote { font-size: 12px; opacity: 0.85; text-align: center; margin-top: 4px; }

  /* --- On-screen arrows (only shown in game) --- */
  .controls { position: fixed; inset: 0; pointer-events: none; z-index: 10; display: none; }
  .laneBtn { position: absolute; width: 88px; height: 88px; border: none; border-radius: 20px; background: rgba(0,0,0,0.35); color: #fff; font-size: 28px; box-shadow: 0 6px 18px rgba(0,0,0,0.35); -webkit-tap-highlight-color: transparent; touch-action: manipulation; pointer-events: auto; transform: translate(-50%, -50%); }
  .laneBtn:active { transform: translate(-50%, -50%) scale(0.96); }

  /* Hide arrows on larger screens */
  @media (min-width: 900px) { .controls { display: none !important; } }
</style>
</head>
<body>
<canvas id="gameCanvas"></canvas>

<!-- FRONT (centered actions) -->
<div id="frontStack" class="stack">
  <div class="buttons">
    <button id="playBtn" class="btn btn-primary" type="button">Play</button>
    <button id="frontChangeNameBtn" class="btn btn-indigo" type="button">Change Name</button>
    <div id="fsNote">Tip: Play tries to go fullscreen</div>
  </div>
</div>

<!-- MENU (below leaderboard area) -->
<div id="menuStack" class="stack">
  <div class="buttons">
    <button id="startBtn" class="btn btn-primary" type="button">Start Game</button>
    <!-- Name change is front-only now to keep menu clear of leaderboard -->
  </div>
</div>

<!-- In-game lane arrows -->
<div id="controls" class="controls">
  <button id="btnLane1" class="laneBtn" aria-label="Move left">◀</button>
  <button id="btnLane3" class="laneBtn" aria-label="Move right">▶</button>
</div>

<!-- Supabase client (v2) -->
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>

<script>
/* =========================
   Supabase (Global Leaderboard)
   ========================= */
const SUPABASE_URL = 'https://fvcvrhaxxpsientgggnx.supabase.co';
const SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImZ2Y3ZyaGF4eHBzaWVudGdnZ254Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTYwODczMzYsImV4cCI6MjA3MTY2MzMzNn0.5wTxwGVJDa3gnS61gaDq00xSFGUMEQ0Pda6tJo4VK-A';
const TABLE_NAME = 'highscores'; // name (PK), score, updated_at
const supa = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

const NAME_KEY = 'catdash_name';
function getPlayerName(){
  let n = localStorage.getItem(NAME_KEY);
  if (!n){
    n = prompt('Enter player name (3–12 chars):', 'CAT') || 'CAT';
    n = n.trim().slice(0,12);
    if (n.length < 3) n = (n + 'CAT').slice(0,3);
    localStorage.setItem(NAME_KEY, n);
  }
  return n;
}

let globalBoard = [];   // [{name, score, updated_at}]
let lastBoardFetch = 0;

async function fetchGlobalTop(limit=20){
  try{
    const { data, error } = await supa
      .from(TABLE_NAME)
      .select('name,score,updated_at')
      .order('score', { ascending: false })
      .limit(limit);
    if (error) throw error;
    globalBoard = Array.isArray(data) ? data : [];
    lastBoardFetch = performance.now();
  }catch(err){ console.warn('Leaderboard fetch error:', err.message||err); }
}
async function submitBestIfHigher(name, score){
  try{
    const { data: existing, error: e1 } = await supa
      .from(TABLE_NAME).select('score').eq('name', name).single();
    const prev = Number(existing?.score ?? 0);
    if (!e1 && score <= prev) return false;

    const payload = { name, score, updated_at: new Date().toISOString() };
    const { error: e2 } = await supa
      .from(TABLE_NAME).upsert(payload, { onConflict: 'name' });
    if (e2) throw e2;

    fetchGlobalTop(20);
    return true;
  }catch(err){ console.warn('Submit best error:', err.message||err); return false; }
}

/* =========================
   Canvas & Viewport
   ========================= */
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d', { alpha:false, desynchronized:true });

function getViewportSize(){
  const vv = window.visualViewport;
  if (vv) return { w: Math.floor(vv.width), h: Math.floor(vv.height) };
  return { w: Math.floor(window.innerWidth), h: Math.floor(window.innerHeight) };
}
let { w: W, h: H } = getViewportSize();
let DPR = 1;

function resizeCanvas(){
  DPR = Math.max(1, Math.min(3, window.devicePixelRatio || 1));
  canvas.style.width = W + 'px';
  canvas.style.height = H + 'px';
  canvas.width = Math.floor(W * DPR);
  canvas.height = Math.floor(H * DPR);
  ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
  ctx.imageSmoothingEnabled = true;
  ctx.imageSmoothingQuality = 'high';
}
resizeCanvas();

function handleResize(){
  const v = getViewportSize(); W = v.w; H = v.h; resizeCanvas(); positionButtons(); initFlowerSpots(); }
window.addEventListener('resize', handleResize, { passive: true });
if (window.visualViewport){
  window.visualViewport.addEventListener('resize', handleResize, { passive: true });
  window.visualViewport.addEventListener('scroll', handleResize, { passive: true });
}
function lanesX(){ return [W/4, W/2, (3*W)/4]; }

/* =========================
   Controls: on-screen arrows (only in-game)
   ========================= */
const controls = document.getElementById('controls');
const CAT_Y_LIFT = -30;         // lift cat up a little (negative lifts)
const BUTTON_Y_OFFSET = 50;     // keep buttons comfortably above bottom
function positionButtons(){
  const btn1 = document.getElementById('btnLane1');
  const btn3 = document.getElementById('btnLane3');
  const lx = lanesX();
  const baseY = H - Math.min(120, H * 0.12);
  const y = Math.min(H - 10, baseY + BUTTON_Y_OFFSET);
  btn1.style.left = lx[0] + 'px'; btn1.style.top  = y + 'px';
  btn3.style.left = lx[2] + 'px'; btn3.style.top  = y + 'px';
}
positionButtons();

/* =========================
   Game State + Modes (front/menu/game)
   ========================= */
let mode = 'front'; // 'front' | 'menu' | 'game'
let currentLane = 1; let enemies = []; let pickups = []; let particles = [];
let score = 0, fuel = 100, meters = 0;
let spawnTimer = 0; let last = undefined; let graceTimer = 0.75; let restartDelay = 0;
let roadSpeed = 226, maxSpeed = 704; const baseAccel = 0.6; let slipTimer = 0, slipOffset = 0;
let btn1Down = false, btn3Down = false; let cheatArmTimerMs = 0, cheatCharges = 0, cheatRearmLock = false;
const CHEAT_HOLD_TIME_MS = 50; let cheatToastTimer = 0, cheatToastText = '';
let dashActive = false, dashTimer = 0; const DASH_DURATION = 8.0, DASH_SPEED_MUL = 1.30, DASH_SCORE_MUL = 2;

/* Flowers (colour shifts every 400 m) */
const FLOWER_SEG_METERS = 400;
const FLOWER_COLORS = ['#ffec99','#ffd6e7','#c0ebff','#c3fda7','#ffd8a8','#eebefa','#b2f2bb'];
const flowerSpots = [];
function initFlowerSpots(){
  flowerSpots.length = 0;
  const count = Math.max(80, Math.floor(W*H/11000));
  for (let i=0;i<count;i++){
    flowerSpots.push({ x: Math.random()*W, y: Math.random()*H, r: 1.2 + Math.random()*1.4, rot: Math.random()*Math.PI*2, stem: Math.random()<0.8 });
  }
}
initFlowerSpots();

/* =========================
   Visual helpers
   ========================= */
function withShadow(color='rgba(0,0,0,0.35)', blur=8, offsetY=3, drawFn){ ctx.save(); ctx.shadowColor=color; ctx.shadowBlur=blur; ctx.shadowOffsetX=0; ctx.shadowOffsetY=offsetY; drawFn(); ctx.restore(); }
function strokeAround(strokeStyle='rgba(0,0,0,0.35)', lineWidth=2, drawPathFn){ ctx.save(); ctx.lineWidth=lineWidth; ctx.strokeStyle=strokeStyle; drawPathFn(); ctx.stroke(); ctx.restore(); }

/* =========================
   Background + Flowers + Trails (game scenes)
   ========================= */
function drawBloom(x, y, size, color, rot){
  ctx.save(); ctx.translate(x,y); ctx.rotate(rot);
  const petalR = size*2.1, centerR = size*1.1; ctx.fillStyle = color;
  for (let i=0;i<5;i++){ const ang = (i/5)*Math.PI*2; ctx.beginPath(); ctx.ellipse(Math.cos(ang)*size*1.1, Math.sin(ang)*size*1.1, petalR*0.55, petalR*0.35, ang, 0, Math.PI*2); ctx.fill(); }
  const g = ctx.createRadialGradient(0,0,0,0,0,centerR); g.addColorStop(0,'rgba(255,255,220,0.95)'); g.addColorStop(1,'rgba(255,255,220,0.2)');
  ctx.fillStyle = g; ctx.beginPath(); ctx.arc(0,0,centerR,0,Math.PI*2); ctx.fill(); ctx.restore();
}
function drawBackground(){
  const g = ctx.createLinearGradient(0, 0, 0, H); g.addColorStop(0, '#64b24a'); g.addColorStop(1, '#4d9c3b'); ctx.fillStyle = g; ctx.fillRect(0,0,W,H);
  ctx.fillStyle = 'rgba(40,90,40,0.10)';
  for (let y=0; y<H; y+=40){ for (let x=((y/40)%2===0?0:20); x<W; x+=40){ ctx.fillRect(x,y,10,10); } }
  const seg = Math.floor(meters / FLOWER_SEG_METERS) % FLOWER_COLORS.length; const color = FLOWER_COLORS[seg];
  const lx = lanesX(), trailW = W/6;
  flowerSpots.forEach(f=>{ const inLane = (Math.abs(f.x - lx[0]) < trailW/2) || (Math.abs(f.x - lx[1]) < trailW/2) || (Math.abs(f.x - lx[2]) < trailW/2); if (inLane) return; if (f.stem){ ctx.strokeStyle = 'rgba(20,80,20,0.6)'; ctx.lineWidth = 1; ctx.beginPath(); ctx.moveTo(f.x, f.y+4); ctx.lineTo(f.x, f.y+8); ctx.stroke(); } drawBloom(f.x, f.y, f.r, color, f.rot); });
  for (let i=0;i<3;i++){ const rg = ctx.createLinearGradient(0, 0, 0, H); rg.addColorStop(0, '#8b684f'); rg.addColorStop(1, '#6f523f'); ctx.fillStyle = rg; ctx.fillRect(lx[i]-trailW/2, 0, trailW, H); ctx.fillStyle = 'rgba(0,0,0,0.18)'; ctx.fillRect(lx[i]-1, 0, 2, H); }
}

/* =========================
   Trees & Mud (game)
   ========================= */
function drawTree(x,y,w,h){ withShadow('rgba(0,0,0,0.35)', 10, 4, ()=>{ const trunkW = w*0.28, trunkH = h*0.48; const trunkX = x - trunkW/2, trunkY = y + h*0.12; ctx.fillStyle = '#6d3f17'; ctx.beginPath(); const r = trunkW*0.35; ctx.moveTo(trunkX + r, trunkY); ctx.lineTo(trunkX + trunkW - r, trunkY); ctx.quadraticCurveTo(trunkX + trunkW, trunkY, trunkX + trunkW, trunkY + r); ctx.lineTo(trunkX + trunkW, trunkY + trunkH - r); ctx.quadraticCurveTo(trunkX + trunkW, trunkY + trunkH, trunkX + trunkW - r, trunkY + trunkH); ctx.lineTo(trunkX + r, trunkY + trunkH); ctx.quadraticCurveTo(trunkX, trunkY + trunkH, trunkX, trunkY + trunkH - r); ctx.lineTo(trunkX, trunkY + r); ctx.quadraticCurveTo(trunkX, trunkY, trunkX + r, trunkY); ctx.closePath(); ctx.fill(); const cx = x, cy = y - h*0.06; const cMain = '#2e7d32', cMid = '#2f8c34', cLight = '#399c3a'; ctx.fillStyle = cMid; ctx.beginPath(); ctx.arc(cx - w*0.28, cy + h*0.02, h*0.25, 0, Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(cx + w*0.28, cy + h*0.02, h*0.25, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = cMain; ctx.beginPath(); ctx.arc(cx, cy, h*0.32, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = cLight; ctx.beginPath(); ctx.arc(cx, cy - h*0.22, h*0.18, 0, Math.PI*2); ctx.fill(); ctx.strokeStyle = 'rgba(0,0,0,0.35)'; ctx.lineWidth = 1.5; ctx.beginPath(); ctx.arc(cx, cy, h*0.32, 0, Math.PI*2); ctx.stroke(); }); }
function drawMud(x, y, w, h){ withShadow('rgba(0,0,0,0.3)', 8, 3, ()=>{ const g = ctx.createRadialGradient(x, y, 2, x, y, Math.max(w,h)); g.addColorStop(0, '#6a4a3a'); g.addColorStop(1, '#3e2723'); ctx.fillStyle = g; ctx.beginPath(); ctx.ellipse(x, y, w*0.5, h*0.5, 0, 0, Math.PI*2); ctx.fill(); }); strokeAround('rgba(0,0,0,0.35)', 1.2, ()=>{ ctx.beginPath(); ctx.ellipse(x, y, w*0.5, h*0.5, 0, 0, Math.PI*2); }); }

/* =========================
   Pickups (mouse, bird, lizard, chicken, dash⚡)
   ========================= */
function drawAdditiveGlow(x, y, radius, centerAlpha=0.9){ ctx.save(); ctx.globalCompositeOperation = 'lighter'; const g = ctx.createRadialGradient(x, y, 0, x, y, radius); g.addColorStop(0, `rgba(255,215,0,${centerAlpha})`); g.addColorStop(1, 'rgba(255,215,0,0)'); ctx.fillStyle = g; ctx.beginPath(); ctx.arc(x, y, radius, 0, Math.PI*2); ctx.fill(); ctx.restore(); }
function drawMouse(x, y, scale, golden){ const t = performance.now()*0.006, wiggle = Math.sin(t + x*0.01)*1.2*scale; const body = golden ? '#ffd54f' : '#c7a17a'; const ear   = golden ? '#ffe082' : '#d7b894'; withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{ ctx.fillStyle = body; ctx.beginPath(); ctx.ellipse(x, y+wiggle, 12*scale, 7*scale, 0, 0, Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.ellipse(x+9*scale, y-1*scale+wiggle, 6*scale, 5*scale, 0, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = ear; ctx.beginPath(); ctx.arc(x+12*scale, y-5*scale+wiggle, 2.8*scale, 0, Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(x+7.5*scale, y-6*scale+wiggle, 2.2*scale, 0, Math.PI*2); ctx.fill(); ctx.strokeStyle = golden ? '#ffe082' : '#b78963'; ctx.lineWidth = 1.4*scale; ctx.beginPath(); ctx.moveTo(x-12*scale, y+2*scale+wiggle); ctx.quadraticCurveTo(x-18*scale, y+6*scale+wiggle, x-22*scale, y+3*scale+wiggle); ctx.stroke(); ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(x+11*scale, y-2*scale+wiggle, 1.4*scale, 0, Math.PI*2); ctx.fill(); }); if (golden) drawAdditiveGlow(x, y, 22*scale, 0.85); }
function drawBird(x, y, scale, golden){ const t = performance.now()*0.004, bob = Math.sin(t + x*0.02)*1.2*scale; const body = golden ? '#ffe066' : '#66a9ff'; withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{ ctx.fillStyle = body; ctx.beginPath(); ctx.ellipse(x, y+bob, 10*scale, 7*scale, 0, 0, Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(x+7*scale, y-3*scale+bob, 4*scale, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = golden ? '#ffd54f' : '#4f94f5'; ctx.beginPath(); ctx.ellipse(x-3*scale, y+1*scale+bob, 6*scale, 4*scale, -0.7, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = golden ? '#ffca28' : '#ffb300'; ctx.beginPath(); ctx.moveTo(x+11*scale, y-3*scale+bob); ctx.lineTo(x+15*scale, y-1*scale+bob); ctx.lineTo(x+11*scale, y-1*scale+bob); ctx.closePath(); ctx.fill(); ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(x+6*scale, y-4*scale+bob, 1.2*scale, 0, Math.PI*2); ctx.fill(); }); if (golden) drawAdditiveGlow(x, y, 20*scale, 0.85); }
function drawLizard(x, y, scale, golden){ const t = performance.now()*0.006, sway = Math.sin(t + x*0.03)*1.5*scale; const body = golden ? '#ffd54f' : '#5cb85c'; const belly = golden ? '#ffe082' : '#4cae4c'; withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{ ctx.fillStyle = body; ctx.beginPath(); ctx.ellipse(x, y+sway, 16*scale, 6*scale, 0, 0, Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.moveTo(x-16*scale, y+sway); ctx.quadraticCurveTo(x-26*scale, y+3*scale+sway, x-30*scale, y+sway); ctx.lineTo(x-24*scale, y-2*scale+sway); ctx.closePath(); ctx.fill(); ctx.beginPath(); ctx.ellipse(x+14*scale, y-1*scale+sway, 6*scale, 5*scale, 0, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = belly; ctx.fillRect(x-6*scale, y-2*scale+sway, 12*scale, 4*scale); ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(x+17*scale, y-2*scale+sway, 1.4*scale, 0, Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(x+12*scale, y-2*scale+sway, 1.4*scale, 0, Math.PI*2); ctx.fill(); }); if (golden) drawAdditiveGlow(x, y, 24*scale, 0.85); }
function drawChicken(x, y, scale){ const bob = Math.sin(performance.now()*0.004 + x*0.01) * 1.0 * scale; withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{ ctx.fillStyle = '#ffe082'; ctx.beginPath(); ctx.ellipse(x, y+bob, 12*scale, 9*scale, 0, 0, Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(x+9*scale, y-5*scale+bob, 5*scale, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = '#ffb300'; ctx.beginPath(); ctx.moveTo(x+14*scale, y-5*scale+bob); ctx.lineTo(x+18*scale, y-4*scale+bob); ctx.lineTo(x+14*scale, y-2.5*scale+bob); ctx.closePath(); ctx.fill(); ctx.fillStyle = '#e53935'; ctx.beginPath(); ctx.arc(x+8*scale, y-9*scale+bob, 2.1*scale, 0, Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(x+10.8*scale, y-9.4*scale+bob, 1.8*scale, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = '#ffd54f'; ctx.beginPath(); ctx.ellipse(x-3*scale, y-1*scale+bob, 7*scale, 5*scale, -0.7, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(x+8*scale, y-6*scale+bob, 1.4*scale, 0, Math.PI*2); ctx.fill(); ctx.strokeStyle = '#ffb300'; ctx.lineWidth = 1.3*scale; ctx.beginPath(); ctx.moveTo(x-2*scale, y+9*scale+bob); ctx.lineTo(x-2*scale, y+12*scale+bob); ctx.moveTo(x+1*scale, y+9*scale+bob); ctx.lineTo(x+1*scale, y+12*scale+bob); ctx.stroke(); }); drawAdditiveGlow(x, y, 24*scale, 0.9); }
function drawLightning(x, y, scale=1){ ctx.save(); ctx.translate(x, y); ctx.scale(scale, scale); ctx.fillStyle = 'gold'; ctx.beginPath(); ctx.moveTo(0, -15); ctx.lineTo(8, 0); ctx.lineTo(2, 0); ctx.lineTo(12, 18); ctx.lineTo(-2, 2); ctx.lineTo(4, 2); ctx.closePath(); ctx.fill(); ctx.lineWidth = 2.2; ctx.strokeStyle = '#000'; ctx.stroke(); ctx.shadowColor = 'rgba(255, 215, 0, 0.7)'; ctx.shadowBlur = 12; ctx.fillStyle = 'gold'; ctx.fill(); ctx.restore(); }

/* =========================
   Spawning & placement (game)
   ========================= */
const SPAWN_BUFFER_Y = 70;
function laneIsFree(x, y){ return !enemies.some(e => e.x === x && Math.abs(e.y - y) < SPAWN_BUFFER_Y) && !pickups.some(p => p.x === x && Math.abs(p.y - y) < SPAWN_BUFFER_Y); }
function pickFreeLane(spawnY){ const lx = lanesX(); const candidates = [0,1,2].filter(i => laneIsFree(lx[i], spawnY)); if (!candidates.length) return null; return candidates[Math.floor(Math.random()*candidates.length)]; }
let lastPickupLane = null;
function spawnEnemy(){ const lane = pickFreeLane(-60); if (lane == null) return; const lx = lanesX(); const type = Math.random() < 0.55 ? 'tree' : 'mud'; enemies.push( type==='tree' ? {type, x: lx[lane], y: -50, w: 40, h: 70} : {type, x: lx[lane], y: -40, w: 56, h: 24} ); }
function laneHasAnyEnemy(laneIndex){ const lx = lanesX(); const x = lx[laneIndex]; return enemies.some(e => e.x === x); }
function spawnPickup(){ const spawnY = -40; const sameYEnemy  = enemies.some(e => Math.abs(e.y - spawnY) < SPAWN_BUFFER_Y); const sameYPickup = pickups.some(p => Math.abs(p.y - spawnY) < SPAWN_BUFFER_Y); if (sameYEnemy || sameYPickup) return; const lx = lanesX(); let candidateLanes = [0,1,2].filter(i => laneIsFree(lx[i], spawnY) && !laneHasAnyEnemy(i)); if (!candidateLanes.length) return; let lane; const withoutLast = candidateLanes.filter(l => l !== lastPickupLane); lane = (withoutLast.length ? withoutLast : candidateLanes)[Math.floor(Math.random()*(withoutLast.length?withoutLast.length:candidateLanes.length))]; lastPickupLane = lane; const r = Math.random(); let type, golden=false, scale=1; if (r < 0.06){ type='dash'; scale=1.2; } else if (r < 0.46){ type='mouse'; golden = Math.random()<0.25; scale = golden?1.2:1.0; } else if (r < 0.86){ type='bird'; golden = Math.random()<0.25; scale = golden?1.25:1.05; } else if (r < 0.97){ type='lizard'; golden = Math.random()<0.25; scale = golden?1.25:1.1; } else { type='chicken'; golden=true; scale=1.35; } const baseW = type==='bird' ? 30 : type==='mouse' ? 34 : type==='lizard' ? 36 : type==='chicken' ? 38 : 30; const baseH = type==='bird' ? 18 : type==='mouse' ? 18 : type==='lizard' ? 16 : type==='chicken' ? 22 : 30; const w = baseW * scale; const h = baseH * scale; pickups.push({type, x: lx[lane], y: spawnY, w, h, scale, golden}); }

/* =========================
   Cat (improved proportions & tail wag)
   ========================= */
const CAT_W = 22, CAT_H = 34;
function drawCat(x, y, w, h){ withShadow('rgba(0,0,0,0.35)', 12, 5, ()=>{ const bodyRx = w/1.55, bodyRy = h/1.12; const tailLen = h * 1.35; const tailBaseW = Math.max(3, w * 0.30); const t = performance.now() * 0.008; const wagAmp = h * 0.12; const baseX = x - bodyRx + tailBaseW * 0.8; const baseY = y + bodyRy * 0.60; ctx.strokeStyle = '#9a5a2a'; ctx.lineCap='round'; ctx.lineJoin='round'; const N=16; let prevX=baseX, prevY=baseY; for(let i=1;i<=N;i++){ const u=i/N, k=1-u; const px = baseX - u*tailLen*0.95; const py = baseY - u*tailLen*0.55 + Math.sin(t+u*7.0) * wagAmp * (0.25+0.75*k); ctx.lineWidth = Math.max(1, tailBaseW*(0.3+0.7*k)); ctx.beginPath(); ctx.moveTo(prevX,prevY); ctx.lineTo(px,py); ctx.stroke(); prevX=px; prevY=py; } ctx.fillStyle = '#d07a2b'; ctx.beginPath(); ctx.ellipse(x, y, bodyRx, bodyRy, 0, 0, Math.PI*2); ctx.fill(); const headR = h*0.36; const hx = x, hy = y - h*0.78; ctx.beginPath(); ctx.arc(hx,hy,headR,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.moveTo(hx-headR*0.6,hy-headR*0.15); ctx.lineTo(hx-headR*0.25,hy-headR*1.0); ctx.lineTo(hx,hy-headR*0.15); ctx.closePath(); ctx.fill(); ctx.beginPath(); ctx.moveTo(hx+headR*0.6,hy-headR*0.15); ctx.lineTo(hx+headR*0.25,hy-headR*1.0); ctx.lineTo(hx,hy-headR*0.15); ctx.closePath(); ctx.fill(); ctx.fillStyle = '#a85b24'; ctx.beginPath(); ctx.ellipse(x, y+2, w/2.6, h/2.6, 0, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(hx-headR*0.35, hy, headR*0.15, 0, Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(hx+headR*0.35, hy, headR*0.15, 0, Math.PI*2); ctx.fill(); ctx.strokeStyle = '#fff'; ctx.lineWidth = 1.2; ctx.beginPath(); ctx.moveTo(hx- headR*0.55, hy);   ctx.lineTo(hx- headR*1.1, hy-2); ctx.moveTo(hx- headR*0.55, hy+4); ctx.lineTo(hx- headR*1.1, hy+6); ctx.moveTo(hx- headR*0.55, hy-4); ctx.lineTo(hx- headR*1.1, hy-6); ctx.moveTo(hx+ headR*0.55, hy);   ctx.lineTo(hx+ headR*1.1, hy-2); ctx.moveTo(hx+ headR*0.55, hy+4); ctx.lineTo(hx+ headR*1.1, hy+6); ctx.moveTo(hx+ headR*0.55, hy-4); ctx.lineTo(hx+ headR*1.1, hy-6); ctx.stroke(); }); strokeAround('rgba(0,0,0,0.4)', 1, ()=>{ ctx.beginPath(); ctx.ellipse(x, y, w/1.55, h/1.12, 0, 0, Math.PI*2); }); }

/* =========================
   Particles (sparkles on pickup)
   ========================= */
function spawnSparkles(x, y, baseColor){ const n = 12; for (let i=0;i<n;i++){ const ang = (Math.PI*2) * (i/n) + Math.random()*0.4; const spd = 60 + Math.random()*110; particles.push({ x, y, vx: Math.cos(ang)*spd, vy: Math.sin(ang)*spd - 40, life: 0.6 + Math.random()*0.4, age: 0, color: baseColor }); } }
function updateParticles(dt){ for (let i=particles.length-1; i>=0; i--){ const p = particles[i]; p.age += dt; p.x += p.vx * dt; p.y += p.vy * dt; p.vy += 80 * dt; if (p.age >= p.life) particles.splice(i,1); } }
function drawParticles(){ particles.forEach(p=>{ const a = Math.max(0, 1 - p.age / p.life); ctx.save(); ctx.globalAlpha = a; ctx.fillStyle = p.color; ctx.beginPath(); ctx.arc(p.x, p.y, 2 + 1.2*a, 0, Math.PI*2); ctx.fill(); ctx.restore(); }); }

/* =========================
   HUD & overlays & board (game/menu/front use)
   ========================= */
function drawHUD(){ ctx.fillStyle = '#fff'; ctx.font = '16px system-ui, sans-serif'; const sTxt = 'Score: '  + (dashActive ? Math.round(score) + ' (x2)' : Math.round(score)); ctx.fillText(sTxt, 10, 22); ctx.fillText('Energy: ' + Math.round(fuel), 10, 42); ctx.fillText('Meters: ' + Math.round(meters), 10, 62); const txt = cheatCharges > 0 ? `Shield x${cheatCharges}` : ''; if (txt){ const wTxt = ctx.measureText(txt).width + 12; const x = W - wTxt - 10; ctx.fillText(txt, x, 28); } if (dashActive){ const tTxt = `DASH ${dashTimer.toFixed(1)}s`; const tw = ctx.measureText(tTxt).width; ctx.fillStyle = '#ffeb3b'; ctx.fillText(tTxt, (W - tw)/2, 22); } if (cheatToastTimer > 0){ const a = Math.min(1, cheatToastTimer / 0.3); ctx.save(); ctx.globalAlpha = a; ctx.font = '18px system-ui, sans-serif'; const t = cheatToastText || ''; const tw = ctx.measureText(t).width; const tx = (W - tw)/2; const ty = 44; ctx.fillStyle = 'rgba(0,0,0,0.35)'; ctx.fillRect(tx - 12, ty - 18, tw + 24, 28); ctx.fillStyle = '#ffd54f'; ctx.fillText(t, tx, ty); ctx.restore(); } }
function drawVignette(){ const g = ctx.createRadialGradient(W/2, H*0.58, Math.min(W,H)*0.25, W/2, H*0.58, Math.max(W,H)*0.75); g.addColorStop(0, 'rgba(0,0,0,0)'); g.addColorStop(1, 'rgba(0,0,0,0.35)'); ctx.fillStyle = g; ctx.fillRect(0,0,W,H); }
function drawDashOverlay(){ if (!dashActive) return; ctx.save(); ctx.globalAlpha = 0.18; ctx.fillStyle = '#ff9800'; ctx.fillRect(0,0,W,H); ctx.globalAlpha = 0.16; ctx.strokeStyle = '#ffffff'; ctx.lineWidth = 2; for (let i=0;i<6;i++){ const x = (i+0.5) * (W/6); ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x + (Math.random()*8-4), H); ctx.stroke(); } ctx.restore(); }
function drawGlobalBoard(x, y, maxRows=20){ ctx.fillStyle = '#fff'; ctx.font = '18px system-ui, sans-serif'; ctx.fillText('Global Top 20', x, y); ctx.font = '14px ui-monospace, SFMono-Regular, Menlo, monospace'; if (!globalBoard.length){ ctx.fillText('Loading...', x, y+22); if (performance.now() - lastBoardFetch > 5000) fetchGlobalTop(20); return; } const rows = Math.min(maxRows, globalBoard.length); for (let i=0; i<rows; i++){ const e = globalBoard[i]; const name = (e.name || '???').slice(0,12).padEnd(12, ' '); const line = `${String(i+1).padStart(2,' ')}. ${name}  ${String(Number(e.score||0)).padStart(6,' ')}  ${String((e.updated_at||'').slice(0,10))}`; ctx.fillText(line, x, y + 22 + i*18); } }

/* =========================
   BRAND-NEW FRONT PAGE (clean + fresh + cat animation)
   ========================= */
let frontT = 0;
const clouds = Array.from({length: 6}, ()=>({ x: Math.random(), y: Math.random()*0.25 + 0.05, s: Math.random()*0.5 + 0.7, v: Math.random()*0.015 + 0.01 }));
function drawFrontBackground(){ const sky = ctx.createLinearGradient(0,0,0,H); sky.addColorStop(0, '#6ec8ff'); sky.addColorStop(0.6, '#9be1ff'); sky.addColorStop(1, '#b7f0ff'); ctx.fillStyle = sky; ctx.fillRect(0,0,W,H); const sunR = Math.min(W,H)*0.08 + Math.sin(frontT*2)*3; const sunX = W*0.14, sunY = H*0.18; const g = ctx.createRadialGradient(sunX, sunY, 0, sunX, sunY, sunR*2.2); g.addColorStop(0,'rgba(255,255,160,1)'); g.addColorStop(1,'rgba(255,255,160,0)'); ctx.fillStyle = g; ctx.beginPath(); ctx.arc(sunX, sunY, sunR*2.2, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = '#fff8b0'; ctx.beginPath(); ctx.arc(sunX, sunY, sunR, 0, Math.PI*2); ctx.fill(); clouds.forEach(c=>{ c.x += c.v * 0.016; if (c.x > 1.2) c.x = -0.2; const cx = c.x * W, cy = c.y * H; ctx.save(); ctx.globalAlpha = 0.85; ctx.fillStyle = '#ffffff'; ctx.beginPath(); ctx.ellipse(cx, cy, 70*c.s, 40*c.s, 0, 0, Math.PI*2); ctx.ellipse(cx+50*c.s, cy+5*c.s, 60*c.s, 32*c.s, 0, 0, Math.PI*2); ctx.ellipse(cx-50*c.s, cy+8*c.s, 55*c.s, 30*c.s, 0, 0, Math.PI*2); ctx.fill(); ctx.restore(); }); function hill(yBase, amp, color, scale){ ctx.fillStyle = color; ctx.beginPath(); ctx.moveTo(0,H); for(let x=0; x<=W; x+=8){ const y = yBase + Math.sin((x/W)*Math.PI*2*scale + frontT*0.3)*amp; ctx.lineTo(x, y); } ctx.lineTo(W,H); ctx.closePath(); ctx.fill(); } hill(H*0.78, 14, '#6bbf6b', 1.1); hill(H*0.85, 20, '#46a445', 0.7); }
function drawFrontAnimation(){ const groundY = H*0.86; const trackL = W*0.18, trackR = W*0.82; const cycle = 5.5; const phase = (frontT % cycle) / cycle; const tri = phase < 0.5 ? (phase/0.5) : (1 - (phase-0.5)/0.5); const yarnX = trackL + (trackR - trackL) * tri; const yarnY = groundY - 10 + Math.sin(frontT*8)*1.2; const r = 16; withShadow('rgba(0,0,0,0.25)', 8, 2, ()=>{ ctx.fillStyle = '#ff6fa3'; ctx.beginPath(); ctx.arc(yarnX, yarnY, r, 0, Math.PI*2); ctx.fill(); ctx.strokeStyle = '#ff3f88'; ctx.lineWidth = 1.5; ctx.beginPath(); ctx.arc(yarnX, yarnY, r-3, 0.2+frontT*1.2, Math.PI*1.3+frontT*1.2); ctx.stroke(); ctx.beginPath(); ctx.arc(yarnX, yarnY, r-7, 0.8+frontT*1.2, Math.PI*1.8+frontT*1.2); ctx.stroke(); ctx.beginPath(); ctx.moveTo(yarnX+r-2, yarnY-2); ctx.quadraticCurveTo(yarnX+r+10, yarnY+3, yarnX+r+20, yarnY-4); ctx.stroke(); }); let catX = yarnX - 60, catY = groundY - 6 + Math.sin(frontT*3)*1.5; catX = Math.max(60, Math.min(W-60, catX)); const bat = Math.max(0, 1 - Math.abs(yarnX - (catX+54)) / 50); const pawLift = bat * 8 * Math.sin(frontT*12); withShadow('rgba(0,0,0,0.25)', 10, 3, ()=>{ ctx.strokeStyle = '#9a4c1f'; ctx.lineCap='round'; ctx.lineWidth = 6; ctx.beginPath(); ctx.moveTo(catX-24, catY+6); ctx.quadraticCurveTo(catX-44, catY-6, catX-30, catY-22); ctx.stroke(); ctx.fillStyle = '#d2691e'; ctx.beginPath(); ctx.ellipse(catX, catY, 26, 20, 0, 0, Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(catX+24, catY-20, 12, 0, Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.moveTo(catX+16, catY-24); ctx.lineTo(catX+22, catY-36); ctx.lineTo(catX+28, catY-24); ctx.closePath(); ctx.fill(); ctx.beginPath(); ctx.moveTo(catX+28, catY-24); ctx.lineTo(catX+34, catY-36); ctx.lineTo(catX+40, catY-24); ctx.closePath(); ctx.fill(); ctx.fillStyle = '#a0522d'; ctx.beginPath(); ctx.ellipse(catX, catY+2, 12, 10, 0, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(catX+20, catY-20, 2.6, 0, Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(catX+28, catY-20, 2.6, 0, Math.PI*2); ctx.fill(); ctx.fillStyle = '#d2691e'; ctx.beginPath(); ctx.ellipse(catX+54, catY-2 - pawLift, 8, 5, 0.2, 0, Math.PI*2); ctx.fill(); }); }
function drawFront(){ drawFrontBackground(); // No giant title — keeps top clean
  // Subtle instruction text below clouds
  ctx.save(); ctx.font = '16px system-ui, sans-serif'; ctx.fillStyle = 'rgba(255,255,255,0.9)'; const sub = 'Catch snacks • Avoid trees & mud • Tap Play'; const tw = ctx.measureText(sub).width; ctx.fillText(sub, (W - tw)/2, H*0.30); ctx.restore(); drawFrontAnimation(); // Leaderboard never collides with buttons
  const margin = 24; const baseY = H*0.50; drawGlobalBoard(margin, baseY, Math.min(10, Math.floor((H-baseY-40)/18)-1)); drawVignette(); }

/* =========================
   MENU (kept clear so the board is visible)
   ========================= */
function drawMenu(){ drawFrontBackground(); ctx.fillStyle = '#fff'; ctx.font = '14px system-ui, sans-serif'; const sub = 'Collect snacks • Avoid trees & mud • Grab ⚡ for Dash'; ctx.fillText(sub, (W - ctx.measureText(sub).width)/2, 70); drawGlobalBoard(24, 150, 14); drawVignette(); }

/* =========================
   Update & Draw (game)
   ========================= */
const PLAYER_Y = () => Math.min(H - 190, H * 0.64 + CAT_Y_LIFT);
function armCheatIfHeld(dt){ const bothDown = btn1Down && btn3Down; if (bothDown && !cheatRearmLock && cheatCharges === 0){ cheatArmTimerMs += dt * 1000; if (cheatArmTimerMs >= CHEAT_HOLD_TIME_MS){ cheatCharges = 2; cheatRearmLock = true; cheatToastText = 'Shield armed x2'; cheatToastTimer = 1.0; } } else { cheatArmTimerMs = 0; } if (!btn1Down && !btn3Down && cheatCharges === 0){ cheatRearmLock = false; } }
function startDash(){ if (dashActive) return; dashActive = true; dashTimer = DASH_DURATION; cheatToastText = 'DASH!'; cheatToastTimer = 1.2; }
function updateDash(dt){ if (!dashActive) return; dashTimer -= dt; if (dashTimer <= 0){ dashActive = false; dashTimer = 0; } }
function update(dt){ const accel = baseAccel * (1 - Math.min(1, roadSpeed / maxSpeed)); let speedMul = dashActive ? DASH_SPEED_MUL : 1; roadSpeed = Math.min(maxSpeed, (roadSpeed + accel) * Math.pow(1.00000002, meters)); const actualSpeed = roadSpeed * speedMul; if (slipTimer > 0){ slipTimer = Math.max(0, slipTimer - dt); slipOffset = Math.sin(performance.now()/40) * 4; } else slipOffset = 0; spawnTimer += dt; const spawnInterval = dashActive ? 0.5 : 0.6; if (spawnTimer > spawnInterval){ spawnTimer = 0; if (Math.random() < (dashActive ? 0.65 : 0.70)) spawnEnemy(); else spawnPickup(); if (Math.random() < (dashActive ? 0.50 : 0.35)) spawnPickup(); } enemies.forEach(e => e.y += actualSpeed*dt); pickups.forEach(p => p.y += actualSpeed*dt); enemies = enemies.filter(e => e.y < H + 60); pickups = pickups.filter(p => p.y < H + 60); const px = lanesX()[currentLane] + slipOffset; const py = PLAYER_Y(); const pw = CAT_W - 4, ph = CAT_H - 2; armCheatIfHeld(dt); const collisionsActive = graceTimer <= 0; if (collisionsActive){ for (let i=0; i<enemies.length; i++){ const e = enemies[i]; let ew = e.w, eh = e.h; if (e.type === 'tree'){ ew *= 0.6; eh *= 0.8; } if (Math.abs(e.x - px) < (ew + pw)/2 && Math.abs(e.y - py) < (eh + ph)/2){ if (e.type === 'mud'){ enemies.splice(i,1); fuel = Math.max(0, fuel - 10); score = Math.max(0, score - (dashActive?1:2)); slipTimer = 0.6; } else { if (cheatCharges > 0){ enemies.splice(i,1); cheatCharges--; } else endGame(); } break; } } }
  for (let i=0; i<pickups.length; i++){ const p = pickups[i]; if (Math.abs(p.x - px) < (p.w + pw)/2 && Math.abs(p.y - py) < (p.h + ph)/2){ pickups.splice(i,1); if (p.type === 'dash'){ startDash(); cheatCharges = Math.min(2, cheatCharges + 1); cheatToastText = 'DASH + Shield +1'; cheatToastTimer = 1.2; spawnSparkles(px, py, 'rgba(255,240,130,0.95)'); } else if (p.type === 'chicken'){ fuel = Math.min(100, fuel + 25); score += (dashActive ? 30 : 15); cheatCharges = Math.min(2, cheatCharges + 1); cheatToastText = 'Shield +1'; cheatToastTimer = 1.2; spawnSparkles(px, py, 'rgba(255,230,140,0.95)'); } else if (p.type === 'lizard'){ fuel = Math.min(100, fuel + (p.golden ? 22 : 12)); score += (p.golden ? 12 : 5) * (dashActive?DASH_SCORE_MUL:1); spawnSparkles(px, py, 'rgba(180,255,120,0.95)'); } else if (p.type === 'bird'){ fuel = Math.min(100, fuel + (p.golden ? 20 : 10)); score += (p.golden ? 10 : 3) * (dashActive?DASH_SCORE_MUL:1); spawnSparkles(px, py, 'rgba(180,210,255,0.95)'); } else { fuel = Math.min(100, fuel + (p.golden ? 20 : 10)); score += (p.golden ? 10 : 3) * (dashActive?DASH_SCORE_MUL:1); spawnSparkles(px, py, 'rgba(255,230,150,0.95)'); } break; } }
  updateDash(dt); updateParticles(dt); if (cheatToastTimer > 0) cheatToastTimer = Math.max(0, cheatToastTimer - dt); meters += (actualSpeed * dt) / 120; fuel -= dt * (dashActive ? 2.4 : 2.0); if (fuel <= 0) endGame(); }
function draw(){ if (mode === 'front'){ frontT += 0.016; drawFront(); return; } if (mode === 'menu'){ drawMenu(); return; } drawBackground(); enemies.forEach(e => { if (e.type==='tree') drawTree(e.x,e.y,e.w,e.h); else drawMud(e.x,e.y,e.w,e.h); }); pickups.forEach(p=>{ if (p.type==='mouse') drawMouse(p.x,p.y,p.scale,p.golden); else if (p.type==='bird') drawBird(p.x,p.y,p.scale,p.golden); else if (p.type==='lizard') drawLizard(p.x,p.y,p.scale,p.golden); else if (p.type==='chicken') drawChicken(p.x,p.y,p.scale); else if (p.type==='dash') drawLightning(p.x,p.y,p.scale); }); drawCat(lanesX()[currentLane] + slipOffset, PLAYER_Y(), CAT_W, CAT_H); drawParticles(); drawHUD(); drawDashOverlay(); drawVignette(); }

/* =========================
   Control / Flow
   ========================= */
function endGame(){ mode = 'menu'; const n = getPlayerName(); submitBestIfHigher(n, Math.round(score)); restartDelay = 1.0; }
function resetGame(){ enemies = []; pickups = []; particles = []; score = 0; fuel = 100; meters = 0; roadSpeed = 226; currentLane = 1; spawnTimer = 0; graceTimer = 0.75; last = undefined; cheatArmTimerMs = 0; cheatCharges = 0; cheatRearmLock = false; cheatToastTimer = 0; cheatToastText = ''; dashActive = false; dashTimer = 0; restartDelay = 0; initFlowerSpots(); }
async function goFullscreen(){ try { const el = document.documentElement; if (!document.fullscreenElement && el.requestFullscreen) { await el.requestFullscreen(); } } catch(e){} }
function startGame(){ resetGame(); mode = 'game'; }

/* Main loop */
function loop(ts){ if (last === undefined) last = ts; let dt = (ts - last) / 1000; if (!Number.isFinite(dt) || dt < 0) dt = 0; dt = Math.min(dt, 0.05); last = ts; if (mode === 'game'){ if (restartDelay > 0) restartDelay = Math.max(0, restartDelay - dt); if (graceTimer > 0) graceTimer = Math.max(0, graceTimer - dt); update(dt); controls.style.display = 'block'; frontStack.style.display = 'none'; menuStack.style.display = 'none'; } else if (mode === 'menu'){ controls.style.display = 'none'; frontStack.style.display = 'none'; menuStack.style.display = 'block'; } else { controls.style.display = 'none'; frontStack.style.display = 'block'; menuStack.style.display = 'none'; } draw(); requestAnimationFrame(loop); }

/* ===== Buttons / Inputs ===== */
const menuStack = document.getElementById('menuStack');
const startBtn = document.getElementById('startBtn');
const frontStack = document.getElementById('frontStack');
const playBtn = document.getElementById('playBtn');
const frontChangeNameBtn = document.getElementById('frontChangeNameBtn');

playBtn.addEventListener('click', async ()=>{ await goFullscreen(); startGame(); });
frontChangeNameBtn.addEventListener('click', ()=>{ localStorage.removeItem(NAME_KEY); const n = getPlayerName(); alert('Player name set to: ' + n); });
startBtn.addEventListener('click', async ()=>{ await goFullscreen(); startGame(); });

/* Keyboard (desktop): space or arrows start from front/menu */
let keyLock = false;
document.addEventListener('keydown', async e=>{ if (mode !== 'game'){ if (e.key === ' ' || e.key === 'ArrowLeft' || e.key === 'ArrowRight'){ await goFullscreen(); startGame(); return; } } if (keyLock) return; if (e.key === 'ArrowLeft'){ if (currentLane > 0) currentLane--; keyLock = true; } else if (e.key === 'ArrowRight'){ if (currentLane < 2) currentLane++; keyLock = true; } });
document.addEventListener('keyup', e=>{ if (e.key === 'ArrowLeft' || e.key === 'ArrowRight') keyLock = false; });

/* Touch on canvas — only nudges while playing */
let touchStartX = null;
canvas.addEventListener('touchstart', e=>{ if (mode !== 'game') return; touchStartX = e.touches[0].clientX; }, {passive: true});
canvas.addEventListener('touchmove', e=>{ if (mode !== 'game' || touchStartX === null) return; const dx = e.touches[0].clientX - touchStartX; if (dx > 50 && currentLane < 2){ currentLane++; touchStartX = e.touches[0].clientX; } else if (dx < -50 && currentLane > 0){ currentLane--; touchStartX = e.touches[0].clientX; } }, {passive: true});
canvas.addEventListener('touchend', ()=>{ touchStartX = null; });

/* Lane buttons */
const btn1 = document.getElementById('btnLane1');
const btn3 = document.getElementById('btnLane3');
function nudgeLeft(){ if (currentLane > 0) currentLane--; }
function nudgeRight(){ if (currentLane < 2) currentLane++; }
function onPointerDownBtn1(e){ e.preventDefault(); btn1Down = true; if (mode==='game') nudgeLeft(); }
function onPointerUpBtn1(e){ e.preventDefault(); btn1Down = false; if (!btn3Down && cheatCharges===0) cheatArmTimerMs = 0; }
function onPointerDownBtn3(e){ e.preventDefault(); btn3Down = true; if (mode==='game') nudgeRight(); }
function onPointerUpBtn3(e){ e.preventDefault(); btn3Down = false; if (!btn1Down && cheatCharges===0) cheatArmTimerMs = 0; }
['pointerdown'].forEach(evt=>{ btn1.addEventListener(evt, onPointerDownBtn1, {passive:false}); btn3.addEventListener(evt, onPointerDownBtn3, {passive:false}); });
['pointerup','pointercancel','pointerout','pointerleave'].forEach(evt=>{ btn1.addEventListener(evt, onPointerUpBtn1, {passive:false}); btn3.addEventListener(evt, onPointerUpBtn3, {passive:false}); });

/* Boot */
fetchGlobalTop(20);
requestAnimationFrame(loop);
</script>
</body>
</html>
