<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>簡易カメラアプリ — 作者：ゆいきち</title>
  <style>
    :root{--bg:#0f1724;--card:#0b1220;--accent:#06b6d4;--muted:#cbd5e1}
    *{box-sizing:border-box}
    body{font-family:system-ui,-apple-system,Segoe UI,Roboto,Helvetica,Arial; background:linear-gradient(180deg,#071028 0%,#071a2b 100%);color:var(--muted);min-height:100vh;display:flex;align-items:center;justify-content:center;padding:24px}
    .app{width:100%;max-width:980px;background:linear-gradient(180deg,rgba(255,255,255,0.02),rgba(255,255,255,0.01));border-radius:12px;padding:16px;box-shadow:0 8px 30px rgba(2,6,23,.6)}
    h1{font-size:18px;margin:0 0 12px;color:white}
    .grid{display:grid;grid-template-columns:1fr 360px;gap:12px}
    @media(max-width:880px){.grid{grid-template-columns:1fr}}
    .camera-card{background:rgba(255,255,255,0.02);padding:12px;border-radius:10px;display:flex;flex-direction:column;gap:8px}
    video{width:100%;height:auto;border-radius:8px;background:black;aspect-ratio:4/3;object-fit:cover}
    canvas{display:none}
    .controls{display:flex;flex-wrap:wrap;gap:8px}
    button, select, label{background:transparent;border:1px solid rgba(255,255,255,0.06);color:var(--muted);padding:8px 10px;border-radius:8px;font-size:14px}
    button.primary{background:var(--accent);color:#042028;border:0}
    .thumbs{display:grid;grid-template-columns:repeat(2,1fr);gap:8px}
    .thumbs img{width:100%;height:160px;object-fit:cover;border-radius:8px;border:1px solid rgba(255,255,255,0.04)}
    .footer{margin-top:10px;font-size:12px;color:#9fb6c3}
    .muted{opacity:.8}
    .warning{color:#ffb4a2}
    .hint{font-size:13px;color:#9fb6c3;margin-top:8px}
    .hidden{display:none}
  </style>
</head>
<body>
  <main class="app">
    <h1>簡易カメラアプリ（HTML単一ファイル） — 作者：ゆいきち</h1>
    <div class="grid">
      <section class="camera-card">
        <video id="preview" autoplay playsinline></video>
        <canvas id="snapshot"></canvas>

        <div class="controls">
          <button id="enableBtn" class="primary">カメラを有効にする</button>
          <select id="deviceSelect" title="カメラ選択" class="hidden"></select>
          <button id="switchBtn" class="hidden">前後切替</button>
          <button id="captureBtn" class="primary hidden">写真を撮る</button>
          <button id="downloadBtn" class="hidden">ダウンロード</button>
          <button id="mirrorBtn" class="hidden">鏡像 ON/OFF</button>
          <label class="muted hidden">フィルタ:
            <select id="filterSelect">
              <option value="none">なし</option>
              <option value="grayscale(100%)">モノクロ</option>
              <option value="sepia(70%)">セピア</option>
              <option value="contrast(120%)">コントラスト↑</option>
              <option value="blur(2px)">ぼかし</option>
            </select>
          </label>
        </div>

        <div class="footer">
          <div id="status">準備完了。カメラを使うには「カメラを有効にする」をクリックしてください。</div>
          <div class="warning" id="warning" style="display:none">注意: iPhone の Safari では HTTPS または localhost でないとカメラが使えません。</div>
          <div class="hint" id="hint">許可ダイアログが表示されたら「許可」してください。拒否した場合はブラウザの設定からカメラアクセスを許可する必要があります。</div>
        </div>
      </section>

      <aside class="camera-card">
        <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:8px">
          <strong>スナップショット</strong>
          <button id="clearThumbs">全部消す</button>
        </div>
        <div class="thumbs" id="thumbs"></div>
        <hr style="border:none;height:1px;background:rgba(255,255,255,0.03);margin:12px 0">
        <div style="font-size:13px;color:#9fb6c3">機能: 写真撮影・前後カメラ切替・フィルタ・画像ダウンロード。<br>iPhoneで使う場合はHTTPS（またはlocalhost）でホストしてください。</div>
      </aside>
    </div>
  </main>

  <script>
    const video = document.getElementById('preview');
    const canvas = document.getElementById('snapshot');
    const deviceSelect = document.getElementById('deviceSelect');
    const switchBtn = document.getElementById('switchBtn');
    const captureBtn = document.getElementById('captureBtn');
    const downloadBtn = document.getElementById('downloadBtn');
    const thumbs = document.getElementById('thumbs');
    const clearThumbs = document.getElementById('clearThumbs');
    const status = document.getElementById('status');
    const warning = document.getElementById('warning');
    const mirrorBtn = document.getElementById('mirrorBtn');
    const filterSelect = document.getElementById('filterSelect');
    const enableBtn = document.getElementById('enableBtn');

    let currentStream = null;
    let currentDeviceId = null;
    let isMirrored = false;

    // helper to show / hide controls after permission granted
    function showControls() {
      [deviceSelect, switchBtn, captureBtn, downloadBtn, mirrorBtn, filterSelect].forEach(el => el.classList.remove('hidden'));
      enableBtn.classList.add('hidden');
    }

    async function getDevices() {
      try {
        const devices = await navigator.mediaDevices.enumerateDevices();
        const videoDevices = devices.filter(d => d.kind === 'videoinput');
        deviceSelect.innerHTML = '';
        videoDevices.forEach((d,i) => {
          const opt = document.createElement('option');
          opt.value = d.deviceId;
          // labels are available only after permission in many browsers
          opt.text = d.label || `カメラ ${i+1}`;
          deviceSelect.appendChild(opt);
        });
        if (videoDevices.length === 0) {
          status.textContent = 'カメラが見つかりません。デバイス接続や権限を確認してください。';
        }
        return videoDevices;
      } catch (e) {
        console.error(e);
        status.textContent = 'カメラデバイスの取得に失敗しました。ブラウザの設定を確認してください。';
        throw e;
      }
    }

    async function start(deviceId=null, useFacing=null) {
      stopStream();
      try {
        status.textContent = 'カメラに接続中…';
        // prefer facingMode when deviceId not provided (better on mobiles)
        let constraints = {video: {width: {ideal:1280}, height: {ideal:960}}};
        if (deviceId) constraints.video.deviceId = { exact: deviceId };
        else if (useFacing) constraints.video.facingMode = { ideal: useFacing };

        currentStream = await navigator.mediaDevices.getUserMedia(constraints);
        video.srcObject = currentStream;

        // 現在のデバイスIDを保存
        const tracks = currentStream.getVideoTracks();
        if (tracks && tracks[0]) {
          // some browsers don't expose deviceId in settings; fallback to enumerateDevices
          const settings = tracks[0].getSettings();
          currentDeviceId = settings.deviceId || null;
        }

        status.textContent = 'カメラ接続中 — OK';
        warning.style.display = (isIos() && location.protocol !== 'https:' && location.hostname !== 'localhost') ? 'block' : 'none';

        await getDevices();
        if (currentDeviceId) deviceSelect.value = currentDeviceId;

        showControls();
      } catch (err) {
        console.error(err);
        handleGetUserMediaError(err);
      }
    }

    function stopStream(){
      if (currentStream) {
        currentStream.getTracks().forEach(t => t.stop());
        currentStream = null;
      }
    }

    function handleGetUserMediaError(err) {
      // Provide actionable messages for common errors
      if (!err) err = {};
      const name = err.name || '';
      if (name === 'NotAllowedError' || name === 'PermissionDeniedError') {
        status.textContent = 'カメラの使用が許可されていません。ブラウザのアドレスバーや設定からカメラアクセスを許可してください。';
        status.textContent += ' (エラー: ' + (err.message || name) + ')';
      } else if (name === 'NotFoundError' || name === 'OverconstrainedError') {
        status.textContent = '要求されたカメラが見つかりません。別のカメラを選択するか、接続を確認してください。';
      } else if (name === 'SecurityError') {
        status.textContent = 'セキュリティ上の理由でカメラを起動できません。HTTPSでホストされているか確認してください。';
      } else {
        status.textContent = 'カメラの起動に失敗しました: ' + (err.message || name);
      }
      // Show hint about iOS https requirement
      if (isIos()) warning.style.display = 'block';
    }

    captureBtn.addEventListener('click', () => {
      if (!currentStream) return alert('カメラが起動していません。まず「カメラを有効にする」を押してください。');
      const w = video.videoWidth || 1280;
      const h = video.videoHeight || 960;
      canvas.width = w;
      canvas.height = h;
      const ctx = canvas.getContext('2d');
      // mirror and filter
      ctx.save();
      if (isMirrored) { ctx.translate(w,0); ctx.scale(-1,1); }
      ctx.filter = getComputedStyle(video).filter || 'none';
      try {
        ctx.drawImage(video, 0, 0, w, h);
      } catch (e) {
        console.error('drawImage failed', e);
        alert('画像の取得に失敗しました。カメラストリームを確認してください。');
      }
      ctx.restore();
      const dataUrl = canvas.toDataURL('image/jpeg', 0.92);
      addThumb(dataUrl);
    });

    function addThumb(dataUrl){
      const img = document.createElement('img');
      img.src = dataUrl;
      const a = document.createElement('a');
      a.href = dataUrl;
      a.download = `snapshot_${Date.now()}.jpg`;
      a.appendChild(img);
      thumbs.prepend(a);
    }

    downloadBtn.addEventListener('click', () => {
      const first = thumbs.querySelector('a');
      if (first) first.click();
      else alert('ダウンロードできる画像がありません。まず写真を撮ってください。');
    });

    clearThumbs.addEventListener('click', () => thumbs.innerHTML = '');

    deviceSelect.addEventListener('change', async (e) => {
      const id = e.target.value;
      await start(id, null);
    });

    switchBtn.addEventListener('click', async () => {
      const devices = await navigator.mediaDevices.enumerateDevices();
      const cams = devices.filter(d => d.kind === 'videoinput');
      if (cams.length <= 1) return alert('切替可能なカメラが見つかりません');
      // try to find current index; fallback to facingMode toggle
      let idx = cams.findIndex(c => c.deviceId === (currentDeviceId || ''));
      if (idx === -1) {
        // toggle facingMode between environment and user
        const currentFacing = video.style.transform === 'scaleX(-1)' ? 'user' : 'environment';
        await start(null, currentFacing === 'user' ? 'environment' : 'user');
        return;
      }
      idx = (idx + 1) % cams.length;
      await start(cams[idx].deviceId, null);
    });

    mirrorBtn.addEventListener('click', () => {
      isMirrored = !isMirrored;
      video.style.transform = isMirrored ? 'scaleX(-1)' : 'none';
      mirrorBtn.textContent = isMirrored ? '鏡像: ON' : '鏡像: OFF';
    });

    filterSelect.addEventListener('change', (e) => {
      video.style.filter = e.target.value;
    });

    function isIos(){
      return /iPad|iPhone|iPod/.test(navigator.userAgent) && !window.MSStream;
    }

    // Do not auto-start camera to avoid NotAllowedError in some contexts.
    enableBtn.addEventListener('click', async () => {
      if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
        status.textContent = 'このブラウザはカメラ機能に対応していません。最新のSafari/Chromeをお試しください。';
        return;
      }
      try {
        // try to start with environment facing camera for mobiles
        await start(null, 'environment');
      } catch (e) {
        console.error(e);
      }
    });

    // Best-effort: if page already has permissions (e.g., user granted previously), reveal controls
    (async ()=>{
      try {
        if (navigator.permissions && navigator.permissions.query) {
          const p = await navigator.permissions.query({name: 'camera'}).catch(()=>null);
          if (p && p.state === 'granted') {
            // user already granted camera permission earlier
            showControls();
            // still don't auto-start; let user click Enable to ensure gesture
            status.textContent = 'カメラの権限は既に許可されています。"カメラを有効にする"を押して起動してください。';
          }
        }
      } catch (e) {
        // ignore permission API errors
      }
    })();

    // ページを離れるときにカメラ停止
    window.addEventListener('pagehide', stopStream);
    window.addEventListener('beforeunload', stopStream);
  </script>
</body>
</html>
