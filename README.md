const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const startBtn = document.getElementById('startBtn');
const overlay = document.getElementById('overlay');

const state = {
  mode: 'menu',
  score: 0,
  best: Number(localStorage.getItem('neon-rift-best') || 0),
  wave: 1,
  kills: 0,
  message: 'System boot complete',
  messageTimer: 3,
  bossActive: false,
  gameOver: false,
  victory: false,
};

const input = {
  keys: {},
  mouseX: canvas.width / 2,
  mouseY: canvas.height / 2,
  pointerDown: false,
};

const game = {
  width: canvas.width,
  height: canvas.height,
  player: null,
  bullets: [],
  enemies: [],
  pickups: [],
  particles: [],
  enemySpawnTimer: 1.1,
  time: 0,
  lastTime: 0,
};

const clamp = (value, min, max) => Math.min(Math.max(value, min), max);
const rand = (min, max) => Math.random() * (max - min) + min;

function createPlayer() {
  return {
    x: canvas.width / 2,
    y: canvas.height / 2,
    radius: 16,
    speed: 230,
    health: 100,
    maxHealth: 100,
    fireCooldown: 0,
    dashCooldown: 0,
    dashTime: 0,
    invuln: 0,
    ammo: 999,
    damage: 18,
    level: 1,
    xp: 0,
    nextLevelXP: 8,
    angle: 0,
  };
}

function resetGame() {
  game.player = createPlayer();
  game.bullets = [];
  game.enemies = [];
  game.pickups = [];
  game.particles = [];
  state.score = 0;
  state.wave = 1;
  state.kills = 0;
  state.message = 'Wave 1 initialized';
  state.messageTimer = 2.5;
  state.bossActive = false;
  state.gameOver = false;
  state.victory = false;
  game.enemySpawnTimer = 0.8;
  game.time = 0;
  game.lastTime = 0;
}

function startGame() {
  overlay.classList.remove('visible');
  resetGame();
  state.mode = 'playing';
}

function showOverlay(title, subtitle, buttonText = 'Restart Mission') {
  overlay.innerHTML = `
    <div class="panel">
      <h1>${title}</h1>
      <p>${subtitle}</p>
      <button id="startBtn">${buttonText}</button>
    </div>
  `;

  const restartButton = document.getElementById('startBtn');
  restartButton.addEventListener('click', () => {
    startGame();
  });
  overlay.classList.add('visible');
}

function addParticles(x, y, color, amount = 12) {
  for (let i = 0; i < amount; i += 1) {
    game.particles.push({
      x,
      y,
      vx: rand(-120, 120),
      vy: rand(-120, 120),
      radius: rand(1.5, 4.5),
      life: rand(0.4, 0.9),
      maxLife: rand(0.4, 0.9),
      color,
    });
  }
}

function spawnPickup(x, y) {
  const roll = Math.random();
  let type = 'health';

  if (roll > 0.7) type = 'boost';
  if (roll > 0.9) type = 'charge';

  game.pickups.push({
    x,
    y,
    radius: 10,
    type,
    pulse: Math.random() * Math.PI * 2,
  });
}

function createBullet(x, y, angle, speed, damage, from) {
  game.bullets.push({
    x,
    y,
    radius: from === 'player' ? 4 : 6,
    vx: Math.cos(angle) * speed,
    vy: Math.sin(angle) * speed,
    damage,
    from,
    life: from === 'player' ? 1.7 : 2.3,
  });
}

function spawnEnemy(type = 'scout') {
  let x = 0;
  let y = 0;
  const side = Math.floor(Math.random() * 4);

  if (side === 0) {
    x = rand(-50, game.width + 50);
    y = -30;
  } else if (side === 1) {
    x = game.width + 30;
    y = rand(-50, game.height + 50);
  } else if (side === 2) {
    x = rand(-50, game.width + 50);
    y = game.height + 30;
  } else {
    x = -30;
    y = rand(-50, game.height + 50);
  }

  const base = {
    scout: { hp: 26, speed: 78, radius: 12, color: '#ff61c8', score: 14 },
    brute: { hp: 58, speed: 52, radius: 20, color: '#ff8d68', score: 26 },
    sniper: { hp: 34, speed: 60, radius: 16, color: '#63f3ff', score: 20 },
    boss: { hp: 280, speed: 48, radius: 34, color: '#ffd166', score: 200 },
  };

  const stats = base[type];
  game.enemies.push({
    x,
    y,
    radius: stats.radius,
    speed: stats.speed,
    hp: stats.hp,
    maxHp: stats.hp,
    color: stats.color,
    type,
    score: stats.score,
    fireCooldown: type === 'sniper' ? 1.4 : 2,
    phase: Math.random() * Math.PI * 2,
  });
}

function setMessage(text, duration = 2.5) {
  state.message = text;
  state.messageTimer = duration;
}

function advanceWave() {
  state.wave += 1;
  state.kills = 0;
  setMessage(`Wave ${state.wave} engaged`, 2.8);
 game.enemySpawnTimer = Math.max(0.55, 1.8 - state.wave * 0.16);

  if (state.wave >= 3) {
    state.bossActive = true;
    setMessage('Rift Warden detected', 3.2);
    spawnEnemy('boss');
  }
}

function applyPickup(type) {
  const player = game.player;
  if (type === 'health') {
    player.health = Math.min(player.maxHealth, player.health + 24);
    setMessage('Nanite repair complete', 1.8);
  }

  if (type === 'boost') {
    player.damage += 4;
    player.speed += 15;
    setMessage('Weapon overclocked', 1.8);
  }

  if (type === 'charge') {
    player.fireCooldown = Math.max(0, player.fireCooldown - 0.08);
    player.health = Math.min(player.maxHealth, player.health + 12);
    setMessage('Arc reactor stabilized', 1.8);
  }
}

function gainXP(amount) {
  const player = game.player;
  player.xp += amount;

  while (player.xp >= player.nextLevelXP) {
    player.xp -= player.nextLevelXP;
    player.level += 1;
    player.nextLevelXP = Math.round(player.nextLevelXP * 1.5);
    player.damage += 4;
    player.maxHealth += 10;
    player.health = player.maxHealth;
    setMessage(`Level ${player.level} achieved`, 2.1);
  }
}

function shootPlayerBullet() {
  const player = game.player;
  if (player.fireCooldown > 0) return;

  const angle = Math.atan2(input.mouseY - player.y, input.mouseX - player.x);
  createBullet(player.x, player.y, angle, 540, player.damage, 'player');
  player.fireCooldown = 0.22;
  addParticles(player.x + Math.cos(angle) * 16, player.y + Math.sin(angle) * 16, '#63f3ff', 8);
}

function updatePlayer(dt) {
  const player = game.player;
  const moveX = (input.keys['KeyD'] || input.keys['ArrowRight'] ? 1 : 0) - (input.keys['KeyA'] || input.keys['ArrowLeft'] ? 1 : 0);
  const moveY = (input.keys['KeyS'] || input.keys['ArrowDown'] ? 1 : 0) - (input.keys['KeyW'] || input.keys['ArrowUp'] ? 1 : 0);
  const length = Math.hypot(moveX, moveY) || 1;

  if (moveX || moveY) {
    player.x += (moveX / length) * player.speed * dt;
    player.y += (moveY / length) * player.speed * dt;
  }

  player.x = clamp(player.x, player.radius + 2, game.width - player.radius - 2);
  player.y = clamp(player.y, player.radius + 2, game.height - player.radius - 2);
  player.angle = Math.atan2(input.mouseY - player.y, input.mouseX - player.x);

  player.fireCooldown = Math.max(0, player.fireCooldown - dt);
  player.dashCooldown = Math.max(0, player.dashCooldown - dt);
  player.invuln = Math.max(0, player.invuln - dt);

  if (input.keys['ShiftLeft'] || input.keys['ShiftRight']) {
    if (player.dashCooldown <= 0) {
      const dashX = Math.cos(player.angle) * 180;
      const dashY = Math.sin(player.angle) * 180;
      player.x = clamp(player.x + dashX * dt * 1.8, player.radius + 2, game.width - player.radius - 2);
      player.y = clamp(player.y + dashY * dt * 1.8, player.radius + 2, game.height - player.radius - 2);
      player.dashCooldown = 1.3;
      player.invuln = 0.35;
      addParticles(player.x, player.y, '#aa7dff', 18);
    }
  }

  if (input.pointerDown || input.keys['Space']) {
    shootPlayerBullet();
  }
}

function updateBullets(dt) {
  for (let i = game.bullets.length - 1; i >= 0; i -= 1) {
    const bullet = game.bullets[i];
    bullet.x += bullet.vx * dt;
    bullet.y += bullet.vy * dt;
    bullet.life -= dt;

    if (bullet.life <= 0 || bullet.x < -30 || bullet.x > game.width + 30 || bullet.y < -30 || bullet.y > game.height + 30) {
      game.bullets.splice(i, 1);
      continue;
    }

    if (bullet.from === 'player') {
      for (let j = game.enemies.length - 1; j >= 0; j -= 1) {
        const enemy = game.enemies[j];
        const dist = Math.hypot(bullet.x - enemy.x, bullet.y - enemy.y);
        if (dist < bullet.radius + enemy.radius) {
          enemy.hp -= bullet.damage;
          addParticles(bullet.x, bullet.y, '#63f3ff', 6);
          game.bullets.splice(i, 1);
          if (enemy.hp <= 0) {
            killEnemy(j);
          }
          break;
        }
      }
    } else {
      const player = game.player;
      const dist = Math.hypot(bullet.x - player.x, bullet.y - player.y);
      if (dist < bullet.radius + player.radius) {
        damagePlayer(10);
        game.bullets.splice(i, 1);
        addParticles(bullet.x, bullet.y, '#ff6666', 10);
      }
    }
  }
}

function damagePlayer(amount) {
  const player = game.player;
  if (player.invuln > 0) return;
  player.health -= amount;
  player.invuln = 0.5;
  addParticles(player.x, player.y, '#ff6666', 16);

  if (player.health <= 0) {
    state.gameOver = true;
    state.mode = 'gameover';
    state.best = Math.max(state.best, state.score);
    localStorage.setItem('neon-rift-best', String(state.best));
    showOverlay('Mission Failed', `Final score: ${state.score}. The rift is not done with you yet.`, 'Retry Mission');
  }
}

function killEnemy(index) {
  const enemy = game.enemies[index];
  state.score += enemy.score;
  state.kills += 1;
  addParticles(enemy.x, enemy.y, enemy.color, 18);
  if (Math.random() < 0.25) spawnPickup(enemy.x, enemy.y);
  game.enemies.splice(index, 1);
  gainXP(2);

  if (state.kills >= 8 + state.wave * 2) {
    advanceWave();
  }

  if (state.wave >= 3 && !state.bossActive) {
    const aliveBoss = game.enemies.some((e) => e.type === 'boss');
    if (!aliveBoss) {
      state.bossActive = true;
      spawnEnemy('boss');
      setMessage('Rift Warden engaged', 3.2);
    }
  }
}

function updateEnemies(dt) {
  const player = game.player;

  for (let i = game.enemies.length - 1; i >= 0; i -= 1) {
    const enemy = game.enemies[i];
    const dx = player.x - enemy.x;
    const dy = player.y - enemy.y;
    const dist = Math.hypot(dx, dy) || 1;
    enemy.x += (dx / dist) * enemy.speed * dt;
    enemy.y += (dy / dist) * enemy.speed * dt;

    if (enemy.type === 'boss') {
      enemy.phase += dt * 2.2;
      enemy.fireCooldown -= dt;
      if (enemy.fireCooldown <= 0) {
        const angle = Math.atan2(player.y - enemy.y, player.x - enemy.x);
        for (let burst = 0; burst < 5; burst += 1) {
          const spread = (-2 + burst) * 0.22;
          createBullet(enemy.x, enemy.y, angle + spread, 320, 9, 'enemy');
        }
        enemy.fireCooldown = 1.35;
      }
    }

    if (enemy.type === 'sniper') {
      enemy.fireCooldown -= dt;
      if (enemy.fireCooldown <= 0) {
        const angle = Math.atan2(player.y - enemy.y, player.x - enemy.x);
        createBullet(enemy.x, enemy.y, angle, 360, 14, 'enemy');
        enemy.fireCooldown = 1.9;
      }
    }

    if (dist < enemy.radius + player.radius + 4) {
      damagePlayer(10);
    }
  }
}

function updatePickups(dt) {
  for (let i = game.pickups.length - 1; i >= 0; i -= 1) {
    const pickup = game.pickups[i];
    pickup.pulse += dt * 5;
    const dist = Math.hypot(game.player.x - pickup.x, game.player.y - pickup.y);
    if (dist < pickup.radius + game.player.radius) {
      applyPickup(pickup.type);
      game.pickups.splice(i, 1);
      continue;
    }

    if (pickup.y > game.height + 40) {
      game.pickups.splice(i, 1);
    }
  }
}

function updateParticles(dt) {
  for (let i = game.particles.length - 1; i >= 0; i -= 1) {
    const p = game.particles[i];
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.life -= dt;
    if (p.life <= 0) game.particles.splice(i, 1);
  }
}

function updateSpawning(dt) {
  game.enemySpawnTimer -= dt;
  if (game.enemySpawnTimer <= 0 && !state.bossActive) {
    const enemyType = Math.random() < 0.15 ? 'brute' : Math.random() < 0.35 ? 'sniper' : 'scout';
    spawnEnemy(enemyType);
    game.enemySpawnTimer = Math.max(0.5, 1.7 - state.wave * 0.12);
  }
}

function updateBossVictory() {
  const bossAlive = game.enemies.some((enemy) => enemy.type === 'boss');
  if (!bossAlive && state.bossActive && state.wave >= 3) {
    state.victory = true;
    state.mode = 'victory';
    state.best = Math.max(state.best, state.score);
    localStorage.setItem('neon-rift-best', String(state.best));
    showOverlay('Rift Cleared', `Victory secured. Final score: ${state.score}.`, 'Play Again');
  }
}

function update(dt) {
  if (state.mode !== 'playing') return;

  game.time += dt;
  setMessageIfNeeded(dt);
  updatePlayer(dt);
  updateBullets(dt);
  updateEnemies(dt);
  updatePickups(dt);
  updateParticles(dt);
  updateSpawning(dt);
  updateBossVictory();

  if (game.time > 18 && state.wave === 1) {
    setMessage('Wave 2 surge incoming', 2.5);
    state.wave = 2;
    game.enemySpawnTimer = 1.2;
  }

  if (game.time > 38 && state.wave === 2 && !state.bossActive) {
    state.wave = 3;
    state.bossActive = true;
    setMessage('Rift Warden detected', 2.8);
    spawnEnemy('boss');
  }
}

function setMessageIfNeeded(dt) {
  if (state.messageTimer > 0) {
    state.messageTimer -= dt;
  }
}

function drawBackground() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  const bg = ctx.createLinearGradient(0, 0, 0, canvas.height);
  bg.addColorStop(0, '#050c1e');
  bg.addColorStop(1, '#0b132c');
  ctx.fillStyle = bg;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.strokeStyle = 'rgba(99, 243, 255, 0.08)';
  ctx.lineWidth = 1;
  for (let x = 0; x < canvas.width; x += 32) {
    ctx.beginPath();
    ctx.moveTo(x, 0);
    ctx.lineTo(x, canvas.height);
    ctx.stroke();
  }
  for (let y = 0; y < canvas.height; y += 32) {
    ctx.beginPath();
    ctx.moveTo(0, y);
    ctx.lineTo(canvas.width, y);
    ctx.stroke();
  }

  ctx.fillStyle = 'rgba(99, 243, 255, 0.06)';
  for (let i = 0; i < 35; i += 1) {
    const x = (i * 109.4 + game.time * 25) % (canvas.width + 120) - 60;
    const y = (i * 83.7 + game.time * 30) % (canvas.height + 120) - 60;
    ctx.beginPath();
    ctx.arc(x, y, 2 + (i % 3), 0, Math.PI * 2);
    ctx.fill();
  }
}

function drawPlayer() {
  const player = game.player;
  ctx.save();
  ctx.translate(player.x, player.y);
  ctx.rotate(player.angle);

  ctx.fillStyle = '#0d2038';
  ctx.beginPath();
  ctx.arc(0, 0, player.radius + 5, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#63f3ff';
  ctx.beginPath();
  ctx.arc(0, 0, player.radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#dffaff';
  ctx.fillRect(8, -3, 16, 6);
  ctx.restore();
}

function drawEnemies() {
  for (const enemy of game.enemies) {
    ctx.beginPath();
    ctx.fillStyle = enemy.color;
    ctx.arc(enemy.x, enemy.y, enemy.radius, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = 'rgba(0,0,0,0.4)';
    ctx.fillRect(enemy.x - enemy.radius, enemy.y - enemy.radius - 12, enemy.radius * 2, 6);
    ctx.fillStyle = '#dffaff';
    ctx.fillRect(enemy.x - enemy.radius, enemy.y - enemy.radius - 12, (enemy.hp / enemy.maxHp) * enemy.radius * 2, 6);
  }
}

function drawBullets() {
  for (const bullet of game.bullets) {
    ctx.beginPath();
    ctx.fillStyle = bullet.from === 'player' ? '#63f3ff' : '#ff6666';
    ctx.arc(bullet.x, bullet.y, bullet.radius, 0, Math.PI * 2);
    ctx.fill();
  }
}

function drawPickups() {
  for (const pickup of game.pickups) {
    const colorMap = {
      health: '#adff8a',
      boost: '#ffd166',
      charge: '#aa7dff',
    };

    ctx.save();
    ctx.translate(pickup.x, pickup.y);
    ctx.rotate(pickup.pulse);
    ctx.fillStyle = colorMap[pickup.type];
    ctx.fillRect(-8, -8, 16, 16);
    ctx.restore();
  }
}

function drawParticles() {
  for (const p of game.particles) {
    ctx.fillStyle = p.color + '66';
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
    ctx.fill();
  }
}

function drawHud() {
  const player = game.player;
  ctx.fillStyle = 'rgba(5, 9, 20, 0.7)';
  ctx.fillRect(18, canvas.height - 72, 230, 52);

  ctx.fillStyle = '#edf6ff';
  ctx.font = '16px Segoe UI';
  ctx.fillText(`HP`, 28, canvas.height - 42);
  ctx.fillStyle = '#ff6666';
  ctx.fillRect(62, canvas.height - 50, 150, 12);
  ctx.fillStyle = '#adff8a';
  ctx.fillRect(62, canvas.height - 50, (player.health / player.maxHealth) * 150, 12);

  ctx.fillStyle = '#edf6ff';
  ctx.fillText(`Score: ${state.score}`, 28, canvas.height - 18);
  ctx.fillText(`Wave: ${state.wave}`, 160, canvas.height - 18);

  ctx.fillStyle = '#edf6ff';
  ctx.font = '18px Segoe UI';
  ctx.fillText(`Level ${player.level}`, canvas.width - 120, 32);
  ctx.fillText(`Best ${state.best}`, canvas.width - 120, 56);

  if (state.messageTimer > 0) {
    ctx.fillStyle = 'rgba(99, 243, 255, 0.9)';
    ctx.font = 'bold 18px Segoe UI';
    ctx.fillText(state.message, canvas.width / 2 - ctx.measureText(state.message).width / 2, 36);
  }
}

function draw() {
  drawBackground();
  if (state.mode === 'menu') {
    ctx.fillStyle = 'rgba(255,255,255,0.1)';
    ctx.font = 'bold 42px Segoe UI';
    ctx.fillText('NEON RIFT', canvas.width / 2 - 130, canvas.height / 2 - 30);
    ctx.font = '20px Segoe UI';
    ctx.fillText('Press Start Mission to begin', canvas.width / 2 - 155, canvas.height / 2 + 10);
    return;
  }

  if (game.player) {
    drawPickups();
    drawBullets();
    drawEnemies();
    drawPlayer();
    drawParticles();
    drawHud();
  }
}

function loop(timestamp) {
  const dt = Math.min((timestamp - game.lastTime) / 1000 || 0.016, 0.033);
  game.lastTime = timestamp;
  update(dt);
  draw();
  requestAnimationFrame(loop);
}

window.addEventListener('keydown', (event) => {
  input.keys[event.code] = true;
  if (event.code === 'Space') {
    event.preventDefault();
  }
});

window.addEventListener('keyup', (event) => {
  input.keys[event.code] = false;
});

canvas.addEventListener('mousemove', (event) => {
  const rect = canvas.getBoundingClientRect();
  input.mouseX = ((event.clientX - rect.left) / rect.width) * canvas.width;
  input.mouseY = ((event.clientY - rect.top) / rect.height) * canvas.height;
});

canvas.addEventListener('mousedown', () => {
  input.pointerDown = true;
});

window.addEventListener('mouseup', () => {
  input.pointerDown = false;
});

startBtn.addEventListener('click', () => {
  startGame();
});

showOverlay('Neon Rift', 'Wipe out drones, survive the rift, and destroy the Warden.', 'Start Mission');
requestAnimationFrame(loop);
