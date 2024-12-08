
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
            background-color: #fff;
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
            z-index: -1;
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
    </style>
</head>
<body>

<div id="menu">
    <h2>Particle Simulation Settings</h2>
    <p>Alege o culoare și vezi dacă aceea va domina în final. Setează dificultatea pentru a ajusta viteza particulelor.</p>
    <label for="color">Choose your color:</label>
    <select id="color">
        <option value="rgba(255, 0, 0, 0.5)">Roșu</option>
        <option value="rgba(0, 128, 0, 0.5)">Verde</option>
        <option value="rgba(0, 0, 255, 0.5)">Albastru</option>
        <option value="rgba(255, 165, 0, 0.5)">Portocaliu</option>
        <option value="rgba(128, 0, 128, 0.5)">Mov</option>
        <option value="rgba(255, 192, 203, 0.5)">Roz</option>
        <option value="rgba(0, 255, 255, 0.5)">Cyan</option>
    </select>
    <br><br>
    <label for="numParticles">Number of initial particles:</label>
    <input type="number" id="numParticles" min="10" max="200" value="50">
    <br><br>
    <label for="difficulty">Difficulty:</label>
    <select id="difficulty" onchange="showDifficultyDescription()">
        <option value="easy">Easy</option>
        <option value="medium">Medium</option>
        <option value="hard">Hard</option>
    </select>
    <p id="difficultyDescription">Easy = viteza redusă a particulelor; Hard = viteza crescută.</p>
    <br><br>
    <button onclick="startGame()">Start Game</button>
</div>

<div id="hud" style="display:none">
    <div id="topPerformers"></div>
</div>

<button id="restartButton" style="display:none" onclick="restartGame()">Restart Game</button>

<canvas id="particleCanvas"></canvas>

<script>
    const canvas = document.getElementById('particleCanvas');
    const ctx = canvas.getContext('2d');
    let particlesArray = [];
    let whiteParticle;
    let blackParticles = [];
    let particleColors = [
        'rgba(255, 0, 0, 0.5)',
        'rgba(0, 128, 0, 0.5)',
        'rgba(0, 0, 255, 0.5)',
        'rgba(255, 165, 0, 0.5)',
        'rgba(128, 0, 128, 0.5)',
        'rgba(255, 192, 203, 0.5)',
        'rgba(0, 255, 255, 0.5)'
    ];
    let usedColors = new Set();
    let gameInterval;

    window.addEventListener('resize', function() {
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
        init(particlesArray.length);
    });

    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;

    class Particle {
        constructor(x, y, directionX, directionY, size, color) {
            this.x = x;
            this.y = y;
            this.directionX = directionX;
            this.directionY = directionY;
            this.size = size;
            this.color = color;
        }
        draw() {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2, false);
            ctx.fillStyle = this.color;
            ctx.fill();
        }
        update() {
            if (this.x + this.size > canvas.width || this.x - this.size < 0) {
                this.directionX = -this.directionX;
            }
            if (this.y + this.size > canvas.height || this.y - this.size < 0) {
                this.directionY = -this.directionY;
            }

            this.x += this.directionX;
            this.y += this.directionY;

            this.draw();
        }
    }

    class BlackParticle {
        constructor(x, y, size, targetColor) {
            this.x = x;
            this.y = y;
            this.size = size;
            this.targetColor = targetColor;
            this.speed = 5;
        }
        draw() {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2, false);
            ctx.fillStyle = 'black';
            ctx.fill();
        }
        update() {
            let closestParticle = null;
            let minDistance = Infinity;

            for (const particle of particlesArray) {
                if (particle.color === this.targetColor) {
                    const dx = this.x - particle.x;
                    const dy = this.y - particle.y;
                    const distance = Math.sqrt(dx * dx + dy * dy);

                    if (distance < minDistance) {
                        minDistance = distance;
                        closestParticle = particle;
                    }
                }
            }

            if (closestParticle) {
                const dx = closestParticle.x - this.x;
                const dy = closestParticle.y - this.y;
                const angle = Math.atan2(dy, dx);

                this.x += Math.cos(angle) * this.speed;
                this.y += Math.sin(angle) * this.speed;

                if (minDistance < this.size + closestParticle.size) {
                    const index = particlesArray.indexOf(closestParticle);
                    if (index > -1) {
                        particlesArray.splice(index, 1);
                    }
                }
            }

            this.draw();
        }
    }

    class WhiteParticle {
        constructor(x, y, size) {
            this.x = x;
            this.y = y;
            this.size = size;
            this.angle = 0;
        }
        draw() {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2, false);
            ctx.fillStyle = 'white';
            ctx.fill();
        }
        update() {
            this.angle += 0.1;
            const xOffset = Math.cos(this.angle) * this.size * 1.5;
            const yOffset = Math.sin(this.angle) * this.size * 1.5;

            const size = Math.random() * 3 + 1;
            const directionX = (Math.random() - 0.5);
            const directionY = (Math.random() - 0.5);
            const color = particleColors[Math.floor(Math.random() * particleColors.length)];

            particlesArray.push(new Particle(this.x - xOffset, this.y - yOffset, directionX, directionY, size, color));

            this.draw();
        }
    }

    function adjustDifficulty(value) {
        switch (value) {
            case 'easy':
                return 0.5;
            case 'medium':
                return 1;
            case 'hard':
                return 2;
            default:
                return 1;
        }
    }

    function spawnNewBlackParticle() {
        const availableColors = particleColors.filter(color => !usedColors.has(color));
        if (availableColors.length > 0) {
            const newTargetColor = availableColors[Math.floor(Math.random() * availableColors.length)];
            usedColors.add(newTargetColor);
            const newBlackParticle = new BlackParticle(canvas.width / 2, canvas.height / 2, 10, newTargetColor);
            blackParticles.push(newBlackParticle);
        }
    }

    function init(numParticles) {
        particlesArray = [];
        blackParticles = [];
        usedColors = new Set();
        const difficulty = adjustDifficulty(document.getElementById('difficulty').value);
        targetColor = document.getElementById('color').value;
        usedColors.add(targetColor);
        blackParticles.push(new BlackParticle(canvas.width / 2, canvas.height / 2, 10, targetColor));
        whiteParticle = new WhiteParticle(canvas.width / 2, canvas.height / 2, 10);

        for (let i = 0; i < numParticles; i++) {
            const size = Math.random() * 3 + 1;
            const x = Math.random() * (canvas.width - size * 2);
            const y = Math.random() * (canvas.height - size * 2);
            const directionX = (Math.random() - 0.5) * difficulty;
            const directionY = (Math.random() - 0.5) * difficulty;
            const color = particleColors[Math.floor(Math.random() * particleColors.length)];
            particlesArray.push(new Particle(x, y, directionX, directionY, size, color));
        }
        setInterval(spawnNewBlackParticle, 10000);
        document.getElementById('menu').style.display = 'none';
    }

    function animate() {
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        for (let i = 0; i < blackParticles.length; i++) {
            blackParticles[i].update();
        }
        whiteParticle.update();
        for (let i = 0; i < particlesArray.length; i++) {
            particlesArray[i].update();
        }
        gameInterval = requestAnimationFrame(animate);
    }

    function startGame() {
        const numParticles = parseInt(document.getElementById('numParticles').value);
        if (isNaN(numParticles) || numParticles < 10 || numParticles > 5000) {
            alert('Please enter a valid number of particles (10-5000).');
            return;
        }
        init(numParticles);
        animate();
    }

    function restartGame() {
        cancelAnimationFrame(gameInterval);
        document.getElementById('menu').style.display = 'block';
    }
</script>

</body>
</html>
