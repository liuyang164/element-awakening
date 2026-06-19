<script setup>
import { computed, onMounted, onUnmounted, reactive, ref } from 'vue';

const WORLD_WIDTH = 1800;
const FLOOR_Y = 392;
const PLAYER_SPEED = 4.8;
const ATTACK_RANGE = 98;

const elementInfo = {
  poison: { name: '毒雾', color: '#72d94d', effect: '攻击附带毒雾伤害' },
  vine: { name: '青藤', color: '#36b96a', effect: '攻击短暂缠绕敌人' },
  ember: { name: '火星', color: '#ff7a2f', effect: '提高剑光伤害' },
  shell: { name: '硬壳', color: '#c49455', effect: '降低受到的伤害' },
  gale: { name: '疾风', color: '#74d8ff', effect: '移动速度提高' },
  ooze: { name: '蠕动', color: '#9bd15c', effect: '抗击退但移动变慢' },
  bossVine: { name: '藤王核心', color: '#21e08b', effect: '永久装配：青藤剑气' },
};

const monsterTemplates = [
  { name: '毒藤蛇妖', icon: '蛇', x: 520, hp: 38, elements: ['poison', 'ooze'], speed: 0.55 },
  { name: '刺壳菇', icon: '菇', x: 690, hp: 34, elements: ['shell'], speed: 0.32 },
  { name: '青藤小鬼', icon: '藤', x: 860, hp: 42, elements: ['vine'], speed: 0.8 },
  { name: '火尾狐', icon: '火', x: 1060, hp: 36, elements: ['ember', 'gale'], speed: 1.08 },
  { name: '雾眼蝠', icon: '蝠', x: 1240, hp: 30, elements: ['poison', 'gale'], speed: 1.2 },
  { name: '硬甲虫', icon: '甲', x: 1420, hp: 52, elements: ['shell', 'ooze'], speed: 0.42 },
];

const keys = reactive({});
const player = reactive({
  x: 120,
  y: FLOOR_Y,
  hp: 120,
  maxHp: 120,
  facing: 1,
  attacking: false,
  attackTimer: 0,
  hitCooldown: 0,
  elements: [],
  permanent: [],
});

const monsters = ref([]);
const drops = ref([]);
const effects = ref([]);
const boss = reactive({
  name: '青藤守卫',
  icon: '王',
  x: 1620,
  y: FLOOR_Y - 16,
  hp: 160,
  maxHp: 160,
  elements: ['bossVine', 'poison', 'vine'],
  alive: false,
  active: false,
  speed: 0.72,
  attackCooldown: 0,
});

const message = ref('方向键 / A D 移动，J 攻击，靠近光球自动拾取元素。');
const gameState = ref('playing');
const cameraX = computed(() => Math.max(0, Math.min(WORLD_WIDTH - 960, player.x - 260)));
const aliveMonsters = computed(() => monsters.value.filter((monster) => monster.alive));
const allNormalCleared = computed(() => aliveMonsters.value.length === 0);

function resetGame() {
  player.x = 120;
  player.hp = player.maxHp;
  player.facing = 1;
  player.attacking = false;
  player.attackTimer = 0;
  player.hitCooldown = 0;
  player.elements = [];
  player.permanent = [];
  drops.value = [];
  effects.value = [];
  monsters.value = monsterTemplates.map((template, index) => ({
    ...template,
    id: index + 1,
    y: FLOOR_Y,
    maxHp: template.hp,
    alive: true,
    direction: index % 2 === 0 ? -1 : 1,
    attackCooldown: 0,
    stunned: 0,
  }));
  boss.hp = boss.maxHp;
  boss.alive = true;
  boss.active = false;
  boss.attackCooldown = 0;
  gameState.value = 'playing';
  message.value = '方向键 / A D 移动，J 攻击，靠近光球自动拾取元素。';
}

function clamp(value, min, max) {
  return Math.max(min, Math.min(max, value));
}

function distance(a, b) {
  return Math.abs(a.x - b.x);
}

function currentDamage() {
  let damage = 18;
  if (player.elements.includes('ember')) damage += 8;
  if (player.permanent.includes('bossVine')) damage += 12;
  return damage;
}

function spawnEffect(text, x, y, color = '#fff') {
  effects.value.push({ id: crypto.randomUUID(), text, x, y, color, life: 50 });
}

function spawnDrops(entity, guaranteedBoss = false) {
  const count = guaranteedBoss ? 1 : Math.max(1, Math.round(Math.random() * entity.elements.length));
  const pool = guaranteedBoss ? ['bossVine'] : entity.elements;
  for (let i = 0; i < count; i += 1) {
    const key = pool[Math.floor(Math.random() * pool.length)];
    drops.value.push({
      id: crypto.randomUUID(),
      key,
      x: entity.x + i * 28 - 12,
      y: FLOOR_Y - 30,
      ttl: guaranteedBoss ? Infinity : 900,
    });
  }
}

function attack() {
  if (player.attackTimer > 0 || gameState.value !== 'playing') return;
  player.attacking = true;
  player.attackTimer = 18;
  const targets = [...aliveMonsters.value, boss.alive && boss.active ? boss : null].filter(Boolean);
  for (const target of targets) {
    const inFront = player.facing > 0 ? target.x >= player.x : target.x <= player.x;
    if (inFront && distance(player, target) <= ATTACK_RANGE) {
      const damage = currentDamage();
      target.hp = Math.max(0, target.hp - damage);
      target.stunned = player.elements.includes('vine') || player.permanent.includes('bossVine') ? 24 : target.stunned;
      spawnEffect(`-${damage}`, target.x, target.y - 84, '#fff3a3');
      if (player.elements.includes('poison')) {
        target.hp = Math.max(0, target.hp - 5);
        spawnEffect('毒雾', target.x + 18, target.y - 112, '#a7ff79');
      }
      if (target.hp <= 0) defeatTarget(target);
    }
  }
}

function defeatTarget(target) {
  if (!target.alive) return;
  target.alive = false;
  if (target === boss) {
    spawnDrops(target, true);
    message.value = 'Boss 被击败！拾取藤王核心完成第一关。';
    spawnEffect('Boss 击败', target.x, target.y - 130, '#fff');
    return;
  }
  spawnDrops(target);
  message.value = `${target.name} 掉落了元素，快去拾取。`;
  spawnEffect('元素掉落', target.x, target.y - 112, '#d9ff82');
}

function pickDrop(drop) {
  if (drop.key === 'bossVine') {
    if (!player.permanent.includes(drop.key)) player.permanent.push(drop.key);
    gameState.value = 'won';
    message.value = '第一关完成：获得永久元素「藤王核心」。';
  } else if (!player.elements.includes(drop.key)) {
    player.elements.push(drop.key);
    message.value = `获得临时元素「${elementInfo[drop.key].name}」：${elementInfo[drop.key].effect}`;
  }
  drops.value = drops.value.filter((item) => item.id !== drop.id);
}

function updatePlayer() {
  let move = 0;
  if (keys.ArrowLeft || keys.KeyA) move -= 1;
  if (keys.ArrowRight || keys.KeyD) move += 1;
  const speed = PLAYER_SPEED + (player.elements.includes('gale') ? 1.7 : 0) - (player.elements.includes('ooze') ? 0.8 : 0);
  if (move !== 0) {
    player.facing = move;
    player.x = clamp(player.x + move * speed, 48, WORLD_WIDTH - 80);
  }
  if (keys.KeyJ) attack();
  if (player.attackTimer > 0) player.attackTimer -= 1;
  if (player.attackTimer <= 0) player.attacking = false;
  if (player.hitCooldown > 0) player.hitCooldown -= 1;
}

function updateEnemy(enemy) {
  if (!enemy.alive || enemy.stunned > 0) {
    if (enemy.stunned > 0) enemy.stunned -= 1;
    return;
  }
  const gap = player.x - enemy.x;
  const near = Math.abs(gap) < 62;
  if (near) {
    if (enemy.attackCooldown <= 0 && player.hitCooldown <= 0) {
      const damage = player.elements.includes('shell') ? 7 : 11;
      player.hp = Math.max(0, player.hp - damage);
      player.hitCooldown = 32;
      enemy.attackCooldown = 58;
      spawnEffect(`-${damage}`, player.x, player.y - 110, '#ff755f');
    }
  } else if (Math.abs(gap) < 300) {
    enemy.direction = gap > 0 ? 1 : -1;
    enemy.x += enemy.direction * enemy.speed;
  } else {
    enemy.x += enemy.direction * enemy.speed * 0.5;
    if (enemy.x < 460 || enemy.x > 1510) enemy.direction *= -1;
  }
  if (enemy.attackCooldown > 0) enemy.attackCooldown -= 1;
}

function updateDrops() {
  const remainingDrops = [];
  for (const drop of drops.value) {
    drop.ttl -= 1;
    if (Math.abs(player.x - drop.x) < 38) {
      pickDrop(drop);
      continue;
    }

    const enemyPicker = [...aliveMonsters.value, boss.alive && boss.active ? boss : null]
      .filter(Boolean)
      .find((enemy) => drop.key !== 'bossVine' && Math.abs(enemy.x - drop.x) < 34);
    if (enemyPicker) {
      if (!enemyPicker.elements.includes(drop.key)) enemyPicker.elements.push(drop.key);
      spawnEffect(`拾取${elementInfo[drop.key].name}`, enemyPicker.x, enemyPicker.y - 108, elementInfo[drop.key].color);
      message.value = `${enemyPicker.name} 拾取了「${elementInfo[drop.key].name}」。`;
      continue;
    }

    if (drop.ttl > 0) remainingDrops.push(drop);
  }
  drops.value = remainingDrops;
}

function updateBoss() {
  if (!boss.alive) return;
  if (allNormalCleared.value && !boss.active) {
    boss.active = true;
    message.value = '第一关 Boss「青藤守卫」苏醒了。';
  }
  if (!boss.active) return;
  updateEnemy(boss);
}

function updateEffects() {
  effects.value = effects.value
    .map((effect) => ({ ...effect, y: effect.y - 0.8, life: effect.life - 1 }))
    .filter((effect) => effect.life > 0);
}

function loop() {
  if (gameState.value === 'playing') {
    updatePlayer();
    for (const monster of monsters.value) updateEnemy(monster);
    updateBoss();
    updateDrops();
    updateEffects();
    if (player.hp <= 0) {
      gameState.value = 'lost';
      message.value = '挑战失败，按 R 重新开始。';
    }
  }
  requestAnimationFrame(loop);
}

function onKeyDown(event) {
  keys[event.code] = true;
  if (event.code === 'KeyR') resetGame();
}

function onKeyUp(event) {
  keys[event.code] = false;
}

onMounted(() => {
  resetGame();
  window.addEventListener('keydown', onKeyDown);
  window.addEventListener('keyup', onKeyUp);
  requestAnimationFrame(loop);
});

onUnmounted(() => {
  window.removeEventListener('keydown', onKeyDown);
  window.removeEventListener('keyup', onKeyUp);
});
</script>

<template>
  <main class="game-shell">
    <header class="hud">
      <div>
        <p class="eyebrow">元素觉醒录 MVP-001</p>
        <h1>青藤试炼林</h1>
      </div>
      <div class="status">
        <span>生命 {{ player.hp }}/{{ player.maxHp }}</span>
        <span>怪物 {{ aliveMonsters.length }}/{{ monsters.length }}</span>
        <span>Boss {{ boss.active ? boss.hp : '未苏醒' }}</span>
      </div>
    </header>

    <section class="stage" aria-label="青藤试炼林战斗场景">
      <div class="world" :style="{ transform: `translateX(${-cameraX}px)` }">
        <div class="sky-layer"></div>
        <div class="sunbeam one"></div>
        <div class="sunbeam two"></div>
        <div class="cloud cloud-a"></div>
        <div class="cloud cloud-b"></div>
        <div class="tree-line far"></div>
        <div class="tree-line near"></div>
        <div class="hut">试炼屋</div>
        <div class="sign">试炼开始</div>
        <div class="ground"></div>
        <div class="grass grass-a"></div>
        <div class="grass grass-b"></div>
        <div class="mist"></div>

        <div
          class="hero"
          :class="{ attacking: player.attacking, hurt: player.hitCooldown > 0 }"
          :style="{ left: `${player.x}px`, top: `${player.y - 96}px`, transform: `scaleX(${player.facing})` }"
        >
          <div class="hero-hair"></div>
          <div class="hero-head">剑</div>
          <div class="hero-body"></div>
          <div class="hero-sword"></div>
          <div v-if="player.attacking" class="slash"></div>
        </div>

        <div
          v-for="monster in monsters"
          :key="monster.id"
          class="monster"
          :class="{ dead: !monster.alive, stunned: monster.stunned > 0 }"
          :style="{ left: `${monster.x}px`, top: `${monster.y - 72}px` }"
        >
          <div class="name">{{ monster.name }}</div>
          <div class="monster-body">{{ monster.icon }}</div>
          <div class="hp"><span :style="{ width: `${(monster.hp / monster.maxHp) * 100}%` }"></span></div>
          <div class="badges">
            <i v-for="key in monster.elements" :key="key" :style="{ background: elementInfo[key].color }"></i>
          </div>
        </div>

        <div
          class="boss"
          :class="{ asleep: !boss.active, dead: !boss.alive }"
          :style="{ left: `${boss.x}px`, top: `${boss.y - 116}px` }"
        >
          <div class="name">{{ boss.active ? boss.name : '沉睡的青藤守卫' }}</div>
          <div class="boss-body">{{ boss.icon }}</div>
          <div class="hp"><span :style="{ width: `${(boss.hp / boss.maxHp) * 100}%` }"></span></div>
          <div class="badges">
            <i v-for="key in boss.elements" :key="key" :style="{ background: elementInfo[key].color }"></i>
          </div>
        </div>

        <button
          v-for="drop in drops"
          :key="drop.id"
          class="drop"
          :style="{ left: `${drop.x}px`, top: `${drop.y}px`, '--drop-color': elementInfo[drop.key].color }"
          :title="elementInfo[drop.key].name"
          type="button"
        >
          {{ elementInfo[drop.key].name }}
        </button>

        <div
          v-for="effect in effects"
          :key="effect.id"
          class="float-text"
          :style="{ left: `${effect.x}px`, top: `${effect.y}px`, color: effect.color }"
        >
          {{ effect.text }}
        </div>
      </div>
    </section>

    <footer class="panel">
      <div class="message">{{ message }}</div>
      <div class="elements">
        <strong>临时元素</strong>
        <span v-if="player.elements.length === 0">无</span>
        <span v-for="key in player.elements" :key="key" :style="{ borderColor: elementInfo[key].color }">
          {{ elementInfo[key].name }}
        </span>
      </div>
      <div class="elements">
        <strong>永久元素</strong>
        <span v-if="player.permanent.length === 0">无</span>
        <span v-for="key in player.permanent" :key="key" :style="{ borderColor: elementInfo[key].color }">
          {{ elementInfo[key].name }}
        </span>
      </div>
      <button class="reset" type="button" @click="resetGame">重新开始 R</button>
    </footer>
  </main>
</template>
