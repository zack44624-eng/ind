# ind
小恐龍遊戲
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>小恐龍進化 APP</title>
    
    <link rel="manifest" href="manifest.json">
    <meta name="theme-color" content="#e0f7fa">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">

    <style>
        /* 基礎樣式：確保畫面不是白的 */
        * { box-sizing: border-box; user-select: none; -webkit-tap-highlight-color: transparent; }
        body { 
            margin: 0; padding: 0; 
            display: flex; justify-content: center; align-items: center; 
            height: 100vh; background-color: #e0f7fa; /* 淺藍背景 */
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            overflow: hidden; touch-action: none;
        }

        /* 遊戲容器 */
        #game-container { 
            position: relative; 
            width: 100vw; height: 100vh; /* 手機版改為全螢幕 */
            max-width: 800px; max-height: 400px;
            background-color: white; 
            border-bottom: 4px solid #535353; 
            overflow: hidden; 
            box-shadow: 0 10px 30px rgba(0,0,0,0.2);
            border-radius: 8px;
        }

        /* 恐龍實體 */
        #dino { 
            position: absolute; bottom: 0; left: 50px; 
            width: 40px; height: 44px; 
            background-color: #535353; border-radius: 4px; 
            z-index: 10; transition: background 0.3s, transform 0.2s;
        }
        #dino.evo-gold { background-color: #FFD700; box-shadow: 0 0 20px #FFD700; }
        #dino.evo-flame { 
            background-color: #ff4757; 
            box-shadow: 0 0 30px #ff6b81; 
            animation: flame 0.5s infinite alternate; 
        }
        @keyframes flame { from { transform: scale(1); } to { transform: scale(1.1); } }

        /* 障礙物與雲 */
        .obstacle { position: absolute; bottom: 0; z-index: 5; background-color: #535353; }
        .bird { bottom: 80px; border-radius: 50% 50% 0 0; }
        .cloud { 
            position: absolute; border-radius: 50px; z-index: 1; 
            opacity: 0.6; animation: drift linear infinite; 
        }
        @keyframes drift { from { left: 110%; } to { left: -25%; } }

        /* UI 層 */
        #ui { 
            position: absolute; top: 15px; width: 100%; 
            display: flex; justify-content: space-between; padding: 0 20px; 
            font-weight: bold; color: #535353; z-index: 20; pointer-events: none;
        }
        #msg { 
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); 
            text-align: center; display: none; z-index: 30; 
            background: rgba(255,255,255,0.95); padding: 30px; 
            border-radius: 15px; border: 3px solid #535353;
        }
        .particle { position: absolute; width: 8px; height: 8px; background: #ffa502; border-radius: 50%; animation: pFly 0.6s forwards; }
        @keyframes pFly { to { opacity: 0; transform: translate(-30px, -50px); } }
    </style>
</head>
<body>

    <div id="game-container">
        <div id="ui">
            <div id="evo-status">🐣 幼年體</div>
            <div id="score">SCORE: 00000</div>
        </div>
        <div id="dino"></div>
        <div id="msg">
            <h1 style="margin:0">GAME OVER</h1>
            <p>點擊畫面重新開始</p>
        </div>
    </div>

<script>
    const container = document.getElementById('game-container');
    const dinoEl = document.getElementById('dino');
    const scoreEl = document.getElementById('score');
    const statusEl = document.getElementById('evo-status');
    const msgEl = document.getElementById('msg');

    let isRun = true, score = 0, speed = 6, evo = 'normal';
    let dino = { x: 50, y: 0, vy: 0, w: 40, h: 44, jump: false };
    let obs = [], timer = 0, cTimer = 0;

    // 核心動作：跳躍或重啟
    const handleAction = (e) => {
        if (e.cancelable) e.preventDefault();
        if (!isRun) {
            reset();
        } else if (!dino.jump) {
            dino.vy = -17; // 稍微調強跳躍力
            dino.jump = true;
        }
    };

    // 監聽點擊與鍵盤
    window.addEventListener('touchstart', handleAction, {passive: false});
    window.addEventListener('mousedown', handleAction);
    window.addEventListener('keydown', (e) => { if(e.code === 'Space' || e.code === 'ArrowUp') handleAction(e); });

    function gameLoop() {
        if (!isRun) return;

        // 1. 物理運動
        dino.vy += 0.85; // 重力
        dino.y -= dino.vy;
        if (dino.y <= 0) { dino.y = 0; dino.vy = 0; dino.jump = false; }
        dinoEl.style.bottom = dino.y + 'px';

        // 2. 分數與進化
        score++;
        let s = Math.floor(score/5);
        scoreEl.innerText = `SCORE: ${s.toString().padStart(5, '0')}`;
        
        if (s >= 1000 && evo !== 'flame') {
            evo = 'flame'; dinoEl.className = 'evo-flame'; 
            statusEl.innerText = '🔥 火焰模式 (無敵)'; statusEl.style.color = '#ff4757';
        } else if (s >= 500 && s < 1000 && evo !== 'gold') {
            evo = 'gold'; dinoEl.className = 'evo-gold'; 
            statusEl.innerText = '✨ 金色覺醒'; statusEl.style.color = '#d4af37';
        }

        if (evo === 'flame' && score % 4 === 0) spawnParticle();

        // 3. 障礙物與雲朵生成
        updateObstacles();
        updateClouds();
        
        // 4. 碰撞偵測 (火焰模式略過)
        if (evo !== 'flame') checkCollision();

        requestAnimationFrame(gameLoop);
    }

    function spawnParticle() {
        const p = document.createElement('div'); p.className = 'particle';
        p.style.left = (dino.x + 10 + Math.random()*20)+'px';
        p.style.bottom = (dino.y + 10 + Math.random()*20)+'px';
        container.appendChild(p); setTimeout(() => p.remove(), 600);
    }

    function updateObstacles() {
        for (let i = obs.length - 1; i >= 0; i--) {
            obs[i].x -= speed;
            obs[i].el.style.left = obs[i].x + 'px';
            if (obs[i].x < -60) { obs[i].el.remove(); obs.splice(i, 1); }
        }

        if (++timer > (100 - speed)) {
            let isBird = Math.random() > 0.75;
            let w = isBird ? 45 : 25, h = isBird ? 35 : 50, y = isBird ? 85 : 0;
            const el = document.createElement('div');
            el.className = 'obstacle' + (isBird ? ' bird' : '');
            el.style.width = w+'px'; el.style.height = h+'px';
            el.style.left = '100%'; el.style.bottom = y+'px';
            container.appendChild(el);
            obs.push({ el, x: container.offsetWidth, y, w, h });
            timer = 0;
        }
        if (score % 800 === 0) speed += 0.4;
    }

    function updateClouds() {
        if (++cTimer > 120) {
            const c = document.createElement('div'); c.className = 'cloud';
            const colors = ['#ffffff', '#fff9c4', '#e1f5fe', '#f3e5f5'];
            let w = 70 + Math.random() * 70;
            c.style.width = w + 'px'; c.style.height = (w * 0.4) + 'px';
            c.style.background = colors[Math.floor(Math.random()*4)];
            c.style.top = (30 + Math.random() * 100) + 'px';
            c.style.animationDuration = (12 + Math.random() * 15) + 's';
            container.appendChild(c); cTimer = 0;
            setTimeout(() => c.remove(), 27000);
        }
    }

    function checkCollision() {
        let d = { l: dino.x + 8, r: dino.x + dino.w - 8, t: dino.y + dino.h - 5, b: dino.y };
        for (let o of obs) {
            if (d.r > o.x && d.l < o.x + o.w && d.t > o.y && d.b < o.y + o.h) {
                isRun = false; msgEl.style.display = 'block';
            }
        }
    }

    function reset() {
        score = 0; speed = 6; evo = 'normal'; dino.y = 0; dino.vy = 0; isRun = true;
        msgEl.style.display = 'none'; dinoEl.className = ''; 
        statusEl.innerText = '🐣 幼年體'; statusEl.style.color = '#535353';
        obs.forEach(o => o.el.remove()); obs = [];
        gameLoop();
    }

    // 啟動
    gameLoop();
</script>
</body>
</html>
