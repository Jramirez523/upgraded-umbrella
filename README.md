const gameContainer = document.getElementById('game-container');
const startButton = document.getElementById('start-btn');
const timerElement = document.getElementById('time');
const messageElement = document.getElementById('start game message');

const players = [
  {
    id: 1,
    name: 'Player 1',
    score: 0,
    bucket: document.getElementById('bucket1'),
    scoreDisplay: document.getElementById('player1-score'),
    leftKey: 'a',
    rightKey: 'd',
  },
  {
    id: 2,
    name: 'Player 2',
    score: 0,
    bucket: document.getElementById('bucket2'),
    scoreDisplay: document.getElementById('player2-score'),
    leftKey: 'ArrowLeft',
    rightKey: 'ArrowRight',
  },
];

const GAME_DURATION = 30;
const BUCKET_STEP = 22;
const DROP_LIMIT = 24;
const DROP_SPEED_MIN = 1.6;
const DROP_SPEED_MAX = 2.6;

let timeLeft = GAME_DURATION;
let gameRunning = false;
let drops = [];
let spawnInterval = 900;
let lastSpawnTime = 0;
let lastFrameTime = 0;
let animationId = null;
let timerInterval = null;

startButton.addEventListener('click', startGame);

window.addEventListener('keydown', (event) => {
  if (!gameRunning) return;

  for (const player of players) {
    if (event.key === player.leftKey) {
      moveBucket(player, -BUCKET_STEP);
      event.preventDefault();
    }
    if (event.key === player.rightKey) {
      moveBucket(player, BUCKET_STEP);
      event.preventDefault();
    }
  }
});

function startGame() {
  if (gameRunning) return;

  gameRunning = true;
  timeLeft = GAME_DURATION;
  timerElement.textContent = timeLeft;
  messageElement.textContent = '';
  spawnInterval = 900;
  lastSpawnTime = 0;
  lastFrameTime = performance.now();

  clearDrops();
  resetPlayers();
  positionBuckets();

  startButton.disabled = true;
  timerInterval = window.setInterval(updateTimer, 1000);
  animationId = window.requestAnimationFrame(gameLoop);
}

function endGame() {
  gameRunning = false;
  window.cancelAnimationFrame(animationId);
  window.clearInterval(timerInterval);
  startButton.disabled = false;

  const result = determineWinner();
  messageElement.textContent = result;
}

function updateTimer() {
  if (!gameRunning) return;

  timeLeft -= 1;
  timerElement.textContent = timeLeft;

  if (timeLeft <= 0) {
    endGame();
  }
}

function gameLoop(timestamp) {
  if (!gameRunning) return;

  const delta = timestamp - lastFrameTime;
  lastFrameTime = timestamp;

  if (timestamp - lastSpawnTime > spawnInterval) {
    spawnDrop();
    lastSpawnTime = timestamp;
    spawnInterval = Math.max(350, spawnInterval - 8);
  }

  updateDrops(delta);
  animationId = window.requestAnimationFrame(gameLoop);
}

function spawnDrop() {
  if (!gameRunning) return;

  const drop = document.createElement('div');
  drop.className = 'drop';

  const size = 28 + Math.random() * 24;
  drop.style.width = `${size}px`;
  drop.style.height = `${size}px`;

  const x = Math.random() * (gameContainer.clientWidth - size);
  drop.style.left = `${x}px`;
  drop.style.top = `-${size}px`;
  drop.dataset.y = '0';
  drop.dataset.speed = `${DROP_SPEED_MIN + Math.random() * (DROP_SPEED_MAX - DROP_SPEED_MIN)}`;

  gameContainer.appendChild(drop);
  drops.push(drop);
}

function updateDrops(delta) {
  const bucketRects = players.map((player) => ({
    player,
    rect: player.bucket.getBoundingClientRect(),
  }));

  drops = drops.filter((drop) => {
    const y = parseFloat(drop.dataset.y) + parseFloat(drop.dataset.speed) * delta * 0.08;
    drop.dataset.y = y;
    drop.style.top = `${y}px`;

    const dropRect = drop.getBoundingClientRect();

    for (const { player, rect } of bucketRects) {
      if (isOverlap(dropRect, rect)) {
        collectDrop(player, drop);
        return false;
      }
    }

    if (y > gameContainer.clientHeight) {
      drop.remove();
      return false;
    }

    return true;
  });
}

function collectDrop(player, drop) {
  player.score += 1;
  player.scoreDisplay.textContent = player.score;
  drop.remove();
}

function determineWinner() {
  const [first, second] = [...players].sort((a, b) => b.score - a.score);

  if (first.score === second.score) {
    return `Tie! Both players scored ${first.score} points.`;
  }

  return `${first.name} wins with ${first.score} points! ${second.name} scored ${second.score}.`;
}

function moveBucket(player, delta) {
  const containerRect = gameContainer.getBoundingClientRect();
  const currentLeft = parseFloat(window.getComputedStyle(player.bucket).left);
  const nextLeft = Math.max(0, Math.min(containerRect.width - player.bucket.clientWidth, currentLeft + delta));
  player.bucket.style.left = `${nextLeft}px`;
}

function resetPlayers() {
  players.forEach((player) => {
    player.score = 0;
    player.scoreDisplay.textContent = '0';
  });
}

function positionBuckets() {
  players[0].bucket.style.left = '12%';
  players[1].bucket.style.left = '72%';
}

function clearDrops() {
  drops.forEach((drop) => drop.remove());
  drops = [];
}

function isOverlap(rect1, rect2) {
  return (
    rect1.left < rect2.right &&
    rect1.right > rect2.left &&
    rect1.top < rect2.bottom &&
    rect1.bottom > rect2.top
  );
