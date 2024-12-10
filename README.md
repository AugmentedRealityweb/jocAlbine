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
            /* Background image */
            background: url('https://img.freepik.com/premium-photo/background-honeycomb-is-made-honeycomb_947814-201763.jpg') no-repeat center center fixed;
            background-size: cover;
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

        /* Joystick styles */
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
        <option value="500">500</option>
        <option value="2000">2000</option>
        <option value="5000">5000</option>
    </select>
    <br /><br />
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

<!-- Joystick elements -->
<div id="joystickContainer" style="display:none">
    <div id="joystick"></div>
</div>

<canvas id="particleCanvas"></canvas>

<script>
    let isMobile = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent);
    let mouse = { x: null, y: null };
    let role = "bees"; // Default role

    // Un singur flag pentru bondarul uriaș
    let giantWaspAdded = false;

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

    // Culori pastelate moderne pentru particulele fara polen
    const particleColors = [
        { name: 'Pastel Pink', rgba: 'rgba(224, 118, 37, 0.8)' },
        { name: 'Pastel Orange', rgba: 'rgba(32, 98, 26, 0.8)' },
        { name: 'Pastel Yellow', rgba: 'rgba(34, 38, 170, 0.78)' },
        { name: 'Pastel Green', rgba: 'rgba(134, 2, 66, 0.78)' },
        { name: 'Pastel Blue', rgba: 'rgba(121, 10, 0, 0.78)' },
        { name: 'Pastel Purple', rgba: 'rgba(85, 17, 122, 0.94)' }
    ];

    const joystickPosition = { x: 0, y: 0 };

    // Imagine polen
    const pollenImage = new Image();
    pollenImage.src = 'https://clipart-library.com/images/5cRdXKbca.png';

    // Imagine bondar
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
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2, false);
            ctx.fillStyle = 'gold';
            ctx.fill();
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
            this.speed = 2; // viteza normală
            this.frozen = false;
            this.controlled = false;
            this.vx = 0;
            this.vy = 0;
        }

        draw() {
            ctx.drawImage(waspImage, this.x - this.size, this.y - this.size, this.size * 2, this.size * 2);
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
                    const distance = Math.sqrt(dx * dx + dy * dy);
                    if (distance > 1) {
                        this.vx = (dx / distance) * this.speed;
                        this.vy = (dy / distance) * this.speed;
                    } else {
                        this.vx = 0;
                        this.vy = 0;
                    }
                }

                this.x += this.vx;
                this.y += this.vy;

                particlesArray.forEach(particle => {
                    if (particle.hasHoney) {
                        // Bondarul nu mananca daca particula e in cercul auriu
                        if (!isInsideGoldenCircle(particle)) {
                            const distX = particle.x - this.x;
                            const distY = particle.y - this.y;
                            const distance = Math.sqrt(distX * distX + distY * distY);
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
                    }
                });

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

            } else {
                let closestParticle = null;
                let minDistance = Infinity;

                for (let particle of particlesArray) {
                    if (particle.hasHoney) {
                        const dx = particle.x - this.x;
                        const dy = particle.y - this.y;
                        const distance = Math.sqrt(dx * dx + dy * dy);
                        if (distance < minDistance) {
                            minDistance = distance;
                            closestParticle = particle;
                        }
                    }
                }

                if (closestParticle && !isInsideGoldenCircle(closestParticle)) {
                    // Daca particula cu miere nu e in cercul auriu, bondarul poate incerca sa o manance
                    const dx = closestParticle.x - this.x;
                    const dy = closestParticle.y - this.y;
                    const distance = Math.sqrt(dx * dx + dy * dy);

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
                        this.vx = (dx / distance) * this.speed;
                        this.vy = (dy / distance) * this.speed;
                    }
                } else {
                    // Nicio țintă sau ținta e in cerc auriu
                    this.vx = 0;
                    this.vy = 0;
                }

                this.x += this.vx;
                this.y += this.vy;

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
            if (this.hasHoney) {
                ctx.drawImage(pollenImage, this.x - this.size, this.y - this.size, this.size * 5, this.size * 5);
            } else {
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2, false);
                ctx.fillStyle = this.originalColor;
                ctx.fill();
            }
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

        if (freezeEnergy >= 1000) {
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
        if (preyScore >= 3000) {
            alert('Game Over! Bees Win!');
            gameOver = true;
        } else if (predatorScore >= 3000) {
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
        giantWaspAdded = false;

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
        if (isMobile && role === "wasps") {
            document.getElementById('joystickContainer').style.display = 'block';
        } else {
            document.getElementById('joystickContainer').style.display = 'none';
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

    function restartGame() {
        location.reload();
    }
</script>

</body>
</html>
