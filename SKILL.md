---
name: buddy-reroll
description: Use when user wants to change, reroll, or get a specific species/rarity for their Claude Code /buddy companion pet. Triggers on requests like "换宠物", "刷龙", "legendary", "重置宠物", "change buddy", "reroll pet".
---

# buddy-reroll

## Overview

Claude Code 的 `/buddy` 宠物外观由 `hash(userID + SALT)` 确定性生成，修改 `~/.claude.json` 中的 `userID` 即可换宠物。用 FNV-1a 算法可精准预测并定向刷指定物种和稀有度。

## 物种列表（18种）

`duck goose blob cat dragon octopus owl penguin turtle snail ghost axolotl capybara cactus robot rabbit mushroom chonk`

## 稀有度

| 等级 | 权重 | 概率 |
|------|------|------|
| common | 60 | ~60% |
| uncommon | 25 | ~25% |
| rare | 10 | ~10% |
| epic | 4 | ~4% |
| legendary | 1 | ~1% |

另有 1% 概率 shiny（闪光）版。

## 快速重置（随机换一个）

编辑 `~/.claude.json`，删除 `userID` 和 `companion` 字段，重启 Claude Code，执行 `/buddy`。

## 定向刷指定宠物

需要 Node.js，用 `node buddy-reroll.js` 运行。

### 核心算法（已校验）

```javascript
const SALT = 'friend-2026-401';
const SPECIES = ['duck','goose','blob','cat','dragon','octopus','owl','penguin',
                 'turtle','snail','ghost','axolotl','capybara','cactus','robot',
                 'rabbit','mushroom','chonk'];
const RARITIES = ['common','uncommon','rare','epic','legendary'];
const WEIGHTS = {common:60, uncommon:25, rare:10, epic:4, legendary:1};
const EYES = ['·','✦','×','◉','@','°'];
const HATS = ['none','crown','tophat','propeller','halo','wizard','beanie','tinyduck'];

function fnv1a(str) {
  let h = 2166136261;
  for (let i = 0; i < str.length; i++) {
    h ^= str.charCodeAt(i);
    h = Math.imul(h, 16777619) >>> 0;
  }
  return h;
}

function mulberry32(seed) {
  let K = seed >>> 0;
  return function() {
    K |= 0; K = K + 1831565813 | 0;
    let t = Math.imul(K ^ K >>> 15, 1 | K);
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  };
}

function predict(uid) {
  const rng = mulberry32(fnv1a(uid + SALT));
  const total = Object.values(WEIGHTS).reduce((a,b)=>a+b, 0);
  let r = rng() * total, rarity = 'common';
  for (const k of RARITIES) { r -= WEIGHTS[k]; if (r < 0) { rarity = k; break; } }
  const species = SPECIES[Math.floor(rng() * SPECIES.length)];
  const eye = EYES[Math.floor(rng() * EYES.length)];
  const hat = HATS[Math.floor(rng() * HATS.length)];
  const d=Math.floor(rng()*100), p=Math.floor(rng()*100),
        c=Math.floor(rng()*100), w=Math.floor(rng()*100), s=Math.floor(rng()*100);
  return { species, rarity, eye, hat, debugging:d, patience:p, chaos:c, wisdom:w, snark:s };
}
```

### 完整刷脚本

```javascript
// buddy-reroll.js — node buddy-reroll.js
const crypto = require('crypto');

// （粘贴上方核心算法代码）

const TARGET_SPECIES = 'dragon';   // 修改为目标物种
const TARGET_RARITY = 'legendary'; // 修改为目标稀有度
const WANT_COUNT = 3;

console.log(`搜索 ${TARGET_RARITY} ${TARGET_SPECIES}...`);
const results = [];
let i = 0;
while (i < 5000000 && results.length < WANT_COUNT) {
  const uid = crypto.randomBytes(32).toString('hex');
  const p = predict(uid);
  if (p.species === TARGET_SPECIES && p.rarity === TARGET_RARITY) {
    results.push({ uid, ...p });
    console.log(`\n找到第 ${results.length} 个:`);
    console.log(`  userID: ${uid}`);
    console.log(`  眼睛:${p.eye} 帽子:${p.hat}`);
    console.log(`  调试:${p.debugging} 耐心:${p.patience} 混沌:${p.chaos} 智慧:${p.wisdom} 毒舌:${p.snark}`);
  }
  i++;
}
console.log(`\n共搜索 ${i} 个 UID`);
```

### 应用选中的 userID

1. 编辑 `~/.claude.json`
2. 将 `userID` 字段替换为脚本输出的值
3. 删除 `companion` 字段
4. 重启 Claude Code，执行 `/buddy`

## 注意事项

- 登录了 Claude 账号（OAuth）时，宠物由账号 UUID 决定，改 `userID` 无效
- 算法使用 FNV-1a，**不是** `Bun.hash()`（即使安装了 Bun 也一样）
- 50万次迭代内基本能稳定找到 legendary 目标物种
