
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

<canvas id="particleCanvas"></canvas>

<script>
    const canvas = document.getElementById('particleCanvas');
    const ctx = canvas.getContext('2d');
    let particlesArray = [];
    let goals = [];
    let blackParticles = [];
    let ants = [];
    let particleColors = [
        { name: 'Roșu', rgba: 'rgba(255, 0, 0, 0.5)' },
        { name: 'Verde', rgba: 'rgba(0, 128, 0, 0.5)' },
        { name: 'Albastru', rgba: 'rgba(0, 0, 255, 0.5)' },
        { name: 'Portocaliu', rgba: 'rgba(255, 165, 0, 0.5)' },
        { name: 'Mov', rgba: 'rgba(128, 0, 128, 0.5)' },
        { name: 'Cyan', rgba: 'rgba(0, 255, 255, 0.5)' }
    ];
    let preyScore = 0;
    let predatorScore = 0;
    let colorScores = {};
    let goldenCircle = { x: canvas.width / 2, y: canvas.height / 2, radius: 100, points: 10000 };

    class Goal {
        constructor(x, y, color) {
            this.x = x;
            this.y = y;
            this.color = color;
            this.size = 20;
            this.hits = 0;
            this.score = 0;
        }
        draw() {
            const hexRadius = this.size;
            ctx.beginPath();
            for (let i = 0; i < 6; i++) {
                const angle = Math.PI / 3 * i;
                const x = this.x + hexRadius * Math.cos(angle);
                const y = this.y + hexRadius * Math.sin(angle);
                if (i === 0) {
                    ctx.moveTo(x, y);
                } else {
                    ctx.lineTo(x, y);
                }
            }
            ctx.closePath();
            ctx.fillStyle = this.color.rgba;
            ctx.fill();
        }
        decreaseSize() {
            this.hits++;
            if (this.hits >= 3) {
                this.size = Math.max(10, this.size / 2);
                this.hits = 0;
            }
        }
        increaseSize() {
            this.size += 0.05;
            this.score += 0.05;
        }
        relocate() {
            let newX, newY;
            do {
                newX = Math.random() * (canvas.width - 100) + 50;
                newY = Math.random() * (canvas.height - 100) + 50;
            } while (Math.sqrt((newX - goldenCircle.x) ** 2 + (newY - goldenCircle.y) ** 2) < goldenCircle.radius + 900);

            this.x = newX;
            this.y = newY;
        }
    }

    function relocateGoalsPeriodically() {
        setInterval(() => {
            goals.forEach(goal => {
                goal.relocate();
            });
        }, 10000);
    }

    class Ant {
        constructor(x, y, speed) {
            this.x = x;
            this.y = y;
            this.speed = speed;
            this.target = null;
        }

        draw() {
            ctx.beginPath();
            ctx.arc(this.x, this.y, 5, 0, Math.PI * 2, false);
            ctx.fillStyle = 'brown';
            ctx.fill();
        }

        update() {
            if (!this.target || this.target.size <= 10) {
                this.target = goals[Math.floor(Math.random() * goals.length)];
            }

            const dx = this.target.x - this.x;
            const dy = this.target.y - this.y;
            const distance = Math.sqrt(dx * dx + dy * dy);

            if (distance < this.target.size) {
                this.target.decreaseSize();
                this.target = null;
            } else {
                this.x += (dx / distance) * this.speed;
                this.y += (dy / distance) * this.speed;
            }

            blackParticles.forEach(blackParticle => {
                const dx = blackParticle.x - this.x;
                const dy = blackParticle.y - this.y;
                const distance = Math.sqrt(dx * dx + dy * dy);

                if (distance < blackParticle.size) {
                    blackParticle.paralyze();
                }
            });

            this.draw();
        }
    }

    class GoldenCircle {
        constructor(x, y, radius, points) {
            this.x = x;
            this.y = y;
            this.radius = radius;
            this.points = points;
        }

        draw() {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2, false);
            ctx.fillStyle = 'gold';
            ctx.fill();
        }

        shrink() {
            if (this.points > 0) {
                this.points -= 0.05;
            }
        }
    }

    class BlackParticle {
        constructor(x, y, size) {
            this.x = x;
            this.y = y;
            this.size = size;
            this.speed = 1.5;
            this.paralyzed = false;
            this.paralyzeTimer = 0;
        }

        draw() {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2, false);
            ctx.fillStyle = this.paralyzed ? 'gray' : 'black';
            ctx.fill();
        }

        paralyze() {
            this.paralyzed = true;
            this.paralyzeTimer = 300; // 5 seconds at 60 FPS
        }

        update() {
            if (this.paralyzed) {
                this.paralyzeTimer--;
                if (this.paralyzeTimer <= 0) {
                    this.paralyzed = false;
                }
                this.draw();
                return;
            }

            let closestParticle = null;
            let minDistance = Infinity;

            particlesArray.forEach(particle => {
                if (!particle.hasHoney) {
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
                    }
                } else {
                    this.x += (dx / distance) * this.speed;
                    this.y += (dy / distance) * this.speed;
                }
            }

            this.draw();
        }
    }

    window.addEventListener('resize', function() {
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
        goldenCircle = new GoldenCircle(canvas.width / 2, canvas.height / 2, 100, 10000);
        init(particlesArray.length);
    });

    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;

    const goldenCircleInstance = new GoldenCircle(canvas.width / 2, canvas.height / 2, 100, 10000);

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
            this.x += this.directionX;
            this.y += this.directionY;

            if (this.x + this.size > canvas.width || this.x - this.size < 0) {
                this.directionX = -this.directionX;
            }
            if (this.y + this.size > canvas.height || this.y - this.size < 0) {
                this.directionY = -this.directionY;
            }

            const dx = goldenCircleInstance.x - this.x;
            const dy = goldenCircleInstance.y - this.y;
            const distance = Math.sqrt(dx * dx + dy * dy);

            if (!this.hasHoney && distance < goldenCircleInstance.radius) {
                this.hasHoney = true;
                goldenCircleInstance.shrink();
            }

            if (this.hasHoney) {
                const goal = goals.find(goal => goal.color.rgba === this.color.rgba);
                const goalDx = goal.x - this.x;
                const goalDy = goal.y - this.y;
                const goalDistance = Math.sqrt(goalDx * goalDx + goalDy * goalDy);

                if (goalDistance < this.size) {
                    goal.increaseSize();
                    preyScore++;
                    document.getElementById('preyScore').textContent = preyScore;
                    this.hasHoney = false;

                    // Deposit honey around the goal
                    ctx.beginPath();
                    ctx.arc(goal.x + Math.random() * 10 - 5, goal.y + Math.random() * 10 - 5, 3, 0, Math.PI * 2, false);
                    ctx.fillStyle = 'gold';
                    ctx.fill();
                } else {
                    this.directionX = goalDx / goalDistance;
                    this.directionY = goalDy / goalDistance;
                }
            }

            this.draw();
        }
    }

    function createGoals() {
        goals = [];
        const margin = 50;
        const positions = [
            { x: margin, y: margin },
            { x: canvas.width - margin, y: margin },
            { x: margin, y: canvas.height - margin },
            { x: canvas.width - margin, y: canvas.height - margin },
            { x: canvas.width / 2, y: margin },
            { x: canvas.width / 2, y: canvas.height - margin }
        ];

        particleColors.forEach((color, index) => {
            const position = positions[index % positions.length];
            goals.push(new Goal(position.x, position.y, color));
        });
    }

    function drawGoals() {
        goals.forEach(goal => goal.draw());
    }

    function updateGoalRanking() {
        const sortedGoals = goals.slice().sort((a, b) => b.size - a.size);
        const rankingDiv = document.getElementById('goalRanking');
        rankingDiv.innerHTML = '<h3>Clasament Pătrate:</h3>' + sortedGoals.map(goal => `${goal.color.name}: ${goal.size.toFixed(2)}`).join('<br>');
    }

    function init(numParticles) {
        particlesArray = [];
        blackParticles = [];
        ants = [];
        colorScores = {};
        createGoals();
        relocateGoalsPeriodically();

        for (let i = 0; i < numParticles; i++) {
            const size = Math.random() * 3 + 1;
            const x = Math.random() * (canvas.width - size * 2);
            const y = Math.random() * (canvas.height - size * 2);
            const directionX = (Math.random() - 0.5) * 2; // Adjusted random initial directions
            const directionY = (Math.random() - 0.5) * 2;
            const color = particleColors[Math.floor(Math.random() * particleColors.length)];
            particlesArray.push(new Particle(x, y, directionX, directionY, size, color));
        }

        // Create three black particles (predators)
        for (let i = 0; i < 3; i++) {
            const x = Math.random() * canvas.width;
            const y = Math.random() * canvas.height;
            blackParticles.push(new BlackParticle(x, y, 15));
        }

        // Create one ant
        const x = Math.random() * canvas.width;
        const y = Math.random() * canvas.height;
        ants.push(new Ant(x, y, 1));

        document.getElementById('menu').style.display = 'none';
        document.getElementById('hud').style.display = 'block';
    }

    function animate() {
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        goldenCircleInstance.draw();
        drawGoals();
        particlesArray.forEach(particle => particle.update());
        blackParticles.forEach(blackParticle => blackParticle.update());
        ants.forEach(ant => ant.update());
        updateGoalRanking();
        requestAnimationFrame(animate);

        if (particlesArray.length === 0) {
            alert('All particles have been eaten! Game over.');
            restartGame();
        }
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
