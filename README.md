<!DOCTYPE html>
<html lang="cs">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Tapeta — plný editor (pinch, presets, crop, export)</title>
  <style>
    :root{
      --bg:#071321; --panel:#0b1220; --muted:#9fb0c8; --accent:#4fc3f7;
      --radius:12px;
    }
    *{box-sizing:border-box}
    body{
      margin:0; font-family:Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
      background:linear-gradient(180deg,#071121,#0b1624); color:#e6f0f8;
      display:flex; flex-direction:column; align-items:center; gap:14px; padding:18px;
    }
    header{ text-align:center }
    h1{ margin:6px 0; font-size:20px }
    .layout{ width:100%; max-width:1100px; display:grid; grid-template-columns: 420px 1fr; gap:14px; align-items:start; }
    .panel{ background:var(--panel); border-radius:var(--radius); padding:12px; box-shadow:0 6px 24px rgba(0,0,0,0.6); }

    .controls{ display:flex; flex-direction:column; gap:10px; max-height:80vh; overflow:auto; padding-right:6px }
    .small{ font-size:13px; color:var(--muted) }
    .row{ display:flex; gap:8px; align-items:center; flex-wrap:wrap }
    select,input[type="color"],input[type="range"],button,input[type="file"]{
      background:#071825; color:#e6f0f8; border:none; padding:8px; border-radius:8px;
    }
    input[type="range"]{ width:100% }
    .previewWrap{ display:flex; flex-direction:column; gap:8px; align-items:center; justify-content:center }

    #previewBox{
      width:360px; height:780px; border-radius:20px; overflow:hidden; position:relative;
      background:#111; box-shadow:0 18px 50px rgba(0,0,0,0.7);
      touch-action: none;
    }
    #previewBg{ position:absolute; inset:0; background-size:cover; background-position:center; }
    #previewCar{
      position:absolute; left:50%; top:50%; transform-origin:center center; will-change:transform;
      pointer-events:auto; -webkit-user-drag:none; user-select:none;
      transition: transform 0.06s linear;
    }
    .meta{ font-size:12px; color:var(--muted) }
    .actions{ display:flex; gap:8px; align-items:center; justify-content:flex-end; flex-wrap:wrap }

    button.primary{ background:linear-gradient(90deg,var(--accent),#7ef0ff); color:#012; padding:10px 12px; font-weight:600; border-radius:10px }
    button.ghost{ background:transparent; border:1px solid rgba(255,255,255,0.06); padding:8px 10px; border-radius:8px; color:var(--muted) }

    @media(max-width:980px){
      .layout{ grid-template-columns: 1fr; }
      #previewBox{ width:320px; height:680px }
    }

    /* small helper */
    .muted { color:var(--muted); font-size:13px }
    .presetRow{ display:flex; gap:8px; align-items:center }
    .presetThumb{ width:48px; height:48px; object-fit:cover; border-radius:6px; border:1px solid rgba(255,255,255,0.06) }
    .toolbar { display:flex; gap:8px; align-items:center; flex-wrap:wrap; margin-top:8px }
  </style>
</head>
<body>
  <header>
    <h1>Tapeta — editor (pinch, presets, crop, export)</h1>
    <div class="meta">Vyber / nahraj obrázky, uprav je (drag/pinch/posuvníky), ulož preset a stáhni PNG.</div>
  </header>

  <div class="layout">
    <!-- Controls -->
    <div class="panel controls" aria-label="Ovládání">
      <div>
        <div class="row">
          <label class="small">Výchozí pozadí:</label>
          <select id="bgSelect">
            <option value="city">Město</option>
            <option value="beach">Pláž</option>
            <option value="mountains">Hory</option>
            <option value="eiffel">Eiffel</option>
          </select>
        </div>

        <div class="row" style="margin-top:6px">
          <label class="small">Výchozí auto:</label>
          <select id="carSelect">
            <option value="sport">Sportovní</option>
            <option value="nissan">Nissan-like</option>
            <option value="subaru">Subaru-like</option>
            <option value="truck">Náklaďák</option>
          </select>
        </div>

        <hr style="border:none; height:1px; background:rgba(255,255,255,0.03); margin:10px 0">

        <div>
          <div class="small">Nahrát vlastní pozadí</div>
          <input id="bgUpload" type="file" accept="image/*">
          <div class="small" style="margin-top:6px">nebo použij výchozí</div>
        </div>

        <div style="margin-top:8px">
          <div class="small">Nahrát vlastní auto (PNG s průhledem doporučeno)</div>
          <input id="carUpload" type="file" accept="image/*">
        </div>
      </div>

      <hr style="border:none; height:1px; background:rgba(255,255,255,0.03); margin:8px 0">

      <div>
        <div class="small" style="margin-bottom:6px">Nastavení auta</div>
        <label class="small">Posun X</label>
        <input id="posX" type="range" min="-100" max="100" value="0">

        <label class="small">Posun Y</label>
        <input id="posY" type="range" min="-100" max="100" value="30">

        <label class="small">Měřítko (scale)</label>
        <input id="scale" type="range" min="10" max="200" value="68">

        <label class="small">Rotace (stupně)</label>
        <input id="rotation" type="range" min="-180" max="180" value="0">

        <label class="small">Průhlednost</label>
        <input id="opacity" type="range" min="0" max="100" value="100">

        <div class="row" style="margin-top:6px; align-items:center;">
          <label class="small">Barva (tint):</label>
          <input id="tint" type="color" value="#ff3b3b">
          <label class="small" style="margin-left:6px"><input id="useTint" type="checkbox" checked> Použít tint</label>
        </div>

        <div class="row" style="margin-top:6px;">
          <label class="small"><input id="mirror" type="checkbox"> Zrcadlit</label>
          <button id="resetCar" class="ghost" style="margin-left:auto">Reset</button>
        </div>
      </div>

      <hr style="border:none; height:1px; background:rgba(255,255,255,0.03); margin:8px 0">

      <!-- Crop tool -->
      <div>
        <div class="small">Crop / mask pro auto</div>
        <div class="row" style="align-items:center;">
          <label class="small"><input id="useCrop" type="checkbox"> Zapnout ořez (crop)</label>
          <button id="resetCrop" class="ghost" style="margin-left:auto">Reset crop</button>
        </div>

        <div id="cropControls" style="display:none; margin-top:8px">
          <label class="small">Crop X (source %)</label>
          <input id="cropX" type="range" min="0" max="100" value="0">
          <label class="small">Crop Y (source %)</label>
          <input id="cropY" type="range" min="0" max="100" value="0">
          <label class="small">Crop width (% of source)</label>
          <input id="cropW" type="range" min="5" max="100" value="100">
          <label class="small">Crop height (% of source)</label>
          <input id="cropH" type="range" min="5" max="100" value="100">
        </div>
      </div>

      <hr style="border:none; height:1px; background:rgba(255,255,255,0.03); margin:8px 0">

      <div>
        <div class="small">Presets (ulož a načti nastavení)</div>
        <div class="presetRow" style="align-items:center">
          <select id="presetList" style="flex:1"><option value="">— vyber preset —</option></select>
          <img id="presetThumb" class="presetThumb" src="" alt="" style="display:none; margin-left:6px">
        </div>
        <div class="toolbar">
          <button id="savePreset" class="ghost">Uložit</button>
          <button id="loadPreset" class="ghost">Načíst</button>
          <button id="delPreset" class="ghost">Smazat</button>
          <button id="exportPresets" class="ghost">Export presetů (JSON)</button>
          <input id="importPresetsFile" type="file" accept="application/json" style="display:none">
          <button id="importPresetsBtn" class="ghost">Import presetů (JSON)</button>
        </div>
        <div class="small" style="margin-top:6px">Preset uloží nastavení + miniaturu. Data se ukládají do localStorage.</div>
      </div>

      <hr style="border:none; height:1px; background:rgba(255,255,255,0.03); margin:8px 0">

      <div>
        <div class="small">Export / rozlišení</div>
        <select id="resolution">
          <option value="1080x1920">1080 × 1920</option>
          <option value="1170x2532">1170 × 2532 (iPhone)</option>
          <option value="1242x2688">1242 × 2688</option>
          <option value="720x1280">720 × 1280</option>
        </select>

        <div class="row" style="margin-top:8px">
          <button id="download" class="primary">Vytvořit a stáhnout PNG</button>
          <button id="saveLocal" class="ghost">Stáhnout náhled</button>
        </div>

        <div class="small" style="margin-top:8px; color:#cfe9ff">Pokud export selže, nahraj obrázek přes upload (řeší CORS).</div>

        <div class="toolbar" style="margin-top:10px">
          <button id="undoBtn" class="ghost">Undo</button>
          <button id="redoBtn" class="ghost">Redo</button>
          <button id="resetAll" class="ghost">Reset vše</button>
        </div>
      </div>

      <div style="height:10px;"></div>
    </div>

    <!-- Preview -->
    <div class="panel previewWrap" aria-label="Náhled">
      <div class="meta" style="align-self:flex-start; margin-bottom:6px">Náhled — táhni / pinch / double-tap pro toggle zoom. Posuvníky pro precizní úpravy.</div>

      <div id="previewBox" title="Náhled tapety">
        <div id="previewBg"></div>
        <img id="previewCar" src="" alt="auto" />
      </div>

      <div style="width:100%; display:flex; justify-content:space-between; margin-top:8px">
        <div class="meta">Dotyk: táhni pro posun, pinch pro zoom, dvojité ťuknutí přepne zoom</div>
        <div class="meta">Autor: pvlmkl2708-create</div>
      </div>
    </div>
  </div>

  <canvas id="exportCanvas" style="display:none"></canvas>

  <script>
  /* ----------------- Zdrojové obrázky ----------------- */
  const backgrounds = {
    city: "https://cdn.pixabay.com/photo/2016/11/29/05/08/city-1868007_1280.jpg",
    beach: "https://cdn.pixabay.com/photo/2015/03/26/09/41/beach-690034_1280.jpg",
    mountains: "https://cdn.pixabay.com/photo/2016/11/29/09/32/mountains-1866531_1280.jpg",
    eiffel: "https://cdn.pixabay.com/photo/2016/03/27/22/16/paris-1284543_1280.jpg"
  };
  const cars = {
    sport: "https://cdn.pixabay.com/photo/2016/03/27/22/16/sports-car-1284544_1280.png",
    nissan: "https://cdn.pixabay.com/photo/2017/03/27/13/16/vintage-car-2176947_1280.png",
    subaru: "https://cdn.pixabay.com/photo/2013/07/13/12/49/car-160803_1280.png",
    truck: "https://cdn.pixabay.com/photo/2013/07/13/12/49/truck-160803_1280.png"
  };

  /* ----------------- DOM ----------------- */
  const bgSelect = document.getElementById('bgSelect');
  const carSelect = document.getElementById('carSelect');
  const bgUpload = document.getElementById('bgUpload');
  const carUpload = document.getElementById('carUpload');

  const previewBox = document.getElementById('previewBox');
  const previewBg = document.getElementById('previewBg');
  const previewCar = document.getElementById('previewCar');

  const posX = document.getElementById('posX');
  const posY = document.getElementById('posY');
  const scaleEl = document.getElementById('scale');
  const rotationEl = document.getElementById('rotation');
  const opacityEl = document.getElementById('opacity');
  const tintEl = document.getElementById('tint');
  const useTint = document.getElementById('useTint');
  const mirrorEl = document.getElementById('mirror');
  const resetCarBtn = document.getElementById('resetCar');

  const useCrop = document.getElementById('useCrop');
  const cropControls = document.getElementById('cropControls');
  const cropX = document.getElementById('cropX');
  const cropY = document.getElementById('cropY');
  const cropW = document.getElementById('cropW');
  const cropH = document.getElementById('cropH');
  const resetCropBtn = document.getElementById('resetCrop');

  const presetList = document.getElementById('presetList');
  const presetThumb = document.getElementById('presetThumb');
  const savePresetBtn = document.getElementById('savePreset');
  const loadPresetBtn = document.getElementById('loadPreset');
  const delPresetBtn = document.getElementById('delPreset');
  const exportPresetsBtn = document.getElementById('exportPresets');
  const importPresetsFile = document.getElementById('importPresetsFile');
  const importPresetsBtn = document.getElementById('importPresetsBtn');

  const downloadBtn = document.getElementById('download');
  const saveLocalBtn = document.getElementById('saveLocal');
  const resolutionSel = document.getElementById('resolution');
  const exportCanvas = document.getElementById('exportCanvas');

  const undoBtn = document.getElementById('undoBtn');
  const redoBtn = document.getElementById('redoBtn');
  const resetAllBtn = document.getElementById('resetAll');

  /* ----------------- Stav ----------------- */
  let state = {
    bgUrl: backgrounds[bgSelect.value],
    carUrl: cars[carSelect.value],
    posX: parseInt(posX.value),
    posY: parseInt(posY.value),
    scale: parseInt(scaleEl.value)/100,
    rotation: parseInt(rotationEl.value),
    opacity: parseInt(opacityEl.value)/100,
    tint: tintEl.value,
    useTint: useTint.checked,
    mirror: mirrorEl.checked,
    useCrop: false,
    cropX: parseInt(cropX?.value || 0),
    cropY: parseInt(cropY?.value || 0),
    cropW: parseInt(cropW?.value || 100),
    cropH: parseInt(cropH?.value || 100)
  };

  /* ---------- History (undo/redo) ---------- */
  const history = [];
  let historyIndex = -1;
  function pushHistory() {
    // store shallow copy of state (including bg/car urls)
    const snapshot = JSON.stringify(state);
    // if we undid and then change, drop forward states
    history.splice(historyIndex + 1);
    history.push(snapshot);
    historyIndex = history.length - 1;
    updateUndoRedoButtons();
  }
  function updateUndoRedoButtons() {
    undoBtn.disabled = historyIndex <= 0;
    redoBtn.disabled = historyIndex >= history.length - 1;
  }
  function undo() {
    if(historyIndex <= 0) return;
    historyIndex--;
    Object.assign(state, JSON.parse(history[historyIndex]));
    applyStateToUI();
  }
  function redo() {
    if(historyIndex >= history.length - 1) return;
    historyIndex++;
    Object.assign(state, JSON.parse(history[historyIndex]));
    applyStateToUI();
  }

  /* ---------- Helpers ---------- */
  function setStateFromInputs(){
    state.posX = parseInt(posX.value);
    state.posY = parseInt(posY.value);
    state.scale = parseInt(scaleEl.value)/100;
    state.rotation = parseInt(rotationEl.value);
    state.opacity = parseInt(opacityEl.value)/100;
    state.tint = tintEl.value;
    state.useTint = useTint.checked;
    state.mirror = mirrorEl.checked;
    state.useCrop = useCrop.checked;
    state.cropX = parseInt(cropX.value || 0);
    state.cropY = parseInt(cropY.value || 0);
    state.cropW = parseInt(cropW.value || 100);
    state.cropH = parseInt(cropH.value || 100);
  }

  function applyStateToUI(){
    // update inputs from state and preview
    try{
      posX.value = state.posX; posY.value = state.posY; scaleEl.value = Math.round(state.scale * 100);
      rotationEl.value = state.rotation; opacityEl.value = Math.round(state.opacity * 100);
      tintEl.value = state.tint; useTint.checked = !!state.useTint; mirrorEl.checked = !!state.mirror;
      useCrop.checked = !!state.useCrop; cropX.value = state.cropX; cropY.value = state.cropY; cropW.value = state.cropW; cropH.value = state.cropH;
      // background and car
      // try find keys for selects
      const bgKey = Object.keys(backgrounds).find(k => backgrounds[k] === state.bgUrl);
      if(bgKey) bgSelect.value = bgKey; else bgSelect.value = 'city';
      const carKey = Object.keys(cars).find(k => cars[k] === state.carUrl);
      if(carKey) carSelect.value = carKey; else carSelect.value = 'sport';
    } catch(e){ console.warn('applyStateToUI err', e) }
    cropControls.style.display = state.useCrop ? 'block' : 'none';
    updatePreview();
    pushHistory(); // record this state (so undo works after apply)
  }

  function updatePreview(){
    previewBg.style.backgroundImage = `url("${state.bgUrl}")`;
    previewCar.src = state.carUrl;
    const px = state.posX;
    const py = state.posY;
    const s = state.scale;
    const r = state.rotation;
    const mirror = state.mirror ? -1 : 1;
    previewCar.style.opacity = state.opacity;
    previewCar.style.transform = `translate(-50%,-50%) translate(${px}%, ${py}%) scale(${s * mirror}) rotate(${r}deg)`;
    if(state.useCrop){
      const top = state.cropY;
      const left = state.cropX;
      const bottom = 100 - (state.cropY + state.cropH);
      const right = 100 - (state.cropX + state.cropW);
      previewCar.style.clipPath = `inset(${top}% ${right}% ${bottom}% ${left}%)`;
    } else {
      previewCar.style.clipPath = 'none';
    }
  }

  /* read file as dataURL */
  function dataURLFromFile(file){
    return new Promise((res, rej)=>{
      const reader = new FileReader();
      reader.onload = ()=> res(reader.result);
      reader.onerror = rej;
      reader.readAsDataURL(file);
    });
  }

  /* load image (supports data: URIs and remote with crossOrigin) */
  function loadImage(urlOrData){
    return new Promise((res, rej)=>{
      const img = new Image();
      if(!urlOrData.startsWith('data:')) img.crossOrigin = "anonymous";
      img.onload = ()=> res(img);
      img.onerror = ()=> rej(new Error('Nelze načíst obrázek: ' + urlOrData));
      img.src = urlOrData;
    });
  }

  /* drawImage cover (background) */
  function drawImageCover(ctx, img, cw, ch){
    const cr = img.width / img.height;
    const wr = cw / ch;
    let dw, dh, dx, dy;
    if(cr > wr){
      dh = ch;
      dw = img.width * (ch / img.height);
      dx = (cw - dw) / 2;
      dy = 0;
    } else {
      dw = cw;
      dh = img.height * (cw / img.width);
      dx = 0;
      dy = (ch - dh) / 2;
    }
    ctx.drawImage(img, dx, dy, dw, dh);
  }

  /* ---------- Events: selects & uploads ---------- */
  bgSelect.addEventListener('change', ()=>{ state.bgUrl = backgrounds[bgSelect.value]; updatePreview(); pushHistory(); });
  carSelect.addEventListener('change', ()=>{ state.carUrl = cars[carSelect.value]; updatePreview(); pushHistory(); });

  bgUpload.addEventListener('change', async (e)=>{
    const f = e.target.files && e.target.files[0]; if(!f) return;
    try{ state.bgUrl = await dataURLFromFile(f); updatePreview(); pushHistory(); }catch(err){ alert('Chyba pozadí: '+err.message) }
  });
  carUpload.addEventListener('change', async (e)=>{
    const f = e.target.files && e.target.files[0]; if(!f) return;
    try{ state.carUrl = await dataURLFromFile(f); updatePreview(); pushHistory(); }catch(err){ alert('Chyba auta: '+err.message) }
  });

  /* ---------- Bind sliders to state ---------- */
  [posX,posY,scaleEl,rotationEl,opacityEl,tintEl,useTint,mirrorEl,useCrop,cropX,cropY,cropW,cropH].forEach(el=>{
    if(!el) return;
    el.addEventListener('input', ()=>{
      setStateFromInputs();
      cropControls.style.display = state.useCrop ? 'block' : 'none';
      updatePreview();
    });
    el.addEventListener('change', ()=>{
      setStateFromInputs();
      cropControls.style.display = state.useCrop ? 'block' : 'none';
      updatePreview();
      pushHistory(); // commit major change
    });
  });

  /* Reset buttons */
  resetCarBtn.addEventListener('click', ()=>{
    posX.value = 0; posY.value = 30; scaleEl.value = 68; rotationEl.value = 0; opacityEl.value = 100;
    tintEl.value = '#ff3b3b'; useTint.checked = true; mirrorEl.checked = false;
    setStateFromInputs(); updatePreview(); pushHistory();
  });
  resetCropBtn.addEventListener('click', ()=>{
    cropX.value = 0; cropY.value = 0; cropW.value = 100; cropH.value = 100; useCrop.checked = false;
    setStateFromInputs(); cropControls.style.display = state.useCrop ? 'block' : 'none'; updatePreview(); pushHistory();
  });

  /* ---------- Drag & Pinch (with wheel) & double-tap toggle zoom ---------- */
  let dragging=false, dragStart=null, startPos={x:0,y:0};
  let pinchActive=false, pinchStartDist=0, pinchStartScale=1, pinchCenterStart=null;
  function rect(){ return previewBox.getBoundingClientRect(); }
  function clientToPercent(dx, dy){
    const r = rect();
    return {px: (dx / r.width) * 100, py: (dy / r.height) * 100};
  }
  function touchDist(t1,t2){ const dx=t2.clientX-t1.clientX, dy=t2.clientY-t1.clientY; return Math.hypot(dx,dy); }
  function touchCenter(t1,t2){ return {x:(t1.clientX+t2.clientX)/2, y:(t1.clientY+t2.clientY)/2}; }

  // double-tap toggle zoom (between default 0.68 and zoomed 1.35)
  let lastTap = 0;
  const DOUBLE_TAP_THRESHOLD = 320;
  let zoomToggled = false;
  const ZOOM_TOGGLE_VALUE = 1.35;
  const DEFAULT_SCALE = 0.68;

  function detectDoubleTap(e){
    const t = Date.now();
    if(t - lastTap < DOUBLE_TAP_THRESHOLD){
      // toggle zoom
      zoomToggled = !zoomToggled;
      scaleEl.value = Math.round((zoomToggled ? ZOOM_TOGGLE_VALUE : DEFAULT_SCALE) * 100);
      setStateFromInputs(); updatePreview(); pushHistory();
    }
    lastTap = t;
  }

  function onMouseDown(e){
    e.preventDefault();
    dragging=true;
    dragStart={x:e.clientX,y:e.clientY};
    startPos={x:state.posX,y:state.posY};
  }
  function onMouseMove(e){
    if(!dragging) return;
    const cur = {x:e.clientX,y:e.clientY};
    const dx = cur.x - dragStart.x, dy = cur.y - dragStart.y;
    const p = clientToPercent(dx, dy);
    posX.value = Math.round(startPos.x + p.px);
    posY.value = Math.round(startPos.y + p.py);
    setStateFromInputs(); updatePreview();
  }
  function onMouseUp(){ dragging=false; pushHistory(); }

  function onTouchStart(e){
    if(e.touches.length===1){
      dragging=true;
      dragStart={x:e.touches[0].clientX,y:e.touches[0].clientY};
      startPos={x:state.posX,y:state.posY};
    } else if(e.touches.length>=2){
      pinchActive=true;
      pinchStartDist = touchDist(e.touches[0], e.touches[1]);
      pinchStartScale = state.scale;
      pinchCenterStart = touchCenter(e.touches[0], e.touches[1]);
    }
    detectDoubleTap(e);
  }
  function onTouchMove(e){
    if(pinchActive && e.touches.length>=2){
      const curDist = touchDist(e.touches[0], e.touches[1]);
      const factor = curDist / pinchStartDist;
      let newScale = pinchStartScale * factor;
      newScale = Math.max(0.1, Math.min(2.0, newScale));
      scaleEl.value = Math.round(newScale*100);
      setStateFromInputs(); updatePreview();

      // adjust pos to keep center stable
      const curCenter = touchCenter(e.touches[0], e.touches[1]);
      const dx = curCenter.x - pinchCenterStart.x;
      const dy = curCenter.y - pinchCenterStart.y;
      const perc = clientToPercent(dx,dy);
      posX.value = Math.round(state.posX + perc.px);
      posY.value = Math.round(state.posY + perc.py);
      setStateFromInputs(); updatePreview();
    } else if(dragging && e.touches.length===1){
      const cur = {x:e.touches[0].clientX,y:e.touches[0].clientY};
      const dx = cur.x - dragStart.x, dy = cur.y - dragStart.y;
      const p = clientToPercent(dx,dy);
      posX.value = Math.round(startPos.x + p.px);
      posY.value = Math.round(startPos.y + p.py);
      setStateFromInputs(); updatePreview();
    }
  }
  function onTouchEnd(e){
    if(e.touches.length < 2) pinchActive=false;
    if(e.touches.length === 0) dragging=false;
    scaleEl.value = Math.round(state.scale*100);
    pushHistory();
  }

  function onWheel(e){
    e.preventDefault();
    const delta = -e.deltaY || e.wheelDelta;
    const factor = delta > 0 ? 1.05 : 0.95;
    let newScale = state.scale * factor;
    newScale = Math.max(0.1, Math.min(2.0, newScale));
    scaleEl.value = Math.round(newScale*100);
    setStateFromInputs(); updatePreview();
    pushHistory();
  }

  // attach pointer handlers
  previewBox.addEventListener('mousedown', onMouseDown);
  window.addEventListener('mousemove', onMouseMove);
  window.addEventListener('mouseup', onMouseUp);

  previewBox.addEventListener('touchstart', onTouchStart, {passive:false});
  previewBox.addEventListener('touchmove', onTouchMove, {passive:false});
  previewBox.addEventListener('touchend', onTouchEnd);

  previewBox.addEventListener('wheel', onWheel, {passive:false});

  /* ---------- Presets (localStorage) with thumbnails ---------- */
  const PRESET_KEY = 'tapeta_presets_v2';

  function loadPresetList(){
    presetList.innerHTML = '<option value="">— vyber preset —</option>';
    const raw = localStorage.getItem(PRESET_KEY);
    if(!raw) return;
    try{
      const obj = JSON.parse(raw);
      Object.keys(obj).forEach(name=>{
        const opt = document.createElement('option'); opt.value = name; opt.textContent = name;
        presetList.appendChild(opt);
      });
    }catch(e){ console.warn('preset parse error', e) }
  }

  // create thumbnail (small preview) by rendering to canvas (uses current state)
  async function createThumbnailDataURL(w = 160, h = 120){
    try{
      const tmpCanvas = document.createElement('canvas');
      tmpCanvas.width = w; tmpCanvas.height = h;
      const ctx = tmpCanvas.getContext('2d');

      const bgImg = await loadImage(state.bgUrl);
      const carImg = await loadImage(state.carUrl);
      drawImageCover(ctx, bgImg, w, h);

      const carW = Math.round(w * state.scale);
      const carH = Math.round(carW * (carImg.height / carImg.width));
      const centerX = Math.round(w / 2 + (state.posX/100) * w);
      const centerY = Math.round(h / 2 + (state.posY/100) * h);

      // handle crop on source
      let sx=0, sy=0, sw=carImg.width, sh=carImg.height;
      if(state.useCrop){
        sx = Math.round((state.cropX/100)*carImg.width);
        sy = Math.round((state.cropY/100)*carImg.height);
        sw = Math.round((state.cropW/100)*carImg.width);
        sh = Math.round((state.cropH/100)*carImg.height);
      }

      if(state.useTint){
        const tmp = document.createElement('canvas'); tmp.width = carW; tmp.height = carH;
        const tctx = tmp.getContext('2d');
        tctx.drawImage(carImg, sx, sy, sw, sh, 0, 0, tmp.width, tmp.height);
        tctx.globalCompositeOperation = 'source-in';
        tctx.fillStyle = state.tint; tctx.fillRect(0,0,tmp.width,tmp.height);
        ctx.save();
        ctx.translate(centerX, centerY); ctx.rotate(state.rotation*Math.PI/180); ctx.scale(state.mirror?-1:1,1);
        ctx.globalAlpha = state.opacity;
        ctx.drawImage(tmp, -carW/2, -carH/2, carW, carH);
        ctx.globalAlpha = Math.min(1, 0.6*state.opacity+0.4);
        ctx.drawImage(carImg, sx, sy, sw, sh, -carW/2, -carH/2, carW, carH);
        ctx.restore(); ctx.globalAlpha = 1;
      } else {
        ctx.save();
        ctx.translate(centerX, centerY); ctx.rotate(state.rotation*Math.PI/180); ctx.scale(state.mirror?-1:1,1);
        ctx.globalAlpha = state.opacity;
        ctx.drawImage(carImg, sx, sy, sw, sh, -carW/2, -carH/2, carW, carH);
        ctx.restore(); ctx.globalAlpha = 1;
      }

      return tmpCanvas.toDataURL('image/png');
    } catch(e){ console.warn('thumb err', e); return ''; }
  }

  savePresetBtn.addEventListener('click', async ()=>{
    const name = prompt('Jméno presetu (krátké):');
    if(!name) return;
    const raw = localStorage.getItem(PRESET_KEY);
    const obj = raw ? JSON.parse(raw) : {};
    // create thumbnail snapshot
    const thumb = await createThumbnailDataURL(160,120).catch(()=> '');
    obj[name] = {
      bgUrl: state.bgUrl,
      carUrl: state.carUrl,
      posX: state.posX, posY: state.posY, scale: state.scale,
      rotation: state.rotation, opacity: state.opacity,
      tint: state.tint, useTint: state.useTint, mirror: state.mirror,
      useCrop: state.useCrop, cropX: state.cropX, cropY: state.cropY, cropW: state.cropW, cropH: state.cropH,
      thumb
    };
    localStorage.setItem(PRESET_KEY, JSON.stringify(obj));
    loadPresetList();
    alert('Preset uložen: ' + name);
  });

  loadPresetBtn.addEventListener('click', ()=>{
    const name = presetList.value;
    if(!name) { alert('Vyber preset v seznamu.'); return; }
    const raw = localStorage.getItem(PRESET_KEY);
    if(!raw) return;
    const obj = JSON.parse(raw);
    const p = obj[name];
    if(!p) { alert('Preset nenalezen'); return; }
    // restore
    Object.assign(state, {
      bgUrl: p.bgUrl, carUrl: p.carUrl,
      posX: p.posX, posY: p.posY, scale: p.scale,
      rotation: p.rotation, opacity: p.opacity,
      tint: p.tint, useTint: p.useTint, mirror: p.mirror,
      useCrop: p.useCrop, cropX: p.cropX, cropY: p.cropY, cropW: p.cropW, cropH: p.cropH
    });
    applyStateToUI();
    // show thumb
    if(p.thumb){ presetThumb.src = p.thumb; presetThumb.style.display = 'block'; } else { presetThumb.style.display = 'none'; }
    alert('Preset načten: ' + name);
  });

  delPresetBtn.addEventListener('click', ()=>{
    const name = presetList.value;
    if(!name){ alert('Vyber preset k smazání'); return; }
    if(!confirm('Smazat preset "'+name+'"?')) return;
    const raw = localStorage.getItem(PRESET_KEY); if(!raw) return;
    const obj = JSON.parse(raw); delete obj[name]; localStorage.setItem(PRESET_KEY, JSON.stringify(obj));
    loadPresetList(); presetThumb.style.display = 'none'; alert('Preset smazán: ' + name);
  });

  // export presets (download JSON)
  exportPresetsBtn.addEventListener('click', ()=>{
    const raw = localStorage.getItem(PRESET_KEY) || '{}';
    const blob = new Blob([raw], {type: 'application/json'});
    const link = document.createElement('a'); link.href = URL.createObjectURL(blob);
    link.download = 'tapeta-presets.json'; document.body.appendChild(link); link.click(); link.remove();
  });

  importPresetsBtn.addEventListener('click', ()=> importPresetsFile.click());
  importPresetsFile.addEventListener('change', async (e)=>{
    const f = e.target.files && e.target.files[0]; if(!f) return;
    try{
      const text = await f.text();
      const parsed = JSON.parse(text);
      const raw = localStorage.getItem(PRESET_KEY);
      const obj = raw ? JSON.parse(raw) : {};
      Object.assign(obj, parsed);
      localStorage.setItem(PRESET_KEY, JSON.stringify(obj));
      loadPresetList();
      alert('Presety importovány.');
    } catch(err){ alert('Import selhal: ' + err.message) }
  });

  presetList.addEventListener('change', ()=>{
    const raw = localStorage.getItem(PRESET_KEY); if(!raw) { presetThumb.style.display = 'none'; return; }
    const obj = JSON.parse(raw);
    const p = obj[presetList.value];
    if(p && p.thumb){ presetThumb.src = p.thumb; presetThumb.style.display = 'block'; } else { presetThumb.style.display = 'none'; }
  });

  /* ---------- Export (canvas) ---------- */
  async function generateAndDownload(full=true){
    downloadBtn.disabled = true; downloadBtn.textContent = 'Vytvářím…';
    try{
      let W,H;
      if(full){ const [w,h] = resolutionSel.value.split('x').map(n=>parseInt(n,10)); W=w; H=h; }
      else { const r = previewBox.getBoundingClientRect(); W=Math.round(r.width); H=Math.round(r.height); }

      exportCanvas.width = W; exportCanvas.height = H;
      const ctx = exportCanvas.getContext('2d');

      const bgImg = await loadImage(state.bgUrl);
      const carImg = await loadImage(state.carUrl);

      drawImageCover(ctx, bgImg, W, H);

      const carTargetWidth = Math.round(W * state.scale);
      const aspect = (carImg.width / carImg.height) || 1;
      const carTargetHeight = Math.round(carTargetWidth / aspect);

      const centerX = Math.round(W / 2);
      const centerY = Math.round(H / 2);
      const px = Math.round(centerX + (state.posX/100) * W);
      const py = Math.round(centerY + (state.posY/100) * H);

      let sx = 0, sy = 0, sw = carImg.width, sh = carImg.height;
      if(state.useCrop){
        sx = Math.round((state.cropX/100) * carImg.width);
        sy = Math.round((state.cropY/100) * carImg.height);
        sw = Math.round((state.cropW/100) * carImg.width);
        sh = Math.round((state.cropH/100) * carImg.height);
        if(sw < 1) sw = 1; if(sh < 1) sh = 1;
      }

      if(state.useTint){
        const tmp = document.createElement('canvas');
        tmp.width = carTargetWidth; tmp.height = carTargetHeight;
        const tctx = tmp.getContext('2d');
        tctx.drawImage(carImg, sx, sy, sw, sh, 0, 0, tmp.width, tmp.height);
        tctx.globalCompositeOperation = 'source-in';
        tctx.fillStyle = state.tint;
        tctx.fillRect(0,0,tmp.width,tmp.height);

        ctx.save();
        ctx.translate(px, py);
        ctx.rotate((state.rotation * Math.PI)/180);
        ctx.scale(state.mirror ? -1 : 1, 1);
        ctx.globalAlpha = state.opacity;
        ctx.drawImage(tmp, -carTargetWidth/2, -carTargetHeight/2, carTargetWidth, carTargetHeight);
        ctx.globalAlpha = Math.min(1, 0.6 * state.opacity + 0.4);
        ctx.drawImage(carImg, sx, sy, sw, sh, -carTargetWidth/2, -carTargetHeight/2, carTargetWidth, carTargetHeight);
        ctx.restore();
        ctx.globalAlpha = 1.0;
      } else {
        ctx.save();
        ctx.translate(px, py);
        ctx.rotate((state.rotation * Math.PI)/180);
        ctx.scale(state.mirror ? -1 : 1, 1);
        ctx.globalAlpha = state.opacity;
        ctx.drawImage(carImg, sx, sy, sw, sh, -carTargetWidth/2, -carTargetHeight/2, carTargetWidth, carTargetHeight);
        ctx.restore();
        ctx.globalAlpha = 1.0;
      }

      const dataURL = exportCanvas.toDataURL('image/png');
      const link = document.createElement('a');
      link.href = dataURL;
      link.download = `tapeta_${W}x${H}.png`;
      document.body.appendChild(link);
      link.click();
      link.remove();

      downloadBtn.textContent = 'Hotovo — staženo!';
      setTimeout(()=> downloadBtn.textContent = 'Vytvořit a stáhnout PNG', 1400);
    } catch(err){
      console.error(err);
      alert('Chyba při exportu: ' + err.message + '\nPoužij upload vlastních obrázků pokud externí URL blokují export (CORS).');
      downloadBtn.textContent = 'Vytvořit a stáhnout PNG';
    } finally { downloadBtn.disabled = false; }
  }

  downloadBtn.addEventListener('click', ()=> generateAndDownload(true));
  saveLocalBtn.addEventListener('click', ()=> generateAndDownload(false));

  /* ---------- Undo/Redo and Reset ---------- */
  undoBtn.addEventListener('click', ()=> { undo(); applyStateToUI(); });
  redoBtn.addEventListener('click', ()=> { redo(); applyStateToUI(); });
  resetAllBtn.addEventListener('click', ()=>{
    if(!confirm('Resetovat všechno nastavení na výchozí?')) return;
    state = {
      bgUrl: backgrounds['city'],
      carUrl: cars['sport'],
      posX: 0, posY: 30, scale: DEFAULT_SCALE, rotation: 0, opacity: 1,
      tint: '#ff3b3b', useTint: true, mirror: false,
      useCrop: false, cropX:0, cropY:0, cropW:100, cropH:100
    };
    applyStateToUI();
  });

  /* ---------- Init ---------- */
  function init(){
    state.bgUrl = backgrounds[bgSelect.value];
    state.carUrl = cars[carSelect.value];
    setStateFromInputs();
    cropControls.style.display = state.useCrop ? 'block' : 'none';
    loadPresetList();
    updatePreview();
    pushHistory(); // initial state
  }
  init();

  // small sync: update slider-driven state every 200ms to ensure state reflects pinch/wheel changes
  setInterval(()=> {
    const s = parseInt(scaleEl.value)/100;
    if(Math.abs(s - state.scale) > 0.001){
      setStateFromInputs(); updatePreview();
    }
  }, 200);

  </script>
</body>
</html>
