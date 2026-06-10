<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Neo Dash</title>
    <style>
        body {
            margin: 0;
            background-color: #0a0a16;
            color: #fff;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            overflow: hidden;
        }
        h1 {
            margin: 5px 0;
            font-size: 24px;
            letter-spacing: 2px;
            text-shadow: 0 0 10px #00ffff;
        }
        #gameContainer {
            position: relative;
            box-shadow: 0 0 30px rgba(0, 255, 255, 0.2);
            border-radius: 4px;
            overflow: hidden;
        }
        canvas {
            background: linear-gradient(to bottom, #111126, #070714);
            display: block;
        }
        .instructions {
            margin-top: 10px;
            color: #8a8ab0;
            font-size: 14px;
        }
    </style>
</head>
<body>

    <h1>NEO DASH</h1>
    <div id="gameContainer">
        <canvas id="gameCanvas" width="800" height="400"></canvas>
    </div>
    <div class="instructions">Press <strong>SPACEBAR</strong> or <strong>CLICK</strong> to Jump</div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        // Game Configuration & State
        const GRAVITY = 1.4;
        let gameActive = true;
        let gameWon = false;
        let spacePressed = false;

        // Level length setup
        const LEVEL_WIDTH = 6000; 

        // Player properties
        const player = {
            x: 100,
            y: 300,
            size: 30,
            vy: 0,
            jumpForce: -16,
            isGrounded: false,
            rotation: 0,
            color: '#00ffff'
        };

        // Floor level
        const FLOOR_Y = canvas.height - 70;

        // Generate level obstacles (X positions of spikes)
        // Spikes are placed relative to the map blueprint
        const spikePositions = [
            500, 800, 1100, 1400, 1440, 1800, 2100, 2140, 2180, 
            2500, 2800, 3100, 3150, 3500, 3800, 3840, 4200, 4500, 
            4800, 4840, 4880, 5200, 5500
        ];

        let obstacles = [];
        function initObstacles() {
            obstacles = spikePositions.map(posX => ({
                x: posX,
                y: FLOOR_Y,
                width: 30,
                height: 34
            }));
        }
        initObstacles();

        // Camera offset to scroll screen
        let cameraX = 0;

        // Listeners for Controls
        window.addEventListener('keydown', (e) => {
            if (e.code === 'Space') {
                spacePressed = true;
                if (!gameActive) restartGame();
            }
        });

        window.addEventListener('keyup', (e) => {
            if (e.code === 'Space') spacePressed = false;
        });

        canvas.addEventListener('mousedown', () => {
            spacePressed = true;
            if (!gameActive) restartGame();
        });

        canvas.addEventListener('mouseup', () => {
            spacePressed = false;
        });

        function restartGame() {
            player.x = 100;
            player.y = FLOOR_Y - player.size;
            player.vy = 0;
            player.rotation = 0;
            cameraX = 0;
            gameActive = true;
            gameWon = false;
            initObstacles();
        }

        // Check triangle-to-box collision
        function checkCollision(rect, triangle) {
            // Simple bounding box check for performance/ease
            return rect.x < triangle.x + triangle.width &&
                   rect.x + rect.size > triangle.x &&
                   rect.y < triangle.y &&
                   rect.y + rect.size > triangle.y - triangle.height;
        }

        // Main Loop
        function update() {
            if (gameActive && !gameWon) {
                // Move player forward continuously
                player.x += 6.5; 
                cameraX = player.x - 100; // Track camera

                // Apply Physics
                player.vy += GRAVITY;
                player.y += player.vy;

                // Floor collision
                if (player.y >= FLOOR_Y - player.size) {
                    player.y = FLOOR_Y - player.size;
                    player.vy = 0;
                    player.isGrounded = true;
                    
                    // Snap rotation to nearest 90 degrees when landing
                    player.rotation = Math.round(player.rotation / (Math.PI / 2)) * (Math.PI / 2);
                } else {
                    player.isGrounded = false;
                }

                // Jump input handling (allows holding down button to bunny-hop)
                if (spacePressed && player.isGrounded) {
                    player.vy = player.jumpForce;
                    player.isGrounded = false;
                }

                // Spin character when in the air
                if (!player.isGrounded) {
                    player.rotation += 0.09;
                }

                // Check Win Condition
                if (player.x >= LEVEL_WIDTH) {
                    gameWon = true;
                }

                // Check Collisions
                for (let obs of obstacles) {
                    if (checkCollision(player, obs)) {
                        gameActive = false; // Crash!
                    }
                }
            }
        }

        function draw() {
            // Clear Screen
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Draw Parallax Grid Background lines
            ctx.strokeStyle = '#1d1d3d';
            ctx.lineWidth = 1;
            let startGridX = Math.floor(cameraX / 40) * 40;
            for (let x = startGridX; x < startGridX + canvas.width + 40; x += 40) {
                ctx.beginPath();
                ctx.moveTo(x - cameraX, 0);
                ctx.lineTo(x - cameraX, FLOOR_Y);
                ctx.stroke();
            }

            // Draw Ground Floor
            ctx.fillStyle = '#08081a';
            ctx.fillRect(0, FLOOR_Y, canvas.width, canvas.height - FLOOR_Y);
            ctx.strokeStyle = '#ff007f';
            ctx.lineWidth = 4;
            ctx.beginPath();
            ctx.moveTo(0, FLOOR_Y);
            ctx.lineTo(canvas.width, FLOOR_Y);
            ctx.stroke();

            // Draw Obstacles (Spikes)
            ctx.fillStyle = '#ff0055';
            for (let obs of obstacles) {
                // Only draw visible objects
                if (obs.x * player.x > -100) { 
                    ctx.beginPath();
                    ctx.moveTo(obs.x - cameraX, obs.y);
                    ctx.lineTo((obs.x + obs.width / 2) - cameraX, obs.y - obs.height);
                    ctx.lineTo((obs.x + obs.width) - cameraX, obs.y);
                    ctx.closePath();
                    ctx.fill();
                    
                    // Neon spike glow lines
                    ctx.strokeStyle = '#ffffff';
                    ctx.lineWidth = 1;
                    ctx.stroke();
                }
            }

            // Draw Player with Rotation matrix
            ctx.save();
            // Move origin to player center
            ctx.translate(player.x + player.size / 2 - cameraX, player.y + player.size / 2);
            ctx.rotate(player.rotation);
            
            // Draw square face body
            ctx.fillStyle = player.color;
            ctx.fillRect(-player.size / 2, -player.size / 2, player.size, player.size);
            
            // Neon inner details
            ctx.strokeStyle = '#ffffff';
            ctx.lineWidth = 3;
            ctx.strokeRect(-player.size / 3, -player.size / 3, (player.size / 3) * 2, (player.size / 3) * 2);
            ctx.restore();

            // Draw Progress Bar Percentage
            let progress = Math.min(100, Math.floor((player.x / LEVEL_WIDTH) * 100));
            ctx.fillStyle = '#ffffff';
            ctx.font = 'bold 20px sans-serif';
            ctx.textAlign = 'center';
            ctx.fillText(progress + '%', canvas.width / 2, 40);

            // UI Text Overlays
            if (!gameActive) {
                ctx.fillStyle = 'rgba(0, 0, 0, 0.75)';
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                ctx.fillStyle = '#ff0055';
                ctx.font = 'bold 40px sans-serif';
                ctx.fillText('CRASHED!', canvas.width / 2, canvas.height / 2 - 10);
                ctx.fillStyle = '#ffffff';
                ctx.font = '18px sans-serif';
                ctx.fillText('Press SPACEBAR or CLICK to try again', canvas.width / 2, canvas.height / 2 + 30);
            }

            if (gameWon) {
                ctx.fillStyle = 'rgba(0, 0, 0, 0.75)';
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                ctx.fillStyle = '#00ffaa';
                ctx.font = 'bold 40px sans-serif';
                ctx.fillText('LEVEL COMPLETE!', canvas.width / 2, canvas.height / 2 - 10);
                ctx.fillStyle = '#ffffff';
                ctx.font = '18px sans-serif';
                ctx.fillText('Press SPACEBAR or CLICK to play again', canvas.width / 2, canvas.height / 2 + 30);
            }
        }

        // Engine Loop execution
        function loop() {
            update();
            draw();
            requestAnimationFrame(loop);
        }

        loop();
    </script>
</body>
</html>
