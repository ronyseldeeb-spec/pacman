<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Pac-Man: Smooth Mobile</title>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap');
        
        * { touch-action: none; user-select: none; -webkit-tap-highlight-color: transparent; }
        
        body { 
            background: #000; color: #fff; font-family: 'Press Start 2P', cursive; 
            display: flex; flex-direction: column; align-items: center; 
            margin: 0; padding: 0; height: 100vh; overflow: hidden;
        }

        #ui {
            display: flex; justify-content: space-between; width: 100%; max-width: 400px;
            padding: 15px; box-sizing: border-box; font-size: 12px; color: #ff0055;
        }

        #canvas-box { position: relative; border: 3px solid #11f; border-radius: 8px; box-shadow: 0 0 20px #11f; }
        canvas { display: block; background: #000; width: 90vw; height: 90vw; max-width: 380px; max-height: 380px; }

        /* VIRTUAL JOYPAD - NO LAG */
        #pad {
            margin-top: 20px; display: grid;
            grid-template-areas: ". up ." "left . right" ". down .";
            gap: 15px;
        }
        .btn {
            width: 70px; height: 70px; background: #222; border: 3px solid #0ff;
            border-radius: 50%; display: flex; align-items: center; justify-content: center;
            font-size: 28px; color: #0ff; box-shadow: 0 4px 0 #088;
        }
        .btn:active { background: #0ff; color: #000; transform: translateY(4px); box-shadow: none; }
        #up { grid-area: up; } #left { grid-area: left; } #right { grid-area: right; } #down { grid-area: down; }
    </style>
</head>
<body onload="window.focus()">

<div id="ui">
    <div>SCORE <span style="color:#fff" id="score">0</span></div>
    <div>LIVES <span style="color:#fff" id="lives">3</span></div>
</div>

<div id="canvas-box">
    <canvas id="game" width="456" height="456"></canvas>
</div>

<div id="pad">
    <div class="btn" id="up">▲</div>
    <div class="btn" id="left">◀</div>
    <div class="btn" id="right">▶</div>
    <div class="btn" id="down">▼</div>
</div>

<script>
const canvas = document.getElementById('game'), ctx = canvas.getContext('2d');
const scoreEl = document.getElementById('score'), livesEl = document.getElementById('lives');
const TILE = 24;
let score = 0, lives = 3, power = 0;
let nextDir = {x:0, y:0};

// Fix for Mobile Lag: Force focus and prevent default behaviors
function setMove(x, y) {
    nextDir = {x, y};
}

// Button Listeners
const btns = {
    'up': [0, -1], 'down': [0, 1], 'left': [-1, 0], 'right': [1, 0]
};
Object.keys(btns).forEach(id => {
    const el = document.getElementById(id);
    const handler = (e) => { e.preventDefault(); setMove(...btns[id]); };
    el.addEventListener('touchstart', handler, {passive: false});
    el.addEventListener('mousedown', handler);
});

// Map Data
const MAP = [
    [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
    [1,3,2,2,2,2,2,2,2,1,2,2,2,2,2,2,2,3,1],
    [1,2,1,1,2,1,1,1,2,1,2,1,1,1,2,1,1,2,1],
    [1,2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,1],
    [1,2,1,1,2,1,0,1,1,1,1,1,0,1,2,1,1,2,1],
    [1,1,1,1,2,1,0,0,0,0,0,0,0,1,2,1,1,1,1],
    [1,1,1,1,2,1,1,1,1,0,1,1,1,1,2,1,1,1,1],
    [1,2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,1],
    [1,2,1,1,2,1,1,1,2,1,2,1,1,1,2,1,1,2,1],
    [1,3,2,2,2,2,2,2,2,0,2,2,2,2,2,2,2,3,1],
    [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1]
];

let grid = [];

class Entity {
    constructor(isP, x, y, color) {
        this.isP = isP; this.sx = x; this.sy = y; this.color = color;
        this.reset();
    }
    reset() {
        this.tx = this.sx; this.ty = this.sy; this.nx = this.sx; this.ny = this.sy;
        this.vx = 0; this.vy = 0; this.timer = 0;
    }
    update() {
        const speed = this.isP ? 6 : 8; // Constant speed to prevent jitter
        if (this.timer === 0) {
            this.tx = this.nx; this.ty = this.ny;

            if (this.isP) {
                if (grid[this.ty][this.tx] === 2) { grid[this.ty][this.tx] = 0; score += 10; scoreEl.innerText = score; }
                if (grid[this.ty][this.tx] === 3) { grid[this.ty][this.tx] = 0; power = 300; }
                
                // Try to change direction
                if (nextDir.x !== 0 || nextDir.y !== 0) {
                    if (grid[this.ty + nextDir.y][this.tx + nextDir.x] !== 1) {
                        this.vx = nextDir.x; this.vy = nextDir.y;
                    }
                }
            } else {
                // Ghost AI
                let dirs = [{x:0,y:1},{x:0,y:-1},{x:1,y:0},{x:-1,y:0}].filter(d => 
                    grid[this.ty+d.y][this.tx+d.x] !== 1 && (d.x !== -this.vx || d.y !== -this.vy)
                );
                let d = dirs[Math.floor(Math.random()*dirs.length)] || {x:-this.vx, y:-this.vy};
                this.vx = d.x; this.vy = d.y;
            }

            if (grid[this.ty + this.vy][this.tx + this.vx] !== 1) {
                this.nx = this.tx + this.vx; this.ny = this.ty + this.vy;
                this.timer = speed;
            } else {
                this.vx = 0; this.vy = 0;
            }
        }
        if (this.timer > 0) this.timer--;
        this.px = (this.tx + (this.nx - this.tx) * (1 - this.timer/speed)) * TILE + 12;
        this.py = (this.ty + (this.ny - this.ty) * (1 - this.timer/speed)) * TILE + 12;
    }
}

let player = new Entity(true, 9, 7), ghosts = [
    new Entity(false, 1, 1, "#f00"), new Entity(false, 17, 1, "#0ff"), new Entity(false, 1, 9, "#f0f")
];

function init() { grid = MAP.map(r => [...r]); player.reset(); ghosts.forEach(g => g.reset()); }

function draw() {
    ctx.fillStyle = "#000"; ctx.fillRect(0,0,456,456);
    
    // Draw Map
    for(let r=0; r<grid.length; r++) for(let c=0; c<grid[r].length; c++) {
        let x = c*TILE, y = r*TILE;
        if(grid[r][c] === 1) { ctx.fillStyle = "#11f"; ctx.fillRect(x+2, y+2, 20, 20); }
        if(grid[r][c] === 2) { ctx.fillStyle = "#fff"; ctx.beginPath(); ctx.arc(x+12,y+12,2,0,7); ctx.fill(); }
        if(grid[r][c] === 3) { ctx.fillStyle = "#ff0"; ctx.beginPath(); ctx.arc(x+12,y+12,6,0,7); ctx.fill(); }
    }

    player.update();
    ctx.fillStyle = power > 0 ? "#0ff" : "#ff0";
    ctx.beginPath(); ctx.arc(player.px, player.py, 10, 0, 7); ctx.fill();

    ghosts.forEach(g => {
        g.update();
        ctx.fillStyle = power > 0 ? "#00f" : g.color;
        ctx.beginPath(); ctx.arc(g.px, g.py, 10, 0, 7); ctx.fill();
        if(Math.hypot(player.px-g.px, player.py-g.py) < 15) {
            if(power > 0) { g.reset(); score += 200; scoreEl.innerText = score; }
            else { lives--; livesEl.innerText = lives; if(lives<=0) location.reload(); else init(); }
        }
    });

    if(power > 0) power--;
    requestAnimationFrame(draw);
}

init();
draw();
</script>
</body>
</html>
