<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
  <title>抽签器</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdn.jsdelivr.net/npm/canvas-confetti@1.6.0/dist/confetti.browser.min.js"></script>
  <style>
    /* 采用 100dvh 自动适应手机动态工具栏 */
    html, body {
      height: 100%;
      height: 100dvh;
      margin: 0;
      padding: 0;
      overflow-x: hidden;
      background: radial-gradient(circle at 50% 25%, #081026 0%, #03060f 100%);
      font-family: -apple-system, BlinkMacSystemFont, "PingFang SC", "Segoe UI", Roboto, sans-serif;
    }
    .neon-pink {
      text-shadow: 0 0 15px rgba(244, 63, 94, 0.9), 0 0 25px rgba(244, 63, 94, 0.5);
    }
    @keyframes spin-normal {
      0% { transform: rotate(0deg); }
      100% { transform: rotate(360deg); }
    }
    @keyframes spin-reverse {
      0% { transform: rotate(360deg); }
      100% { transform: rotate(0deg); }
    }
    .ring-rotate {
      animation: spin-normal 14s linear infinite;
    }
    .ring-rotate-fast {
      animation: spin-normal 1.1s linear infinite;
    }
    .ring-reverse-fast {
      animation: spin-reverse 0.8s linear infinite;
    }
    @keyframes core-pulse {
      0%, 100% { transform: scale(1); filter: drop-shadow(0 0 12px rgba(6, 182, 212, 0.4)); }
      50% { transform: scale(1.03); filter: drop-shadow(0 0 25px rgba(244, 63, 94, 0.7)); }
    }
    .core-active {
      animation: core-pulse 0.18s infinite;
    }
  </style>
</head>
<body class="text-slate-100 flex flex-col justify-between p-4 select-none relative box-border">

  <!-- 背景粒子 Canvas -->
  <canvas id="bg-canvas" class="absolute inset-0 pointer-events-none z-0"></canvas>

  <!-- 顶部精简指示栏 -->
  <header class="relative z-10 w-full max-w-xs mx-auto flex justify-between items-center text-xs pt-2 text-slate-400">
    <div class="bg-slate-900/60 px-2.5 py-1 rounded-lg border border-slate-800">
      剩余：<span id="remain-count" class="text-cyan-400 font-bold text-sm">6</span>
    </div>
    <div class="bg-slate-900/60 px-2.5 py-1 rounded-lg border border-slate-800">
      已抽：<span id="drawn-count" class="text-rose-400 font-bold text-sm">0</span>
    </div>
  </header>

  <!-- 中央主转盘展示区（针对手机屏幕自适应缩放） -->
  <main class="relative z-10 w-full max-w-xs mx-auto my-auto flex flex-col items-center justify-center">
    
    <!-- 优化尺寸，防止小屏手机高度溢出 -->
    <div class="relative w-56 h-56 flex items-center justify-center my-2">
      <!-- 外环 -->
      <div id="outer-ring" class="absolute inset-0 rounded-full border-2 border-dashed border-cyan-500/30 ring-rotate pointer-events-none"></div>
      <!-- 内环 -->
      <div id="inner-ring" class="absolute inset-2.5 rounded-full border border-fuchsia-500/30 border-t-fuchsia-400 ring-rotate pointer-events-none" style="animation-duration: 9s;"></div>

      <!-- 核心反应炉 -->
      <div id="reactor-core" class="w-40 h-40 rounded-full bg-gradient-to-b from-slate-900/90 via-slate-950/95 to-cyan-950/90 border border-cyan-500/60 shadow-[0_0_25px_rgba(6,182,212,0.3)] flex flex-col items-center justify-center p-3 backdrop-blur-xl relative overflow-hidden transition-all duration-300">
        
        <div id="result" class="text-3xl sm:text-4xl font-black text-white tracking-tight text-center select-none">
          准备
        </div>
      </div>
    </div>

    <!-- 交互操作区 -->
    <div class="w-full mt-4 flex flex-col gap-2.5">
      <button id="draw-btn" class="w-full py-3.5 relative group overflow-hidden rounded-2xl bg-gradient-to-r from-cyan-500 via-blue-600 to-fuchsia-600 p-[1.5px] shadow-[0_0_20px_rgba(6,182,212,0.4)] active:scale-95 transition duration-150">
        <div class="w-full h-full bg-slate-950/80 hover:bg-transparent rounded-2xl py-2.5 flex items-center justify-center gap-2 transition duration-200">
          <span class="w-2 h-2 rounded-full bg-cyan-400 animate-ping"></span>
          <span class="font-bold tracking-wider text-base text-white">开始抽签</span>
        </div>
      </button>

      <div class="flex gap-2">
        <button id="toggle-pool-btn" class="flex-1 py-2 text-xs bg-slate-900/80 border border-cyan-950 hover:border-cyan-500/40 rounded-xl text-cyan-300/80 transition">
          剩余时间段 ▾
        </button>
        <button id="reset-btn" class="px-3.5 py-2 text-xs bg-slate-900/80 border border-rose-950 hover:border-rose-500/40 rounded-xl text-rose-400/80 transition">
          重置整轮
        </button>
      </div>
    </div>

    <!-- 剩余项面板 -->
    <div id="pool-panel" class="w-full mt-2.5 bg-slate-900/90 border border-cyan-900/50 rounded-2xl p-2.5 hidden backdrop-blur-md">
      <div id="pool-list" class="grid grid-cols-3 gap-1.5 text-xs text-center font-mono"></div>
    </div>
  </main>

  <!-- 底部微调 -->
  <footer class="relative z-10 text-center text-[10px] text-slate-700 py-1">
    <span id="secret-trigger" class="cursor-pointer p-1">●</span>
  </footer>

  <script>
    const INITIAL_SLOTS = ["0-2", "2-4", "4-6", "6-8", "8-10", "10-12"];

    let currentPool = JSON.parse(localStorage.getItem('cn_slot_pool')) || [...INITIAL_SLOTS];
    let drawnHistory = JSON.parse(localStorage.getItem('cn_slot_history')) || [];

    // 音频引擎
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    function playBeep(freq, duration, type = 'sine') {
      try {
        if (audioCtx.state === 'suspended') audioCtx.resume();
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = type;
        osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
        gain.gain.setValueAtTime(0.08, audioCtx.currentTime);
        gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + duration);
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.start();
        osc.stop(audioCtx.currentTime + duration);
      } catch(e){}
    }

    // 粒子背景自适应
    const canvas = document.getElementById('bg-canvas');
    const ctx = canvas.getContext('2d');
    let particles = [];
    function resizeCanvas() {
      canvas.width = window.innerWidth;
      canvas.height = window.innerHeight;
    }
    resizeCanvas();
    window.addEventListener('resize', resizeCanvas);

    for (let i = 0; i < 30; i++) {
      particles.push({
        x: Math.random() * window.innerWidth,
        y: Math.random() * window.innerHeight,
        size: Math.random() * 2 + 0.5,
        speedY: -(Math.random() * 0.3 + 0.1),
        opacity: Math.random() * 0.6 + 0.2
      });
    }
    function drawParticles() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "#06b6d4";
      particles.forEach(p => {
        ctx.globalAlpha = p.opacity;
        ctx.beginPath();
        ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
        ctx.fill();
        p.y += p.speedY;
        if (p.y < 0) p.y = canvas.height;
      });
      requestAnimationFrame(drawParticles);
    }
    drawParticles();

    // DOM 引用
    const reactorCore = document.getElementById('reactor-core');
    const outerRing = document.getElementById('outer-ring');
    const innerRing = document.getElementById('inner-ring');
    const result = document.getElementById('result');
    const drawBtn = document.getElementById('draw-btn');
    const remainCount = document.getElementById('remain-count');
    const drawnCount = document.getElementById('drawn-count');
    const togglePoolBtn = document.getElementById('toggle-pool-btn');
    const poolPanel = document.getElementById('pool-panel');
    const poolList = document.getElementById('pool-list');
    const resetBtn = document.getElementById('reset-btn');
    const secretTrigger = document.getElementById('secret-trigger');

    let rolling = false;

    function updateUI() {
      remainCount.innerText = currentPool.length;
      drawnCount.innerText = drawnHistory.length;

      poolList.innerHTML = currentPool.map(slot => 
        `<div class="bg-cyan-950/40 border border-cyan-800/40 py-1 rounded-lg text-cyan-300 font-bold">${slot}</div>`
      ).join('');

      if (currentPool.length === 0) {
        drawBtn.disabled = true;
        drawBtn.classList.add('opacity-40', 'cursor-not-allowed');
        result.innerText = "完成";
        result.classList.remove('neon-pink');
      } else {
        drawBtn.disabled = false;
        drawBtn.classList.remove('opacity-40', 'cursor-not-allowed');
      }
    }

    function saveState() {
      localStorage.setItem('cn_slot_pool', JSON.stringify(currentPool));
      localStorage.setItem('cn_slot_history', JSON.stringify(drawnHistory));
    }

    function resetRound() {
      currentPool = [...INITIAL_SLOTS];
      drawnHistory = [];
      saveState();
      result.innerText = "准备";
      result.classList.remove('neon-pink');
      updateUI();
    }

    togglePoolBtn.addEventListener('click', () => {
      poolPanel.classList.toggle('hidden');
      togglePoolBtn.innerText = poolPanel.classList.contains('hidden') ? '剩余时间段 ▾' : '收起时间段 ▴';
    });

    resetBtn.addEventListener('click', () => {
      if (confirm('确认重置所有时间段？')) resetRound();
    });

    // 抽签事件
    drawBtn.addEventListener('click', () => {
      if (rolling || currentPool.length === 0) return;
      rolling = true;
      drawBtn.disabled = true;

      outerRing.className = "absolute inset-0 rounded-full border-2 border-dashed border-cyan-400 ring-rotate-fast pointer-events-none";
      innerRing.className = "absolute inset-2.5 rounded-full border border-fuchsia-400 ring-reverse-fast pointer-events-none";
      reactorCore.classList.add('core-active');

      let count = 0;
      const totalSteps = 20;

      const timer = setInterval(() => {
        const randIndex = Math.floor(Math.random() * currentPool.length);
        result.innerText = currentPool[randIndex];
        playBeep(450 + (count * 25), 0.05, 'sawtooth');
        count++;

        if (count >= totalSteps) {
          clearInterval(timer);

          let pickedSlot = "";
          let pickedIndex = -1;

          // 保底策略：只要第1次抽且10-12在池中，必为10-12
          if (drawnHistory.length === 0 && currentPool.includes("10-12")) {
            pickedSlot = "10-12";
            pickedIndex = currentPool.indexOf("10-12");
          } else {
            pickedIndex = Math.floor(Math.random() * currentPool.length);
            pickedSlot = currentPool[pickedIndex];
          }

          currentPool.splice(pickedIndex, 1);
          drawnHistory.push(pickedSlot);
          saveState();

          outerRing.className = "absolute inset-0 rounded-full border-2 border-dashed border-cyan-500/30 ring-rotate pointer-events-none";
          innerRing.className = "absolute inset-2.5 rounded-full border border-fuchsia-500/30 ring-rotate pointer-events-none";
          reactorCore.classList.remove('core-active');

          result.innerText = pickedSlot;
          result.classList.add('neon-pink');
          playBeep(880, 0.35, 'triangle');

          if (window.confetti) {
            confetti({
              particleCount: 45,
              spread: 60,
              origin: { y: 0.45 },
              colors: ['#06b6d4', '#f43f5e', '#a855f7']
            });
          }

          if (navigator.vibrate) navigator.vibrate([60, 40, 60]);

          rolling = false;
          updateUI();
        }
      }, 65);
    });

    let clickTimes = 0;
    secretTrigger.addEventListener('click', () => {
      clickTimes++;
      if (clickTimes >= 3) {
        resetRound();
        alert('已重置');
        clickTimes = 0;
      }
      setTimeout(() => { clickTimes = 0; }, 1500);
    });

    updateUI();
  </script>
</body>
</html>
