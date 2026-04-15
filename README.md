# Dino.draw
小恐龍遊戲
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <title>Dino Pro: Asset Driven Engine</title>
    <style>
        body { margin: 0; background: #222; display: flex; justify-content: center; align-items: center; height: 100vh; color: white; font-family: sans-serif; }
        canvas { background: #fff; box-shadow: 0 0 20px rgba(0,0,0,0.5); cursor: pointer; }
        .ui { position: absolute; top: 20px; left: 20px; pointer-events: none; }
    </style>
</head>
<body>

<div class="ui">
    <div id="score">SCORE: 00000</div>
    <div id="state" style="font-size: 12px; color: #888;">LOADING ASSETS...</div>
</div>
<canvas id="gameCanvas"></canvas>

<script>
/**
 * 專業資源載入器
 */
const Assets = {
    images: {},
    manifest: {
        dino: 'assets/dino_sheet.png',     // 請在 GitHub 建立 assets 資料夾並放置檔案
        cactus: 'assets/cactus_sheet.png',
        ground: 'assets/ground.png'
    },
    async loadAll() {
        const promises = Object.entries(this.manifest).map(([key, src]) => {
            return new Promise(resolve => {
                const img = new Image();
                img.src = src;
                img.onload = () => { this.images[key] = img; resolve(true); };
                img.onerror = () => { console.warn(`Failed to load: ${src}`); resolve(false); };
            });
        });
        return Promise.all(promises);
    }
};

/**
 * 遊戲實體基底 (Entity)
 */
class Entity {
    constructor(x, y, w, h) {
        this.x = x; this.y = y; this.width = w; this.height = h;
    }
    get hitbox() {
        const pad = 5; // 碰撞箱內縮
        return {
            x: this.x + pad, y: this.y + pad,
            w: this.width - pad * 2, h: this.height - pad * 2
        };
    }
}

/**
 * 恐龍實體 - 含動畫狀態機
 */
class Dino extends Entity {
    constructor() {
        super(50, 0, 60, 60);
        this.baseY = 200 - 60;
        this.y = this.baseY;
        this.dy = 0;
        this.jumpForce = -800;
        this.gravity = 2500;
        this.isGrounded = true;
        
        // 動態定義切換幀
        this.animTimer = 0;
        this.frame = 0;
        this.state = 'RUN'; // RUN, JUMP, EVOLVE
    }

    update(dt, score) {
        // 物理更新
        this.dy += this.gravity * dt;
        this.y += this.dy * dt;

        if (this.y > this.baseY) {
            this.y = this.baseY;
            this.dy = 0;
            this.isGrounded = true;
        }

        // 狀態機邏輯
        if (score >= 500) this.state = 'EVOLVE';
        else if (!this.isGrounded) this.state = 'JUMP';
        else this.state = 'RUN';

        // 動畫幀切換 (每 0.1 秒換一幀)
        this.animTimer += dt;
        if (this.animTimer > 0.1) {
            this.frame = (this.frame + 1) % 2;
            this.animTimer = 0;
        }
    }

    draw(ctx) {
        const img = Assets.images.dino;
        if (img) {
            // 專業繪圖：根據狀態選取 Spritesheet 座標
            // 這裡假設你的 sheet 每一格是 100x100
            let sx = this.frame * 100;
            let sy = (this.state === 'EVOLVE') ? 100 : 0;
            ctx.drawImage(img, sx, sy, 100, 100, this.x, this.y, this.width, this.height);
        } else {
            // Debug Fallback: 沒圖片時畫方塊
            ctx.fillStyle = this.state === 'EVOLVE' ? '#2a9d8f' : '#535353';
            ctx.fillRect(this.x, this.y, this.width, this.height);
        }
    }
}

/**
 * 障礙物實體
 */
class Obstacle extends Entity {
    constructor(speed) {
        const w = 30 + Math.random() * 20;
        const h = 40 + Math.random() * 40;
        super(800, 200 - h, w, h);
        this.speed = speed * (0.8 + Math.random() * 0.4); // 隨機速度攝動
        this.type = Math.floor(Math.random() * 3); // 隨機樣式
    }

    update(dt) { this.x -= this.speed * dt; }

    draw(ctx) {
        const img = Assets.images.cactus;
        if (img) {
            ctx.drawImage(img, this.type * 50, 0, 50, 100, this.x, this.y, this.width, this.height);
        } else {
            ctx.fillStyle = '#ff4d4d';
            ctx.fillRect(this.x, this.y, this.width, this.height);
        }
    }
}

/**
 * 遊戲主引擎
 */
class Engine {
    constructor() {
        this.canvas = document.getElementById('gameCanvas');
        this.ctx = this.canvas.width = 800, this.canvas.height = 250, this.canvas.getContext('2d');
        this.score = 0;
        this.gameSpeed = 400;
        this.entities = { dino: new Dino(), obstacles: [] };
        this.lastTime = 0;
        this.spawnTimer = 0;
        this.isGameOver = false;

        window.addEventListener('keydown', e => {
            if (e.code === 'Space' && !this.isGameOver) this.entities.dino.jump();
            if (e.code === 'Space' && this.isGameOver) location.reload();
        });

        requestAnimationFrame(t => this.loop(t));
    }

    loop(t) {
        const dt = Math.min((t - this.lastTime) / 1000, 0.1);
        this.lastTime = t;

        if (!this.isGameOver) {
            this.update(dt);
            this.draw();
            requestAnimationFrame(t => this.loop(t));
        }
    }

    update(dt) {
        this.score += dt * 10;
        this.gameSpeed = 400 + Math.pow(this.score, 0.6) * 5;
        
        this.entities.dino.update(dt, this.score);

        // 生成障礙
        this.spawnTimer += dt;
        if (this.spawnTimer > 1.5 * (400 / this.gameSpeed)) {
            this.entities.obstacles.push(new Obstacle(this.gameSpeed));
            this.spawnTimer = 0;
        }

        // 更新障礙與碰撞偵測
        this.entities.obstacles.forEach((obs, i) => {
            obs.update(dt);
            if (this.checkCollision(this.entities.dino, obs)) this.isGameOver = true;
            if (obs.x < -100) this.entities.obstacles.splice(i, 1);
        });

        document.getElementById('score').innerText = `SCORE: ${Math.floor(this.score).toString().padStart(5, '0')}`;
    }

    checkCollision(a, b) {
        const r1 = a.hitbox; const r2 = b.hitbox;
        return r1.x < r2.x + r2.w && r1.x + r1.w > r2.x && r1.y < r2.y + r2.h && r1.y + r1.h > r2.y;
    }

    draw() {
        this.ctx.clearRect(0, 0, 800, 250);
        // 畫地面
        this.ctx.strokeStyle = '#535353';
        this.ctx.beginPath(); this.ctx.moveTo(0, 200); this.ctx.lineTo(800, 200); this.ctx.stroke();
        
        this.entities.dino.draw(this.ctx);
        this.entities.obstacles.forEach(o => o.draw(this.ctx));

        if (this.isGameOver) {
            this.ctx.fillStyle = 'rgba(0,0,0,0.7)';
            this.ctx.fillRect(0,0,800,250);
            this.ctx.fillStyle = 'white';
            this.ctx.fillText("GAME OVER - PRESS SPACE", 320, 125);
        }
    }
}

// 啟動
Assets.loadAll().then(() => {
    document.getElementById('state').innerText = "ASSETS LOADED (OR USING FALLBACK)";
    new Engine();
});
</script>
</body>
</html>
