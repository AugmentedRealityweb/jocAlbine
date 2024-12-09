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

<canvas id="particleCanvas"></canvas>

<script>
    let mouse = {
        x: null,
        y: null
    };

    let role = "bees"; // Default role

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
                const dx = mouse.x - this.x;
                const dy = mouse.y - this.y;
                const distance = Math.sqrt(dx * dx + dy * dy);
                if (distance > 1) {
                    this.x += (dx / distance) * this.speed;
                    this.y += (dy / distance) * this.speed;
                }

                // Check collision with honey particles
                particlesArray.forEach(particle => {
                    if (particle.hasHoney) {
                        const dx = particle.x - this.x;
                        const dy = particle.y - this.y;
                        const distance = Math.sqrt(dx * dx + dy * dy);
                        if (distance < this.size) {
                            const index = particlesArray.indexOf(particle);
                            if (index > -1) {
                                particlesArray.splice(index, 1);
                                predatorScore++;
                                document.getElementById('predatorScore').textContent = predatorScore;
                                checkGameOver();
                            }
                        }
                    }
                });
            } else {
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
                if (role === "bees") {
                    const dx = mouse.x - this.x;
                    const dy = mouse.y - this.y;
                    const distance = Math.sqrt(dx * dx + dy * dy);

                    if (distance > 1) {
                        this.x += (dx / distance) * 2;
                        this.y += (dy / distance) * 2;
                    }
                } else {
                    const dx = hive.x - this.x;
                    const dy = hive.y - this.y;
                    const distance = Math.sqrt(dx * dx + dy * dy);

                    if (distance > 1) {
                        this.x += (dx / distance) * 1.5;
                        this.y += (dy / distance) * 1.5;
                    }
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
    const distance = Math.sqrt(dx * dx + dy * dy);
    const maxDistance = rect.width / 2;

    if (distance < maxDistance) {
        joystick.style.transform = `translate(${dx}px, ${dy}px)`;
        joystickPosition.x = dx / maxDistance;
        joystickPosition.y = dy / maxDistance;
    } else {
        const angle = Math.atan2(dy, dx);
        const limitedX = Math.cos(angle) * maxDistance;
        const limitedY = Math.sin(angle) * maxDistance;
        joystick.style.transform = `translate(${limitedX}px, ${limitedY}px)`;
        joystickPosition.x = limitedX / maxDistance;
        joystickPosition.y = limitedY / maxDistance;
    }
}

function handleTouchEnd() {
    isDragging = false;
    joystick.style.transform = "translate(-50%, -50%)";
    joystickPosition.x = 0;
    joystickPosition.y = 0;
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
