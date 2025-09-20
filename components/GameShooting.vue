<script setup>
const CANVAS_SIZE = 600;
const BULLET_RADIUS = 5;
const STAR_COUNT = 50;
const BOMB_FLASH_FRAMES = 10;

class Star {
  constructor() {
    this.x = Math.random() * CANVAS_SIZE;
    this.y = Math.random() * CANVAS_SIZE;
    this.r = Math.random() * 5 + 1;
  }

  tick() {
    this.y += this.r;
    if (this.y > CANVAS_SIZE) {
      this.y -= CANVAS_SIZE;
    }
    drawCircle(this.x, this.y, this.r, "#888800");
  }
}

class Ship {
  constructor() {
    this.img = shipImage.value;
    this.x = CANVAS_SIZE / 2;
    this.y = CANVAS_SIZE - 100;
    this.sx = 0;
    this.sy = 0;
  }

  move(mouseX, mouseY) {
    this.sx = (mouseX - this.x) / 10;
    this.sy = (mouseY - this.y) / 10;
  }

  tick(bombIntensity = 0) {
    this.x += this.sx;
    this.y += this.sy;
    if (!ctx) {
      return;
    }
    if (bombIntensity > 0) {
      drawShipGlow(this, bombIntensity);
    }
    if (this.img) {
      ctx.drawImage(this.img, this.x - 50, this.y - 50);
    }
  }

  shoot() {
    bullets.push(new Bullet(this.x, this.y, 0, -25, true));
  }
}

class Enemy {
  constructor() {
    this.img = enemyImage.value;
    this.x = Math.random() * 400 + 100;
    this.y = 0;
    this.sx = Math.random() * 5 - 2.5;
    this.sy = Math.random() * 15 + 15;
    this.shoot = false;
  }

  tick() {
    this.sy -= 1;
    this.x += this.sx;
    this.y += this.sy;
    if (ctx && this.img) {
      ctx.drawImage(this.img, this.x - 50, this.y - 50);
    }

    if (!this.shoot && this.sy < 0 && ship) {
      const theta = Math.atan2(ship.y - this.y, ship.x - this.x);
      const sx = Math.cos(theta) * 10;
      const sy = Math.sin(theta) * 10;
      bullets.push(new Bullet(this.x, this.y, sx, sy, false));
      this.shoot = true;
    }
  }
}

class Bullet {
  constructor(x, y, sx, sy, isShip) {
    this.x = x;
    this.y = y;
    this.sx = sx;
    this.sy = sy;
    this.isShip = isShip;
  }

  tick() {
    this.x += this.sx;
    this.y += this.sy;
    drawCircle(this.x, this.y, BULLET_RADIUS, this.isShip ? "blue" : "red");
  }
}

function drawCircle(x, y, r, color) {
  if (!ctx) {
    return;
  }
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.moveTo(x, y);
  ctx.arc(x, y, r, 0, Math.PI * 2);
  ctx.closePath();
  ctx.fill();
}

function drawBombFlash(intensity) {
  if (!ctx || !ship) {
    return;
  }
  const baseAlpha = 0.3;
  const maxAlpha = 0.75;
  const overlayAlpha = baseAlpha + (maxAlpha - baseAlpha) * intensity;

  ctx.save();
  ctx.globalAlpha = overlayAlpha;
  ctx.fillStyle = "#ffffff";
  ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);
  ctx.restore();

  ctx.save();
  const gradientRadius = CANVAS_SIZE * 0.6;
  const gradient = ctx.createRadialGradient(
    ship.x,
    ship.y,
    0,
    ship.x,
    ship.y,
    gradientRadius
  );
  gradient.addColorStop(0, "rgba(255, 255, 255, 0)");
  const gradientAlpha = Math.min(0.9, overlayAlpha + 0.1);
  gradient.addColorStop(0.45, "rgba(255, 255, 255, " + gradientAlpha + ")");
  gradient.addColorStop(1, "rgba(255, 255, 255, 0)");
  ctx.globalCompositeOperation = "lighter";
  ctx.fillStyle = gradient;
  ctx.fillRect(
    ship.x - gradientRadius,
    ship.y - gradientRadius,
    gradientRadius * 2,
    gradientRadius * 2
  );
  ctx.restore();
}

function drawShipGlow(currentShip, intensity) {
  if (!ctx) {
    return;
  }
  const glowRadius = 60 + intensity * 40;
  const innerRadius = glowRadius * 0.4;
  const gradient = ctx.createRadialGradient(
    currentShip.x,
    currentShip.y,
    innerRadius,
    currentShip.x,
    currentShip.y,
    glowRadius
  );
  const centerAlpha = Math.min(0.9, 0.5 + 0.4 * intensity);
  gradient.addColorStop(0, "rgba(255, 255, 255, " + centerAlpha + ")");
  gradient.addColorStop(1, "rgba(255, 255, 255, 0)");

  ctx.save();
  ctx.globalCompositeOperation = "lighter";
  ctx.fillStyle = gradient;
  ctx.fillRect(
    currentShip.x - glowRadius,
    currentShip.y - glowRadius,
    glowRadius * 2,
    glowRadius * 2
  );
  ctx.restore();
}

const field = ref(null);
let ctx;
let ship = null;
const shipImage = ref(null);
const enemyImage = ref(null);
const back = ref(null);
const isButtonHidden = ref(false);
let count = 0;
let interval = 50;
let timerId = null;
let bullets = [];
let bombEffect = 0;
let enemies = [];
const stars = [];
const score = ref(0);
const scoreList = ref([0, 0, 0, 0, 0]);
const resizeCanvas = ref(null);

const pointerMoveHandler = (event) => {
  if (!ship || !field.value) {
    return;
  }
  const rect = field.value.getBoundingClientRect();
  const mouseX = event.clientX - rect.left;
  const mouseY = event.clientY - rect.top;
  ship.move(mouseX, mouseY);
};

const clickHandler = () => {
  if (!ship) {
    return;
  }
  ship.shoot();
};

const contextMenuHandler = (event) => {
  if (!ship) {
    return;
  }
  event.preventDefault();
  bomb();
};

const gameStart = () => {
  isButtonHidden.value = true;
  count = 0;
  interval = 50;
  clearGameLoop();
  bullets = [];
  enemies = [];
  stars.length = 0;
  bombEffect = 0;
  score.value = 0;
  ship = new Ship();
  for (let i = 0; i < STAR_COUNT; i++) {
    stars.push(new Star());
  }
  startGameLoop();
};

function startGameLoop() {
  clearGameLoop();
  timerId = setInterval(tick, 50);
}

function clearGameLoop() {
  if (timerId !== null) {
    clearInterval(timerId);
    timerId = null;
  }
}

function tick() {
  if (!ctx || !ship) {
    return;
  }
  count += 1;
  ctx.fillStyle = "black";
  ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);
  if (back.value) {
    ctx.drawImage(back.value, 0, 0);
  }

  const currentBombEffect = bombEffect;
  let bombIntensity = 0;
  if (currentBombEffect > 0) {
    bombIntensity = currentBombEffect / BOMB_FLASH_FRAMES;
    drawBombFlash(bombIntensity);
    bombEffect -= 1;
  }

  stars.forEach((star) => star.tick());

  ship.tick(bombIntensity);

  if (count % interval === 0) {
    enemies.push(new Enemy());
    interval = Math.max(5, interval - 5);
  }

  let gameOver = false;
  const activeEnemies = [];

  enemies.forEach((enemy) => {
    enemy.tick();
    if (dist(enemy, ship) < 100) {
      gameOver = true;
      return;
    }
    activeEnemies.push(enemy);
  });
  enemies = activeEnemies;

  let defeatedEnemies = 0;
  const activeBullets = [];

  bullets.forEach((bullet) => {
    bullet.tick();

    if (!ship) {
      return;
    }

    if (!bullet.isShip && dist(bullet, ship) < 30) {
      gameOver = true;
      return;
    }

    const outOfBounds =
      bullet.x < -BULLET_RADIUS ||
      bullet.x > CANVAS_SIZE + BULLET_RADIUS ||
      bullet.y < -BULLET_RADIUS ||
      bullet.y > CANVAS_SIZE + BULLET_RADIUS;

    if (outOfBounds) {
      return;
    }

    if (bullet.isShip) {
      const before = enemies.length;
      enemies = enemies.filter((enemy) => dist(enemy, bullet) >= 50);
      const defeated = before - enemies.length;
      if (defeated > 0) {
        defeatedEnemies += defeated;
        return;
      }
    }

    activeBullets.push(bullet);
  });

  bullets = activeBullets;
  score.value += defeatedEnemies;
  ctx.fillStyle = "yellow";

  if (gameOver) {
    endGame();
  }
}

function endGame() {
  clearGameLoop();
  if (ctx) {
    ctx.fillStyle = "yellow";
    ctx.fillText("GAME OVER", 200, 200);
  }
  rearrangementScoreList();
  isButtonHidden.value = false;
  ship = null;
}

function bomb() {
  if (!ship) {
    return;
  }
  const prevNum = enemies.length;
  enemies = [];
  score.value += prevNum;
  bombEffect = BOMB_FLASH_FRAMES;
}

function rearrangementScoreList() {
  scoreList.value.push(score.value);
  scoreList.value.sort((a, b) => a - b);
  scoreList.value.shift();
}

function dist(e0, e1) {
  return Math.sqrt((e0.x - e1.x) ** 2 + (e0.y - e1.y) ** 2);
}

const choiceEnemyFile = (event) => {
  const file = event.target.files?.[0];
  if (!file) {
    return;
  }
  const reader = new FileReader();
  reader.onload = () => {
    resizeImage(reader.result, enemyImage);
  };
  reader.readAsDataURL(file);
};

const choiceShipFile = (event) => {
  const file = event.target.files?.[0];
  if (!file) {
    return;
  }
  const reader = new FileReader();
  reader.onload = () => {
    resizeImage(reader.result, shipImage);
  };
  reader.readAsDataURL(file);
};

async function resizeImage(imageData, targetImage) {
  const img = new Image();
  img.src = imageData;
  await new Promise((resolve) => {
    img.onload = resolve;
  });
  const aspectRatio = img.width / img.height;
  const newWidth = 134;
  const newHeight = newWidth / aspectRatio;
  if (!resizeCanvas.value) {
    return;
  }
  resizeCanvas.value.width = newWidth;
  resizeCanvas.value.height = newHeight;
  const resizeContext = resizeCanvas.value.getContext("2d");
  if (!resizeContext) {
    return;
  }
  resizeContext.clearRect(0, 0, newWidth, newHeight);
  resizeContext.drawImage(img, 0, 0, newWidth, newHeight);
  if (targetImage.value) {
    targetImage.value.src = resizeCanvas.value.toDataURL("image/png");
  }
}

onMounted(() => {
  if (!field.value) {
    return;
  }
  ctx = field.value.getContext("2d");
  if (ctx) {
    ctx.font = "32px 'Times New Roman'";
  }
  window.addEventListener("pointermove", pointerMoveHandler);
  window.addEventListener("click", clickHandler);
  window.addEventListener("contextmenu", contextMenuHandler);
});

onBeforeUnmount(() => {
  clearGameLoop();
  ship = null;
  window.removeEventListener("pointermove", pointerMoveHandler);
  window.removeEventListener("click", clickHandler);
  window.removeEventListener("contextmenu", contextMenuHandler);
});
</script>
<template>
  <div class="flex">
    <div class="flex border justify-center items-center">
      <button
        v-if="!isButtonHidden"
        ref="buttonStart"
        v-on:click="gameStart"
        class="btn absolute"
      >
        Game Start
      </button>
      <canvas ref="field" width="600" height="600"></canvas>
    </div>
    <div>
      <div class="border rounded-md m-2">
        <p class="text-center">Enemy Image</p>
        <input
          type="file"
          v-on:change="choiceEnemyFile"
          class="file-input w-full max-w-xs"
        />
        <canvas ref="resizeCanvas" class="hidden"></canvas>
        <img ref="back" src="/images/back.png" class="hidden" />
        <div class="flex justify-center">
          <img ref="enemyImage" src="/images/enemy.png" />
        </div>
      </div>
      <div class="border rounded-md m-2">
        <p class="text-center">Ship Image</p>
        <input
          type="file"
          v-on:change="choiceShipFile"
          class="file-input w-full max-w-xs"
        />
        <div class="flex justify-center">
          <img ref="shipImage" src="/images/ship.png" />
        </div>
      </div>
      <div class="border rounded-md m-2">
        <p class="text-lg text-center">SCORE</p>
        <p class="text-center">{{ score }}</p>
        <p class="text-lg text-center">Ranking</p>
        <p
          v-for="scoreItem in scoreList.slice().reverse()"
          :key="scoreItem"
          class="text-base text-center p-2"
        >
          {{ scoreItem }}
        </p>
      </div>
    </div>
  </div>
</template>
