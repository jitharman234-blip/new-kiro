<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1.0,maximum-scale=1.0,user-scalable=no">
<title>Space Shooter</title>
<style>
*{margin:0;padding:0;box-sizing:border-box;}
body{
  background:#000;
  display:flex;flex-direction:column;
  align-items:center;justify-content:center;
  min-height:100vh;overflow:hidden;
  font-family:'Segoe UI',Arial,sans-serif;
  touch-action:none;user-select:none;
}
#wrap{
  position:relative;
  width:480px;height:720px;
  max-width:100vw;
  flex-shrink:0;
}
canvas{
  display:block;width:100%;height:100%;
  background:#000;
  border-left:2px solid #00cfff22;
  border-right:2px solid #00cfff22;
}
#mob{
  display:none;
  position:absolute;
  bottom:12px;left:0;right:0;
  flex-direction:row;
  align-items:flex-end;
  justify-content:space-between;
  padding:0 14px;
  pointer-events:none;
  z-index:20;
}
.bgrp{display:flex;gap:10px;pointer-events:all;}
.cb{
  background:rgba(0,180,255,0.13);
  border:2px solid rgba(0,207,255,0.55);
  color:#00cfff;font-size:1.6rem;
  width:64px;height:64px;border-radius:16px;
  cursor:pointer;
  touch-action:manipulation;
  display:flex;align-items:center;justify-content:center;
  -webkit-tap-highlight-color:transparent;
  backdrop-filter:blur(6px);
  transition:background .08s,transform .07s;
  pointer-events:all;
  -webkit-user-select:none;user-select:none;
}
.cb:active,.cb.held{background:rgba(0,207,255,0.38);transform:scale(0.9);}
#fbtn{
  background:rgba(255,200,0,0.13);
  border-color:rgba(255,210,0,0.65);
  color:#ffdc00;
  width:76px;height:76px;
  border-radius:50%;font-size:1.9rem;
  pointer-events:all;
}
#fbtn:active,#fbtn.held{background:rgba(255,210,0,0.42);transform:scale(0.9);}
@media(max-width:500px){
  #wrap{width:100vw;height:calc(100vw * 1.5);}
  #mob{display:flex;}
}
@media(max-height:740px) and (max-width:500px){
  #wrap{height:100vh;width:calc(100vh / 1.5);}
}
@media(orientation:landscape) and (max-height:500px){
  #wrap{height:98vh;width:calc(98vh / 1.5);}
  #mob{display:flex;}
}
</style>
</head>
<body>
<div id="wrap">
  <canvas id="gc" width="480" height="720"></canvas>
  <div id="mob">
    <div class="bgrp">
      <button class="cb" id="lbtn">&#8592;</button>
      <button class="cb" id="rbtn">&#8594;</button>
    </div>
    <button class="cb" id="fbtn">&#9650;</button>
  </div>
</div>
<script>
'use strict';
const canvas = document.getElementById('gc');
const ctx    = canvas.getContext('2d');
const W = 480, H = 720;

// ═══════════════════════════════════════════════════════════════
//  AUDIO ENGINE
// ═══════════════════════════════════════════════════════════════
let AC = null;
function ensureAC() {
  if (!AC) AC = new (window.AudioContext || window.webkitAudioContext)();
  if (AC.state === 'suspended') AC.resume();
}

function tone(freq, type, dur, vol, sweep) {
  if (!AC) return;
  const o = AC.createOscillator();
  const g = AC.createGain();
  const t = AC.currentTime;
  o.connect(g); g.connect(AC.destination);
  o.type = type;
  o.frequency.setValueAtTime(sweep || freq, t);
  if (sweep) o.frequency.exponentialRampToValueAtTime(freq, t + dur * 0.6);
  g.gain.setValueAtTime(vol, t);
  g.gain.exponentialRampToValueAtTime(0.0001, t + dur);
  o.start(t); o.stop(t + dur);
}

// noise burst for explosions
function noise(dur, vol) {
  if (!AC) return;
  const buf  = AC.createBuffer(1, AC.sampleRate * dur, AC.sampleRate);
  const data = buf.getChannelData(0);
  for (let i = 0; i < data.length; i++) data[i] = Math.random() * 2 - 1;
  const src  = AC.createBufferSource();
  const g    = AC.createGain();
  const t    = AC.currentTime;
  src.buffer = buf;
  src.connect(g); g.connect(AC.destination);
  g.gain.setValueAtTime(vol, t);
  g.gain.exponentialRampToValueAtTime(0.0001, t + dur);
  src.start(t); src.stop(t + dur);
}

const SFX = {
  shoot   : () => { tone(900, 'square',   0.10, 0.15, 500); },
  hit     : () => { tone(220, 'sawtooth', 0.18, 0.20, 700); noise(0.12, 0.12); },
  bossHit : () => { tone(140, 'sawtooth', 0.25, 0.28, 500); noise(0.15, 0.15); },
  bossDie : () => { tone(80,  'sawtooth', 0.55, 0.35, 300); noise(0.55, 0.30); },
  powerUp : () => { tone(1400,'sine',     0.40, 0.22, 700); tone(1800,'sine', 0.25, 0.15, 1400); },
  loseLife: () => { tone(220, 'square',   0.45, 0.28, 380); },
  levelUp : () => { tone(660, 'sine', 0.18, 0.20); tone(880,'sine',0.18,0.18); tone(1100,'sine',0.28,0.22); },
  bigShip : () => { tone(300, 'sine', 0.6, 0.25, 150); }
};

// ── BGM  ──────────────────────────────────────────────────────
// Two-layer: bass drone + melody arpeggio
const BASS  = [65, 65, 73, 73, 82, 82, 87, 87];
const MELO  = [261,293,329,349,392,440,494,523,494,440,392,349];
let bgmTick = 0, bgmMelo = 0;
let bgmTimer = null;

function startBGM() {
  if (!AC) return;
  stopBGM();
  bgmTick = 0; bgmMelo = 0;
  bgmTimer = setInterval(() => {
    if (!AC) return;
    // bass note every 2 ticks
    if (bgmTick % 2 === 0) {
      const o = AC.createOscillator(), g = AC.createGain();
      const t = AC.currentTime;
      o.connect(g); g.connect(AC.destination);
      o.type = 'triangle';
      o.frequency.value = BASS[(bgmTick/2) % BASS.length];
      g.gain.setValueAtTime(0.08, t);
      g.gain.exponentialRampToValueAtTime(0.0001, t + 0.55);
      o.start(t); o.stop(t + 0.6);
    }
    // melody note every tick
    {
      const o = AC.createOscillator(), g = AC.createGain();
      const t = AC.currentTime;
      o.connect(g); g.connect(AC.destination);
      o.type = 'sine';
      o.frequency.value = MELO[bgmMelo % MELO.length];
      g.gain.setValueAtTime(0.05, t);
      g.gain.exponentialRampToValueAtTime(0.0001, t + 0.22);
      o.start(t); o.stop(t + 0.25);
      bgmMelo++;
    }
    bgmTick++;
  }, 280);
}

function stopBGM() {
  if (bgmTimer) { clearInterval(bgmTimer); bgmTimer = null; }
}

// ═══════════════════════════════════════════════════════════════
//  STAR FIELD  (parallax 3-layer)
// ═══════════════════════════════════════════════════════════════
const STARS = [];
function initStars() {
  STARS.length = 0;
  for (let i = 0; i < 160; i++) {
    const layer = Math.floor(Math.random() * 3); // 0=far 1=mid 2=near
    STARS.push({
      x: Math.random() * W,
      y: Math.random() * H,
      r: [0.5, 1.1, 1.9][layer],
      spd: [0.18, 0.55, 1.2][layer],
      a: Math.random() * 0.5 + [0.2, 0.45, 0.7][layer],
      twinkle: Math.random() * Math.PI * 2
    });
  }
}

function drawStars(dt) {
  STARS.forEach(s => {
    s.y += s.spd;
    s.twinkle += 0.04;
    if (s.y > H + 2) { s.y = -2; s.x = Math.random() * W; }
    const a = s.a * (0.75 + 0.25 * Math.sin(s.twinkle));
    ctx.globalAlpha = a;
    ctx.fillStyle = '#fff';
    ctx.beginPath();
    ctx.arc(s.x, s.y, s.r, 0, Math.PI * 2);
    ctx.fill();
  });
  ctx.globalAlpha = 1;
}

// ═══════════════════════════════════════════════════════════════
//  DRAW PLAYER  (3D-style fighter jet look)
// ═══════════════════════════════════════════════════════════════
function drawPlayer(p) {
  ctx.save();
  ctx.translate(p.x, p.y);
  const w = p.w, h = p.h;

  // ── Engine exhaust flame ──
  const flameH = 18 + Math.random() * 10;
  const fg = ctx.createLinearGradient(0, h * 0.5, 0, h * 0.5 + flameH);
  fg.addColorStop(0,   'rgba(0,180,255,0.95)');
  fg.addColorStop(0.4, 'rgba(80,120,255,0.7)');
  fg.addColorStop(1,   'rgba(0,40,180,0)');
  ctx.fillStyle = fg;
  ctx.beginPath();
  ctx.moveTo(-w * 0.18, h * 0.45);
  ctx.lineTo(0, h * 0.5 + flameH);
  ctx.lineTo( w * 0.18, h * 0.45);
  ctx.closePath();
  ctx.fill();

  // ── Left wing ──
  const lwg = ctx.createLinearGradient(-w * 0.5, 0, 0, 0);
  lwg.addColorStop(0, '#0a1a3a');
  lwg.addColorStop(1, '#1a4a8a');
  ctx.fillStyle = lwg;
  ctx.beginPath();
  ctx.moveTo(-w * 0.08, -h * 0.1);
  ctx.lineTo(-w * 0.5,   h * 0.38);
  ctx.lineTo(-w * 0.12,  h * 0.42);
  ctx.lineTo(-w * 0.08,  h * 0.08);
  ctx.closePath();
  ctx.fill();
  // wing highlight
  ctx.strokeStyle = '#2a7aff';
  ctx.lineWidth = 1;
  ctx.stroke();

  // ── Right wing ──
  const rwg = ctx.createLinearGradient(0, 0, w * 0.5, 0);
  rwg.addColorStop(0, '#1a4a8a');
  rwg.addColorStop(1, '#0a1a3a');
  ctx.fillStyle = rwg;
  ctx.beginPath();
  ctx.moveTo( w * 0.08, -h * 0.1);
  ctx.lineTo( w * 0.5,   h * 0.38);
  ctx.lineTo( w * 0.12,  h * 0.42);
  ctx.lineTo( w * 0.08,  h * 0.08);
  ctx.closePath();
  ctx.fill();
  ctx.strokeStyle = '#2a7aff';
  ctx.lineWidth = 1;
  ctx.stroke();

  // ── Fuselage body ──
  const col = p.big ? '#00ffcc' : '#00cfff';
  const bg = ctx.createLinearGradient(-w * 0.15, -h * 0.5, w * 0.15, h * 0.5);
  bg.addColorStop(0,   p.big ? '#00ffcc' : '#00e0ff');
  bg.addColorStop(0.4, p.big ? '#007755' : '#005577');
  bg.addColorStop(1,   '#001122');
  ctx.fillStyle = bg;
  ctx.shadowColor = col;
  ctx.shadowBlur  = 14;
  ctx.beginPath();
  ctx.moveTo(0, -h * 0.5);          // nose tip
  ctx.bezierCurveTo(w*0.14, -h*0.2, w*0.16, h*0.1, w*0.12, h*0.44);
  ctx.lineTo(-w*0.12, h*0.44);
  ctx.bezierCurveTo(-w*0.16, h*0.1, -w*0.14, -h*0.2, 0, -h*0.5);
  ctx.closePath();
  ctx.fill();

  // fuselage centre stripe
  ctx.shadowBlur = 0;
  const sg = ctx.createLinearGradient(0, -h*0.45, 0, h*0.4);
  sg.addColorStop(0, 'rgba(255,255,255,0.55)');
  sg.addColorStop(1, 'rgba(255,255,255,0)');
  ctx.fillStyle = sg;
  ctx.beginPath();
  ctx.moveTo(0, -h*0.5);
  ctx.bezierCurveTo( w*0.04,-h*0.25,  w*0.04, h*0.1, 0, h*0.44);
  ctx.bezierCurveTo(-w*0.04, h*0.1, -w*0.04,-h*0.25, 0,-h*0.5);
  ctx.fill();

  // ── Cockpit glass ──
  const cg = ctx.createRadialGradient(-w*0.03,-h*0.22, 1, 0,-h*0.18, w*0.1);
  cg.addColorStop(0, 'rgba(180,240,255,0.95)');
  cg.addColorStop(1, 'rgba(0,80,160,0.5)');
  ctx.fillStyle = cg;
  ctx.beginPath();
  ctx.ellipse(0, -h*0.18, w*0.07, h*0.12, 0, 0, Math.PI*2);
  ctx.fill();

  // ── Wing-tip lights ──
  ctx.shadowColor = '#ff4444';
  ctx.shadowBlur  = 8;
  ctx.fillStyle   = '#ff4444';
  ctx.beginPath();
  ctx.arc(-w*0.48, h*0.32, 2.5, 0, Math.PI*2);
  ctx.fill();
  ctx.shadowColor = '#44ff44';
  ctx.fillStyle   = '#44ff44';
  ctx.beginPath();
  ctx.arc( w*0.48, h*0.32, 2.5, 0, Math.PI*2);
  ctx.fill();

  ctx.shadowBlur = 0;
  ctx.restore();
}

// ═══════════════════════════════════════════════════════════════
//  DRAW ENEMIES  (3D-style alien saucer)
// ═══════════════════════════════════════════════════════════════
function drawEnemy(e) {
  ctx.save();
  ctx.translate(e.x, e.y);
  const r = e.size / 2;

  // glow
  const gl = ctx.createRadialGradient(0, 0, r * 0.2, 0, 0, r * 1.6);
  gl.addColorStop(0, 'rgba(255,40,40,0.35)');
  gl.addColorStop(1, 'rgba(255,0,0,0)');
  ctx.fillStyle = gl;
  ctx.beginPath();
  ctx.arc(0, 0, r * 1.6, 0, Math.PI * 2);
  ctx.fill();

  // saucer rim (ellipse)
  const rimG = ctx.createLinearGradient(-r, -r*0.25, r, r*0.25);
  rimG.addColorStop(0, '#550000');
  rimG.addColorStop(0.5,'#cc2020');
  rimG.addColorStop(1, '#330000');
  ctx.fillStyle = rimG;
  ctx.shadowColor = '#ff3030';
  ctx.shadowBlur  = 12;
  ctx.beginPath();
  ctx.ellipse(0, 0, r, r * 0.38, 0, 0, Math.PI * 2);
  ctx.fill();

  // dome on top
  const dg = ctx.createRadialGradient(-r*0.2, -r*0.3, r*0.05, 0, -r*0.15, r*0.55);
  dg.addColorStop(0, 'rgba(255,120,120,0.9)');
  dg.addColorStop(1, 'rgba(120,0,0,0.7)');
  ctx.fillStyle = dg;
  ctx.shadowBlur = 0;
  ctx.beginPath();
  ctx.ellipse(0, -r*0.12, r*0.42, r*0.32, 0, Math.PI, 0, true);
  ctx.fill();

  // rim highlight
  ctx.strokeStyle = 'rgba(255,120,80,0.7)';
  ctx.lineWidth   = 1.5;
  ctx.beginPath();
  ctx.ellipse(0, 0, r * 0.85, r * 0.25, 0, Math.PI, 0, true);
  ctx.stroke();

  // underbelly lights
  const lc = ['#ff8800','#ff4400','#ffcc00'];
  for (let i = -1; i <= 1; i++) {
    ctx.fillStyle   = lc[i+1];
    ctx.shadowColor = lc[i+1];
    ctx.shadowBlur  = 6;
    ctx.beginPath();
    ctx.arc(i * r * 0.38, r * 0.1, 2.5, 0, Math.PI * 2);
    ctx.fill();
  }

  ctx.shadowBlur = 0;
  ctx.restore();
}

// ═══════════════════════════════════════════════════════════════
//  DRAW BOSS  (3D mothership)
// ═══════════════════════════════════════════════════════════════
function drawBoss(b) {
  if (!b) return;
  ctx.save();
  ctx.translate(b.x, b.y);
  const r = b.size / 2;

  // outer energy ring
  const pulse = 0.7 + 0.3 * Math.sin(Date.now() * 0.004);
  const ring = ctx.createRadialGradient(0, 0, r * 0.9, 0, 0, r * 1.7);
  ring.addColorStop(0,   `rgba(160,0,255,${0.4 * pulse})`);
  ring.addColorStop(0.6, `rgba(100,0,200,${0.2 * pulse})`);
  ring.addColorStop(1,   'rgba(80,0,160,0)');
  ctx.fillStyle = ring;
  ctx.beginPath();
  ctx.arc(0, 0, r * 1.7, 0, Math.PI * 2);
  ctx.fill();

  // main body hull — dark hexagonal feel via path
  const hull = ctx.createLinearGradient(-r, -r*0.6, r, r*0.6);
  hull.addColorStop(0,   '#1a0030');
  hull.addColorStop(0.35,'#6600cc');
  hull.addColorStop(0.6, '#440088');
  hull.addColorStop(1,   '#0d0020');
  ctx.fillStyle = hull;
  ctx.shadowColor = '#cc44ff';
  ctx.shadowBlur  = 22;
  // hexagonal hull shape
  ctx.beginPath();
  const sides = 6;
  for (let i = 0; i < sides; i++) {
    const a = (i / sides) * Math.PI * 2 - Math.PI / 2;
    const rx = Math.cos(a) * r, ry = Math.sin(a) * r * 0.6;
    i === 0 ? ctx.moveTo(rx, ry) : ctx.lineTo(rx, ry);
  }
  ctx.closePath();
  ctx.fill();

  // hull edge
  ctx.strokeStyle = '#cc44ff';
  ctx.lineWidth   = 2.5;
  ctx.stroke();

  // dome
  ctx.shadowBlur = 0;
  const dg = ctx.createRadialGradient(-r*0.15,-r*0.25, r*0.04, 0,-r*0.1, r*0.38);
  dg.addColorStop(0, 'rgba(220,150,255,0.95)');
  dg.addColorStop(1, 'rgba(80,0,160,0.6)');
  ctx.fillStyle = dg;
  ctx.beginPath();
  ctx.ellipse(0, -r*0.1, r*0.34, r*0.28, 0, Math.PI, 0, true);
  ctx.fill();

  // engine ports bottom
  for (let i = -2; i <= 2; i++) {
    const px = i * r * 0.32, py = r * 0.32;
    const eg = ctx.createRadialGradient(px, py, 1, px, py, 7);
    eg.addColorStop(0, '#ff88ff');
    eg.addColorStop(1, 'rgba(160,0,255,0)');
    ctx.fillStyle = eg;
    ctx.beginPath();
    ctx.arc(px, py, 7, 0, Math.PI * 2);
    ctx.fill();
  }

  // HP bar
  const bw = r * 2.2, bh = 9;
  const bx = -bw / 2, by = r + 10;
  ctx.fillStyle = 'rgba(0,0,0,0.6)';
  ctx.fillRect(bx - 1, by - 1, bw + 2, bh + 2);
  ctx.fillStyle = '#330033';
  ctx.fillRect(bx, by, bw, bh);
  const hpFrac = b.hp / b.maxHp;
  const hpG = ctx.createLinearGradient(bx, by, bx + bw * hpFrac, by);
  hpG.addColorStop(0, '#ff44ff');
  hpG.addColorStop(1, '#8800cc');
  ctx.fillStyle = hpG;
  ctx.fillRect(bx, by, bw * hpFrac, bh);
  ctx.strokeStyle = '#cc44ff';
  ctx.lineWidth   = 1;
  ctx.strokeRect(bx, by, bw, bh);

  // BOSS label
  ctx.fillStyle   = '#ff88ff';
  ctx.font        = 'bold 13px Arial';
  ctx.textAlign   = 'center';
  ctx.textBaseline = 'alphabetic';
  ctx.fillText('★ MOTHERSHIP ★', 0, by - 5);

  ctx.restore();
}

// ═══════════════════════════════════════════════════════════════
//  DRAW BULLET  (plasma bolt with 3D glow)
// ═══════════════════════════════════════════════════════════════
function drawBullet(b) {
  ctx.save();
  // outer glow
  const g = ctx.createRadialGradient(b.x, b.y, 0, b.x, b.y, 7);
  g.addColorStop(0, 'rgba(255,240,80,0.9)');
  g.addColorStop(1, 'rgba(255,160,0,0)');
  ctx.fillStyle = g;
  ctx.beginPath();
  ctx.arc(b.x, b.y, 7, 0, Math.PI * 2);
  ctx.fill();
  // core
  ctx.shadowColor = '#ffee44';
  ctx.shadowBlur  = 10;
  ctx.fillStyle   = '#fff9aa';
  ctx.fillRect(b.x - 2.5, b.y - 10, 5, 12);
  ctx.restore();
}

// ═══════════════════════════════════════════════════════════════
//  PARTICLES
// ═══════════════════════════════════════════════════════════════
let particles = [];
function spawnParticles(x, y, n, colors) {
  for (let i = 0; i < n; i++) {
    const a = Math.random() * Math.PI * 2;
    const s = Math.random() * 4 + 1;
    particles.push({
      x, y,
      vx: Math.cos(a) * s, vy: Math.sin(a) * s - Math.random() * 1.5,
      r: Math.random() * 4 + 1.5,
      color: colors[Math.floor(Math.random() * colors.length)],
      alpha: 1,
      decay: 0.016 + Math.random() * 0.024
    });
  }
}
function updateParticles() {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i];
    p.x += p.vx; p.y += p.vy; p.vy += 0.08;
    p.alpha -= p.decay;
    if (p.alpha <= 0) particles.splice(i, 1);
  }
}
function drawParticles() {
  particles.forEach(p => {
    ctx.save();
    ctx.globalAlpha = Math.max(0, p.alpha);
    ctx.shadowColor = p.color; ctx.shadowBlur = 6;
    ctx.fillStyle   = p.color;
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2);
    ctx.fill();
    ctx.restore();
  });
}

// ═══════════════════════════════════════════════════════════════
//  DRAW POWER-UP  (green orb)
// ═══════════════════════════════════════════════════════════════
function drawPowerUp(pu) {
  ctx.save();
  const sc = 1 + 0.12 * Math.sin(pu.pulse);
  ctx.translate(pu.x, pu.y);
  ctx.scale(sc, sc);
  const g = ctx.createRadialGradient(-pu.r*0.3,-pu.r*0.3,1,0,0,pu.r+8);
  g.addColorStop(0,'rgba(120,255,180,0.9)');
  g.addColorStop(0.5,'rgba(0,200,80,0.75)');
  g.addColorStop(1,'rgba(0,80,20,0)');
  ctx.fillStyle = g;
  ctx.shadowColor='#00ff88'; ctx.shadowBlur=16;
  ctx.beginPath(); ctx.arc(0,0,pu.r+8,0,Math.PI*2); ctx.fill();
  ctx.fillStyle='#00ff88';
  ctx.beginPath(); ctx.arc(0,0,pu.r,0,Math.PI*2); ctx.fill();
  ctx.fillStyle='#003310'; ctx.shadowBlur=0;
  ctx.font='bold 11px Arial'; ctx.textAlign='center'; ctx.textBaseline='middle';
  ctx.fillText('RF',0,0);
  ctx.restore();
}

// ═══════════════════════════════════════════════════════════════
//  DIFFICULTY CONFIG
// ═══════════════════════════════════════════════════════════════
const DIFF = {
  easy: {
    enemyBaseSpeed   : 0.9,
    enemySpawnMs     : 1400,
    enemySpeedVar    : 0.3,
    bossSpawnMs      : 35000,
    powerUpMs        : 15000,
    speedScalePerLvl : 0.25,
    spawnScalePerLvl : 90,
    label            : 'EASY',
    color            : '#44ff88'
  },
  medium: {
    enemyBaseSpeed   : 1.5,
    enemySpawnMs     : 1000,
    enemySpeedVar    : 0.6,
    bossSpawnMs      : 28000,
    powerUpMs        : 20000,
    speedScalePerLvl : 0.4,
    spawnScalePerLvl : 110,
    label            : 'MEDIUM',
    color            : '#ffdd00'
  },
  hard: {
    enemyBaseSpeed   : 2.5,
    enemySpawnMs     : 650,
    enemySpeedVar    : 1.0,
    bossSpawnMs      : 22000,
    powerUpMs        : 25000,
    speedScalePerLvl : 0.6,
    spawnScalePerLvl : 130,
    label            : 'HARD',
    color            : '#ff4444'
  }
};

// ═══════════════════════════════════════════════════════════════
//  GAME STATE
// ═══════════════════════════════════════════════════════════════
// screen: 'home' | 'playing' | 'gameover'
let screen       = 'home';
let difficulty   = 'medium';   // set on home screen
let score        = 0;
let highScore    = 0;
let lives        = 3;
let level        = 1;
let frameId      = null;

let player, bullets, enemies, boss, powerUps;
let lastShot, lastEnemySpawn, lastBossSpawn, lastPowerUpSpawn;
let rapidFire, rapidFireEnd;
let bigMode, bigModeHitUsed, survivalStart;
let enemySpeed, enemySpawnMs;
let levelFlash   = 0;          // timestamp of last level-up flash
let hitFlash     = 0;          // red screen flash when player hit

// home-screen hover tracking for difficulty buttons
const homeHover  = { diff: null };

// ═══════════════════════════════════════════════════════════════
//  INIT GAME
// ═══════════════════════════════════════════════════════════════
function initGame() {
  const d        = DIFF[difficulty];
  score          = 0;
  lives          = 3;
  level          = 1;
  bullets        = [];
  enemies        = [];
  boss           = null;
  powerUps       = [];
  particles      = [];
  rapidFire      = false;
  rapidFireEnd   = 0;
  bigMode        = false;
  bigModeHitUsed = false;
  levelFlash     = 0;
  hitFlash       = 0;
  enemySpeed     = d.enemyBaseSpeed;
  enemySpawnMs   = d.enemySpawnMs;

  const now      = performance.now();
  lastShot          = 0;
  lastEnemySpawn    = now;
  lastBossSpawn     = now;
  lastPowerUpSpawn  = now;
  survivalStart     = now;

  player = {
    x: W / 2, y: H - 90,
    w: 46, h: 50,
    speed: 5.5,
    big: false
  };

  if (!STARS.length) initStars();
  screen = 'playing';
  ensureAC();
  startBGM();
}

// ── Spawn helpers ─────────────────────────────────────────────
function spawnEnemy() {
  const d    = DIFF[difficulty];
  const size = 30 + Math.random() * 16;
  enemies.push({
    x    : size / 2 + 10 + Math.random() * (W - size - 20),
    y    : -size,
    size,
    speed: enemySpeed + Math.random() * d.enemySpeedVar,
    angle: 0
  });
}

function spawnBoss() {
  boss = {
    x: W / 2, y: -80,
    size: 110,
    hp: 5, maxHp: 5,
    speed: 0.85,
    dir: 1,
    hitFlash: 0
  };
}

function spawnPowerUp() {
  powerUps.push({
    x    : 30 + Math.random() * (W - 60),
    y    : -20,
    r    : 15,
    speed: 1.1,
    pulse: 0
  });
}

// ── Level scaling ─────────────────────────────────────────────
function checkLevelUp() {
  const d        = DIFF[difficulty];
  const newLevel = Math.floor(score / 100) + 1;
  if (newLevel > level) {
    level        = newLevel;
    enemySpeed   = d.enemyBaseSpeed + (level - 1) * d.speedScalePerLvl;
    enemySpawnMs = Math.max(300, d.enemySpawnMs - (level - 1) * d.spawnScalePerLvl);
    SFX.levelUp();
    levelFlash   = performance.now();
    spawnParticles(W / 2, H / 2, 50, ['#ffdd00','#ffffff','#00cfff','#ff88ff']);
  }
}

// ── Big-ship after 50 s survival ─────────────────────────────
function checkBigMode(now) {
  if (bigMode || screen !== 'playing') return;
  if (now - survivalStart >= 50000) {
    bigMode        = true;
    bigModeHitUsed = false;
    player.w       = 80;
    player.h       = 86;
    player.speed   = 4;
    player.big     = true;
    SFX.bigShip();
    spawnParticles(player.x, player.y, 40, ['#00ffcc','#ffffff','#00cfff']);
  }
}

// ── Collision helpers ─────────────────────────────────────────
function rectHit(ax,ay,aw,ah, bx,by,bw,bh) {
  return ax-aw/2 < bx+bw/2 && ax+aw/2 > bx-bw/2 &&
         ay-ah/2 < by+bh/2 && ay+ah/2 > by-bh/2;
}
function circRect(cx,cy,cr, rx,ry,rw,rh) {
  const nx = Math.max(rx-rw/2, Math.min(cx, rx+rw/2));
  const ny = Math.max(ry-rh/2, Math.min(cy, ry+rh/2));
  const dx = cx-nx, dy = cy-ny;
  return dx*dx+dy*dy < cr*cr;
}

// ═══════════════════════════════════════════════════════════════
//  INPUT
// ═══════════════════════════════════════════════════════════════
const keys = {};
document.addEventListener('keydown', e => {
  keys[e.code] = true;
  if (['ArrowLeft','ArrowRight','Space'].includes(e.code)) e.preventDefault();
  // keyboard shortcuts on game-over screen
  if (screen === 'gameover') {
    if (e.code === 'KeyR') { initGame(); }
    if (e.code === 'KeyH') { goHome(); }
  }
});
document.addEventListener('keyup', e => { keys[e.code] = false; });

// canvas click — used for home difficulty buttons & game-over buttons
canvas.addEventListener('click', onCanvasClick);
canvas.addEventListener('touchend', e => {
  e.preventDefault();
  const t   = e.changedTouches[0];
  const rect = canvas.getBoundingClientRect();
  const scaleX = W / rect.width;
  const scaleY = H / rect.height;
  onCanvasClick({ offsetX: (t.clientX - rect.left) * scaleX,
                  offsetY: (t.clientY - rect.top)  * scaleY });
}, { passive: false });

// track mouse for hover on home screen
canvas.addEventListener('mousemove', e => {
  const rect   = canvas.getBoundingClientRect();
  const scaleX = W / rect.width;
  const scaleY = H / rect.height;
  const mx = (e.clientX - rect.left) * scaleX;
  const my = (e.clientY - rect.top)  * scaleY;
  homeHover.diff = null;
  if (screen === 'home') {
    ['easy','medium','hard'].forEach((d, i) => {
      const bx = W/2 - 80, by = 390 + i * 70, bw = 160, bh = 44;
      if (mx>=bx && mx<=bx+bw && my>=by && my<=by+bh) homeHover.diff = d;
    });
  }
});

function onCanvasClick(e) {
  const mx = e.offsetX, my = e.offsetY;

  if (screen === 'home') {
    ['easy','medium','hard'].forEach((d, i) => {
      const bx = W/2 - 80, by = 390 + i * 70, bw = 160, bh = 44;
      if (mx>=bx && mx<=bx+bw && my>=by && my<=by+bh) {
        difficulty = d;
        ensureAC();
        initGame();
      }
    });
  }

  if (screen === 'gameover') {
    // Restart button
    if (mx >= W/2-100 && mx <= W/2+100 && my >= H/2+60 && my <= H/2+110) {
      initGame();
    }
    // Home button
    if (mx >= W/2-100 && mx <= W/2+100 && my >= H/2+125 && my <= H/2+175) {
      goHome();
    }
  }
}

function goHome() {
  stopBGM();
  screen   = 'home';
  particles = [];
  cancelAnimationFrame(frameId);
  requestAnimationFrame(loop);
}

// ── Shoot ─────────────────────────────────────────────────────
function shoot(now) {
  const cd = rapidFire ? 120 : 260;
  if (now - lastShot < cd) return;
  lastShot = now;
  SFX.shoot();
  bullets.push({ x: player.x, y: player.y - player.h/2 - 4, speed: 11 });
}

// ═══════════════════════════════════════════════════════════════
//  UPDATE  (called every frame while playing)
// ═══════════════════════════════════════════════════════════════
function update(now) {
  if (screen !== 'playing') return;

  // ── player movement ───────────────────────────────────────
  if (keys['ArrowLeft'])  player.x = Math.max(player.w/2,     player.x - player.speed);
  if (keys['ArrowRight']) player.x = Math.min(W - player.w/2, player.x + player.speed);
  if (keys['Space'])      shoot(now);

  // ── timers ────────────────────────────────────────────────
  if (rapidFire && now > rapidFireEnd) rapidFire = false;
  checkBigMode(now);

  // ── bullets ───────────────────────────────────────────────
  for (let i = bullets.length-1; i >= 0; i--) {
    bullets[i].y -= bullets[i].speed;
    if (bullets[i].y < -20) bullets.splice(i, 1);
  }

  // ── spawn enemies ─────────────────────────────────────────
  if (now - lastEnemySpawn > enemySpawnMs) {
    spawnEnemy();
    lastEnemySpawn = now;
  }

  // ── move enemies ──────────────────────────────────────────
  for (let i = enemies.length-1; i >= 0; i--) {
    const e = enemies[i];
    e.y     += e.speed;
    e.angle += 0.04;
    if (e.y - e.size/2 > H) {
      enemies.splice(i, 1);
      loseLife(now);
      if (screen !== 'playing') return;
    }
  }

  // ── boss spawn ────────────────────────────────────────────
  const d = DIFF[difficulty];
  if (!boss && now - lastBossSpawn >= d.bossSpawnMs) {
    spawnBoss();
    lastBossSpawn = now;
  }

  // ── boss move ─────────────────────────────────────────────
  if (boss) {
    if (boss.y < 120) {
      boss.y += boss.speed * 1.8;
    } else {
      boss.x += boss.speed * 2.2 * boss.dir;
      if (boss.x + boss.size/2 >= W-5 || boss.x - boss.size/2 <= 5) boss.dir *= -1;
    }
  }

  // ── power-ups ─────────────────────────────────────────────
  if (now - lastPowerUpSpawn >= d.powerUpMs) {
    spawnPowerUp();
    lastPowerUpSpawn = now;
  }
  for (let i = powerUps.length-1; i >= 0; i--) {
    powerUps[i].y     += powerUps[i].speed;
    powerUps[i].pulse += 0.1;
    if (powerUps[i].y > H+30) powerUps.splice(i, 1);
  }

  updateParticles();

  // ═════════════════════════════════════════════════════════
  //  COLLISION DETECTION
  // ═════════════════════════════════════════════════════════

  // bullets vs enemies
  outer:
  for (let bi = bullets.length-1; bi >= 0; bi--) {
    const b = bullets[bi];
    for (let ei = enemies.length-1; ei >= 0; ei--) {
      const e = enemies[ei];
      if (circRect(e.x, e.y, e.size*0.46, b.x, b.y-5, 5, 12)) {
        spawnParticles(e.x, e.y, 22, ['#ff6600','#ff3300','#ffaa00','#ffee44','#ff8800']);
        enemies.splice(ei, 1);
        bullets.splice(bi, 1);
        score += 10;
        SFX.hit();
        checkLevelUp();
        continue outer;
      }
    }
    // bullets vs boss
    if (boss) {
      if (circRect(boss.x, boss.y, boss.size*0.46, b.x, b.y-5, 5, 12)) {
        boss.hp--;
        boss.hitFlash = now;
        bullets.splice(bi, 1);
        SFX.bossHit();
        spawnParticles(boss.x, boss.y, 14, ['#cc44ff','#ff88ff','#ffffff']);
        if (boss.hp <= 0) {
          spawnParticles(boss.x, boss.y, 70,
            ['#cc44ff','#ff88ff','#ffee00','#ff4400','#ffffff']);
          score += 50;
          SFX.bossDie();
          boss = null;
          lastBossSpawn = now;
          checkLevelUp();
        }
      }
    }
  }

  // enemies vs player
  for (let ei = enemies.length-1; ei >= 0; ei--) {
    const e = enemies[ei];
    if (circRect(e.x, e.y, e.size*0.44, player.x, player.y, player.w*0.7, player.h*0.7)) {
      spawnParticles(e.x, e.y, 20, ['#ff4444','#ffaa00','#ff8800']);
      enemies.splice(ei, 1);
      if (bigMode && !bigModeHitUsed) {
        bigModeHitUsed = true;
        bigMode = false;
        player.w = 46; player.h = 50; player.speed = 5.5; player.big = false;
        spawnParticles(player.x, player.y, 35, ['#00ffcc','#ffffff','#00cfff']);
      } else {
        loseLife(now);
        if (screen !== 'playing') return;
      }
    }
  }

  // boss vs player
  if (boss) {
    if (circRect(boss.x, boss.y, boss.size*0.44, player.x, player.y, player.w*0.7, player.h*0.7)) {
      if (bigMode && !bigModeHitUsed) {
        bigModeHitUsed = true;
        bigMode = false;
        player.w = 46; player.h = 50; player.speed = 5.5; player.big = false;
      } else {
        loseLife(now);
        if (screen !== 'playing') return;
      }
    }
  }

  // power-ups vs player
  for (let i = powerUps.length-1; i >= 0; i--) {
    const pu = powerUps[i];
    const dx = pu.x - player.x, dy = pu.y - player.y;
    if (Math.sqrt(dx*dx+dy*dy) < pu.r + player.w*0.45) {
      powerUps.splice(i, 1);
      rapidFire    = true;
      rapidFireEnd = now + 5000;
      SFX.powerUp();
      spawnParticles(player.x, player.y, 28, ['#00ff88','#aaffcc','#ffffff']);
    }
  }
}

// ── Lose a life ───────────────────────────────────────────────
function loseLife(now) {
  lives--;
  SFX.loseLife();
  hitFlash = now;
  if (lives <= 0) {
    triggerGameOver();
  }
}

// ── Trigger game over ─────────────────────────────────────────
function triggerGameOver() {
  screen = 'gameover';
  if (score > highScore) highScore = score;
  stopBGM();
  // big explosion
  spawnParticles(player.x, player.y, 80,
    ['#ff4400','#ffaa00','#ffee00','#ff6600','#ffffff']);
}

// ═══════════════════════════════════════════════════════════════
//  UTILITY – rounded rect helper
// ═══════════════════════════════════════════════════════════════
function roundRect(x, y, w, h, r) {
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.lineTo(x + w - r, y);
  ctx.quadraticCurveTo(x + w, y, x + w, y + r);
  ctx.lineTo(x + w, y + h - r);
  ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
  ctx.lineTo(x + r, y + h);
  ctx.quadraticCurveTo(x, y + h, x, y + h - r);
  ctx.lineTo(x, y + r);
  ctx.quadraticCurveTo(x, y, x + r, y);
  ctx.closePath();
}

// ═══════════════════════════════════════════════════════════════
//  HOME SCREEN
// ═══════════════════════════════════════════════════════════════
function drawHome(now) {
  // background gradient
  const bg = ctx.createLinearGradient(0, 0, 0, H);
  bg.addColorStop(0, '#000010');
  bg.addColorStop(1, '#05001a');
  ctx.fillStyle = bg;
  ctx.fillRect(0, 0, W, H);

  drawStars();

  // title glow ring
  const pulse = 0.75 + 0.25 * Math.sin(now * 0.002);
  const tg = ctx.createRadialGradient(W/2, 180, 10, W/2, 180, 160);
  tg.addColorStop(0, `rgba(0,180,255,${0.18 * pulse})`);
  tg.addColorStop(1, 'rgba(0,0,40,0)');
  ctx.fillStyle = tg;
  ctx.fillRect(0, 0, W, 360);

  // title
  ctx.save();
  ctx.textAlign    = 'center';
  ctx.textBaseline = 'middle';
  ctx.shadowColor  = '#00cfff';
  ctx.shadowBlur   = 32 * pulse;
  ctx.fillStyle    = '#00e8ff';
  ctx.font         = 'bold 52px "Segoe UI",Arial';
  ctx.fillText('SPACE', W/2, 150);
  ctx.shadowColor  = '#4488ff';
  ctx.fillStyle    = '#88ccff';
  ctx.font         = 'bold 44px "Segoe UI",Arial';
  ctx.fillText('SHOOTER', W/2, 210);

  // subtitle
  ctx.shadowBlur  = 0;
  ctx.fillStyle   = 'rgba(180,220,255,0.7)';
  ctx.font        = '15px Arial';
  ctx.fillText('Choose your difficulty', W/2, 358);

  // draw a small demo ship
  const dp = { x: W/2, y: 290, w: 40, h: 44, big: false };
  drawPlayer(dp);

  ctx.restore();

  // ── Difficulty buttons ─────────────────────────────────────
  const diffs = ['easy', 'medium', 'hard'];
  diffs.forEach((d, i) => {
    const cfg  = DIFF[d];
    const bx   = W/2 - 80;
    const by   = 390 + i * 70;
    const bw   = 160;
    const bh   = 44;
    const hov  = homeHover.diff === d;
    const sel  = difficulty === d;

    ctx.save();
    // button background
    const grad = ctx.createLinearGradient(bx, by, bx, by + bh);
    if (sel) {
      grad.addColorStop(0, `${cfg.color}55`);
      grad.addColorStop(1, `${cfg.color}22`);
    } else if (hov) {
      grad.addColorStop(0, 'rgba(255,255,255,0.12)');
      grad.addColorStop(1, 'rgba(255,255,255,0.04)');
    } else {
      grad.addColorStop(0, 'rgba(255,255,255,0.06)');
      grad.addColorStop(1, 'rgba(255,255,255,0.02)');
    }
    ctx.fillStyle = grad;
    roundRect(bx, by, bw, bh, 10);
    ctx.fill();

    // border
    ctx.strokeStyle = sel ? cfg.color : (hov ? 'rgba(255,255,255,0.5)' : 'rgba(255,255,255,0.18)');
    ctx.lineWidth   = sel ? 2.5 : 1.5;
    ctx.shadowColor = sel ? cfg.color : 'transparent';
    ctx.shadowBlur  = sel ? 14 : 0;
    roundRect(bx, by, bw, bh, 10);
    ctx.stroke();

    // label
    ctx.shadowBlur  = sel ? 10 : 0;
    ctx.fillStyle   = sel ? cfg.color : (hov ? '#ffffff' : 'rgba(220,230,255,0.8)');
    ctx.font        = `bold ${sel ? 19 : 17}px Arial`;
    ctx.textAlign   = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(cfg.label, W/2, by + bh/2);
    ctx.restore();
  });

  // tap-to-start hint
  ctx.save();
  ctx.textAlign    = 'center';
  ctx.textBaseline = 'middle';
  const hint = 0.5 + 0.5 * Math.sin(now * 0.003);
  ctx.globalAlpha  = 0.45 + 0.45 * hint;
  ctx.fillStyle    = '#aaccff';
  ctx.font         = '14px Arial';
  ctx.fillText('Tap a difficulty to begin', W/2, 618);
  ctx.fillText('← → move  |  SPACE shoot  |  R restart', W/2, 642);
  ctx.restore();

  // high score
  if (highScore > 0) {
    ctx.save();
    ctx.textAlign    = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillStyle    = '#ffdd44';
    ctx.shadowColor  = '#ffdd44';
    ctx.shadowBlur   = 8;
    ctx.font         = 'bold 15px Arial';
    ctx.fillText('Best: ' + highScore, W/2, 672);
    ctx.restore();
  }
}

// ═══════════════════════════════════════════════════════════════
//  GAME OVER SCREEN
// ═══════════════════════════════════════════════════════════════
function drawGameOver(now) {
  // dark overlay with vignette
  ctx.fillStyle = 'rgba(0,0,0,0.72)';
  ctx.fillRect(0, 0, W, H);

  const vg = ctx.createRadialGradient(W/2, H/2, 80, W/2, H/2, W*0.85);
  vg.addColorStop(0, 'rgba(0,0,0,0)');
  vg.addColorStop(1, 'rgba(0,0,20,0.55)');
  ctx.fillStyle = vg;
  ctx.fillRect(0, 0, W, H);

  ctx.save();
  ctx.textAlign    = 'center';
  ctx.textBaseline = 'middle';

  // "GAME OVER" text
  const goPulse = 0.8 + 0.2 * Math.sin(now * 0.004);
  ctx.shadowColor = '#ff2222';
  ctx.shadowBlur  = 40 * goPulse;
  ctx.fillStyle   = '#ff3333';
  ctx.font        = 'bold 62px "Segoe UI",Arial';
  ctx.fillText('GAME OVER', W/2, H/2 - 145);

  // score panel
  ctx.shadowBlur = 0;
  const px = W/2 - 140, py = H/2 - 110, pw = 280, ph = 140;
  const pg = ctx.createLinearGradient(px, py, px, py+ph);
  pg.addColorStop(0, 'rgba(20,30,60,0.92)');
  pg.addColorStop(1, 'rgba(10,10,30,0.92)');
  ctx.fillStyle = pg;
  roundRect(px, py, pw, ph, 14);
  ctx.fill();
  ctx.strokeStyle = 'rgba(0,200,255,0.35)';
  ctx.lineWidth   = 1.5;
  roundRect(px, py, pw, ph, 14);
  ctx.stroke();

  // score values
  ctx.fillStyle = 'rgba(160,200,255,0.7)';
  ctx.font      = '15px Arial';
  ctx.fillText('YOUR SCORE', W/2, H/2 - 88);

  ctx.shadowColor = '#ffffff';
  ctx.shadowBlur  = 12;
  ctx.fillStyle   = '#ffffff';
  ctx.font        = 'bold 44px "Segoe UI",Arial';
  ctx.fillText(score, W/2, H/2 - 50);

  ctx.shadowBlur  = 0;
  ctx.fillStyle   = 'rgba(160,200,255,0.6)';
  ctx.font        = '14px Arial';
  ctx.fillText('Level reached: ' + level, W/2, H/2 - 12);

  // high score
  const isNew = score >= highScore && score > 0;
  if (isNew) {
    ctx.shadowColor = '#ffdd00';
    ctx.shadowBlur  = 14;
    ctx.fillStyle   = '#ffdd00';
    ctx.font        = 'bold 16px Arial';
    ctx.fillText('★ NEW HIGH SCORE! ★', W/2, H/2 + 20);
    ctx.shadowBlur  = 0;
  } else if (highScore > 0) {
    ctx.fillStyle = 'rgba(255,210,60,0.75)';
    ctx.font      = '14px Arial';
    ctx.fillText('Best: ' + highScore, W/2, H/2 + 20);
  }

  // difficulty badge
  const dc  = DIFF[difficulty].color;
  const dlb = DIFF[difficulty].label;
  ctx.fillStyle   = dc + '33';
  roundRect(W/2 - 44, H/2 + 32, 88, 24, 7);
  ctx.fill();
  ctx.strokeStyle = dc;
  ctx.lineWidth   = 1;
  roundRect(W/2 - 44, H/2 + 32, 88, 24, 7);
  ctx.stroke();
  ctx.fillStyle = dc;
  ctx.font      = 'bold 13px Arial';
  ctx.fillText(dlb, W/2, H/2 + 44);

  // ── RESTART button ──────────────────────────────────────────
  const hoverR = isButtonHovered(W/2-100, H/2+60, 200, 50);
  drawButton(W/2-100, H/2+60, 200, 50, '▶  PLAY AGAIN', '#00cfff', hoverR, 16);

  // ── HOME button ─────────────────────────────────────────────
  const hoverH = isButtonHovered(W/2-100, H/2+125, 200, 50);
  drawButton(W/2-100, H/2+125, 200, 50, '⌂  MAIN MENU', '#aaaaff', hoverH, 16);

  // keyboard hint
  ctx.shadowBlur  = 0;
  ctx.fillStyle   = 'rgba(150,170,220,0.5)';
  ctx.font        = '13px Arial';
  ctx.fillText('R = Restart   H = Menu', W/2, H/2 + 196);

  ctx.restore();
}

// shared button helpers
let _mx = 0, _my = 0;
canvas.addEventListener('mousemove', e => {
  const r = canvas.getBoundingClientRect();
  _mx = (e.clientX - r.left) * (W / r.width);
  _my = (e.clientY - r.top)  * (H / r.height);
});
function isButtonHovered(x, y, w, h) {
  return _mx >= x && _mx <= x+w && _my >= y && _my <= y+h;
}
function drawButton(x, y, w, h, label, color, hovered, fs) {
  ctx.save();
  const g = ctx.createLinearGradient(x, y, x, y+h);
  if (hovered) {
    g.addColorStop(0, color + '55');
    g.addColorStop(1, color + '22');
  } else {
    g.addColorStop(0, color + '25');
    g.addColorStop(1, color + '0a');
  }
  ctx.fillStyle = g;
  roundRect(x, y, w, h, 12);
  ctx.fill();

  ctx.shadowColor = hovered ? color : 'transparent';
  ctx.shadowBlur  = hovered ? 16 : 0;
  ctx.strokeStyle = hovered ? color : color + '77';
  ctx.lineWidth   = hovered ? 2.5 : 1.5;
  roundRect(x, y, w, h, 12);
  ctx.stroke();

  ctx.fillStyle   = hovered ? '#ffffff' : color + 'cc';
  ctx.shadowColor = hovered ? color : 'transparent';
  ctx.shadowBlur  = hovered ? 10 : 0;
  ctx.font        = `bold ${fs}px Arial`;
  ctx.textAlign   = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(label, x + w/2, y + h/2);
  ctx.restore();
}

// ═══════════════════════════════════════════════════════════════
//  IN-GAME HUD
// ═══════════════════════════════════════════════════════════════
function drawHUD(now) {
  ctx.save();
  ctx.textBaseline = 'top';

  // semi-transparent top bar
  const hb = ctx.createLinearGradient(0, 0, 0, 42);
  hb.addColorStop(0, 'rgba(0,5,20,0.78)');
  hb.addColorStop(1, 'rgba(0,5,20,0)');
  ctx.fillStyle = hb;
  ctx.fillRect(0, 0, W, 42);

  // Score
  ctx.shadowColor = '#00cfff'; ctx.shadowBlur = 8;
  ctx.fillStyle   = '#00e0ff';
  ctx.font        = 'bold 17px Arial';
  ctx.textAlign   = 'left';
  ctx.fillText('SCORE  ' + score, 12, 10);

  // Level – centre
  const dc = DIFF[difficulty].color;
  ctx.shadowColor = dc; ctx.shadowBlur = 8;
  ctx.fillStyle   = dc;
  ctx.textAlign   = 'center';
  ctx.fillText('LV ' + level + '  ' + DIFF[difficulty].label, W/2, 10);

  // Lives – right
  ctx.shadowColor = '#ff5555'; ctx.shadowBlur = 6;
  ctx.fillStyle   = '#ff6666';
  ctx.textAlign   = 'right';
  ctx.fillText('♥'.repeat(Math.max(0, lives)), W - 12, 10);

  // rapid-fire bar
  if (rapidFire) {
    const frac = Math.max(0, (rapidFireEnd - now) / 5000);
    ctx.shadowBlur = 0;
    ctx.fillStyle  = 'rgba(0,255,120,0.15)';
    ctx.fillRect(W/2 - 90, H - 22, 180, 10);
    const rg = ctx.createLinearGradient(W/2-90, 0, W/2+90, 0);
    rg.addColorStop(0,'#00ff88');
    rg.addColorStop(1,'#00ffcc');
    ctx.fillStyle  = rg;
    ctx.fillRect(W/2 - 90, H - 22, 180 * frac, 10);
    ctx.strokeStyle = 'rgba(0,255,128,0.4)';
    ctx.lineWidth   = 1;
    ctx.strokeRect(W/2 - 90, H - 22, 180, 10);
    ctx.fillStyle   = '#00ffaa';
    ctx.font        = '11px Arial';
    ctx.textAlign   = 'center';
    ctx.fillText('RAPID FIRE', W/2, H - 28);
  }

  // big-ship badge
  if (bigMode) {
    ctx.shadowColor = '#00ffcc'; ctx.shadowBlur = 10;
    ctx.fillStyle   = '#00ffcc';
    ctx.font        = 'bold 13px Arial';
    ctx.textAlign   = 'left';
    ctx.fillText('⬡ BIG SHIP', 12, 32);
  }

  // screen-edge hit flash
  if (now - hitFlash < 350) {
    const f = 1 - (now - hitFlash) / 350;
    ctx.globalAlpha = f * 0.45;
    ctx.fillStyle   = '#ff1111';
    ctx.fillRect(0, 0, W, H);
    ctx.globalAlpha = 1;
  }

  // level-up flash overlay
  if (now - levelFlash < 900) {
    const f = 1 - (now - levelFlash) / 900;
    ctx.globalAlpha  = f * 0.18;
    ctx.fillStyle    = '#ffdd00';
    ctx.fillRect(0, 0, W, H);
    ctx.globalAlpha  = 1;
    ctx.shadowColor  = '#ffdd00';
    ctx.shadowBlur   = 20 * f;
    ctx.fillStyle    = `rgba(255,220,0,${f})`;
    ctx.font         = `bold ${Math.round(28 + 10*f)}px Arial`;
    ctx.textAlign    = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText('LEVEL ' + level + '!', W/2, H/2);
  }

  ctx.restore();
}

// ═══════════════════════════════════════════════════════════════
//  MAIN DRAW  (called every frame regardless of screen)
// ═══════════════════════════════════════════════════════════════
function draw(now) {
  ctx.clearRect(0, 0, W, H);

  if (screen === 'home') {
    drawHome(now);
    return;
  }

  // ── playing or gameover — draw game world first ────────────
  const bg = ctx.createLinearGradient(0, 0, 0, H);
  bg.addColorStop(0, '#000010');
  bg.addColorStop(1, '#05001a');
  ctx.fillStyle = bg;
  ctx.fillRect(0, 0, W, H);

  drawStars();

  // subtle grid lines for depth illusion
  ctx.save();
  ctx.strokeStyle = 'rgba(0,80,180,0.06)';
  ctx.lineWidth   = 1;
  for (let x = 0; x < W; x += 48) {
    ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, H); ctx.stroke();
  }
  ctx.restore();

  // game objects
  powerUps.forEach(pu  => drawPowerUp(pu));
  bullets.forEach(b    => drawBullet(b));
  enemies.forEach(e    => drawEnemy(e));
  if (boss)              drawBoss(boss);
  if (screen === 'playing') drawPlayer(player);
  drawParticles();
  drawHUD(now);

  // game-over overlay on top
  if (screen === 'gameover') {
    drawGameOver(now);
  }
}

// ═══════════════════════════════════════════════════════════════
//  MAIN LOOP
// ═══════════════════════════════════════════════════════════════
function loop(now) {
  update(now);
  draw(now);
  frameId = requestAnimationFrame(loop);
}

// ═══════════════════════════════════════════════════════════════
//  MOBILE CONTROLS
// ═══════════════════════════════════════════════════════════════
const lbtn = document.getElementById('lbtn');
const rbtn = document.getElementById('rbtn');
const fbtn = document.getElementById('fbtn');

function bindBtn(btn, code) {
  let iv = null;

  function press(e) {
    e.preventDefault();
    ensureAC();

    // start game from home by tapping any button
    if (screen === 'home' && code !== 'Space') return;
    if (screen === 'home' && code === 'Space') return; // handled by canvas tap

    // restart from gameover
    if (screen === 'gameover') {
      if (code === 'Space') { initGame(); return; }
      return;
    }

    keys[code] = true;
    btn.classList.add('held');

    if (code === 'Space') {
      shoot(performance.now());
      iv = setInterval(() => {
        if (screen === 'playing') shoot(performance.now());
      }, 80);
    }
  }

  function release(e) {
    e.preventDefault();
    keys[code] = false;
    btn.classList.remove('held');
    if (iv) { clearInterval(iv); iv = null; }
  }

  btn.addEventListener('touchstart',  press,   { passive: false });
  btn.addEventListener('touchend',    release, { passive: false });
  btn.addEventListener('touchcancel', release, { passive: false });
  btn.addEventListener('mousedown',   press);
  btn.addEventListener('mouseup',     release);
  btn.addEventListener('mouseleave',  release);
}

bindBtn(lbtn, 'ArrowLeft');
bindBtn(rbtn, 'ArrowRight');
bindBtn(fbtn, 'Space');

// tapping the fire button on home starts the game with current difficulty
fbtn.addEventListener('touchstart', e => {
  e.preventDefault();
  ensureAC();
  if (screen === 'home') {
    initGame();
  } else if (screen === 'gameover') {
    initGame();
  }
}, { passive: false });

// keyboard: H goes home from gameover, already handled in keydown above
// also allow Enter to start from home
document.addEventListener('keydown', e => {
  if (e.code === 'Enter' && screen === 'home') {
    ensureAC();
    initGame();
  }
});

// ═══════════════════════════════════════════════════════════════
//  CANVAS SCALE  (keeps internal 480×720 logic)
// ═══════════════════════════════════════════════════════════════
function scaleWrap() {
  const wrap  = document.getElementById('wrap');
  const vw    = window.innerWidth;
  const vh    = window.innerHeight;
  const ratio = 480 / 720;
  let   w     = Math.min(vw, 480);
  let   h     = w / ratio;
  if (h > vh) { h = vh; w = h * ratio; }
  wrap.style.width  = Math.floor(w) + 'px';
  wrap.style.height = Math.floor(h) + 'px';
}
scaleWrap();
window.addEventListener('resize', scaleWrap);

// ═══════════════════════════════════════════════════════════════
//  BOOT
// ═══════════════════════════════════════════════════════════════
initStars();
screen = 'home';
requestAnimationFrame(loop);

</script>
</body>
</html>
