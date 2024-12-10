<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <title>Advanced Particle Simulation Level 8</title>
    <style>
        html, body {
            height: 100%;
            margin: 0;
            padding: 0;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: url('https://img.freepik.com/premium-photo/background-honeycomb-is-made-honeycomb_947814-201763.jpg') no-repeat center center fixed;
            background-size: cover;
            overflow: hidden;
        }

        #particleCanvas {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            z-index: 1;
        }

        .menu {
            position: fixed;
            top: 20%;
            left: 50%;
            transform: translate(-50%, -20%);
            background-color: rgba(255,255,255,0.9);
            padding: 20px;
            border-radius: 15px;
            box-shadow: 0 8px 24px rgba(0,0,0,0.2);
            z-index: 20;
            text-align: center;
            opacity: 1;
        }

        .hud {
            position: fixed;
            top: 10px;
            left: 10px;
            background-color: rgba(255, 255, 255, 0.9);
            padding: 10px;
            border-radius: 10px;
            font-size: 14px;
            z-index: 10;
            box-shadow: 0 4px 16px rgba(0,0,0,0.1);
            opacity:0;
        }

        .restart-button {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background-color: #a1887f;
            color: #ffffff;
            border: none;
            border-radius: 10px;
            padding: 20px 40px;
            cursor: pointer;
            z-index: 25;
            font-size: 18px;
            box-shadow: 0 4px 16px rgba(0,0,0,0.2);
            display: none;
        }

        .freeze-button {
            position: fixed;
            bottom: 10px;
            right: 10px;
            background-color: #4caf50;
            color: #ffffff;
            border: none;
            border-radius: 10px;
            padding: 10px;
            cursor: pointer;
            z-index: 15;
            display: none;
        }

        .joystick-container {
            position: fixed;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            width: 100px;
            height: 100px;
            background: rgba(0, 0, 0, 0.2);
            border-radius: 50%;
            touch-action: none;
            z-index: 15;
        }

        .joystick {
            position: absolute;
            top: 50%;
            left: 50%;
            width: 50px;
            height: 50px;
            background: rgba(255, 255, 255, 0.7);
            border-radius: 50%;
            transform: translate(-50%, -50%);
        }
    </style>
    <!-- p5.js pentru efecte vizuale -->
    <script src="https://cdn.jsdelivr.net/npm/p5@latest/lib/p5.min.js"></script>
    <!-- Howler.js pentru sunet -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/howler/2.2.3/howler.min.js"></script>
    <!-- GSAP pentru animații UI -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.11.5/gsap.min.js"></script>
</head>
<body>
    <div id="menu" class="menu">
        <h2>Particle Simulation Settings</h2>
        <p>Alege rolul tău în joc:</p>
        <label for="role">Select Role:</label>
        <select id="role">
            <option value="bees">Albine</option>
            <option value="wasps">Bondari</option>
        </select>
        <br><br>
        <label for="numParticles">Număr de albine:</label>
        <select id="numParticles">
            <option value="2000">2000</option>
            <option value="5000">5000</option>
        </select>
        <br><br>
        <label for="mode">Mod de joc:</label>
        <select id="mode">
            <option value="timeAttack">Time Attack</option>
            <option value="survival">Survival</option>
            <option value="strategie">Strategie</option>
        </select>
        <br><br>
        <button id="startButton">Start Game</button>
    </div>

    <div id="hud" class="hud">
        <p>Scor Albine: <span id="preyScore">0</span></p>
        <p>Scor Prădători: <span id="predatorScore">0</span></p>
        <p>Număr Prădători: <span id="numPredators">3</span></p>
        <p id="highScoreText" style="display:none;">High Score (Bees): <span id="highScore">0</span></p>
        <div id="modeInfo"></div>
    </div>

    <button id="restartButton" class="restart-button">Restart Game</button>
    <button id="freezeButton" class="freeze-button">Freeze Predators</button>

    <div id="joystickContainer" class="joystick-container" style="display:none">
        <div id="joystick" class="joystick"></div>
    </div>

    <canvas id="particleCanvas"></canvas>

<script type="module">
////////////////////////////////////////////////////////////////////////////////
// Cod avansat "nivel 8" într-un singur fișier HTML
////////////////////////////////////////////////////////////////////////////////

// CONFIG & UTILS
const particleColors = [
    'rgba(224,118,37,0.8)',
    'rgba(32,98,26,0.8)',
    'rgba(34,38,170,0.78)',
    'rgba(134,2,66,0.78)',
    'rgba(121,10,0,0.78)',
    'rgba(85,17,122,0.94)'
];

function getRandomColor() {
    return particleColors[Math.floor(Math.random()*particleColors.length)];
}

// Aducem scorul din localStorage pentru Bees
function getHighScore() {
    let hs = localStorage.getItem('beesHighScore');
    return hs?parseInt(hs,10):0;
}

function setHighScore(value) {
    localStorage.setItem('beesHighScore',value.toString());
}

// ENTITATI

class Hive {
    constructor(x, y, size) {
        this.x = x;
        this.y = y;
        this.size = size;
        this.scaleFactor = 1;
        this.scaleDir=1;
    }

    update() {
        // Mic puls
        this.scaleFactor += this.scaleDir*0.005;
        if (this.scaleFactor>1.05) {this.scaleDir=-1;}
        if (this.scaleFactor<0.95){this.scaleDir=1;}
    }

    draw(ctx) {
        ctx.save();
        ctx.translate(this.x,this.y);
        ctx.scale(this.scaleFactor,this.scaleFactor);
        ctx.beginPath();
        for (let i = 0; i < 6; i++) {
            const angle = (Math.PI / 3) * i;
            const xx = this.size * Math.cos(angle);
            const yy = this.size * Math.sin(angle);
            if (i === 0) {
                ctx.moveTo(xx, yy);
            } else {
                ctx.lineTo(xx, yy);
            }
        }
        ctx.closePath();
        ctx.fillStyle = 'yellow';
        ctx.fill();
        ctx.restore();
    }

    relocate(width,height) {
        this.x = Math.random()*(width - this.size*2) + this.size;
        this.y = Math.random()*(height - this.size*2) + this.size;
    }
}

class GoldenCircle {
    constructor(x,y,radius) {
        this.x = x;
        this.y = y;
        this.baseRadius = radius;
        this.radius = radius;
        this.radDir=1;
    }
    update() {
        // Puls subtil
        this.radius += this.radDir*0.3;
        if (this.radius>this.baseRadius+5) {this.radDir=-1;}
        if (this.radius<this.baseRadius-5){this.radDir=1;}
    }

    draw(ctx) {
        const gradient = ctx.createRadialGradient(this.x, this.y, this.radius*0.1, this.x, this.y, this.radius);
        gradient.addColorStop(0,'rgba(255,255,150,1)');
        gradient.addColorStop(1,'rgba(255,204,0,0.8)');
        ctx.beginPath();
        ctx.arc(this.x,this.y,this.radius,0,Math.PI*2,false);
        ctx.fillStyle = gradient;
        ctx.fill();

        ctx.strokeStyle = 'rgba(255,204,0,0.5)';
        ctx.lineWidth=2;
        ctx.stroke();
    }
}

class Particle {
    constructor(x,y,dx,dy,size,color) {
        this.x = x;
        this.y = y;
        this.dx = dx;
        this.dy = dy;
        this.size = size;
        this.color = color;
        this.hasHoney = false;
        this.fromResource = false;
    }

    static createRandom(width, height) {
        const size = Math.random()*3+1;
        const x = Math.random()*(width - size*2);
        const y = Math.random()*(height - size*2);
        const dx = (Math.random()-0.5)*2;
        const dy = (Math.random()-0.5)*2;
        const color = getRandomColor();
        return new Particle(x,y,dx,dy,size,color);
    }

    static getPollenImage() {
        if (!Particle._pollenImg) {
            Particle._pollenImg = new Image();
            Particle._pollenImg.src = 'https://clipart-library.com/images/5cRdXKbca.png';
        }
        return Particle._pollenImg;
    }

    draw(ctx) {
        // Glow effect: desenăm de mai multe ori cu alpha scăzut
        ctx.save();
        for (let i=0;i<3;i++){
            ctx.globalAlpha = 1 - i*0.3;
            if (this.hasHoney) {
                const pollenImage = Particle.getPollenImage();
                ctx.drawImage(pollenImage, this.x - this.size*1.5, this.y - this.size*1.5, this.size*4, this.size*4);
            } else {
                ctx.beginPath();
                ctx.arc(this.x,this.y,this.size,0,Math.PI*2,false);
                ctx.fillStyle = this.color;
                ctx.fill();
            }
        }
        ctx.restore();
    }

    update(engine) {
        const canvas = engine.canvas;
        if (!this.hasHoney) {
            const distToGold = Math.sqrt((this.x - engine.goldenCircle.x)**2 + (this.y - engine.goldenCircle.y)**2);
            if (distToGold < engine.goldenCircle.radius) {
                this.hasHoney = true;
                this.fromResource = false;
            } else if (engine.mode==='strategie') {
                // Check resources
                for (let res of engine.resources) {
                    const ddx = res.x - this.x;
                    const ddy = res.y - this.y;
                    const dist = Math.sqrt(ddx*ddx+ddy*ddy);
                    if (dist < res.size) {
                        this.hasHoney = true;
                        this.fromResource = true;
                        break;
                    }
                }
            }
        } else {
            let speedFactor = this.fromResource ? 1.5 : 1;
            if (engine.role==='bees') {
                const {x:mx,y:my} = engine.inputManager.getMousePos();
                const dx = mx - this.x;
                const dy = my - this.y;
                const distance = Math.sqrt(dx*dx+dy*dy);
                if (distance>1) {
                    this.x += (dx/distance)*2*speedFactor;
                    this.y += (dy/distance)*2*speedFactor;
                }
            } else {
                const dx = engine.hive.x - this.x;
                const dy = engine.hive.y - this.y;
                const distance = Math.sqrt(dx*dx + dy*dy);
                if (distance>1) {
                    this.x += (dx/distance)*1.5*speedFactor;
                    this.y += (dy/distance)*1.5*speedFactor;
                }
            }

            const distToHive = Math.sqrt((this.x - engine.hive.x)**2+(this.y - engine.hive.y)**2);
            if (distToHive<engine.hive.size) {
                this.hasHoney = false;
                engine.incrementPreyScore();

                if (engine.preyScore %100===0) engine.hive.relocate(engine.canvas.width,engine.canvas.height);
            }
        }

        this.x += this.dx;
        this.y += this.dy;

        if (this.x+this.size>canvas.width || this.x - this.size <0) {
            this.dx = -this.dx;
        }
        if (this.y+this.size>canvas.height || this.y - this.size <0) {
            this.dy = -this.dy;
        }

        this.draw(engine.ctx);
    }
}

class BlackParticle {
    constructor(x,y,size,controlled=false) {
        this.x = x;
        this.y = y;
        this.size = size;
        this.speed = 2;
        this.frozen = false;
        this.controlled = controlled;
        this.vx = 0;
        this.vy = 0;
        this.roamTarget = null;
        this.speedBoostTimer=0; // pentru wasps
    }

    static getWaspImage() {
        if (!BlackParticle._waspImg) {
            BlackParticle._waspImg = new Image();
            BlackParticle._waspImg.src = 'https://www.pngall.com/wp-content/uploads/13/Wasp-Bee.png';
        }
        return BlackParticle._waspImg;
    }

    static createRandom(width,height,controlled=false) {
        const size = 10;
        const x = Math.random()*(width - size*2) + size;
        const y = Math.random()*(height - size*2) + size;
        return new BlackParticle(x,y,size,controlled);
    }

    static createGiant(width,height) {
        const size = 20;
        const x = Math.random()*(width - size*2)+size;
        const y = Math.random()*(height - size*2)+size;
        const giant = new BlackParticle(x,y,size,false);
        giant.speed = 0.75;
        return giant;
    }

    draw(ctx,role) {
        const waspImage = BlackParticle.getWaspImage();
        ctx.save();
        // Glow effect similar
        for (let i=0;i<2;i++){
            ctx.globalAlpha = 1 - i*0.3;
            ctx.drawImage(waspImage, this.x - this.size, this.y - this.size, this.size*2, this.size*2);
        }
        ctx.restore();

        if (this.controlled && role === "wasps") {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.size*2,0,Math.PI*2);
            ctx.strokeStyle='rgba(255,215,0,0.3)';
            ctx.lineWidth=3;
            ctx.stroke();
        }
    }

    pickRoamTarget(width,height) {
        this.roamTarget = {
            x: Math.random()*width,
            y: Math.random()*height
        };
    }

    update(engine) {
        if (this.frozen || engine.gameOver) return;

        const canvas = engine.canvas;
        const role = engine.role;
        const isMobile = engine.inputManager.isMobile();
        const mouse = engine.inputManager.getMousePos();

        // Aplica dificultate dinamică: crește viteza treptat cu scor
        let difficultyFactor = 1 + (engine.predatorScore+engine.preyScore)/2000;
        let actualSpeed = this.speed * difficultyFactor;

        if (this.speedBoostTimer>0) {
            // Wasps speed boost
            actualSpeed*=1.5;
            this.speedBoostTimer--;
        }

        if (this.controlled) {
            if (isMobile && role === "wasps") {
                const dx = engine.inputManager.joystickPosition.x;
                const dy = engine.inputManager.joystickPosition.y;
                if (Math.abs(dx)>0.1 || Math.abs(dy)>0.1) {
                    this.vx = dx*actualSpeed*1.5;
                    this.vy = dy*actualSpeed*1.5;
                } else {
                    this.vx=0; this.vy=0;
                }
            } else {
                const dx = mouse.x - this.x;
                const dy = mouse.y - this.y;
                const distance = Math.sqrt(dx*dx + dy*dy);
                if (distance>1) {
                    this.vx = (dx/distance)*actualSpeed;
                    this.vy = (dy/distance)*actualSpeed;
                } else {
                    this.vx=0;this.vy=0;
                }
            }

            this.x+=this.vx;
            this.y+=this.vy;

            engine.particlesArray.forEach(p=>{
                if (p.hasHoney && !engine.isInsideGoldenCircle(p)) {
                    const distX = p.x - this.x;
                    const distY = p.y - this.y;
                    const distance = Math.sqrt(distX*distX+distY*distY);
                    if (distance < this.size + p.size) {
                        engine.removeParticle(p);
                        engine.incrementPredatorScore();
                    }
                }
            });

        } else {
            let closestParticle = null;
            let minDistance = Infinity;

            for (let particle of engine.particlesArray) {
                if (particle.hasHoney && !engine.isInsideGoldenCircle(particle)) {
                    const dx = particle.x - this.x;
                    const dy = particle.y - this.y;
                    const distance = Math.sqrt(dx*dx + dy*dy);
                    if (distance<minDistance) {
                        minDistance=distance;
                        closestParticle=particle;
                    }
                }
            }

            if (closestParticle) {
                this.roamTarget = null;
                const dx = closestParticle.x - this.x;
                const dy = closestParticle.y - this.y;
                const distance = Math.sqrt(dx*dx + dy*dy);

                if (distance < this.size) {
                    engine.removeParticle(closestParticle);
                    engine.incrementPredatorScore();
                    this.vx=0;this.vy=0;
                } else {
                    this.vx=(dx/distance)*actualSpeed;
                    this.vy=(dy/distance)*actualSpeed;
                }
                this.x+=this.vx;this.y+=this.vy;
            } else {
                if (!this.roamTarget) this.pickRoamTarget(canvas.width, canvas.height);
                const dx = this.roamTarget.x - this.x;
                const dy = this.roamTarget.y - this.y;
                const distance = Math.sqrt(dx*dx + dy*dy);
                if (distance<20) {
                    this.pickRoamTarget(canvas.width, canvas.height);
                } else {
                    this.vx=(dx/distance)*actualSpeed*0.5;
                    this.vy=(dy/distance)*actualSpeed*0.5;
                }
                this.x+=this.vx;this.y+=this.vy;
            }
        }

        // Bounce la margini
        if (this.x+this.size>canvas.width) {
            this.x=canvas.width-this.size;
            this.vx=-this.vx;
        } else if (this.x-this.size<0) {
            this.x=this.size;
            this.vx=-this.vx;
        }
        if (this.y+this.size>canvas.height) {
            this.y=canvas.height-this.size;
            this.vy=-this.vy;
        } else if (this.y-this.size<0) {
            this.y=this.size;
            this.vy=-this.vy;
        }

        this.draw(engine.ctx,role);
    }
}

// Input Manager
class InputManager {
    constructor(canvas, joystickContainer, joystick) {
        this.canvas = canvas;
        this.joystickContainer = joystickContainer;
        this.joystick = joystick;
        this.mouse = {x:null,y:null};
        this.joystickPosition = {x:0,y:0};
        this.isDragging=false;

        window.addEventListener('mousemove',(e)=>{
            this.mouse.x=e.clientX;
            this.mouse.y=e.clientY;
        });

        window.addEventListener('touchmove',(e)=>{
            if (e.touches && e.touches.length>0) {
                this.mouse.x=e.touches[0].clientX;
                this.mouse.y=e.touches[0].clientY;
            }
        });
        
        if (this.isMobile()) {
            this.joystickContainer.addEventListener('touchstart',(ev)=>this.handleTouchStart(ev),{passive:true});
            this.joystickContainer.addEventListener('touchmove',(ev)=>this.handleTouchMove(ev),{passive:false});
            this.joystickContainer.addEventListener('touchend',()=>this.handleTouchEnd(),{passive:true});
        }
    }

    getMousePos() {
        if (!this.mouse.x) {
            return {x: this.canvas.width/2,y:this.canvas.height/2};
        }
        return this.mouse;
    }

    isMobile() {
        return /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent);
    }

    handleTouchStart(e) {
        this.isDragging = true;
        this.handleTouchMove(e);
    }

    handleTouchMove(e) {
        if (!this.isDragging) return;
        const rect = this.joystickContainer.getBoundingClientRect();
        const touch = e.touches[0];
        const dx = touch.clientX - rect.left - rect.width/2;
        const dy = touch.clientY - rect.top - rect.height/2;
        const distance = Math.sqrt(dx*dx+dy*dy);
        const maxDistance = rect.width/2;

        if (distance<maxDistance) {
            this.joystick.style.transform=`translate(${dx}px,${dy}px)`;
            this.joystickPosition.x = dx/maxDistance;
            this.joystickPosition.y = dy/maxDistance;
        } else {
            const angle = Math.atan2(dy,dx);
            const limitedX = Math.cos(angle)*maxDistance;
            const limitedY = Math.sin(angle)*maxDistance;
            this.joystick.style.transform=`translate(${limitedX}px,${limitedY}px)`;
            this.joystickPosition.x=limitedX/maxDistance;
            this.joystickPosition.y=limitedY/maxDistance;
        }
    }

    handleTouchEnd() {
        this.isDragging=false;
        this.joystick.style.transform="translate(-50%,-50%)";
        this.joystickPosition.x=0;
        this.joystickPosition.y=0;
    }
}

// Audio Manager
class AudioManager {
    constructor() {
        this.bgMusic = new Howl({
            src: ['https://www.bensound.com/bensound-music/bensound-cute.mp3'],
            loop: true,
            volume: 0.3
        });
    }

    playBackgroundMusic() {
        this.bgMusic.play();
    }

    stopBackgroundMusic() {
        this.bgMusic.stop();
    }
}

// Mode Base Class
class BaseMode {
    constructor(engine) {
        this.engine = engine;
    }
    start() {}
    checkGameOver(){}
}

// TimeAttack Mode
class TimeAttackMode extends BaseMode {
    constructor(engine) {
        super(engine);
        this.timeLeft=90;
        this.timer=null;
    }

    start() {
        const modeInfo = document.getElementById('modeInfo');
        modeInfo.textContent=`Mode: Time Attack - Time left: ${this.timeLeft}s`;
        this.timer = setInterval(()=>{
            if (!this.engine.gameOver) {
                this.timeLeft--;
                modeInfo.textContent=`Mode: Time Attack - Time left: ${this.timeLeft}s`;
                if (this.timeLeft===60 || this.timeLeft===30) {
                    const bp = BlackParticle.createRandom(this.engine.canvas.width,this.engine.canvas.height);
                    this.engine.blackParticles.push(bp);
                    document.getElementById('numPredators').textContent=this.engine.blackParticles.length;
                }
                if (this.timeLeft<=0) {
                    this.checkGameOver();
                    clearInterval(this.timer);
                }
            } else {
                clearInterval(this.timer);
            }
        },1000);
    }

    checkGameOver() {
        if (this.timeLeft<=0) {
            this.engine.gameOverFunction('Time Attack Over! Scor: '+this.engine.preyScore);
        }
        if (this.engine.preyScore>=3000) {
            this.engine.gameOverFunction('Game Over! Bees Win!');
        } else if (this.engine.predatorScore>=3000) {
            this.engine.gameOverFunction('Game Over! Predators Win!');
        }
    }
}

// Survival Mode
class SurvivalMode extends BaseMode {
    constructor(engine) {
        super(engine);
        this.waveCounter=0;
        this.waveInterval=null;
    }

    start() {
        const modeInfo = document.getElementById('modeInfo');
        modeInfo.textContent='Mode: Survival - Supravietuieste cat mai mult!';
        this.waveInterval = setInterval(()=>{
            if (!this.engine.gameOver) {
                this.waveCounter++;
                this.spawnWave();
            } else {
                clearInterval(this.waveInterval);
            }
        },60000);
    }

    spawnWave() {
        for (let i=0;i<2;i++){
            const bp = BlackParticle.createRandom(this.engine.canvas.width,this.engine.canvas.height);
            bp.speed = 2 + this.waveCounter*0.2;
            this.engine.blackParticles.push(bp);
            document.getElementById('numPredators').textContent=this.engine.blackParticles.length;
        }
    }

    checkGameOver() {
        if (this.engine.preyScore>=3000) {
            this.engine.gameOverFunction('Game Over! Bees Win!');
        } else if (this.engine.predatorScore>=3000) {
            this.engine.gameOverFunction('Game Over! Predators Win!');
        }
    }
}

// Strategie Mode
class StrategieMode extends BaseMode {
    constructor(engine) {
        super(engine);
    }

    start() {
        const modeInfo = document.getElementById('modeInfo');
        modeInfo.textContent='Mode: Strategie - Resurse pe harta! Albinele iau polen instant si devin mai rapide.';
    }

    checkGameOver() {
        if (this.engine.preyScore>=3000) {
            this.engine.gameOverFunction('Game Over! Bees Win!');
        } else if (this.engine.predatorScore>=3000) {
            this.engine.gameOverFunction('Game Over! Predators Win!');
        }
    }
}

// GameEngine
class GameEngine {
    constructor(config) {
        this.config=config;
        this.canvas=document.getElementById(config.canvasId);
        this.ctx=this.canvas.getContext('2d');

        this.hud=document.getElementById(config.hudId);
        this.menu=document.getElementById(config.menuId);
        this.freezeButton=document.getElementById(config.freezeButtonId);
        this.restartButton=document.getElementById(config.restartButtonId);
        this.joystickContainer=document.getElementById(config.joystickContainerId);

        this.gameOver=false;
        this.role='bees';
        this.mode='timeAttack';
        this.numParticles=2000;

        this.particlesArray=[];
        this.blackParticles=[];
        this.resources=[];

        this.preyScore=0;
        this.predatorScore=0;
        this.freezeEnergy=0;

        this.giantWaspAdded=false;

        this.inputManager=new InputManager(this.canvas,this.joystickContainer,document.getElementById(this.config.joystickId));
        this.audioManager=new AudioManager();

        this.currentModeHandler=null;
        this.highScore=getHighScore();

        window.addEventListener('resize',()=>this.onResize());
        this.onResize();

        this.initP5Overlay();

        window.addEventListener('keydown',(e)=>{
            if (e.code==='Space' && !this.gameOver) {
                this.handleSpaceAction();
            }
        });
    }

    initP5Overlay() {
        // p5 pentru linii + gradient overlay
        new p5((sk)=>{
            let lines=[];
            sk.setup=()=>{
                sk.createCanvas(sk.windowWidth,sk.windowHeight).id('p5Canvas');
                sk.noStroke();
                for (let i=0;i<50;i++){
                    lines.push({
                        x:sk.random(sk.width),
                        y:sk.random(sk.height),
                        angle:sk.random(sk.TWO_PI),
                        speed:sk.random(0.2,0.5),
                        col:sk.color(255,255,255,30)
                    });
                }
            };
            sk.draw=()=>{
                sk.clear();
                // Linii parallax subtile
                lines.forEach(l=>{
                    sk.fill(l.col);
                    sk.ellipse(l.x,l.y,3,3);
                    l.x+=Math.cos(l.angle)*l.speed;
                    l.y+=Math.sin(l.angle)*l.speed;
                    if (l.x<0) l.x=sk.width; if(l.x>sk.width)l.x=0;
                    if (l.y<0) l.y=sk.height;if(l.y>sk.height)l.y=0;
                });

                // Gradient overlay de culoare
                let g=sk.drawingContext.createLinearGradient(0,0,sk.width,sk.height);
                g.addColorStop(0,"rgba(255,200,200,0.05)");
                g.addColorStop(1,"rgba(200,200,255,0.05)");
                sk.drawingContext.fillStyle=g;
                sk.rect(0,0,sk.width,sk.height);
                sk.fill();
            };
            sk.windowResized=()=>{
                sk.resizeCanvas(sk.windowWidth,sk.windowHeight);
            };
        });
    }

    startGame({role,mode,numParticles}) {
        this.role=role;
        this.mode=mode;
        this.numParticles=numParticles;

        // Animăm dispariția meniului și apariția HUD
        gsap.to(this.menu, {duration:0.5, opacity:0, onComplete:()=>{
            this.menu.style.display='none';
        }});
        
        // După ce dispare meniul, inițiem jocul
        setTimeout(()=>{
            this.hud.style.display='block';
            gsap.to(this.hud,{duration:0.5, opacity:1});
            
            this.initEntities();
            if (this.role==='bees') {
                document.getElementById('highScoreText').style.display='block';
                document.getElementById('highScore').textContent=this.highScore;
            } else {
                document.getElementById('highScoreText').style.display='none';
            }

            if (this.mode==='timeAttack') {
                this.currentModeHandler=new TimeAttackMode(this);
            } else if (this.mode==='survival') {
                this.currentModeHandler=new SurvivalMode(this);
            } else if (this.mode==='strategie') {
                this.currentModeHandler=new StrategieMode(this);
            }

            this.currentModeHandler.start();
            this.animate();
            this.audioManager.playBackgroundMusic();

        },600);
    }

    initEntities() {
        this.particlesArray=[];
        this.blackParticles=[];
        this.resources=[];
        this.preyScore=0;
        this.predatorScore=0;
        this.freezeEnergy=0;
        this.gameOver=false;
        this.giantWaspAdded=false;

        for (let i=0;i<this.numParticles;i++){
            this.particlesArray.push(Particle.createRandom(this.canvas.width,this.canvas.height));
        }

        for (let i=0;i<3;i++){
            const bp=BlackParticle.createRandom(this.canvas.width,this.canvas.height,this.role==='wasps' && i===0);
            this.blackParticles.push(bp);
        }

        this.goldenCircle=new GoldenCircle(this.canvas.width/2,this.canvas.height/2,100);
        this.hive=new Hive(this.canvas.width/2,this.canvas.height/2+200,40);

        if (this.mode==='strategie') {
            for (let i=0;i<5;i++){
                this.resources.push({
                    x:Math.random()*this.canvas.width,
                    y:Math.random()*this.canvas.height,
                    size:20
                });
            }
        }
    }

    onResize() {
        this.canvas.width=window.innerWidth;
        this.canvas.height=window.innerHeight;
    }

    animate() {
        if (this.gameOver) return;
        this.ctx.clearRect(0,0,this.canvas.width,this.canvas.height);

        this.goldenCircle.update();
        this.goldenCircle.draw(this.ctx);

        this.hive.update();
        this.hive.draw(this.ctx);

        if (this.mode==='strategie') {
            this.ctx.fillStyle='green';
            this.resources.forEach(res=>{
                this.ctx.beginPath();
                this.ctx.arc(res.x,res.y,res.size,0,Math.PI*2);
                this.ctx.fill();
            });
        }

        this.blackParticles.forEach(bp=>bp.update(this));
        this.particlesArray.forEach(p=>p.update(this));

        requestAnimationFrame(()=>this.animate());
    }

    incrementPreyScore() {
        this.preyScore++;
        document.getElementById('preyScore').textContent=this.preyScore;
        this.freezeEnergy+=10;
        this.checkEvents();
    }

    incrementPredatorScore() {
        this.predatorScore++;
        document.getElementById('predatorScore').textContent=this.predatorScore;
        this.checkEvents();
    }

    checkEvents() {
        if (!this.giantWaspAdded && (this.preyScore>=2000 || this.predatorScore>=2000)) {
            this.addGiantWasp();
            this.giantWaspAdded=true;
        }

        if (this.freezeEnergy>=10000 && this.role==='bees') {
            this.freezeButton.style.display='block';
        }

        this.currentModeHandler.checkGameOver();
    }

    addGiantWasp() {
        const giant=BlackParticle.createGiant(this.canvas.width,this.canvas.height);
        this.blackParticles.push(giant);
        document.getElementById('numPredators').textContent=this.blackParticles.length;
    }

    gameOverFunction(message) {
        alert(message);
        this.gameOver=true;
        this.restartButton.style.display='block';
        this.audioManager.stopBackgroundMusic();

        // Update highScore dacă role este bees
        if (this.role==='bees' && this.preyScore>this.highScore) {
            this.highScore=this.preyScore;
            setHighScore(this.highScore);
        }
    }

    freezeBlackParticles() {
        if (this.freezeActive || this.freezeEnergy<10000) return;
        this.freezeActive=true;
        this.freezeEnergy=0;
        this.freezeButton.style.display='none';

        this.blackParticles.forEach(b=>b.frozen=true);
        setTimeout(()=>{
            this.blackParticles.forEach(b=>b.frozen=false);
            this.freezeActive=false;
        },5000);
    }

    restartGame() {
        location.reload();
    }

    isInsideGoldenCircle(p) {
        const dx=p.x - this.goldenCircle.x;
        const dy=p.y - this.goldenCircle.y;
        const distance=Math.sqrt(dx*dx+dy*dy);
        return distance<this.goldenCircle.radius;
    }

    removeParticle(p) {
        const idx=this.particlesArray.indexOf(p);
        if (idx>-1) this.particlesArray.splice(idx,1);
    }

    handleSpaceAction() {
        // Dacă role=bees => val de șoc ce respinge prădătorii în apropiere
        // Dacă role=wasps => boost de viteză pentru bondarul controlat
        if (this.role==='bees') {
            // val de șoc: dacă un prădător e aproape de mouse/hive, îl respingem
            const center={x:this.canvas.width/2,y:this.canvas.height/2};
            const {x:mx,y:my}=this.inputManager.getMousePos();
            // Considerăm originea șocului la poziția mouse-ului
            const shockOrigin={x:mx,y:my};
            this.blackParticles.forEach(bp=>{
                const dx=bp.x - shockOrigin.x;
                const dy=bp.y - shockOrigin.y;
                const dist=Math.sqrt(dx*dx+dy*dy);
                if (dist<200) {
                    // respinge
                    bp.x+=dx/dist*50;
                    bp.y+=dy/dist*50;
                }
            });
        } else if (this.role==='wasps') {
            // boost: controlledBlackParticle
            const controlled = this.blackParticles.find(b=>b.controlled);
            if (controlled) {
                controlled.speedBoostTimer=60; // 1 secundă de boost
            }
        }
    }
}

// MAIN
document.addEventListener('DOMContentLoaded',()=>{
    const startButton=document.getElementById('startButton');
    const restartButton=document.getElementById('restartButton');
    const freezeButton=document.getElementById('freezeButton');

    const roleSelect=document.getElementById('role');
    const numParticlesSelect=document.getElementById('numParticles');
    const modeSelect=document.getElementById('mode');

    const game=new GameEngine({
        canvasId:'particleCanvas',
        hudId:'hud',
        menuId:'menu',
        joystickContainerId:'joystickContainer',
        joystickId:'joystick',
        freezeButtonId:'freezeButton',
        restartButtonId:'restartButton'
    });

    startButton.addEventListener('click',()=>{
        const role=roleSelect.value;
        const mode=modeSelect.value;
        const numParticles=parseInt(numParticlesSelect.value,10)||2000;

        game.startGame({role,mode,numParticles});
    });

    restartButton.addEventListener('click',()=>{
        game.restartGame();
    });

    freezeButton.addEventListener('click',()=>{
        game.freezeBlackParticles();
    });
});
elevenlabs-convai agent-id="5mz0QGMTS6vciobpmiXO"></elevenlabs-convai><script src="https://elevenlabs.io/convai-widget/index.js" async type="text/javascript"></script>
</body>
</html>
