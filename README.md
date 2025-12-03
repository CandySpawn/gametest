# gametest
Game Test
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>The Yellow Cat</title>
<style>
  :root {
    --sky-day: #bfe9ff;
    --sky-night: #6aa0bf;
    --grass: #7ec850;
    --hud-bg: rgba(255,255,255,0.85);
    --hud-text: #273043;
    --cat: #f7d23e;
    --cat-dark: #e07d93;
    --ladybug: #2b2b2b;
    --apple: #e14b3b;
    --shadow: rgba(0,0,0,0.15);
  }
  html, body {
    margin: 0; height: 100%; background: var(--sky-day); font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif;
  }
  .wrap {
    display: grid; place-items: center; height: 100%;
  }
  canvas {
    box-shadow: 0 10px 30px var(--shadow);
    border-radius: 12px;
    background: linear-gradient(var(--sky-day), var(--sky-night));
  }
  .hud {
    position: absolute; top: 16px; left: 50%; transform: translateX(-50%);
    display: flex; gap: 16px; padding: 8px 12px; border-radius: 10px;
    background: var(--hud-bg); color: var(--hud-text); font-weight: 600;
  }
  .hud span { min-width: 90px; display: inline-flex; gap: 8px; align-items: center; justify-content: center; }
  .panel {
    position: absolute; inset: 0; display: grid; place-items: center; pointer-events: none;
  }
  .card {
    pointer-events: auto; background: var(--hud-bg); padding: 18px 20px; border-radius: 12px; box-shadow: 0 10px 24px var(--shadow);
    text-align: center; color: var(--hud-text); max-width: 340px;
  }
  .card h1 { font-size: 22px; margin: 0 0 8px; }
  .card p { margin: 6px 0; }
  .btns { display: flex; gap: 10px; justify-content: center; margin-top: 12px; }
  button {
    border: none; background: #3a86ff; color: white; padding: 10px 14px; border-radius: 10px; font-weight: 700; cursor: pointer;
  }
  button.secondary { background: #adb5bd; color: #1c1f23; }
  .footer { position: absolute; bottom: 10px; left: 50%; transform: translateX(-50%); font-size: 12px; color: #1c1f23; opacity: 0.7; }
</style>
</head>
<body>
<div class="wrap">
  <div class="hud">
    <span>Score: <strong id="score">0</strong></span>
    <span>Lives: <strong id="lives">3</strong></span>
    <span>Best: <strong id="best">0</strong></span>
  </div>
  <canvas id="game" width="800" height="450" aria-label="Little cat Meadow"></canvas>
  <div class="panel" id="panel">
    <div class="card">
      <h1>Little cat Meadow</h1>
      <p>Collect apples, avoid ladybugs, and enjoy a gentle stroll.</p>
      <p>Controls: ← → move, Space to jump</p>
      <div class="btns">
        <button id="play">Play</button>
        <button id="mute" class="secondary">Sound On</button>
      </div>
    </div>
  </div>
  <div class="footer">© Your Studio</div>
</div>

<script>
(() => {
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const W = canvas.width, H = canvas.height;
  const scoreEl = document.getElementById('score');
  const livesEl = document.getElementById('lives');
  const bestEl = document.getElementById('best');
  const panel = document.getElementById('panel');
  const btnPlay = document.getElementById('play');
  const btnMute = document.getElementById('mute');

  // Game state
  const state = {
    playing: false,
    t: 0,
    score: 0,
    lives: 3,
    best: Number(localStorage.getItem('lp_best') || 0),
    gravity: 0.65,
    muted: false,
    dayNight: 0
  };
  bestEl.textContent = state.best;

  // Input
  const keys = { left: false, right: false, jump: false };
  window.addEventListener('keydown', (e) => {
    if (e.code === 'ArrowLeft') keys.left = true;
    if (e.code === 'ArrowRight') keys.right = true;
    if (e.code === 'Space') keys.jump = true;
  });
  window.addEventListener('keyup', (e) => {
    if (e.code === 'ArrowLeft') keys.left = false;
    if (e.code === 'ArrowRight') keys.right = false;
    if (e.code === 'Space') keys.jump = false;
  });

  // Simple sound generator (ladybugps)
  let audioCtx;
  function ladybugp(freq=440, dur=0.08, vol=0.03){
    if (state.muted) return;
    audioCtx = audioCtx || new (window.AudioContext || window.webkitAudioContext)();
    const o = audioCtx.createOscillator();
    const g = audioCtx.createGain();
    o.type = 'sine';
    o.frequency.value = freq;
    g.gain.value = vol;
    o.connect(g).connect(audioCtx.destination);
    o.start();
    o.stop(audioCtx.currentTime + dur);
  }

  // World
  const groundY = H - 80;
  const cat = {
    x: W/2, y: groundY, w: 40, h: 28,
    vx: 0, vy: 0, speed: 3.2,
    onGround: true, invuln: 0
  };

  const particles = [];
  const apples = [];
  const ladybugs = [];
  const clouds = Array.from({length: 6}, (_,i)=>({
    x: Math.random()*W, y: 30 + i*35 + Math.random()*25, s: 60 + Math.random()*60, vx: 0.2 + Math.random()*0.5
  }));

  function spawnApple(){
    apples.push({
      x: Math.random()*(W-80)+40, y: groundY - (30+Math.random()*80),
      r: 8, taken: false
    });
  }
  function spawnladybug(){
    const left = Math.random() < 0.5;
    ladybugs.push({
      x: left ? -40 : W+40,
      y: groundY - (25 + Math.random()*110),
      w: 26, h: 16,
      vx: (1.6 + Math.random()*1.4) * (left ? 1 : -1),
      sway: Math.random()*Math.PI*2
    });
  }
  for (let i=0;i<4;i++) spawnApple();
  for (let i=0;i<2;i++) spawnladybug();

  // Helpers
  function rectsOverlap(a,b){
    return (Math.abs((a.x+a.w/2)-(b.x+b.w/2)) < (a.w+b.w)/2) &&
           (Math.abs((a.y+a.h/2)-(b.y+b.h/2)) < (a.h+b.h)/2);
  }

  // Draw
  function drawBackground(){
    // Sky gradient animated subtly
    const g = ctx.createLinearGradient(0,0,0,H);
    const dayMix = 0.5 + 0.5*Math.sin(state.dayNight);
    const col1 = lerpColor([191,233,255],[106,160,191], dayMix); // day to dusk
    const col2 = lerpColor([231,245,255],[96,128,160], dayMix);
    g.addColorStop(0, `rgb(${col1.join(',')})`);
    g.addColorStop(1, `rgb(${col2.join(',')})`);
    ctx.fillStyle = g; ctx.fillRect(0,0,W,H);

    // Sun
    const sunX = (W/2) + Math.sin(state.dayNight)*W*0.35;
    const sunY = 80 + 20*Math.cos(state.dayNight);
    ctx.fillStyle = 'rgba(255,210,120,0.9)';
    ctx.beginPath(); ctx.arc(sunX, sunY, 22, 0, Math.PI*2); ctx.fill();

    // Clouds
    clouds.forEach(c=>{
      c.x += c.vx; if (c.x > W+80) c.x = -80;
      drawCloud(c.x, c.y, c.s);
    });

    // Hills
    drawHill(0.0015, 35, '#87d46c');
    drawHill(0.0023, 20, '#75c058');

    // Grass
    ctx.fillStyle = '#77c857';
    ctx.fillRect(0, groundY, W, H-groundY);
    // Tufts
    for(let i=0;i<35;i++){
      const x = i*24 + (state.t%24);
      const h = 8 + Math.sin(i*0.7 + state.t*0.05)*2;
      ctx.strokeStyle = '#5aa63e';
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(x, groundY);
      ctx.lineTo(x+2, groundY-h);
      ctx.stroke();
    }
  }
  function drawCloud(x,y,s){
    ctx.fillStyle = 'rgba(255,255,255,0.9)';
    blob(x,y,s); blob(x+s*0.3,y-10,s*0.8); blob(x-s*0.25,y-4,s*0.7);
  }
  function blob(x,y,s){
    ctx.beginPath();
    ctx.moveTo(x,y);
    for(let i=0;i<10;i++){
      const ang = (i/10)*Math.PI*2;
      const r = s*0.25 + Math.sin(i*1.7)*s*0.05;
      ctx.quadraticCurveTo(x + Math.cos(ang)*r, y + Math.sin(ang)*r, x + Math.cos(ang+0.2)*r, y + Math.sin(ang+0.2)*r);
    }
    ctx.fill();
  }
  function drawHill(freq, amp, color){
    ctx.fillStyle = color;
    ctx.beginPath();
    ctx.moveTo(0, H);
    for(let x=0; x<=W; x+=10){
      const y = groundY - 40 + Math.sin(x*freq + state.t*0.01)*amp;
      ctx.lineTo(x, y);
    }
    ctx.lineTo(W,H); ctx.closePath(); ctx.fill();
  }

  function drawcat(){
    // body
    ctx.fillStyle = '#f7d23e';
    roundRect(cat.x-20, cat.y-16, cat.w, cat.h, 8, true);
    // ear
    ctx.fillStyle = '#e07d93';
    roundRect(cat.x-8, cat.y-28, 12, 12, 3, true);
    // snout
    ctx.fillStyle = '#ffc3ce';
    roundRect(cat.x+8, cat.y-6, 14, 12, 5, true);
    ctx.fillStyle = '#d46b80';
    ctx.beginPath(); ctx.arc(cat.x+12, cat.y, 2, 0, Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(cat.x+16, cat.y, 2, 0, Math.PI*2); ctx.fill();
    // eye
    ctx.fillStyle = '#2b2b2b';
    ctx.beginPath(); ctx.arc(cat.x-2, cat.y-4, 2.2, 0, Math.PI*2); ctx.fill();
    // legs
    ctx.fillStyle = '#e07d93';
    roundRect(cat.x-16, cat.y+8, 8, 10, 3, true);
    roundRect(cat.x+4, cat.y+8, 8, 10, 3, true);
    // tail
    ctx.strokeStyle = '#e07d93'; ctx.lineWidth = 2;
    ctx.beginPath(); ctx.moveTo(cat.x-22, cat.y-2);
    ctx.quadraticCurveTo(cat.x-30, cat.y-10, cat.x-26, cat.y-14);
    ctx.stroke();

    if (cat.invuln > 0 && Math.floor(state.t*10)%2===0){
      ctx.fillStyle = 'rgba(255,255,255,0.4)';
      roundRect(cat.x-22, cat.y-18, cat.w+4, cat.h+4, 10, true);
    }
  }

  function roundRect(x,y,w,h,r,fill){
    ctx.beginPath();
    ctx.moveTo(x+r,y);
    ctx.arcTo(x+w,y,x+w,y+h,r);
    ctx.arcTo(x+w,y+h,x,y+h,r);
    ctx.arcTo(x,y+h,x,y,r);
    ctx.arcTo(x,y,x+w,y,r);
    if(fill) ctx.fill(); else ctx.stroke();
  }

  function drawApple(a){
    // fruit
    ctx.fillStyle = '#e14b3b';
    ctx.beginPath(); ctx.arc(a.x, a.y, a.r, 0, Math.PI*2); ctx.fill();
    // leaf
    ctx.fillStyle = '#3aa85b';
    ctx.beginPath();
    ctx.ellipse(a.x-4, a.y-9, 5, 3, -0.5, 0, Math.PI*2);
    ctx.fill();
    // highlight
    ctx.fillStyle = 'rgba(255,255,255,0.6)';
    ctx.beginPath(); ctx.arc(a.x+3, a.y-2, 2, 0, Math.PI*2); ctx.fill();
  }

  function drawladybug(b){
    const t = state.t;
    const sway = Math.sin(t*0.12 + b.sway)*3;
    b.x += b.vx; b.y += Math.sin(t*0.08 + b.sway)*0.8;
    ctx.save();
    ctx.translate(b.x, b.y+sway);
    // body
    ctx.fillStyle = '#2b2b2b';
    roundRect(-b.w/2, -b.h/2, b.w, b.h, 8, true);
    // stripes
    ctx.fillStyle = '#f7d23e';
    roundRect(-b.w/2+3, -b.h/2, 6, b.h, 6, true);
    roundRect(-b.w/2+13, -b.h/2, 6, b.h, 6, true);
    // wing
    ctx.fillStyle = 'rgba(200,230,255,0.75)';
    ctx.beginPath(); ctx.ellipse(0, -b.h/2, 6, 10, 0, 0, Math.PI*2); ctx.fill();
    ctx.restore();
  }

  function emitDust(x,y,count=4){
    for(let i=0;i<count;i++){
      particles.push({
        x, y, vx: (Math.random()-0.5)*2.2, vy: -Math.random()*1.6,
        life: 16 + Math.random()*12
      });
    }
  }
  function drawParticles(){
    particles.forEach(p=>{
      p.life -= 1;
      p.x += p.vx; p.y += p.vy;
      p.vy += 0.03;
      ctx.fillStyle = 'rgba(150,140,120,0.3)';
      ctx.beginPath(); ctx.arc(p.x, p.y, 2, 0, Math.PI*2); ctx.fill();
    });
    for(let i=particles.length-1;i>=0;i--) if(particles[i].life<=0) particles.splice(i,1);
  }

  // Update
  function updatecat(){
    cat.vx = 0;
    if (keys.left) cat.vx -= cat.speed;
    if (keys.right) cat.vx += cat.speed;
    cat.x += cat.vx;
    cat.x = Math.max(20, Math.min(W-20, cat.x));

    // Jump
    if (keys.jump && cat.onGround){
      cat.vy = -10.8;
      cat.onGround = false;
      emitDust(cat.x, groundY);
      ladybugp(600,0.06,0.04);
    }
    cat.y += cat.vy;
    cat.vy += state.gravity;

    if (cat.y >= groundY){
      cat.y = groundY;
      cat.vy = 0;
      if (!cat.onGround) emitDust(cat.x, groundY, 6);
      cat.onGround = true;
    }

    // Invulnerability decay
    if (cat.invuln > 0) cat.invuln -= 1;
  }

  function updateObjects(){
    // Apples
    for (let i=apples.length-1;i>=0;i--){
      const a = apples[i];
      // gentle bobbing
      a.y += Math.sin(state.t*0.1 + i)*0.2;
      const catBox = {x:cat.x-20,y:cat.y-16,w:cat.w,h:cat.h};
      const aBox = {x:a.x-a.r, y:a.y-a.r, w:a.r*2, h:a.r*2};
      if (!a.taken && rectsOverlap(catBox,aBox)){
        a.taken = true;
        state.score += 10;
        scoreEl.textContent = state.score;
        ladybugp(880,0.08,0.05);
        // respawn
        apples.splice(i,1);
        spawnApple();
      }
    }
    // ladybugs
    for (let i=ladybugs.length-1;i>=0;i--){
      const b = ladybugs[i];
      drawladybug(b);
      const catBox = {x:cat.x-20,y:cat.y-16,w:cat.w,h:cat.h};
      const bBox = {x:b.x-b.w/2, y:b.y-b.h/2, w:b.w, h:b.h};
      if (rectsOverlap(catBox,bBox) && cat.invuln<=0){
        state.lives -= 1;
        livesEl.textContent = state.lives;
        cat.invuln = 60;
        ladybugp(220,0.1,0.06);
        emitDust(cat.x, cat.y, 10);
      }
      // remove offscreen
      if (b.x < -60 || b.x > W+60){
        ladybugs.splice(i,1);
        spawnladybug();
      }
    }
    // Difficulty ramp: more ladybugs over time
    if (state.playing && state.t % 600 === 0 && ladybugs.length < 6) spawnladybug();
  }

  function lerpColor(a,b,t){
    return [
      Math.round(a[0] + (b[0]-a[0])*t),
      Math.round(a[1] + (b[1]-a[1])*t),
      Math.round(a[2] + (b[2]-a[2])*t),
    ];
  }

  function drawHUD(){
    // HUD labels are native DOM; here we can add subtle vignette
    const grd = ctx.createRadialGradient(W/2, H/2, H*0.1, W/2, H/2, H*0.7);
    grd.addColorStop(0, 'rgba(0,0,0,0)');
    grd.addColorStop(1, 'rgba(0,0,0,0.05)');
    ctx.fillStyle = grd;
    ctx.fillRect(0,0,W,H);
  }

  function gameOver(){
    state.playing = false;
    panel.style.display = 'grid';
    panel.querySelector('h1').textContent = 'Game Over';
    const pEls = panel.querySelectorAll('p');
    pEls[0].textContent = `Final Score: ${state.score}`;
    pEls[1].textContent = 'Press Play to try again';
    if (state.score > state.best){
      state.best = state.score;
      localStorage.setItem('lp_best', String(state.best));
    }
    bestEl.textContent = state.best;
  }

  function reset(){
    state.t = 0;
    state.score = 0;
    scoreEl.textContent = state.score;
    state.lives = 3;
    livesEl.textContent = state.lives;
    cat.x = W/2; cat.y = groundY; cat.vx = 0; cat.vy = 0; cat.onGround = true; cat.invuln = 0;
    apples.splice(0, apples.length);
    ladybugs.splice(0, ladybugs.length);
    for (let i=0;i<5;i++) spawnApple();
    for (let i=0;i<3;i++) spawnladybug();
  }

  function loop(){
    ctx.clearRect(0,0,W,H);
    state.t++;
    state.dayNight += 0.002;

    drawBackground();
    updatecat();
    drawcat();
    updateObjects();
    drawParticles();
    drawHUD();

    if (state.lives <= 0) gameOver();

    if (state.playing) requestAnimationFrame(loop);
  }

  btnPlay.addEventListener('click', () => {
    reset();
    panel.style.display = 'none';
    state.playing = true;
    requestAnimationFrame(loop);
  });

  btnMute.addEventListener('click', () => {
    state.muted = !state.muted;
    btnMute.textContent = state.muted ? 'Sound Off' : 'Sound On';
  });

  // Accessibility: pause when tab inactive
  document.addEventListener('visibilitychange', () => {
    if (document.hidden && state.playing){
      state.playing = false;
      panel.style.display = 'grid';
      panel.querySelector('h1').textContent = 'Paused';
      panel.querySelectorAll('p')[0].textContent = 'Game paused';
      panel.querySelectorAll('p')[1].textContent = 'Click Play to resume';
    }
  });
})();
</script>
</body>
</html>
