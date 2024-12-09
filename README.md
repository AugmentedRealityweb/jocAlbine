<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Particle Simulation</title>
    <style>
        body, html {
            height: 100%;
            margin: 0;
            padding: 0;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: #53500d;
            display: flex;
            justify-content: center;
            align-items: center;
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
            background-color: #fff;
            padding: 20px;
            border-radius: 10px;
            box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
            z-index: 2;
        }
        #hud {
            position: fixed;
            top: 10px;
            left: 10px;
            background-color: rgba(255, 255, 255, 0.8);
            padding: 10px;
            border-radius: 5px;
            font-size: 14px;
            z-index: 1;
        }
        #restartButton {
            position: fixed;
            top: 10px;
            right: 10px;
            background-color: #a1887f;
            color: #ffffff;
            border: none;
            border-radius: 5px;
            padding: 10px;
            cursor: pointer;
            z-index: 2;
        }
        #freezeButton {
            position: fixed;
            bottom: 10px;
            right: 10px;
            background-color: #4caf50;
            color: #ffffff;
            border: none;
            border-radius: 5px;
            padding: 10px;
            cursor: pointer;
            z-index: 2;
            display: none;
        }
        #joystickContainer {
            position: fixed;
            bottom: 10%;
            left: 50%;
            transform: translate(-50%, 0);
            width: 150px;
            height: 150px;
            display: none;
            z-index: 3;
        }
        #joystick {
            width: 100%;
            height: 100%;
            background-color: rgba(0, 0, 0, 0.2);
            border-radius: 50%;
            position: relative;
        }
        #joystickThumb {
            width: 50px;
            height: 50px;
            background-color: rgba(0, 0, 0, 0.6);
            border-radius: 50%;
            position: absolute;
            top: 50%;
            left: 50%;
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
    <br><br>
    <label for="numParticles">Număr de albine:</label>
    <select id="numParticles">
        <option value="500">500</option>
        <option value="2000">2000</option>
        <option value="5000">5000</option>
    </select>
    <br><br>
    <button onclick="startGame()">Start Game</button>
</div>

<div id="hud" style="display:none">
    <p>Scor Albine: <span id="preyScore">0</span></p>
    <p>Scor Prădători: <span id="predatorScore">0</span></p>
    <p>Număr Prădători: <span id="numPredators">3</span></p>
    <div id="colorScores"></div>
    <div id="goalRanking"></div>
</div>

<button id="restartButton" style="display:none" onclick="restartGame()">Restart Game</button>
<button id="freezeButton" onclick="freezeBlackParticles()">Freeze Predators</button>

<div id="joystickContainer">
    <div id="joystick">
        <div id="joystickThumb"></div>
    </div>
</div>

<canvas id="particleCanvas"></canvas>

<script>
    let mouse = {
        x: null,
        y: null
    };

    let joystick = {
        x: 0,
        y: 0
    };

    let role = "bees"; // Default role

    const joystickContainer = document.getElementById('joystickContainer');
    const joystickThumb = document.getElementById('joystickThumb');
    let joystickActive = false;

    joystickContainer.addEventListener('touchstart', (e) => {
        joystickActive = true;
        joystickContainer.style.display = 'block';
        updateJoystick(e);
    });

    joystickContainer.addEventListener('touchmove', (e) => {
        if (joystickActive) {
            updateJoystick(e);
        }
    });

    joystickContainer.addEventListener('touchend', () => {
        joystickActive = false;
        joystickContainer.style.display = 'none';
        joystick.x = 0;
        joystick.y = 0;
    });

    function updateJoystick(event) {
        const rect = joystickContainer.getBoundingClientRect();
        const touch = event.touches[0];
        const centerX = rect.left + rect.width / 2;
        const centerY = rect.top + rect.height / 2;
        const dx = touch.clientX - centerX;
        const dy = touch.clientY - centerY;
        const distance = Math.sqrt(dx * dx + dy * dy);
        const maxDistance = rect.width / 2;

        if (distance > maxDistance) {
            const angle = Math.atan2(dy, dx);
            joystick.x = Math.cos(angle) * maxDistance;
            joystick.y = Math.sin(angle) * maxDistance;
        } else {
            joystick.x = dx;
            joystick.y = dy;
        }

        joystickThumb.style.transform = `translate(${joystick.x - joystickThumb.offsetWidth / 2}px, ${joystick.y - joystickThumb.offsetHeight / 2}px)`;
    }

    window.addEventListener('touchstart', (e) => {
        joystickContainer.style.display = 'block';
        updateJoystick(e);
    });

    window.addEventListener('touchend', () => {
        joystickContainer.style.display = 'none';
    });

    const canvas = document.getElementById('particleCanvas');
    const ctx = canvas.getContext('2d');
    let particlesArray = [];
    let blackParticles = [];
    let controlledBlackParticle;
    let goldenCircle;
    let hive;
    let preyScore = 0;
    let predatorScore = 0;
    let freezeActive = false;
    let freezeEnergy = 0;
    let gameOver = false;

    const particleColors = [
        { name: 'Roșu', rgba: 'rgba(255, 0, 0, 0.5)' },
        { name: 'Verde', rgba: 'rgba(0, 128, 0, 0.5)' },
        { name: 'Albastru', rgba: 'rgba(0, 0, 255, 0.5)' },
        { name: 'Portocaliu', rgba: 'rgba(255, 165, 0, 0.5)' },
        { name: 'Mov', rgba: 'rgba(128, 0, 128, 0.5)' },
        { name: 'Cyan', rgba: 'rgba(0, 255, 255, 0.5)' }
    ];

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
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2, false);
            ctx.fillStyle = 'gold';
            ctx.fill();
        }
    }

    class BlackParticle {
        constructor(x, y, size) {
            this.x = x;
            this.y = y;
            this.size = size;
            this.speed = 1.5;
            this.frozen = false;
            this.controlled = false;
        }

        draw() {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2, false);
            ctx.fillStyle = this.frozen ? 'gray' : (this.controlled ? 'blue' : 'black');
            ctx.fill();
        }

        update() {
            if (this.frozen || gameOver) return;

            if (this.controlled) {
                const dx = joystick.x / 50;
                const dy = joystick.y / 50;
                this.x += dx * this.speed;
                this.y += dy * this.speed;
            }

            this.draw();
        }
    }

    // Rest of the code remains unchanged

    function init() {
        particlesArray = [];
        blackParticles = [];

        const numParticles = parseInt(document.getElementById('numParticles').value) || 500;
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
    }

    function animate() {
        if (gameOver) return;

        ctx.clearRect(0, 0, canvas.width, canvas.height);
        goldenCircle.draw();
        hive.draw();
        blackParticles.forEach(blackParticle => blackParticle.update());
        particlesArray.forEach(particle => particle.update());
        requestAnimationFrame(animate);
    }

    function startGame() {
        role = document.getElementById('role').value;
        document.getElementById('menu').style.display = 'none';
        document.getElementById('hud').style.display = 'block';
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
        init();
        animate();
    }

    window.addEventListener('resize', function() {
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
    });

    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
</script>

</body>
</html>
