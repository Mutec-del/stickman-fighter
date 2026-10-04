const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const ui = {
  playerName: document.getElementById('playerName'),
  enemyName: document.getElementById('enemyName'),
  playerStyle: document.getElementById('playerStyle'),
  enemyStyle: document.getElementById('enemyStyle'),
  playerHealth: document.getElementById('playerHealth'),
  enemyHealth: document.getElementById('enemyHealth'),
  playerStamina: document.getElementById('playerStamina'),
  enemyStamina: document.getElementById('enemyStamina'),
  playerSpecial: document.getElementById('playerSpecial'),
  enemySpecial: document.getElementById('enemySpecial'),
  comboText: document.getElementById('comboText'),
  roundText: document.getElementById('roundText')
};

const WORLD = {
  width: 1800,
  height: 540,
  groundY: 420,
  left: 90,
  right: 1710
};

const GRAVITY = 1850;
const TAU = Math.PI * 2;
const clamp = (v, min, max) => Math.min(max, Math.max(min, v));
const lerp = (a, b, t) => a + (b - a) * t;
const dist = (a, b) => Math.hypot(a.x - b.x, a.y - b.y);

const profiles = {
  FLASH: {
    name: 'FLASH',
    role: 'SPEED',
    color: '#ffe066',
    stats: {
      speed: 260,
      acceleration: 1800,
      strength: 5,
      defense: 7,
      balance: 8,
      attackSpeed: 1.0,
      jump: 760,
      airControl: 1.0,
      dashDistance: 210,
      dashRecovery: 0.25,
      stamina: 100,
      knockback: 0.9,
      recovery: 0.85,
      comboPotential: 9,
      counter: 8,
      special: 100
    },
    specialName: 'Phantom Rush'
  },
  TITAN: {
    name: 'TITAN',
    role: 'STRENGTH',
    color: '#ffd59e',
    stats: {
      speed: 170,
      acceleration: 1200,
      strength: 10,
      defense: 10,
      balance: 10,
      attackSpeed: 0.72,
      jump: 650,
      airControl: 0.75,
      dashDistance: 150,
      dashRecovery: 0.42,
      stamina: 95,
      knockback: 1.35,
      recovery: 1.2,
      comboPotential: 6,
      counter: 7,
      special: 100
    },
    specialName: 'Ground Slam'
  },
  SHADOW: {
    name: 'SHADOW',
    role: 'COUNTER',
    color: '#f5b9ff',
    stats: {
      speed: 220,
      acceleration: 1700,
      strength: 6,
      defense: 9,
      balance: 8,
      attackSpeed: 0.95,
      jump: 720,
      airControl: 0.92,
      dashDistance: 180,
      dashRecovery: 0.34,
      stamina: 100,
      knockback: 1.05,
      recovery: 0.9,
      comboPotential: 8,
      counter: 9,
      special: 100
    },
    specialName: 'Vanish Counter'
  },
  VOLT: {
    name: 'VOLT',
    role: 'DASH',
    color: '#7fe0ff',
    stats: {
      speed: 240,
      acceleration: 2000,
      strength: 6,
      defense: 6,
      balance: 7,
      attackSpeed: 1.1,
      jump: 700,
      airControl: 0.9,
      dashDistance: 220,
      dashRecovery: 0.22,
      stamina: 92,
      knockback: 1.05,
      recovery: 0.82,
      comboPotential: 8,
      counter: 7,
      special: 100
    },
    specialName: 'Triple Burst'
  },
  AERO: {
    name: 'AERO',
    role: 'AERIAL',
    color: '#98f0c7',
    stats: {
      speed: 210,
      acceleration: 1600,
      strength: 6,
      defense: 7,
      balance: 8,
      attackSpeed: 0.95,
      jump: 830,
      airControl: 1.3,
      dashDistance: 175,
      dashRecovery: 0.3,
      stamina: 95,
      knockback: 1,
      recovery: 0.8,
      comboPotential: 8,
      counter: 8,
      special: 100
    },
    specialName: 'Meteor Strike'
  },
  IRON: {
    name: 'IRON',
    role: 'TANK',
    color: '#c6d0dd',
    stats: {
      speed: 150,
      acceleration: 1100,
      strength: 9,
      defense: 10,
      balance: 10,
      attackSpeed: 0.6,
      jump: 620,
      airControl: 0.7,
      dashDistance: 120,
      dashRecovery: 0.48,
      stamina: 100,
      knockback: 1.55,
      recovery: 1.35,
      comboPotential: 5,
      counter: 8,
      special: 100
    },
    specialName: 'Iron Guard'
  }
};

const keyState = {};
const pressedOnce = {};
window.addEventListener('keydown', (event) => {
  if (!keyState[event.key]) {
    pressedOnce[event.key] = true;
  }
  keyState[event.key] = true;
  if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', ' '].includes(event.key)) {
    event.preventDefault();
  }
});

window.addEventListener('keyup', (event) => {
  keyState[event.key] = false;
  pressedOnce[event.key] = false;
});

function getPlayerInput() {
  const left = keyState['a'] || keyState['A'] || keyState['ArrowLeft'];
  const right = keyState['d'] || keyState['D'] || keyState['ArrowRight'];
  const jump = keyState['w'] || keyState['W'] || keyState['ArrowUp'];
  const block = keyState['s'] || keyState['S'] || keyState['ArrowDown'];
  const dash = keyState['Shift'];
  const dodge = keyState[' '];
  const jab = pressedOnce['j'] || pressedOnce['J'];
  const heavy = pressedOnce['k'] || pressedOnce['K'];
  const kick = pressedOnce['l'] || pressedOnce['L'];
  const special = pressedOnce['e'] || pressedOnce['E'];

  return {
    left,
    right,
    jump,
    block,
    dash,
    dodge,
    jab,
    heavy,
    kick,
    special,
    justPressedJump: pressedOnce['w'] || pressedOnce['W'] || pressedOnce['ArrowUp'],
    justPressedDodge: pressedOnce[' '] 
  };
}

function resetPressedOnce() {
  Object.keys(pressedOnce).forEach((key) => {
    pressedOnce[key] = false;
  });
}

class Fighter {
  constructor(profile, x, isPlayer) {
    this.profile = profile;
    this.name = profile.name;
    this.role = profile.role;
    this.color = profile.color;
    this.x = x;
    this.y = WORLD.groundY;
    this.vx = 0;
    this.vy = 0;
    this.width = 28;
    this.height = 112;
    this.facing = x < WORLD.width / 2 ? 1 : -1;
    this.isPlayer = isPlayer;
    this.health = 100;
    this.stamina = 100;
    this.special = 0;
    this.balance = 100;
    this.momentum = 0;
    this.state = 'stable';
    this.onGround = true;
    this.attack = null;
    this.attackTimer = 0;
    this.attackCooldown = 0;
    this.dashCooldown = 0;
    this.dashTimer = 0;
    this.hitStop = 0;
    this.blocking = false;
    this.stunned = 0;
    this.recovery = 0;
    this.comboCount = 0;
    this.comboWindow = 0;
    this.hitFlash = 0;
    this.airJumpAvailable = true;
    this.perfectDodgeTimer = 0;
    this.trail = [];
    this.animTime = 0;
    this.lastMove = 0;
    this.lastHitAt = 0;
    this.knockbackScale = 1;
    this.vulnerable = true;
  }

  setFacing(opponent) {
    this.facing = this.x < opponent.x ? 1 : -1;
  }

  beginAttack(kind) {
    const data = attackLibrary[kind] || attackLibrary.jab;
    const attack = {
      kind,
      name: data.name,
      damage: data.damage,
      range: data.range,
      startup: data.startup,
      active: data.active,
      recovery: data.recovery,
      knockback: data.knockback * this.profile.stats.knockback,
      impact: data.impact,
      balanceLoss: data.balanceLoss,
      upward: data.upward || 0,
      time: 0,
      hasHit: false,
      isHeavy: data.isHeavy || false,
      facing: this.facing
    };
    this.attack = attack;
    this.attackCooldown = attack.startup + attack.active + attack.recovery;
    this.comboWindow = 0.8;
    this.momentum = clamp(this.momentum + 18, 0, 100);
    this.trail.push({ x: this.x, y: this.y, alpha: 0.7 });
  }

  updateAttack(dt, opponent) {
    if (!this.attack) return;

    this.attack.time += dt;
    const activeStart = this.attack.startup;
    const activeEnd = this.attack.startup + this.attack.active;

    if (this.attack.time >= activeStart && this.attack.time <= activeEnd && !this.attack.hasHit) {
      const dx = Math.abs(this.x - opponent.x);
      const dy = Math.abs(this.y - opponent.y);
      if (dx < this.attack.range && dy < 80) {
        this.attack.hasHit = true;
        opponent.takeHit(this, this.attack);
      }
    }

    if (this.attack.time >= activeEnd + this.attack.recovery * 0.55) {
      this.attack = null;
    }
  }

  takeHit(attacker, attack) {
    if (!this.vulnerable) return;

    let blocked = false;
    if (this.blocking && this.onGround) {
      blocked = true;
    }

    const defenseFactor = 1 - (this.profile.stats.defense * 0.04);
    let damage = attack.damage * defenseFactor;
    if (blocked) {
      damage *= 0.26;
      this.stamina = clamp(this.stamina - 20, 0, 100);
      this.balance = clamp(this.balance - attack.balanceLoss * 0.5, 0, 100);
    } else {
      this.health = clamp(this.health - damage, 0, 100);
      this.balance = clamp(this.balance - attack.balanceLoss, 0, 100);
      this.special = clamp(this.special + 7, 0, 100);
    }

    this.vx = attack.facing * (attack.knockback * (blocked ? 0.45 : 1.2)) * (1 + attacker.momentum / 100);
    this.vy = attack.upward ? -220 : -70;
    this.recovery = 0.15 + attack.impact * 0.12;
    this.stunned = blocked ? 0.12 : 0.2 + attack.impact * 0.2;
    this.hitFlash = 0.2;
    this.momentum = clamp(this.momentum + (blocked ? 6 : 14), 0, 100);

    if (this.balance <= 25) {
      this.state = 'break';
      this.stunned = Math.max(this.stunned, 0.7);
      this.vx *= 1.35;
    } else if (this.balance <= 55) {
      this.state = 'unstable';
    } else if (this.balance <= 80) {
      this.state = 'slightly unstable';
    } else {
      this.state = 'stable';
    }

    this.lastHitAt = performance.now();
    if (this.health <= 0) {
      this.health = 0;
      stage.winner = attacker;
    }
  }

  triggerPerfectDodge(attacker) {
    this.perfectDodgeTimer = 0.38;
    this.vulnerable = false;
    this.stunned = 0;
    this.trail.push({ x: this.x, y: this.y, alpha: 0.8 });
    stage.slowMo = 0.4;
    stage.cameraShake = 12;
  }

  dashBurst() {
    if (this.dashCooldown > 0) return;
    this.dashCooldown = this.profile.stats.dashRecovery;
    this.dashTimer = 0.17;
    this.vx = this.facing * (this.profile.stats.dashDistance * 1.4);
    this.momentum = clamp(this.momentum + 16, 0, 100);
    this.trail.push({ x: this.x, y: this.y, alpha: 0.8 });
  }

  doSpecial(opponent) {
    if (this.special < 45) return;
    this.special = clamp(this.special - 45, 0, 100);
    this.beginAttack('special');
    this.attack.damage *= 1.4;
    this.attack.knockback *= 1.35;
    this.attack.range *= 1.25;
    this.attack.name = this.profile.specialName;
    this.momentum = clamp(this.momentum + 20, 0, 100);
    if (this.name === 'FLASH') {
      this.vx = this.facing * 240;
    }
    if (this.name === 'TITAN') {
      stage.cameraShake = 18;
    }
    if (this.name === 'AERO') {
      this.vy = -280;
    }
    if (this.name === 'SHADOW') {
      this.triggerPerfectDodge(opponent);
    }
  }

  update(dt, opponent, controls) {
    this.animTime += dt;
    if (this.hitStop > 0) {
      this.hitStop -= dt;
    }

    if (this.dashCooldown > 0) this.dashCooldown -= dt;
    if (this.dashTimer > 0) {
      this.dashTimer -= dt;
      this.vx *= 0.94;
    }
    if (this.recovery > 0) this.recovery -= dt;
    if (this.stunned > 0) this.stunned -= dt;
    if (this.attackCooldown > 0) this.attackCooldown -= dt;
    if (this.comboWindow > 0) this.comboWindow -= dt;
    if (this.perfectDodgeTimer > 0) {
      this.perfectDodgeTimer -= dt;
      if (this.perfectDodgeTimer <= 0) {
        this.vulnerable = true;
      }
    }

    if (this.attack) {
      this.updateAttack(dt, opponent);
    }

    if (this.hitFlash > 0) this.hitFlash -= dt;
    if (this.comboWindow <= 0) this.comboCount = 0;

    const moveInput = (controls.left ? -1 : 0) + (controls.right ? 1 : 0);
    const targetSpeed = moveInput * this.profile.stats.speed;
    const acceleration = this.profile.stats.acceleration;

    if (!this.attack && this.stunned <= 0) {
      if (moveInput !== 0) {
        this.vx = lerp(this.vx, targetSpeed, clamp(dt * (acceleration / 420), 0, 1));
        this.facing = moveInput > 0 ? 1 : -1;
      } else {
        this.vx = lerp(this.vx, 0, dt * 10);
      }
    }

    if (controls.jump && this.onGround && !this.attack && this.stunned <= 0) {
      this.onGround = false;
      this.vy = -this.profile.stats.jump;
      this.airJumpAvailable = true;
    }

    if (controls.dash && !this.attack && this.stunned <= 0) {
      this.dashBurst();
    }

    if (controls.block && !this.attack && this.onGround && this.stunned <= 0) {
      this.blocking = true;
      this.stamina = clamp(this.stamina + dt * 12, 0, 100);
    } else {
      this.blocking = false;
    }

    if (controls.jab && !this.attack && this.attackCooldown <= 0 && this.stunned <= 0) {
      this.beginAttack('jab');
      this.attackCooldown = 0.25;
    }
    if (controls.heavy && !this.attack && this.attackCooldown <= 0 && this.stunned <= 0) {
      this.beginAttack('heavy');
      this.attackCooldown = 0.35;
    }
    if (controls.kick && !this.attack && this.attackCooldown <= 0 && this.stunned <= 0) {
      this.beginAttack('kick');
      this.attackCooldown = 0.3;
    }
    if (controls.special && !this.attack && this.special >= 45 && this.stunned <= 0) {
      this.doSpecial(opponent);
    }

    this.vy += GRAVITY * dt;
    this.x += this.vx * dt;
    this.y += this.vy * dt;

    if (this.x < WORLD.left) {
      this.x = WORLD.left;
      this.vx *= -0.08;
    }
    if (this.x > WORLD.right) {
      this.x = WORLD.right;
      this.vx *= -0.08;
    }

    if (this.y >= WORLD.groundY) {
      this.y = WORLD.groundY;
      this.vy = 0;
      this.onGround = true;
      this.airJumpAvailable = true;
      if (Math.abs(this.vx) > 70) {
        this.balance = clamp(this.balance - Math.abs(this.vx) * 0.005, 0, 100);
      }
    }

    if (!this.onGround && this.y > WORLD.groundY + 220) {
      this.y = WORLD.groundY;
      this.vy = 0;
    }

    this.trail.push({ x: this.x, y: this.y, alpha: 0.76 });
    if (this.trail.length > 10) this.trail.shift();
    this.trail.forEach((t) => (t.alpha *= 0.93));

    this.balance = clamp(this.balance + (this.onGround ? 4 : 1.5) * dt * (this.blocking ? 0.25 : 1), 0, 100);
    this.momentum = clamp(this.momentum * 0.995 - dt * 1.5, 0, 100);
    this.special = clamp(this.special + dt * 7, 0, 100);
    this.stamina = clamp(this.stamina + dt * 8, 0, 100);

    if (this.balance <= 25) this.state = 'break';
    else if (this.balance <= 55) this.state = 'unstable';
    else if (this.balance <= 80) this.state = 'slightly unstable';
    else this.state = 'stable';

    if (this.health <= 0) {
      this.health = 0;
    }
  }

  draw(cameraX) {
    const drawX = this.x - cameraX;
    const drawY = this.y;
    const stride = Math.sin(this.animTime * 10) * (this.onGround ? Math.min(Math.abs(this.vx) / 140, 1.3) : 0.35);

    this.trail.forEach((t, idx) => {
      const alpha = Math.max(0, t.alpha * 0.4);
      ctx.fillStyle = `rgba(255,255,255,${alpha})`;
      ctx.beginPath();
      ctx.arc(t.x - cameraX, t.y, 2 + idx * 0.3, 0, TAU);
      ctx.fill();
    });

    ctx.save();
    ctx.translate(drawX, drawY);
    ctx.lineCap = 'round';
    ctx.lineJoin = 'round';
    ctx.strokeStyle = this.color;
    ctx.lineWidth = 3;

    const facingScale = this.facing;
    ctx.scale(facingScale, 1);
    const s = this.attack ? 1.04 : 1;
    const headY = -94;
    const torsoBase = -40;
    const hipY = 4;
    const armSwing = this.attack ? 0.85 : stride * 1.4;
    const legSwing = this.attack ? 0.6 : stride;

    // body
    ctx.beginPath();
    ctx.moveTo(0, torsoBase);
    ctx.lineTo(0, hipY);
    ctx.stroke();

    // head
    ctx.beginPath();
    ctx.arc(0, headY, 12, 0, TAU);
    ctx.stroke();

    // left arm
    ctx.beginPath();
    ctx.moveTo(0, torsoBase - 6);
    ctx.lineTo(-18, torsoBase + 16 + armSwing * 8);
    ctx.stroke();

    // right arm
    ctx.beginPath();
    ctx.moveTo(0, torsoBase - 6);
    ctx.lineTo(19, torsoBase + 14 + armSwing * 9);
    ctx.stroke();

    // left leg
    ctx.beginPath();
    ctx.moveTo(0, hipY);
    ctx.lineTo(-10, hipY + 42 + legSwing * 12);
    ctx.stroke();

    // right leg
    ctx.beginPath();
    ctx.moveTo(0, hipY);
    ctx.lineTo(10, hipY + 40 - legSwing * 12);
    ctx.stroke();

    // attack emphasis
    if (this.attack) {
      ctx.strokeStyle = '#fff5b8';
      ctx.lineWidth = 2.5;
      const r = this.attack.range * 0.9;
      ctx.beginPath();
      ctx.arc(this.facing * (r * 0.3), torsoBase, 6, 0, TAU);
      ctx.stroke();
    }

    if (this.blocking) {
      ctx.strokeStyle = '#9ad5ff';
      ctx.lineWidth = 2.2;
      ctx.beginPath();
      ctx.moveTo(0, torsoBase - 8);
      ctx.lineTo(20, torsoBase - 26);
      ctx.stroke();
    }

    ctx.restore();
  }
}

const attackLibrary = {
  jab: { name: 'Jab', damage: 8, range: 56, startup: 0.08, active: 0.09, recovery: 0.18, knockback: 90, impact: 1, balanceLoss: 11 },
  heavy: { name: 'Heavy Punch', damage: 15, range: 64, startup: 0.12, active: 0.12, recovery: 0.28, knockback: 160, impact: 2.2, balanceLoss: 19 },
  kick: { name: 'Kick', damage: 13, range: 72, startup: 0.12, active: 0.12, recovery: 0.24, knockback: 145, impact: 2, balanceLoss: 17 },
  special: { name: 'Special', damage: 18, range: 92, startup: 0.1, active: 0.2, recovery: 0.28, knockback: 220, impact: 3, balanceLoss: 25 }
};

const player = new Fighter(profiles.FLASH, 320, true);
const enemy = new Fighter(profiles.TITAN, 1280, false);
const fighters = [player, enemy];

const stage = {
  cameraX: 0,
  shake: 0,
  slowMo: 0,
  winner: null,
  round: 1,
  timer: 90
};

function refreshHud() {
  ui.playerName.textContent = player.name;
  ui.enemyName.textContent = enemy.name;
  ui.playerStyle.textContent = player.role;
  ui.enemyStyle.textContent = enemy.role;

  ui.playerHealth.style.width = `${player.health}%`;
  ui.enemyHealth.style.width = `${enemy.health}%`;
  ui.playerStamina.style.width = `${player.stamina}%`;
  ui.enemyStamina.style.width = `${enemy.stamina}%`;
  ui.playerSpecial.style.width = `${player.special}%`;
  ui.enemySpecial.style.width = `${enemy.special}%`;
  ui.comboText.textContent = `COMBO ${Math.max(player.comboCount, enemy.comboCount)}`;
  ui.roundText.textContent = `ROUND ${stage.round}`;
}

function updateArena(dt) {
  player.setFacing(enemy);
  enemy.setFacing(player);

  const input = getPlayerInput();
  const playerControls = {
    left: input.left,
    right: input.right,
    jump: input.justPressedJump,
    block: input.block,
    dash: input.dash,
    dodge: input.justPressedDodge,
    jab: input.jab,
    heavy: input.heavy,
    kick: input.kick,
    special: input.special
  };

  const enemyControls = {
    left: enemy.x > player.x + 35,
    right: enemy.x < player.x - 35,
    jump: false,
    block: Math.abs(player.x - enemy.x) < 90 && enemy.attack === null,
    dash: Math.abs(player.x - enemy.x) > 180 && enemy.dashCooldown <= 0,
    dodge: false,
    jab: Math.abs(player.x - enemy.x) < 70 && enemy.attack === null && Math.random() < 0.02,
    heavy: Math.abs(player.x - enemy.x) < 82 && enemy.attack === null && Math.random() < 0.013,
    kick: Math.abs(player.x - enemy.x) < 92 && enemy.attack === null && Math.random() < 0.018,
    special: Math.abs(player.x - enemy.x) < 110 && enemy.special > 45 && Math.random() < 0.006
  };

  if (player.health > 0 && enemy.health > 0) {
    player.update(dt, enemy, playerControls);
    enemy.update(dt, player, enemyControls);

    if (Math.abs(player.x - enemy.x) < 60 && player.attack && !player.attack.hasHit && player.attack.time > 0.08) {
      if (enemy.blocking) {
        player.attack.hasHit = true;
        enemy.takeHit(player, { ...player.attack, damage: player.attack.damage * 0.25, knockback: 45, impact: 0.5, balanceLoss: 8 });
      }
    }

    if (Math.abs(player.x - enemy.x) < 60 && enemy.attack && !enemy.attack.hasHit && enemy.attack.time > 0.08) {
      if (player.blocking) {
        enemy.attack.hasHit = true;
        player.takeHit(enemy, { ...enemy.attack, damage: enemy.attack.damage * 0.25, knockback: 50, impact: 0.5, balanceLoss: 8 });
      }
    }

    if (player.attack && player.attack.hasHit && player.comboWindow > 0) {
      player.comboCount += 1;
      player.comboWindow = 0.8;
    }
    if (enemy.attack && enemy.attack.hasHit && enemy.comboWindow > 0) {
      enemy.comboCount += 1;
      enemy.comboWindow = 0.8;
    }
  }

  stage.cameraX = lerp(stage.cameraX, (player.x + enemy.x) / 2 - canvas.width / 2, 0.08);
  stage.cameraX = clamp(stage.cameraX, 0, WORLD.width - canvas.width);
  stage.cameraX += (Math.random() - 0.5) * stage.shake;
  stage.shake *= 0.75;

  if (stage.slowMo > 0) {
    stage.slowMo -= dt;
  }

  stage.timer -= dt;
  refreshHud();
  resetPressedOnce();
}

function drawBackground() {
  ctx.fillStyle = '#0b1016';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  const skyGrad = ctx.createLinearGradient(0, 0, 0, canvas.height);
  skyGrad.addColorStop(0, '#1a2330');
  skyGrad.addColorStop(1, '#0a1017');
  ctx.fillStyle = skyGrad;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  for (let i = 0; i < 18; i++) {
    const x = ((i * 74) - (stage.cameraX * 0.25)) % (canvas.width + 120);
    const h = 24 + ((i * 17) % 34);
    ctx.fillStyle = 'rgba(255,255,255,0.05)';
    ctx.fillRect(x, 26 + (i % 8) * 18, 48, h);
  }

  ctx.fillStyle = '#1d2c38';
  ctx.fillRect(0, WORLD.groundY + 20, canvas.width, canvas.height - WORLD.groundY - 20);

  ctx.fillStyle = '#4d5b6c';
  ctx.fillRect(-50, WORLD.groundY + 18, canvas.width + 100, 8);
  ctx.fillStyle = '#364756';
  ctx.fillRect(-50, WORLD.groundY + 27, canvas.width + 100, 3);
}

function drawFighters() {
  player.draw(stage.cameraX);
  enemy.draw(stage.cameraX);
}

function drawEffects() {
  if (player.attack) {
    ctx.fillStyle = 'rgba(255,255,255,0.16)';
    ctx.fillRect((player.x - stage.cameraX) - 20, WORLD.groundY + 4, 40, 6);
  }
  if (enemy.attack) {
    ctx.fillStyle = 'rgba(255,255,255,0.16)';
    ctx.fillRect((enemy.x - stage.cameraX) - 20, WORLD.groundY + 4, 40, 6);
  }
}

function render() {
  drawBackground();
  drawFighters();
  drawEffects();

  if (stage.winner) {
    ctx.fillStyle = 'rgba(0,0,0,0.5)';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = '#ffffff';
    ctx.font = '700 46px Segoe UI';
    ctx.textAlign = 'center';
    ctx.fillText(`${stage.winner.name} WINS`, canvas.width / 2, canvas.height / 2 - 10);
    ctx.font = '500 20px Segoe UI';
    ctx.fillText('Press F5 to reset fight', canvas.width / 2, canvas.height / 2 + 30);
  }
}

let lastTime = 0;
function gameLoop(timestamp) {
  const dt = Math.min((timestamp - lastTime) / 1000 || 0.016, 0.016);
  lastTime = timestamp;

  if (!stage.winner) {
    updateArena(dt);
  }
  render();
  requestAnimationFrame(gameLoop);
}

requestAnimationFrame(gameLoop);
