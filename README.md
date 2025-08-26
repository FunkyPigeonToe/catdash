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
    height: 100dvh;              /* dynamic viewport height to avoid top gap */
    overflow: hidden;
  }
  canvas {
    display: block;
    margin: 0 auto;
    background: #6dbb4a; /* static fallback */
    touch-action: none;

    /* Android/GPU stability hints */
    backface-visibility: hidden;
    -webkit-backface-visibility: hidden;
    transform: translateZ(0);
    will-change: transform;
  }

  /* Mobile lane buttons */
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

<!-- Supabase client (global leaderboard) -->
<script src="https://unpkg.com/@supabase/supabase-js@2"></script>
</head>
<body>
<canvas id="gameCanvas"></canvas>

<div id="controls" class="controls">
  <button id="btnLane1" class="laneBtn" aria-label="Move left">◀</button>
  <button id="btnLane3" class="laneBtn" aria-label="Move right">▶</button>
</div>

<script>
/* =========================
   Supabase (Strict highscores)
   ========================= */
const SUPABASE_URL  = 'https://fvcvrhaxxpsientgggnx.supabase.co';
const SUPABASE_ANON = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImZ2Y3ZyaGF4eHBzaWVudGdnZ254Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTYwODczMzYsImV4cCI6MjA3MTY2MzMzNn0.5wTxwGVJDa3gnS61gaDq00xSFGUMEQ0Pda6tJo4VK-A';

let supabaseClient = null;
let leaderboardCache = [];
let leaderboardLoading = false;

async function initSupabase(){
  try{
    if (window.supabase && SUPABASE_URL && SUPABASE_ANON){
      supabaseClient = supabase.createClient(SUPABASE_URL, SUPABASE_ANON, {
        auth: { persistSession: false }
      });
    }
  } catch(e){ console.warn('Supabase init failed', e); }
  await refreshLeaderboard();
}

/* One-time player name */
function getPlayerName(){
  const raw = localStorage.getItem('catdash_player_name');
  return raw ? raw.trim().slice(0, 10) : null;
}
function setPlayerName(name){
  const clean = (name || 'CAT').trim().slice(0, 10) || 'CAT';
  localStorage.setItem('catdash_player_name', clean);
  return clean;
}
function ensurePlayerName(){
  let n = getPlayerName();
  if (!n){
    n = setPlayerName(prompt('Enter your name or initials', 'CAT') || 'CAT');
  }
  return n;
}

/* UPSERT best score per name */
async function submitScore(name, sc){
  if (!supabaseClient) return;
  const cleanName = (name || 'CAT').trim().slice(0,10) || 'CAT';
  const cleanScore = Math.max(0, sc|0);

  const { data: existing } = await supabaseClient
    .from('highscores')
    .select('score')
    .eq('name', cleanName)
    .maybeSingle();

  const best = existing && typeof existing.score === 'number'
    ? Math.max(existing.score|0, cleanScore)
    : cleanScore;

  const { error: upErr } = await supabaseClient
    .from('highscores')
    .upsert({ name: cleanName, score: best, updated_at: new Date().toISOString() }, { onConflict: 'name' });

  if (upErr) console.warn('Highscore upsert error:', upErr);
}

/* Leaderboard: top 20 */
async function refreshLeaderboard(){
  leaderboardLoading = true;
  leaderboardCache = [];
  if (!supabaseClient){ leaderboardLoading = false; return; }

  let ok = false;
  try{
    const { data, error } = await supabaseClient
      .from('highscores')
      .select('name, score, updated_at')
      .order('score', { ascending: false })
      .limit(20);
    if (!error){
      leaderboardCache = (data || []).map(r => ({
        name: (r.name || 'CAT').slice(0,10),
        score: r.score|0,
        date: (r.updated_at || '').slice(0,10)
      }));
      ok = true;
    }
  }catch(e){}

  if (!ok){
    try{
      const { data, error } = await supabaseClient
        .from('scores')
        .select('name, score:max(score)')
        .group('name')
        .order('score', { ascending: false })
        .limit(20);
      if (!error){
        leaderboardCache = (data || []).map(r => ({
          name: (r.name || 'CAT').slice(0,10),
          score: r.score|0,
          date: ''
        }));
      }
    }catch(e){ console.warn('Fallback leaderboard error:', e); }
  }

  leaderboardLoading = false;
}

async function maybeRecordScore(sc){
  if (!Number.isFinite(sc) || sc <= 0) return;
  const name = ensurePlayerName();
  await submitScore(name, sc);
  await refreshLeaderboard();
}

function changePlayerName(){
  const current = getPlayerName() || 'CAT';
  const next = prompt('Change name or initials', current);
  if (next !== null) setPlayerName(next);
}

/* =========================
   Canvas setup (HD + visualViewport + offscreen)
   ========================= */
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d', {
  alpha: false,
  desynchronized: true,
  willReadFrequently: false
});

/* visual viewport helper (no wasted top space) */
function getViewportSize(){
  const vv = window.visualViewport;
  if (vv) return { w: Math.floor(vv.width), h: Math.floor(vv.height) };
  return { w: Math.floor(window.innerWidth), h: Math.floor(window.innerHeight) };
}

let { w: W, h: H } = getViewportSize();
let DPR = 1;

function setHDCanvas(){
  DPR = Math.max(1, Math.min(3, window.devicePixelRatio || 1));
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

/* Offscreen: static grass + lanes + flowers */
let off_bg = document.createElement('canvas');
let off_bg_ctx = off_bg.getContext('2d', { alpha:false });
let off_flowers = document.createElement('canvas');
let off_flowers_ctx = off_flowers.getContext('2d', { alpha:true });

function resizeOffscreens(){
  off_bg.width = Math.floor(W * DPR);
  off_bg.height = Math.floor(H * DPR);
  off_bg_ctx.setTransform(DPR,0,0,DPR,0,0);

  off_flowers.width = Math.floor(W * DPR);
  off_flowers.height = Math.floor(H * DPR);
  off_flowers_ctx.setTransform(DPR,0,0,DPR,0,0);
}
resizeOffscreens();

function prerenderGrassAndTracks(){
  const g = off_bg_ctx.createLinearGradient(0, 0, 0, H);
  g.addColorStop(0, '#64b24a'); g.addColorStop(1, '#4d9c3b');
  off_bg_ctx.fillStyle = g; off_bg_ctx.fillRect(0,0,W,H);

  off_bg_ctx.fillStyle = 'rgba(40,90,40,0.10)';
  for (let y=0; y<H; y+=40){
    for (let x=((y/40)%2===0?0:20); x<W; x+=40){ off_bg_ctx.fillRect(x,y,10,10); }
  }

  const trailW = W/6; const lx = lanesX();
  for (let i=0;i<3;i++){
    const rg = off_bg_ctx.createLinearGradient(0, 0, 0, H);
    rg.addColorStop(0, '#8b684f'); rg.addColorStop(1, '#6f523f');
    off_bg_ctx.fillStyle = rg; off_bg_ctx.fillRect(lx[i]-trailW/2, 0, trailW, H);
    off_bg_ctx.fillStyle = 'rgba(0,0,0,0.18)'; off_bg_ctx.fillRect(lx[i]-1, 0, 2, H);
  }
}
prerenderGrassAndTracks();

/* Flowers: palettes rotate every 400m */
const flowerPalettes = [
  ['#ff6b6b','#ff8ea1','#ffb3b3'],
  ['#6ba8ff','#8ec5ff','#b3d4ff'],
  ['#ffe066','#ffd43b','#fab005'],
  ['#f8f9fa','#e9ecef','#dee2e6'],
  ['#b197fc','#d0bfff','#9775fa']
];
let currentPaletteIndex = 0;
let lastFlowerPaletteMeters = 0;

function prerenderFlowers(paletteIndex){
  off_flowers_ctx.clearRect(0,0,W,H);
  const colors = flowerPalettes[(paletteIndex % flowerPalettes.length + flowerPalettes.length) % flowerPalettes.length];

  const count = Math.floor((W*H) / 16000);
  for (let i=0; i<count; i++){
    const x = Math.random()*W;
    const y = Math.random()*H;
    const lx = lanesX();
    const trailW = W/6;
    const nearLane = lx.some(cx => Math.abs(x - cx) < trailW/2 - 6);
    if (nearLane && Math.random() < 0.75) continue;

    const c = colors[Math.floor(Math.random()*colors.length)];
    off_flowers_ctx.fillStyle = c;

    if (Math.random() < 0.5){
      off_flowers_ctx.beginPath(); off_flowers_ctx.arc(x, y, 1.4, 0, Math.PI*2); off_flowers_ctx.fill();
    } else {
      off_flowers_ctx.fillRect(x-0.8,y,1.6,0.8);
      off_flowers_ctx.fillRect(x,y-0.8,0.8,1.6);
    }
  }
}
prerenderFlowers(currentPaletteIndex);

/* Shared vertical offset (cat + buttons) */
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

/* Handle resize + visual viewport changes */
function handleResize(){
  const s = getViewportSize();
  W = s.w; H = s.h;
  setHDCanvas();
  resizeOffscreens();
  prerenderGrassAndTracks();
  prerenderFlowers(currentPaletteIndex);
  positionButtons();
}
window.addEventListener('resize', handleResize, { passive: true });
if (window.visualViewport){
  window.visualViewport.addEventListener('resize', handleResize, { passive: true });
  window.visualViewport.addEventListener('scroll', handleResize, { passive: true });
}

/* =========================
   Game state
   ========================= */
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

/* Cheat (hold both buttons ≥50ms to arm 2 tree ignores) */
let btn1Down = false, btn3Down = false;
let cheatArmTimerMs = 0;
let cheatCharges = 0;
let cheatRearmLock = false;
const CHEAT_HOLD_TIME_MS = 50;
let cheatToastTimer = 0;
let cheatToastText = '';

/* Dash Mode */
let dashActive = false;
let dashTimer = 0;
const DASH_DURATION = 7.0;        // seconds
const DASH_SPEED_MUL = 1.30;      // +30% speed
const DASH_SCORE_MUL = 2;         // double points during dash
let goldenCountForDash = 0;       // collect 3 golden pickups to trigger

/* Sounds (WebAudio, mobile-safe after first interaction) */
let ac = null;
let unlockedAudio = false;
function ensureAudio(){
  if (ac) return;
  ac = new (window.AudioContext || window.webkitAudioContext)();
}
function unlockAudio(){
  if (unlockedAudio) return;
  ensureAudio();
  const b = ac.createBuffer(1, 1, 22050);
  const s = ac.createBufferSource(); s.buffer = b; s.connect(ac.destination); s.start(0);
  unlockedAudio = true;
}
function envGain(duration=0.2, attack=0.01, release=0.15){
  const g = ac.createGain();
  const t = ac.currentTime;
  g.gain.setValueAtTime(0, t);
  g.gain.linearRampToValueAtTime(0.9, t+attack);
  g.gain.exponentialRampToValueAtTime(0.0001, t+duration);
  return g;
}
function playMeow(){
  if (!ac) return;
  const g = envGain(0.25, 0.01, 0.22);
  const o = ac.createOscillator();
  o.type='triangle';
  const t = ac.currentTime;
  o.frequency.setValueAtTime(420, t);
  o.frequency.exponentialRampToValueAtTime(620, t+0.07);
  o.frequency.exponentialRampToValueAtTime(300, t+0.22);
  o.connect(g).connect(ac.destination);
  o.start(); o.stop(t+0.25);
}
function playChime(){
  if (!ac) return;
  const g = envGain(0.35, 0.005, 0.3);
  const o = ac.createOscillator(); o.type='sine';
  const m = ac.createOscillator(); m.type='sine';
  const mg = ac.createGain(); mg.gain.value = 6;
  m.connect(mg); mg.connect(o.frequency);
  o.frequency.value = 880;
  m.frequency.value = 6;
  o.connect(g).connect(ac.destination);
  const t = ac.currentTime; o.start(t); m.start(t); o.stop(t+0.35); m.stop(t+0.35);
}
function playSplat(){
  if (!ac) return;
  const noise = ac.createBufferSource();
  const buffer = ac.createBuffer(1, ac.sampleRate*0.2, ac.sampleRate);
  const data = buffer.getChannelData(0);
  for (let i=0;i<data.length;i++){ data[i] = (Math.random()*2-1) * (1 - i/data.length); }
  noise.buffer = buffer;
  const g = envGain(0.2, 0.001, 0.2);
  noise.connect(g).connect(ac.destination);
  const t = ac.currentTime; noise.start(t); noise.stop(t+0.2);
}
function playGameOver(){
  if (!ac) return;
  const g = envGain(0.5, 0.005, 0.45);
  const o = ac.createOscillator(); o.type='square';
  const t = ac.currentTime;
  o.frequency.setValueAtTime(440, t);
  o.frequency.linearRampToValueAtTime(220, t+0.4);
  o.connect(g).connect(ac.destination);
  o.start(); o.stop(t+0.5);
}
function playDash(){
  if (!ac) return;
  const g = envGain(0.6, 0.005, 0.55);
  const o = ac.createOscillator(); o.type='sawtooth';
  const t = ac.currentTime;
  o.frequency.setValueAtTime(300, t);
  o.frequency.linearRampToValueAtTime(600, t+0.25);
  o.frequency.linearRampToValueAtTime(420, t+0.5);
  o.connect(g).connect(ac.destination);
  o.start(); o.stop(t+0.6);
}

/* Visual helpers */
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

/* =========================
   Static background blit
   ========================= */
function blitBackground(){
  ctx.drawImage(off_bg, 0, 0, off_bg.width/DPR, off_bg.height/DPR);
  ctx.drawImage(off_flowers, 0, 0, off_flowers.width/DPR, off_flowers.height/DPR);
}

/* =========================
   Trees & Mud
   ========================= */
function drawTree(x,y,w,h){
  withShadow('rgba(0,0,0,0.35)', 10, 4, ()=>{
    const trunkW = w*0.28, trunkH = h*0.48;
    const trunkX = x - trunkW/2, trunkY = y + h*0.12;
    ctx.fillStyle = '#6d3f17';
    ctx.beginPath();
    const r = trunkW*0.35;
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

    const cx = x, cy = y - h*0.06;
    const cMain = '#2e7d32', cMid = '#2f8c34', cLight = '#399c3a';
    ctx.fillStyle = cMid;
    ctx.beginPath(); ctx.arc(cx - w*0.28, cy + h*0.02, h*0.25, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(cx + w*0.28, cy + h*0.02, h*0.25, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = cMain;
    ctx.beginPath(); ctx.arc(cx, cy, h*0.32, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = cLight;
    ctx.beginPath(); ctx.arc(cx, cy - h*0.22, h*0.18, 0, Math.PI*2); ctx.fill();
    ctx.strokeStyle = 'rgba(0,0,0,0.35)';
    ctx.lineWidth = 1.5;
    ctx.beginPath(); ctx.arc(cx, cy, h*0.32, 0, Math.PI*2); ctx.stroke();
  });
}
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

/* =========================
   Pickups
   ========================= */
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
function drawLizard(x, y, scale, golden){
  const t = performance.now()*0.006;
  const sway = Math.sin(t + x*0.03)*1.5*scale;
  const body = golden ? '#ffd54f' : '#5cb85c';
  const belly = golden ? '#ffe082' : '#4cae4c';

  withShadow('rgba(0,0,0,0.25)', 6, 2, ()=>{
    ctx.fillStyle = body;
    ctx.beginPath(); ctx.ellipse(x, y+sway, 16*scale, 6*scale, 0, 0, Math.PI*2); ctx.fill();
    ctx.beginPath();
    ctx.moveTo(x-16*scale, y+sway);
    ctx.quadraticCurveTo(x-26*scale, y+3*scale+sway, x-30*scale, y+sway);
    ctx.lineTo(x-24*scale, y-2*scale+sway);
    ctx.closePath();
    ctx.fill();
    ctx.beginPath(); ctx.ellipse(x+14*scale, y-1*scale+sway, 6*scale, 5*scale, 0, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = belly;
    ctx.fillRect(x-6*scale, y-2*scale+sway, 12*scale, 4*scale);
    ctx.strokeStyle = body; ctx.lineWidth = 2*scale;
    ctx.beginPath();
    ctx.moveTo(x-4*scale, y+5*scale+sway); ctx.lineTo(x-8*scale, y+9*scale+sway);
    ctx.moveTo(x+4*scale, y+5*scale+sway); ctx.lineTo(x+8*scale, y+9*scale+sway);
    ctx.moveTo(x-4*scale, y-5*scale+sway); ctx.lineTo(x-8*scale, y-9*scale+sway);
    ctx.moveTo(x+4*scale, y-5*scale+sway); ctx.lineTo(x+8*scale, y-9*scale+sway);
    ctx.stroke();
    ctx.fillStyle = '#000';
    ctx.beginPath(); ctx.arc(x+17*scale, y-2*scale+sway, 1.4*scale, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(x+12*scale, y-2*scale+sway, 1.4*scale, 0, Math.PI*2); ctx.fill();
  });

  if (golden) drawAdditiveGlow(x, y, 24*scale, 0.85);
}
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

/* =========================
   Spawning & placement
   ========================= */
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
  lane = (withoutLast.length ? withoutLast : candidateLanes)[Math.floor(Math.random()*(withoutLast.length?withoutLast.length:candidateLanes.length))];
  lastPickupLane = lane;

  const inDash = dashActive;
  const r = Math.random();
  let type, golden=false, scale=1;
  if (r < 0.40){ type='mouse';   golden = Math.random()<0.25; scale = golden?1.2:1.0; }
  else if (r < 0.80){ type='bird';   golden = Math.random()<0.25; scale = golden?1.25:1.05; }
  else if (r < (inDash ? 0.98 : 0.95)){ type='lizard'; golden = Math.random()<0.25; scale = golden?1.25:1.1; }
  else { type='chicken'; golden=true; scale=1.35; } // always golden

  const baseW = type==='bird' ? 30 : type==='mouse' ? 34 : type==='lizard' ? 36 : 38;
  const baseH = type==='bird' ? 18 : type==='mouse' ? 18 : type==='lizard' ? 16 : 22;
  const w = baseW * scale;
  const h = baseH * scale;

  pickups.push({type, x: lx[lane], y: spawnY, w, h, scale, golden});
}

/* =========================
   Cat (tail behind body)
   ========================= */
const CAT_W = 20, CAT_H = 30;
function drawCat(x, y, w, h){
  withShadow('rgba(0,0,0,0.35)', 12, 5, ()=>{

    // ---- Tail (drawn FIRST so it's behind body) ----
    const anchorX = x - w*0.9;     // start further back on the left side
    const anchorY = y;             // mid-body height
    const len     = h * 1.4;       // longer
    const baseR   = h * 0.15;      // thicker at base
    const tNow    = performance.now()*0.004;
    const wagAmp  = h * 0.25;      // bigger wag

    ctx.fillStyle = '#b85c1b'; // slightly darker so it reads behind
    const segments = 14;
    for (let i = 0; i <= segments; i++){
      const u  = i / segments;
      const r  = baseR * (1 - 0.8*u);  // taper
      const wag = Math.sin(tNow + u*3) * wagAmp * (1-u);
      const px = anchorX - u*len;      // extend leftwards
      const py = anchorY + wag;
      ctx.beginPath();
      ctx.arc(px, py, r, 0, Math.PI*2);
      ctx.fill();
    }

    // ---- Body ----
    ctx.fillStyle = '#d2691e';
    ctx.beginPath();
    ctx.ellipse(x, y, w/1.6, h/1.15, 0, 0, Math.PI*2);
    ctx.fill();

    // ---- Head ----
    const headR = h*0.36;
    const hx = x, hy = y - h*0.78;
    ctx.beginPath(); ctx.arc(hx,hy,headR,0,Math.PI*2); ctx.fill();

    // ---- Ears ----
    ctx.beginPath();
    ctx.moveTo(hx-headR*0.6,hy-headR*0.15);
    ctx.lineTo(hx-headR*0.25,hy-headR*1.0);
    ctx.lineTo(hx,hy-headR*0.15);
    ctx.closePath(); ctx.fill();
    ctx.beginPath();
    ctx.moveTo(hx+headR*0.6,hy-headR*0.15);
    ctx.lineTo(hx+headR*0.25,hy-headR*1.0);
    ctx.lineTo(hx,hy-headR*0.15);
    ctx.closePath(); ctx.fill();

    // ---- Belly patch ----
    ctx.fillStyle = '#a0522d';
    ctx.beginPath();
    ctx.ellipse(x, y+2, w/2.6, h/2.6, 0, 0, Math.PI*2);
    ctx.fill();

    // ---- Eyes ----
    ctx.fillStyle = '#000';
    ctx.beginPath(); ctx.arc(hx-headR*0.35, hy, headR*0.15, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(hx+headR*0.35, hy, headR*0.15, 0, Math.PI*2); ctx.fill();

    // ---- Whiskers ----
    ctx.strokeStyle = '#fff'; ctx.lineWidth = 1.2;
    ctx.beginPath();
    ctx.moveTo(hx- headR*0.55, hy);   ctx.lineTo(hx- headR*1.1, hy-2);
    ctx.moveTo(hx- headR*0.55, hy+4); ctx.lineTo(hx- headR*1.1, hy+6);
    ctx.moveTo(hx- headR*0.55, hy-4); ctx.lineTo(hx- headR*1.1, hy-6);
    ctx.moveTo(hx+ headR*0.55, hy);   ctx.lineTo(hx+ headR*1.1, hy-2);
    ctx.moveTo(hx+ headR*0.55, hy+4); ctx.lineTo(hx+ headR*1.1, hy+6);
    ctx.moveTo(hx+ headR*0.55, hy-4); ctx.lineTo(hx+ headR*1.1, hy-6);
    ctx.stroke();
  });

  strokeAround('rgba(0,0,0,0.4)', 1, ()=>{
    ctx.beginPath(); ctx.ellipse(x, y, w/1.6, h/1.15, 0, 0, Math.PI*2);
  });
}
/* =========================
   Particles
   ========================= */
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
    ctx.save(); ctx.globalAlpha = a;
    ctx.fillStyle = p.color;
    ctx.beginPath(); ctx.arc(p.x, p.y, 2 + 1.2*a, 0, Math.PI*2); ctx.fill();
    ctx.restore();
  });
}

/* =========================
   HUD & overlays
   ========================= */
function drawHUD(){
  ctx.fillStyle = '#fff'; ctx.font = '16px system-ui, sans-serif';
  ctx.fillText('Score: '  + (dashActive ? Math.round(score) + ' (x2)' : Math.round(score)), 10, 22);
  ctx.fillText('Energy: ' + Math.round(fuel), 10, 42);
  ctx.fillText('Meters: ' + Math.round(meters), 10, 62);

  const txt = cheatCharges > 0 ? `Shield x${cheatCharges}` : '';
  if (txt){
    const w = ctx.measureText(txt).width + 12;
    const x = W - w - 10;
    ctx.fillStyle = '#fff';
    ctx.fillText(txt, x, 28);
  }

  if (cheatToastTimer > 0){
    const a = Math.min(1, cheatToastTimer / 0.3);
    ctx.save(); ctx.globalAlpha = a;
    ctx.font = '18px system-ui, sans-serif';
    const t = cheatToastText || '';
    const tw = ctx.measureText(t).width;
    const tx = (W - tw)/2;
    const ty = 44;
    ctx.fillStyle = 'rgba(0,0,0,0.35)';
    ctx.fillRect(tx - 12, ty - 18, tw + 24, 28);
    ctx.fillStyle = '#ffd54f';
    ctx.fillText(t, tx, ty);
    ctx.restore();
  }
}
function drawVignette(){
  const g = ctx.createRadialGradient(W/2, H*0.58, Math.min(W,H)*0.25, W/2, H*0.58, Math.max(W,H)*0.75);
  g.addColorStop(0, 'rgba(0,0,0,0)');
  g.addColorStop(1, 'rgba(0,0,0,0.35)');
  ctx.fillStyle = g;
  ctx.fillRect(0,0,W,H);
}
function drawDashOverlay(){
  if (!dashActive) return;
  ctx.save();
  ctx.globalAlpha = 0.18;
  ctx.fillStyle = '#ff9800';
  ctx.fillRect(0,0,W,H);
  ctx.globalAlpha = 0.16;
  ctx.strokeStyle = '#ffffff';
  ctx.lineWidth = 2;
  for (let i=0;i<6;i++){
    const x = (i+0.5) * (W/6);
    ctx.beginPath();
    ctx.moveTo(x, 0);
    ctx.lineTo(x + (Math.random()*8-4), H);
    ctx.stroke();
  }
  ctx.restore();
}

/* =========================
   Update & Draw
   ========================= */
const PLAYER_Y = () => Math.min(H - 190, H * 0.64 + CAT_AND_BUTTON_OFFSET);

function armCheatIfHeld(dt){
  const bothDown = btn1Down && btn3Down;
  if (bothDown && !cheatRearmLock && cheatCharges === 0){
    cheatArmTimerMs += dt * 1000;
    if (cheatArmTimerMs >= CHEAT_HOLD_TIME_MS){
      cheatCharges = 2;
      cheatRearmLock = true;
      cheatToastText = 'Shield armed x2';
      cheatToastTimer = 1.0;
      playChime();
    }
  } else {
    cheatArmTimerMs = 0;
  }
  if (!btn1Down && !btn3Down && cheatCharges === 0){
    cheatRearmLock = false;
  }
}

function maybeChangeFlowerPalette(){
  if (meters - lastFlowerPaletteMeters >= 400){
    lastFlowerPaletteMeters = meters - (meters % 400);
    currentPaletteIndex = (currentPaletteIndex + 1) % flowerPalettes.length;
    prerenderFlowers(currentPaletteIndex);
  }
}

function startDash(){
  if (dashActive) return;
  dashActive = true;
  dashTimer = DASH_DURATION;
  cheatToastText = 'DASH!';
  cheatToastTimer = 1.2;
  playDash();
}
function updateDash(dt){
  if (!dashActive) return;
  dashTimer -= dt;
  if (dashTimer <= 0){
    dashActive = false;
    dashTimer = 0;
  }
}

function update(dt){
  const accel = baseAccel * (1 - Math.min(1, roadSpeed / maxSpeed));
  let speedMul = 1;
  if (dashActive) speedMul *= DASH_SPEED_MUL;
  roadSpeed = Math.min(maxSpeed, (roadSpeed + accel) * Math.pow(1.00000002, meters));
  const actualSpeed = roadSpeed * speedMul;

  if (slipTimer > 0){
    slipTimer = Math.max(0, slipTimer - dt);
    slipOffset = Math.sin(performance.now()/40) * 4;
  } else slipOffset = 0;

  spawnTimer += dt;
  const spawnInterval = dashActive ? 0.5 : 0.6;
  if (spawnTimer > spawnInterval){
    spawnTimer = 0;
    if (Math.random() < (dashActive ? 0.65 : 0.70)) spawnEnemy(); else spawnPickup();
    if (Math.random() < (dashActive ? 0.50 : 0.35)) spawnPickup();
  }

  enemies.forEach(e => e.y += actualSpeed*dt);
  pickups.forEach(p => p.y += actualSpeed*dt);
  enemies = enemies.filter(e => e.y < H + 60);
  pickups = pickups.filter(p => p.y < H + 60);

  const px = lanesX()[currentLane] + slipOffset;
  const py = PLAYER_Y();
  const pw = CAT_W - 4;
  const ph = CAT_H - 2;

  armCheatIfHeld(dt);

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
          score = Math.max(0, score - (dashActive?1:2));
          slipTimer = 0.6;
          playSplat();
        } else {
          if (cheatCharges > 0){
            enemies.splice(i,1);
            cheatCharges--;
            playChime();
          } else {
            playGameOver();
            endGame();
          }
        }
        break;
      }
    }
  }

  for (let i=0; i<pickups.length; i++){
    const p = pickups[i];
    if (Math.abs(p.x - px) < (p.w + pw)/2 && Math.abs(p.y - py) < (p.h + ph)/2){
      pickups.splice(i,1);
      let addFuel=0, addScore=0, sparkle='rgba(255,230,150,0.95)';
      if (p.type === 'chicken'){
        addFuel = 25; addScore = 15; sparkle='rgba(255,230,140,0.95)';
        cheatCharges = Math.min(2, cheatCharges + 1);
        cheatToastText = 'Shield +1';
        cheatToastTimer = 1.2;
        goldenCountForDash++;
        if (goldenCountForDash >= 3){ goldenCountForDash = 0; startDash(); }
        playChime();
      } else if (p.type === 'lizard'){
        addFuel = p.golden ? 22 : 12; addScore = p.golden ? 12 : 5; sparkle='rgba(180,255,120,0.95)';
        if (p.golden){ goldenCountForDash++; if (goldenCountForDash >= 3){ goldenCountForDash = 0; startDash(); } }
        playMeow();
      } else if (p.type === 'bird'){
        addFuel = p.golden ? 20 : 10; addScore = p.golden ? 10 : 3; sparkle='rgba(180,210,255,0.95)';
        if (p.golden){ goldenCountForDash++; if (goldenCountForDash >= 3){ goldenCountForDash = 0; startDash(); } }
        playMeow();
      } else { // mouse
        addFuel = p.golden ? 20 : 10; addScore = p.golden ? 10 : 3; sparkle='rgba(255,230,150,0.95)';
        if (p.golden){ goldenCountForDash++; if (goldenCountForDash >= 3){ goldenCountForDash = 0; startDash(); } }
        playMeow();
      }

      if (dashActive) addScore *= DASH_SCORE_MUL;

      fuel  = Math.min(100, fuel + addFuel);
      score += addScore;
      spawnSparkles(px, py, sparkle);
      break;
    }
  }

  updateDash(dt);
  updateParticles(dt);
  if (cheatToastTimer > 0) cheatToastTimer = Math.max(0, cheatToastTimer - dt);

  meters += (actualSpeed * dt) / 120;
  maybeChangeFlowerPalette();

  fuel -= dt * (dashActive ? 2.4 : 2.0);
  if (fuel <= 0){ playGameOver(); endGame(); }
}

function draw(){
  blitBackground();
  enemies.forEach(e => { if (e.type==='tree') drawTree(e.x,e.y,e.w,e.h); else drawMud(e.x,e.y,e.w,e.h); });
  pickups.forEach(p=>{
    if (p.type==='mouse') drawMouse(p.x,p.y,p.scale,p.golden);
    else if (p.type==='bird') drawBird(p.x,p.y,p.scale,p.golden);
    else if (p.type==='lizard') drawLizard(p.x,p.y,p.scale,p.golden);
    else drawChicken(p.x,p.y,p.scale);
  });
  drawCat(lanesX()[currentLane] + slipOffset, PLAYER_Y(), CAT_W, CAT_H);
  drawParticles();
  drawHUD();
  drawDashOverlay();
  drawVignette();
}

function drawMenu(){
  blitBackground();
  ctx.fillStyle = '#fff';

  ctx.font = '16px system-ui, sans-serif';
  const title = 'Cat Dash';
  ctx.fillText(title, (W - ctx.measureText(title).width)/2, 56);

  ctx.font = '14px system-ui, sans-serif';
  const sub = 'Tap, swipe, or use arrows to start';
  ctx.fillText(sub, (W - ctx.measureText(sub).width)/2, 78);

  const player = getPlayerName();
  if (player){
    const who = `Player: ${player}  (press N to change)`;
    ctx.fillText(who, (W - ctx.measureText(who).width)/2, 98);
  }

  const linesY = player ? 116 : 98;
  const lines = ['Aim: Reach a high score', 'Collect critters (golden = best)', 'Chain 3 golden to DASH! (x2 score)'];
  lines.forEach((line,i)=> ctx.fillText(line, (W - ctx.measureText(line).width)/2, linesY + i*16));

  // Leaderboard (Top 20)
  ctx.fillStyle = '#fff';
  ctx.font = '18px system-ui, sans-serif';
  ctx.fillText('Global Highscores (Top 20, Best per Name)', 24, linesY + 70);
  ctx.font = '14px ui-monospace, SFMono-Regular, Menlo, monospace';
  if (leaderboardLoading){
    ctx.fillText('Loading…', 24, linesY + 92);
  } else {
    const board = leaderboardCache || [];
    if (board.length === 0){
      ctx.fillText('No scores yet. Be the first!', 24, linesY + 92);
    } else {
      const n = Math.min(20, board.length);
      for (let i=0; i<n; i++){
        const e = board[i];
        const line = `${String(i+1).padStart(2,' ')}. ${e.name.padEnd(10,' ')}  ${String(e.score).padStart(5,' ')}  ${e.date||''}`;
        ctx.fillText(line, 24, linesY + 92 + 18*i);
      }
    }
  }
  drawVignette();
}

/* =========================
   Control
   ========================= */
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
  dashActive = false;
  dashTimer = 0;
  goldenCountForDash = 0;
  currentPaletteIndex = 0;
  lastFlowerPaletteMeters = 0;
  prerenderFlowers(currentPaletteIndex);
}

function loop(ts){
  if (last === undefined) last = ts;
  let dt = (ts - last) / 1000;
  if (!Number.isFinite(dt) || dt < 0) dt = 0;
  dt = Math.min(dt, 0.05);
  last = ts;

  // We blit offscreen buffers; no full-canvas clear per frame needed (helps Android)
  if (gameRunning){
    if (graceTimer > 0) graceTimer = Math.max(0, graceTimer - dt);
    update(dt);
    draw();
  } else {
    drawMenu();
  }
  requestAnimationFrame(loop);
}

/* Keyboard: lanes + change name + unlock audio */
let keyLock = false;
document.addEventListener('keydown', e=>{
  unlockAudio();
  if (!gameRunning && (e.key === 'n' || e.key === 'N')){ changePlayerName(); return; }
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

/* Touch/Pointer: unlock audio on first interaction */
let touchStartX = null;
canvas.addEventListener('touchstart', e=>{
  unlockAudio();
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

/* Lane buttons — pointer events so we can track hold state */
const btn1 = document.getElementById('btnLane1');
const btn3 = document.getElementById('btnLane3');

function nudgeLeft(){ if (!gameRunning){ unlockAudio(); resetGame(); gameRunning = true; } if (currentLane > 0) currentLane--; }
function nudgeRight(){ if (!gameRunning){ unlockAudio(); resetGame(); gameRunning = true; } if (currentLane < 2) currentLane++; }

function onPointerDownBtn1(e){ e.preventDefault(); unlockAudio(); btn1Down = true; nudgeLeft(); }
function onPointerUpBtn1(e){ e.preventDefault(); btn1Down = false; if (!btn3Down && cheatCharges===0) cheatArmTimerMs = 0; }
function onPointerDownBtn3(e){ e.preventDefault(); unlockAudio(); btn3Down = true; nudgeRight(); }
function onPointerUpBtn3(e){ e.preventDefault(); btn3Down = false; if (!btn1Down && cheatCharges===0) cheatArmTimerMs = 0; }

['pointerdown'].forEach(evt=>{
  btn1.addEventListener(evt, onPointerDownBtn1, {passive:false});
  btn3.addEventListener(evt, onPointerDownBtn3, {passive:false});
});
['pointerup','pointercancel','pointerout','pointerleave'].forEach(evt=>{
  btn1.addEventListener(evt, onPointerUpBtn1, {passive:false});
  btn3.addEventListener(evt, onPointerUpBtn3, {passive:false});
});

/* Tap anywhere to nudge toward tap side + unlock audio */
canvas.addEventListener('click', e=>{
  unlockAudio();
  if (!gameRunning){ resetGame(); gameRunning = true; return; }
  const x = e.clientX;
  const center = lanesX()[currentLane];
  if (x < center) nudgeLeft(); else if (x > center) nudgeRight();
});

/* Boot */
initSupabase();
requestAnimationFrame(loop);
</script>
</body>
</html>
