<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no" />
  <title>Dodge the Dots – Smooth</title>
  <style>
    html, body { margin:0; padding:0; height:100%; background:#070a10; overflow:hidden; font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; }
    canvas { display:block; width:100vw; height:100vh; touch-action:none; }
    #ui {
      position: fixed; left: 12px; top: 12px; right: 12px;
      display:flex; justify-content:space-between; align-items:center; gap:12px;
      color:#eaf2ff; font-weight:700; font-size:14px;
      pointer-events:none; text-shadow: 0 2px 10px rgba(0,0,0,.55);
      user-select:none;
    }
    #pill {
      display:flex; gap:10px; align-items:center;
      padding:8px 10px; border-radius: 999px;
      background: rgba(255,255,255,0.06);
      border: 1px solid rgba(255,255,255,0.10);
      backdrop-filter: blur(6px);
    }
    #banner {
      position: fixed; left: 50%; top: 50%; transform: translate(-50%,-50%);
      color:#eaf2ff; text-align:center; padding:16px 18px;
      border:1px solid rgba(255,255,255,.14);
      background: rgba(9,12,18,.78);
      border-radius: 16px; width:min(340px, 92vw);
      box-shadow: 0 12px 38px rgba(0,0,0,.55);
      user-select:none;
    }
    #banner h1 { margin:0 0 6px 0; font-size:18px; letter-spacing:0.2px; }
    #banner p { margin:6px 0; font-size:13px; opacity:.92; line-height:1.35; }
    #banner button {
      margin-top:10px; pointer-events:auto;
      border:0; border-radius: 12px; padding:12px 12px; font-weight:800;
      background: linear-gradient(135deg, #2f7cff, #6a5cff);
      color:white; width:100%;
    }
    #hint { opacity:.85; font-weight:600; }
  </style>
</head>
<body>
  <div id="ui">
    <div id="pill">
      <div id="score">Score: 0</div>
      <div id="sep">•</div>
      <div id="best">Best: 0</div>
    </div>
    <div id="hint">Drag to move</div>
  </div>

  <div id="banner">
    <h1>Dodge the Dots (Smooth)</h1>
    <p>Drag anywhere to move the blue orb. Avoid red dots.</p>
    <p>Survive to increase difficulty.</p>
    <button id="startBtn">Start</button>
  </div>

  <canvas id="c"></canvas>

<script>
(() => {
  const canvas = document.getElementById("c");
  const ctx = canvas.getContext("2d", { alpha: true });

  const scoreEl = document.getElementById("score");
  const bestEl  = document.getElementById("best");
  const banner  = document.getElementById("banner");
  const startBtn= document.getElementById("startBtn");

  // ---------- Resize / DPR ----------
  let W = innerWidth, H = innerHeight, DPR = 1;
  function resize() {
    DPR = Math.max(1, Math.min(2, window.devicePixelRatio || 1));
    W = innerWidth; H = innerHeight;
    canvas.width  = Math.floor(W * DPR);
    canvas.height = Math.floor(H * DPR);
    ctx.setTransform(DPR,0,0,DPR,0,0);
    buildStars();
  }
  addEventListener("resize", resize, { passive:true });

  // ---------- Persistent best score ----------
  const bestKey = "dodge_best_smooth_v1";
  let best = Number(localStorage.getItem(bestKey) || 0);
  bestEl.textContent = `Best: ${best}`;

  // ---------- Background: Parallax stars ----------
  let stars = [];
  function buildStars() {
    stars = [];
    const layers = 3;
    for (let L=0; L<layers; L++) {
      const count = Math.floor((W*H) / (L===0 ? 12000 : L===1 ? 18000 : 26000)) + 40;
      for (let i=0; i<count; i++) {
        stars.push({
          x: Math.random()*W,
          y: Math.random()*H,
          r: (L===0 ? 1.6 : L===1 ? 1.2 : 0.9) * (0.6 + Math.random()*0.9),
          v: (L===0 ? 18 : L===1 ? 10 : 6) * (0.7 + Math.random()*0.8),
          a: (L===0 ? 0.55 : L===1 ? 0.40 : 0.28) * (0.6 + Math.random()*0.8),
          layer: L
        });
      }
    }
  }

  // ---------- Helpers ----------
  const rand = (a,b) => Math.random()*(b-a)+a;
  const clamp = (v,a,b) => Math.max(a, Math.min(b, v));

  // ---------- Game state ----------
  const state = {
    running:false,
    t:0,
    score:0,
    difficulty:1,
    spawnTimer:0,
    spawnEvery:0.95,
    dots:[],
    // nicer, smoother movement: critically damped spring
    player:{
      x: W*0.5, y: H*0.72,
      vx: 0, vy: 0,
      r: 18,
      targetX: null, targetY: null
    },
    // for subtle motion blur / trails
    trail: []
  };

  function reset() {
    state.t = 0;
    state.score = 0;
    state.difficulty = 1;
    state.spawnTimer = 0;
    state.spawnEvery = 0.95;
    state.dots.length = 0;
    state.player.x = W*0.5; state.player.y = H*0.72;
    state.player.vx = 0; state.player.vy = 0;
    state.player.targetX = null; state.player.targetY = null;
    state.trail.length = 0;
    scoreEl.textContent = "Score: 0";
  }

  function collide(ax,ay,ar, bx,by,br) {
    const dx = ax-bx, dy = ay-by;
    return (dx*dx + dy*dy) < (ar+br)*(ar+br);
  }

  function spawnDot() {
    const edge = (Math.random()*4)|0;
    let x,y;
    if (edge===0) { x = rand(0,W); y = -20; }
    if (edge===1) { x = W+20; y = rand(0,H); }
    if (edge===2) { x = rand(0,W); y = H+20; }
    if (edge===3) { x = -20; y = rand(0,H); }

    const baseSpeed = 85 + 18*state.difficulty + rand(0,45);
    const px = state.player.x, py = state.player.y;
    const dx = (px - x) + rand(-140, 140);
    const dy = (py - y) + rand(-140, 140);
    const mag = Math.hypot(dx,dy) || 1;

    const r = rand(8, 14);
    state.dots.push({
      x,y,r,
      vx:(dx/mag)*baseSpeed,
      vy:(dy/mag)*baseSpeed,
      spin: rand(-2,2),
      wob: rand(0, Math.PI*2)
    });
  }

  function gameOver() {
    state.running = false;
    const s = Math.floor(state.score);
    if (s > best) {
      best = s;
      localStorage.setItem(bestKey, String(best));
      bestEl.textContent = `Best: ${best}`;
    }
    banner.querySelector("h1").textContent = "Game Over";
    banner.querySelectorAll("p")[0].textContent = `Score: ${s}  |  Best: ${best}`;
    banner.querySelectorAll("p")[1].textContent = "Tap Start to try again.";
    startBtn.textContent = "Start";
    banner.style.display = "block";
  }

  // ---------- Controls ----------
  let dragging = false;
  function setTarget(x,y){ state.player.targetX = x; state.player.targetY = y; }

  canvas.addEventListener("pointerdown", (e) => {
    dragging = true;
    canvas.setPointerCapture(e.pointerId);
    setTarget(e.clientX, e.clientY);
  }, { passive:false });

  canvas.addEventListener("pointermove", (e) => {
    if (!dragging) return;
    setTarget(e.clientX, e.clientY);
  }, { passive:false });

  canvas.addEventListener("pointerup", () => { dragging = false; }, { passive:true });
  canvas.addEventListener("pointercancel", () => { dragging = false; }, { passive:true });

  startBtn.addEventListener("click", () => {
    reset();
    banner.style.display = "none";
    state.running = true;
  });

  // ---------- Main loop ----------
  let last = performance.now();
  function loop(now) {
    const dt = Math.min(0.033, (now-last)/1000);
    last = now;

    // smooth background always
    updateBackground(dt);
    if (state.running) update(dt);
    render();

    requestAnimationFrame(loop);
  }

  function updateBackground(dt) {
    // Stars drift downward; parallax by layer
    for (const s of stars) {
      s.y += s.v * dt;
      if (s.y > H + 10) { s.y = -10; s.x = Math.random()*W; }
    }
  }

  function update(dt) {
    state.t += dt;
    state.score += dt * 12; // slightly faster scoring
    scoreEl.textContent = `Score: ${Math.floor(state.score)}`;

    state.difficulty = 1 + state.t / 7.5;
    state.spawnEvery = Math.max(0.22, 0.95 - state.t / 20);

    // spawn logic
    state.spawnTimer += dt;
    while (state.spawnTimer >= state.spawnEvery) {
      state.spawnTimer -= state.spawnEvery;
      spawnDot();
      if (state.difficulty > 3 && Math.random() < 0.28) spawnDot();
    }

    // player: critically damped spring (smooth + responsive)
    const p = state.player;
    if (p.targetX == null) { p.targetX = p.x; p.targetY = p.y; }

    const stiffness = 70;          // higher = snappier
    const damping   = 2*Math.sqrt(stiffness); // near-critical damping

    const ax = stiffness*(p.targetX - p.x) - damping*p.vx;
    const ay = stiffness*(p.targetY - p.y) - damping*p.vy;

    p.vx += ax * dt;
    p.vy += ay * dt;
    p.x  += p.vx * dt;
    p.y  += p.vy * dt;

    p.x = clamp(p.x, p.r, W - p.r);
    p.y = clamp(p.y, p.r, H - p.r);

    // trail
    state.trail.push({ x:p.x, y:p.y, r:p.r, a:0.35 });
    if (state.trail.length > 18) state.trail.shift();
    for (const t of state.trail) t.a *= Math.pow(0.02, dt);

    // dots
    for (let i = state.dots.length-1; i>=0; i--) {
      const d = state.dots[i];
      d.wob += dt * (1.2 + 0.25*state.difficulty);
      d.x += d.vx * dt;
      d.y += d.vy * dt;
      // gentle wobble for feel
      d.x += Math.cos(d.wob) * d.spin * 0.6;
      d.y += Math.sin(d.wob) * d.spin * 0.6;

      if (d.x < -120 || d.x > W+120 || d.y < -120 || d.y > H+120) {
        state.dots.splice(i,1);
        continue;
      }
      if (collide(p.x,p.y,p.r, d.x,d.y,d.r)) {
        gameOver();
        return;
      }
    }
  }

  // ---------- Render ----------
  function render() {
    // subtle motion blur by drawing a translucent rect rather than clearing hard
    ctx.fillStyle = "rgba(7,10,16,0.28)";
    ctx.fillRect(0,0,W,H);

    drawNebula();
    drawStars();
    drawDots();
    drawPlayer();
  }

  function drawNebula() {
    // soft background glow blobs
    const g1 = ctx.createRadialGradient(W*0.25, H*0.25, 10, W*0.25, H*0.25, Math.max(W,H)*0.7);
    g1.addColorStop(0, "rgba(80,110,255,0.12)");
    g1.addColorStop(1, "rgba(0,0,0,0)");
    ctx.fillStyle = g1; ctx.fillRect(0,0,W,H);

    const g2 = ctx.createRadialGradient(W*0.78, H*0.62, 10, W*0.78, H*0.62, Math.max(W,H)*0.75);
    g2.addColorStop(0, "rgba(130,90,255,0.10)");
    g2.addColorStop(1, "rgba(0,0,0,0)");
    ctx.fillStyle = g2; ctx.fillRect(0,0,W,H);
  }

  function drawStars() {
    for (const s of stars) {
      ctx.globalAlpha = s.a;
      ctx.beginPath();
      ctx.arc(s.x, s.y, s.r, 0, Math.PI*2);
      ctx.fillStyle = "white";
      ctx.fill();
    }
    ctx.globalAlpha = 1;
  }

  function drawDots() {
    for (const d of state.dots) {
      // dot glow
      ctx.beginPath();
      ctx.arc(d.x, d.y, d.r*2.2, 0, Math.PI*2);
      ctx.fillStyle = "rgba(255, 70, 70, 0.10)";
      ctx.fill();

      // dot core
      ctx.beginPath();
      ctx.arc(d.x, d.y, d.r, 0, Math.PI*2);
      ctx.fillStyle = "#ff3b3b";
      ctx.fill();
    }
  }

  function drawPlayer() {
    const p = state.player;

    // trail
    for (let i=0; i<state.trail.length; i++) {
      const t = state.trail[i];
      const a = t.a * (i/state.trail.length);
      if (a < 0.01) continue;
      ctx.globalAlpha = a;
      ctx.beginPath();
      ctx.arc(t.x, t.y, t.r*(0.85 + 0.5*i/state.trail.length), 0, Math.PI*2);
      ctx.fillStyle = "#2f7cff";
      ctx.fill();
    }
    ctx.globalAlpha = 1;

    // player glow
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.r*2.3, 0, Math.PI*2);
    ctx.fillStyle = "rgba(70,140,255,0.18)";
    ctx.fill();

    // player body
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.r, 0, Math.PI*2);
    ctx.fillStyle = "#2f7cff";
    ctx.fill();

    // highlight
    ctx.beginPath();
    ctx.arc(p.x - p.r*0.35, p.y - p.r*0.35, p.r*0.35, 0, Math.PI*2);
    ctx.fillStyle = "rgba(255,255,255,0.28)";
    ctx.fill();
  }

  // init
  resize();
  requestAnimationFrame(loop);
})();
</script>
</body>
</html>
