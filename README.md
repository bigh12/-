<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
  <title>时间段抽签系统</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <!-- 粒子礼花组件 -->
  <script src="https://cdn.jsdelivr.net/npm/canvas-confetti@1.6.0/dist/confetti.browser.min.js"></script>
  <style>
    body {
      background: radial-gradient(circle at 50% 20%, #081026 0%, #03060f 100%);
      font-family: -apple-system, BlinkMacSystemFont, "PingFang SC", "Segoe UI", Roboto, "Microsoft YaHei", sans-serif;
    }

    /* 赛博霓虹字体发光 */
    .neon-cyan {
      text-shadow: 0 0 10px rgba(6, 182, 212, 0.9), 0 0 20px rgba(6, 182, 212, 0.5), 0 0 35px rgba(6, 182, 212, 0.3);
    }
    .neon-pink {
      text-shadow: 0 0 12px rgba(244, 63, 94, 0.95), 0 0 25px rgba(244, 63, 94, 0.6), 0 0 40px rgba(244, 63, 94, 0.3);
    }

    /* 旋转能量外环动画 */
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

    /* 能量共振脉冲 */
    @keyframes core-pulse {
      0%, 100% { transform: scale(1); filter: drop-shadow(0 0 15px rgba(6, 182, 212, 0.4)); }
      50% { transform: scale(1.03); filter: drop-shadow(0 0 30px rgba(244, 63, 94, 0.7)); }
    }
    .core-active {
      animation: core-pulse 0.18s infinite;
    }
  </style>
</head>
<body class="text-slate-100 min-h-screen flex flex-col justify-between p-4 sm:p-6 select-none overflow-x-hidden relative">

  <!-- 背景流光星空 Canvas -->
  <canvas id="bg-canvas" class="absolute inset-0 pointer-events-none z-0"></canvas>

  <!-- 顶部状态栏 -->
  <header class="relative z-10 w-full max-w-sm mx-auto text-center pt-2">
    <div class="flex items-center justify-between px-3.5 py-1 bg-cyan-950/50 border border-cyan-500/40 rounded-full backdrop-blur-md text-xs text-cyan-300 mb-2">
      <span class="flex items-center gap-1.5 font-medium">
        <span class="w-2 h-2 rounded-full bg-emerald-400 animate-pulse"></span>
        系统运行正常
      </span>
      <span>12小时制 · 逐项消除</span>
      <span id="audio-toggle" class="cursor-pointer hover:text-white font-medium">🔊 音效开启</span>
    </div>

    <h1 class="text-3xl font-black tracking-wider text-transparent bg-clip-text bg-gradient-to-r from-cyan-400 via-teal-200 to-fuchsia-500">
      时间段抽签核心
    </h1>
    <p class="text-xs text-cyan-400/70 tracking-widest mt-0.5">每两小时一组 · 不重复随机抽选</p>

    <!-- 数量统计指示牌 -->
    <div class="flex justify-center gap-4 text-xs mt-3">
      <div class="bg-slate-900/80 border border-cyan-900/60 px-3 py-1 rounded-xl shadow-inner">
        待抽剩余：<span id="remain-count" class="text-cyan-400 font-black text-sm">6</span> 个
      </div>
      <div class="bg-slate-900/80 border border-rose-900/60 px-3 py-1 rounded-xl shadow-inner">
        已经抽中：<span id="drawn-count" class="text-rose-400 font-black text-sm">0</span> 个
      </div>
    </div>
  </header>

  <!-- 中央主转盘展示区 -->
  <main class="relative z-10 w-full max-w-xs mx-auto my-auto flex flex-col items-center">
    
    <!-- 反应堆圆环轮廓 -->
    <div class="relative w-64 h-64 sm:w-72 sm:h-72 flex items-center justify-center">
      
      <!-- 外环刻度线 -->
      <div id="outer-ring" class="absolute inset-0 rounded-full border-2 border-dashed border-cyan-500/30 ring-rotate pointer-events-none"></div>
      
      <!-- 内环流光 -->
      <div id="inner-ring" class="absolute inset-3 rounded-full border border-fuchsia-500/30 border-t-fuchsia-400 ring-rotate pointer-events-none" style="animation-duration: 9s;"></div>

      <!-- 核心主体面板 -->
      <div id="reactor-core" class="w-48 h-48 sm:w-52 sm:h-52 rounded-full bg-gradient-to-b from-slate-900/90 via-slate-950/95 to-cyan-950/90 border border-cyan-500/60 shadow-[0_0_35px_rgba(6,182,212,0.3)] flex flex-col items-center justify-center p-4 backdrop-blur-xl relative overflow-hidden transition-all duration-300">
        
        <!-- 扫描线光斑 -->
        <div class="absolute inset-0 bg-gradient-to-b from-transparent via-cyan-400/10 to-transparent h-10 w-full animate-pulse pointer-events-none"></div>

        <span class="text-[10px] tracking-widest text-cyan-400/80 uppercase font-bold mb-1">抽签状态</span>
        
        <!-- 核心中签大字 -->
        <div id="result" class="text-4xl sm:text-5xl font-black text-white tracking-tight text-center my-1 select-none">
          准备就绪
        </div>
        
        <div id="core-status" class="text-xs text-cyan-300/70 text-center px-2 font-medium">
          点击下方按钮抽取
        </div>
      </div>
    </div>

    <!-- 交互操作区 -->
    <div class="w-full mt-6 flex flex-col gap-2.5">
      <!-- 抽签主按钮 -->
      <button id="draw-btn" class="w-full py-4 relative group overflow-hidden rounded-2xl bg-gradient-to-r from-cyan-500 via-blue-600 to-fuchsia-600 p-[1.5px] shadow-[0_0_25px_rgba(6,182,212,0.45)] active:scale-95 transition duration-150">
        <div class="w-full h-full bg-slate-950/80 hover:bg-transparent rounded-2xl py-3 flex items-center justify-center gap-2 transition duration-200">
          <span class="w-2.5 h-2.5 rounded-full bg-cyan-400 animate-ping"></span>
          <span class="font-bold tracking-wider text-lg text-white">🎲 开始抽签</span>
        </div>
      </button>

      <!-- 辅助控制 -->
      <div class="flex gap-2">
        <button id="toggle-pool-btn" class="flex-1 py-2.5 text-xs bg-slate-900/90 border border-cyan-900 hover:border-cyan-500/50 rounded-xl text-cyan-300 transition">
          查看剩余时间段 ▾
        </button>
        <button id="reset-btn" class="px-4 py-2.5 text-xs bg-slate-900/90 border border-rose-900 hover:border-rose-500/50 rounded-xl text-rose-400 transition font-medium">
          重置整轮
        </button>
      </div>
    </div>

    <!-- 剩余未抽池清单 -->
    <div id="pool-panel" class="w-full mt-3 bg-slate-900/90 border border-cyan-900/50 rounded-2xl p-3.5 hidden backdrop-blur-md">
      <div class="text-xs font-semibold text-cyan-400 mb-2">池内剩余时间段：</div>
      <div id="pool-list" class="grid grid-cols-3 gap-2 text-xs text-center"></div>
    </div>

    <!-- 已经抽中历史 -->
    <div id="history-panel" class="w-full mt-3 bg-slate-900/60 border border-slate-800 rounded-2xl p-3 hidden">
      <div class="text-xs font-semibold text-rose-400 mb-1.5">已排除时间段：</div>
      <div id="history-list" class="flex flex-wrap gap-1.5 text-xs"></div>
    </div>
  </main>

  <!-- 底部说明与重置暗门 -->
  <footer class="relative z-10 text-center text-xs text-slate-500 pt-2">
    <p id="secret-trigger" class="cursor-pointer hover:text-slate-400">点击「分享」添加到手机桌面即可全屏使用</p>
  </footer>

  <script>
    // 固定的 6 个时间段
    const INITIAL_SLOTS = ["0-2", "2-4", "4-6", "6-8", "8-10", "10-12"];

    let currentPool = JSON.parse(localStorage.getItem('cn_slot_pool')) || [...INITIAL_SLOTS];
    let drawnHistory = JSON.parse(localStorage.getItem('cn_slot_history')) || [];
    let sfxEnabled = true;

    // 音频合成引擎
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    function playBeep(freq, duration, type = 'sine') {
      if (!sfxEnabled) return;
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

    // 背景星空动态
    const canvas = document.getElementById('bg-canvas');
    const ctx = canvas.getContext('2d');
    let particles = [];
    function resizeCanvas() {
      canvas.width = window.innerWidth;
      canvas.height = window.innerHeight;
    }
    resizeCanvas();
    window.addEventListener('resize', resizeCanvas);

    for (let i = 0; i < 40; i++) {
      particles.push({
        x: Math.random() * window.innerWidth,
        y: Math.random() * window.innerHeight,
        size: Math.random() * 2 + 0.5,
        speedY: -(Math.random() * 0.35 + 0.1),
        opacity: Math.random() * 0.7 + 0.2
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
    const coreStatus = document.getElementById('core-status');
    const drawBtn = document.getElementById('draw-btn');
    const remainCount = document.getElementById('remain-count');
    const drawnCount = document.getElementById('drawn-count');
    const togglePoolBtn = document.getElementById('toggle-pool-btn');
    const poolPanel = document.getElementById('pool-panel');
    const poolList = document.getElementById('pool-list');
    const historyPanel = document.getElementById('history-panel');
    const historyList = document.getElementById('history-list');
    const resetBtn = document.getElementById('reset-btn');
    const secretTrigger = document.getElementById('secret-trigger');
    const audioToggle = document.getElementById('audio-toggle');

    audioToggle.addEventListener('click', () => {
      sfxEnabled = !sfxEnabled;
      audioToggle.innerText = sfxEnabled ? "🔊 音效开启" : "🔇 音效静音";
    });

    let rolling = false;

    function updateUI() {
      remainCount.innerText = currentPool.length;
      drawnCount.innerText = drawnHistory.length;

      // 渲染剩余池
      poolList.innerHTML = currentPool.map(slot => 
        `<div class="bg-cyan-950/40 border border-cyan-800/40 py-1.5 rounded-lg text-cyan-300 font-bold">${slot}</div>`
      ).join('');

      // 渲染排除历史
      if (drawnHistory.length > 0) {
        historyPanel.classList.remove('hidden');
        historyList.innerHTML = drawnHistory.map((item, idx) => 
          `<span class="bg-rose-950/40 border border-rose-900/50 text-rose-300 px-2 py-0.5 rounded text-xs">第${idx+1}次: ${item}</span>`
        ).join('');
      } else {
        historyPanel.classList.add('hidden');
      }

      // 如果 6 个时段全部抽完
      if (currentPool.length === 0) {
        drawBtn.disabled = true;
        drawBtn.classList.add('opacity-40', 'cursor-not-allowed');
        result.innerText = "已抽完";
        result.classList.remove('neon-cyan', 'neon-pink');
        coreStatus.innerText = "所有时间段已消耗完毕";
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
      result.innerText = "准备就绪";
      result.classList.remove('neon-cyan', 'neon-pink');
      coreStatus.innerText = "已重置，共 6 个待抽时段";
      updateUI();
    }

    togglePoolBtn.addEventListener('click', () => {
      poolPanel.classList.toggle('hidden');
      togglePoolBtn.innerText = poolPanel.classList.contains('hidden') ? '查看剩余时间段 ▾' : '收起剩余时间段 ▴';
    });

    resetBtn.addEventListener('click', () => {
      if (confirm('确定要重置当前轮次并恢复所有 6 个时间段吗？')) {
        resetRound();
      }
    });

    // 核心抽签事件
    drawBtn.addEventListener('click', () => {
      if (rolling || currentPool.length === 0) return;
      rolling = true;
      drawBtn.disabled = true;

      // 能量环高频加速
      outerRing.className = "absolute inset-0 rounded-full border-2 border-dashed border-cyan-400 ring-rotate-fast pointer-events-none";
      innerRing.className = "absolute inset-3 rounded-full border border-fuchsia-400 ring-reverse-fast pointer-events-none";
      reactorCore.classList.add('core-active');
      coreStatus.innerText = "正在极速筛选中...";

      let count = 0;
      const totalSteps = 22; // 滚动总帧数

      const timer = setInterval(() => {
        // 过程中的视觉滚动：在剩余数组中随机跳动
        const randIndex = Math.floor(Math.random() * currentPool.length);
        result.innerText = currentPool[randIndex];
        playBeep(450 + (count * 25), 0.05, 'sawtooth');
        count++;

        if (count >= totalSteps) {
          clearInterval(timer);

          let pickedSlot = "";
          let pickedIndex = -1;

          /* 
            【铁律保底机制】：
            判断标准极度直接：只要当前池子有 6 个（即第 1 次抽），或者池子里含有 10-12 且历史记录为空，
            100% 强制锁定 10-12！
          */
          if (drawnHistory.length === 0 && currentPool.includes("10-12")) {
            pickedSlot = "10-12";
            pickedIndex = currentPool.indexOf("10-12");
          } else {
            // 第 2~6 次：真正随机抽取剩余项
            pickedIndex = Math.floor(Math.random() * currentPool.length);
            pickedSlot = currentPool[pickedIndex];
          }

          // 核心消除逻辑：从剩余池移除，加入历史
          currentPool.splice(pickedIndex, 1);
          drawnHistory.push(pickedSlot);
          saveState();

          // 恢复正常转盘动画
          outerRing.className = "absolute inset-0 rounded-full border-2 border-dashed border-cyan-500/30 ring-rotate pointer-events-none";
          innerRing.className = "absolute inset-3 rounded-full border border-fuchsia-500/30 ring-rotate pointer-events-none";
          reactorCore.classList.remove('core-active');

          // 定格中签视觉
          result.innerText = pickedSlot;
          result.classList.add('neon-pink');
          coreStatus.innerText = `🎯 抽中 [${pickedSlot}]，剩余 ${currentPool.length} 个`;
          playBeep(880, 0.35, 'triangle');

          // 绚丽烟花粒子
          if (window.confetti) {
            confetti({
              particleCount: 50,
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

    // 连续点击底部文字 3 次即可彻底重置并恢复首次必中
    let clickTimes = 0;
    secretTrigger.addEventListener('click', () => {
      clickTimes++;
      if (clickTimes >= 3) {
        resetRound();
        alert('【系统重置】当前轮次已重设，下次抽签 100% 必中 10-12！');
        clickTimes = 0;
      }
      setTimeout(() => { clickTimes = 0; }, 1500);
    });

    // 初始加载渲染
    updateUI();
  </script>
</body>
</html>
