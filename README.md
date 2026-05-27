<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Jardín vs Zombies - Versión Corregida</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #2c3e50;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            font-family: 'Arial', sans-serif;
            user-select: none;
            overflow: hidden;
        }
        #gameWrapper {
            position: relative;
            width: 1000px;
            height: 650px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.8);
            border-radius: 12px;
            overflow: hidden;
            background: #6ab04c;
        }
        #uiBar {
            position: absolute;
            top: 0; left: 0; width: 100%; height: 85px;
            background: linear-gradient(to bottom, #8e44ad, #5e3370);
            display: flex;
            align-items: center;
            padding: 0 15px;
            box-sizing: border-box;
            border-bottom: 4px solid #3c1e4a;
            z-index: 10;
        }
        .sun-counter {
            background: #f1c40f;
            border: 3px solid #e67e22;
            border-radius: 30px;
            padding: 5px 20px;
            font-size: 26px;
            font-weight: bold;
            color: #fff;
            text-shadow: 2px 2px 0px #d35400;
            display: flex;
            align-items: center;
            margin-right: 25px;
        }
        .cards-container {
            display: flex;
            gap: 8px;
        }
        .seed-card {
            background: #ecf0f1;
            border: 3px solid #2c3e50;
            border-radius: 8px;
            width: 65px;
            height: 75px;
            cursor: pointer;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: space-between;
            padding: 2px 0;
            transition: transform 0.1s;
        }
        .seed-card:hover { transform: translateY(-3px); }
        .seed-card.selected { border-color: #2ecc71; background: #d5f5e3; box-shadow: 0 0 15px #2ecc71; }
        .seed-card.disabled { opacity: 0.5; filter: grayscale(100%); cursor: not-allowed; }
        .card-emoji { font-size: 32px; line-height: 1; margin-top: 5px; }
        .card-cost { font-size: 14px; font-weight: bold; background: #fff; width: 100%; text-align: center; border-top: 1px solid #ccc; }
        
        #shovelBtn { margin-left: auto; background: #95a5a6; }
        #shovelBtn.selected { border-color: #e74c3c; background: #fadbd8; box-shadow: 0 0 15px #e74c3c; }

        canvas { display: block; }

        .overlay {
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            display: none;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            z-index: 20;
            text-align: center;
        }
        #roundMessage {
            font-size: 80px; font-weight: bold; color: #f1c40f;
            text-shadow: 4px 4px 0 #000, -2px -2px 0 #000, 2px -2px 0 #000, -2px 2px 0 #000;
            pointer-events: none;
        }
        #gameOverScreen { background: rgba(0,0,0,0.85); color: #e74c3c; }
        #victoryScreen { background: rgba(0,0,0,0.85); color: #2ecc71; }
        .screen-title { font-size: 60px; font-weight: bold; text-shadow: 3px 3px 5px #000; margin-bottom: 20px; }
        .action-btn {
            padding: 15px 30px; font-size: 24px; font-weight: bold; cursor: pointer;
            background: #f39c12; border: 4px solid #d35400; border-radius: 15px; color: white;
        }
    </style>
</head>
<body>

<div id="gameWrapper">
    <div id="uiBar">
        <div class="sun-counter">☀️ <span id="sunScore">150</span></div>
        <div class="cards-container">
            <div class="seed-card" data-type="sunflower" data-cost="25">
                <div class="card-emoji">🌻</div><div class="card-cost">25</div>
            </div>
            <div class="seed-card" data-type="peashooter" data-cost="50">
                <div class="card-emoji">🌱</div><div class="card-cost">50</div>
            </div>
            <div class="seed-card" data-type="cherry" data-cost="100">
                <div class="card-emoji">🍒</div><div class="card-cost">100</div>
            </div>
            <div class="seed-card" data-type="snowpea" data-cost="125">
                <div class="card-emoji">🧊</div><div class="card-cost">125</div>
            </div>
            <div class="seed-card" data-type="chomper" data-cost="150">
                <div class="card-emoji">👄</div><div class="card-cost">150</div>
            </div>
            <div class="seed-card" data-type="wallnut" data-cost="200">
                <div class="card-emoji">🥜</div><div class="card-cost">200</div>
            </div>
        </div>
        <div class="seed-card" id="shovelBtn" data-type="shovel">
            <div class="card-emoji">⛏️</div><div class="card-cost">Quitar</div>
        </div>
    </div>

    <canvas id="gameCanvas" width="1000" height="650"></canvas>

    <div class="overlay" id="roundOverlay"><div id="roundMessage">RONDA 1</div></div>
    <div class="overlay" id="gameOverScreen">
        <div class="screen-title">¡LOS ZOMBIES ENTRARON A TU CASA!</div>
        <button class="action-btn" onclick="location.reload()">Reintentar</button>
    </div>
    <div class="overlay" id="victoryScreen">
        <div class="screen-title">¡HAS SOBREVIVIDO A LAS RONDAS!</div>
        <button class="action-btn" onclick="location.reload()">Jugar de nuevo</button>
    </div>
</div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const sunScoreEl = document.getElementById('sunScore');

    // Medidas del Mapa
    const UI_HEIGHT = 85;
    const HOUSE_WIDTH = 130;
    const MOWER_ZONE = 60;
    const ROWS = 5;
    const COLS = 9;
    const CELL_W = Math.floor((canvas.width - HOUSE_WIDTH - MOWER_ZONE) / COLS);
    const CELL_H = Math.floor((canvas.height - UI_HEIGHT) / ROWS);

    // Estado del Juego
    let frame = 0;
    let sunsAmount = 150;
    let selectedType = null;
    let selectedCost = 0;
    let gameState = 'playing';

    let plants = [];
    let zombies = [];
    let projectiles = [];
    let suns = [];
    let mowers = [];
    let explosions = [];

    // Sistema de Rondas
    const rounds = [
        { id: 1, totalZombies: 4, types: ['normal'], spawnRate: 350, message: "RONDA 1" },
        { id: 2, totalZombies: 10, types: ['normal', 'cone'], spawnRate: 250, message: "RONDA 2" },
        { id: 3, totalZombies: 18, types: ['normal', 'cone', 'bucket'], spawnRate: 180, message: "RONDA FINAL" }
    ];
    let currentRoundIdx = 0;
    let zombiesSpawnedThisRound = 0;
    let zombiesKilledThisRound = 0;
    let roundTimer = 0;
    let isTransitioning = true;

    const plantDefs = {
        sunflower: { emoji: '🌻', hp: 120, isShooter: false },
        peashooter: { emoji: '🌱', hp: 120, isShooter: true, color: '#2ecc71', ice: false, dmg: 20 },
        snowpea: { emoji: '🧊', hp: 120, isShooter: true, color: '#54a0ff', ice: true, dmg: 20 },
        cherry: { emoji: '🍒', hp: 999, isShooter: false },
        chomper: { emoji: '👄', hp: 200, isShooter: false },
        wallnut: { emoji: '🥜', hp: 1000, isShooter: false }
    };

    // Crear Podadoras fijas
    for (let r = 0; r < ROWS; r++) {
        mowers.push({
            x: HOUSE_WIDTH,
            y: UI_HEIGHT + r * CELL_H + CELL_H / 2,
            row: r, active: false, used: false
        });
    }

    // --- MANEJO DE SELECCIÓN ---
    const cards = document.querySelectorAll('.seed-card');
    cards.forEach(card => {
        card.addEventListener('click', (e) => {
            e.stopPropagation(); // Evita clics cruzados
            if (card.classList.contains('disabled')) return;
            
            if (card.classList.contains('selected')) {
                deselect();
            } else {
                cards.forEach(c => c.classList.remove('selected'));
                card.classList.add('selected');
                selectedType = card.dataset.type;
                selectedCost = parseInt(card.dataset.cost) || 0;
            }
        });
    });

    function updateUI() {
        sunScoreEl.innerText = sunsAmount;
        cards.forEach(card => {
            const cost = parseInt(card.dataset.cost);
            if (cost) {
                if (sunsAmount < cost) {
                    card.classList.add('disabled');
                    if (selectedType === card.dataset.type) deselect();
                } else {
                    card.classList.remove('disabled');
                }
            }
        });
    }

    function deselect() {
        selectedType = null;
        selectedCost = 0;
        cards.forEach(c => c.classList.remove('selected'));
    }

    // --- DETECCIÓN DE CLICS MEJORADA ---
    let mouse = { x: -1, y: -1 };
    canvas.addEventListener('mousemove', e => {
        const rect = canvas.getBoundingClientRect();
        mouse.x = e.clientX - rect.left;
        mouse.y = e.clientY - rect.top;
    });

    canvas.addEventListener('click', (e) => {
        if (gameState !== 'playing') return;

        // 1. Recoger Soles primero (Prioridad absoluta)
        for (let i = suns.length - 1; i >= 0; i--) {
            let s = suns[i];
            if (Math.hypot(mouse.x - s.x, mouse.y - s.y) < 40) {
                sunsAmount += s.val;
                suns.splice(i, 1);
                updateUI();
                return; 
            }
        }

        // Limitar clics fuera del campo jugable
        if (mouse.y < UI_HEIGHT || mouse.x < HOUSE_WIDTH + MOWER_ZONE) return;
        
        const c = Math.floor((mouse.x - (HOUSE_WIDTH + MOWER_ZONE)) / CELL_W);
        const r = Math.floor((mouse.y - UI_HEIGHT) / CELL_H);
        
        if (c < 0 || c >= COLS || r < 0 || r >= ROWS) return;

        const cellX = HOUSE_WIDTH + MOWER_ZONE + c * CELL_W + CELL_W / 2;
        const cellY = UI_HEIGHT + r * CELL_H + CELL_H / 2;
        const existingIdx = plants.findIndex(p => p.col === c && p.row === r);

        // Acción de la Pala
        if (selectedType === 'shovel') {
            if (existingIdx > -1) plants.splice(existingIdx, 1);
            deselect();
            return;
        }

        // Acción de Plantar
        if (selectedType && existingIdx === -1 && sunsAmount >= selectedCost) {
            sunsAmount -= selectedCost;
            let pDef = plantDefs[selectedType];
            plants.push({
                type: selectedType, emoji: pDef.emoji,
                x: cellX, y: cellY, row: r, col: c,
                hp: pDef.hp, maxHp: pDef.hp, timer: 0, state: 'idle',
                isShooter: pDef.isShooter
            });
            updateUI();
            deselect();
        }
    });

    function startRoundTransition() {
        isTransitioning = true;
        const overlay = document.getElementById('roundOverlay');
        const msg = document.getElementById('roundMessage');
        msg.innerText = rounds[currentRoundIdx].message;
        overlay.style.display = 'flex';
        
        setTimeout(() => {
            overlay.style.display = 'none';
            isTransitioning = false;
            zombiesSpawnedThisRound = 0;
            zombiesKilledThisRound = 0;
            roundTimer = 0;
        }, 2500);
    }

    function drawBackground() {
        // Césped bicolor alternado
        for (let r = 0; r < ROWS; r++) {
            for (let c = 0; c < COLS; c++) {
                ctx.fillStyle = (r + c) % 2 === 0 ? '#7bed9f' : '#2ed573';
                ctx.fillRect(HOUSE_WIDTH + MOWER_ZONE + c * CELL_W, UI_HEIGHT + r * CELL_H, CELL_W, CELL_H);
            }
        }
        
        // Zona gris de podadoras
        ctx.fillStyle = '#a4b0be';
        ctx.fillRect(HOUSE_WIDTH, UI_HEIGHT, MOWER_ZONE, canvas.height - UI_HEIGHT);

        // Casa izquierda
        ctx.fillStyle = '#f39c12';
        ctx.fillRect(0, UI_HEIGHT, HOUSE_WIDTH, canvas.height - UI_HEIGHT);
        
        ctx.fillStyle = '#c0392b';
        ctx.beginPath();
        ctx.moveTo(0, UI_HEIGHT + 30);
        ctx.lineTo(HOUSE_WIDTH / 2, UI_HEIGHT - 15);
        ctx.lineTo(HOUSE_WIDTH, UI_HEIGHT + 30);
        ctx.fill();

        ctx.fillStyle = '#8e44ad';
        ctx.fillRect(HOUSE_WIDTH - 50, canvas.height - 90, 40, 90);
    }

    function handleRounds() {
        if (isTransitioning) return;
        const currentData = rounds[currentRoundIdx];

        roundTimer++;
        if (roundTimer % currentData.spawnRate === 0 && zombiesSpawnedThisRound < currentData.totalZombies) {
            let r = Math.floor(Math.random() * ROWS);
            let zType = currentData.types[Math.floor(Math.random() * currentData.types.length)];
            
            let zHp = 100; 
            let zSpeed = 0.35 + Math.random() * 0.15;
            
            if (zType === 'cone') zHp = 250;
            if (zType === 'bucket') zHp = 450;

            zombies.push({
                type: zType, 
                x: canvas.width + 30, 
                y: UI_HEIGHT + r * CELL_H + CELL_H / 2,
                row: r, hp: zHp, maxHp: zHp, speed: zSpeed, baseSpeed: zSpeed,
                isEating: false, iceTimer: 0
            });
            zombiesSpawnedThisRound++;
        }

        if (zombiesSpawnedThisRound >= currentData.totalZombies && zombiesKilledThisRound >= currentData.totalZombies) {
            currentRoundIdx++;
            if (currentRoundIdx >= rounds.length) {
                gameState = 'victory';
                document.getElementById('victoryScreen').style.display = 'flex';
            } else {
                startRoundTransition();
            }
        }
    }

    function handlePlants() {
        for (let i = plants.length - 1; i >= 0; i--) {
            let p = plants[i];
            p.timer++;
            
            let yBounce = Math.sin(frame * 0.06 + p.x) * 3;
            ctx.font = "55px Arial";
            ctx.textAlign = "center"; 
            ctx.textBaseline = "middle";

            if (p.type === 'chomper' && p.state === 'digesting') {
                ctx.fillText("😴", p.x, p.y + yBounce);
            } else {
                ctx.fillText(p.emoji, p.x, p.y + yBounce);
            }

            // Vida visual para Nuez o Plantas heridas
            if (p.hp < p.maxHp) {
                ctx.fillStyle = 'rgba(0,0,0,0.6)'; ctx.fillRect(p.x - 20, p.y - 45, 40, 5);
                ctx.fillStyle = '#2ecc71'; ctx.fillRect(p.x - 20, p.y - 45, 40 * (p.hp / p.maxHp), 5);
            }

            // Producción del Girasol
            if (p.type === 'sunflower' && p.timer % 550 === 0) {
                suns.push({ x: p.x, y: p.y, targetY: p.y + 25, vy: -2.5, val: 25, life: 500 });
            }

            // Disparos continuos seguros
            if (p.isShooter && p.timer % 100 === 0) {
                let hasTarget = zombies.some(z => z.row === p.row && z.x > p.x);
                if (hasTarget) {
                    let def = plantDefs[p.type];
                    projectiles.push({ 
                        x: p.x + 25, y: p.y - 10, 
                        speed: 5.5, row: p.row, 
                        color: def.color, ice: def.ice, dmg: def.dmg 
                    });
                }
            }

            // Cereza Explosiva instantánea si hay zombies cerca
            if (p.type === 'cherry') {
                let triggerExplosion = (p.timer >= 50);
                if (!triggerExplosion) {
                    triggerExplosion = zombies.some(z => z.row === p.row && Math.abs(z.x - p.x) < 90);
                }
                
                if (triggerExplosion) {
                    explosions.push({ x: p.x, y: p.y, radius: 10, maxRadius: 150 });
                    zombies.forEach(z => {
                        if (Math.hypot(z.x - p.x, z.y - p.y) < 150) z.hp -= 1000;
                    });
                    p.hp = 0; 
                }
            }

            // Planta carnívora limpia
            if (p.type === 'chomper') {
                if (p.state === 'idle') {
                    let targetIdx = zombies.findIndex(z => z.row === p.row && z.x > p.x && z.x < p.x + 85);
                    if (targetIdx > -1) {
                        zombies[targetIdx].hp = 0; // Se elimina en el siguiente loop
                        p.state = 'digesting'; 
                        p.timer = 0;
                    }
                } else if (p.state === 'digesting' && p.timer > 900) {
                    p.state = 'idle';
                }
            }

            if (p.hp <= 0) plants.splice(i, 1);
        }
    }

    function handleZombies() {
        for (let i = zombies.length - 1; i >= 0; i--) {
            let z = zombies[i];
            
            // Si el zombie murió por disparo/explosión en este frame, limpiarlo ya
            if (z.hp <= 0) {
                zombies.splice(i, 1);
                zombiesKilledThisRound++;
                continue;
            }

            if (z.iceTimer > 0) {
                z.speed = z.baseSpeed * 0.5;
                z.iceTimer--;
            } else {
                z.speed = z.baseSpeed;
            }

            // Detección segura de colisión/mordisco
            let currentPlant = plants.find(p => p.row === z.row && z.x > p.x - 35 && z.x < p.x + 35);
            if (currentPlant) {
                z.isEating = true;
                if (frame % 30 === 0) currentPlant.hp -= 15;
            } else {
                z.isEating = false;
            }

            if (!z.isEating) z.x -= z.speed;

            // Renderizado base del Zombie
            let wobble = z.isEating ? Math.sin(frame * 0.4) * 4 : Math.sin(frame * 0.08) * 2;
            ctx.font = "60px Arial";
            ctx.textAlign = "center"; 
            ctx.textBaseline = "middle";
            
            if (z.iceTimer > 0) {
                ctx.fillStyle = "rgba(116, 185, 255, 0.5)";
                ctx.beginPath(); ctx.arc(z.x, z.y, 35, 0, Math.PI*2); ctx.fill();
            }

            ctx.fillText("🧟", z.x + wobble, z.y);

            // Aditamentos visuales de cascos/conos
            if (z.type === 'cone') {
                ctx.fillStyle = '#e67e22';
                ctx.beginPath(); 
                ctx.moveTo(z.x + wobble - 12, z.y - 25); 
                ctx.lineTo(z.x + wobble + 12, z.y - 25); 
                ctx.lineTo(z.x + wobble, z.y - 55); 
                ctx.fill();
            } else if (z.type === 'bucket') {
                ctx.fillStyle = '#b2bec3';
                ctx.fillRect(z.x + wobble - 18, z.y - 48, 36, 25);
            }

            // Activar Cortadoras
            let mower = mowers[z.row];
            if (!mower.used && z.x < mower.x + 30) {
                mower.active = true;
            }

            // Derrota absoluta si cruza la línea final de la casa
            if (z.x < HOUSE_WIDTH - 15) {
                gameState = 'gameover';
                document.getElementById('gameOverScreen').style.display = 'flex';
            }
        }
    }

    function handleProjectiles() {
        for (let i = projectiles.length - 1; i >= 0; i--) {
            let pj = projectiles[i];
            pj.x += pj.speed;

            ctx.fillStyle = pj.color;
            ctx.beginPath(); ctx.arc(pj.x, pj.y, 9, 0, Math.PI * 2); ctx.fill();

            let hit = false;
            for (let z of zombies) {
                if (z.row === pj.row && pj.x > z.x - 25 && pj.x < z.x + 25 && z.hp > 0) {
                    z.hp -= pj.dmg;
                    if (pj.ice) z.iceTimer = 200; 
                    hit = true;
                    break;
                }
            }

            if (hit || pj.x > canvas.width) projectiles.splice(i, 1);
        }
    }

    function handleSuns() {
        if (frame % 400 === 0 && !isTransitioning) {
            let rx = HOUSE_WIDTH + MOWER_ZONE + Math.random() * (canvas.width - HOUSE_WIDTH - MOWER_ZONE - 60);
            let ty = UI_HEIGHT + 40 + Math.random() * (canvas.height - UI_HEIGHT - 90);
            suns.push({ x: rx, y: -40, targetY: ty, vy: 1.5, val: 25, life: 550 });
        }

        for (let i = suns.length - 1; i >= 0; i--) {
            let s = suns[i];
            if (s.y < s.targetY) {
                s.y += s.vy;
                if (s.vy < 0) s.vy += 0.12; 
            } else {
                s.life--;
                if (s.life <= 0) { suns.splice(i, 1); continue; }
            }

            ctx.save();
            ctx.translate(s.x, s.y);
            ctx.rotate(frame * 0.02);
            ctx.font = "45px Arial";
            ctx.textAlign = "center"; 
            ctx.textBaseline = "middle";
            ctx.globalAlpha = s.life < 90 ? s.life / 90 : 1;
            ctx.fillText("☀️", 0, 0);
            ctx.restore();
        }
    }

    function handleMowers() {
        ctx.font = "50px Arial";
        ctx.textAlign = "center"; 
        ctx.textBaseline = "middle";
        
        for (let m of mowers) {
            if (!m.used) {
                let shake = m.active ? Math.random() * 4 - 2 : 0;
                ctx.fillText("🚜", m.x + shake, m.y);
                
                if (m.active) {
                    m.x += 7;
                    zombies.forEach(z => {
                        if (z.row === m.row && z.x > m.x - 35 && z.x < m.x + 35) z.hp = 0;
                    });
                    if (m.x > canvas.width + 40) {
                        m.used = true; 
                        m.active = false;
                    }
                }
            }
        }
    }

    function drawEffects() {
        // Vista previa traslúcida al colocar plantas
        if (selectedType && mouse.y > UI_HEIGHT && mouse.x > HOUSE_WIDTH + MOWER_ZONE) {
            const c = Math.floor((mouse.x - (HOUSE_WIDTH + MOWER_ZONE)) / CELL_W);
            const r = Math.floor((mouse.y - UI_HEIGHT) / CELL_H);
            
            if (c >= 0 && c < COLS && r >= 0 && r < ROWS) {
                let cx = HOUSE_WIDTH + MOWER_ZONE + c * CELL_W + CELL_W / 2;
                let cy = UI_HEIGHT + r * CELL_H + CELL_H / 2;
                
                ctx.globalAlpha = 0.4;
                ctx.fillStyle = selectedType === 'shovel' ? '#e74c3c' : '#ffffff';
                ctx.fillRect(HOUSE_WIDTH + MOWER_ZONE + c * CELL_W, UI_HEIGHT + r * CELL_H, CELL_W, CELL_H);
                
                ctx.font = "55px Arial"; 
                ctx.fillText(selectedType === 'shovel' ? "⛏️" : plantDefs[selectedType].emoji, cx, cy);
                ctx.globalAlpha = 1.0;
            }
        }

        // Círculos expansivos de explosión
        for (let i = explosions.length - 1; i >= 0; i--) {
            let exp = explosions[i];
            exp.radius += 8;
            ctx.fillStyle = `rgba(235, 94, 85, ${1 - exp.radius / exp.maxRadius})`;
            ctx.beginPath(); ctx.arc(exp.x, exp.y, exp.radius, 0, Math.PI * 2); ctx.fill();
            if (exp.radius >= exp.maxRadius) explosions.splice(i, 1);
        }
    }

    // --- LOOP PRINCIPAL ---
    function gameLoop() {
        if (gameState !== 'playing') return;

        ctx.clearRect(0, 0, canvas.width, canvas.height);

        drawBackground();
        handleMowers();
        handlePlants();
        handleProjectiles();
        handleZombies();
        handleSuns();
        handleRounds();
        drawEffects();

        frame++;
        requestAnimationFrame(gameLoop);
    }

    // Encendido
    updateUI();
    startRoundTransition();
    gameLoop();
</script>

</body>
</html>
