# ind
小恐龍遊戲
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>小恐龍進化 APP - 精緻版</title>
    
    <link rel="manifest" href="manifest.json">
    <meta name="theme-color" content="#e0f7fa">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">

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
            position: relative; 
            width: 100vw; height: 100vh; 
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
            width: 44px; height: 47px; 
            /* 移除了 background-color，只有圖片載入才會顯示內容 */
            background-size: contain; background-repeat: no-repeat;
            z-index: 10; transition: filter 0.3s;
        }
        
        /* 進化濾鏡 (取消無敵，僅視覺改變) */
        .evo-gold { filter: drop-shadow(0 0 10px #FFD700) sepia(0.5) brightness(1.2); }
        .evo-flame { 
            filter: drop-shadow(0 0 15px #ff4757) hue-rotate(-20deg) saturate(2); 
            animation: pulse 0.5s infinite alternate;
        }
        @keyframes pulse { from { transform: scale(1); } to { transform: scale(1.05); } }

        /* 障礙物與雲 */
        .obstacle { 
            position: absolute; bottom: 0; z-index: 5; 
            /* ✅ 重點修改：移除了 background-color，保持背景透明 */
            background-size: contain; background-repeat: no-repeat; 
            /* 增加過渡效果，圖好了才浮現 */
            transition: opacity 0.2s; opacity: 0; 
        }
        .bird { bottom: 85px; }
        
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
        .particle { position: absolute; width: 8px; height: 8px; background: #ffa502; border-radius: 50%; animation: pFly 0.6s forwards; z-index: 9; }
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

    // --- 圖片預載入邏輯 ---
    const imgAssets = {
        dino: './dino.png',
        cactus: './cactus.png',
        bird: './bird.png'
    };
    const images = {};
    let loadedCount = 0;
    Object.entries(imgAssets).forEach(([key, src]) => {
        const img = new Image();
        img.src = src;
        img.onload = () => { 
            images[key] = src;
            loadedCount++;
            if(key === 'dino') {
                dinoEl.style.backgroundImage = `url(${src})`;
                // 恐龍預設顯示
                dinoEl.style.opacity = 1;
            }
        };
        img.onerror = () => {
            console.log(`❌ 圖片載入失敗: ${key}`);
            // 如果失敗，標記為 null
            images[key] = null;
            loadedCount++;
        };
    });

    let isRun = true, score = 0, speed = 6, evo = 'normal';
    let dino = { x: 50, y: 0, vy: 0, w: 40, h: 44, jump: false };
    let obs = [], timer = 0, cTimer = 0;

    const handleAction = (e) => {
        if (e && e.cancelable) e.preventDefault();
        if (!isRun) {
            location.reload(); 
        } else if (!dino.jump) {
            dino.vy = -17;
            dino.jump = true;
        }
    };

    window.addEventListener('touchstart', handleAction, {passive: false});
    window.addEventListener('mousedown', handleAction);
    window.addEventListener('keydown', (e) => { if(e.code === 'Space' || e.code === 'ArrowUp') handleAction(e); });

    function gameLoop() {
        if (!isRun) return;

        dino.vy += 0.85;
        dino.y -= dino.vy;
        if (dino.y <= 0) { dino.y = 0; dino.vy = 0; dino.jump = false; }
        dinoEl.style.bottom = dino.y + 'px';

        score++;
        let s = Math.floor(score/5);
        scoreEl.innerText = `SCORE: ${s.toString().padStart(5, '0')}`;
        
        // 進化邏輯 (取消無敵狀態)
        if (s >= 1000 && evo !== 'flame') {
            evo = 'flame'; dinoEl.className = 'evo-flame'; 
            statusEl.innerText = '🔥 火焰型態 (無無敵)'; statusEl.style.color = '#ff4757';
        } else if (s >= 500 && s < 1000 && evo !== 'gold') {
            evo = 'gold'; dinoEl.className = 'evo-gold'; 
            statusEl.innerText = '✨ 金色覺醒'; statusEl.style.color = '#d4af37';
        }

        if (evo === 'flame' && score % 4 === 0) spawnParticle();

        updateObstacles();
        updateClouds();
        
        // 永遠進行碰撞偵測
        checkCollision();

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
            let w = isBird ? 45 : 30, h = isBird ? 35 : 50, y = isBird ? 85 : 0;
            const el = document.createElement('div');
            el.className = 'obstacle' + (isBird ? ' bird' : '');
            el.style.width = w+'px'; el.style.height = h+'px';
            el.style.left = '100%'; el.style.bottom = y+'px';
            
            // 設定障礙物圖片
            const imgKey = isBird ? 'bird' : 'cactus';
            if (images[imgKey]) {
                el.style.backgroundImage = `url(${images[imgKey]})`;
                // 圖片好了才浮現
                el.style.opacity = 1;
            } else {
                // 如果圖片失敗，這個障礙物將會是透明的（但依然能撞到）
                el.style.opacity = 0;
                console.log(`⚠️ 障礙物 ${imgKey} 圖片遺失，將以透明狀態生成`);
            }
            
            container.appendChild(el);
            obs.push({ el, x: container.offsetWidth, y, w, h });
            timer = 0;
        }
        if (score % 1000 === 0) speed += 0.5;
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
            setTimeout(() => { if(c.parentNode) c.remove(); }, 27000);
        }
    }

    function checkCollision() {
        // 判定框調整 (Padding 縮小判定範圍，讓手感更好)
        let d = { l: dino.x + 10, r: dino.x + dino.w - 10, t: dino.y + dino.h - 8, b: dino.y + 5 };
        for (let o of obs) {
            // 碰撞偵測 (即使圖片失敗是透明的也會死)
            if (d.r > o.x && d.l < o.x + o.w && d.t > o.y && d.b < o.y + o.h) {
                isRun = false; msgEl.style.display = 'block';
            }
        }
    }

    // 初始化檢測：若資源沒下載完，等待一小段時間再開始遊戲
    function checkStart() {
        if (loadedCount >= assetKeys.length) {
            gameLoop();
        } else {
            // 最長等 1.5 秒，沒載入完就強行開始（方塊模式）
            setTimeout(gameLoop, 1500);
        }
    }
    const assetKeys = Object.keys(imgAssets);
    checkStart();
</script>
</body>
</html>
