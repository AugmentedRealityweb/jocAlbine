<html lang="en">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <title>Particle Simulation</title>
    <style>
        body, html {
            height: 100%;
            margin: 0;
            padding: 0;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: url('https://img.freepik.com/premium-photo/background-honeycomb-is-made-honeycomb_947814-201763.jpg') no-repeat center center fixed;
            background-size: cover;
            overflow: hidden;
        }

        canvas {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            z-index: 1;
        }

        #menu {
            position: fixed;
            top: 20%;
            left: 50%;
            transform: translate(-50%, -20%);
            background-color: rgba(255,255,255,0.9);
            padding: 20px;
            border-radius: 15px;
            box-shadow: 0 8px 24px rgba(0,0,0,0.2);
            z-index: 2;
            text-align: center;
        }
        #menu h2 {
            margin-top: 0;
            font-size: 24px;
            color: #333;
        }
        #menu label, #menu select {
            font-size: 16px;
            color: #333;
        }
        #menu button {
            margin-top: 20px;
            background: #fbc02d;
            border: none;
            border-radius: 10px;
            padding: 10px 20px;
            cursor: pointer;
            font-size: 16px;
            color: #333;
            box-shadow: 0 4px 16px rgba(0,0,0,0.1);
        }
        #menu button:hover {
            background: #fdd835;
        }

        #hud {
            position: fixed;
            top: 10px;
            left: 10px;
            background-color: rgba(255, 255, 255, 0.9);
            padding: 10px;
            border-radius: 10px;
            font-size: 14px;
            z-index: 1;
            box-shadow: 0 4px 16px rgba(0,0,0,0.1);
        }

        #restartButton {
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
            z-index: 3;
            display: none;
            font-size: 18px;
            box-shadow: 0 4px 16px rgba(0,0,0,0.2);
        }
        #restartButton:hover {
            background-color: #8d6e63;
        }

        #freezeButton {
            position: fixed;
            bottom: 10px;
            right: 10px;
            background-color: #4caf50;
            color: #ffffff;
            border: none;
            border-radius: 10px;
            padding: 10px;
            cursor: pointer;
            z-index: 2;
            display: none;
        }
        #freezeButton:hover {
            background-color: #43a047;
        }

        #joystickContainer {
            position: fixed;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            width: 100px;
            height: 100px;
            background: rgba(0, 0, 0, 0.2);
            border-radius: 50%;
            touch-action: none;
            z-index: 2;
        }

        #joystick {
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
</head>
<body>

<div id="menu">
    <h2>Particle Simulation Settings</h2>
    <p>Alege rolul tău în joc:</p>
    <label for="role">Select Role:</label>
    <select id="role">
        <option value="bees">Albine</option>
        <option value="wasps">Bondari</option>
    </select>
    <br /><br />
    <label for="numParticles">Număr de albine:</label>
    <select id="numParticles">
        <option value="2000">2000</option>
        <option value="5000">5000</option>
    </select>
    <br /><br />
    <label for="mode">Mod de joc:</label>
    <select id="mode">
        <option value="timeAttack">Time Attack</option>
        <option value="survival">Survival</option>
        <option value="strategie">Strategie</option>
    </select>
    <br /><br />
    <button onclick="startGame()">Start Game</button>
</div>

<div id="hud" style="display:none">
    <p>Scor Albine: <span id="preyScore">0</span></p>
    <p>Scor Prădători: <span id="predatorScore">0</span></p>
    <p>Număr Prădători: <span id="numPredators">3</span></p>
    <div id="modeInfo"></div>
</div>

<button id="restartButton" onclick="restartGame()">Restart Game</button>
<button id="freezeButton" onclick="freezeBlackParticles()">Freeze Predators</button>

<div id="joystickContainer" style="display:none">
    <div id="joystick"></div>
</div>

<canvas id="particleCanvas"></canvas>

<script>
// Logica similară cu cea anterioară, cu ajustările cerute

let isMobile = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent);
let mouse = { x: null, y: null };
let role = "bees";
let mode = "timeAttack";

let giantWaspAdded = false;
let gameOver = false;

let timer = null;
let timeLeft = 90; // Time Attack: 90 sec
let waveCounter = 0; // Survival: creste la fiecare 60 de sec
let resources = []; // Strategie

let preyScore = 0;
let predatorScore = 0;
let freezeActive = false;
let freezeEnergy = 0;

window.addEventListener('mousemove', function (event) {
    mouse.x = event.clientX;
    mouse.y = event.clientY;
});

window.addEventListener('touchmove', function (event) {
    if (event.touches && event.touches.length > 0) {
        mouse.x = event.touches[0].clientX;
        mouse.y = event.touches[0].clientY;
    }
});

const canvas = document.getElementById('particleCanvas');
const ctx = canvas.getContext('2d');

let particlesArray = [];
let blackParticles = [];
let controlledBlackParticle;
let goldenCircle;
let hive;

const particleColors = [
    { name: 'Pastel Pink', rgba: 'rgba(224, 118, 37, 0.8)' },
    { name: 'Pastel Orange', rgba: 'rgba(32, 98, 26, 0.8)' },
    { name: 'Pastel Yellow', rgba: 'rgba(34, 38, 170, 0.78)' },
    { name: 'Pastel Green', rgba: 'rgba(134, 2, 66, 0.78)' },
    { name: 'Pastel Blue', rgba: 'rgba(121, 10, 0, 0.78)' },
    { name: 'Pastel Purple', rgba: 'rgba(85, 17, 122, 0.94)' }
];

const joystickPosition = { x: 0, y: 0 };

const pollenImage = new Image();
pollenImage.src = 'https://clipart-library.com/images/5cRdXKbca.png';

const waspImage = new Image();
waspImage.src = 'https://www.pngall.com/wp-content/uploads/13/Wasp-Bee.png';

class Hive {
    constructor(x, y, size) {
        this.x = x;
        this.y = y;
        this.size = size;
    }

    draw() {
        ctx.beginPath();
        for (let i = 0; i < 6; i++) {
            const angle = (Math.PI / 3) * i;
            const x = this.x + this.size * Math.cos(angle);
            const y = this.y + this.size * Math.sin(angle);
            if (i === 0) {
                ctx.moveTo(x, y);
            } else {
                ctx.lineTo(x, y);
            }
        }
        ctx.closePath();
        ctx.fillStyle = 'yellow';
        ctx.fill();
    }

    relocate() {
        this.x = Math.random() * (canvas.width - this.size * 2) + this.size;
        this.y = Math.random() * (canvas.height - this.size * 2) + this.size;
    }
}

class GoldenCircle {
    constructor(x, y, radius) {
        this.x = x;
        this.y = y;
        this.radius = radius;
    }

    draw() {
        const gradient = ctx.createRadialGradient(this.x, this.y, this.radius * 0.1, this.x, this.y, this.radius);
        gradient.addColorStop(0, 'rgba(255, 255, 150, 1)');
        gradient.addColorStop(1, 'rgba(255, 204, 0, 0.8)');
        ctx.beginPath();
        ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2, false);
        ctx.fillStyle = gradient;
        ctx.fill();

        ctx.strokeStyle = 'rgba(255, 204, 0, 0.5)';
        ctx.lineWidth = 2;
        ctx.stroke();
    }
}

function isInsideGoldenCircle(particle) {
    const dx = particle.x - goldenCircle.x;
    const dy = particle.y - goldenCircle.y;
    const distance = Math.sqrt(dx * dx + dy * dy);
    return distance < goldenCircle.radius;
}

class BlackParticle {
    constructor(x, y, size) {
        this.x = x;
        this.y = y;
        this.size = size;
        this.speed = 2;
        this.frozen = false;
        this.controlled = false;
        this.vx = 0;
        this.vy = 0;
        this.roamTarget = null;
    }

    draw() {
        ctx.drawImage(waspImage, this.x - this.size, this.y - this.size, this.size * 2, this.size * 2);

        if (this.controlled && role === "wasps") {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.size*2, 0, Math.PI*2);
            ctx.strokeStyle = 'rgba(255,215,0,0.3)'; 
            ctx.lineWidth = 3;
            ctx.stroke();
        }
    }

    pickRoamTarget() {
        this.roamTarget = {
            x: Math.random() * canvas.width,
            y: Math.random() * canvas.height
        };
    }

    update() {
        if (this.frozen || gameOver) return;

        if (this.controlled) {
            if (isMobile && role === "wasps") {
                const dx = joystickPosition.x;
                const dy = joystickPosition.y;
                if (Math.abs(dx) > 0.1 || Math.abs(dy) > 0.1) {
                    this.vx = dx * this.speed * 1.5;
                    this.vy = dy * this.speed * 1.5;
                } else {
                    this.vx = 0;
                    this.vy = 0;
                }
            } else {
                const dx = mouse.x - this.x;
                const dy = mouse.y - this.y;
                const distance = Math.sqrt(dx*dx + dy*dy);
                if (distance > 1) {
                    this.vx = (dx/distance)*this.speed;
                    this.vy = (dy/distance)*this.speed;
                } else {
                    this.vx = 0;
                    this.vy = 0;
                }
            }

            this.x += this.vx;
            this.y += this.vy;

            particlesArray.forEach(particle => {
                if (particle.hasHoney && !isInsideGoldenCircle(particle)) {
                    const distX = particle.x - this.x;
                    const distY = particle.y - this.y;
                    const distance = Math.sqrt(distX*distX + distY*distY);
                    if (distance < this.size + particle.size) {
                        const index = particlesArray.indexOf(particle);
                        if (index > -1) {
                            particlesArray.splice(index, 1);
                            predatorScore++;
                            document.getElementById('predatorScore').textContent = predatorScore;
                            handleScoreEvents();
                        }
                    }
                }
            });

        } else {
            let closestParticle = null;
            let minDistance = Infinity;

            for (let particle of particlesArray) {
                if (particle.hasHoney && !isInsideGoldenCircle(particle)) {
                    const dx = particle.x - this.x;
                    const dy = particle.y - this.y;
                    const distance = Math.sqrt(dx*dx + dy*dy);
                    if (distance < minDistance) {
                        minDistance = distance;
                        closestParticle = particle;
                    }
                }
            }

            if (closestParticle) {
                this.roamTarget = null;
                const dx = closestParticle.x - this.x;
                const dy = closestParticle.y - this.y;
                const distance = Math.sqrt(dx*dx + dy*dy);

                if (distance < this.size) {
                    const index = particlesArray.indexOf(closestParticle);
                    if (index > -1) {
                        particlesArray.splice(index, 1);
                        predatorScore++;
                        document.getElementById('predatorScore').textContent = predatorScore;
                        handleScoreEvents();
                    }
                    this.vx = 0;
                    this.vy = 0;
                } else {
                    this.vx = (dx/distance)*this.speed;
                    this.vy = (dy/distance)*this.speed;
                }
                this.x += this.vx;
                this.y += this.vy;
            } else {
                // Nicio tinta -> roam
                if (!this.roamTarget) {
                    this.pickRoamTarget();
                }
                const dx = this.roamTarget.x - this.x;
                const dy = this.roamTarget.y - this.y;
                const distance = Math.sqrt(dx*dx + dy*dy);
                if (distance < 20) {
                    this.pickRoamTarget();
                } else {
                    this.vx = (dx/distance)*this.speed*0.5; 
                    this.vy = (dy/distance)*this.speed*0.5;
                }
                this.x += this.vx;
                this.y += this.vy;
            }
        }

        // Bounce la margini
        if (this.x + this.size > canvas.width) {
            this.x = canvas.width - this.size;
            this.vx = -this.vx;
        } else if (this.x - this.size < 0) {
            this.x = this.size;
            this.vx = -this.vx;
        }

        if (this.y + this.size > canvas.height) {
            this.y = canvas.height - this.size;
            this.vy = -this.vy;
        } else if (this.y - this.size < 0) {
            this.y = this.size;
            this.vy = -this.vy;
        }

        this.draw();
    }
}

class Particle {
    constructor(x, y, directionX, directionY, size, color) {
        this.x = x;
        this.y = y;
        this.directionX = directionX;
        this.directionY = directionY;
        this.size = size;
        this.color = color;
        this.originalColor = color.rgba;
        this.hasHoney = false;
        this.fromResource = false; // pentru strategie: daca a luat polen din resource point
    }

    draw() {
        if (this.hasHoney) {
            ctx.drawImage(pollenImage, this.x - this.size*1.5, this.y - this.size*1.5, this.size*4, this.size*4);
        } else {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.size, 0, Math.PI*2, false);
            ctx.fillStyle = this.originalColor;
            ctx.fill();
        }
    }

    update() {
        if (!this.hasHoney) {
            const dx = goldenCircle.x - this.x;
            const dy = goldenCircle.y - this.y;
            const distance = Math.sqrt(dx*dx + dy*dy);

            if (distance < goldenCircle.radius) {
                this.hasHoney = true;
                this.fromResource = false;
            } else if (mode==='strategie') {
                // Check resource points
                for (let res of resources) {
                    const ddx = res.x - this.x;
                    const ddy = res.y - this.y;
                    const dist = Math.sqrt(ddx*ddx + ddy*ddy);
                    if (dist < res.size) {
                        // Ia polen instant
                        this.hasHoney = true;
                        this.fromResource = true;
                        break;
                    }
                }
            }
        } else {
            let speedFactor = 1;
            if (this.fromResource) speedFactor = 1.5; // Mai rapid daca a luat polen din resource

            if (role === "bees") {
                const dx = mouse.x - this.x;
                const dy = mouse.y - this.y;
                const distance = Math.sqrt(dx*dx + dy*dy);
                if (distance > 1) {
                    this.x += (dx/distance)*2*speedFactor;
                    this.y += (dy/distance)*2*speedFactor;
                }
            } else {
                const dx = hive.x - this.x;
                const dy = hive.y - this.y;
                const distance = Math.sqrt(dx*dx + dy*dy);
                if (distance > 1) {
                    this.x += (dx/distance)*1.5*speedFactor;
                    this.y += (dy/distance)*1.5*speedFactor;
                }
            }

            const distToHive = Math.sqrt((this.x - hive.x)**2 + (this.y - hive.y)**2);
            if (distToHive < hive.size) {
                this.hasHoney = false;
                preyScore++;
                document.getElementById('preyScore').textContent = preyScore;
                freezeEnergy += 10;

                if (preyScore % 100 === 0) {
                    hive.relocate();
                }

                handleScoreEvents();
            }
        }

        this.x += this.directionX;
        this.y += this.directionY;

        if (this.x + this.size > canvas.width || this.x - this.size < 0) {
            this.directionX = -this.directionX;
        }
        if (this.y + this.size > canvas.height || this.y - this.size < 0) {
            this.directionY = -this.directionY;
        }

        this.draw();
    }
}

function handleScoreEvents() {
    if (!giantWaspAdded && (preyScore >= 2000 || predatorScore >= 2000)) {
        addGiantWasp();
        giantWaspAdded = true;
    }

    // Daca e bees si freezeEnergy >=10000, apar butonul freeze
    if (freezeEnergy >= 10000 && role === "bees") {
        document.getElementById('freezeButton').style.display = 'block';
    }

    checkGameOver();
}

function addGiantWasp() {
    const size = 20;
    const x = Math.random() * (canvas.width - size * 2) + size;
    const y = Math.random() * (canvas.height - size * 2) + size;
    const giantWasp = new BlackParticle(x, y, size);
    giantWasp.speed = 0.75;
    blackParticles.push(giantWasp);
    document.getElementById('numPredators').textContent = blackParticles.length;
}

function checkGameOver() {
    if (mode === 'timeAttack') {
        if (timeLeft <= 0) {
            alert('Time Attack Over! Scor: ' + preyScore);
            gameOver = true;
            document.getElementById('restartButton').style.display = 'block';
        }
    }

    if (preyScore >= 3000) {
        alert('Game Over! Bees Win!');
        gameOver = true;
        document.getElementById('restartButton').style.display = 'block';
    } else if (predatorScore >= 3000) {
        alert('Game Over! Predators Win!');
        gameOver = true;
        document.getElementById('restartButton').style.display = 'block';
    }
}

function freezeBlackParticles() {
    if (freezeActive || freezeEnergy < 10000) return;

    freezeActive = true;
    freezeEnergy = 0;
    document.getElementById('freezeButton').style.display = 'none';

    blackParticles.forEach(particle => particle.frozen = true);
    setTimeout(() => {
        blackParticles.forEach(particle => particle.frozen = false);
        freezeActive = false;
    }, 5000);
}

function spawnWave() {
    // Pentru survival: la fiecare val, adaugam 2 bondari
    for (let i=0;i<2;i++){
        const size = 10;
        const x = Math.random() * (canvas.width - size * 2) + size;
        const y = Math.random() * (canvas.height - size * 2) + size;
        const bp = new BlackParticle(x,y,size);
        // Crestere viteza odata cu valurile
        bp.speed = 2 + waveCounter*0.2;
        blackParticles.push(bp);
        document.getElementById('numPredators').textContent = blackParticles.length;
    }
}

function init() {
    particlesArray = [];
    blackParticles = [];
    giantWaspAdded = false;

    const numParticles = parseInt(document.getElementById('numParticles').value) || 2000;
    for (let i = 0; i < numParticles; i++) {
        const size = Math.random() * 3 + 1;
        const x = Math.random() * (canvas.width - size * 2);
        const y = Math.random() * (canvas.height - size * 2);
        const directionX = (Math.random() - 0.5) * 2;
        const directionY = (Math.random() - 0.5) * 2;
        const color = particleColors[Math.floor(Math.random() * particleColors.length)];
        particlesArray.push(new Particle(x, y, directionX, directionY, size, color));
    }

    for (let i = 0; i < 3; i++) {
        const size = 10;
        const x = Math.random() * (canvas.width - size * 2) + size;
        const y = Math.random() * (canvas.height - size * 2) + size;
        const blackParticle = new BlackParticle(x, y, size);
        if (role === "wasps" && i === 0) {
            blackParticle.controlled = true;
            controlledBlackParticle = blackParticle;
        }
        blackParticles.push(blackParticle);
    }

    goldenCircle = new GoldenCircle(canvas.width / 2, canvas.height / 2, 100);
    hive = new Hive(canvas.width / 2, canvas.height / 2 + 200, 40);

    if (mode==='strategie') {
        // 5 puncte de resurse
        for (let i=0;i<5;i++){
            const rx = Math.random()*canvas.width;
            const ry = Math.random()*canvas.height;
            resources.push({x:rx,y:ry,size:20});
        }
    }
}

let timeInterval = null;
let waveInterval = null;

function animate() {
    if (gameOver) return;

    ctx.clearRect(0, 0, canvas.width, canvas.height);
    goldenCircle.draw();
    hive.draw();

    if (mode==='strategie') {
        // Desenam resource point-urile
        ctx.fillStyle='green';
        resources.forEach(res=>{
            ctx.beginPath();
            ctx.arc(res.x,res.y,res.size,0,Math.PI*2);
            ctx.fill();
        });
    }

    blackParticles.forEach(blackParticle => blackParticle.update());
    particlesArray.forEach(particle => particle.update());
    requestAnimationFrame(animate);
}

function startGame() {
    role = document.getElementById('role').value;
    mode = document.getElementById('mode').value;
    document.getElementById('menu').style.display = 'none';
    document.getElementById('hud').style.display = 'block';
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
    init();

    if (role === "wasps") {
        document.getElementById('freezeButton').style.display = 'none';
    }

    if (isMobile && role === "wasps") {
        document.getElementById('joystickContainer').style.display = 'block';
    } else {
        document.getElementById('joystickContainer').style.display = 'none';
    }

    if (mode === 'timeAttack') {
        timeLeft = 90;
        document.getElementById('modeInfo').innerHTML = 'Mode: Time Attack - Time left: '+timeLeft+'s';
        timer = setInterval(()=>{
            if (!gameOver) {
                timeLeft--;
                document.getElementById('modeInfo').innerHTML = 'Mode: Time Attack - Time left: '+timeLeft+'s';
                if (timeLeft===60 || timeLeft===30) {
                    // cresc dificultatea - adaug un bondar
                    const size = 10;
                    const x = Math.random()*canvas.width;
                    const y = Math.random()*canvas.height;
                    const bp = new BlackParticle(x,y,size);
                    blackParticles.push(bp);
                    document.getElementById('numPredators').textContent=blackParticles.length;
                }
                if (timeLeft <=0) {
                    checkGameOver();
                    clearInterval(timer);
                }
            } else {
                clearInterval(timer);
            }
        },1000);
    } else if (mode==='survival') {
        document.getElementById('modeInfo').innerHTML = 'Mode: Survival - Supravietuieste cat mai mult!';
        waveInterval = setInterval(()=>{
            if (!gameOver) {
                waveCounter++;
                spawnWave();
            } else {
                clearInterval(waveInterval);
            }
        },60000);
    } else if (mode==='strategie') {
        document.getElementById('modeInfo').innerHTML = 'Mode: Strategie - Puncte de resurse pe harta! Albinele iau polen instant si devin mai rapide.';
    }

    animate();
}

window.addEventListener('resize', function() {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
});

canvas.width = window.innerWidth;
canvas.height = window.innerHeight;

let isDragging = false;
const joystickContainer = document.getElementById('joystickContainer');
const joystick = document.getElementById('joystick');

if (isMobile) {
    joystickContainer.addEventListener('touchstart', handleTouchStart, {passive:true});
    joystickContainer.addEventListener('touchmove', handleTouchMove, {passive:false});
    joystickContainer.addEventListener('touchend', handleTouchEnd, {passive:true});
}

function handleTouchStart(event) {
    isDragging = true;
    handleTouchMove(event);
}

function handleTouchMove(event) {
    if (!isDragging) return;

    const rect = joystickContainer.getBoundingClientRect();
    const touch = event.touches[0];
    const dx = touch.clientX - rect.left - rect.width / 2;
    const dy = touch.clientY - rect.top - rect.height / 2;
    const distance = Math.sqrt(dx*dx + dy*dy);
    const maxDistance = rect.width / 2;

    if (distance < maxDistance) {
        joystick.style.transform = `translate(${dx}px, ${dy}px)`;
        joystickPosition.x = dx / maxDistance;
        joystickPosition.y = dy / maxDistance;
    } else {
        const angle = Math.atan2(dy, dx);
        const limitedX = Math.cos(angle)*maxDistance;
        const limitedY = Math.sin(angle)*maxDistance;
        joystick.style.transform = `translate(${limitedX}px, ${limitedY}px)`;
        joystickPosition.x = limitedX/maxDistance;
        joystickPosition.y = limitedY/maxDistance;
    }
}

function handleTouchEnd() {
    isDragging = false;
    joystick.style.transform = "translate(-50%, -50%)";
    joystickPosition.x = 0;
    joystickPosition.y = 0;
}

function restartGame() {
    location.reload();
}

</script>

</body>
</html>
