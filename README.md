const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const carNameEl = document.getElementById('car-name');
const speedDisplay = document.getElementById('speed-display');
const boostDisplay = document.getElementById('boost-display');
const zDisplay = document.getElementById('z-display');
const ballDisplay = document.getElementById('ball-display');
const controlsListEl = document.getElementById('controls-list');
const resetBtn = document.getElementById('reset-btn');

const world = {
  width: 2200,
  height: 1500,
  boundary: 60,
};

const defaultKeybinds = {
  throttle: 'KeyW',
  reverse: 'KeyS',
  left: 'KeyA',
  right: 'KeyD',
  boost: 'ShiftLeft',
  jump: 'Space',
  rollLeft: 'KeyQ',
  rollRight: 'KeyE',
  reset: 'KeyR',
};

const keybinds = { ...defaultKeybinds };
const pressed = new Set();

const carStats = {
  octane: {
    name: 'Octane',
    color: '#58d9ff',
    accent: '#0f3f63',
    accel: 900,
    maxSpeed: 620,
    turnRate: 2.2,
    boostPower: 1200,
    jumpPower: 520,
    grip: 0.92,
    radius: 26,
  },
  fennec: {
    name: 'Fennec',
    color: '#ffb754',
    accent: '#5c3200',
    accel: 980,
    maxSpeed: 670,
    turnRate: 2.0,
    boostPower: 1280,
    jumpPower: 500,
    grip: 0.9,
    radius: 24,
  },
};

const state = {
  selectedCar: 'octane',
  camera: { x: 0, y: 0 },
  awaitingKey: null,
};

function loadBindings() {
  const raw = localStorage.getItem('rlKeybinds');
  if (!raw) return;
  try {
    const parsed = JSON.parse(raw);
    Object.assign(keybinds, parsed);
  } catch (e) {
    console.warn('Could not parse saved keybinds');
  }
}

function saveBindings() {
  localStorage.setItem('rlKeybinds', JSON.stringify(keybinds));
}

function createBall() {
  return {
    x: world.width / 2,
    y: world.height / 2 + 80,
    radius: 20,
    vx: 0,
    vy: 0,
    z: 0,
    vz: 0,
    ground: true,
  };
}

function createCar(type) {
  const stats = carStats[type];
  const centerX = world.width / 2 - 180;
  const centerY = world.height / 2;

  return {
    type,
    x: centerX,
    y: centerY,
    angle: 0,
    speed: 0,
    boost: 100,
    radius: stats.radius,
    color: stats.color,
    accent: stats.accent,
    stats,
    z: 0,
    vz: 0,
    airborne: false,
    jumpHeld: false,
    boostHeld: false,
  };
}

let car = createCar(state.selectedCar);
let ball = createBall();

function resetCar() {
  car = createCar(state.selectedCar);
  ball = createBall();
}

function setSelectedCar(type) {
  state.selectedCar = type;
  car = createCar(type);
  carNameEl.textContent = carStats[type].name;
  document.querySelectorAll('.car-btn').forEach((button) => {
    button.classList.toggle('active', button.dataset.car === type);
  });
}

function formatKey(code) {
  if (!code) return 'Unbound';
  if (code.startsWith('Key')) return code.replace('Key', '');
  if (code.startsWith('Digit')) return code.replace('Digit', '');
  if (code === 'Space') return 'Space';
  if (code === 'ShiftLeft' || code === 'ShiftRight') return 'Shift';
  if (code === 'ArrowUp') return '↑';
  if (code === 'ArrowDown') return '↓';
  if (code === 'ArrowLeft') return '←';
  if (code === 'ArrowRight') return '→';
  return code;
}

function renderControlList() {
  const actions = [
    ['Throttle', 'throttle'],
    ['Reverse', 'reverse'],
    ['Steer Left', 'left'],
    ['Steer Right', 'right'],
    ['Boost', 'boost'],
    ['Jump', 'jump'],
    ['Roll Left', 'rollLeft'],
    ['Roll Right', 'rollRight'],
    ['Reset', 'reset'],
  ];

  controlsListEl.innerHTML = '';

  actions.forEach(([label, action]) => {
    const row = document.createElement('div');
    row.className = 'control-row';

    const name = document.createElement('span');
    name.className = 'name';
    name.textContent = label;

    const bind = document.createElement('span');
    bind.className = 'bind';
    bind.textContent = formatKey(keybinds[action]);

    const button = document.createElement('button');
    button.type = 'button';
    button.textContent = 'Rebind';
    button.addEventListener('click', () => startRebind(action));

    row.appendChild(name);
    row.appendChild(bind);
    row.appendChild(button);
    controlsListEl.appendChild(row);
  });
}

function startRebind(action) {
  state.awaitingKey = action;
  document.body.style.cursor = 'pointer';
  const rows = document.querySelectorAll('.control-row');
  rows.forEach((row) => row.style.outline = 'none');
}

function onKeyDown(event) {
  const { code } = event;

  if (state.awaitingKey) {
    event.preventDefault();
    keybinds[state.awaitingKey] = code;
    saveBindings();
    renderControlList();
    state.awaitingKey = null;
    document.body.style.cursor = 'default';
    return;
  }

  if (['Space', 'ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight'].includes(code)) {
    event.preventDefault();
  }

  pressed.add(code);

  if (code === keybinds.reset) {
    resetCar();
  }
}

function onKeyUp(event) {
  pressed.delete(event.code);
}

document.addEventListener('keydown', onKeyDown);
document.addEventListener('keyup', onKeyUp);

const carButtons = document.querySelectorAll('.car-btn');
carButtons.forEach((button) => {
  button.addEventListener('click', () => {
    setSelectedCar(button.dataset.car);
  });
});

resetBtn.addEventListener('click', () => {
  resetCar();
});

function updateInput() {
  const throttle = pressed.has(keybinds.throttle);
  const reverse = pressed.has(keybinds.reverse);
  const left = pressed.has(keybinds.left);
  const right = pressed.has(keybinds.right);
  const boost = pressed.has(keybinds.boost);
  const jump = pressed.has(keybinds.jump);
  const rollLeft = pressed.has(keybinds.rollLeft);
  const rollRight = pressed.has(keybinds.rollRight);

  if (jump && !car.jumpHeld && !car.airborne) {
    car.vz = car.stats.jumpPower;
    car.airborne = true;
    car.z = 1;
  }
  car.jumpHeld = jump;

  const steeringInput = (left ? 1 : 0) - (right ? 1 : 0);
  const throttleInput = (throttle ? 1 : 0) - (reverse ? 1 : 0);

  if (!car.airborne) {
    const accelForce = throttleInput * car.stats.accel;
    car.speed += accelForce * dt;
    if (Math.abs(car.speed) > car.stats.maxSpeed) {
      car.speed = Math.sign(car.speed) * car.stats.maxSpeed;
    }

    if (throttleInput === 0) {
      car.speed *= 1 - 2.2 * dt;
      if (Math.abs(car.speed) < 3) car.speed = 0;
    }

    if (boost && car.boost > 0) {
      car.speed += car.stats.boostPower * dt;
      car.boost = Math.max(0, car.boost - 28 * dt);
      if (Math.abs(car.speed) > car.stats.maxSpeed * 1.55) {
        car.speed = Math.sign(car.speed) * car.stats.maxSpeed * 1.55;
      }
    } else {
      car.boost = Math.min(100, car.boost + 12 * dt);
    }

    car.angle += steeringInput * car.stats.turnRate * dt * (0.4 + Math.abs(car.speed) / car.stats.maxSpeed);
    car.x += Math.cos(car.angle) * car.speed * dt;
    car.y += Math.sin(car.angle) * car.speed * dt;
  } else {
    const aerialTurn = steeringInput * 1.8;
    car.angle += aerialTurn * dt * 2.1;
    car.x += Math.cos(car.angle) * (car.speed * 0.7) * dt;
    car.y += Math.sin(car.angle) * (car.speed * 0.7) * dt;

    if (rollLeft) {
      car.angle -= dt * 2.6;
    }
    if (rollRight) {
      car.angle += dt * 2.6;
    }

    car.vz -= 700 * dt;
    car.z += car.vz * dt;
    if (car.z <= 0) {
      car.z = 0;
      car.vz = 0;
      car.airborne = false;
      car.speed *= 0.9;
    }
  }

  if (car.x < world.boundary + car.radius) {
    car.x = world.boundary + car.radius;
    car.speed *= -0.25;
  }
  if (car.x > world.width - world.boundary - car.radius) {
    car.x = world.width - world.boundary - car.radius;
    car.speed *= -0.25;
  }
  if (car.y < world.boundary + car.radius) {
    car.y = world.boundary + car.radius;
    car.speed *= -0.25;
  }
  if (car.y > world.height - world.boundary - car.radius) {
    car.y = world.height - world.boundary - car.radius;
    car.speed *= -0.25;
  }
}

function updateBall(dt) {
  ball.vx *= 0.995;
  ball.vy *= 0.995;
  ball.x += ball.vx * dt;
  ball.y += ball.vy * dt;

  ball.vz -= 820 * dt;
  ball.z += ball.vz * dt;

  if (ball.z <= 0) {
    ball.z = 0;
    ball.vz *= -0.45;
    if (Math.abs(ball.vz) < 12) ball.vz = 0;
  }

  if (ball.x - ball.radius < world.boundary) {
    ball.x = world.boundary + ball.radius;
    ball.vx *= -0.78;
  }
  if (ball.x + ball.radius > world.width - world.boundary) {
    ball.x = world.width - world.boundary - ball.radius;
    ball.vx *= -0.78;
  }
  if (ball.y - ball.radius < world.boundary) {
    ball.y = world.boundary + ball.radius;
    ball.vy *= -0.78;
  }
  if (ball.y + ball.radius > world.height - world.boundary) {
    ball.y = world.height - world.boundary - ball.radius;
    ball.vy *= -0.78;
  }

  const dx = ball.x - car.x;
  const dy = ball.y - car.y;
  const dist = Math.hypot(dx, dy);
  const minDist = ball.radius + car.radius + 8;

  if (dist < minDist) {
    const nx = dx / (dist || 1);
    const ny = dy / (dist || 1);
    const overlap = minDist - dist;
    ball.x += nx * overlap;
    ball.y += ny * overlap;

    const impact = Math.max(80, Math.abs(car.speed) * 0.9);
    ball.vx += nx * impact * 0.45;
    ball.vy += ny * impact * 0.45;
    ball.vz = Math.max(ball.vz, 150);
  }

  if (Math.hypot(ball.vx, ball.vy) < 3 && ball.z === 0) {
    ball.vx *= 0.98;
    ball.vy *= 0.98;
  }
}

function updateHud() {
  const speed = Math.round(Math.abs(car.speed));
  const boostPercent = Math.round(car.boost);
  carNameEl.textContent = carStats[car.type].name;
  speedDisplay.textContent = `${speed} km/h`;
  boostDisplay.textContent = `${boostPercent}%`;
  zDisplay.textContent = `${Math.round(car.z)} m`;
  ballDisplay.textContent = ball.z > 2 ? 'Airborne' : 'Grounded';
}

function drawRoads() {
  ctx.fillStyle = '#1a2b2b';
  ctx.fillRect(0, 0, world.width, world.height);

  ctx.fillStyle = '#b9c0c3';
  ctx.fillRect(0, 170, world.width, 130);
  ctx.fillRect(0, 980, world.width, 130);
  ctx.fillRect(370, 0, 120, world.height);
  ctx.fillRect(1160, 0, 120, world.height);

  ctx.fillStyle = '#3b4044';
  ctx.fillRect(0, 240, world.width, 20);
  ctx.fillRect(0, 1010, world.width, 20);
  ctx.fillRect(440, 0, 20, world.height);
  ctx.fillRect(1230, 0, 20, world.height);

  ctx.fillStyle = '#0d1d28';
  ctx.fillRect(130, 300, 260, 700);
  ctx.fillRect(830, 300, 260, 700);
  ctx.fillRect(1520, 300, 260, 700);

  ctx.fillStyle = '#355b69';
  for (let i = 0; i < 18; i++) {
    ctx.fillRect(180 + i * 90, 350, 50, 150);
    ctx.fillRect(180 + i * 90, 980, 50, 150);
  }

  ctx.fillStyle = '#649a46';
  for (let i = 0; i < 26; i++) {
    ctx.fillRect(100 + i * 80, 125, 16, 80);
    ctx.fillRect(100 + i * 80, 1290, 16, 80);
  }
}

function drawBackground() {
  ctx.save();
  ctx.translate(-state.camera.x, -state.camera.y);
  drawRoads();
  ctx.restore();
}

function drawBall() {
  const sx = ball.x - state.camera.x;
  const sy = ball.y - state.camera.y - ball.z * 0.65;

  ctx.save();
  ctx.translate(sx, sy);
  ctx.fillStyle = '#dce8f5';
  ctx.beginPath();
  ctx.arc(0, 0, ball.radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.strokeStyle = '#6d89a6';
  ctx.lineWidth = 4;
  ctx.stroke();

  ctx.fillStyle = '#c7e8ff';
  ctx.beginPath();
  ctx.arc(-7, -7, 6, 0, Math.PI * 2);
  ctx.fill();
  ctx.restore();
}

function drawCar() {
  const sx = car.x - state.camera.x;
  const sy = car.y - state.camera.y - car.z * 0.7;

  ctx.save();
  ctx.translate(sx, sy);
  ctx.rotate(car.angle);

  ctx.fillStyle = 'rgba(0,0,0,0.18)';
  ctx.beginPath();
  ctx.ellipse(0, 18, 30, 18, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = car.color;
  ctx.fillRect(-20, -12, 40, 24);

  ctx.fillStyle = car.accent;
  ctx.fillRect(-10, -7, 20, 14);

  ctx.fillStyle = '#f8f9fb';
  ctx.fillRect(-17, -9, 6, 5);
  ctx.fillRect(11, -9, 6, 5);
  ctx.fillRect(-17, 4, 6, 5);
  ctx.fillRect(11, 4, 6, 5);

  ctx.restore();
}

function drawHudText() {
  ctx.save();
  ctx.fillStyle = 'rgba(0,0,0,0.25)';
  ctx.fillRect(18, 18, 190, 92);
  ctx.fillStyle = '#f1fbff';
  ctx.font = 'bold 20px sans-serif';
  ctx.fillText('LOS SANTOS', 30, 46);
  ctx.font = '15px sans-serif';
  ctx.fillText(`Car: ${carStats[car.type].name}`, 30, 72);
  ctx.fillText(`Boost: ${Math.round(car.boost)}%`, 30, 92);
  ctx.restore();
}

function draw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  state.camera.x = clamp(car.x - canvas.width / 2, 0, world.width - canvas.width);
  state.camera.y = clamp(car.y - canvas.height / 2, 0, world.height - canvas.height);

  drawBackground();
  drawBall();
  drawCar();
  drawHudText();
}

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

let lastTime = 0;
let dt = 0;

function gameLoop(timestamp) {
  dt = Math.min((timestamp - lastTime) / 1000 || 0.016, 0.033);
  lastTime = timestamp;

  updateInput();
  updateBall(dt);
  updateHud();
  draw();

  requestAnimationFrame(gameLoop);
}

loadBindings();
renderControlList();
setSelectedCar(state.selectedCar);
requestAnimationFrame(gameLoop);

window.addEventListener('blur', () => {
  pressed.clear();
});
