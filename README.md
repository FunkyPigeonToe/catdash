<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
  <meta name="apple-mobile-web-app-capable" content="yes" />
  <meta name="mobile-web-app-capable" content="yes" />
  <meta name="theme-color" content="#12161c" />
  <title></title>
  <style>
    :root{
      --pink:#ff4081; --pink-hi:#ff6fa3; --indigo:#5865F2; --bg:#12161c;
    }
    html, body { margin:0; padding:0; background:var(--bg); color:#fff; font-family:system-ui, -apple-system, Segoe UI, Roboto, "Helvetica Neue", Arial, "Noto Sans", sans-serif; height:100dvh; overflow:hidden; }
    canvas { display:block; margin:0 auto; background:#000; touch-action:none; backface-visibility:hidden; -webkit-backface-visibility:hidden; transform:translateZ(0); will-change:transform; }

    /* Front & Menu buttons */
    .stack { position:fixed; display:none; z-index:9; left:50%; transform:translateX(-50%); pointer-events:auto; width:min(92vw,520px); }
    #frontStack { top: 56%; }
    #menuStack  { top: 78%; }
    .stack .buttons { display:flex; flex-direction:column; gap:12px; align-items:center; }
    .btn { border:0; border-radius:14px; padding:12px 18px; font-size:17px; color:#fff; box-shadow:0 8px 22px rgba(0,0,0,.35); -webkit-tap-highlight-color:transparent; }
    .btn:active { transform:translateY(1px) scale(.98); }
    .btn-primary { background:linear-gradient(180deg,var(--pink-hi) 0%, var(--pink) 100%); font-weight:800; min-width:220px; }
    .btn-indigo  { background:var(--indigo); min-width:220px; }
    #fsNote { font-size:12px; opacity:.85; text-align:center; margin-top:4px; }

    /* On-screen arrows (in game only) */
    .controls { position:fixed; inset:0; pointer-events:none; z-index:10; display:none; }
    .laneBtn  { position:absolute; width:80px; height:80px; border:none; border-radius:20px; background:rgba(0,0,0,.35); color:#fff; font-size:28px; box-shadow:0 6px 18px rgba(0,0,0,.35); -webkit-tap-highlight-color:transparent; touch-action:manipulation; pointer-events:auto; transform:translate(-50%, -50%); }
    .laneBtn:active { transform:translate(-50%, -50%) scale(.96); }
    @media (min-width:900px){ .controls{ display:none !important; } }
  </style>
</head>
<body>
  <canvas id="gameCanvas"></canvas>

  <!-- FRONT -->
  <div id="frontStack" class="stack">
    <div class="buttons">
      <button id="playBtn" class="btn btn-primary" type="button">Play</button>
      <button id="frontChangeNameBtn" class="btn btn-indigo" type="button">Change Name</button>
      <div id="fsNote">Tip: Play tries to go fullscreen</div>
    </div>
  </div>

  <!-- MENU -->
  <div id="menuStack" class="stack">
    <div class="buttons">
      <button id="startBtn" class="btn btn-primary" type="button">Start Game</button>
    </div>
  </div>

  <!-- In-game lane arrows -->
  <div id="controls" class="controls">
    <button id="btnLane1" class="laneBtn" aria-label="Move left">◀</button>
    <button id="btnLane3" class="laneBtn" aria-label="Move right">▶</button>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
  <script>
  /* ========================= Supabase (Leaderboard) ========================= */
  const SUPABASE_URL = 'https://fvcvrhaxxpsientgggnx.supabase.co';
  const SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImZ2Y3ZyaGF4eHBzaWVudGdnZ254Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTYwODczMzYsImV4cCI6MjA3MTY2MzMzNn0.5wTxwGVJDa3gnS61gaDq00xSFGUMEQ0Pda6tJo4VK-A';
  const TABLE_NAME = 'highscores';
  const supa = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
  const NAME_KEY = 'catdash_name';
  function getPlayerName(){ let n = localStorage.getItem(NAME_KEY); if(!n){ n = prompt('Enter player name (3–12 chars):','CAT')||'CAT'; n=n.trim().slice(0,12); if(n.length<3) n=(n+'CAT').slice(0,3); localStorage.setItem(NAME_KEY,n);} return n; }
  let globalBoard = []; let lastBoardFetch = 0;
  async function fetchGlobalTop(limit=20){ try{ const {data,error}=await supa.from(TABLE_NAME).select('name,score,updated_at').order('score',{ascending:false}).limit(limit); if(error) throw error; globalBoard=Array.isArray(data)?data:[]; lastBoardFetch=performance.now(); }catch(e){ console.warn('LB fetch error',e.message||e); } }
  async function submitBestIfHigher(name,score){ try{ const {data:existing,error:e1}=await supa.from(TABLE_NAME).select('score').eq('name',name).single(); const prev=Number(existing?.score||0); if(!e1 && score<=prev) return false; const payload={name,score,updated_at:new Date().toISOString()}; const {error:e2}=await supa.from(TABLE_NAME).upsert(payload,{onConflict:'name'}); if(e2) throw e2; fetchGlobalTop(20); return true; }catch(e){ console.warn('LB submit error',e.message||e); return false; } }

  /* ========================= Canvas / viewport ========================= */
  const canvas=document.getElementById('gameCanvas');
  const ctx=canvas.getContext('2d',{alpha:false, desynchronized:true});
  function getViewportSize(){ const vv=window.visualViewport; if(vv) return {w:Math.floor(vv.width), h:Math.floor(vv.height)}; return {w:Math.floor(window.innerWidth), h:Math.floor(window.innerHeight)}; }
  let {w:W,h:H}=getViewportSize(); let DPR=1;
  function resizeCanvas(){ DPR=Math.max(1,Math.min(3,window.devicePixelRatio||1)); canvas.style.width=W+'px'; canvas.style.height=H+'px'; canvas.width=Math.floor(W*DPR); canvas.height=Math.floor(H*DPR); ctx.setTransform(DPR,0,0,DPR,0,0); ctx.imageSmoothingEnabled=true; ctx.imageSmoothingQuality='high'; }
  resizeCanvas();
  function handleResize(){ const v=getViewportSize(); W=v.w; H=v.h; resizeCanvas(); positionButtons(); initFlowerSpots(); }
  window.addEventListener('resize',handleResize,{passive:true});
  if(window.visualViewport){ visualViewport.addEventListener('resize',handleResize,{passive:true}); visualViewport.addEventListener('scroll',handleResize,{passive:true}); }
  function lanesX(){ return [W/4, W/2, (3*W)/4]; }

  /* ========================= Controls ========================= */
  const controls=document.getElementById('controls');
  const CAT_Y_LIFT=-30; const BUTTON_Y_OFFSET=50;
  function positionButtons(){ const btn1=document.getElementById('btnLane1'); const btn3=document.getElementById('btnLane3'); const lx=lanesX(); const baseY=H-Math.min(120,H*.12); const y=Math.min(H-10, baseY+BUTTON_Y_OFFSET); btn1.style.left=lx[0]+'px'; btn1.style.top=y+'px'; btn3.style.left=lx[2]+'px'; btn3.style.top=y+'px'; }
  positionButtons();

  /* ========================= Game state (unchanged core) ========================= */
  let mode='front', currentLane=1, enemies=[], pickups=[], particles=[]; let score=0, fuel=100, meters=0; let spawnTimer=0, last=undefined, graceTimer=.75, restartDelay=0; let roadSpeed=226, maxSpeed=704, baseAccel=.6, slipTimer=0, slipOffset=0; let btn1Down=false, btn3Down=false, cheatArmTimerMs=0, cheatCharges=0, cheatRearmLock=false; const CHEAT_HOLD_TIME_MS=50; let cheatToastTimer=0, cheatToastText=''; let dashActive=false, dashTimer=0; const DASH_DURATION=8.0, DASH_SPEED_MUL=1.30, DASH_SCORE_MUL=2;

  /* ========================= Visual helpers ========================= */
  function withShadow(color='rgba(0,0,0,.35)', blur=8, oy=3, fn){ ctx.save(); ctx.shadowColor=color; ctx.shadowBlur=blur; ctx.shadowOffsetX=0; ctx.shadowOffsetY=oy; fn(); ctx.restore(); }
  function strokeAround(color='rgba(0,0,0,.35)', lw=2, fn){ ctx.save(); ctx.lineWidth=lw; ctx.strokeStyle=color; fn(); ctx.stroke(); ctx.restore(); }

  /* ========================= Background (game scenes) ========================= */
  const FLOWER_SEG_METERS=400; const FLOWER_COLORS=['#ffec99','#ffd6e7','#c0ebff','#c3fda7','#ffd8a8','#eebefa','#b2f2bb']; const flowerSpots=[]; function initFlowerSpots(){ flowerSpots.length=0; const count=Math.max(80,Math.floor(W*H/11000)); for(let i=0;i<count;i++){ flowerSpots.push({x:Math.random()*W,y:Math.random()*H,r:1.2+Math.random()*1.4,rot:Math.random()*Math.PI*2,stem:Math.random()<.8}); } } initFlowerSpots();
  function drawBloom(x,y,s,color,rot){ ctx.save(); ctx.translate(x,y); ctx.rotate(rot); const pr=s*2.1, cr=s*1.1; ctx.fillStyle=color; for(let i=0;i<5;i++){ const a=(i/5)*Math.PI*2; ctx.beginPath(); ctx.ellipse(Math.cos(a)*s*1.1, Math.sin(a)*s*1.1, pr*.55, pr*.35, a, 0, Math.PI*2); ctx.fill(); } const g=ctx.createRadialGradient(0,0,0,0,0,cr); g.addColorStop(0,'rgba(255,255,220,.95)'); g.addColorStop(1,'rgba(255,255,220,.2)'); ctx.fillStyle=g; ctx.beginPath(); ctx.arc(0,0,cr,0,Math.PI*2); ctx.fill(); ctx.restore(); }
  function drawBackground(){ const g=ctx.createLinearGradient(0,0,0,H); g.addColorStop(0,'#64b24a'); g.addColorStop(1,'#4d9c3b'); ctx.fillStyle=g; ctx.fillRect(0,0,W,H); ctx.fillStyle='rgba(40,90,40,.10)'; for(let y=0;y<H;y+=40){ for(let x=((y/40)%2===0?0:20); x<W; x+=40){ ctx.fillRect(x,y,10,10);} } const seg=Math.floor(meters/FLOWER_SEG_METERS)%FLOWER_COLORS.length; const color=FLOWER_COLORS[seg]; const lx=lanesX(), trailW=W/6; flowerSpots.forEach(f=>{ const inLane=(Math.abs(f.x-lx[0])<trailW/2)||(Math.abs(f.x-lx[1])<trailW/2)||(Math.abs(f.x-lx[2])<trailW/2); if(inLane) return; if(f.stem){ ctx.strokeStyle='rgba(20,80,20,.6)'; ctx.lineWidth=1; ctx.beginPath(); ctx.moveTo(f.x,f.y+4); ctx.lineTo(f.x,f.y+8); ctx.stroke(); } drawBloom(f.x,f.y,f.r,color,f.rot); }); for(let i=0;i<3;i++){ const rg=ctx.createLinearGradient(0,0,0,H); rg.addColorStop(0,'#8b684f'); rg.addColorStop(1,'#6f523f'); ctx.fillStyle=rg; ctx.fillRect(lx[i]-trailW/2,0,trailW,H); ctx.fillStyle='rgba(0,0,0,.18)'; ctx.fillRect(lx[i]-1,0,2,H);} }

  /* ========================= Obstacles, pickups, cat, HUD (unchanged) ========================= */
  function drawTree(x,y,w,h){ withShadow('rgba(0,0,0,.35)',10,4,()=>{ const tw=w*.28, th=h*.48, tx=x-tw/2, ty=y+h*.12, r=tw*.35; ctx.fillStyle='#6d3f17'; ctx.beginPath(); ctx.moveTo(tx+r,ty); ctx.lineTo(tx+tw-r,ty); ctx.quadraticCurveTo(tx+tw,ty,tx+tw,ty+r); ctx.lineTo(tx+tw,ty+th-r); ctx.quadraticCurveTo(tx+tw,ty+th,tx+tw-r,ty+th); ctx.lineTo(tx+r,ty+th); ctx.quadraticCurveTo(tx,ty+th,tx,ty+th-r); ctx.lineTo(tx,ty+r); ctx.quadraticCurveTo(tx,ty,tx+r,ty); ctx.closePath(); ctx.fill(); const cx=x, cy=y-h*.06; const c1='#2e7d32', c2='#2f8c34', c3='#399c3a'; ctx.fillStyle=c2; ctx.beginPath(); ctx.arc(cx-w*.28,cy+h*.02,h*.25,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(cx+w*.28,cy+h*.02,h*.25,0,Math.PI*2); ctx.fill(); ctx.fillStyle=c1; ctx.beginPath(); ctx.arc(cx,cy,h*.32,0,Math.PI*2); ctx.fill(); ctx.fillStyle=c3; ctx.beginPath(); ctx.arc(cx,cy-h*.22,h*.18,0,Math.PI*2); ctx.fill(); ctx.strokeStyle='rgba(0,0,0,.35)'; ctx.lineWidth=1.5; ctx.beginPath(); ctx.arc(cx,cy,h*.32,0,Math.PI*2); ctx.stroke(); }); }
  function drawMud(x,y,w,h){ withShadow('rgba(0,0,0,.3)',8,3,()=>{ const g=ctx.createRadialGradient(x,y,2,x,y,Math.max(w,h)); g.addColorStop(0,'#6a4a3a'); g.addColorStop(1,'#3e2723'); ctx.fillStyle=g; ctx.beginPath(); ctx.ellipse(x,y,w*.5,h*.5,0,0,Math.PI*2); ctx.fill(); }); strokeAround('rgba(0,0,0,.35)',1.2,()=>{ ctx.beginPath(); ctx.ellipse(x,y,w*.5,h*.5,0,0,Math.PI*2); }); }
  function drawAdditiveGlow(x,y,r,a=.9){ ctx.save(); ctx.globalCompositeOperation='lighter'; const g=ctx.createRadialGradient(x,y,0,x,y,r); g.addColorStop(0,`rgba(255,215,0,${a})`); g.addColorStop(1,'rgba(255,215,0,0)'); ctx.fillStyle=g; ctx.beginPath(); ctx.arc(x,y,r,0,Math.PI*2); ctx.fill(); ctx.restore(); }
  function drawMouse(x,y,s,gold){ const t=performance.now()*0.006, wig=Math.sin(t+x*.01)*1.2*s; const body=gold?'#ffd54f':'#c7a17a', ear=gold?'#ffe082':'#d7b894'; withShadow('rgba(0,0,0,.25)',6,2,()=>{ ctx.fillStyle=body; ctx.beginPath(); ctx.ellipse(x,y+wig,12*s,7*s,0,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.ellipse(x+9*s,y-1*s+wig,6*s,5*s,0,0,Math.PI*2); ctx.fill(); ctx.fillStyle=ear; ctx.beginPath(); ctx.arc(x+12*s,y-5*s+wig,2.8*s,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(x+7.5*s,y-6*s+wig,2.2*s,0,Math.PI*2); ctx.fill(); ctx.strokeStyle=gold?'#ffe082':'#b78963'; ctx.lineWidth=1.4*s; ctx.beginPath(); ctx.moveTo(x-12*s,y+2*s+wig); ctx.quadraticCurveTo(x-18*s,y+6*s+wig,x-22*s,y+3*s+wig); ctx.stroke(); ctx.fillStyle='#000'; ctx.beginPath(); ctx.arc(x+11*s,y-2*s+wig,1.4*s,0,Math.PI*2); ctx.fill(); }); if(gold) drawAdditiveGlow(x,y,22*s,.85); }
  function drawBird(x,y,s,gold){ const t=performance.now()*0.004, bob=Math.sin(t+x*.02)*1.2*s; const body=gold?'#ffe066':'#66a9ff'; withShadow('rgba(0,0,0,.25)',6,2,()=>{ ctx.fillStyle=body; ctx.beginPath(); ctx.ellipse(x,y+bob,10*s,7*s,0,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(x+7*s,y-3*s+bob,4*s,0,Math.PI*2); ctx.fill(); ctx.fillStyle=gold?'#ffd54f':'#4f94f5'; ctx.beginPath(); ctx.ellipse(x-3*s,y+1*s+bob,6*s,4*s,-.7,0,Math.PI*2); ctx.fill(); ctx.fillStyle=gold?'#ffca28':'#ffb300'; ctx.beginPath(); ctx.moveTo(x+11*s,y-3*s+bob); ctx.lineTo(x+15*s,y-1*s+bob); ctx.lineTo(x+11*s,y-1*s+bob); ctx.closePath(); ctx.fill(); ctx.fillStyle='#000'; ctx.beginPath(); ctx.arc(x+6*s,y-4*s+bob,1.2*s,0,Math.PI*2); ctx.fill(); }); if(gold) drawAdditiveGlow(x,y,20*s,.85); }
  function drawLizard(x,y,s,gold){ const t=performance.now()*0.006, sway=Math.sin(t+x*.03)*1.5*s; const body=gold?'#ffd54f':'#5cb85c', belly=gold?'#ffe082':'#4cae4c'; withShadow('rgba(0,0,0,.25)',6,2,()=>{ ctx.fillStyle=body; ctx.beginPath(); ctx.ellipse(x,y+sway,16*s,6*s,0,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.moveTo(x-16*s,y+sway); ctx.quadraticCurveTo(x-26*s,y+3*s+sway,x-30*s,y+sway); ctx.lineTo(x-24*s,y-2*s+sway); ctx.closePath(); ctx.fill(); ctx.beginPath(); ctx.ellipse(x+14*s,y-1*s+sway,6*s,5*s,0,0,Math.PI*2); ctx.fill(); ctx.fillStyle=belly; ctx.fillRect(x-6*s,y-2*s+sway,12*s,4*s); ctx.fillStyle='#000'; ctx.beginPath(); ctx.arc(x+17*s,y-2*s+sway,1.4*s,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(x+12*s,y-2*s+sway,1.4*s,0,Math.PI*2); ctx.fill(); }); if(gold) drawAdditiveGlow(x,y,24*s,.85); }
  function drawChicken(x,y,s){ const bob=Math.sin(performance.now()*0.004+x*0.01)*1.0*s; withShadow('rgba(0,0,0,.25)',6,2,()=>{ ctx.fillStyle='#ffe082'; ctx.beginPath(); ctx.ellipse(x,y+bob,12*s,9*s,0,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(x+9*s,y-5*s+bob,5*s,0,Math.PI*2); ctx.fill(); ctx.fillStyle='#ffb300'; ctx.beginPath(); ctx.moveTo(x+14*s,y-5*s+bob); ctx.lineTo(x+18*s,y-4*s+bob); ctx.lineTo(x+14*s,y-2.5*s+bob); ctx.closePath(); ctx.fill(); ctx.fillStyle='#e53935'; ctx.beginPath(); ctx.arc(x+8*s,y-9*s+bob,2.1*s,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(x+10.8*s,y-9.4*s+bob,1.8*s,0,Math.PI*2); ctx.fill(); ctx.fillStyle='#ffd54f'; ctx.beginPath(); ctx.ellipse(x-3*s,y-1*s+bob,7*s,5*s,-.7,0,Math.PI*2); ctx.fill(); ctx.fillStyle='#000'; ctx.beginPath(); ctx.arc(x+8*s,y-6*s+bob,1.4*s,0,Math.PI*2); ctx.fill(); ctx.strokeStyle='#ffb300'; ctx.lineWidth=1.3*s; ctx.beginPath(); ctx.moveTo(x-2*s,y+9*s+bob); ctx.lineTo(x-2*s,y+12*s+bob); ctx.moveTo(x+1*s,y+9*s+bob); ctx.lineTo(x+1*s,y+12*s+bob); ctx.stroke(); }); drawAdditiveGlow(x,y,24*s,.9); }
  const CAT_W=22, CAT_H=34;
  function drawCat(x,y,w,h){ withShadow('rgba(0,0,0,.35)',12,5,()=>{ const rx=w/1.55, ry=h/1.12, tail=h*1.35, tb=Math.max(3,w*.30), t=performance.now()*0.008, amp=h*.12; const bx=x-rx+tb*.8, by=y+ry*.60; ctx.strokeStyle='#9a5a2a'; ctx.lineCap='round'; ctx.lineJoin='round'; let px=bx, py=by; for(let i=1;i<=16;i++){ const u=i/16, k=1-u; const xx=bx-u*tail*.95; const yy=by-u*tail*.55+Math.sin(t+u*7)*amp*(.25+.75*k); ctx.lineWidth=Math.max(1,tb*(.3+.7*k)); ctx.beginPath(); ctx.moveTo(px,py); ctx.lineTo(xx,yy); ctx.stroke(); px=xx; py=yy; } ctx.fillStyle='#d07a2b'; ctx.beginPath(); ctx.ellipse(x,y,rx,ry,0,0,Math.PI*2); ctx.fill(); const hr=h*.36, hx=x, hy=y-h*.78; ctx.beginPath(); ctx.arc(hx,hy,hr,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.moveTo(hx-hr*.6,hy-hr*.15); ctx.lineTo(hx-hr*.25,hy-hr*1.0); ctx.lineTo(hx,hy-hr*.15); ctx.closePath(); ctx.fill(); ctx.beginPath(); ctx.moveTo(hx+hr*.6,hy-hr*.15); ctx.lineTo(hx+hr*.25,hy-hr*1.0); ctx.lineTo(hx,hy-hr*.15); ctx.closePath(); ctx.fill(); ctx.fillStyle='#a85b24'; ctx.beginPath(); ctx.ellipse(x,y+2,w/2.6,h/2.6,0,0,Math.PI*2); ctx.fill(); ctx.fillStyle='#000'; ctx.beginPath(); ctx.arc(hx-hr*.35,hy,hr*.15,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(hx+hr*.35,hy,hr*.15,0,Math.PI*2); ctx.fill(); ctx.strokeStyle='#fff'; ctx.lineWidth=1.2; ctx.beginPath(); ctx.moveTo(hx-hr*.55,hy); ctx.lineTo(hx-hr*1.1,hy-2); ctx.moveTo(hx-hr*.55,hy+4); ctx.lineTo(hx-hr*1.1,hy+6); ctx.moveTo(hx-hr*.55,hy-4); ctx.lineTo(hx-hr*1.1,hy-6); ctx.moveTo(hx+hr*.55,hy); ctx.lineTo(hx+hr*1.1,hy-2); ctx.moveTo(hx+hr*.55,hy+4); ctx.lineTo(hx+hr*1.1,hy+6); ctx.moveTo(hx+hr*.55,hy-4); ctx.lineTo(hx+hr*1.1,hy-6); ctx.stroke(); }); strokeAround('rgba(0,0,0,.4)',1,()=>{ ctx.beginPath(); ctx.ellipse(x,y,w/1.55,h/1.12,0,0,Math.PI*2); }); }

  /* ========================= Particles & HUD ========================= */
  function spawnSparkles(x,y,c){ for(let i=0;i<12;i++){ const a=(Math.PI*2)*(i/12)+Math.random()*.4; const sp=60+Math.random()*110; particles.push({x,y,vx:Math.cos(a)*sp,vy:Math.sin(a)*sp-40,life:.6+Math.random()*.4,age:0,color:c}); } }
  function updateParticles(dt){ for(let i=particles.length-1;i>=0;i--){ const p=particles[i]; p.age+=dt; p.x+=p.vx*dt; p.y+=p.vy*dt; p.vy+=80*dt; if(p.age>=p.life) particles.splice(i,1); } }
  function drawParticles(){ particles.forEach(p=>{ const a=Math.max(0,1-p.age/p.life); ctx.save(); ctx.globalAlpha=a; ctx.fillStyle=p.color; ctx.beginPath(); ctx.arc(p.x,p.y,2+1.2*a,0,Math.PI*2); ctx.fill(); ctx.restore(); }); }
  function drawHUD(){ ctx.fillStyle='#fff'; ctx.font='16px system-ui, sans-serif'; const sTxt='Score: ' + (dashActive?Math.round(score)+' (x2)':Math.round(score)); ctx.fillText(sTxt,10,22); ctx.fillText('Energy: '+Math.round(fuel),10,42); ctx.fillText('Meters: '+Math.round(meters),10,62); const txt=cheatCharges>0?`Shield x${cheatCharges}`:''; if(txt){ const tw=ctx.measureText(txt).width+12; ctx.fillText(txt, W-tw-10, 28); }
    if(dashActive){ const tTxt=`DASH ${dashTimer.toFixed(1)}s`; const tw=ctx.measureText(tTxt).width; ctx.fillStyle='#ffeb3b'; ctx.fillText(tTxt,(W-tw)/2,22); }
    if(cheatToastTimer>0){ const a=Math.min(1,cheatToastTimer/.3); ctx.save(); ctx.globalAlpha=a; ctx.font='18px system-ui,sans-serif'; const t=cheatToastText||''; const tw=ctx.measureText(t).width; const tx=(W-tw)/2, ty=44; ctx.fillStyle='rgba(0,0,0,.35)'; ctx.fillRect(tx-12,ty-18,tw+24,28); ctx.fillStyle='#ffd54f'; ctx.fillText(t,tx,ty); ctx.restore(); }
  }
  function drawVignette(){ const g=ctx.createRadialGradient(W/2,H*.58,Math.min(W,H)*.25,W/2,H*.58,Math.max(W,H)*.75); g.addColorStop(0,'rgba(0,0,0,0)'); g.addColorStop(1,'rgba(0,0,0,.35)'); ctx.fillStyle=g; ctx.fillRect(0,0,W,H); }
  function drawDashOverlay(){ if(!dashActive) return; ctx.save(); ctx.globalAlpha=.18; ctx.fillStyle='#ff9800'; ctx.fillRect(0,0,W,H); ctx.globalAlpha=.16; ctx.strokeStyle='#fff'; ctx.lineWidth=2; for(let i=0;i<6;i++){ const x=(i+.5)*(W/6); ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x+(Math.random()*8-4),H); ctx.stroke(); } ctx.restore(); }
  function drawGlobalBoard(x,y,maxRows=20){ ctx.fillStyle='#fff'; ctx.font='18px system-ui, sans-serif'; ctx.fillText('Global Top 20', x, y); ctx.font='14px ui-monospace, SFMono-Regular, Menlo, monospace'; if(!globalBoard.length){ ctx.fillText('Loading...', x, y+22); if(performance.now()-lastBoardFetch>5000) fetchGlobalTop(20); return; } const rows=Math.min(maxRows, globalBoard.length); for(let i=0;i<rows;i++){ const e=globalBoard[i]; const name=(e.name||'???').slice(0,12).padEnd(12,' '); const line=`${String(i+1).padStart(2,' ')}. ${name}  ${String(Number(e.score||0)).padStart(6,' ')}  ${String((e.updated_at||'').slice(0,10))}`; ctx.fillText(line, x, y + 22 + i*18); } }

  /* ========================= Anime Front Page ========================= */
  const CAT_MASCOT_NAME = 'Gingerbolt';
  let frontT=0;
  function drawFrontBackground(){
    // Sunrise anime gradient
    const g=ctx.createLinearGradient(0,0,0,H); g.addColorStop(0,'#ff3d7f'); g.addColorStop(.45,'#ff7a3d'); g.addColorStop(1,'#ffe27a'); ctx.fillStyle=g; ctx.fillRect(0,0,W,H);
    // Radial burst
    const cx=W*.5, cy=H*.75, rays=90; ctx.save(); ctx.globalAlpha=.18; for(let i=0;i<rays;i++){ const a=(i/rays)*Math.PI*2 + Math.sin(frontT*.9+i)*.05; const len=Math.max(W,H)*1.2; const w=2 + (i%4===0?3:0) + Math.sin(frontT*2+i)*.5; ctx.strokeStyle = i%3===0? 'rgba(255,255,255,.75)': 'rgba(255,255,255,.45)'; ctx.lineWidth=w; ctx.beginPath(); ctx.moveTo(cx,cy); ctx.lineTo(cx+Math.cos(a)*len, cy+Math.sin(a)*len); ctx.stroke(); } ctx.restore();
    // Vignette
    const vg=ctx.createRadialGradient(W/2,H*.65,Math.min(W,H)*.2, W/2,H*.65, Math.max(W,H)*.85); vg.addColorStop(0,'rgba(0,0,0,0)'); vg.addColorStop(1,'rgba(0,0,0,.38)'); ctx.fillStyle=vg; ctx.fillRect(0,0,W,H);
  }
  function drawFrontAnimation(){
    const baseY=H*.74; const travel=Math.min(280, W*.35); const phase=(Math.sin(frontT*.8)*.5+.5); const catX=W*.5 - travel/2 + phase*travel; const catY=baseY - 10 + Math.sin(frontT*3)*4;
    // Aura
    for(let i=0;i<3;i++){ const r=60+i*18+Math.sin(frontT*2+i)*3; const ag=ctx.createRadialGradient(catX,catY-20,0,catX,catY-20,r); ag.addColorStop(0,'rgba(255,230,120,.55)'); ag.addColorStop(1,'rgba(255,230,120,0)'); ctx.fillStyle=ag; ctx.beginPath(); ctx.arc(catX,catY-20,r,0,Math.PI*2); ctx.fill(); }
    // Speed streaks
    ctx.save(); ctx.globalAlpha=.9; ctx.strokeStyle='rgba(255,255,255,.9)'; ctx.lineWidth=3; for(let i=0;i<8;i++){ const ox=-50 - i*14; const oy=Math.sin(frontT*10+i)*8; ctx.beginPath(); ctx.moveTo(catX+ox,catY+oy); ctx.lineTo(catX+ox-60, catY+oy-6); ctx.stroke(); } ctx.restore();
    // Ginger dashing cat
    withShadow('rgba(0,0,0,.35)',14,6,()=>{
      ctx.strokeStyle='#d9782e'; ctx.lineWidth=8; ctx.lineCap='round'; ctx.beginPath(); ctx.moveTo(catX-26,catY-6); ctx.lineTo(catX-44,catY-16); ctx.lineTo(catX-34,catY-28); ctx.lineTo(catX-52,catY-38); ctx.stroke();
      ctx.fillStyle='#e0812c'; ctx.beginPath(); ctx.ellipse(catX,catY,38,26,0,0,Math.PI*2); ctx.fill();
      const hx=catX+26, hy=catY-26, hr=16; ctx.beginPath(); ctx.arc(hx,hy,hr,0,Math.PI*2); ctx.fill();
      ctx.beginPath(); ctx.moveTo(hx-10,hy-4); ctx.lineTo(hx-6,hy-22); ctx.lineTo(hx,hy-6); ctx.closePath(); ctx.fill();
      ctx.beginPath(); ctx.moveTo(hx+10,hy-4); ctx.lineTo(hx+6,hy-22); ctx.lineTo(hx,hy-6); ctx.closePath(); ctx.fill();
      ctx.fillStyle='#d26f1f'; for(let i=0;i<5;i++){ const ax=hx-12+i*6, ay=hy+4+(i%2?2:-2); ctx.beginPath(); ctx.moveTo(ax,ay); ctx.lineTo(ax+4,ay+10); ctx.lineTo(ax-2,ay+8); ctx.closePath(); ctx.fill(); }
      ctx.fillStyle='#c76918'; ctx.beginPath(); ctx.ellipse(catX,catY+2,16,12,0,0,Math.PI*2); ctx.fill();
      ctx.fillStyle='#000'; ctx.beginPath(); ctx.ellipse(hx-6,hy-2,3.6,5.2,0,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.ellipse(hx+6,hy-2,3.6,5.2,0,0,Math.PI*2); ctx.fill(); ctx.fillStyle='#fff'; ctx.beginPath(); ctx.arc(hx-7.5,hy-4.5,1.4,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(hx+4.5,hy-4.5,1.2,0,Math.PI*2); ctx.fill();
      ctx.strokeStyle='#fff'; ctx.lineWidth=1.6; ctx.beginPath(); ctx.moveTo(hx-4,hy+3); ctx.quadraticCurveTo(hx,hy+6,hx+4,hy+3); ctx.stroke(); for(const s of [-1,1]){ ctx.beginPath(); ctx.moveTo(hx+s*6,hy+2); ctx.lineTo(hx+s*18,hy); ctx.stroke(); ctx.beginPath(); ctx.moveTo(hx+s*6,hy+6); ctx.lineTo(hx+s*18,hy+8); ctx.stroke(); }
      ctx.fillStyle='#e0812c'; ctx.beginPath(); ctx.ellipse(catX+46,catY-6,10,6,.2,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.ellipse(catX-6,catY+10,11,7,-.2,0,Math.PI*2); ctx.fill();
    });
  }
  function drawFront(){
    drawFrontBackground();
    // Anime title
    const title='CAT DASH'; ctx.save(); const fs=Math.floor(Math.min(W,H)*.13); ctx.font=`${fs}px Impact, 'Trebuchet MS', system-ui, sans-serif`; ctx.textAlign='center'; ctx.lineWidth=Math.max(6,fs*.06); ctx.strokeStyle='rgba(255,255,255,.95)'; ctx.fillStyle='#2a0a0a'; const tx=W*.5, ty=H*.20; ctx.strokeText(title,tx,ty); ctx.fillText(title,tx,ty); ctx.restore();
    // Subtitle
    const sub=`Starring ${CAT_MASCOT_NAME} — the ginger bolt!`; ctx.save(); ctx.font='18px system-ui, sans-serif'; ctx.fillStyle='rgba(255,255,255,.95)'; ctx.fillText(sub, (W-ctx.measureText(sub).width)/2, H*.24); ctx.restore();
    drawFrontAnimation();
    // Leaderboard bottom-left
    const margin=18; const baseY=H*.52; drawGlobalBoard(margin, baseY, Math.min(10, Math.floor((H-baseY-40)/18)-1)); drawVignette();
  }

  /* ========================= Spawning & collisions (unchanged) ========================= */
  const SPAWN_BUFFER_Y=70; function laneIsFree(x,y){ return !enemies.some(e=>e.x===x && Math.abs(e.y-y)<SPAWN_BUFFER_Y) && !pickups.some(p=>p.x===x && Math.abs(p.y-y)<SPAWN_BUFFER_Y); }
  function pickFreeLane(spawnY){ const lx=lanesX(); const c=[0,1,2].filter(i=>laneIsFree(lx[i],spawnY)); if(!c.length) return null; return c[Math.floor(Math.random()*c.length)]; }
  let lastPickupLane=null; function spawnEnemy(){ const lane=pickFreeLane(-60); if(lane==null) return; const lx=lanesX(); const type=Math.random()<.55?'tree':'mud'; enemies.push(type==='tree'?{type,x:lx[lane],y:-50,w:40,h:70}:{type,x:lx[lane],y:-40,w:56,h:24}); }
  function laneHasAnyEnemy(li){ const lx=lanesX(), x=lx[li]; return enemies.some(e=>e.x===x); }
  function spawnPickup(){ const y=-40; if(enemies.some(e=>Math.abs(e.y-y)<SPAWN_BUFFER_Y) || pickups.some(p=>Math.abs(p.y-y)<SPAWN_BUFFER_Y)) return; const lx=lanesX(); let c=[0,1,2].filter(i=>laneIsFree(lx[i],y) && !laneHasAnyEnemy(i)); if(!c.length) return; const without=c.filter(l=>l!==lastPickupLane); const lane=(without.length?without:c)[Math.floor(Math.random()* (without.length?without.length:c.length))]; lastPickupLane=lane; const r=Math.random(); let type,golden=false, scale=1; if(r<.06){ type='dash'; scale=1.2;} else if(r<.46){ type='mouse'; golden=Math.random()<.25; scale=golden?1.2:1.0;} else if(r<.86){ type='bird'; golden=Math.random()<.25; scale=golden?1.25:1.05;} else if(r<.97){ type='lizard'; golden=Math.random()<.25; scale=golden?1.25:1.1;} else { type='chicken'; golden=true; scale=1.35;} const baseW= type==='bird'?30 : type==='mouse'?34 : type==='lizard'?36 : type==='chicken'?38 : 30; const baseH= type==='bird'?18 : type==='mouse'?18 : type==='lizard'?16 : type==='chicken'?22 : 30; const w=baseW*scale, h=baseH*scale; pickups.push({type,x:lx[lane],y,w,h,scale,golden}); }

  /* ========================= Update & draw loop ========================= */
  const PLAYER_Y = () => Math.min(H - 190, H * 0.64 + CAT_Y_LIFT);
  function armCheatIfHeld(dt){ const both=btn1Down && btn3Down; if(both && !cheatRearmLock && cheatCharges===0){ cheatArmTimerMs+=dt*1000; if(cheatArmTimerMs>=CHEAT_HOLD_TIME_MS){ cheatCharges=2; cheatRearmLock=true; cheatToastText='Shield armed x2'; cheatToastTimer=1.0; } } else cheatArmTimerMs=0; if(!btn1Down && !btn3Down && cheatCharges===0) cheatRearmLock=false; }
  function startDash(){ if(dashActive) return; dashActive=true; dashTimer=DASH_DURATION; cheatToastText='DASH!'; cheatToastTimer=1.2; }
  function updateDash(dt){ if(!dashActive) return; dashTimer-=dt; if(dashTimer<=0){ dashActive=false; dashTimer=0; } }
  function update(dt){ const accel=baseAccel*(1-Math.min(1,roadSpeed/maxSpeed)); const speedMul=dashActive?DASH_SPEED_MUL:1; roadSpeed=Math.min(maxSpeed, (roadSpeed+accel)*Math.pow(1.00000002, meters)); const actual=roadSpeed*speedMul; if(slipTimer>0){ slipTimer=Math.max(0, slipTimer-dt); slipOffset=Math.sin(performance.now()/40)*4; } else slipOffset=0; spawnTimer+=dt; const spawnInt=dashActive?.5:.6; if(spawnTimer>spawnInt){ spawnTimer=0; if(Math.random()<(dashActive?.65:.70)) spawnEnemy(); else spawnPickup(); if(Math.random()<(dashActive?.50:.35)) spawnPickup(); }
    enemies.forEach(e=>e.y+=actual*dt); pickups.forEach(p=>p.y+=actual*dt); enemies=enemies.filter(e=>e.y<H+60); pickups=pickups.filter(p=>p.y<H+60);
    const px=lanesX()[currentLane]+slipOffset, py=PLAYER_Y(), pw=CAT_W-4, ph=CAT_H-2; armCheatIfHeld(dt); const collide=graceTimer<=0;
    if(collide){ for(let i=0;i<enemies.length;i++){ const e=enemies[i]; let ew=e.w, eh=e.h; if(e.type==='tree'){ ew*=.6; eh*=.8;} if(Math.abs(e.x-px)<(ew+pw)/2 && Math.abs(e.y-py)<(eh+ph)/2){ if(e.type==='mud'){ enemies.splice(i,1); fuel=Math.max(0,fuel-10); score=Math.max(0, score-(dashActive?1:2)); slipTimer=.6; } else { if(cheatCharges>0){ enemies.splice(i,1); cheatCharges--; } else endGame(); } break; } } }
    for(let i=0;i<pickups.length;i++){ const p=pickups[i]; if(Math.abs(p.x-px)<(p.w+pw)/2 && Math.abs(p.y-py)<(p.h+ph)/2){ pickups.splice(i,1); if(p.type==='dash'){ startDash(); cheatCharges=Math.min(2, cheatCharges+1); cheatToastText='DASH + Shield +1'; cheatToastTimer=1.2; spawnSparkles(px,py,'rgba(255,240,130,.95)'); } else if(p.type==='chicken'){ fuel=Math.min(100, fuel+25); score+=(dashActive?30:15); cheatCharges=Math.min(2, cheatCharges+1); cheatToastText='Shield +1'; cheatToastTimer=1.2; spawnSparkles(px,py,'rgba(255,230,140,.95)'); } else if(p.type==='lizard'){ fuel=Math.min(100, fuel+(p.golden?22:12)); score+=(p.golden?12:5)*(dashActive?DASH_SCORE_MUL:1); spawnSparkles(px,py,'rgba(180,255,120,.95)'); } else if(p.type==='bird'){ fuel=Math.min(100, fuel+(p.golden?20:10)); score+=(p.golden?10:3)*(dashActive?DASH_SCORE_MUL:1); spawnSparkles(px,py,'rgba(180,210,255,.95)'); } else { fuel=Math.min(100, fuel+(p.golden?20:10)); score+=(p.golden?10:3)*(dashActive?DASH_SCORE_MUL:1); spawnSparkles(px,py,'rgba(255,230,150,.95)'); } break; } }
    updateDash(dt); updateParticles(dt); if(cheatToastTimer>0) cheatToastTimer=Math.max(0, cheatToastTimer-dt); meters+=(actual*dt)/120; fuel-=dt*(dashActive?2.4:2.0); if(fuel<=0) endGame(); }
  function draw(){ if(mode==='front'){ frontT+=.016; drawFront(); return; } if(mode==='menu'){ drawMenu(); return; } drawBackground(); enemies.forEach(e=>{ if(e.type==='tree') drawTree(e.x,e.y,e.w,e.h); else drawMud(e.x,e.y,e.w,e.h); }); pickups.forEach(p=>{ if(p.type==='mouse') drawMouse(p.x,p.y,p.scale,p.golden); else if(p.type==='bird') drawBird(p.x,p.y,p.scale,p.golden); else if(p.type==='lizard') drawLizard(p.x,p.y,p.scale,p.golden); else if(p.type==='chicken') drawChicken(p.x,p.y,p.scale); else if(p.type==='dash') drawLightning(p.x,p.y,p.scale); }); drawCat(lanesX()[currentLane]+slipOffset, PLAYER_Y(), CAT_W, CAT_H); drawParticles(); drawHUD(); drawDashOverlay(); drawVignette(); }

  /* ========================= Flow ========================= */
  function drawMenu(){ drawFrontBackground(); ctx.fillStyle='#fff'; ctx.font='14px system-ui, sans-serif'; const sub='Collect snacks • Avoid trees & mud • Grab ⚡ for Dash'; ctx.fillText(sub,(W-ctx.measureText(sub).width)/2,70); drawGlobalBoard(18,150,14); drawVignette(); }
  function endGame(){ mode='menu'; const n=getPlayerName(); submitBestIfHigher(n, Math.round(score)); restartDelay=1.0; }
  function resetGame(){ enemies=[]; pickups=[]; particles=[]; score=0; fuel=100; meters=0; roadSpeed=226; currentLane=1; spawnTimer=0; graceTimer=.75; last=undefined; cheatArmTimerMs=0; cheatCharges=0; cheatRearmLock=false; cheatToastTimer=0; cheatToastText=''; dashActive=false; dashTimer=0; restartDelay=0; initFlowerSpots(); }
  async function goFullscreen(){ try{ const el=document.documentElement; if(!document.fullscreenElement && el.requestFullscreen){ await el.requestFullscreen(); } }catch(e){} }
  function startGame(){ resetGame(); mode='game'; }
  function loop(ts){ if(last===undefined) last=ts; let dt=(ts-last)/1000; if(!Number.isFinite(dt)||dt<0) dt=0; dt=Math.min(dt,.05); last=ts; if(mode==='game'){ if(restartDelay>0) restartDelay=Math.max(0,restartDelay-dt); if(graceTimer>0) graceTimer=Math.max(0,graceTimer-dt); update(dt); controls.style.display='block'; frontStack.style.display='none'; menuStack.style.display='none'; } else if(mode==='menu'){ controls.style.display='none'; frontStack.style.display='none'; menuStack.style.display='block'; } else { controls.style.display='none'; frontStack.style.display='block'; menuStack.style.display='none'; } draw(); requestAnimationFrame(loop); }

  /* ========================= Inputs ========================= */
  const menuStack=document.getElementById('menuStack');
  const startBtn=document.getElementById('startBtn');
  const frontStack=document.getElementById('frontStack');
  const playBtn=document.getElementById('playBtn');
  const frontChangeNameBtn=document.getElementById('frontChangeNameBtn');
  playBtn.addEventListener('click', async()=>{ await goFullscreen(); startGame(); });
  frontChangeNameBtn.addEventListener('click', ()=>{ localStorage.removeItem(NAME_KEY); const n=getPlayerName(); alert('Player name set to: '+n); });
  startBtn.addEventListener('click', async()=>{ await goFullscreen(); startGame(); });
  let keyLock=false; document.addEventListener('keydown', async e=>{ if(mode!=='game'){ if(e.key===' '||e.key==='ArrowLeft'||e.key==='ArrowRight'){ await goFullscreen(); startGame(); return; } } if(keyLock) return; if(e.key==='ArrowLeft'){ if(currentLane>0) currentLane--; keyLock=true; } else if(e.key==='ArrowRight'){ if(currentLane<2) currentLane++; keyLock=true; } }); document.addEventListener('keyup', e=>{ if(e.key==='ArrowLeft'||e.key==='ArrowRight') keyLock=false; });
  let touchStartX=null; canvas.addEventListener('touchstart', e=>{ if(mode!=='game') return; touchStartX=e.touches[0].clientX; }, {passive:true}); canvas.addEventListener('touchmove', e=>{ if(mode!=='game'||touchStartX===null) return; const dx=e.touches[0].clientX - touchStartX; if(dx>50 && currentLane<2){ currentLane++; touchStartX=e.touches[0].clientX; } else if(dx<-50 && currentLane>0){ currentLane--; touchStartX=e.touches[0].clientX; } }, {passive:true}); canvas.addEventListener('touchend', ()=>{ touchStartX=null; });
  const btn1=document.getElementById('btnLane1'); const btn3=document.getElementById('btnLane3');
  function nudgeLeft(){ if(currentLane>0) currentLane--; } function nudgeRight(){ if(currentLane<2) currentLane++; }
  function onPointerDownBtn1(e){ e.preventDefault(); btn1Down=true; if(mode==='game') nudgeLeft(); }
  function onPointerUpBtn1(e){ e.preventDefault(); btn1Down=false; if(!btn3Down && cheatCharges===0) cheatArmTimerMs=0; }
  function onPointerDownBtn3(e){ e.preventDefault(); btn3Down=true; if(mode==='game') nudgeRight(); }
  function onPointerUpBtn3(e){ e.preventDefault(); btn3Down=false; if(!btn1Down && cheatCharges===0) cheatArmTimerMs=0; }
  ['pointerdown'].forEach(evt=>{ btn1.addEventListener(evt,onPointerDownBtn1,{passive:false}); btn3.addEventListener(evt,onPointerDownBtn3,{passive:false}); });
  ['pointerup','pointercancel','pointerout','pointerleave'].forEach(evt=>{ btn1.addEventListener(evt,onPointerUpBtn1,{passive:false}); btn3.addEventListener(evt,onPointerUpBtn3,{passive:false}); });

  /* Boot */
  fetchGlobalTop(20);
  requestAnimationFrame(loop);
  </script>
</body>
</html>
