# arull.ganteng.slebew
<!doctype html>
<html lang="id">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Biraa Semangatt — efek teks & hati</title>
  <style>
    /* Reset & layout */
    html,body{height:100%;margin:0;background:#000;overflow:hidden;font-family: Inter, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial;}
    canvas{display:block;width:100%;height:100%;}

    /* small controls in corner */
    .controls{
      position:fixed;left:12px;top:12px;z-index:10;color:#fff;font-size:13px;backdrop-filter:blur(6px);
      background:rgba(255,255,255,0.04);padding:8px 10px;border-radius:10px;border:1px solid rgba(255,255,255,0.04)
    }
    .controls label{display:flex;gap:8px;align-items:center}
    .controls input[type=range]{width:120px}
    .credits{position:fixed;right:12px;bottom:12px;color:rgba(255,255,255,0.35);font-size:12px}
  </style>
</head>
<body>
  <div class="controls">
    <label>Kecepatan <input id="speed" type="range" min="0" max="2" step="0.01" value="1"></label>
    <label>Jumlah <input id="count" type="range" min="50" max="350" step="1" value="160"></label>
  </div>
  <div class="credits">Teks: "biraa semangatt" — disiapkan oleh Anda</div>
  <canvas id="c"></canvas>

  <script>
    // Config
    const TEXT = "biraa semangatt"; // teks yang diminta
    const HEART_RATIO = 0.38; // proporsi partikel hati

    // Canvas setup
    const canvas = document.getElementById('c');
    const ctx = canvas.getContext('2d');

    let DPR = Math.max(1, window.devicePixelRatio || 1);
    function resize(){
      DPR = Math.max(1, window.devicePixelRatio || 1);
      canvas.width = Math.floor(innerWidth * DPR);
      canvas.height = Math.floor(innerHeight * DPR);
      canvas.style.width = innerWidth + 'px';
      canvas.style.height = innerHeight + 'px';
      ctx.setTransform(DPR,0,0,DPR,0,0);
    }
    window.addEventListener('resize', resize);
    resize();

    // Camera / perspective
    const FOV = 800; // focal length

    // Particle class
    class Particle {
      constructor(type){
        this.reset(type);
      }
      reset(type){
        this.type = type || (Math.random() < HEART_RATIO ? 'heart' : 'text');
        // positions in 3D space
        this.x = (Math.random()*2 -1) * (innerWidth*0.6);
        this.y = (Math.random()*2 -1) * (innerHeight*0.6);
        this.z = Math.random() * 4000 + 200; // distance from camera
        this.speedZ = Math.random()*4 + 0.5; // base speed toward camera
        this.rotation = Math.random()*Math.PI*2;
        this.scale = 0.6 + Math.random()*1.6;

        // random color tint for variety
        this.tint = { r: 255, g: 50 + Math.floor(Math.random()*100), b: 120 + Math.floor(Math.random()*80) };
      }
      update(dt, speedMultiplier){
        this.z -= this.speedZ * dt * speedMultiplier;
        this.rotation += 0.002 * dt;
        if(this.z < 20) this.reset(this.type);
      }
      project(){
        // simple perspective projection
        const scale = FOV / (FOV + this.z);
        this.screenX = innerWidth/2 + this.x * scale;
        this.screenY = innerHeight/2 + this.y * scale;
        this.screenScale = scale * this.scale;
      }
      drawText(ctx){
        const s = this.screenScale;
        ctx.save();
        ctx.translate(this.screenX, this.screenY);
        ctx.rotate(this.rotation * 0.25);

        // glowing pink text style
        ctx.font = Math.max(10, 48 * s) + 'px "Helvetica Neue", Arial, sans-serif';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';

        // outer glow
        ctx.globalCompositeOperation = 'lighter';
        ctx.fillStyle = 'rgba(255, 150, 230, 0.95)';
        ctx.shadowColor = 'rgba(255,140,240,0.9)';
        ctx.shadowBlur = 30 * s;
        ctx.fillText(TEXT, 0, 0);

        // softer core
        ctx.shadowBlur = 6 * s;
        ctx.fillStyle = 'rgba(255,220,255,0.98)';
        ctx.fillText(TEXT, 0, 0);

        ctx.restore();
      }
      drawHeart(ctx){
        const s = this.screenScale * 1.6;
        ctx.save();
        ctx.translate(this.screenX, this.screenY);
        ctx.rotate(this.rotation);

        // heart shape path (relative coords)
        const size = 22 * s;
        ctx.beginPath();
        ctx.moveTo(0, -size/4);
        ctx.bezierCurveTo(size/2, -size*0.9, size*1.4, size*0.2, 0, size);
        ctx.bezierCurveTo(-size*1.4, size*0.2, -size/2, -size*0.9, 0, -size/4);
        ctx.closePath();

        // glow & fill
        ctx.globalCompositeOperation = 'lighter';
        ctx.shadowColor = 'rgba(255,60,80,0.95)';
        ctx.shadowBlur = 30 * s;
        ctx.fillStyle = `rgba(${this.tint.r},${this.tint.g},${this.tint.b},0.98)`;
        ctx.fill();

        // inner small highlight
        ctx.shadowBlur = 6 * s;
        ctx.globalCompositeOperation = 'source-over';
        ctx.fillStyle = 'rgba(255,90,110,0.85)';
        ctx.scale(0.9,0.9);
        ctx.fill();

        ctx.restore();
      }
    }

    // Particle system
    let particles = [];
    function populate(count){
      particles = [];
      for(let i=0;i<count;i++) particles.push(new Particle());
    }

    // controls
    const speedControl = document.getElementById('speed');
    const countControl = document.getElementById('count');
    populate(parseInt(countControl.value,10));
    countControl.addEventListener('input', ()=> populate(parseInt(countControl.value,10)));

    // animation loop
    let last = performance.now();
    function frame(t){
      const dt = (t - last) * 0.06; // make dt usable
      last = t;

      // fade background slightly to produce star-like specks
      ctx.fillStyle = 'rgba(0,0,0,0.45)';
      ctx.fillRect(0,0,innerWidth,innerHeight);

      // draw tiny stars
      drawStars();

      const speedMultiplier = parseFloat(speedControl.value) || 1;

      for(let p of particles){
        p.update(dt, speedMultiplier);
        p.project();
        if(p.type === 'text') p.drawText(ctx); else p.drawHeart(ctx);
      }

      requestAnimationFrame(frame);
    }

    // generate subtle stars once per second for variety
    let starCanvas = null;
    function makeStarCanvas(){
      starCanvas = document.createElement('canvas');
      starCanvas.width = innerWidth; starCanvas.height = innerHeight;
      const sctx = starCanvas.getContext('2d');
      sctx.fillStyle = 'black';
      sctx.fillRect(0,0,innerWidth,innerHeight);
      for(let i=0;i<400;i++){
        const x = Math.random()*innerWidth;
        const y = Math.random()*innerHeight;
        const r = Math.random()*1.6;
        sctx.globalAlpha = 0.6*Math.random();
        sctx.fillStyle = 'white';
        sctx.beginPath();
        sctx.arc(x,y,r,0,Math.PI*2);
        sctx.fill();
      }
    }
    function drawStars(){
      if(!starCanvas) makeStarCanvas();
      ctx.globalAlpha = 0.12;
      ctx.drawImage(starCanvas,0,0,innerWidth,innerHeight);
      ctx.globalAlpha = 1;
    }

    // start
    requestAnimationFrame(frame);

    // helpful: regenerate stars on resize
    window.addEventListener('resize', ()=>{ makeStarCanvas(); });

    // keyboard: space to pause
    let paused=false; window.addEventListener('keydown', e=>{ if(e.code==='Space'){ paused=!paused; if(!paused){ last=performance.now(); requestAnimationFrame(frame);} }});
  </script>
</body>
</html>
