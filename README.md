# Mo-phong-tham-thau-va-khuech-tan
<!doctype html>
<html lang="vi">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Mô phỏng: Khuếch tán và Thẩm thấu</title>
<style>
  :root{--bg:#f6f9fb;--panel:#ffffff;--accent:#2b6cb0}
  body{font-family:Inter,Segoe UI,Roboto,Arial; background:var(--bg); margin:0; padding:18px; color:#0b2340}
  .app{max-width:1100px;margin:0 auto;display:grid;grid-template-columns:360px 1fr;gap:18px}
  .panel{background:var(--panel);padding:14px;border-radius:10px;box-shadow:0 6px 18px rgba(11,35,64,0.06)}
  h2{margin:6px 0 12px;font-size:18px}
  label{display:block;font-size:13px;margin:8px 0}
  input[type=range]{width:100%}
  .row{display:flex;gap:8px;align-items:center}
  .btn{padding:8px 10px;border-radius:8px;border:1px solid rgba(11,35,64,0.08);background:white;cursor:pointer}
  .btn.primary{background:var(--accent);color:white;border:none}
  canvas{width:100%;height:520px;background:linear-gradient(180deg,#eaf3fb,white);border-radius:8px}
  .stat{font-size:13px;margin-top:8px}
  .small{font-size:12px;color:#445}
  .control-grid{display:grid;grid-template-columns:1fr 1fr;gap:8px}
  .legend{display:flex;gap:8px;align-items:center;margin-top:8px}
  .swatch{width:14px;height:14px;border-radius:50%}
</style>
</head>
<body>
<div class="app">
  <div class="panel">
    <h2>Mô phỏng: Khuếch tán và Thẩm thấu</h2>
    <div><label>Chế độ</label>
      <div class="row">
        <select id="mode" class="btn" style="flex:1">
          <option value="diffusion">Khuếch tán (Diffusion)</option>
          <option value="osmosis">Thẩm thấu (Osmosis)</option>
        </select>
        <button id="start" class="btn primary">Chạy</button>
        <button id="pause" class="btn">Tạm dừng</button>
      </div>
    </div>

    <label>Loại tế bào</label>
    <div class="row">
      <select id="cellType" class="btn" style="flex:1">
        <option value="animal">Tế bào động vật</option>
        <option value="plant">Tế bào thực vật</option>
      </select>
    </div>

    <label>Nồng độ phân tử bên trái (0-100)</label>
    <input id="leftConc" type="range" min="0" max="100" value="70" />
    <div class="row small"><div>0</div><div style="flex:1"></div><div>100</div></div>

    <label>Nồng độ phân tử bên phải (0-100)</label>
    <input id="rightConc" type="range" min="0" max="100" value="30" />

    <label>Độ thấm màng (permeability 0-1)</label>
    <input id="perm" type="range" min="0" max="1" step="0.01" value="0.6" />

    <label>Tốc độ nhiệt (ảnh hưởng chuyển động ngẫu nhiên)</label>
    <input id="temp" type="range" min="0" max="2" step="0.01" value="0.8" />

    <div class="row" style="margin-top:10px">
      <button id="reset" class="btn">Đặt lại</button>
      <button id="step" class="btn">Bước 1</button>
    </div>

    <div class="stat">
      <div>Gradient hiện tại: <span id="gradient">40</span></div>
      <div>Số phân tử trái: <span id="countL">0</span> — phải: <span id="countR">0</span></div>
    </div>

    <div class="legend">
      <div style="display:flex;flex-direction:column;gap:6px">
        <div style="display:flex;gap:6px;align-items:center"><div class="swatch" style="background:#e53e3e"></div><div class="small">Phân tử hoà tan (solute)</div></div>
        <div style="display:flex;gap:6px;align-items:center"><div class="swatch" style="background:#3182ce"></div><div class="small">Phân tử nước (water)</div></div>
      </div>
    </div>

    <p class="small" style="margin-top:10px">Hướng dẫn ngắn: Chọn chế độ "Khuếch tán" để quan sát phân tử di chuyển theo gradient; chọn "Thẩm thấu" để quan sát nước di chuyển và thể tích tế bào thay đổi. Điều chỉnh nồng độ và độ thấm để thấy tác dụng.</p>
  </div>

  <div class="panel">
    <canvas id="sim" width="800" height="520"></canvas>
  </div>
</div>

<script>
(function(){
  const canvas = document.getElementById('sim');
  const ctx = canvas.getContext('2d');
  const width = canvas.width, height = canvas.height;

  // UI references
  const modeSel = document.getElementById('mode');
  const leftConc = document.getElementById('leftConc');
  const rightConc = document.getElementById('rightConc');
  const perm = document.getElementById('perm');
  const temp = document.getElementById('temp');
  const startBtn = document.getElementById('start');
  const pauseBtn = document.getElementById('pause');
  const resetBtn = document.getElementById('reset');
  const stepBtn = document.getElementById('step');
  const countL = document.getElementById('countL');
  const countR = document.getElementById('countR');
  const gradient = document.getElementById('gradient');
  const cellType = document.getElementById('cellType');

  let running = false;

  // Compartments: left and right separated by membrane at center
  const memX = width/2;
  const memW = 8;

  // Particles
  let solutes = []; // red
  let waters = [];  // blue

  function resetParticles(){
    solutes = [];
    waters = [];
    const L = parseInt(leftConc.value);
    const R = parseInt(rightConc.value);

    // number of solute particles proportional to conc
    const nL = Math.round(120 * L / 100);
    const nR = Math.round(120 * R / 100);

    // water particles: complementary count but larger
    const wL = Math.round(200 * (100 - L) / 100);
    const wR = Math.round(200 * (100 - R) / 100);

    for(let i=0;i<nL;i++) solutes.push(randomParticle('L'));
    for(let i=0;i<nR;i++) solutes.push(randomParticle('R'));
    for(let i=0;i<wL;i++) waters.push(randomParticle('L','water'));
    for(let i=0;i<wR;i++) waters.push(randomParticle('R','water'));

    updateCounts();
  }

  function randomParticle(side,type='solute'){
    const r = 4 + Math.random()*6;
    const y = 60 + Math.random()*(height-120);
    const x = side==='L' ? (20 + Math.random()*(memX-60)) : (memX + memW + 20 + Math.random()*(width - memX - memW - 60));
    return {x,y,r,side,type,vx:(Math.random()-0.5)*0.6,vy:(Math.random()-0.5)*0.6};
  }

  function updateCounts(){
    countL.textContent = solutes.filter(p=>p.x < memX).length + ' solute / ' + waters.filter(p=>p.x < memX).length + ' water';
    countR.textContent = solutes.filter(p=>p.x > memX).length + ' solute / ' + waters.filter(p=>p.x > memX).length + ' water';
    const grad = Math.abs(parseInt(leftConc.value) - parseInt(rightConc.value));
    gradient.textContent = grad;
  }

  // Simple physics step
  function step(dt){
    const p = parseFloat(perm.value);
    const T = parseFloat(temp.value);
    const mode = modeSel.value;

    // Determine concentrations (approx) inside each side
    const solL = solutes.filter(s=>s.x < memX).length;
    const solR = solutes.filter(s=>s.x > memX).length;
    const watL = waters.filter(s=>s.x < memX).length;
    const watR = waters.filter(s=>s.x > memX).length;

    // For diffusion: solute particles tend to move from high count side to low count side across membrane according to permeability
    if(mode === 'diffusion'){
      const totalSol = solL + solR;
      // compute desired flux probability per particle crossing membrane
      const diffFactor = (Math.max(0, solL - solR) / (totalSol+1));

      function moveParticle(particle){
        // random walk with bias away from higher concentration
        let bias = 0;
        if(particle.x < memX){ bias = -diffFactor; } else { bias = diffFactor; }
        // membrane region reduces crossing probability according to permeability
        particle.vx += (Math.random()-0.5)*T*0.6 + bias*0.6;
        particle.vy += (Math.random()-0.5)*T*0.4;
        particle.x += particle.vx;
        particle.y += particle.vy;
        // boundary reflections
        if(particle.y < 60) particle.y = 60 + Math.random()*6;
        if(particle.y > height-60) particle.y = height-60 - Math.random()*6;
        if(particle.x < 10) particle.x = 10 + Math.random()*4;
        if(particle.x > width-10) particle.x = width-10 - Math.random()*4;

        // membrane collision: crossing permitted with probability p
        if(particle.x > memX - 6 && particle.x < memX + memW + 6){
          if(Math.random() > p){
            // bounce
            particle.vx *= -0.6; particle.x += particle.vx*2;
          }
        }
      }

      solutes.forEach(moveParticle);
      waters.forEach(moveParticle);
    }

    // For osmosis: only water moves preferentially from low solute concentration to high solute concentration
    if(mode === 'osmosis'){
      // compute average solute concentration on each side
      const cL = solL / (solL + watL + 1);
      const cR = solR / (solR + watR + 1);
      const osmoticDiff = cR - cL; // positive => right side more concentrated
      // water flux proportional to permeability * osmoticDiff
      const flux = p * osmoticDiff * 0.5; // tuned constant

      // move water particles according to flux
      waters.forEach(w=>{
        // base random motion
        w.vx += (Math.random()-0.5)*T*0.4;
        w.vy += (Math.random()-0.5)*T*0.3;
        // directional bias: if flux>0 move left->right? osmoticDiff>0 means right more concentrated so water moves left->right? Actually water moves to higher solute -> from low c to high c
        const direction = Math.sign(osmoticDiff); // positive -> water flows right
        w.vx += direction * Math.abs(flux) * (1 + Math.random()*0.6);
        w.x += w.vx; w.y += w.vy;
        // bounce boundaries
        if(w.y < 60) w.y = 60 + Math.random()*6;
        if(w.y > height-60) w.y = height-60 - Math.random()*6;
        if(w.x < 10) w.x = 10 + Math.random()*4;
        if(w.x > width-10) w.x = width-10 - Math.random()*4;
        // membrane filter
        if(w.x > memX - 6 && w.x < memX + memW + 6){
          if(Math.random() > p){ w.vx *= -0.6; w.x += w.vx*2; }
        }
      });

      // Additionally, adjust a 'cell volume' representation on the right side according to net water change
      // approximate net water change = (#water moving into right) - (#moving out)
      // compute a simple heuristic: if osmoticDiff>0 -> waters slowly accumulate on right
      if(Math.abs(osmoticDiff) > 0.001){
        // if right more concentrated, water flows right (from left to right)
        // move small number of water particles across probabilistically
        const moveProb = Math.min(0.02 * Math.abs(osmoticDiff) * p * 10, 0.5);
        if(osmoticDiff > 0){
          // attempt to move some water from left to right by relocating a few left waters
          for(let i=0;i<3;i++){
            const idx = waters.findIndex(w=>w.x < memX);
            if(idx>=0 && Math.random() < moveProb){ waters[idx].x = memX + memW + 10 + Math.random()*100; }
          }
        } else {
          for(let i=0;i<3;i++){
            const idx = waters.findIndex(w=>w.x > memX);
            if(idx>=0 && Math.random() < moveProb){ waters[idx].x = memX - 10 - Math.random()*100; }
          }
        }
      }
    }

    // keep particles within boxes
    [solutes, waters].flat().forEach(particle=>{
      if(particle.y < 60) particle.y = 60 + 2*Math.random();
      if(particle.y > height-60) particle.y = height-60 - 2*Math.random();
    });

    updateCounts();
    render();
  }

  function render(){
    ctx.clearRect(0,0,width,height);
    // draw compartments
    ctx.fillStyle = '#e8f3ff'; ctx.fillRect(10,60,memX-20,height-120);
    ctx.fillStyle = '#fef7e6'; ctx.fillRect(memX+memW+10,60,width-memX-memW-20,height-120);
    // membrane
    ctx.fillStyle = '#bfcfe0'; ctx.fillRect(memX,40,memW,height-80);
    // membrane label
    ctx.save(); ctx.translate(memX,40); ctx.rotate(-Math.PI/2); ctx.fillStyle='#344'; ctx.font='12px Arial'; ctx.fillText('Màng bán thấm', -height/2 + 40, -6); ctx.restore();

    // draw cell outline on right to represent volume
    const cType = cellType.value;
    const cellX = memX + memW + 60; const cellW = 160;
    // compute water imbalance to show cell swelling
    const watL = waters.filter(s=>s.x < memX).length;
    const watR = waters.filter(s=>s.x > memX).length;
    const solL = solutes.filter(s=>s.x < memX).length;
    const solR = solutes.filter(s=>s.x > memX).length;
    const osmotic = (solR/(solR+watR+1)) - (solL/(solL+watL+1));
    // map to cell height
    const baseH = 120; let cellH = baseH - Math.round(osmotic*220);
    if(cType==='plant'){
      // plant cell limited by wall: cannot exceed max expansion, and shows turgor pressure
      cellH = Math.min(200, Math.max(80, cellH));
    } else {
      cellH = Math.min(260, Math.max(60, cellH));
    }
    const cellY = 90 + (120 - cellH/2);
    // draw cell
    ctx.save();
    ctx.strokeStyle = '#2d3748'; ctx.lineWidth = 2; ctx.beginPath();
    if(cType === 'plant'){
      ctx.rect(cellX, cellY, cellW, cellH);
    } else {
      // rounded for animal cell
      roundRect(ctx, cellX, cellY, cellW, cellH, 24);
    }
    ctx.stroke();
    ctx.restore();

    // draw particles
    solutes.forEach(s=>{
      ctx.beginPath(); ctx.fillStyle = '#e53e3e'; ctx.arc(s.x,s.y,s.r,0,Math.PI*2); ctx.fill();
    });
    waters.forEach(w=>{
      ctx.beginPath(); ctx.fillStyle = '#3182ce'; ctx.arc(w.x,w.y,w.r-1,0,Math.PI*2); ctx.fill();
    });

    // info overlay
    ctx.fillStyle='#123'; ctx.font='13px Arial';
    ctx.fillText('Nồng độ trái: '+leftConc.value, 20, 36);
    ctx.fillText('Nồng độ phải: '+rightConc.value, width-160, 36);
    ctx.fillText('Độ thấm: '+perm.value, width/2 - 40, 36);

    // small guides
    ctx.fillStyle='rgba(0,0,0,0.06)'; ctx.fillRect(10,60,memX-20,24);
  }

  function roundRect(ctx,x,y,w,h,r){
    ctx.beginPath(); ctx.moveTo(x+r,y); ctx.arcTo(x+w,y,x+w,y+h,r); ctx.arcTo(x+w,y+h,x,y+h,r); ctx.arcTo(x,y+h,x,y,r); ctx.arcTo(x,y,x+w,y,r); ctx.closePath(); ctx.fill();
  }

  // animation loop
  let last = performance.now();
  function loop(t){
    if(!running) return;
    const dt = (t-last)/1000; last = t;
    step(dt);
    requestAnimationFrame(loop);
  }

  // wire UI
  startBtn.addEventListener('click', ()=>{ if(!running){ running=true; last = performance.now(); requestAnimationFrame(loop); } });
  pauseBtn.addEventListener('click', ()=>{ running=false; });
  resetBtn.addEventListener('click', ()=>{ resetParticles(); render(); });
  stepBtn.addEventListener('click', ()=>{ step(0.016); });

  [leftConc, rightConc, perm, temp, modeSel, cellType].forEach(el=>el.addEventListener('input', ()=>{ resetParticles(); render(); }));

  // initial
  resetParticles(); render();

})();
</script>
</body>
</html>
