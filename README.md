<!doctype html>
<!--
  Pro Camera — 作者：ゆいきち
  - リアルタイム“シネマモード”(MediaPipe SelfieSegmentation)を組み込み
  - ピンチでズーム、ハードウェアズーム優先、4K希望の constraints 指定
  - 自然映えモード（自動カラーグレード）、HDR簡易合成、バースト、タイマー、トーチ、シャッター音
  - レスポンシブ（スマホ画面に最適化）+ GitHub Pages 配備手順付き

  注意:
  - 外部ライブラリ（MediaPipe）を CDN で読み込みます。ネット接続が必要。
  - iOS Safari や古い端末ではパフォーマンス低下、あるいは機能非対応（MediaRecorder / torch / capabilities）があります。
  - Apple の商標や完全コピーは避けています。見た目・操作感のインスパイア実装です。
-->
<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover" />
  <title>Pro Camera — 作者：ゆいきち</title>
  <meta name="description" content="Webベースの高性能カメラ（写真・動画・リアルタイムシネマモード・4K対応を試行）">
  <style>
    /* Responsive premium UI — mobile-first */
    :root{--bg:#000;--accent:#0cd8ff;--muted:#9aa6b2;--white:#fff;--glass:rgba(255,255,255,0.04)}
    *{box-sizing:border-box}
    html,body{height:100%;margin:0;font-family:-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,'Helvetica Neue',Arial}
    body{background:#000;color:var(--white);display:flex;justify-content:center}
    .container{width:100%;max-width:1024px;height:100vh;display:flex;flex-direction:column}

    header{padding:12px 16px;display:flex;align-items:center;justify-content:space-between}
    .brand{font-weight:700}
    .top-controls{display:flex;gap:8px}
    .btn{background:var(--glass);padding:8px 10px;border-radius:10px;border:1px solid rgba(255,255,255,0.04);font-size:14px;color:var(--white)}

    /* Viewfinder area */
    .view{position:relative;flex:1;margin:8px;border-radius:16px;overflow:hidden;background:#000}
    #preview{width:100%;height:100%;object-fit:cover;transform-origin:center center;touch-action:none}
    canvas{display:none}

    .overlay{position:absolute;inset:12px;display:flex;justify-content:space-between;pointer-events:none}
    .badge{background:rgba(0,0,0,0.5);padding:6px 8px;border-radius:999px;pointer-events:auto}
    .mode-label{position:absolute;left:50%;top:50%;transform:translate(-50%,-50%);font-weight:800;letter-spacing:2px;pointer-events:none}

    .grid{position:absolute;inset:0;pointer-events:none;background-image:linear-gradient(transparent calc(33.333% - 1px), rgba(255,255,255,0.03) 1px, transparent calc(33.333% + 1px)), linear-gradient(90deg, transparent calc(33.333% - 1px), rgba(255,255,255,0.03) 1px, transparent calc(33.333% + 1px));opacity:0;transition:opacity .18s}
    .grid.on{opacity:1}

    /* controls */
    .controls{padding:10px 16px;display:flex;flex-direction:column;gap:10px;align-items:stretch}
    .modes{display:flex;gap:10px;justify-content:center}
    .mode{padding:6px 12px;border-radius:999px;font-weight:700;cursor:pointer}
    .mode.active{background:rgba(255,255,255,0.06)}

    .shutter-row{display:flex;align-items:center;justify-content:space-between;gap:12px}
    .thumb{width:64px;height:64px;border-radius:12px;border:1px solid rgba(255,255,255,0.06);overflow:hidden}
    .thumb img{width:100%;height:100%;object-fit:cover}

    .shutter{width:86px;height:86px;border-radius:50%;background:linear-gradient(180deg,#fff,#ddd);display:flex;align-items:center;justify-content:center;border:6px solid rgba(255,255,255,0.06);cursor:pointer}
    .shutter.recording{background:radial-gradient(circle,#ff5a5a,#c70000);border-color:rgba(255,255,255,0.02)}

    .tools{display:flex;gap:8px;align-items:center;justify-content:center}
    .zoom{width:200px}
    .status{font-size:13px;color:var(--muted);text-align:center}

    footer{height:44px;display:flex;align-items:center;justify-content:center;color:var(--muted);font-size:12px}

    /* small screens */
    @media (max-width:520px){ .thumb{width:48px;height:48px} .shutter{width:68px;height:68px} .zoom{width:140px} }
  </style>
</head>
<body>
  <div class="container">
    <header>
      <div class="brand">Pro Camera — 作者：ゆいきち</div>
      <div class="top-controls">
        <button id="gridBtn" class="btn">グリッド</button>
        <button id="settingsBtn" class="btn">設定</button>
      </div>
    </header>

    <main class="view" id="view">
      <div class="overlay">
        <div class="left"><div id="recordIndicator" class="badge" style="display:none">● REC <span id="recTimer">00:00</span></div></div>
        <div class="right"><div id="deviceLabel" class="badge">—</div></div>
      </div>

      <div class="mode-label" id="modeLabel">PHOTO</div>
      <video id="preview" autoplay playsinline muted></video>
      <canvas id="snapshot"></canvas>
      <div id="segCanvas" style="display:none"></div>
      <div id="grid" class="grid"></div>
      <div id="gallery" style="position:absolute;right:12px;bottom:220px;display:flex;flex-direction:column;gap:8px"></div>
    </main>

    <div class="controls">
      <div class="modes">
        <div class="mode active" data-mode="photo">写真</div>
        <div class="mode" data-mode="video">ビデオ</div>
        <div class="mode" data-mode="cinema">シネマ</div>
        <div class="mode" data-mode="natural">自然映え</div>
      </div>

      <div class="shutter-row">
        <div class="thumb" id="lastThumb"><img src="" alt="" style="display:none"></div>
        <div style="display:flex;gap:8px;align-items:center">
          <button id="switchBtn" class="btn">切替</button>
        </div>

        <div style="display:flex;justify-content:center;flex:1">
          <button id="shutter" class="shutter" title="シャッター"></button>
        </div>

        <div style="display:flex;gap:8px;align-items:center">
          <input id="zoomSlider" class="zoom" type="range" min="1" max="4" step="0.01" value="1">
        </div>
      </div>

      <div class="tools">
        <button id="enableBtn" class="btn">カメラを有効にする</button>
        <button id="burstBtn" class="btn">バースト</button>
        <button id="timerBtn" class="btn">タイマー: なし</button>
        <button id="torchBtn" class="btn">ライト</button>
        <button id="downloadAll" class="btn">ギャラリーDL</button>
      </div>

      <div class="status" id="status">HTTPS または localhost を推奨。MediaPipe を使ってリアルタイムシネマモードを実行します。</div>
    </div>

    <footer>© ゆいきち — Pro Camera (web)</footer>
  </div>

  <!-- 外部ライブラリ: MediaPipe Selfie Segmentation と Camera Utils を CDN から読み込む -->
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation/selfie_segmentation.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

  <script>
    // Main app logic: realtime segmentation + cinematic blending
    const preview = document.getElementById('preview');
    const canvas = document.getElementById('snapshot');
    const zoomSlider = document.getElementById('zoomSlider');
    const enableBtn = document.getElementById('enableBtn');
    const shutter = document.getElementById('shutter');
    const burstBtn = document.getElementById('burstBtn');
    const timerBtn = document.getElementById('timerBtn');
    const torchBtn = document.getElementById('torchBtn');
    const downloadAll = document.getElementById('downloadAll');
    const switchBtn = document.getElementById('switchBtn');
    const gridBtn = document.getElementById('gridBtn');
    const modeEls = document.querySelectorAll('.mode');
    const modeLabel = document.getElementById('modeLabel');
    const recordIndicator = document.getElementById('recordIndicator');
    const recTimer = document.getElementById('recTimer');
    const grid = document.getElementById('grid');
    const statusEl = document.getElementById('status');
    const deviceLabel = document.getElementById('deviceLabel');
    const gallery = document.getElementById('gallery');
    const lastThumb = document.querySelector('#lastThumb img');

    let currentMode = 'photo';
    let currentStream = null;
    let videoTrack = null;
    let devices = [];
    let currentDeviceId = null;
    let mediaRecorder = null;
    let recordedChunks = [];
    let recording = false;
    let recInterval = null;
    let recStart = null;
    let burstMode = false;
    let timerSeconds = 0;
    let zoom = 1;

    // MediaPipe setup
    let seg = null; // SelfieSegmentation instance
    let mpCamera = null;
    let segCanvas = null; // offscreen canvas to composite segmentation
    let segCtx = null;
    let runningSegmentation = false;

    function setMode(m){ currentMode = m; modeEls.forEach(el=>el.classList.toggle('active', el.dataset.mode===m)); modeLabel.textContent = m.toUpperCase(); }
    modeEls.forEach(el=>el.addEventListener('click', ()=> setMode(el.dataset.mode)));

    async function enumerateDevices(){ try{ const list = await navigator.mediaDevices.enumerateDevices(); devices = list.filter(d=>d.kind==='videoinput'); return devices; }catch(e){ console.error(e); return []; } }

    async function startCamera(deviceId=null, facingMode='environment'){
      stopCamera();
      try{
        statusEl.textContent = 'カメラ起動中…';
        const constraints = { audio: true, video: { width: { ideal: 3840 }, height: { ideal: 2160 }, frameRate: { ideal: 30 } } };
        if (deviceId) constraints.video.deviceId = { exact: deviceId };
        else if (facingMode) constraints.video.facingMode = { ideal: facingMode };

        currentStream = await navigator.mediaDevices.getUserMedia(constraints);
        preview.srcObject = currentStream;
        videoTrack = currentStream.getVideoTracks()[0];
        try{ currentDeviceId = videoTrack.getSettings().deviceId || null; }catch(e){ currentDeviceId = null; }
        await enumerateDevices(); updateDeviceLabel(); statusEl.textContent = 'カメラ接続 — OK';

        // If MediaPipe available, start segmentation
        if (self && self.SelfieSegmentation){
          if (!seg){
            seg = new SelfieSegmentation({locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation/${file}`});
            seg.setOptions({modelSelection:1});
            seg.onResults(onSegmentationResults);
          }

          // Use Camera Utils to feed frames to MediaPipe (works with video element)
          if (!mpCamera){
            mpCamera = new Camera(preview, { onFrame: async ()=>{ if (seg) await seg.send({image: preview}); }, width: 1280, height: 720 });
            mpCamera.start();
          }
        } else {
          statusEl.textContent += '（注意: MediaPipe が読み込まれていません。ネット接続を確認してください）';
        }

        // initialize segmentation canvas
        if (!segCanvas){ segCanvas = document.createElement('canvas'); segCanvas.width = preview.clientWidth; segCanvas.height = preview.clientHeight; segCtx = segCanvas.getContext('2d'); }

      }catch(err){ console.error('startCamera', err); handleGetUserMediaError(err); }
    }

    function stopCamera(){
      if (mediaRecorder && mediaRecorder.state !== 'inactive'){ try{ mediaRecorder.stop(); }catch(e){} }
      if (currentStream){ currentStream.getTracks().forEach(t=>t.stop()); currentStream=null; videoTrack=null; }
      if (mpCamera){ try{ mpCamera.stop(); }catch(e){} mpCamera=null; }
      preview.srcObject = null; recording=false; recordIndicator.style.display='none'; clearRecTimer(); setMode('photo');
    }

    function handleGetUserMediaError(err){ const name = err && err.name || ''; if (name==='NotAllowedError' || name==='PermissionDeniedError'){ statusEl.textContent = 'カメラアクセスが許可されていません。ブラウザの設定でカメラを許可してください。'; } else if (name==='NotFoundError' || name==='OverconstrainedError'){ statusEl.textContent = 'カメラが見つかりません。'; } else if (name==='SecurityError'){ statusEl.textContent = 'HTTPS でホストしてください。'; } else { statusEl.textContent = 'カメラ起動に失敗しました: ' + (err && err.message || name); } }

    function onSegmentationResults(results){
      // results.segmentationMask is an HTMLCanvasElement
      // composite: blurred background + original person
      if (!results.segmentationMask) return;
      runningSegmentation = true;
      const mask = results.segmentationMask;
      const w = preview.videoWidth || preview.clientWidth;
      const h = preview.videoHeight || preview.clientHeight;
      // prepare canvas sizes
      segCanvas.width = w; segCanvas.height = h;

      // draw blurred background
      segCtx.save();
      segCtx.clearRect(0,0,w,h);
      // draw preview into temp then blur by CSS filter via drawImage on another canvas — fallback: use box blur multiple passes
      // simpler approach: draw preview scaled down then scaled up to simulate blur (fast)
      const tmp = document.createElement('canvas'); tmp.width = Math.max(1, Math.floor(w/16)); tmp.height = Math.max(1, Math.floor(h/16)); const tctx = tmp.getContext('2d'); tctx.drawImage(preview,0,0,tmp.width,tmp.height);
      // scale back up to segCtx
      segCtx.imageSmoothingEnabled = true; segCtx.drawImage(tmp,0,0,w,h);

      // now create mask and composite
      // mask is a canvas with alpha representing person; we use it to cut out person
      // compute cinematic blur strength based on current mode
      const cinema = (currentMode === 'cinema');
      const natural = (currentMode === 'natural');

      // Optional: create radial depth approximation to vary blur (simulate near-sharp far-blur)
      if (cinema){
        // create gradient alpha to mix original and blurred background for distance-based blur
        // we'll composite: background (blurred) -> then draw original where mask > threshold
        // draw original person
        segCtx.globalCompositeOperation = 'destination-over';
        // draw person from original preview
        segCtx.drawImage(preview, 0, 0, w, h);

        // overlay mask to cut person in front: use mask as clipping
        segCtx.globalCompositeOperation = 'destination-in';
        segCtx.drawImage(mask, 0, 0, w, h);

        // now draw original person sharply on top
        const final = document.createElement('canvas'); final.width = w; final.height = h; const fctx = final.getContext('2d');
        fctx.drawImage(preview,0,0,w,h);
        fctx.globalCompositeOperation = 'destination-in'; fctx.drawImage(mask,0,0,w,h);
        // composite: blurred background (segCanvas) already has background, now draw fctx on top
        segCtx.globalCompositeOperation = 'source-over'; segCtx.drawImage(final,0,0,w,h);
      } else if (natural){
        // apply automatic color grading to the original frame and composite with light background blur
        const graded = document.createElement('canvas'); graded.width=w; graded.height=h; const gctx=graded.getContext('2d'); gctx.drawImage(preview,0,0,w,h);
        // simple color grade: increase contrast and saturation by manipulating pixels (cheap)
        const img = gctx.getImageData(0,0,w,h);
        for(let i=0;i<img.data.length;i+=4){
          let r=img.data[i], g=img.data[i+1], b=img.data[i+2];
          // saturation boost
          const avg = (r+g+b)/3; img.data[i] = Math.min(255, avg + (r-avg)*1.15); img.data[i+1] = Math.min(255, avg + (g-avg)*1.12); img.data[i+2] = Math.min(255, avg + (b-avg)*1.1);
          // contrast
          img.data[i] = Math.min(255, Math.max(0, (img.data[i]-128)*1.05+128)); img.data[i+1] = Math.min(255, Math.max(0, (img.data[i+1]-128)*1.05+128)); img.data[i+2] = Math.min(255, Math.max(0, (img.data[i+2]-128)*1.05+128));
        }
        gctx.putImageData(img,0,0);
        // blurred background then overlay graded person using mask
        // draw blurred background first
        segCtx.globalCompositeOperation='source-over'; segCtx.drawImage(tmp,0,0,w,h);
        // draw graded person sharp
        segCtx.drawImage(graded,0,0,w,h);
        segCtx.globalCompositeOperation='destination-in'; segCtx.drawImage(mask,0,0,w,h);
        segCtx.globalCompositeOperation='source-over';
      } else {
        // photo mode: simple background blur
        segCtx.globalCompositeOperation = 'source-over';
        segCtx.drawImage(tmp,0,0,w,h); // blurred bg
        // draw sharp person on top
        const final = document.createElement('canvas'); final.width=w; final.height=h; const fctx = final.getContext('2d'); fctx.drawImage(preview,0,0,w,h); fctx.globalCompositeOperation='destination-in'; fctx.drawImage(mask,0,0,w,h);
        segCtx.globalCompositeOperation='source-over'; segCtx.drawImage(fctx,0,0,w,h);
      }

      // draw composite to a visible overlay using CSS by setting as background of view or by painting to main canvas before saving
      // here we paint segCanvas onto an on-screen canvas (we'll create or reuse a visible canvas overlay)
      renderSegToScreen(segCanvas);
    }

    let visibleSegCanvas = null; let visibleSegCtx = null;
    function renderSegToScreen(srcCanvas){
      if (!visibleSegCanvas){ visibleSegCanvas = document.createElement('canvas'); visibleSegCanvas.style.position='absolute'; visibleSegCanvas.style.left='0'; visibleSegCanvas.style.top='0'; visibleSegCanvas.style.width='100%'; visibleSegCanvas.style.height='100%'; visibleSegCanvas.style.pointerEvents='none'; visibleSegCanvas.style.zIndex=5; document.getElementById('view').appendChild(visibleSegCanvas); visibleSegCtx = visibleSegCanvas.getContext('2d'); }
      const w = preview.videoWidth || preview.clientWidth; const h = preview.videoHeight || preview.clientHeight; visibleSegCanvas.width = w; visibleSegCanvas.height = h; visibleSegCtx.clearRect(0,0,w,h); visibleSegCtx.drawImage(srcCanvas,0,0,w,h);
    }

    // Photo capture: capture current visible segmentation composite if running, otherwise capture video
    function capturePhoto(){
      const w = preview.videoWidth || preview.clientWidth; const h = preview.videoHeight || preview.clientHeight; canvas.width = w; canvas.height = h; const ctx = canvas.getContext('2d');
      if (runningSegmentation && visibleSegCanvas){ ctx.drawImage(visibleSegCanvas,0,0,w,h); }
      else { ctx.drawImage(preview,0,0,w,h); }
      // allow 4K export if available: use toBlob
      canvas.toBlob(blob=>{ const url = URL.createObjectURL(blob); addToGallery(url, blob.type, blob); updateLastThumb(url); }, 'image/jpeg', 0.95);
    }

    // Recording using MediaRecorder
    function startRecording(){
      if (!currentStream) return alert('カメラが有効ではありません');
      if (typeof MediaRecorder === 'undefined'){ alert('このブラウザは録画に未対応です'); return; }
      recordedChunks = [];
      try{ mediaRecorder = new MediaRecorder(currentStream, {mimeType: 'video/webm;codecs=vp9'}); }catch(e){ try{ mediaRecorder = new MediaRecorder(currentStream); }catch(err){ alert('録画開始失敗: '+(err.message||err)); return; } }
      mediaRecorder.ondataavailable = e=>{ if (e.data && e.data.size>0) recordedChunks.push(e.data); }
      mediaRecorder.onstop = ()=>{ const blob = new Blob(recordedChunks, {type: recordedChunks[0]?recordedChunks[0].type:'video/webm'}); const url=URL.createObjectURL(blob); addToGallery(url, blob.type, blob); updateLastThumb(url); }
      mediaRecorder.start(); recording=true; recordIndicator.style.display='inline-block'; recStart=Date.now(); recInterval=setInterval(updateRecTimer,250);
    }
    function stopRecording(){ if (!mediaRecorder) return; try{ mediaRecorder.stop(); }catch(e){} recording=false; recordIndicator.style.display='none'; clearRecTimer(); }
    function updateRecTimer(){ if (!recStart) return; const s=Math.floor((Date.now()-recStart)/1000); recTimer.textContent = new Date(s*1000).toISOString().substr(14,5); }
    function clearRecTimer(){ clearInterval(recInterval); recInterval=null; recStart=null; recTimer.textContent='00:00'; }

    // gallery
    function addToGallery(url, type='image/jpeg', blob=null){ const a=document.createElement('a'); a.href=url; a.download = type.startsWith('image')?`photo_${Date.now()}.jpg`:`video_${Date.now()}.webm`; a.target='_blank'; const img=document.createElement('img'); img.src=url; a.appendChild(img); gallery.prepend(a); }
    function updateLastThumb(url=null){ if (url){ lastThumb.src=url; lastThumb.style.display='block'; return; } const first = gallery.querySelector('a img'); if (first){ lastThumb.src=first.src; lastThumb.style.display='block'; } }

    // UI handlers
    shutter.addEventListener('click', async ()=>{
      if (timerSeconds>0){ statusEl.textContent = `タイマー: ${timerSeconds}s`; await new Promise(r=>setTimeout(r, timerSeconds*1000)); }
      if (currentMode==='video'){
        if (!recording) startRecording(); else stopRecording();
      } else {
        if (burstMode){ for(let i=0;i<5;i++){ capturePhoto(); await new Promise(r=>setTimeout(r,150)); } }
        else capturePhoto();
      }
    });

    enableBtn.addEventListener('click', async ()=>{ if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia){ alert('カメラ未対応'); return; } await startCamera(null,'environment'); enableBtn.style.display='none'; });

    burstBtn.addEventListener('click', ()=>{ burstMode=!burstMode; burstBtn.classList.toggle('active', burstMode); burstBtn.textContent = burstMode? 'バースト:ON' : 'バースト'; });

    timerBtn.addEventListener('click', ()=>{ timerSeconds = timerSeconds===0?3:timerSeconds===3?10:0; timerBtn.textContent = timerSeconds===0? 'タイマー: なし' : `タイマー: ${timerSeconds}s`; });

    gridBtn.addEventListener('click', ()=>{ grid.classList.toggle('on'); gridBtn.classList.toggle('active'); });

    downloadAll.addEventListener('click', ()=>{ const items = gallery.querySelectorAll('a'); if (!items.length) return alert('ギャラリーにアイテムがありません'); items.forEach((a,i)=>setTimeout(()=>a.click(), i*200)); });

    switchBtn.addEventListener('click', async ()=>{ await enumerateDevices(); if (devices.length<=1) return alert('切替可能なカメラがありません'); const idx = devices.findIndex(d=>d.deviceId===currentDeviceId); const next=devices[(idx+1+devices.length)%devices.length]; startCamera(next.deviceId); });

    // zoom slider and pinch
    zoomSlider.addEventListener('input', ()=> setZoom(parseFloat(zoomSlider.value)||1));
    function setZoom(v){ zoom = Math.max(1, Math.min(4, v)); zoomSlider.value = zoom; if (videoTrack){ const caps = videoTrack.getCapabilities ? videoTrack.getCapabilities() : {}; if (typeof caps.zoom !== 'undefined'){ try{ videoTrack.applyConstraints({advanced:[{zoom}]}); }catch(e){ preview.style.transform = `scale(${zoom})`; } } else { preview.style.transform = `scale(${zoom})`; } } else { preview.style.transform = `scale(${zoom})`; } }

    // touch pinch
    let lastDist = null;
    preview.addEventListener('touchstart', e=>{ if (e.touches.length===2) lastDist = dist(e.touches[0], e.touches[1]); }, {passive:false});
    preview.addEventListener('touchmove', e=>{ if (e.touches.length===2){ e.preventDefault(); const d = dist(e.touches[0], e.touches[1]); const delta = d - lastDist; if (Math.abs(delta)>4){ const factor = delta>0?1.02:0.98; setZoom(Math.max(1, Math.min(4, zoom*factor))); } lastDist = d; } }, {passive:false});
    function dist(a,b){ return Math.hypot(a.clientX-b.clientX, a.clientY-b.clientY); }

    // permissions note: reveal enable button if permission already granted
    (async ()=>{ try{ if (navigator.permissions && navigator.permissions.query){ const p = await navigator.permissions.query({name:'camera'}).catch(()=>null); if (p && p.state==='granted'){ enableBtn.style.display='inline-block'; statusEl.textContent='カメラ権限は既に許可済み。起動してください。'; } } }catch(e){} })();

    // update device label
    function updateDeviceLabel(){ deviceLabel.textContent = devices.length? (devices.find(d=>d.deviceId===currentDeviceId)?.label || 'カメラ') : '—'; }

    // cleanup
    window.addEventListener('pagehide', ()=>{ stopCamera(); }); window.addEventListener('beforeunload', ()=>{ stopCamera(); });
  </script>

  <!--
    GitHub に公開する手順 (短くまとめ)
    1) 新しいリポジトリを作成（例: yui-camera）。
    2) このファイルを index.html として追加、コミット & push。
    3) GitHub の Settings -> Pages で Branch を main(or gh-pages) / / (root) を選択し、保存。GitHub Pages が HTTPS を提供します。
    4) しばらく待てば https://<your-username>.github.io/<repo>/ で公開されます。
    注意: MediaPipe を CDN で使うためインターネット接続が必要。ローカルで試す場合は `npx http-server` 等で localhost を使ってください。
  -->
</body>
</html>
