<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover" />
  <title>Pro Camera — 作者：ゆいきち</title>
  <style>
    /* iOS-like camera UI (inspired look — not a trademark copy) */
    :root{--bg:#000;--accent:#00e5ff;--muted:#9aa6b2;--white:#fff}
    *{box-sizing:border-box}
    html,body{height:100%;margin:0;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,'Helvetica Neue',Arial}
    body{background:linear-gradient(180deg,#000 0%,#05060a 100%);color:var(--white);display:flex;align-items:stretch;justify-content:center}
    .app{width:100%;max-width:820px;height:100vh;display:flex;flex-direction:column;align-items:stretch}

    /* Top bar */
    header{height:88px;padding:12px 18px;display:flex;flex-direction:column;justify-content:flex-end;gap:6px}
    .toprow{display:flex;justify-content:space-between;align-items:center}
    .btn{background:rgba(255,255,255,0.06);padding:8px 10px;border-radius:10px;border:1px solid rgba(255,255,255,0.04);font-size:14px;color:var(--white)}

    /* Viewfinder */
    .viewfinder{position:relative;flex:1;overflow:hidden;border-radius:14px;margin:0 12px;background:#000}
    video#preview{width:100%;height:100%;object-fit:cover;transform-origin:center center}
    canvas#snapshot{display:none}

    /* Overlays */
    .overlay-top{position:absolute;top:10px;left:12px;right:12px;display:flex;justify-content:space-between;pointer-events:none}
    .overlay-top .left, .overlay-top .right{pointer-events:auto}
    .mode-label{position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);font-weight:600;letter-spacing:2px}

    /* Bottom controls */
    .controls{height:160px;display:flex;flex-direction:column;align-items:center;justify-content:center;padding:10px 18px;gap:8px}
    .modes{display:flex;gap:16px;align-items:center}
    .mode{padding:6px 12px;border-radius:999px;font-weight:600;opacity:.8}
    .mode.active{background:rgba(255,255,255,0.08)}

    .shutter-row{display:flex;align-items:center;gap:20px}
    .thumbs{width:56px;height:56px;border-radius:12px;overflow:hidden;border:1px solid rgba(255,255,255,0.06)}
    .thumbs img{width:100%;height:100%;object-fit:cover}

    .shutter{width:88px;height:88px;border-radius:50%;background:linear-gradient(180deg,#fff,#ddd);display:flex;align-items:center;justify-content:center;box-shadow:0 6px 30px rgba(0,0,0,.5);border:6px solid rgba(255,255,255,0.06)}
    .shutter.recording{background:radial-gradient(circle at 50% 50%, #ff4d4d 0%, #c60000 60%);border-color:rgba(255,255,255,0.02)}
    .small-btn{width:44px;height:44px;border-radius:999px;background:rgba(255,255,255,0.04);display:flex;align-items:center;justify-content:center}

    .toolbar{display:flex;gap:8px;align-items:center}
    .zoom-slider{width:200px}
    .status{font-size:13px;color:var(--muted);text-align:center;margin-top:6px}

    /* gallery */
    .gallery{position:absolute;right:12px;bottom:180px;display:flex;flex-direction:column;gap:8px}
    .gallery a{width:80px;height:80px;border-radius:10px;overflow:hidden;border:1px solid rgba(255,255,255,0.06)}
    .gallery img{width:100%;height:100%;object-fit:cover}

    /* small helpers */
    .hidden{display:none}
    .badge{background:rgba(0,0,0,0.6);padding:6px 8px;border-radius:999px}
    footer{height:40px;text-align:center;color:var(--muted);font-size:12px}

    @media(max-width:520px){.shutter{width:68px;height:68px}.shutter-row{gap:12px}}
  </style>
</head>
<body>
  <main class="app">
    <header>
      <div class="toprow">
        <div style="display:flex;gap:8px;align-items:center">
          <button id="flashBtn" class="btn">⚡ 自動</button>
          <button id="timerBtn" class="btn">⏱ なし</button>
        </div>
        <div style="display:flex;gap:8px;align-items:center">
          <button id="settingsBtn" class="btn">設定</button>
        </div>
      </div>
      <div style="display:flex;justify-content:center;margin-top:6px"><div class="status" id="status">準備完了 — 作者：ゆいきち</div></div>
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

      <div style="position:absolute;inset:0;display:flex;align-items:center;justify-content:center;pointer-events:none"><div class="mode-label" id="modeLabel">PHOTO</div></div>

      <video id="preview" autoplay playsinline muted></video>
      <canvas id="snapshot"></canvas>

      <div class="gallery" id="gallery"></div>
    </section>

    <section class="controls">
      <div class="modes" role="tablist" aria-label="カメラモード">
        <div class="mode active" data-mode="photo">写真</div>
        <div class="mode" data-mode="video">ビデオ</div>
        <div class="mode" data-mode="portrait">ポートレート</div>
      </div>

      <div class="toolbar">
        <div class="thumbs" id="lastThumb"><img src="" alt="" class="hidden"></div>
        <div style="display:flex;align-items:center;" id="controlsLeft">
          <button id="switchCam" class="small-btn" title="切替">⇄</button>
        </div>
        <div style="flex:1;display:flex;justify-content:center">
          <button id="shutter" class="shutter" title="シャッター"></button>
        </div>
        <div style="display:flex;align-items:center;gap:8px" id="controlsRight">
          <input type="range" id="zoom" class="zoom-slider" min="1" max="3" step="0.01" value="1" title="ズーム">
        </div>
      </div>

      <div style="display:flex;justify-content:center;gap:12px;align-items:center">
        <button id="enableBtn" class="btn">カメラを有効にする</button>
        <button id="downloadAll" class="btn">ギャラリーをダウンロード</button>
      </div>

      <div class="status" id="hint">iPhoneではHTTPS（またはlocalhost）で動作させてください。動画はブラウザのサポートによります。</div>
    </section>

    <footer>© ゆいきち — Pro Camera (web版)</footer>
  </main>

  <script>
    // Advanced camera app: photo + video + UI like Apple Camera (inspired)
    // Important: still requires user gesture to start camera

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

    function updateModeUI(){
      modeEls.forEach(el=>el.classList.toggle('active', el.dataset.mode===currentMode));
      modeLabel.textContent = currentMode.toUpperCase();
      // shutter style for video
      shutter.classList.toggle('recording', currentMode==='video' && recording);
    }

    modeEls.forEach(el=>el.addEventListener('click', ()=>{
      currentMode = el.dataset.mode;
      updateModeUI();
    }));

    // get available video devices
    async function enumerateDevices(){
      try{
        const list = await navigator.mediaDevices.enumerateDevices();
        devices = list.filter(d=>d.kind==='videoinput');
        return devices;
      }catch(e){
        console.error('enumerateDevices', e);
        return [];
      }
    }

    async function startCamera(deviceId=null, facingMode='environment'){
      stopCamera();
      try{
        status.textContent = 'カメラ起動中…';
        let constraints = {audio: true, video: {width:{ideal:1280},height:{ideal:960}}};
        if (deviceId) constraints.video.deviceId = {exact: deviceId};
        else if (facingMode) constraints.video.facingMode = {ideal: facingMode};

        currentStream = await navigator.mediaDevices.getUserMedia(constraints);
        preview.srcObject = currentStream;
        // try to get deviceId from track settings
        const track = currentStream.getVideoTracks()[0];
        try{ currentDeviceId = track.getSettings().deviceId || null; }catch(e){currentDeviceId=null}

        await enumerateDevices();
        status.textContent = 'カメラ接続 — OK';

        // set last thumbnail if any
        updateLastThumb();
      }catch(err){
        console.error('startCamera', err);
        handleGetUserMediaError(err);
      }
    }

    function stopCamera(){
      if (mediaRecorder && mediaRecorder.state !== 'inactive'){
        try{ mediaRecorder.stop(); }catch(e){}
      }
      if (currentStream){
        currentStream.getTracks().forEach(t=>t.stop());
        currentStream = null;
      }
      preview.srcObject = null;
      recording = false;
      recordIndicator.style.display = 'none';
      clearRecTimer();
      updateModeUI();
    }

    function handleGetUserMediaError(err){
      const name = err && err.name || '';
      if (name==='NotAllowedError' || name==='PermissionDeniedError'){
        status.textContent = 'カメラアクセスが許可されていません。ブラウザの設定でカメラを許可してください。';
      } else if (name==='NotFoundError' || name==='OverconstrainedError'){
        status.textContent = 'カメラが見つかりません。接続や機種を確認してください。';
      } else if (name==='SecurityError'){
        status.textContent = 'セキュリティ制約: HTTPS で開いているか確認してください。';
      } else {
        status.textContent = 'カメラの起動に失敗しました: ' + (err && err.message || name);
      }
      hint.textContent = 'iPhoneではHTTPSまたはlocalhostで実行してください。';
    }

    // Photo capture
    function takePhoto(){
      if (!currentStream) return alert('カメラが有効ではありません');
      const videoTrack = currentStream.getVideoTracks()[0];
      const w = preview.videoWidth || 1280;
      const h = preview.videoHeight || 960;
      canvas.width = w;
      canvas.height = h;
      const ctx = canvas.getContext('2d');
      ctx.save();
      // apply zoom by scaling drawImage center
      const sx = (w - w/zoom) / 2;
      const sy = (h - h/zoom) / 2;
      const sw = w/zoom;
      const sh = h/zoom;
      try{ ctx.drawImage(preview, sx, sy, sw, sh, 0, 0, w, h); }catch(e){ console.error('drawImage failed',e); }
      ctx.restore();
      const dataUrl = canvas.toDataURL('image/jpeg', 0.92);
      addToGallery(dataUrl, 'image/jpeg');
      updateLastThumb();
    }

    // Video recording
    function startRecording(){
      if (!currentStream) return alert('カメラが有効ではありません');
      if (typeof MediaRecorder === 'undefined'){
        alert('このブラウザはMediaRecorder APIに対応していません。動画は記録できません。');
        return;
      }
      recordedChunks = [];
      try{
        mediaRecorder = new MediaRecorder(currentStream, {mimeType: 'video/webm;codecs=vp9'})
      }catch(e){
        try{ mediaRecorder = new MediaRecorder(currentStream); }catch(err){ console.error('MediaRecorder error', err); alert('録画開始に失敗しました: '+(err.message||err)); return; }
      }
      mediaRecorder.ondataavailable = e=>{ if (e.data && e.data.size>0) recordedChunks.push(e.data); }
      mediaRecorder.onstop = ()=>{
        const blob = new Blob(recordedChunks, {type: recordedChunks[0] ? recordedChunks[0].type : 'video/webm'});
        const url = URL.createObjectURL(blob);
        addToGallery(url, blob.type, blob);
        updateLastThumb(url);
      }
      mediaRecorder.start();
      recording = true;
      recordIndicator.style.display = 'inline-block';
      recStart = Date.now();
      recInterval = setInterval(updateRecTimer, 250);
      updateModeUI();
    }

    function stopRecording(){
      if (!mediaRecorder) return;
      try{ mediaRecorder.stop(); }catch(e){ console.error(e); }
      recording = false;
      recordIndicator.style.display = 'none';
      clearRecTimer();
      updateModeUI();
    }

    function updateRecTimer(){
      if (!recStart) return; const s = Math.floor((Date.now()-recStart)/1000); recTimer.textContent = new Date(s*1000).toISOString().substr(14,5);
    }
    function clearRecTimer(){ clearInterval(recInterval); recInterval=null; recStart=null; recTimer.textContent='00:00'; }

    // Gallery
    function addToGallery(url, type='image/jpeg', blob=null){
      const a = document.createElement('a');
      a.href = url;
      a.download = type.startsWith('image') ? `photo_${Date.now()}.jpg` : `video_${Date.now()}.webm`;
      a.target = '_blank';
      const img = document.createElement('img');
      // for videos, show a poster frame if blob provided by creating an object URL (we already did)
      img.src = url;
      a.appendChild(img);
      gallery.prepend(a);
    }

    function updateLastThumb(url=null){
      if (url){ lastThumb.src = url; lastThumb.classList.remove('hidden'); return; }
      const first = gallery.querySelector('a img');
      if (first){ lastThumb.src = first.src; lastThumb.classList.remove('hidden'); }
    }

    // Zoom control (simple digital zoom via CSS transform for preview and scaled draw for capture)
    zoomSlider.addEventListener('input', (e)=>{
      zoom = parseFloat(e.target.value)||1;
      preview.style.transform = `scale(${zoom})`;
    });

    // Shutter behaviour depends on mode
    shutter.addEventListener('click', ()=>{
      if (currentMode==='photo' || currentMode==='portrait'){
        // for portrait we could apply simple blur simulation; for now capture and mark
        takePhoto();
      } else if (currentMode==='video'){
        if (!recording) startRecording(); else stopRecording();
      }
    });

    // switch camera - pick next available device
    switchCam.addEventListener('click', async ()=>{
      await enumerateDevices();
      if (devices.length<=1) return alert('切替可能なカメラが見つかりません');
      const idx = devices.findIndex(d=>d.deviceId===currentDeviceId);
      const next = devices[(idx+1+devices.length)%devices.length];
      startCamera(next.deviceId);
    });

    // enable button
    enableBtn.addEventListener('click', async ()=>{
      if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia){ alert('このブラウザはカメラに対応していません'); return; }
      await startCamera(null, 'environment');
      // hide enable
      enableBtn.style.display='none';
    });

    // download all gallery items as individual links (not a zip) — zipping requires server or JS library
    document.getElementById('downloadAll').addEventListener('click', ()=>{
      const items = gallery.querySelectorAll('a');
      if (!items.length) return alert('ギャラリーにアイテムがありません');
      items.forEach((a, i)=> setTimeout(()=> a.click(), i*200));
    });

    // permissions: if already granted, reveal enable
    (async ()=>{
      try{
        if (navigator.permissions && navigator.permissions.query){
          const p = await navigator.permissions.query({name:'camera'}).catch(()=>null);
          if (p && p.state==='granted'){ enableBtn.style.display='inline-block'; status.textContent='カメラ権限は既に許可済み。起動してください。'; }
        }
      }catch(e){}
    })();

    // cleanup
    window.addEventListener('pagehide', stopCamera);
    window.addEventListener('beforeunload', stopCamera);

    // Inform the user about MediaRecorder availability
    (function(){ if (typeof MediaRecorder === 'undefined') hint.textContent += '（注: このブラウザでは動画録画がサポートされていない可能性があります）'; })();

    // initial UI
    updateModeUI();
  </script>
</body>
</html>
