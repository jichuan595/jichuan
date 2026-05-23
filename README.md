// ============================================================
//  粒子照片展示  —  canvas 全绘图实现
// ============================================================
(function () {
  'use strict';

  let canvas, ctx, W, H;
  let images = [];
  let mode = 'dragon';
  const _isMobile = /Android|iPhone|iPad|iPod|webOS|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent) || innerWidth < 768;
  const _dpr = _isMobile ? 1 : Math.min(devicePixelRatio, 2); // 手机降级DPR省性能

  // 根据当前模式过滤图片：mode='all'的图片在所有模式中显示
  function modeImages() {
    return images.filter(img => !img.mode || img.mode === 'all' || img.mode === mode);
  }
  let t = 0;
  let mx = -999, my = -999;
  let _cards = []; // 抽卡状态（多张）
  let _sparkles = []; // 特效粒子
  let _smx = 0, _smy = 0; // 平滑鼠标位置
  let _petals = []; // 氛围飘浮花瓣
  let hovered = null;
  let enlarged = null;
  let bg = [];
  let eHP = [];
  let ePhase = 'hearts';
  let eTimer = 0;
  let eFormT = 0;
  let s3d = null; // 3D场景状态

  /* ========== 启动 ========== */
  window.addEventListener('DOMContentLoaded', async () => {
    canvas = document.getElementById('mainCanvas');
    ctx = canvas.getContext('2d');
    rsz();
    window.addEventListener('resize', rsz);
    bind();
    bindUI();
    initBg();

    try {
      const data = await fetch('/api/images').then(r => r.json());
      images = data.map((d, i) => ({
        ...d, i,
        cx: W / 2, cy: H / 2, tx: W / 2, ty: H / 2,
        sc: 1, tsc: 1, al: 0, ta: 1, ro: 0, tr: 0,
        imgObj: null, loaded: false, hover: false, _sa: 1
      }));

      // 逐张加载，加载一张显示一张
      function loadOne(it) {
        return new Promise(ok => {
          const im = new Image();
          im.crossOrigin = 'anonymous';
          const timer = setTimeout(() => { im.src = ''; ok(); }, 8000);
          im.onload = () => { clearTimeout(timer); it.imgObj = im; it.loaded = true; if (!it.width) it.width = im.naturalWidth; if (!it.height) it.height = im.naturalHeight; ok(); };
          im.onerror = () => { clearTimeout(timer); ok(); };
          im.src = it.thumb || it.url;
        });
      }

      // 先启动动画和爱心入场
      mkHearts();
      if (images.length && images.length <= 60) { mode = 'heart'; activeBtn('heart'); }
      else { mode = 'dragon'; activeBtn('dragon'); }
      ePhase = 'hearts'; eTimer = 0;
      loop();

      // 后台逐批加载图片
      const BATCH = 6;
      for (let i = 0; i < images.length; i += BATCH) {
        const batch = images.slice(i, i + BATCH);
        await Promise.all(batch.map(loadOne));
        const loaded = images.filter(b => b.loaded).length;
        document.getElementById('info').textContent = loaded + '/' + data.length + ' 张照片';
        reposition();
      }

      images = images.filter(i => i.loaded);
      document.getElementById('info').textContent = images.length + ' 张照片';

      if (!images.length) {
        const d = document.createElement('div');
        d.style.cssText = 'position:fixed;top:50%;left:50%;transform:translate(-50%,-50%);z-index:10;color:#666;font:16px sans-serif;text-align:center';
        d.innerHTML = '未找到图片<br><span style="font-size:12px;color:#444">请将图片放入 photos/ 文件夹</span>';
        document.body.appendChild(d);
      }
      reposition();
    } catch (e) {
      document.getElementById('info').textContent = '加载失败: ' + e.message;
      console.error(e);
      mkHearts();
      ePhase = 'hearts'; eTimer = 0;
      loop();
    }
  });

  function rsz() {
    W = innerWidth; H = innerHeight;
    canvas.width = W * _dpr; canvas.height = H * _dpr;
    canvas.style.width = W + 'px'; canvas.style.height = H + 'px';
    ctx.setTransform(_dpr, 0, 0, _dpr, 0, 0);
  }

  /* ========== 输入 ========== */
  let _touchX = 0, _touchY = 0, _isTouch = false;
  function bind() {
    const c = canvas;
    // 检测触摸设备
    _isTouch = 'ontouchstart' in window || navigator.maxTouchPoints > 0;
    if (_isTouch) {
      document.getElementById('hint').textContent = '点击图片放大 | 点击空白处退出';
    }
    // 鼠标
    c.addEventListener('mousemove', e => { mx = e.clientX; my = e.clientY; });
    c.addEventListener('mouseleave', () => { mx = my = -999; });
    c.addEventListener('click', onClick);
    // 触摸
    c.addEventListener('touchstart', e => {
      const p = e.touches[0];
      mx = p.clientX; my = p.clientY;
      _touchX = p.clientX; _touchY = p.clientY;
    }, { passive: true });
    c.addEventListener('touchmove', e => {
      mx = e.touches[0].clientX; my = e.touches[0].clientY;
    }, { passive: true });
    c.addEventListener('touchend', e => {
      if (!e.changedTouches.length) return;
      const p = e.changedTouches[0];
      mx = p.clientX; my = p.clientY;
      // 只有触摸移动距离小于阈值才算点击
      const dx = p.clientX - _touchX, dy = p.clientY - _touchY;
      if (dx * dx + dy * dy < 400) {
        if (enlarged) dismiss();
        else onClick();
      }
    });
    document.addEventListener('keydown', e => { if (e.key === 'Escape' && enlarged) dismiss(); });
  }
  function bindUI() {
    document.querySelectorAll('.mode-btn[data-mode]').forEach(b => b.addEventListener('click', () => swMode(b.dataset.mode)));
  }
  function activeBtn(n) {
    document.querySelectorAll('.mode-btn').forEach(b => b.classList.remove('active'));
    const b = document.querySelector('[data-mode="' + n + '"]');
    if (b) b.classList.add('active');
  }

  /* ========== 工具 ========== */
  function iSz() {
    const n = images.length; if (!n) return 100;
    if (mode === 'heart') {
      return Math.min(100, Math.max(35, Math.min(W, H) / (Math.sqrt(n) * 2.5)));
    }
    const cols = Math.ceil(Math.sqrt(n * W / H));
    const rows = Math.ceil(n / cols);
    return Math.min(150, Math.max(55, Math.min((W - 80) / cols, (H - 160) / rows) * .8));
  }
  function rr(c, x, y, w, h, r) {
    c.beginPath(); c.moveTo(x + r, y);
    c.lineTo(x + w - r, y); c.quadraticCurveTo(x + w, y, x + w, y + r);
    c.lineTo(x + w, y + h - r); c.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
    c.lineTo(x + r, y + h); c.quadraticCurveTo(x, y + h, x, y + h - r);
    c.lineTo(x, y + r); c.quadraticCurveTo(x, y, x + r, y); c.closePath();
  }
  function heartPath(c, s) {
    c.beginPath(); c.moveTo(0, -s * .4);
    c.bezierCurveTo(-s * .5, -s, -s, -s * .3, 0, s * .7);
    c.moveTo(0, -s * .4);
    c.bezierCurveTo(s * .5, -s, s, -s * .3, 0, s * .7);
  }

  /* ========== 背景爱心粒子（轰轰烈烈的爱意） ========== */
  let _heartbeat = 0; // 心跳节拍

  function initBg() {
    const hot = [
      'rgba(255,40,80,.85)', 'rgba(255,60,100,.8)', 'rgba(255,20,60,.9)',
      'rgba(255,100,130,.7)', 'rgba(255,80,60,.75)'
    ];
    const warm = [
      'rgba(255,130,160,.6)', 'rgba(255,160,180,.55)', 'rgba(255,180,200,.5)',
      'rgba(255,100,140,.65)', 'rgba(255,200,150,.5)'
    ];
    const cx = W / 2, cy = H / 2;
    const mf = _isMobile ? .4 : 1; // 手机降为40%粒子

    // 底层密集粒子（1200个，整片红雾）
    for (let i = 0, n = Math.round(1200 * mf); i < n; i++) bg.push({
      x: Math.random() * W, y: Math.random() * H,
      sz: 1 + Math.random() * 3,
      vx: (Math.random() - .5) * .3, vy: (Math.random() - .5) * .3,
      a: .15 + Math.random() * .25,
      ph: Math.random() * 6.28,
      ps: .4 + Math.random() * .8,
      col: warm[Math.floor(Math.random() * warm.length)],
      layer: 0
    });

    // 爆冲层（800个，从中心疯狂向外喷射）
    for (let i = 0, n = Math.round(800 * mf); i < n; i++) {
      const ang = Math.random() * 6.28;
      const dist = Math.random() * Math.max(W, H) * .7;
      const speed = .8 + Math.random() * 3.5;
      const sz = 3 + Math.random() * 12;
      bg.push({
        x: cx + Math.cos(ang) * dist,
        y: cy + Math.sin(ang) * dist,
        sz,
        a: .4 + Math.random() * .5,
        ph: Math.random() * 6.28,
        ps: 1 + Math.random() * 2.5,
        col: hot[Math.floor(Math.random() * hot.length)],
        layer: 1,
        ang, spd: speed, dist,
        maxDist: Math.max(W, H) * .85,
        szBase: sz
      });
    }

    // 巨型爱心（50个，冲击力最强）
    for (let i = 0, n = Math.round(50 * mf); i < n; i++) {
      const ang = Math.random() * 6.28;
      const dist = Math.random() * Math.max(W, H) * .5;
      const speed = 1.5 + Math.random() * 4;
      bg.push({
        x: cx + Math.cos(ang) * dist,
        y: cy + Math.sin(ang) * dist,
        sz: 15 + Math.random() * 25,
        a: .2 + Math.random() * .35,
        ph: Math.random() * 6.28,
        ps: .8 + Math.random() * 1.5,
        col: hot[Math.floor(Math.random() * hot.length)],
        layer: 2,
        ang, spd: speed, dist,
        maxDist: Math.max(W, H) * .9
      });
    }
  }

  /* ========== 入场爱心 ========== */
  function mkHearts() {
    eHP = [];
    const cols = ['rgba(255,100,120,.7)', 'rgba(255,150,180,.6)', 'rgba(255,200,220,.5)', 'rgba(255,80,100,.65)', 'rgba(255,170,190,.55)'];
    for (let i = 0; i < 150; i++) eHP.push({
      x: Math.random() * W, y: Math.random() * H,
      ox: W * .2 + Math.random() * W * .6, oy: H * .2 + Math.random() * H * .6,
      vx: (Math.random() - .5) * 2, vy: (Math.random() - .5) * 2,
      sc: .3 + Math.random() * .7, a: .25 + Math.random() * .45,
      rot: (Math.random() - .5) * .5, col: cols[Math.floor(Math.random() * 5)]
    });
  }

  /* ========== 主循环 ========== */
  function loop() {
    requestAnimationFrame(loop);
    t += 1 / 60;

    // 入场相位
    if (ePhase === 'hearts') { eTimer += 1 / 60; if (eTimer > 3) { ePhase = 'forming'; eFormT = t; } }
    else if (ePhase === 'forming') { if (t - eFormT > 1.5) ePhase = 'done'; }

    // 模式运动
    updMode();
    updGParts();
    updCard();

    // 平滑鼠标
    _smx += (mx - _smx) * .15;
    _smy += (my - _smy) * .15;

    // 飘浮花瓣（每隔一段时间生成，手机减少）
    const petalMax = _isMobile ? 15 : 60;
    while (_petals.length > petalMax) _petals.shift();
    if (Math.random() < (_isMobile ? .02 : .06)) {
      _petals.push({
        x: Math.random() * W, y: H + 10,
        vx: (Math.random() - .5) * .4, vy: -.3 - Math.random() * .6,
        sz: 3 + Math.random() * 5, rot: Math.random() * 6.28,
        rv: (Math.random() - .5) * .02,
        a: .15 + Math.random() * .2,
        ph: Math.random() * 6.28,
        col: ['rgba(255,100,140,.5)', 'rgba(255,150,180,.4)', 'rgba(255,200,220,.35)'][Math.floor(Math.random() * 3)]
      });
    }

    // 放大态粒子更新
    if (enlarged && enlarged.state === 'particles' && enlarged.particles.length) {
      const el = t - enlarged.convT;
      const dur = 1.2; // 总时长
      let allDone = true;
      for (const pt of enlarged.particles) {
        if (pt.done) continue;
        // 每个粒子根据距离有不同延迟，形成波浪汇聚
        const dp = Math.min(1, Math.max(0, (el - pt.delay) / (dur - pt.delay)));
        if (dp <= 0) { allDone = false; continue; }
        const ease = 1 - Math.pow(1 - dp, 4); // 缓出
        pt.x = pt.sx + (pt.ox - pt.sx) * ease;
        pt.y = pt.sy + (pt.oy - pt.sy) * ease;
        if (dp >= 1) pt.done = true;
        else allDone = false;
      }
      if (el >= dur && allDone) {
        enlarged.state = 'fading';
        enlarged.fadeT = t;
      }
    }
    // 粒子→图片过渡
    if (enlarged && enlarged.state === 'fading') {
      if (t - enlarged.fadeT > .5) enlarged.state = 'done';
    }

    // 入场爱心
    for (const p of eHP) {
      if (ePhase === 'hearts') {
        p.x += p.vx; p.y += p.vy;
        p.vx += (Math.random() - .5) * .04; p.vy += (Math.random() - .5) * .04;
        if (p.x < 0 || p.x > W) p.vx *= -.7; if (p.y < 0 || p.y > H) p.vy *= -.7;
        p.x = Math.max(0, Math.min(W, p.x)); p.y = Math.max(0, Math.min(H, p.y));
      } else if (ePhase === 'forming') {
        const pr = Math.min(1, (t - eFormT) / 1.5);
        const ease = 1 - Math.pow(1 - pr, 3);
        p.x += (p.ox - p.x) * ease * .1; p.y += (p.oy - p.y) * ease * .1;
      }
    }

    // 图片平滑插值
    for (const img of images) {
      img.cx += (img.tx - img.cx) * .07;
      img.cy += (img.ty - img.cy) * .07;
      img.sc += (img.tsc - img.sc) * (img.hover ? .06 : .1);
      img.al += (img.ta - img.al) * .08;
      img.ro += (img.tr - img.ro) * .1;
    }

    // 悬停检测（距离驱动，涟漪式放大，手机跳过）
    hovered = null;
    const drawnImgs = new Set(_cards.map(c => c.img));
    if (!_isMobile && !enlarged && mx > 0) {
      const hoverR = 280;
      let closest = null, closestD = 1e9;
      for (let i = 0; i < images.length; i++) {
        const im = images[i];
        if (drawnImgs.has(im)) continue;
        const d = Math.hypot(mx - im.cx, my - im.cy);
        if (d < hoverR) {
          const ratio = 1 - d / hoverR;
          const eased = ratio * ratio * (3 - 2 * ratio);
          const target = 1 + eased * 2;
          im.tsc = target;
          im.hover = eased > .05;
          if (d < closestD) { closestD = d; closest = im; }
        } else {
          im.hover = false;
          im.tsc = 1;
        }
      }
      hovered = closest;
    } else if (!enlarged) {
      for (const im of images) {
        if (!drawnImgs.has(im)) { im.hover = false; im.tsc = 1; }
      }
    }

    /* ========== 绘制 ========== */
    ctx.clearRect(0, 0, W, H);

    // 背景爱心粒子（轰轰烈烈）
    const cx = W / 2, cy = H / 2;
    // 心跳节拍：约每0.85秒一次强烈脉冲
    _heartbeat = (Math.sin(t * 7.4) + 1) / 2; // 0~1
    const hb = Math.pow(_heartbeat, 3); // 立方加强峰值感

    for (const p of bg) {
      p.ph += p.ps / 60;

      if (p.layer === 1 || p.layer === 2) {
        // 爆冲层 + 巨型层：从中心向外喷射
        p.dist += p.spd * (1 + hb * .6); // 心跳时加速
        if (p.dist > p.maxDist) {
          p.dist = 10 + Math.random() * 30;
          p.ang = Math.random() * 6.28;
        }
        p.x = cx + Math.cos(p.ang) * p.dist;
        p.y = cy + Math.sin(p.ang) * p.dist;

        const depth = Math.min(1, p.dist / p.maxDist);
        const pulse = 1 + hb * .5; // 心跳放大
        const sc = p.layer === 2
          ? (p.sz * (.4 + depth * .8) * pulse)
          : (p.sz * (.3 + depth * 1.5) * pulse);
        const al = p.layer === 2
          ? (p.a * (.3 + depth * .7) * (.6 + .4 * Math.abs(Math.sin(p.ph))) * (1 + hb * .5))
          : (p.a * (.2 + depth * .8) * (.5 + .5 * Math.abs(Math.sin(p.ph))) * (1 + hb * .4));

        ctx.save();
        ctx.globalAlpha = Math.min(1, al);
        ctx.fillStyle = p.col;
        ctx.shadowColor = p.col;
        ctx.shadowBlur = (p.layer === 2 ? 40 : 20) + depth * 30 + hb * 20;
        ctx.translate(p.x, p.y);
        ctx.scale(sc, sc);
        heartPath(ctx, 1); ctx.fill();
        ctx.restore();

      } else {
        // 底层密雾：随心跳脉动
        p.x += p.vx + (Math.random() - .5) * .1;
        p.y += p.vy + (Math.random() - .5) * .1;
        if (p.x < -10) p.x = W + 10; if (p.x > W + 10) p.x = -10;
        if (p.y < -10) p.y = H + 10; if (p.y > H + 10) p.y = -10;

        ctx.save();
        ctx.globalAlpha = p.a * (.3 + .7 * Math.abs(Math.sin(p.ph))) * (1 + hb * .3);
        ctx.fillStyle = p.col;
        ctx.shadowColor = p.col;
        ctx.shadowBlur = 6 + hb * 12;
        ctx.translate(p.x, p.y);
        ctx.scale(p.sz * (1 + hb * .2), p.sz * (1 + hb * .2));
        heartPath(ctx, 1); ctx.fill();
        ctx.restore();
      }
    }

    // 中心强光脉冲（心跳感）
    if (hb > .05) {
      ctx.save();
      const glow = ctx.createRadialGradient(cx, cy, 0, cx, cy, 200 + hb * 200);
      glow.addColorStop(0, 'rgba(255,60,80,' + (.12 * hb) + ')');
      glow.addColorStop(.4, 'rgba(255,40,60,' + (.06 * hb) + ')');
      glow.addColorStop(1, 'rgba(255,20,40,0)');
      ctx.fillStyle = glow;
      ctx.fillRect(0, 0, W, H);
      ctx.restore();
    }

    // 入场爱心粒子
    if (ePhase !== 'done') {
      const fadeOut = ePhase === 'forming' ? Math.max(0, 1 - (t - eFormT) / 1.5) : 1;
      for (const p of eHP) {
        ctx.save();
        ctx.globalAlpha = p.a * fadeOut;
        if (ctx.globalAlpha > .01) {
          ctx.fillStyle = p.col;
          ctx.translate(p.x, p.y); ctx.rotate(p.rot); ctx.scale(p.sc, p.sc);
          heartPath(ctx, 1); ctx.fill();
        }
        ctx.restore();
      }
    }

    // 图片（跳过正在抽卡中的图片）
    const eFade = ePhase === 'done' ? 1 : Math.max(0, Math.min(1, (eTimer - 2.5) / 1));
    const drawnSet = new Set(_cards.filter(c => c.phase === 'display').map(c => c.img));
    for (const img of images) {
      if (!img.loaded || !img.imgObj) continue;
      if (drawnSet.has(img)) continue;
      const a = img.al * eFade;
      if (a < .01) continue;
      ctx.save(); ctx.globalAlpha = a;
      ctx.translate(img.cx, img.cy);
      ctx.rotate(img.ro * Math.PI / 180);
      const s = iSz() * img.sc, h = s * (img.height / img.width || 1);
      // 点击闪光效果
      if (img._clickFlash) {
        const fp = Math.min(1, (t - img._clickFlash) / .35);
        if (fp < 1) {
          const fa = (1 - fp) * .8;
          const fr = 1 + fp * .3;
          ctx.save();
          ctx.globalAlpha = fa * a;
          ctx.strokeStyle = 'rgba(255,180,210,' + fa + ')'; ctx.lineWidth = 2;
          ctx.shadowColor = 'rgba(255,140,190,.9)'; ctx.shadowBlur = 20 + fp * 20;
          rr(ctx, -s / 2 * fr, -h / 2 * fr, s * fr, h * fr, 8); ctx.stroke();
          ctx.restore();
        } else { img._clickFlash = 0; }
      }
      // 放大进度（驱动发光和边框同步渐变）
      const hoverP = Math.max(0, Math.min(1, (img.sc - 1) / 1.5));
      if (hoverP > .01) {
        ctx.shadowColor = 'rgba(255,120,170,' + (.6 * hoverP) + ')';
        ctx.shadowBlur = 25 * hoverP;
      }
      rr(ctx, -s / 2, -h / 2, s, h, 6); ctx.clip();
      ctx.drawImage(img.imgObj, -s / 2, -h / 2, s, h);
      ctx.restore();
      // 悬停边框（跟放大同步渐变）
      if (hoverP > .01) {
        ctx.save(); ctx.globalAlpha = a * hoverP;
        ctx.translate(img.cx, img.cy);
        ctx.rotate(img.ro * Math.PI / 180);
        ctx.strokeStyle = 'rgba(255,130,180,' + (.7 * hoverP) + ')';
        ctx.lineWidth = 2;
        ctx.shadowColor = 'rgba(255,120,170,' + (.5 * hoverP) + ')';
        ctx.shadowBlur = 20 * hoverP;
        rr(ctx, -s / 2, -h / 2, s, h, 6); ctx.stroke();
        ctx.restore();
      }
    }

    // 抽卡渲染
    drawCard();

    // 飘浮花瓣
    for (let i = _petals.length - 1; i >= 0; i--) {
      const p = _petals[i];
      p.x += p.vx + Math.sin(t * .8 + p.ph) * .3;
      p.y += p.vy;
      p.rot += p.rv;
      if (p.y < -20) { _petals.splice(i, 1); continue; }
      ctx.save();
      ctx.globalAlpha = p.a;
      ctx.translate(p.x, p.y);
      ctx.rotate(p.rot);
      ctx.fillStyle = p.col;
      ctx.shadowColor = p.col; ctx.shadowBlur = 6;
      heartPath(ctx, p.sz); ctx.fill();
      ctx.restore();
    }

    // 特效粒子（点击 + 落地，手机限数量）
    if (_isMobile && _sparkles.length > 40) _sparkles.splice(0, _sparkles.length - 40);
    for (let i = _sparkles.length - 1; i >= 0; i--) {
      const s = _sparkles[i];
      s.age += 1 / 60;
      if (s.age >= s.life) { _sparkles.splice(i, 1); continue; }
      s.x += s.vx; s.y += s.vy;
      s.vx *= .94; s.vy *= .94;
      s.a = 1 - s.age / s.life;
      ctx.save();
      ctx.globalAlpha = s.a;
      ctx.fillStyle = s.col;
      ctx.shadowColor = s.col; ctx.shadowBlur = 10;
      ctx.beginPath();
      ctx.arc(s.x, s.y, s.sz * (1 - s.age / s.life * .5), 0, 6.28);
      ctx.fill();
      ctx.restore();
    }

    // 鼠标跟随柔光（手机不需要）
    if (!_isMobile && mx > 0 && !enlarged) {
      ctx.save();
      const glow = ctx.createRadialGradient(_smx, _smy, 0, _smx, _smy, 120);
      glow.addColorStop(0, 'rgba(255,150,190,.08)');
      glow.addColorStop(.5, 'rgba(255,100,150,.03)');
      glow.addColorStop(1, 'rgba(255,80,130,0)');
      ctx.fillStyle = glow;
      ctx.beginPath(); ctx.arc(_smx, _smy, 120, 0, 6.28); ctx.fill();
      ctx.restore();
    }

    // 聚散模式粒子
    drawGParts();
    drawFwEffects();

    // 悬停名称（已移除）

    // 放大态
    if (enlarged) {
      const im = enlarged.img;
      if (enlarged.state === 'fading' && enlarged.fadeT) {
        // 粒子→图片过渡中
        const fadeP = Math.min(1, (t - enlarged.fadeT) / .5);
        const fadeEase = fadeP * fadeP;
        ctx.save(); ctx.fillStyle = 'rgba(0,0,0,.85)'; ctx.fillRect(0, 0, W, H); ctx.restore();
        if (fadeEase < 1) {
          for (const p of enlarged.particles) {
            const a = Math.max(0, (p.a / 255) * (1 - fadeEase));
            if (a < .01) continue;
            ctx.save(); ctx.globalAlpha = a;
            ctx.fillStyle = 'rgb(' + p.r + ',' + p.g + ',' + p.b + ')';
            ctx.beginPath(); ctx.arc(p.x, p.y, p.sz, 0, 6.283); ctx.fill();
            ctx.restore();
          }
        }
        if (fadeEase > 0) {
          ctx.save();
          const src = im.fullImg || im.imgObj;
          const scl = Math.min(W * .88 / (im.fullW || im.width), H * .85 / (im.fullH || im.height));
          const w = (im.fullW || im.width) * scl, h = (im.fullH || im.height) * scl;
          ctx.globalAlpha = fadeEase;
          ctx.shadowColor = 'rgba(0,0,0,.6)'; ctx.shadowBlur = 50;
          rr(ctx, W / 2 - w / 2, H / 2 - h / 2, w, h, 10); ctx.clip();
          ctx.drawImage(src, W / 2 - w / 2, H / 2 - h / 2, w, h);
          ctx.restore();
        }
      }
      else if (enlarged.state === 'particles') {
        // 粒子汇聚中
        ctx.save(); ctx.fillStyle = 'rgba(0,0,0,.85)'; ctx.fillRect(0, 0, W, H); ctx.restore();
        for (const p of enlarged.particles) {
          ctx.save(); ctx.globalAlpha = Math.max(0, p.a / 255);
          ctx.fillStyle = 'rgb(' + p.r + ',' + p.g + ',' + p.b + ')';
          ctx.beginPath(); ctx.arc(p.x, p.y, p.sz, 0, 6.283); ctx.fill();
          ctx.restore();
        }
      } else if (enlarged.state === 'scatter') {
        // 粒子散开
        const el = t - enlarged.scatT;
        const fade = Math.max(0, 1 - el / .5);
        ctx.save(); ctx.fillStyle = 'rgba(0,0,0,' + (.85 * fade) + ')'; ctx.fillRect(0, 0, W, H); ctx.restore();
        for (const p of enlarged.particles) {
          const a = Math.max(0, (p.a / 255) * fade);
          if (a < .01) continue;
          ctx.save(); ctx.globalAlpha = a;
          ctx.fillStyle = 'rgb(' + p.r + ',' + p.g + ',' + p.b + ')';
          ctx.beginPath(); ctx.arc(p.x, p.y, p.sz * (.5 + fade * .5), 0, 6.283); ctx.fill();
          ctx.restore();
        }
      } else if (enlarged.state === 'done') {
        // 粒子汇聚完成 → 显示放大图片
        ctx.save();
        ctx.fillStyle = 'rgba(0,0,0,.85)'; ctx.fillRect(0, 0, W, H);
        const src = im.fullImg || im.imgObj;
        const scl = Math.min(W * .88 / (im.fullW || im.width), H * .85 / (im.fullH || im.height));
        const w = (im.fullW || im.width) * scl, h = (im.fullH || im.height) * scl;
        ctx.shadowColor = 'rgba(0,0,0,.6)'; ctx.shadowBlur = 50;
        rr(ctx, W / 2 - w / 2, H / 2 - h / 2, w, h, 10); ctx.clip();
        ctx.globalAlpha = 1;
        ctx.drawImage(src, W / 2 - w / 2, H / 2 - h / 2, w, h);
        ctx.restore();
        ctx.save(); ctx.globalAlpha = .4;
        ctx.fillStyle = '#fff'; ctx.font = '13px sans-serif'; ctx.textAlign = 'center';
        ctx.fillText('按 ESC 退出', W / 2, H - 30);
        ctx.restore();
      }
    }
  }

  /* ========== 模式运动 ========== */
  function updMode() {
    if (enlarged || !images.length) return;
    switch (mode) {
      case 'dragon': updDragon(); break;
      case 'heart': updHeart(); break;
      case 'ripple': updRipple(); break;
      case 'waterfall': updWater(); break;
      case 'gather': updGather(); break;
      case 'firework': updFirework(); break;
      case 'scene3d': break;
    }
  }

  /* ========== 模式切换 ========== */
  function swMode(m) {
    if (enlarged) dismiss();
    for (const c of _cards) { c.img.tsc = 1; c.img.tr = 0; }
    _cards = [];
    mode = m;
    activeBtn(m);
    const c3d = document.getElementById('canvas3d');
    if (m === 'scene3d') {
      canvas.style.display = 'none';
      c3d.style.display = 'block';
      init3d();
    } else {
      c3d.style.display = 'none';
      canvas.style.display = 'block';
      if (s3d) s3d.active = false;
    }
    reposition();
  }

  /* ========== 模式定位 ========== */
  function reposition() {
    if (!images.length) return;
    switch (mode) {
      case 'dragon': updDragon(); break;
      case 'heart': posHeart(); break;
      case 'ripple': posRipple(); break;
      case 'waterfall': posWaterInit(); break;
      case 'gather': posGather(); break;
      case 'firework': posFirework(); break;
      case 'scene3d': break;
    }
  }

  /* -- 游龙 -- */
  /* 舞蝶 — 倒8动态流动 */
  function updDragon() {
    const imgs = modeImages();
    const n = imgs.length; if (!n) return;
    const spd = .6;
    const rx = W * .38, ry = H * .3;
    const cx = W / 2, cy = H / 2 + 20;

    for (let i = 0; i < n; i++) {
      const phase = (i / n) * Math.PI * 2 + t * spd;
      imgs[i].tx = cx + rx * Math.sin(phase);
      imgs[i].ty = cy + ry * Math.sin(2 * phase) * .5;
      imgs[i].tr = 0;
      imgs[i].tsc = 1;
    }
  }

  /* -- 爱心（小图片组成，呼吸动态） -- */
  /* -- 五心模式 -- */
  function posHeart() {
    const imgs = modeImages();
    const n = imgs.length; if (!n) return;

    // 心形曲线参数范围
    let xMin = 1e9, xMax = -1e9, yMin = 1e9, yMax = -1e9;
    const STEPS = 300;
    for (let i = 0; i <= STEPS; i++) {
      const tt = i / STEPS * 2 * Math.PI;
      const x = 16 * Math.pow(Math.sin(tt), 3);
      const y = -(13 * Math.cos(tt) - 5 * Math.cos(2 * tt) - 2 * Math.cos(3 * tt) - Math.cos(4 * tt));
      xMin = Math.min(xMin, x); xMax = Math.max(xMax, x);
      yMin = Math.min(yMin, y); yMax = Math.max(yMax, y);
    }

    // 5颗心配置：位置偏移、大小比例
    const configs = [
      { dx: 0, dy: 0, size: 1 },        // 中间最大
      { dx: -.38, dy: -.32, size: .42 },  // 左上
      { dx: .38, dy: -.32, size: .42 },   // 右上
      { dx: -.3, dy: .22, size: .38 },    // 左下
      { dx: .3, dy: .22, size: .38 }      // 右下
    ];

    // 中心基础缩放
    const s = iSz(), pad = s * 2;
    const sx = (W - pad * 2) / (xMax - xMin);
    const sy = (H - pad * 2 - 60) / (yMax - yMin);
    const baseSc = Math.min(sx, sy) * .95;

    // 分配图片：中间占75%，剩余均分给4颗小心
    const midCount = Math.round(n * 0.75);
    const restCount = n - midCount;
    const smallCount = restCount > 0 ? Math.ceil(restCount / 4) : 0;

    for (let i = 0; i < n; i++) {
      let hi;
      if (i < midCount) {
        hi = 0;
      } else {
        hi = 1 + Math.floor((i - midCount) / smallCount);
        if (hi > 4) hi = 4;
      }
      const c = configs[hi];
      const sc = baseSc * c.size;
      const idxInHeart = hi === 0 ? i : (i - midCount) % smallCount;
      const cnt = hi === 0 ? midCount : Math.min(smallCount, n - midCount - (hi - 1) * smallCount);

      const tt = (idxInHeart / cnt) * 2 * Math.PI;
      const x = 16 * Math.pow(Math.sin(tt), 3);
      const y = -(13 * Math.cos(tt) - 5 * Math.cos(2 * tt) - 2 * Math.cos(3 * tt) - Math.cos(4 * tt));

      imgs[i]._hIdx = hi;
      imgs[i]._hSc = sc;
      imgs[i]._hDx = c.dx;
      imgs[i]._hDy = c.dy;
      imgs[i]._hIdxIn = idxInHeart;
      imgs[i]._hCnt = cnt;
      imgs[i].tx = W / 2 + c.dx * W + x * sc;
      imgs[i].ty = H / 2 + 20 + c.dy * H + y * sc;
    }
  }

  function updHeart() {
    const imgs = modeImages();
    const n = imgs.length; if (!n) return;
    const speed = .3;

    for (let i = 0; i < n; i++) {
      const sc = imgs[i]._hSc || 1;
      const dx = imgs[i]._hDx || 0;
      const dy = imgs[i]._hDy || 0;
      const idx = imgs[i]._hIdxIn || 0;
      const cnt = imgs[i]._hCnt || n;
      const hi = imgs[i]._hIdx || 0;

      const tt = ((idx / cnt) + t * speed * .1) % 1 * 2 * Math.PI;
      const x = 16 * Math.pow(Math.sin(tt), 3);
      const y = -(13 * Math.cos(tt) - 5 * Math.cos(2 * tt) - 2 * Math.cos(3 * tt) - Math.cos(4 * tt));

      // 每颗心独立呼吸节奏
      const breath = 1 + Math.sin(t * 1.2 + hi * 1.5) * .08;
      imgs[i].tx = W / 2 + dx * W + x * sc * breath;
      imgs[i].ty = H / 2 + 20 + dy * H + y * sc * breath;
      // 中心心的图片略大
      imgs[i].tsc = hi === 0 ? 1.05 : 1;
    }
  }

  /* -- 涟漪（同心圆旋转） -- */
  function posRipple() {
    const imgs = modeImages();
    const n = imgs.length; if (!n) return;
    const cx = W / 2, cy = H / 2 + 20;
    const maxR = Math.min(W, H) * .45;
    const rings = Math.max(2, Math.ceil(Math.sqrt(n / 5))); // 圈数

    let idx = 0;
    for (let ring = 0; ring < rings && idx < n; ring++) {
      const radius = maxR * ((ring + 1) / rings);
      const count = Math.max(5, Math.round(8 + ring * 6)); // 外圈更多
      for (let j = 0; j < count && idx < n; j++) {
        const angle = (j / count) * 2 * Math.PI;
        imgs[idx]._ripR = radius;
        imgs[idx]._ripBaseAngle = angle;
        imgs[idx]._ripRing = ring;
        idx++;
      }
    }
    // 剩余的也放入最外圈
    while (idx < n) {
      imgs[idx]._ripR = maxR;
      imgs[idx]._ripBaseAngle = (idx / n) * 2 * Math.PI;
      imgs[idx]._ripRing = rings - 1;
      idx++;
    }
    updRipple();
  }

  function updRipple() {
    const imgs = modeImages();
    const n = imgs.length; if (!n) return;
    const cx = W / 2, cy = H / 2 + 20;
    for (let i = 0; i < n; i++) {
      const r = imgs[i]._ripR || 100;
      const base = imgs[i]._ripBaseAngle || 0;
      const ring = imgs[i]._ripRing || 0;
      const dir = ring % 2 === 0 ? 1 : -1;
      const angle = base + t * .2 * dir;
      imgs[i].tx = cx + r * Math.cos(angle);
      imgs[i].ty = cy + r * Math.sin(angle);
      imgs[i].tr = (angle * 180 / Math.PI) % 360;
      imgs[i].tsc = 1;
    }
  }

  /* -- 瀑布（满屏错峰） -- */
  function posWaterInit() {
    const imgs = modeImages();
    const n = imgs.length; if (!n) return;
    const s = iSz() * .7;
    const pad = s;
    let seed = 77;
    const rnd = () => { seed = (seed * 16807) % 2147483647; return seed / 2147483647; };

    for (let i = 0; i < n; i++) {
      imgs[i]._wX = pad + rnd() * (W - pad * 2);
      imgs[i]._wSpeed = 30 + rnd() * 60;        // 每张图下落速度不同
      imgs[i]._wStartY = -s - rnd() * (H + s * 2); // 随机初始偏移
      imgs[i]._wDrift = (rnd() - .5) * 40;       // 水平漂移幅度
      imgs[i]._wDriftSpd = .3 + rnd() * .5;      // 漂移频率
    }
    updWater();
  }

  function updWater() {
    const imgs = modeImages();
    const n = imgs.length; if (!n) return;
    const s = iSz() * .7;
    const cycleH = H + s * 3;

    for (let i = 0; i < n; i++) {
      const yRaw = (t * imgs[i]._wSpeed + imgs[i]._wStartY) % cycleH;
      const y = yRaw - s;
      const drift = Math.sin(t * imgs[i]._wDriftSpd + i) * imgs[i]._wDrift;
      imgs[i].tx = imgs[i]._wX + drift;
      imgs[i].ty = y;
      // 下落旋转
      imgs[i].tr = Math.sin(t * .4 + i * .7) * 8;
      // 顶部小、底部大（深度感）
      const depth = Math.max(0, Math.min(1, (y + s) / H));
      const baseSc = .55 + depth * .55;
      // 入场滑动放大：每轮循环顶部小 → 逐渐放大
      const enterP = Math.min(1, yRaw / (s * 3));
      const enterEase = 1 - Math.pow(1 - enterP, 3);
      imgs[i].tsc = baseSc * (.4 + enterEase * .6);
      // 入场淡入
      imgs[i].ta = Math.min(1, enterP * 2);
    }
  }

  /* -- 聚散（粒子特效轮播 + 聚散） -- */
  let _gPhase = 'enlarge', _gTimer = 0, _gIdx = 0, _gConvT = 0;

  function posGather() {
    _gPhase = 'enlarge'; _gTimer = 0; _gIdx = 0;
    for (const img of modeImages()) img._gParts = null;
    updGather();
  }

  // 为图片创建粒子
  function mkGParts(img) {
    const src = img.imgObj;
    if (!src) return [];
    const tc = document.createElement('canvas');
    const maxD = Math.min(350, Math.max(W, H) * .5);
    const scl = maxD / Math.max(src.naturalWidth || src.width, src.naturalHeight || img.height);
    tc.width = Math.floor((src.naturalWidth || src.width) * scl);
    tc.height = Math.floor((src.naturalHeight || src.height) * scl);
    if (tc.width < 1 || tc.height < 1) return [];
    const tcx = tc.getContext('2d');
    tcx.drawImage(src, 0, 0, tc.width, tc.height);
    let px;
    try { px = tcx.getImageData(0, 0, tc.width, tc.height).data; } catch (e) { return []; }
    const intv = Math.max(5, Math.floor(Math.min(tc.width, tc.height) / 40));
    const parts = [];
    const ccx = W / 2, ccy = H / 2;
    for (let y = 0; y < tc.height; y += intv) {
      for (let x = 0; x < tc.width; x += intv) {
        const idx = (y * tc.width + x) * 4;
        const r = px[idx], g = px[idx + 1], b = px[idx + 2], a = px[idx + 3];
        if (a < 30) continue;
        const ox = ccx - tc.width / 2 + x;
        const oy = ccy - tc.height / 2 + y;
        const ang = Math.atan2(y - tc.height / 2, x - tc.width / 2) + (Math.random() - .5) * 1.2;
        const dist = 250 + Math.random() * 350;
        const dx = (x - tc.width / 2) / (tc.width / 2);
        const dy = (y - tc.height / 2) / (tc.height / 2);
        const delay = Math.sqrt(dx * dx + dy * dy) * .35 + Math.random() * .08;
        parts.push({
          ox, oy,
          sx: ox + Math.cos(ang) * dist,
          sy: oy + Math.sin(ang) * dist,
          x: ox + Math.cos(ang) * dist,
          y: oy + Math.sin(ang) * dist,
          r, g, b, sz: intv * .45,
          delay, a: 1
        });
      }
    }
    return parts;
  }

  function updGather() {
    const imgs = modeImages();
    const n = imgs.length; if (!n) return;
    const cx = W / 2, cy = H / 2 + 20;

    if (_gPhase === 'enlarge') {
      const img = imgs[_gIdx];
      if (!img._gParts) img._gParts = mkGParts(img);
      _gConvT = t;
      img.ta = 0;
      _gPhase = 'converge';
    }
    else if (_gPhase === 'converge') {
      const dur = 1;
      const el = t - _gConvT;
      const parts = imgs[_gIdx]._gParts || [];
      for (const pt of parts) {
        const dp = Math.min(1, Math.max(0, (el - pt.delay) / (dur - pt.delay)));
        if (dp <= 0) continue;
        const ease = 1 - Math.pow(1 - dp, 4);
        pt.x = pt.sx + (pt.ox - pt.sx) * ease;
        pt.y = pt.sy + (pt.oy - pt.sy) * ease;
      }
      if (el >= dur + .2) { _gPhase = 'display'; _gTimer = 0; }
    }
    else if (_gPhase === 'display') {
      imgs[_gIdx].ta = 1;
      imgs[_gIdx].tsc = 2.2;
      imgs[_gIdx].tx = cx;
      imgs[_gIdx].ty = cy;
      _gTimer += 1 / 60;
      if (_gTimer >= 1.04) {
        _gPhase = 'scatter'; _gConvT = t;
      }
    }
    else if (_gPhase === 'scatter') {
      const dur = .8;
      const el = t - _gConvT;
      const p = Math.min(1, el / dur);
      const parts = imgs[_gIdx]._gParts || [];
      for (const pt of parts) {
        const ang = Math.atan2(pt.oy - cy, pt.ox - cx);
        const dist = 300 + Math.random() * 200;
        pt.x = pt.ox + Math.cos(ang) * dist * p;
        pt.y = pt.oy + Math.sin(ang) * dist * p;
        pt.a = 1 - p;
      }
      imgs[_gIdx].ta = 0;
      if (p >= 1) {
        imgs[_gIdx]._gParts = null;
        imgs[_gIdx].ta = 1;
        imgs[_gIdx].tsc = 1;
        _gIdx++;
        if (_gIdx >= n) { _gPhase = 'finalConv'; _gConvT = t; }
        else { _gPhase = 'enlarge'; }
      }
    }
    else if (_gPhase === 'finalConv') {
      const dur = 1.5;
      const p = Math.min(1, (t - _gConvT) / dur);
      const ease = 1 - Math.pow(1 - p, 3);
      for (let i = 0; i < n; i++) {
        const angle = (i / n) * 6.28 + t * .8;
        const radius = Math.min(W, H) * .36;
        const rx = cx + radius * Math.cos(angle);
        const ry = cy + radius * Math.sin(angle);
        imgs[i].tx = rx + (cx - rx) * ease;
        imgs[i].ty = ry + (cy - ry) * ease;
        imgs[i].tsc = 1 + .8 * ease;
        imgs[i].tr = 0;
      }
      if (p >= 1) { _gPhase = 'finalScat'; _gConvT = t; }
    }
    else if (_gPhase === 'finalScat') {
      const dur = 1.5;
      const p = Math.min(1, (t - _gConvT) / dur);
      const ease = 1 - Math.pow(1 - p, 2);
      for (let i = 0; i < n; i++) {
        const angle = (i / n) * 6.28;
        imgs[i].tx = cx + Math.cos(angle) * W * ease;
        imgs[i].ty = cy + Math.sin(angle) * H * ease;
        imgs[i].tsc = 1 + .8 * (1 - ease);
        imgs[i].tr += 3;
      }
      if (p >= 1) {
        _gPhase = 'enlarge'; _gIdx = 0;
        for (const img of imgs) { img.ta = 1; img.tsc = 1; img._gParts = null; }
      }
    }

    // 圆环上的图片（围在展示图片周围）
    const sz = iSz();
    const baseR = sz * 5.2;
    const expandR = sz * 2.2 / 2 + sz / 2 + 100;
    let ringR = baseR;
    if (_gPhase === 'converge') {
      const p = Math.min(1, (t - _gConvT) / 1);
      ringR = baseR + (expandR - baseR) * (1 - Math.pow(1 - p, 3));
    } else if (_gPhase === 'display') {
      ringR = expandR;
    }
    const rotSpd = _gPhase === 'display' ? .5 : 1.2;

    for (let i = 0; i < n; i++) {
      if (_gPhase === 'enlarge' || _gPhase === 'converge' || _gPhase === 'display' || _gPhase === 'scatter') {
        if (i === _gIdx) {
          if (_gPhase === 'display') {
            imgs[i].tx = cx; imgs[i].ty = cy; imgs[i].tsc = 2.2; imgs[i].ta = 1;
          }
          continue;
        }
      }
      const angle = (i / n) * 6.28 + t * rotSpd;
      imgs[i].tx = cx + ringR * Math.cos(angle);
      imgs[i].ty = cy + ringR * Math.sin(angle);
      imgs[i].tsc = 1;
      imgs[i].ta = 1;
      imgs[i].tr = 0;
    }
  }

  // 更新聚散粒子（主循环调用）
  function updGParts() {
    if (mode !== 'gather') return;
    if (_gPhase !== 'converge' && _gPhase !== 'scatter') return;
    const img = images[_gIdx];
    if (!img || !img._gParts) return;
    const el = t - _gConvT;
    const isConv = _gPhase === 'converge';
    for (const pt of img._gParts) {
      if (isConv) {
        const dp = Math.min(1, Math.max(0, (el - pt.delay) / (1 - pt.delay)));
        if (dp <= 0) continue;
        const ease = 1 - Math.pow(1 - dp, 4);
        pt.x = pt.sx + (pt.ox - pt.sx) * ease;
        pt.y = pt.sy + (pt.oy - pt.sy) * ease;
        pt.a = 1;
      } else {
        const p = Math.min(1, el / .8);
        const ang = Math.atan2(pt.oy - H / 2, pt.ox - W / 2);
        pt.x = pt.ox + Math.cos(ang) * 300 * p;
        pt.y = pt.oy + Math.sin(ang) * 300 * p;
        pt.a = 1 - p;
      }
    }
  }

  // 绘制聚散粒子
  function drawGParts() {
    if (mode !== 'gather') return;
    if (_gPhase !== 'converge' && _gPhase !== 'scatter') return;
    const img = images[_gIdx];
    if (!img || !img._gParts) return;
    for (const pt of img._gParts) {
      if (pt.a < .01) continue;
      ctx.save();
      ctx.globalAlpha = pt.a;
      ctx.fillStyle = 'rgb(' + pt.r + ',' + pt.g + ',' + pt.b + ')';
      ctx.beginPath();
      ctx.arc(pt.x, pt.y, pt.sz, 0, 6.28);
      ctx.fill();
      ctx.restore();
    }
  }

  /* ===== 烟花模式（点击触发烟花，伴随图片出现） ===== */
  let _fwIdx = 0;
  let _fwList = []; // 正在进行的烟花列表
  const FW_COLS = [[255,120,170],[255,180,220],[255,80,130],[255,200,100],[200,150,255]];

  function posFirework() {
    _fwIdx = 0;
    _fwList = [];
    for (const img of modeImages()) { img.ta = 0; }
  }

  function updFirework() {
    for (let i = _fwList.length - 1; i >= 0; i--) {
      const fw = _fwList[i];
      const el = t - fw.t0;
      if (el < 0) continue; // 延迟发射

      if (fw.phase === 'rise') {
        fw.ry += fw.vy;
        fw.vy += .06; // 减速
        // 小点上升
        if (fw.vy >= 0 || fw.ry <= fw.peakY) {
          fw.phase = 'burst';
          fw.burstT = t;
          fw.bx = fw.rx;
          fw.by = fw.ry;
          // 生成爆炸碎片
          const col = fw.col;
          for (let j = 0; j < 40; j++) {
            const ang = Math.random() * 6.28;
            const spd = 1.5 + Math.random() * 4;
            fw.sparks.push({
              x: fw.bx, y: fw.by,
              vx: Math.cos(ang) * spd,
              vy: Math.sin(ang) * spd - 1,
              life: .6 + Math.random() * .6,
              sz: 2 + Math.random() * 4
            });
          }
          // 图片出现在爆炸点
          const img = fw.img;
          img.tx = fw.bx;
          img.ty = fw.by;
          img.ta = 1;
          img.tsc = .15;
        }
      }
      else if (fw.phase === 'burst') {
        const bp = Math.min(1, (t - fw.burstT) / .4);
        const ease = 1 - Math.pow(1 - bp, 3);
        fw.img.tsc = .15 + ease * .85;
        // 更新碎片
        for (const s of fw.sparks) {
          s.x += s.vx;
          s.y += s.vy;
          s.vy += .12;
          s.vx *= .97;
          s.life -= .025;
        }
        if (t - fw.burstT > 1.2) {
          fw.phase = 'done';
        }
      }

      if (fw.phase === 'done') {
        _fwList.splice(i, 1);
      }
    }
  }

  function drawFwEffects() {
    if (mode !== 'firework') return;
    for (const fw of _fwList) {
      if (t < fw.t0) continue; // 未到发射时间
      // 上升小点
      if (fw.phase === 'rise') {
        ctx.save();
        ctx.globalAlpha = .9;
        const c = fw.col;
        ctx.fillStyle = 'rgb(' + c[0] + ',' + c[1] + ',' + c[2] + ')';
        ctx.shadowColor = ctx.fillStyle;
        ctx.shadowBlur = 18;
        ctx.beginPath();
        ctx.arc(fw.rx, fw.ry, 4, 0, 6.28);
        ctx.fill();
        // 尾迹
        ctx.globalAlpha = .25;
        ctx.beginPath();
        ctx.moveTo(fw.rx, fw.ry);
        ctx.lineTo(fw.rx, fw.ry + 25);
        ctx.strokeStyle = ctx.fillStyle;
        ctx.lineWidth = 2;
        ctx.stroke();
        ctx.restore();
      }
      // 爆炸碎片
      if (fw.phase === 'burst') {
        const c = fw.col;
        for (const s of fw.sparks) {
          if (s.life <= 0) continue;
          ctx.save();
          ctx.globalAlpha = Math.max(0, s.life);
          ctx.fillStyle = 'rgb(' + c[0] + ',' + c[1] + ',' + c[2] + ')';
          ctx.shadowColor = ctx.fillStyle;
          ctx.shadowBlur = 10;
          ctx.beginPath();
          ctx.arc(s.x, s.y, s.sz * Math.max(0, s.life), 0, 6.28);
          ctx.fill();
          ctx.restore();
        }
      }
    }
  }

  /* -- 粒子散布 -- */
  function posPartsInit() {
    const n = images.length; if (!n) return;
    let seed = 123;
    const rnd = () => { seed = (seed * 16807) % 2147483647; return seed / 2147483647; };
    const pad = iSz() * .6;
    for (let i = 0; i < n; i++) {
      images[i]._partX = pad + rnd() * (W - pad * 2);
      images[i]._partY = pad + 50 + rnd() * (H - pad * 2 - 50);
      images[i].tx = images[i]._partX;
      images[i].ty = images[i]._partY;
    }
  }

  function updPartsMode() {
    const n = images.length; if (!n) return;
    for (let i = 0; i < n; i++) {
      if (images[i]._partX !== undefined) {
        images[i].tx = images[i]._partX + Math.sin(t * .5 + i * 1.2) * 50;
        images[i].ty = images[i]._partY + Math.cos(t * .4 + i * .7) * 40;
        images[i].tsc = 1 + .1 * Math.sin(t * .8 + i);
        images[i].tr = 10 * Math.sin(t * .3 + i * .6);
      }
    }
  }

  /* ========== 点击放大 / 抽卡 ========== */
  function onClick() {
    if (enlarged) return;
    if (mode === 'waterfall') { onClickOld(); return; }

    // 烟花模式：点击触发射烟花，爆炸后图片出现（2张，左右分开）
    if (mode === 'firework') {
      const imgs = modeImages();
      const n = imgs.length; if (!n) return;
      const offsets = [-60, 60];
      for (let k = 0; k < 2; k++) {
        const img = imgs[_fwIdx % n];
        _fwIdx++;
        img.ta = 0;
        const col = FW_COLS[Math.floor(Math.random() * FW_COLS.length)];
        const ox = offsets[k];
        _fwList.push({
          img, col,
          rx: mx + ox, ry: my,
          vy: -(4 + Math.random() * 2),
          peakY: my - 120 - Math.random() * 80,
          phase: 'rise', t0: t + k * .12,
          sparks: []
        });
      }
      return;
    }

    // 检查是否点在已展示的卡片上
    let hitCard = null;
    for (const c of _cards) {
      if (c.phase !== 'display') continue;
      const im = c.img, s = iSz() * 2.2, h = s * (im.height / im.width || 1);
      if (Math.abs(mx - c.ex) < s / 2 + 10 && Math.abs(my - c.ey) < h / 2 + 10) { hitCard = c; break; }
    }
    if (hitCard) return; // 点卡片上不操作

    // 点空白处 → 收回所有卡片
    if (_cards.length && !hitCard) {
      let hitImg = null;
      for (let i = images.length - 1; i >= 0; i--) {
        const im = images[i], s = iSz() * im.sc, h = s * (im.height / im.width || 1);
        if (Math.abs(mx - im.cx) < s / 2 + 8 && Math.abs(my - im.cy) < h / 2 + 8) { hitImg = im; break; }
      }
      if (!hitImg) { dismissAllCards(); return; }
    }

    // 点击图片 → 抽卡（支持多张）
    let hit = null;
    for (let i = images.length - 1; i >= 0; i--) {
      const im = images[i], s = iSz() * im.sc, h = s * (im.height / im.width || 1);
      if (Math.abs(mx - im.cx) < s / 2 + 8 && Math.abs(my - im.cy) < h / 2 + 8) { hit = im; break; }
    }
    if (hit) {
      // 已在展示中的不重复抽
      if (_cards.some(c => c.img === hit && c.phase === 'display')) return;
      startCard(hit);
    }
  }

  function onClickOld() {
    let hit = null;
    for (let i = images.length - 1; i >= 0; i--) {
      const im = images[i], s = iSz() * im.sc, h = s * (im.height / im.width || 1);
      if (Math.abs(mx - im.cx) < s / 2 + 8 && Math.abs(my - im.cy) < h / 2 + 8) { hit = im; break; }
    }
    if (hit) zoom(hit);
  }

  /* ===== 多卡抽卡系统 ===== */
  function findEmptySpot() {
    const cardW = iSz() * 1.8, cardH = cardW * .75;
    const pad = 20;

    function overlaps(cx1, cy1, w1, h1, cx2, cy2, w2, h2) {
      return Math.abs(cx1 - cx2) < (w1 + w2) / 2 + pad &&
             Math.abs(cy1 - cy2) < (h1 + h2) / 2 + pad;
    }

    // 收集所有已占位置（含运动中的抽卡图片和画布上的图片）
    const rects = [];
    for (const c of _cards) {
      const im = c.img;
      const s = iSz() * 1.8, h = s * (im.height / im.width || 1);
      const px = c.phase === 'display' ? c.ex : im.tx;
      const py = c.phase === 'display' ? c.ey : im.ty;
      rects.push({ cx: px, cy: py, w: s, h });
    }
    for (const im of images) {
      if (_cards.some(c => c.img === im)) continue;
      const s = iSz() * Math.max(im.sc, 1), h = s * (im.height / im.width || 1);
      rects.push({ cx: im.tx, cy: im.ty, w: s, h });
    }

    // 全屏密集网格 + 随机偏移
    const marginX = cardW / 2 + 25;
    const marginTop = 90, marginBottom = 110;
    const cols = 10, rows = 7;
    const candidates = [];
    for (let r = 0; r < rows; r++) {
      for (let c = 0; c < cols; c++) {
        candidates.push({
          x: marginX + (W - marginX * 2) * c / (cols - 1) + (Math.random() - .5) * 24,
          y: marginTop + (H - marginTop - marginBottom) * r / (rows - 1) + (Math.random() - .5) * 16
        });
      }
    }

    // 筛选不重叠位置
    const valid = [];
    for (const sp of candidates) {
      let hit = false;
      for (const rc of rects) {
        if (overlaps(sp.x, sp.y, cardW, cardH, rc.cx, rc.cy, rc.w, rc.h)) { hit = true; break; }
      }
      if (!hit) valid.push(sp);
    }

    if (valid.length) return valid[Math.floor(Math.random() * valid.length)];

    // 全部重叠 → 选离所有障碍物最远的位置
    candidates.forEach(sp => {
      let minD = 1e9;
      for (const rc of rects) minD = Math.min(minD, Math.hypot(sp.x - rc.cx, sp.y - rc.cy));
      sp._d = minD;
    });
    candidates.sort((a, b) => b._d - a._d);
    return candidates[0];
  }

  function startCard(img) {
    mkSparkle(img.cx, img.cy);
    img._clickFlash = t;
    const spot = findEmptySpot();
    img._cardSX = img.cx; img._cardSY = img.cy;
    img._cardEX = spot.x; img._cardEY = spot.y;
    _cards.push({ img, phase: 'lift', startT: t + .15, ex: spot.x, ey: spot.y, fp: Math.random() * 6.28 });
    img.tsc = 1.8;
  }

  /* ===== 点击闪光粒子 ===== */
  function mkSparkle(x, y) {
    const cnt = _isMobile ? 6 : 14;
    for (let i = 0; i < cnt; i++) {
      const ang = (i / cnt) * 6.28 + Math.random() * .4;
      const spd = 1.5 + Math.random() * 3;
      _sparkles.push({
        x, y,
        vx: Math.cos(ang) * spd, vy: Math.sin(ang) * spd,
        a: 1, sz: 1.5 + Math.random() * 2,
        life: .4 + Math.random() * .2, age: 0,
        col: ['rgba(255,180,210,.9)', 'rgba(255,140,190,.8)', 'rgba(255,220,230,.7)', 'rgba(255,100,160,.8)'][Math.floor(Math.random() * 4)]
      });
    }
  }

  function dismissAllCards() {
    for (const c of _cards) {
      if (c.phase === 'display') { c.phase = 'return'; c.startT = t; }
    }
  }

  function updCard() {
    for (let i = _cards.length - 1; i >= 0; i--) {
      const c = _cards[i];
      const im = c.img;
      const el = t - c.startT;

      if (c.phase === 'lift') {
        const p = Math.min(1, el / .3);
        const ease = 1 - Math.pow(1 - p, 3);
        im.tsc = 1 + ease * .8;
        im.tr = ease * -5;
        if (p >= 1) { c.phase = 'move'; c.startT = t; }
      }
      else if (c.phase === 'move') {
        const p = Math.min(1, el / .5);
        const ease = 1 - Math.pow(1 - p, 3);
        im.tx = im._cardSX + (im._cardEX - im._cardSX) * ease;
        im.ty = im._cardSY + (im._cardEY - im._cardSY) * ease;
        im.tsc = 1.8;
        im.tr = -5 + ease * 5;
        if (p >= 1) {
          c.phase = 'display'; c.startT = t;
          c.ex = im._cardEX; c.ey = im._cardEY;
          // 落地闪光
          mkSparkle(c.ex, c.ey);
        }
      }
      else if (c.phase === 'display') {
        const fp = c.fp;
        im.tx = c.ex + Math.sin(t * .8 + fp) * 8;
        im.ty = c.ey + Math.cos(t * .6 + fp) * 5;
        im.tsc = 1.8;
        im.tr = Math.sin(t * .5 + fp) * 2;
      }
      else if (c.phase === 'return') {
        const p = Math.min(1, el / .4);
        const ease = 1 - Math.pow(1 - p, 3);
        im.tx = c.ex + (im._cardSX - c.ex) * ease;
        im.ty = c.ey + (im._cardSY - c.ey) * ease;
        im.tsc = 1.8 - ease * .8;
        im.tr = (1 - ease) * -3;
        if (p >= 1) {
          im.tsc = 1; im.tr = 0;
          _cards.splice(i, 1);
        }
      }
    }
  }

  function drawCard() {
    for (const c of _cards) {
      const im = c.img;
      if (!im.loaded || !im.imgObj) continue;
      const s = iSz() * Math.max(im.sc, 1.6), h = s * (im.height / im.width || 1);
      ctx.save();
      ctx.globalAlpha = 1;
      ctx.translate(im.cx, im.cy);
      ctx.rotate(im.ro * Math.PI / 180);
      // 柔和光晕
      ctx.shadowColor = 'rgba(255,140,190,.5)'; ctx.shadowBlur = 35;
      ctx.fillStyle = 'rgba(255,200,220,.05)';
      rr(ctx, -s / 2 - 6, -h / 2 - 6, s + 12, h + 12, 12); ctx.fill();
      // 渐变边框
      ctx.strokeStyle = 'rgba(255,150,200,.45)'; ctx.lineWidth = 1.5;
      rr(ctx, -s / 2 - 3, -h / 2 - 3, s + 6, h + 6, 10); ctx.stroke();
      // 图片
      ctx.shadowBlur = 0;
      rr(ctx, -s / 2, -h / 2, s, h, 8); ctx.clip();
      ctx.drawImage(im.imgObj, -s / 2, -h / 2, s, h);
      ctx.restore();
    }
  }

  function zoom(img) {
    if (enlarged) return;
    enlarged = { img, state: 'fading', particles: [], convT: 0 };
    for (const i of images) i._sa = i.ta;

    // 加载原图
    if (!img.fullImg) {
      const fm = new Image();
      fm.onload = () => { img.fullImg = fm; img.fullW = fm.naturalWidth; img.fullH = fm.naturalHeight; doZoom(img); };
      fm.onerror = () => { img.fullImg = img.imgObj; img.fullW = img.width; img.fullH = img.height; doZoom(img); };
      fm.src = img.url;
    } else {
      doZoom(img);
    }
  }

  function doZoom(img) {
    // 生成粒子
    mkParts(img);
    // 淡出其他图片
    for (const i of images) i.ta = 0;
  }

  function mkParts(img) {
    const src = img.fullImg || img.imgObj;
    if (!src) { enlarged.state = 'done'; return; }
    const tc = document.createElement('canvas');
    const maxD = Math.min(400, Math.max(W, H) * .55);
    const scl = maxD / Math.max(src.naturalWidth || src.width, src.naturalHeight || img.height);
    tc.width = Math.floor((src.naturalWidth || src.width) * scl);
    tc.height = Math.floor((src.naturalHeight || src.height) * scl);
    if (tc.width < 1 || tc.height < 1) { enlarged.state = 'done'; return; }
    const tcx = tc.getContext('2d');
    tcx.drawImage(src, 0, 0, tc.width, tc.height);
    let px;
    try { px = tcx.getImageData(0, 0, tc.width, tc.height).data; } catch (e) { enlarged.state = 'done'; return; }
    const intv = Math.max(4, Math.floor(Math.min(tc.width, tc.height) / 50));
    const parts = [];
    const cx = W / 2, cy = H / 2;
    for (let y = 0; y < tc.height; y += intv) {
      for (let x = 0; x < tc.width; x += intv) {
        const idx = (y * tc.width + x) * 4;
        const r = px[idx], g = px[idx + 1], b = px[idx + 2], a = px[idx + 3];
        if (a < 30) continue;
        const ox = cx - tc.width / 2 + x;
        const oy = cy - tc.height / 2 + y;
        // 起始位置：从目标点沿方向向外散开
        const ang = Math.atan2(y - tc.height / 2, x - tc.width / 2) + (Math.random() - .5) * 1.5;
        const dist = 200 + Math.random() * 300;
        // 延迟：中心先到，边缘后到
        const dx = (x - tc.width / 2) / (tc.width / 2);
        const dy = (y - tc.height / 2) / (tc.height / 2);
        const delay = Math.sqrt(dx * dx + dy * dy) * .4 + Math.random() * .1;
        parts.push({
          ox, oy,
          sx: ox + Math.cos(ang) * dist,
          sy: oy + Math.sin(ang) * dist,
          x: ox + Math.cos(ang) * dist,
          y: oy + Math.sin(ang) * dist,
          r, g, b, a, sz: intv * .42,
          delay
        });
      }
    }
    enlarged.particles = parts;
    enlarged.state = 'particles';
    enlarged.convT = t;
  }

  /* ========== 关闭放大 ========== */
  function dismiss() {
    if (!enlarged) return;
    const img = enlarged.img;

    // 如果有粒子，先散开
    if (enlarged.particles && enlarged.particles.length && (enlarged.state === 'particles' || enlarged.state === 'done' || enlarged.state === 'fading')) {
      enlarged.state = 'scatter';
      const cx = W / 2, cy = H / 2;
      for (const p of enlarged.particles) {
        const ang = Math.atan2(p.y - cy, p.x - cx) + (Math.random() - .5) * .8;
        const spd = 15 + Math.random() * 20;
        p.vx = Math.cos(ang) * spd;
        p.vy = Math.sin(ang) * spd;
      }
      enlarged.scatT = t;
      function scat() {
        if (!enlarged || enlarged.state !== 'scatter') return;
        const el = t - enlarged.scatT;
        for (const p of enlarged.particles) {
          p.x += p.vx; p.y += p.vy;
          p.vx *= .96; p.vy *= .96;
        }
        if (el > .5) {
          finDismiss(img); return;
        }
        requestAnimationFrame(scat);
      }
      scat();
    } else {
      finDismiss(img);
    }
  }

  function finDismiss(img) {
    img.ta = 1; img.tsc = 1;
    img.cx = img.tx; img.cy = img.ty;
    for (const i of images) i.ta = i._sa || 1;
    enlarged = null;
    reposition();
  }

  /* ========== 3D 模式 ========== */
  function loadScript(src) {
    return new Promise((ok, no) => {
      const s = document.createElement('script');
      s.src = src; s.onload = ok; s.onerror = no;
      document.head.appendChild(s);
    });
  }

  async function init3d() {
    if (s3d) {
      s3d.active = true;
      s3d.clock = new THREE.Clock();
      (function reLoop() {
        if (!s3d || !s3d.active || mode !== 'scene3d') return;
        requestAnimationFrame(reLoop);
        s3d.renderFrame();
      })();
      return;
    }
    const c3d = document.getElementById('canvas3d');
    c3d.width = innerWidth; c3d.height = innerHeight;

    // 加载 Three.js (UMD)
    if (typeof THREE === 'undefined') {
      await loadScript('https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js');
    }
    if (typeof THREE === 'undefined') {
      // 备用 CDN
      try { await loadScript('https://cdn.jsdelivr.net/npm/three@0.128.0/build/three.min.js'); } catch(e) {}
    }
    if (typeof THREE === 'undefined') {
      console.error('Three.js 加载失败');
      return;
    }

    // --- 场景 ---
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x050510);
    scene.fog = new THREE.FogExp2(0x050510, 0.015);

    const camera = new THREE.PerspectiveCamera(50, innerWidth / innerHeight, 0.1, 200);
    camera.position.set(0, 4, 30);

    const renderer = new THREE.WebGLRenderer({ canvas: c3d, antialias: true });
    renderer.setSize(innerWidth, innerHeight);
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.toneMapping = THREE.ACESFilmicToneMapping;

    // --- 灯光 ---
    scene.add(new THREE.AmbientLight(0x1a0a20, 1));
    const pl1 = new THREE.PointLight(0xff6090, 5, 60); pl1.position.set(0, 8, 5); scene.add(pl1);
    const pl2 = new THREE.PointLight(0xff3070, 3, 40); pl2.position.set(-10, 5, -5); scene.add(pl2);
    const pl3 = new THREE.PointLight(0xff80a0, 3, 40); pl3.position.set(10, 3, -8); scene.add(pl3);
    const pl4 = new THREE.PointLight(0xff4070, 4, 25); pl4.position.set(0, 2, 4); scene.add(pl4);

    // --- 玻璃爱心着色器 ---
    const glassVS = `
      varying vec3 vN; varying vec3 vV; varying vec3 vW;
      void main(){
        vN = normalize(normalMatrix * normal);
        vec4 w = modelMatrix * vec4(position,1.0);
        vW = w.xyz;
        vV = normalize(cameraPosition - w.xyz);
        gl_Position = projectionMatrix * viewMatrix * w;
      }`;
    const glassFS = `
      uniform float uTime; uniform vec3 uCol; uniform float uAlpha;
      varying vec3 vN; varying vec3 vV; varying vec3 vW;
      void main(){
        vec3 n = normalize(vN); vec3 v = normalize(vV);
        float fr = pow(1.0 - max(dot(n,v),0.0), 2.5);
        float ph = pow(max(dot(reflect(vec3(1,2,1),n),v),0.0), 32.0);
        vec3 c = uCol * fr * 1.2 + vec3(1) * ph * 1.0;
        c += uCol * pow(1.0-max(dot(n,v),0.0),4.0) * 0.5;
        c.r += fr * 0.1;
        float pulse = sin(uTime*2.0) * 0.15 + 0.85;
        c *= pulse;
        gl_FragColor = vec4(c, mix(uAlpha,1.0,fr));
      }`;

    // --- 3D爱心几何体 ---
    function mkHeartGeo(sc) {
      const sh = new THREE.Shape();
      sh.moveTo(0, 0);
      sh.bezierCurveTo(0, -3*sc, -5*sc, -3*sc, -5*sc, 0);
      sh.bezierCurveTo(-5*sc, 3*sc, 0, 6*sc, 0, 9*sc);
      sh.bezierCurveTo(0, 6*sc, 5*sc, 3*sc, 5*sc, 0);
      sh.bezierCurveTo(5*sc, -3*sc, 0, -3*sc, 0, 0);
      const g = new THREE.ExtrudeGeometry(sh, {
        depth: 2.5*sc, bevelEnabled: true, bevelSegments: 6,
        steps: 2, bevelSize: .8*sc, bevelThickness: .8*sc, curveSegments: 24
      });
      g.center();
      return g;
    }

    // 主爱心（大）
    const heartMat = new THREE.ShaderMaterial({
      uniforms: { uTime:{value:0}, uCol:{value:new THREE.Color(0xff4070)}, uAlpha:{value:.35} },
      vertexShader: glassVS, fragmentShader: glassFS,
      transparent: true, side: THREE.DoubleSide, depthWrite: false, blending: THREE.AdditiveBlending
    });
    const mainHeart = new THREE.Mesh(mkHeartGeo(.8), heartMat);
    mainHeart.position.set(0, 2, 2);
    mainHeart.rotation.z = Math.PI;
    scene.add(mainHeart);

    // 爱心边框
    const heartEdge = new THREE.LineSegments(
      new THREE.EdgesGeometry(mkHeartGeo(.8)),
      new THREE.LineBasicMaterial({ color: 0xff6090, transparent: true, opacity: .6 })
    );
    heartEdge.position.copy(mainHeart.position);
    heartEdge.rotation.copy(mainHeart.rotation);
    scene.add(heartEdge);

    // 小爱心群
    const smallHearts = [];
    const hColors = [0xff4070, 0xff6090, 0xff80b0, 0xcc4080, 0xff50a0];
    for (let i = 0; i < 12; i++) {
      const m = new THREE.ShaderMaterial({
        uniforms: { uTime:{value:0}, uCol:{value:new THREE.Color(hColors[i%hColors.length])}, uAlpha:{value:.25} },
        vertexShader: glassVS, fragmentShader: glassFS,
        transparent: true, side: THREE.DoubleSide, depthWrite: false, blending: THREE.AdditiveBlending
      });
      const geo = mkHeartGeo(.12 + Math.random() * .15);
      const mesh = new THREE.Mesh(geo, m);
      const a = (i / 12) * Math.PI * 2;
      mesh.position.set(Math.cos(a)*10, 2+Math.random()*4, Math.sin(a)*10 - 5);
      mesh.rotation.z = Math.PI;
      scene.add(mesh);
      const e = new THREE.LineSegments(new THREE.EdgesGeometry(geo),
        new THREE.LineBasicMaterial({color:hColors[i%hColors.length], transparent:true, opacity:.4}));
      e.position.copy(mesh.position); e.rotation.copy(mesh.rotation);
      scene.add(e);
      smallHearts.push({ mesh, edge: e, baseY: mesh.position.y, ph: Math.random()*6.28 });
    }

    // ===== 旋转木马效果 =====

    // ===== 旋转木马效果 =====
    // 照片像竖直挡板，宽边连接旋转柱，绕柱旋转
    const texLoader = new THREE.TextureLoader();
    const CAROUSEL_COUNT = 16;      // 旋转木马数量
    const PANELS_PER = 6;           // 每个木马上照片数
    const CAROUSEL_R = 2;           // 挡板到旋转柱距离
    const CAROUSEL_CIRCLE_R = 20;   // 木马排列半径
    const allImgs = modeImages();
    const totalCar = CAROUSEL_COUNT * PANELS_PER;
    const carImgs = allImgs.slice(0, Math.min(allImgs.length, totalCar));
    const carousels = [];
    const carouselRing = new THREE.Group(); // 公转组
    scene.add(carouselRing);

    for (let c = 0; c < CAROUSEL_COUNT; c++) {
      const group = new THREE.Group();
      const ang = (c / CAROUSEL_COUNT) * Math.PI * 2;
      group.position.set(Math.cos(ang) * CAROUSEL_CIRCLE_R, 6, Math.sin(ang) * CAROUSEL_CIRCLE_R);

      // 中心轴
      const axleMat = new THREE.MeshBasicMaterial({ color: 0xff80a0, transparent: true, opacity: 0.4 });
      group.add(new THREE.Mesh(new THREE.CylinderGeometry(0.08, 0.08, 9, 8), axleMat));

      // 上下轴承环
      const ringMat = new THREE.MeshBasicMaterial({ color: 0xff6090, transparent: true, opacity: 0.5 });
      const ring1 = new THREE.Mesh(new THREE.TorusGeometry(CAROUSEL_R + 0.3, 0.1, 8, 48), ringMat);
      ring1.rotation.x = Math.PI / 2; ring1.position.y = 4;
      group.add(ring1);
      const ring2 = ring1.clone(); ring2.position.y = -1;
      group.add(ring2);

      // 照片挡板
      const startI = c * PANELS_PER;
      const endI = Math.min(startI + PANELS_PER, carImgs.length);
      const cnt = endI - startI;

      for (let j = startI; j < endI; j++) {
        const img = carImgs[j];
        let tex;
        try {
          tex = await new Promise((ok, no) => {
            texLoader.load(img.thumb || img.url, ok, undefined, no);
          });
        } catch(e) { continue; }
        if (tex.colorSpace !== undefined) tex.colorSpace = THREE.SRGBColorSpace;
        else if (tex.encoding !== undefined) tex.encoding = THREE.sRGBEncoding;

        const asp = (img.width || 1) / (img.height || 1);
        const h = 1.8, w = h * Math.min(Math.max(asp, 0.5), 1.5);

        // 挡板：照片平面
        const geo = new THREE.PlaneGeometry(w, h);
        const mat = new THREE.MeshBasicMaterial({ map: tex, transparent: true, opacity: .88, side: THREE.DoubleSide });
        const mesh = new THREE.Mesh(geo, mat);

        const fa = ((j - startI) / cnt) * Math.PI * 2;
        // 照片中心在圆周上，面朝外（切线方向）
        mesh.position.set(Math.cos(fa) * CAROUSEL_R, 1.5, Math.sin(fa) * CAROUSEL_R);
        mesh.rotation.y = -fa + Math.PI / 2;
        group.add(mesh);

        // 连接杆：从中心到照片宽边中点
        const rodLen = CAROUSEL_R;
        const rodGeo = new THREE.CylinderGeometry(0.03, 0.03, rodLen, 4);
        const rodMat = new THREE.MeshBasicMaterial({ color: 0xff6090, transparent: true, opacity: 0.3 });
        const rod = new THREE.Mesh(rodGeo, rodMat);
        rod.position.set(Math.cos(fa) * CAROUSEL_R * 0.5, 1.5, Math.sin(fa) * CAROUSEL_R * 0.5);
        rod.rotation.z = Math.PI / 2;
        rod.rotation.y = -fa;
        group.add(rod);

        // 边框
        const border = new THREE.LineSegments(
          new THREE.EdgesGeometry(geo),
          new THREE.LineBasicMaterial({ color: 0xff80a0, transparent: true, opacity: .45 })
        );
        border.position.copy(mesh.position);
        border.rotation.copy(mesh.rotation);
        group.add(border);
      }

      carouselRing.add(group);
      carousels.push({ group });
    }

    // --- 粒子系统 ---
    const PC = 15000;
    const pGeo = new THREE.BufferGeometry();
    const pPos = new Float32Array(PC * 3);
    const pRnd = new Float32Array(PC);
    for (let i = 0; i < PC; i++) {
      pPos[i*3] = (Math.random()-.5)*60;
      pPos[i*3+1] = Math.random()*30 - 5;
      pPos[i*3+2] = (Math.random()-.5)*60 - 10;
      pRnd[i] = Math.random();
    }
    pGeo.setAttribute('position', new THREE.BufferAttribute(pPos, 3));
    pGeo.setAttribute('aRnd', new THREE.BufferAttribute(pRnd, 1));
    const pMat = new THREE.ShaderMaterial({
      uniforms: { uTime:{value:0}, uSz:{value:4} },
      vertexShader: `
        uniform float uTime; uniform float uSz;
        attribute float aRnd; varying float vA;
        void main(){
          vec3 p = position;
          p.x += sin(uTime*.3 + position.z*.5 + aRnd*10.) * 2.;
          p.y += sin(uTime*.2 + position.x*.3 + aRnd*5.) * 1.5;
          p.z += cos(uTime*.25 + position.y*.4 + aRnd*7.) * 2.;
          vec4 mv = modelViewMatrix * vec4(p,1.);
          gl_PointSize = uSz * (60./-mv.z) * (.5+aRnd*.5);
          gl_Position = projectionMatrix * mv;
          vA = smoothstep(50.,10.,length(mv.xyz)) * (.3+aRnd*.4);
        }`,
      fragmentShader: `
        varying float vA;
        void main(){
          float d = length(gl_PointCoord-.5)*2.;
          float a = smoothstep(1.,0.,d)*vA;
          gl_FragColor = vec4(1.,.4,.6,a);
        }`,
      transparent: true, depthWrite: false, blending: THREE.AdditiveBlending
    });
    scene.add(new THREE.Points(pGeo, pMat));

    // --- 网格地面 ---
    const floorMat = new THREE.ShaderMaterial({
      uniforms: { uTime:{value:0} },
      vertexShader: `
        varying vec2 vUv; varying vec3 vW;
        void main(){ vUv=uv; vec4 w=modelMatrix*vec4(position,1.); vW=w.xyz; gl_Position=projectionMatrix*viewMatrix*w; }`,
      fragmentShader: `
        uniform float uTime; varying vec2 vUv; varying vec3 vW;
        void main(){
          vec2 g = abs(fract(vW.xz*.5)-.5);
          float l = min(g.x,g.y);
          float line = 1.-smoothstep(0.,.03,l);
          float dist = length(vW.xz);
          float fade = smoothstep(50.,5.,dist);
          float wave = sin(dist*.5-uTime*.8)*.5+.5;
          vec3 c = vec3(.04,.02,.06) + vec3(1,.4,.6)*line*.25*fade + vec3(1,.4,.6)*exp(-dist*.04)*.35;
          c += vec3(1,.4,.6)*wave*.03*fade;
          gl_FragColor = vec4(c, fade*.8);
        }`,
      transparent: true, side: THREE.DoubleSide
    });
    const floor = new THREE.Mesh(new THREE.PlaneGeometry(100,100), floorMat);
    floor.rotation.x = -Math.PI/2; floor.position.y = -3;
    scene.add(floor);

    // --- 鼠标 + 触摸 ---
    let m3x = 0, m3y = 0, sm3x = 0, sm3y = 0;
    let zoomTarget = 30, zoomCur = 30;
    function onMM(e) {
      m3x = (e.clientX / innerWidth - .5) * 2;
      m3y = (e.clientY / innerHeight - .5) * 2;
    }
    function onWheel(e) {
      e.preventDefault();
      zoomTarget = Math.max(8, Math.min(80, zoomTarget + e.deltaY * .05));
    }
    // 触摸视差
    function on3DTouch(e) {
      if (e.touches.length === 1) {
        const p = e.touches[0];
        m3x = (p.clientX / innerWidth - .5) * 2;
        m3y = (p.clientY / innerHeight - .5) * 2;
      }
    }
    // 双指缩放
    let _pinchDist = 0;
    function on3DPinch(e) {
      if (e.touches.length === 2) {
        const dx = e.touches[0].clientX - e.touches[1].clientX;
        const dy = e.touches[0].clientY - e.touches[1].clientY;
        const dist = Math.sqrt(dx * dx + dy * dy);
        if (_pinchDist > 0) {
          const delta = (_pinchDist - dist) * .15;
          zoomTarget = Math.max(8, Math.min(80, zoomTarget + delta));
        }
        _pinchDist = dist;
      } else {
        _pinchDist = 0;
      }
    }
    window.addEventListener('mousemove', onMM);
    const c3dEl = document.getElementById('canvas3d');
    c3dEl.addEventListener('wheel', onWheel, { passive: false });
    c3dEl.addEventListener('touchmove', e => { on3DTouch(e); on3DPinch(e); }, { passive: true });
    c3dEl.addEventListener('touchend', () => { _pinchDist = 0; });

    // --- 渲染循环 ---
    const clock = new THREE.Clock();
    s3d = { active: true, scene, camera, renderer, heartMat, mainHeart, heartEdge,
            smallHearts, carouselRing, carousels, pMat, floorMat, pl1, clock, m3x:0, m3y:0, sm3x:0, sm3y:0 };

    // --- 渲染帧函数（可重入）---
    function render3dFrame() {
      const el = s3d.clock.getElapsedTime();
      s3d.sm3x += (m3x - s3d.sm3x) * .12;
      s3d.sm3y += (m3y - s3d.sm3y) * .12;

      camera.position.x += (s3d.sm3x * 10 - camera.position.x) * .1;
      camera.position.y += (4 - s3d.sm3y * 6 - camera.position.y) * .1;
      zoomCur += (zoomTarget - zoomCur) * .08;
      camera.position.z = zoomCur;
      camera.position.x += Math.sin(el*.2) * .02;
      camera.lookAt(0, 2, 2);

      // 爱心动画
      const pulse = 1 + Math.sin(el*1.5) * .12;
      mainHeart.scale.setScalar(pulse);
      mainHeart.rotation.y = el * .5;
      heartEdge.scale.copy(mainHeart.scale);
      heartEdge.rotation.copy(mainHeart.rotation);

      smallHearts.forEach((h,i) => {
        h.mesh.rotation.y += .005 + i*.001;
        h.mesh.position.y = h.baseY + Math.sin(el*.6+h.ph) * .8;
        h.edge.rotation.copy(h.mesh.rotation);
        h.edge.position.copy(h.mesh.position);
        h.mesh.material.uniforms.uTime.value = el;
      });
      heartMat.uniforms.uTime.value = el;

      // 旋转木马自转（交替方向）+ 整体公转
      s3d.carouselRing.rotation.y = el * 0.15;
      s3d.carouselRing.position.y = Math.sin(el * 0.3) * 1.5;
      s3d.carousels.forEach((car, ci) => {
        const dir = ci % 2 === 0 ? 1 : -1;
        car.group.rotation.y += 0.005 * dir;
      });

      pMat.uniforms.uTime.value = el;
      floorMat.uniforms.uTime.value = el;
      pl1.position.x = Math.sin(el*.3) * 8;
      pl1.intensity = 5 + Math.sin(el*.5);
      pl4.intensity = 4 + Math.sin(el*1.5) * 2;

      renderer.render(scene, camera);
    }

    // 保存渲染函数到 s3d
    s3d.renderFrame = render3dFrame;

    // 启动渲染循环
    (function loop3d() {
      if (!s3d || !s3d.active || mode !== 'scene3d') return;
      requestAnimationFrame(loop3d);
      render3dFrame();
    })();

    // 窗口大小
    window.addEventListener('resize', () => {
      camera.aspect = innerWidth / innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(innerWidth, innerHeight);
    });
  }

})();
