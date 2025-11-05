<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover" />
  <title>Pro Camera — 作者：ゆいきち</title>
  <style>
    /* Premium Apple-like inspired UI (distinct from Apple) */
    :root{--bg:#000;--accent:#0cd8ff;--muted:#9aa6b2;--white:#fff;--glass:rgba(255,255,255,0.04)}
    *{box-sizing:border-box}
    html,body{height:100%;margin:0;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,'Helvetica Neue',Arial}
    body{background:linear-gradient(180deg,#000 0%,#05060a 100%);color:var(--white);display:flex;align-items:stretch;justify-content:center}
    .app{width:100%;max-width:920px;height:100vh;display:flex;flex-direction:column}
    header{height:92px;padding:14px 18px;display:flex;flex-direction:column;justify-content:flex-end}
    .toprow{display:flex;justify-content:space-between;align-items:center}
    .btn{background:var(--glass);padding:8px 12px;border-radius:12px;border:1px solid rgba(255,255,255,0.04);font-size:14px;color:var(--white)}

    .viewfinder{position:relative;flex:1;margin:8px 12px;border-radius:18px;overflow:hidden;background:#000}
    video#preview{width:100%;height:100%;object-fit:cover;transform-origin:center center;touch-action:none}
    canvas#snapshot{display:none}

    .overlay-top{position:absolute;top:14px;left:14px;right:14px;display:flex;justify-content:space-between;pointer-events:none}
    .badge{background:rgba(0,0,0,0.5);padding:6px 8px;border-radius:999px;pointer-events:auto}
    .mode-label{position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);font-weight:700;letter-spacing:2px;pointer-events:none}

    .controls{height:190px;display:flex;flex-direction:column;align-items:stretch;justify-content:center;padding:10px 18px;gap:10px}
    .modes{display:flex;gap:12px;align-items:center;justify-content:center}
    .mode{padding:8px 14px;border-radius:999px;font-weight:700;opacity:.9;cursor:pointer}
    .mode.active{background:rgba(255,255,255,0.06)}

    .shutter-row{display:flex;align-items:center;gap:18px;justify-content:center}
    .thumbs{width:64px;height:64px;border-radius:14px;overflow:hidden;border:1px solid rgba(255,255,255,0.06)}
    .thumbs img{width:100%;height:100%;object-fit:cover}

    .shutter{width:96px;height:96px;border-radius:50%;background:linear-gradient(180deg,#fff,#ddd);display:flex;align-items:center;justify-content:center;box-shadow:0 10px 40px rgba(0,0,0,.6);border:8px solid rgba(255,255,255,0.06);cursor:pointer}
    .shutter.recording{background:radial-gradient(circle at 50% 50%, #ff4d4d 0%, #c60000 60%);border-color:rgba(255,255,255,0.02)}
    .small-btn{width:48px;height:48px;border-radius:999px;background:var(--glass);display:flex;align-items:center;justify-content:center}

    .toolbar{display:flex;gap:10px;align-items:center;justify-content:center}
    .zoom-slider{width:240px}
    .status{font-size:13px;color:var(--muted);text-align:center;margin-top:6px}
    .grid-overlay{position:absolute;inset:0;pointer-events:none;background-image:linear-gradient(transparent calc(33.333% - 1px), rgba(255,255,255,0.06) 1px, transparent calc(33.333% + 1px)), linear-gradient(90deg, transparent calc(33.333% - 1px), rgba(255,255,255,0.06) 1px, transparent calc(33.333% + 1px));opacity:0;transition:opacity .2s}
    .grid-overlay.visible{opacity:1}

    .gallery{position:absolute;right:14px;bottom:210px;display:flex;flex-direction:column;gap:8px}
    .gallery a{width:86px;height:86px;border-radius:12px;overflow:hidden;border:1px solid rgba(255,255,255,0.06)}
    .gallery img{width:100%;height:100%;object-fit:cover}

    footer{height:36px;text-align:center;color:var(--muted);font-size:12px}

    @media(max-width:520px){.shutter{width:72px;height:72px}.thumbs{width:48px;height:48px}}
  </style>
</head>
<body>
  <main class="app">
    <header>
      <div class="toprow">
        <div style="display:flex;gap:10px;align-items:center">
          <button id="flashBtn" class="btn">⚡ 自動</button>
          <button id="hdrBtn" class="btn">HDR</button>
          <button id="gridBtn" class="btn">グリッド</button>
        </div>
        <div style="display:flex;gap:10px;align-items:center">
          <button id="settingsBtn" class="btn">設定</button>
        </div>
      </div>
      <div style="display:flex;justify-content:center;margin-top:8px"><div class="status" id="status">準備完了 — 作者：ゆいきち</div></div>
    </header>

    <section class="viewfinder" id="viewfinder">
      <div class="overlay-top">
        <div class="left">
          <div class="badge" id="recordIndicator" style="display:none">● REC <span id="recTimer">00:00</span></div>
        </div>
        <div class="right">
          <div class="badge" id="deviceLabel">—</div>
        </div>
      </div>

      <div class="mode-label" id="modeLabel">PHOTO</div>
      <video id="preview" autoplay playsinline muted></video>
      <canvas id="snapshot"></canvas>
      <div class="grid-overlay" id="gridOverlay"></div>
      <div class="gallery" id="gallery"></div>
    </section>

    <section class="controls">
      <div class="modes" role="tablist" aria-label="カメラモード">
        <div class="mode active" data-mode="photo">写真</div>
        <div class="mode" data-mode="video">ビデオ</div>
        <div class="mode" data-mode="portrait">ポートレート</div>
      </div>

      <div class="shutter-row">
        <div class="thumbs" id="lastThumb"><img src="" alt="" class="hidden"></div>
        <div style="display:flex;gap:12px;align-items:center">
          <button id="switchCam" class="small-btn" title="切替">⇄</button>
        </div>
        <div style="display:flex;justify-content:center">
          <button id="shutter" class="shutter" title="シャッター"></button>
        </div>
        <div style="display:flex;gap:10px;align-items:center">
          <input type="range" id="zoom" class="zoom-slider" min="1" max="4" step="0.01" value="1" title="ズーム">
        </div>
      </div>

      <div style="display:flex;justify-content:center;gap:12px;align-items:center">
        <button id="enableBtn" class="btn">カメラを有効にする</button>
        <button id="burstBtn" class="btn">バースト</button>
        <button id="timerBtn" class="btn">タイマー：なし</button>
        <button id="torchBtn" class="btn">ライト</button>
        <button id="shutterSoundBtn" class="btn">シャッター音：ON</button>
        <button id="downloadAll" class="btn">ギャラリーDL</button>
      </div>

      <div class="status" id="hint">iPhoneではHTTPS（またはlocalhost）で動作させてください。動画録画はブラウザ依存です。</div>
    </section>

    <footer>© ゆいきち — Pro Camera (web版)</footer>
  </main>

  <script>
    // Advanced: hardware zoom if available, digital zoom fallback, torch control, exposure compensation (if supported), burst, timer, shutter sound, grid

    const preview = document.getElementById('preview');
    const canvas = document.getElementById('snapshot');
    const status = document.getElementById('status');
    const hint = document.getElementById('hint');
    const enableBtn = document.getElementById('enableBtn');
    const shutter = document.getElementById('shutter');
    const switchCam = document.getElementById('switchCam');
    const modeEls = document.querySelectorAll('.mode');
    const modeLabel = document.getElementById('modeLabel');
    const gallery = document.getElementById('gallery');
    const lastThumb = document.querySelector('#lastThumb img');
    const recordIndicator = document.getElementById('recordIndicator');
    const recTimer = document.getElementById('recTimer');
    const zoomSlider = document.getElementById('zoom');
    const gridBtn = document.getElementById('gridBtn');
    const gridOverlay = document.getElementById('gridOverlay');
    const burstBtn = document.getElementById('burstBtn');
    const timerBtn = document.getElementById('timerBtn');
    const torchBtn = document.getElementById('torchBtn');
    const flashBtn = document.getElementById('flashBtn');
    const hdrBtn = document.getElementById('hdrBtn');
    const shutterSoundBtn = document.getElementById('shutterSoundBtn');

    let currentStream = null;
    let mediaRecorder = null;
    let recordedChunks = [];
    let recording = false;
    let currentMode = 'photo';
    let currentDeviceId = null;
    let devices = [];
    let zoom = 1;
    let recInterval = null;
    let recStart = null;
    let burstMode = false;
    let timerSeconds = 0;
    let torchOn = false;
    let shutterSoundOn = true;

    let hardwareZoomSupported = false;
    let trackCapabilities = null;
    let videoTrack = null;

    function updateModeUI(){
      modeEls.forEach(el=>el.classList.toggle('active', el.dataset.mode===currentMode));
      modeLabel.textContent = currentMode.toUpperCase();
      shutter.classList.toggle('recording', currentMode==='video' && recording);
    }

    modeEls.forEach(el=>el.addEventListener('click', ()=>{
      currentMode = el.dataset.mode;
      updateModeUI();
    }));

    async function enumerateDevices(){
      try{ const list = await navigator.mediaDevices.enumerateDevices(); devices = list.filter(d=>d.kind==='videoinput'); return devices; }catch(e){ console.error('enumerateDevices',e); return []; }
    }

    async function startCamera(deviceId=null, facingMode='environment'){
      stopCamera();
      try{
        status.textContent = 'カメラ起動中…';
        let constraints = {audio: true, video: {width:{ideal:1920},height:{ideal:1440}}};
        if (deviceId) constraints.video.deviceId = {exact: deviceId};
        else if (facingMode) constraints.video.facingMode = {ideal: facingMode};
        currentStream = await navigator.mediaDevices.getUserMedia(constraints);
        preview.srcObject = currentStream;
        videoTrack = currentStream.getVideoTracks()[0];
        try{ currentDeviceId = videoTrack.getSettings().deviceId || null; }catch(e){ currentDeviceId = null; }
        trackCapabilities = videoTrack.getCapabilities ? videoTrack.getCapabilities() : {};
        hardwareZoomSupported = typeof trackCapabilities.zoom !== 'undefined';
        status.textContent = 'カメラ接続 — OK';
        await enumerateDevices();
        updateDeviceLabel();
        updateLastThumb();
      }catch(err){ console.error('startCamera',err); handleGetUserMediaError(err); }
    }

    function stopCamera(){
      if (mediaRecorder && mediaRecorder.state !== 'inactive'){ try{ mediaRecorder.stop(); }catch(e){} }
      if (currentStream){ currentStream.getTracks().forEach(t=>t.stop()); currentStream=null; videoTrack=null; }
      preview.srcObject = null; recording=false; recordIndicator.style.display='none'; clearRecTimer(); updateModeUI();
    }

    function handleGetUserMediaError(err){ const name = err && err.name || ''; if (name==='NotAllowedError' || name==='PermissionDeniedError'){ status.textContent = 'カメラアクセスが許可されていません。ブラウザの設定でカメラを許可してください。'; } else if (name==='NotFoundError' || name==='OverconstrainedError'){ status.textContent = 'カメラが見つかりません。接続や機種を確認してください。'; } else if (name==='SecurityError'){ status.textContent = 'セキュリティ制約: HTTPS で開いているか確認してください。'; } else { status.textContent = 'カメラの起動に失敗しました: ' + (err && err.message || name); } hint.textContent = 'iPhoneではHTTPSまたはlocalhostで実行してください。'; }

    function takePhoto(){
      if (!currentStream) return alert('カメラが有効ではありません');
      const w = preview.videoWidth || 1280; const h = preview.videoHeight || 960;
      canvas.width = w; canvas.height = h;
      const ctx = canvas.getContext('2d');
      // digital zoom capture
      const sx = (w - w/zoom) / 2; const sy = (h - h/zoom) / 2; const sw = w/zoom; const sh = h/zoom;
      try{ ctx.drawImage(preview, sx, sy, sw, sh, 0, 0, w, h); }catch(e){ console.error('drawImage',e); }

      // HDR simulation: if HDR enabled, perform simple exposure merge (lightweight)
      if (hdrBtn.classList.contains('active')){
        // lightweight two-exposure blend simulation
        const base = ctx.getImageData(0,0,w,h);
        // create a slightly bright version and blend
        for (let i=0;i<base.data.length;i+=4){ base.data[i] = Math.min(255, base.data[i]*1.08); base.data[i+1] = Math.min(255, base.data[i+1]*1.08); base.data[i+2] = Math.min(255, base.data[i+2]*1.08); }
        ctx.putImageData(base,0,0);
      }

      const dataUrl = canvas.toDataURL('image/jpeg', 0.92);
      addToGallery(dataUrl, 'image/jpeg'); updateLastThumb();
      if (shutterSoundOn) playShutterSound();
    }

    function playShutterSound(){
      const audio = new Audio();
      // short synthesized click via data URI (tiny WAV-like) — fallback to beep
      audio.src = 'data:audio/wav;base64,UklGRiQAAABXQVZFZm10IBAAAAABAAEAESsAACJWAAACABAAZGF0YQAAAAA=';
      audio.play().catch(()=>{});
    }

    function startRecording(){
      if (!currentStream) return alert('カメラが有効ではありません');
      if (typeof MediaRecorder === 'undefined'){ alert('このブラウザはMediaRecorder APIに対応していません。動画は記録できません。'); return; }
      recordedChunks = [];
      try{ mediaRecorder = new MediaRecorder(currentStream, {mimeType:'video/webm;codecs=vp9'}); }catch(e){ try{ mediaRecorder = new MediaRecorder(currentStream); }catch(err){ console.error('MediaRecorder',err); alert('録画開始に失敗しました: '+(err.message||err)); return; } }
      mediaRecorder.ondataavailable = e=>{ if (e.data && e.data.size>0) recordedChunks.push(e.data); }
      mediaRecorder.onstop = ()=>{ const blob = new Blob(recordedChunks, {type: recordedChunks[0]?recordedChunks[0].type:'video/webm'}); const url = URL.createObjectURL(blob); addToGallery(url, blob.type, blob); updateLastThumb(url); }
      mediaRecorder.start(); recording=true; recordIndicator.style.display='inline-block'; recStart=Date.now(); recInterval=setInterval(updateRecTimer,250); updateModeUI();
    }
    function stopRecording(){ if (!mediaRecorder) return; try{ mediaRecorder.stop(); }catch(e){ console.error(e); } recording=false; recordIndicator.style.display='none'; clearRecTimer(); updateModeUI(); }
    function updateRecTimer(){ if (!recStart) return; const s = Math.floor((Date.now()-recStart)/1000); recTimer.textContent = new Date(s*1000).toISOString().substr(14,5); }
    function clearRecTimer(){ clearInterval(recInterval); recInterval=null; recStart=null; recTimer.textContent='00:00'; }

    function addToGallery(url, type='image/jpeg', blob=null){ const a=document.createElement('a'); a.href=url; a.download = type.startsWith('image')?`photo_${Date.now()}.jpg`:`video_${Date.now()}.webm`; a.target='_blank'; const img=document.createElement('img'); img.src=url; a.appendChild(img); gallery.prepend(a); }
    function updateLastThumb(url=null){ if (url){ lastThumb.src=url; lastThumb.classList.remove('hidden'); return; } const first = gallery.querySelector('a img'); if (first){ lastThumb.src = first.src; lastThumb.classList.remove('hidden'); } }

    // Zoom: hardware if supported else digital (CSS + capture crop)
    async function setZoom(value){ zoom = value; zoomSlider.value = value; if (videoTrack && hardwareZoomSupported){ try{ await videoTrack.applyConstraints({advanced:[{zoom:value}]}); }catch(e){ console.warn('applyConstraints zoom failed', e); preview.style.transform = `scale(${value})`; } } else { preview.style.transform = `scale(${value})`; } }

    zoomSlider.addEventListener('input', (e)=> setZoom(parseFloat(e.target.value)||1));

    // pinch-to-zoom for touch
    let lastDist = null;
    preview.addEventListener('touchstart', e=>{ if (e.touches.length===2) lastDist = getDist(e.touches[0], e.touches[1]); }, {passive:false});
    preview.addEventListener('touchmove', e=>{ if (e.touches.length===2){ e.preventDefault(); const d = getDist(e.touches[0], e.touches[1]); const delta = d - lastDist; if (Math.abs(delta) > 4){ const factor = delta>0?1.02:0.98; let v = Math.max(1, Math.min(4, zoom*factor)); setZoom(v); } lastDist = d; } }, {passive:false});
    preview.addEventListener('touchend', e=>{ if (e.touches.length<2) lastDist=null; });
    function getDist(a,b){ const dx=a.clientX-b.clientX; const dy=a.clientY-b.clientY; return Math.hypot(dx,dy); }

    // Shutter behaviour
    shutter.addEventListener('click', async ()=>{
      if (timerSeconds>0){ status.textContent = `タイマー: ${timerSeconds} 秒`; await wait(timerSeconds*1000); }
      if (currentMode==='photo' || currentMode==='portrait'){
        if (burstMode) { for(let i=0;i<5;i++){ takePhoto(); await wait(150); } }
        else takePhoto();
      } else if (currentMode==='video'){
        if (!recording) startRecording(); else stopRecording();
      }
    });

    // switch camera
    switchCam.addEventListener('click', async ()=>{ await enumerateDevices(); if (devices.length<=1) return alert('切替可能なカメラが見つかりません'); const idx = devices.findIndex(d=>d.deviceId===currentDeviceId); const next = devices[(idx+1+devices.length)%devices.length]; startCamera(next.deviceId); });

    enableBtn.addEventListener('click', async ()=>{ if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia){ alert('このブラウザはカメラに対応していません'); return; } await startCamera(null,'environment'); enableBtn.style.display='none'; });

    // Burst toggle
    burstBtn.addEventListener('click', ()=>{ burstMode = !burstMode; burstBtn.classList.toggle('active', burstMode); burstBtn.textContent = burstMode? 'バースト:ON' : 'バースト'; });

    // Timer cycle (0 -> 3 -> 10 -> 0)
    timerBtn.addEventListener('click', ()=>{ timerSeconds = timerSeconds===0?3: timerSeconds===3?10:0; timerBtn.textContent = timerSeconds===0? 'タイマー：なし' : `タイマー：${timerSeconds}s`; });

    // Grid overlay
    gridBtn.addEventListener('click', ()=>{ gridOverlay.classList.toggle('visible'); gridBtn.classList.toggle('active'); });

    // Torch (if supported)
    torchBtn.addEventListener('click', async ()=>{
      if (!videoTrack) return alert('カメラが有効ではありません');
      const caps = trackCapabilities || (videoTrack.getCapabilities?videoTrack.getCapabilities():{});
      if (!caps.torch){ return alert('この端末/ブラウザはトーチ制御をサポートしていません'); }
      try{ torchOn = !torchOn; await videoTrack.applyConstraints({advanced:[{torch:torchOn}]}); torchBtn.textContent = torchOn? 'ライト:ON' : 'ライト'; }catch(e){ console.error('torch',e); alert('ライト切替に失敗しました'); }
    });

    // Flash and HDR toggles (UI only; flash cannot be controlled from web reliably)
    flashBtn.addEventListener('click', ()=>{ flashBtn.classList.toggle('active'); flashBtn.textContent = flashBtn.classList.contains('active')? '⚡ 強制' : '⚡ 自動'; });
    hdrBtn.addEventListener('click', ()=>{ hdrBtn.classList.toggle('active'); hdrBtn.textContent = hdrBtn.classList.contains('active')? 'HDR:ON' : 'HDR'; });

    // Shutter sound
    shutterSoundBtn.addEventListener('click', ()=>{ shutterSoundOn = !shutterSoundOn; shutterSoundBtn.textContent = shutterSoundOn? 'シャッター音：ON' : 'シャッター音：OFF'; });

    // Download all
    document.getElementById('downloadAll').addEventListener('click', ()=>{ const items = gallery.querySelectorAll('a'); if (!items.length) return alert('ギャラリーにアイテムがありません'); items.forEach((a,i)=> setTimeout(()=> a.click(), i*200)); });

    // Utility
    function wait(ms){ return new Promise(res=>setTimeout(res,ms)); }
    (async ()=>{ try{ if (navigator.permissions && navigator.permissions.query){ const p = await navigator.permissions.query({name:'camera'}).catch(()=>null); if (p && p.state==='granted'){ enableBtn.style.display='inline-block'; status.textContent='カメラ権限は既に許可済み。起動してください。'; } } }catch(e){} })();

    updateModeUI();
    function updateDeviceLabel(){ document.getElementById('deviceLabel').textContent = devices.length? (devices.find(d=>d.deviceId===currentDeviceId)?.label || 'カメラ') : '—'; }

    // cleanup
    window.addEventListener('pagehide', stopCamera);
    window.addEventListener('beforeunload', stopCamera);
  </script>
</body>
</html>
