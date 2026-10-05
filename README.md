<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>ZORDAI 3D 互动卡片展示</title>
  <script src="https://unpkg.com/three@0.160.0/build/three.min.js"></script>
  <script src="https://unpkg.com/three@0.160.0/examples/js/controls/OrbitControls.js"></script>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      background: #0b0f19;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      color: #f8fafc;
      min-height: 100vh;
      display: flex;
      flex-direction: column;
      align-items: center;
      padding: 16px;
      overflow-x: hidden;
    }
    .header { text-align: center; margin-bottom: 12px; }
    h1 { font-size: 20px; font-weight: 700; color: #f8fafc; }
    p.subtitle { font-size: 12px; color: #94a3b8; margin-top: 2px; }

    .viewport-box {
      width: 100%;
      max-width: 420px;
      height: 380px;
      position: relative;
      border-radius: 16px;
      overflow: hidden;
      background: linear-gradient(135deg, #1e293b 0%, #0f172a 100%);
      box-shadow: 0 15px 35px rgba(0,0,0,0.5);
      border: 1px solid rgba(255,255,255,0.08);
      display: flex;
      justify-content: center;
      align-items: center;
    }
    #threeCanvas { width: 100%; height: 100%; display: block; cursor: grab; }
    #threeCanvas:active { cursor: grabbing; }

    .hud {
      position: absolute;
      top: 10px; right: 10px;
      background: rgba(15, 23, 42, 0.85);
      backdrop-filter: blur(8px);
      border: 1px solid rgba(255, 255, 255, 0.12);
      padding: 4px 10px;
      border-radius: 999px;
      color: #38bdf8;
      font-family: monospace;
      font-size: 11px;
      display: flex; align-items: center; gap: 6px;
      z-index: 10;
    }
    .hud-dot { width: 6px; height: 6px; border-radius: 50%; background: #38bdf8; box-shadow: 0 0 6px #38bdf8; }

    .control-panel {
      width: 100%;
      max-width: 420px;
      background: rgba(30, 41, 59, 0.7);
      backdrop-filter: blur(12px);
      border: 1px solid rgba(255, 255, 255, 0.08);
      border-radius: 16px;
      padding: 16px;
      margin-top: 14px;
      display: flex;
      flex-direction: column;
      gap: 12px;
    }

    .slider-group { display: flex; align-items: center; gap: 12px; }
    .slider-label { font-size: 13px; color: #94a3b8; white-space: nowrap; }
    input[type="range"] { flex: 1; accent-color: #38bdf8; cursor: pointer; }
    .val-badge { font-family: monospace; font-size: 13px; color: #f8fafc; min-width: 36px; text-align: right; }

    .presets { display: flex; gap: 8px; justify-content: space-between; }
    .btn {
      flex: 1;
      background: rgba(255, 255, 255, 0.06);
      border: 1px solid rgba(255, 255, 255, 0.12);
      color: #cbd5e1;
      padding: 8px 0;
      border-radius: 8px;
      font-size: 12px;
      font-weight: 500;
      cursor: pointer;
      text-align: center;
      transition: all 0.2s;
    }
    .btn.active { background: #38bdf8; color: #0f172a; border-color: #38bdf8; font-weight: 600; }

    .upload-section { border-top: 1px solid rgba(255, 255, 255, 0.08); padding-top: 12px; }
    .upload-title { font-size: 13px; color: #cbd5e1; margin-bottom: 8px; font-weight: 600; }
    .upload-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 8px; }
    .upload-item {
      position: relative;
      height: 75px;
      background: rgba(15, 23, 42, 0.6);
      border: 1px dashed rgba(255, 255, 255, 0.3);
      border-radius: 8px;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      overflow: hidden;
      cursor: pointer;
    }
    .upload-item img {
      position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover; z-index: 1;
    }
    .upload-item span {
      font-size: 10px; color: #cbd5e1; z-index: 2;
      background: rgba(15, 23, 42, 0.8); padding: 2px 6px; border-radius: 4px;
    }
    .upload-item input {
      position: absolute; inset: 0; opacity: 0; cursor: pointer; z-index: 3;
    }

    .share-tips { font-size: 11px; color: #64748b; text-align: center; margin-top: 8px; }
  </style>
</head>
<body>

  <div class="header">
    <h1>ZORDAI 3D 互动卡片展示</h1>
    <p class="subtitle">点击下方格子上传你的 4 张图片，实时投射到 3D 折面上</p>
  </div>

  <div class="viewport-box">
    <div class="hud">
      <span class="hud-dot"></span>
      <span id="hudAngle">夹角: 90°</span>
    </div>
    <canvas id="threeCanvas"></canvas>
  </div>

  <div class="control-panel">
    <div class="presets">
      <button class="btn" onclick="setAngle(0)">平铺 0°</button>
      <button class="btn active" onclick="setAngle(90)" id="btnHalf">半开 90°</button>
      <button class="btn" onclick="setAngle(170)">合拢 170°</button>
    </div>

    <div class="slider-group">
      <span class="slider-label">折叠角度</span>
      <input type="range" id="angleSlider" min="0" max="175" value="90" step="1" oninput="onSlider(this.value)">
      <span id="sliderVal" class="val-badge">90°</span>
    </div>

    <div class="upload-section">
      <div class="upload-title">📷 点击格子上传图片 (1~4面)</div>
      <div class="upload-grid">
        <div class="upload-item" id="box1">
          <span>封面</span>
          <input type="file" accept="image/*" onchange="handleImageUpload(event, 0)">
        </div>
        <div class="upload-item" id="box2">
          <span>内页左</span>
          <input type="file" accept="image/*" onchange="handleImageUpload(event, 1)">
        </div>
        <div class="upload-item" id="box3">
          <span>内页右</span>
          <input type="file" accept="image/*" onchange="handleImageUpload(event, 2)">
        </div>
        <div class="upload-item" id="box4">
          <span>封底</span>
          <input type="file" accept="image/*" onchange="handleImageUpload(event, 3)">
        </div>
      </div>
    </div>
  </div>

  <div class="share-tips">💡 提示：更新代码后请刷新页面，点击对应格子即可加载图片！</div>

  <script>
    let scene, camera, renderer, controls;
    let leftWingPivot, rightWingPivot;
    let panelMaterials = [];

    // 创建高质量的默认指引画布纹理（绝不黑屏）
    function createDefaultTexture(title, desc) {
      const cv = document.createElement('canvas');
      cv.width = 512; cv.height = 724;
      const ctx = cv.getContext('2d');
      ctx.fillStyle = '#1e293b';
      ctx.fillRect(0, 0, 512, 724);

      ctx.strokeStyle = '#38bdf8';
      ctx.lineWidth = 10;
      ctx.strokeRect(20, 20, 472, 684);

      ctx.fillStyle = '#ffffff';
      ctx.font = 'bold 38px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText(title, 256, 300);

      ctx.fillStyle = '#94a3b8';
      ctx.font = '22px sans-serif';
      ctx.fillText(desc, 256, 360);
      ctx.fillText('点击下方按钮上传', 256, 420);

      const tex = new THREE.CanvasTexture(cv);
      tex.colorSpace = THREE.SRGBColorSpace;
      return tex;
    }

    function init3D() {
      const container = document.querySelector('.viewport-box');
      const canvas = document.getElementById('threeCanvas');

      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(40, container.clientWidth / container.clientHeight, 0.1, 100);
      camera.position.set(0, 0, 5.2);

      renderer = new THREE.WebGLRenderer({ canvas: canvas, antialias: true, alpha: true });
      renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
      renderer.setSize(container.clientWidth, container.clientHeight);
      renderer.outputColorSpace = THREE.SRGBColorSpace;

      controls = new THREE.OrbitControls(camera, renderer.domElement);
      controls.enableDamping = true;
      controls.dampingFactor = 0.05;
      controls.maxPolarAngle = Math.PI / 2 + 0.2;
      controls.minDistance = 2.5;
      controls.maxDistance = 10;

      scene.add(new THREE.AmbientLight(0xffffff, 1.0));
      const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
      dirLight.position.set(3, 5, 4);
      scene.add(dirLight);

      const cardW = 1.6, cardH = 2.2, cardT = 0.01;
      const geom = new THREE.BoxGeometry(cardW, cardH, cardT);

      const t1 = createDefaultTexture('封面 (第1面)', '正面卡面');
      const t2 = createDefaultTexture('内页左 (第2面)', '左侧展开');
      const t3 = createDefaultTexture('内页右 (第3面)', '右侧展开');
      const t4 = createDefaultTexture('封底 (第4面)', '背面封底');

      const leftMats = [
        new THREE.MeshStandardMaterial({color: 0x1e293b}),
        new THREE.MeshStandardMaterial({color: 0x1e293b}),
        new THREE.MeshStandardMaterial({color: 0x1e293b}),
        new THREE.MeshStandardMaterial({color: 0x1e293b}),
        panelMaterials[3] = new THREE.MeshStandardMaterial({map: t4, roughness: 0.3}), // 4-封底
        panelMaterials[1] = new THREE.MeshStandardMaterial({map: t2, roughness: 0.3})  // 2-内页左
      ];

      const rightMats = [
        new THREE.MeshStandardMaterial({color: 0x1e293b}),
        new THREE.MeshStandardMaterial({color: 0x1e293b}),
        new THREE.MeshStandardMaterial({color: 0x1e293b}),
        new THREE.MeshStandardMaterial({color: 0x1e293b}),
        panelMaterials[2] = new THREE.MeshStandardMaterial({map: t3, roughness: 0.3}), // 3-内页右
        panelMaterials[0] = new THREE.MeshStandardMaterial({map: t1, roughness: 0.3})  // 1-封面
      ];

      leftWingPivot = new THREE.Group();
      scene.add(leftWingPivot);
      const leftMesh = new THREE.Mesh(geom, leftMats);
      leftMesh.position.set(-cardW / 2, 0, 0);
      leftWingPivot.add(leftMesh);

      rightWingPivot = new THREE.Group();
      scene.add(rightWingPivot);
      const rightMesh = new THREE.Mesh(geom, rightMats);
      rightMesh.position.set(cardW / 2, 0, 0);
      rightWingPivot.add(rightMesh);

      window.addEventListener('resize', () => {
        camera.aspect = container.clientWidth / container.clientHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(container.clientWidth, container.clientHeight);
      });

      animate();
    }

    function animate() {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    }

    function setAngle(deg) {
      document.getElementById('angleSlider').value = deg;
      onSlider(deg);
      document.querySelectorAll('.presets .btn').forEach(b => b.classList.remove('active'));
      if (deg === 0) event.target.classList.add('active');
      if (deg === 90) document.getElementById('btnHalf').classList.add('active');
      if (deg === 170) event.target.classList.add('active');
    }

    function onSlider(val) {
      const angle = parseFloat(val);
      document.getElementById('sliderVal').textContent = Math.round(angle) + '°';
      document.getElementById('hudAngle').textContent = '夹角: ' + angle.toFixed(0) + '°';

      const halfRad = THREE.MathUtils.degToRad(angle / 2);
      if (leftWingPivot && rightWingPivot) {
        leftWingPivot.rotation.y = halfRad;
        rightWingPivot.rotation.y = -halfRad;
      }
    }

    function handleImageUpload(event, index) {
      const file = event.target.files[0];
      if (!file) return;

      const reader = new FileReader();
      reader.onload = function(e) {
        const img = new Image();
        img.crossOrigin = 'anonymous';
        img.onload = function() {
          const texture = new THREE.Texture(img);
          texture.colorSpace = THREE.SRGBColorSpace;
          texture.needsUpdate = true;
          
          panelMaterials[index].map = texture;
          panelMaterials[index].needsUpdate = true;

          const box = document.getElementById('box' + (index + 1));
          let thumb = box.querySelector('img');
          if (!thumb) {
            thumb = document.createElement('img');
            box.appendChild(thumb);
          }
          thumb.src = e.target.result;
        };
        img.src = e.target.result;
      };
      reader.readAsDataURL(file);
    }

    window.onload = init3D;
  </script>
</body>
</html>
