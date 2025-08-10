<!DOCTYPE html>
<html lang="en" >
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no" />
  <title>Speed Master - Modern Mobile Racer</title>
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@700&display=swap');

    html, body {
      margin: 0; padding: 0;
      background: #111;
      color: #0f0;
      font-family: 'Orbitron', monospace, monospace;
      overflow: hidden;
      user-select: none;
      -webkit-tap-highlight-color: transparent;
      height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
    }

    #container {
      width: 100vw;
      max-width: 380px;
      height: 100vh;
      max-height: 640px;
      position: relative;
      background: linear-gradient(180deg, #223322 0%, #112211 100%);
      border-radius: 20px;
      box-shadow: 0 0 30px #0f0;
      display: flex;
      flex-direction: column;
      align-items: center;
      user-select: none;
    }

    canvas {
      border-radius: 15px;
      background: linear-gradient(to top, #222, #444);
      margin-top: 10px;
      touch-action: none;
      display: block;
      max-width: 100%;
      box-shadow: 0 0 20px #0f0;
    }

    #score {
      font-size: 1.9rem;
      font-weight: 700;
      letter-spacing: 2px;
      margin: 8px 0 6px 0;
      text-shadow: 0 0 8px #0f0;
      user-select: none;
    }

    /* Start & GameOver overlays */
    .overlay {
      position: absolute;
      inset: 0;
      background: rgba(0, 0, 0, 0.92);
      color: #0f0;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      padding: 15px 20px;
      text-align: center;
      border-radius: 20px;
      user-select: none;
      z-index: 99;
    }

    .overlay.hidden {
      display: none;
    }

    .overlay h1 {
      font-size: 3rem;
      margin-bottom: 20px;
      text-shadow: 0 0 8px #0f0;
    }

    .overlay label {
      font-size: 1.2rem;
      margin-bottom: 8px;
    }

    .overlay input {
      padding: 12px 18px;
      font-size: 1.2rem;
      border-radius: 12px;
      border: 2px solid #0f0;
      background: #111;
      color: #0f0;
      font-weight: 700;
      width: 90%;
      max-width: 280px;
      margin-bottom: 18px;
      caret-color: #0f0;
      text-align: center;
      outline-offset: 2px;
      outline-color: #0f0;
      user-select: text;
    }

    .overlay button {
      padding: 14px 28px;
      font-size: 1.3rem;
      border-radius: 14px;
      border: none;
      background: #0a0;
      color: #eee;
      font-weight: 900;
      cursor: pointer;
      box-shadow: 0 0 12px #0f0;
      user-select: none;
      transition: background-color 0.3s ease;
      width: 90%;
      max-width: 280px;
    }
    .overlay button:hover {
      background: #0f0;
      color: #000;
    }

    .leaderboard-list {
      max-height: 170px;
      overflow-y: auto;
      width: 280px;
      margin-top: 20px;
      padding: 12px 18px;
      border: 2px solid #0f0;
      border-radius: 14px;
      text-align: left;
      font-weight: 700;
      font-size: 1.1rem;
      background: #111;
      user-select: none;
    }
    .leaderboard-list li {
      margin-bottom: 6px;
    }

    #instructions {
      font-size: 1rem;
      margin-top: 8px;
      color: #0a0;
      text-shadow: 0 0 5px #020;
      user-select:none;
    }
  </style>
</head>
<body>
  <div id="container">
    <div id="startOverlay" class="overlay">
      <h1>Speed Master</h1>
      <label for="playerNameInput">Enter Your Name</label>
      <input type="text" id="playerNameInput" maxlength="12" autocomplete="off" autocorrect="off" autocapitalize="words" spellcheck="false" placeholder="Your name" />
      <button id="startBtn">Start Race</button>
      <h2>Leaderboard</h2>
      <ol class="leaderboard-list" id="leaderboardList"></ol>
      <div id="instructions">Tap left or right side to move your car</div>
    </div>

    <canvas id="gameCanvas" width="360" height="540"></canvas>

    <div id="score">Score: 0</div>

    <div id="gameOverOverlay" class="overlay hidden">
      <h1>Game Over</h1>
      <p id="finalScoreText"></p>
      <button id="restartBtn">Play Again</button>
      <h2>Leaderboard</h2>
      <ol class="leaderboard-list" id="gameOverLeaderboard"></ol>
    </div>
  </div>

  <audio id="bgMusic" loop src="https://cdn.pixabay.com/download/audio/2022/03/29/audio_ebff882d41.mp3?filename=retro-game-loop-10870.mp3" crossorigin="anonymous"></audio>
  <audio id="crashSound" src="https://cdn.pixabay.com/download/audio/2022/03/23/audio_75bb4940e7.mp3?filename=explosion-2-6590.mp3" crossorigin="anonymous"></audio>

  <script>
    (() => {
      const container = document.getElementById('container');
      const canvas = document.getElementById('gameCanvas');
      const ctx = canvas.getContext('2d');

      const startOverlay = document.getElementById('startOverlay');
      const gameOverOverlay = document.getElementById('gameOverOverlay');

      const playerNameInput = document.getElementById('playerNameInput');
      const startBtn = document.getElementById('startBtn');
      const restartBtn = document.getElementById('restartBtn');

      const leaderboardList = document.getElementById('leaderboardList');
      const gameOverLeaderboard = document.getElementById('gameOverLeaderboard');

      const scoreDisplay = document.getElementById('score');
      const finalScoreText = document.getElementById('finalScoreText');

      const bgMusic = document.getElementById('bgMusic');
      const crashSound = document.getElementById('crashSound');

      const WIDTH = canvas.width;
      const HEIGHT = canvas.height;
      const laneCount = 3;
      const laneWidth = WIDTH / laneCount;

      const carWidth = 52;
      const carHeight = 90;
      const playerY = HEIGHT - carHeight - 25;

      // Car color unlock milestones, with more “realistic” colors & shading
      const unlocks = [
        {score: 0, color: '#2ecc71', name:'Basic Green'},        // green
        {score: 10, color: '#3498db', name:'Cool Blue'},          // blue
        {score: 25, color: '#e74c3c', name:'Sporty Red'},         // red
        {score: 50, color: '#f1c40f', name:'Flashy Yellow'},      // yellow
        {score: 75, color: '#9b59b6', name:'Royal Purple'},       // purple
        {score: 150, color: '#e67e22', name:'Bright Orange'},     // orange
        {score: 300, color: '#1abc9c', name:'Aqua Turbo'},        // turquoise
        {score: 500, color: '#34495e', name:'Steel Gray'},        // gray
        {score: 1000, color: '#ffffff', name:'Ultimate White'},   // white
      ];

      // Get current unlocked car color
      function getCurrentCar() {
        let current = unlocks[0];
        for (let u of unlocks) {
          if(score >= u.score) current = u;
          else break;
        }
        return current;
      }

      let playerLane = 1;
      let targetLane = 1;
      let playerX = laneToX(playerLane);

      let traffic = [];
      let speed = 3.5;
      let score = 0;
      let gameOver = false;
      let explosion = null;
      let playerName = '';

      // Touch input throttle
      let canMove = true;

      // Explosion particles
      class Particle {
        constructor(x, y) {
          this.x = x;
          this.y = y;
          this.radius = Math.random() * 6 + 4;
          this.color = `hsl(${Math.random() * 60 + 20}, 90%, 60%)`;
          this.speedX = (Math.random() - 0.5) * 10;
          this.speedY = (Math.random() - 0.5) * 10;
          this.alpha = 1;
          this.decay = Math.random() * 0.04 + 0.03;
        }
        update() {
          this.x += this.speedX;
          this.y += this.speedY;
          this.alpha -= this.decay;
          this.radius *= 0.94;
        }
        draw() {
          ctx.save();
          ctx.globalAlpha = this.alpha;
          ctx.fillStyle = this.color;
          ctx.beginPath();
          ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
          ctx.fill();
          ctx.restore();
        }
      }

      class Explosion {
        constructor(x, y) {
          this.particles = [];
          for(let i = 0; i < 35; i++) this.particles.push(new Particle(x, y));
          this.done = false;
        }
        update() {
          this.particles.forEach(p => p.update());
          this.particles = this.particles.filter(p => p.alpha > 0.05 && p.radius > 0.5);
          if(this.particles.length === 0) this.done = true;
        }
        draw() {
          this.particles.forEach(p => p.draw());
        }
      }

      // Helpers
      function laneToX(lane) {
        return lane * laneWidth + laneWidth / 2 - carWidth / 2;
      }

      // Draw a stylized, modern car with shading & gradient
      function drawCar(x, y, color) {
        // Create gradient for body
        let grad = ctx.createLinearGradient(x, y, x, y + carHeight);
        grad.addColorStop(0, shadeColor(color, -15));
        grad.addColorStop(0.5, color);
        grad.addColorStop(1, shadeColor(color, -40));
        ctx.fillStyle = grad;

        // Body shape
        ctx.beginPath();
        ctx.moveTo(x + 12, y);
        ctx.lineTo(x + carWidth - 12, y);
        ctx.quadraticCurveTo(x + carWidth - 2, y + 8, x + carWidth - 2, y + 30);
        ctx.lineTo(x + carWidth - 2, y + carHeight - 25);
        ctx.quadraticCurveTo(x + carWidth - 12, y + carHeight - 8, x + 12, y + carHeight - 8);
        ctx.lineTo(x + 2, y + carHeight - 25);
        ctx.lineTo(x + 2, y + 30);
        ctx.quadraticCurveTo(x + 2, y + 8, x + 12, y);
        ctx.closePath();
        ctx.fill();

        // Outline
        ctx.lineWidth = 2;
        ctx.strokeStyle = shadeColor(color, -50);
        ctx.stroke();

        // Windows (slightly transparent)
        ctx.fillStyle = 'rgba(255, 255, 255, 0.35)';
        ctx.beginPath();
        ctx.moveTo(x + 14, y + 12);
        ctx.lineTo(x + carWidth - 14, y + 12);
        ctx.lineTo(x + carWidth - 22, y + 38);
        ctx.lineTo(x + 22, y + 38);
        ctx.closePath();
        ctx.fill();

        // Wheels
        ctx.fillStyle = '#222';
        ctx.beginPath();
        ctx.ellipse(x + 15, y + carHeight - 15, 10, 15, 0, 0, Math.PI * 2);
        ctx.ellipse(x + carWidth - 15, y + carHeight - 15, 10, 15, 0, 0, Math.PI * 2);
        ctx.fill();

        // Wheel rims highlight
        ctx.fillStyle = '#666';
        ctx.beginPath();
        ctx.ellipse(x + 15, y + carHeight - 15, 5, 8, 0, 0, Math.PI * 2);
        ctx.ellipse(x + carWidth - 15, y + carHeight - 15, 5, 8, 0, 0, Math.PI * 2);
        ctx.fill();
      }

      // Utility to darken/lighten color
      function shadeColor(color, percent) {
        let f=parseInt(color.slice(1),16),t=percent<0?0:255,p=percent<0?percent*-1:percent;
        let R=f>>16,G=f>>8&0x00FF,B=f&0x0000FF;
        return "#" + (0x1000000 + (Math.round((t-R)*p)+R)*0x10000 + (Math.round((t-G)*p)+G)*0x100 + (Math.round((t-B)*p)+B)).toString(16).slice(1);
      }

      // Traffic car class
      class TrafficCar {
        constructor(lane, y, color) {
          this.lane = lane;
          this.x = laneToX(lane);
          this.y = y;
          this.color = color;
          this.passed = false;
        }
        update() {
          this.y += speed;
        }
        draw() {
          drawCar(this.x, this.y, this.color);
        }
      }

      // Spawn traffic ensuring at least one lane is free for passing
      function spawnTraffic() {
        const lanes = [0, 1, 2];

        // Filter lanes that are safe (no car too close near top)
        const safeLanes = lanes.filter(lane => !traffic.some(car => car.lane === lane && car.y < 160));

        if (safeLanes.length === 0) return;

        // Number of cars to spawn per tick (max 2 to guarantee gaps)
        let carsToSpawn = safeLanes.length === 1 ? 1 : (Math.random() < 0.3 ? 2 : 1);

        // Shuffle safeLanes array for randomness
        for (let i = safeLanes.length - 1; i > 0; i--) {
          const j = Math.floor(Math.random() * (i + 1));
          [safeLanes[i], safeLanes[j]] = [safeLanes[j], safeLanes[i]];
        }

        for (let i = 0; i < carsToSpawn; i++) {
          const lane = safeLanes[i];
          const maxUnlockIndex = Math.min(getCurrentUnlockIndex() + 2, unlocks.length - 1);
          const color = unlocks[Math.floor(Math.random() * (maxUnlockIndex + 1))].color;
          traffic.push(new TrafficCar(lane, -carHeight - Math.random() * 300, color));
        }
      }

      // Get current unlock index by score
      function getCurrentUnlockIndex() {
        let idx = 0;
        for (let i = 0; i < unlocks.length; i++) {
          if (score >= unlocks[i].score) idx = i;
          else break;
        }
        return idx;
      }

      // Collision detection (AABB)
      function checkCollision(a, b) {
        return !(
          a.x > b.x + carWidth - 12 ||
          a.x + carWidth - 12 < b.x ||
          a.y > b.y + carHeight - 12 ||
          a.y + carHeight - 12 < b.y
        );
      }

      // Draw lane divider
