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
    <p>Scor Albine: <span id="preyScore">0</span></p>
    <p>Scor Prădători: <span id="predatorScore">0</span></p>
    <p>Număr Prădători: <span id="numPredators">3</span></p>
    <div id="colorScores"></div>
    <div id="goalRanking"></div>
</div>

<button id="restartButton" style="display:none" onclick="restartGame()">Restart Game</button>
<button id="freezeButton" onclick="freezeBlackParticles()">Freeze Predators</button>

<canvas id="particleCanvas"></canvas>

<script>
    let mouse = {
        x: null,
        y: null
    };

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
            this.originalSize = size;
            this.speed = 1.5;
            this.frozen = false;
        }

        draw() {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2, false);
            ctx.fillStyle = this.frozen ? 'gray' : 'black';
            ctx.fill();
        }

        adjustSizeForDevice() {
            if (window.innerWidth < 768) { // Mobile devices
                this.size = this.originalSize * 0.5;
            } else {
                this.size = this.originalSize;
            }
        }

        update() {
            if (this.frozen || gameOver) return;

            let closestParticle = null;
            let minDistance = Infinity;

            particlesArray.forEach(particle => {
                if (particle.hasHoney) {
                    const dx = particle.x - this.x;
                    const dy = particle.y - this.y;
                    const distance = Math.sqrt(dx * dx + dy * dy);

                    if (distance < minDistance) {
                        minDistance = distance;
                        closestParticle = particle;
                    }
                }
            });

            const dxToGoldenCircle = this.x - goldenCircle.x;
            const dyToGoldenCircle = this.y - goldenCircle.y;
            const distanceToGoldenCircle = Math.sqrt(dxToGoldenCircle * dxToGoldenCircle + dyToGoldenCircle * dyToGoldenCircle);

            if (distanceToGoldenCircle < goldenCircle.radius + this.size) {
                const angle = Math.atan2(dyToGoldenCircle, dxToGoldenCircle);
                this.x += Math.cos(angle) * this.speed;
                this.y += Math.sin(angle) * this.speed;
                return;
            }

            if (closestParticle) {
                const dx = closestParticle.x - this.x;
                const dy = closestParticle.y - this.y;
                const distance = Math.sqrt(dx * dx + dy * dy);

                if (distance < this.size) {
                    const index = particlesArray.indexOf(closestParticle);
                    if (index > -1) {
                        particlesArray.splice(index, 1);
                        predatorScore++;
                        document.getElementById('predatorScore').textContent = predatorScore;
                        checkGameOver();
                    }
                } else {
                    this.x += (dx / distance) * this.speed;
                    this.y += (dy / distance) * this.speed;
                }
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
            this.wobbleAngle = 0;
        }

        draw() {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2, false);
            ctx.fillStyle = this.hasHoney ? 'gold' : this.originalColor;
            ctx.fill();
        }

        update() {
            if (!this.hasHoney) {
                const dx = goldenCircle.x - this.x;
                const dy = goldenCircle.y - this.y;
                const distance = Math.sqrt(dx * dx + dy * dy);

                if (distance < goldenCircle.radius) {
                    this.hasHoney = true;
                }
            } else {
                const dx = mouse.x - this.x;
                const dy = mouse.y - this.y;
                const distance = Math.sqrt(dx * dx + dy * dy);

                if (distance > 1) {
                    const speed = 2;
                    this.wobbleAngle += 0.1;
                    const wobbleX = Math.sin(this.wobbleAngle) * 2;
                    const wobbleY = Math.cos(this.wobbleAngle) * 2;
                    this.x += (dx / distance) * speed + wobbleX;
                    this.y += (dy / distance) * speed + wobbleY;
                }

                if (Math.sqrt((this.x - hive.x) ** 2 + (this.y - hive.y) ** 2) < hive.size) {
                    this.hasHoney = false;
                    preyScore++;
                    document.getElementById('preyScore').textContent = preyScore;
                    freezeEnergy += 10;

                    if (preyScore % 100 === 0) {
                        hive.relocate();
                    }
                    if (freezeEnergy >= 1000) {
                        document.getElementById('freezeButton').style.display = 'block';
                    }
                    checkGameOver();
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

    function checkGameOver() {
        if (preyScore >= 4500) {
            alert('Game Over! Bees Win!');
            gameOver = true;
        } else if (predatorScore >= 4500) {
            alert('Game Over! Predators Win!');
            gameOver = true;
        }
    }

    function freezeBlackParticles() {
        if (freezeActive || freezeEnergy < 1000) return;

        freezeActive = true;
        freezeEnergy = 0;
        document.getElementById('freezeButton').style.display = 'none';

        blackParticles.forEach(particle => particle.frozen = true);
        setTimeout(() => {
            blackParticles.forEach(particle => particle.frozen = false);
            freezeActive = false;
        }, 5000);
    }

    function init() {
        particlesArray = [];
        blackParticles = [];

        const numParticles = parseInt(document.getElementById('numParticles').value) || 50;
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
            blackParticle.adjustSizeForDevice();
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
        blackParticles.forEach(particle => particle.adjustSizeForDevice());
    });

    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
</script>

</body>
</html>
