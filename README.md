# mini-fortnite-
mini fortnite battle royal 
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no" />
<title>Mini Royale Fortnite-Like</title>
<style>
  body, html {
    margin: 0; padding: 0; overflow: hidden;
    background: #222;
    touch-action: none;
    user-select: none;
    -webkit-user-select: none;
    font-family: Arial, sans-serif;
    color: #eee;
  }
  #gameCanvas {
    background: #4a90e2;
    display: block;
    margin: 0 auto;
    touch-action: none;
  }
  #ui {
    position: fixed;
    bottom: 10px;
    right: 10px;
    display: flex;
    gap: 10px;
  }
  button {
    font-size: 16px;
    padding: 10px 15px;
    background: #222;
    color: #eee;
    border: 2px solid #eee;
    border-radius: 8px;
    user-select: none;
    touch-action: manipulation;
  }
  button:active {
    background: #555;
  }
  #joystickBase {
    position: fixed;
    left: 15px;
    bottom: 15px;
    width: 100px;
    height: 100px;
    background: rgba(0,0,0,0.3);
    border-radius: 50%;
    touch-action: none;
  }
  #joystickStick {
    position: absolute;
    width: 60px;
    height: 60px;
    background: rgba(255,255,255,0.5);
    border-radius: 50%;
    top: 20px;
    left: 20px;
    touch-action: none;
  }
</style>
</head>
<body>
<canvas id="gameCanvas" width="400" height="600"></canvas>

<div id="joystickBase">
  <div id="joystickStick"></div>
</div>

<div id="ui">
  <button id="shootBtn">Shoot</button>
  <button id="buildBtn">Build</button>
</div>

<script>
(() => {
  const canvas = document.getElementById('gameCanvas');
  const ctx = canvas.getContext('2d');

  // Game constants
  const PLAYER_SIZE = 20;
  const PLAYER_SPEED = 2.5;
  const BULLET_SPEED = 6;
  const ENEMY_SIZE = 20;
  const ENEMY_SPEED = 1.3;
  const BUILD_COOLDOWN = 3000; // ms
  const BUILD_DURATION = 3000; // ms
  const SAFE_ZONE_SHRINK_INTERVAL = 10000; // ms
  const SAFE_ZONE_SHRINK_AMOUNT = 20;
  const SAFE_ZONE_MIN_SIZE = 100;
  const CANVAS_WIDTH = canvas.width;
  const CANVAS_HEIGHT = canvas.height;

  // Player object
  const player = {
    x: CANVAS_WIDTH / 2,
    y: CANVAS_HEIGHT / 2,
    width: PLAYER_SIZE,
    height: PLAYER_SIZE,
    health: 100,
    cooldowns: { build: 0 },
  };

  // Bullets
  let bullets = [];
  // Enemies
  let enemies = [];
  const NUM_ENEMIES = 5;
  // Builds (walls)
  let builds = [];

  // Safe zone (circle)
  let safeZone = {
    x: CANVAS_WIDTH / 2,
    y: CANVAS_HEIGHT / 2,
    radius: Math.min(CANVAS_WIDTH, CANVAS_HEIGHT) / 2 - 30,
    lastShrink: 0,
  };

  // Joystick control variables
  let joystickBase = document.getElementById('joystickBase');
  let joystickStick = document.getElementById('joystickStick');
  let joystickActive = false;
  let joystickStartPos = null;
  let joystickCurrentPos = null;
  let joystickVector = { x: 0, y: 0 };

  // Touch / Mouse controls for joystick
  joystickBase.addEventListener('touchstart', e => {
    joystickActive = true;
    joystickStartPos = { x: e.touches[0].clientX, y: e.touches[0].clientY };
    joystickCurrentPos = { ...joystickStartPos };
    updateJoystick();
    e.preventDefault();
  });
  joystickBase.addEventListener('touchmove', e => {
    if (!joystickActive) return;
    joystickCurrentPos = { x: e.touches[0].clientX, y: e.touches[0].clientY };
    updateJoystick();
    e.preventDefault();
  });
  joystickBase.addEventListener('touchend', e => {
    joystickActive = false;
    joystickVector = { x: 0, y: 0 };
    joystickStick.style.transform = `translate(20px, 20px)`;
    e.preventDefault();
  });

  function updateJoystick() {
    let dx = joystickCurrentPos.x - joystickStartPos.x;
    let dy = joystickCurrentPos.y - joystickStartPos.y;
    let maxDist = 40;
    let dist = Math.min(Math.sqrt(dx*dx + dy*dy), maxDist);
    let angle = Math.atan2(dy, dx);
    joystickVector.x = (dist / maxDist) * Math.cos(angle);
    joystickVector.y = (dist / maxDist) * Math.sin(angle);
    let stickX = 20 + joystickVector.x * maxDist;
    let stickY = 20 + joystickVector.y * maxDist;
    joystickStick.style.transform = `translate(${stickX}px, ${stickY}px)`;
  }

  // Shoot button
  const shootBtn = document.getElementById('shootBtn');
  let shooting = false;
  shootBtn.addEventListener('touchstart', e => { shooting = true; e.preventDefault(); });
  shootBtn.addEventListener('touchend', e => { shooting = false; e.preventDefault(); });
  shootBtn.addEventListener('mousedown', e => { shooting = true; });
  shootBtn.addEventListener('mouseup', e => { shooting = false; });

  // Build button
  const buildBtn = document.getElementById('buildBtn');
  buildBtn.addEventListener('click', () => {
    if (player.cooldowns.build > 0) return;
    // Place a wall in front of the player
    const buildX = player.x + (player.width/2) + (joystickVector.x * 30);
    const buildY = player.y + (player.height/2) + (joystickVector.y * 30);
    builds.push({ x: buildX, y: buildY, width: 40, height: 10, created: Date.now() });
    player.cooldowns.build = BUILD_COOLDOWN;
  });

  // Initialize enemies randomly
  function spawnEnemies() {
    enemies = [];
    for(let i=0; i<NUM_ENEMIES; i++) {
      let ex = Math.random()*(CANVAS_WIDTH-ENEMY_SIZE);
      let ey = Math.random()*(CANVAS_HEIGHT-ENEMY_SIZE);
      enemies.push({ x: ex, y: ey, width: ENEMY_SIZE, height: ENEMY_SIZE, health: 50 });
    }
  }

  spawnEnemies();

  // Game loop
  let lastTime = 0;
  function gameLoop(timestamp=0) {
    const dt = timestamp - lastTime;
    lastTime = timestamp;

    update(dt);
    draw();

    requestAnimationFrame(gameLoop);
  }

  // Update game state
  function update(dt) {
    const now = Date.now();

    // Move player by joystick
    player.x += joystickVector.x * PLAYER_SPEED;
    player.y += joystickVector.y * PLAYER_SPEED;

    // Clamp player inside canvas
    player.x = Math.min(Math.max(0, player.x), CANVAS_WIDTH - player.width);
    player.y = Math.min(Math.max(0, player.y), CANVAS_HEIGHT - player.height);

    // Shoot bullets
    if (shooting && bullets.length < 5) {
      // Create bullet moving forward (in joystick direction or default up)
      let angle = Math.atan2(joystickVector.y, joystickVector.x);
      if (joystickVector.x === 0 && joystickVector.y === 0) angle = -Math.PI/2; // up
      const bx = player.x + player.width / 2;
      const by = player.y + player.height / 2;
      bullets.push({ x: bx, y: by, angle, speed: BULLET_SPEED, radius: 5 });
    }

    // Update bullets
    bullets = bullets.filter(b => {
      b.x += Math.cos(b.angle)*b.speed;
      b.y += Math.sin(b.angle)*b.speed;
      // Remove bullets out of screen
      return b.x > 0 && b.x < CANVAS_WIDTH && b.y > 0 && b.y < CANVAS_HEIGHT;
    });

    // Update enemies - simple AI: move toward player
    enemies.forEach(e => {
      let dx = player.x - e.x;
      let dy = player.y - e.y;
      let dist = Math.sqrt(dx*dx + dy*dy);
      if (dist > 0) {
        e.x += (dx / dist) * ENEMY_SPEED;
        e.y += (dy / dist) * ENEMY_SPEED;
      }
    });

    // Check bullet-enemy collisions
    bullets.forEach((b, bi) => {
      enemies.forEach((e, ei) => {
        let dx = b.x - (e.x + e.width/2);
        let dy = b.y - (e.y + e.height/2);
        let dist = Math.sqrt(dx*dx + dy*dy);
        if (dist < b.radius + e.width/2) {
          // Hit enemy
          e.health -= 25;
          bullets.splice(bi, 1);
          if (e.health <= 0) {
            enemies.splice(ei, 1);
          }
        }
      });
    });

    // Update builds lifetime
    builds = builds.filter(b => (now - b.created) < BUILD_DURATION);

    // Update build cooldown
    if (player.cooldowns.build > 0) {
      player.cooldowns.build -= dt;
      if (player.cooldowns.build < 0) player.cooldowns.build = 0;
    }

    // Check safe zone shrinking
    if (now - safeZone.lastShrink > SAFE_ZONE_SHRINK_INTERVAL) {
      safeZone.radius = Math.max(SAFE_ZONE_MIN_SIZE, safeZone.radius - SAFE_ZONE_SHRINK_AMOUNT);
      safeZone.lastShrink = now;
    }

    // Damage player if outside safe zone
    let px = player.x + player.width / 2;
    let py = player.y + player.height / 2;
    let distToCenter = Math.sqrt((px - safeZone.x)**2 + (py - safeZone.y)**2);
    if (distToCenter > safeZone.radius) {
      player.health -= 0.3 * (distToCenter - safeZone.radius);
      if (player.health < 0) player.health = 0;
    }

    // TODO: Heal pickups, more complex AI, etc.
  }

  // Draw game
  function draw() {
    ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

    // Draw safe zone (circle)
    ctx.beginPath();
    ctx.strokeStyle = '#33ff33';
    ctx.lineWidth = 4;
    ctx.arc(safeZone.x, safeZone.y, safeZone.radius, 0, Math.PI * 2);
    ctx.stroke();

    // Draw player
    ctx.fillStyle = '#0055ff';
    ctx.fillRect(player.x, player.y, player.width, player.height);

    // Draw enemies
    enemies.forEach(e => {
      ctx.fillStyle = '#ff4444';
      ctx.fillRect(e.x, e.y, e.width, e.height);
    });

    // Draw bullets
    bullets.forEach(b => {
      ctx.beginPath();
      ctx.fillStyle = '#ffff00';
      ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
      ctx.fill();
    });

    // Draw builds (walls)
    builds.forEach(b => {
      ctx.fillStyle = '#663300';
      ctx.fillRect(b.x, b.y, b.width, b.height);
    });

    // Draw player health bar
    ctx.fillStyle = '#000';
    ctx.fillRect(10, 10, 100, 12);
    ctx.fillStyle = '#33ff33';
    ctx.fillRect(10, 10, player.health, 12);
    ctx.strokeStyle = '#000';
    ctx.strokeRect(10, 10, 100, 12);

    // Draw enemies left
    ctx.fillStyle = '#eee';
    ctx.font = '16px Arial';
    ctx.fillText('Enemies left: ' + enemies.length, 10, 40);

    // Draw build cooldown
    ctx.fillText('Build CD: ' + (player.cooldowns.build > 0 ? (player.cooldowns.build / 1000).toFixed(1) + 's' : 'Ready'), 10, 60);
  }

  // Start game loop
  requestAnimationFrame(gameLoop);
})();
</script>
</body>
</html>