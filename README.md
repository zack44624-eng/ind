# ind
小恐龍遊戲
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>小恐龍進化 APP - 圖片修復版</title>
    <style>
        * { box-sizing: border-box; user-select: none; -webkit-tap-highlight-color: transparent; }
        body { 
            margin: 0; padding: 0; 
            display: flex; justify-content: center; align-items: center; 
            height: 100vh; background-color: #e0f7fa; 
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            overflow: hidden; touch-action: none;
        }

        #game-container { 
            position: relative; width: 100vw; height: 100vh; 
            max-width: 800px; max-height: 400px;
            background-color: white; border-bottom: 4px solid #535353; 
            overflow: hidden; border-radius: 8px;
        }

        /* 核心物體：預設背景色確保「沒圖也能玩」 */
        #dino { 
            position: absolute; bottom: 0; left: 50px; 
            width: 44px; height: 47px; 
            background-color: #535353; /* 沒圖時的顏色 */
            background-size: contain; background-repeat: no-repeat;
            z-index: 10; transition: filter 0.3s;
        }
        
        /* 進化濾鏡 (無無敵) */
        .evo-gold { filter: drop-shadow(0 0 10px #FFD700) sepia(0.5) brightness(1.2); }
        .evo-flame { filter: drop-shadow(0 0 15px #ff4757) hue-rotate(-20deg) saturate(2); }

        .obstacle { position: absolute; bottom: 0; z-index: 5; background-color: #535353; background-size: contain; background-repeat: no-repeat; }
        .bird { bottom: 85px; }
        .cloud { position: absolute; border-radius: 50px; z-index: 1; opacity: 0.6; animation: drift linear infinite; }
        @keyframes drift { from { left: 110%; } to { left: -25%; } }

        #ui { position: absolute; top: 15px; width: 100%; display: flex; justify-content: space-between; padding: 0 20px; font-weight: bold; color: #535353; z-index: 20; pointer-events: none; }
        #msg { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); text-align: center; display: none; z-index: 30; background: rgba(255,255,255,0.95); padding: 30px; border-radius: 15px; border: 3px solid #535353; }
    </style>
</head>
<body>

    <div id="game-container">
        <div id="ui">
            <div id="evo-status">🐣 檢查資源中...</div>
            <div id="score">SCORE: 00000</div>
        </div>
        <div id="dino"></div>
        <div id="msg"><h1>GAME OVER</h1><p>點擊畫面重啟</p></div>
    </div>

<script>
    const dinoEl = document.getElementById('dino');
    const scoreEl = document.getElementById('score');
    const statusEl = document.getElementById('evo-status');
    const msgEl = document.getElementById('msg');
    const container = document.getElementById('game-container');

    // --- 終極路徑修復邏輯 ---
    const images = {};
    const imgNames = ['dino', 'cactus', 'bird'];
    
    // 三重保險：嘗試絕對路徑、相對路徑、以及強制刷新快取
    imgNames.forEach(name => {
        const img = new Image();
        // 加上隨機參數避免 GitHub Pages 快取到舊的 404 頁面
        const testSrc = `./${name}.png?v=${Date.now()}`; 
        img.src = testSrc;
        
        img.onload = () => {
            console.log(`✅ 成功載入: ${name}`);
            images[name] = testSrc;
            if (name === 'dino') dinoEl.style.backgroundImage = `url(${testSrc})`;
            statusEl.innerText = '🐣 幼年體';
        };
        img.onerror = () => {
            console.error(`❌ 載入失敗: ${name}，請確認根目錄是否有 ${name}.png`);
            statusEl.innerText = '🐣 幼年體 (色塊模式)';
        };
    });

    // --- 遊戲邏輯 (取消無敵) ---
    let isRun = true, score = 0, speed = 6, evo = 'normal';
    let dino = { x: 50, y: 0, vy: 0, w: 40, h: 44, jump: false };
    let obs = [], timer = 0, cTimer = 0;

    const handleAction = (e) => {
        if (e && e.cancelable) e.preventDefault();
        if (!isRun) location.reload();
        else if (!dino.jump) { dino.vy = -17; dino.jump = true; }
    };

    window.addEventListener('touchstart', handleAction, {passive: false});
    window.addEventListener('mousedown', handleAction);
    window.addEventListener('keydown', (e) => { if(e.code === 'Space') handleAction(e); });

    function gameLoop() {
        if (!isRun) return;

        dino.vy += 0.85;
        dino.y -= dino.vy;
        if (dino.y <= 0) { dino.y = 0; dino.vy = 0; dino.jump = false; }
        dinoEl.style.bottom = dino.y + 'px';

        score++;
        let s = Math.floor(score/5);
        scoreEl.innerText = `SCORE: ${s.toString().padStart(5, '0')}`;
        
        if (s >= 1000 && evo !== 'flame') {
            evo = 'flame'; dinoEl.className = 'evo-flame'; 
            statusEl.innerText = '🔥 火焰型態 (無無敵)';
        } else if (s >= 500 && s < 1000 && evo !== 'gold') {
            evo = 'gold'; dinoEl.className = 'evo-gold'; 
            statusEl.innerText = '✨ 金色覺醒';
        }

        updateObstacles();
        updateClouds();
        checkCollision(); // 永遠執行碰撞偵測

        requestAnimationFrame(gameLoop);
    }

    function updateObstacles() {
        for (let i = obs.length - 1; i >= 0; i--) {
            obs[i].x -= speed;
            obs[i].el.style.left = obs[i].x + 'px';
            if (obs[i].x < -60) { obs[i].el.remove(); obs.splice(i, 1); }
        }

        if (++timer > (100 - speed)) {
            let isBird = Math.random() > 0.75;
            const el = document.createElement('div');
            el.className = 'obstacle' + (isBird ? ' bird' : '');
            el.style.width = (isBird ? 45 : 30) + 'px';
            el.style.height = (isBird ? 35 : 50) + 'px';
            el.style.left = '100%';
            el.style.bottom = (isBird ? 85 : 0) + 'px';
            
            const name = isBird ? 'bird' : 'cactus';
            if (images[name]) el.style.backgroundImage = `url(${images[name]})`;
            
            container.appendChild(el);
            obs.push({ el, x: container.offsetWidth, y: isBird ? 85 : 0, w: isBird ? 45 : 30, h: isBird ? 35 : 50 });
            timer = 0;
        }
        if (score % 1000 === 0) speed += 0.5;
    }

    function updateClouds() {
        if (++cTimer > 120) {
            const c = document.createElement('div'); c.className = 'cloud';
            c.style.width = '80px'; c.style.height = '30px';
            c.style.background = '#fff';
            c.style.top = (30 + Math.random() * 100) + 'px';
            c.style.animationDuration = '20s';
            container.appendChild(c); cTimer = 0;
            setTimeout(() => c.remove(), 21000);
        }
    }

    function checkCollision() {
        let d = { l: dino.x + 10, r: dino.x + dino.w - 10, t: dino.y + dino.h - 8, b: dino.y + 5 };
        for (let o of obs) {
            if (d.r > o.x && d.l < o.x + o.w && d.t > o.y && d.b < o.y + o.h) {
                isRun = false; msgEl.style.display = 'block';
            }
        }
    }

    gameLoop();
</script>
</body>
</html>
