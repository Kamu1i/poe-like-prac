# PoE-Like ARPG 独立开发冷启动计划

---

## 一、技术与架构基石

### 1.1 引擎选型：Godot 4.x（C# 后端）

| 维度 | Unity | Unreal 5 | Godot 4 |
|------|-------|----------|---------|
| 2D/俯视角支持 | 中等（需大量插件） | 弱（UE 为 FPS/TPS 而生） | 原生支持，Canvas 层+TileMap |
| 同屏物体性能 | DrawInstanced 可用 | Nanite 不适用顶视角 | MultiMesh 实例化，开箱即用 |
| 编译/迭代速度 | 慢（Domain Reload） | 极慢（C++ 编译） | 快（C# 热重载） |
| 授权成本 | 年收入>$200k 需付费 | 总收入>$1M 需 5% 分成 | MIT，零成本 |
| 轻量化 | 臃肿，Editor 启动 30s+ | 极其臃肿 | Editor 秒开 |
| 独立开发友好度 | 中 | 低（编辑器复杂度爆炸） | **极高** |

**结论：选 Godot 4.x + C#，不选 GDScript。**

原因：
- 你需要处理海量数值运算（伤害公式、词缀加权、Roll 率），C# 的性能和类型安全是刚需。
- Godot 的 Node-Scene 模型非常适合技能/状态/Buff 的树状组合结构。
- Godot 原生支持 IL weaving（`[Signal]`、`[Export]`），MVVM 风格的数据绑定对装备/词缀 UI 极有利。

**不推荐 Unity 的理由**：DOTS/ECS 一直在重构，1.0 不稳定；HDRP/URP 对 2.5D 俯视角是杀鸡用牛刀。**不推荐 Unreal 的理由**：UE 工作流设计为 3A FPS，做顶视角 ARPG 每一步都在对抗引擎。

### 1.2 架构核心：自制 ECS-Lite，不用全量 ECS Framework

**不要引入第三方 ECS 库**（如 Entitas、LeoECS）。原因：
- 你不是在做 Vampire Survivors 那种十万物件同屏的游戏。PoE-Like 的同屏怪物量在 200-800 量级，远低于纯 ECS 的阈值。
- 全量 ECS 的代价是：Debug 地狱、团队学习成本、以及与 Godot Scene 系统割裂。

**采用"数据-逻辑分离 + 组件化"的轻量方案**：

```
┌─────────────────────────────────────────────────┐
│                  Godot Scene Tree                │
│                                                  │
│  BattleWorld (Node2D)                            │
│  ├── Player (CharacterBody2D)                    │
│  │   ├── Sprite2D (AnimatedSprite)               │
│  │   ├── CollisionShape2D                        │
│  │   └── SkillComponent (Node, 持有 Skill[])      │
│  ├── MonsterSpawner (Node2D)                     │
│  │   └── Monster[N] (CharacterBody2D)            │
│  │       └── SkillComponent                      │
│  └── ProjectilePool (Node2D)                     │
│      └── Projectile[N] (Area2D)                  │
│                                                  │
│  --- 数据层 (纯 C# 对象, 不挂 Node) ---            │
│                                                  │
│  DataManager (单例, 持有所有数据容器)               │
│  ├── StatSheet        (角色属性数值)               │
│  ├── BuffContainer    (Buff/Aura 状态)             │
│  ├── SkillGemInventory (技能宝石背包)              │
│  ├── EquipmentSlots   (装备槽位)                   │
│  └── PassiveTreeState (天赋树已分配点)             │
└─────────────────────────────────────────────────┘
```

**三大核心解耦原则**：

1. **技能系统与数值彻底分离**：
   - 技能只描述"行为"：投射物运动曲线（直线/抛物线/扩散）、索敌逻辑、AOE 形状、碰撞大小。
   - 数值由 `DamagePipeline` 统一计算：`原始伤害 → 攻击加成 → 穿透/减抗 → 护盾吸收 → 生命扣除`。
   - 技能配置不写死数值，只引用 `StatKey`（如 `StatKey.PhysicalDamage`），数值在运行时从 `StatSheet` 取。

2. **Buff/Aura 系统使用标记-效果模式**：
   ```
   Buff "点燃" {
       Tag: [Elemental, Fire, DoT],
       OnApply: SpawnParticles,
       OnTick: DamagePipeline.Fire(statSheet.Get(StatKey.FireDamage) * 0.3),
       OnRemove: DespawnParticles,
       Modifiers: {
           StatKey.FireResistance: -25%  // 减抗
       }
   }
   ```
   所有 Buff 通过统一接口注册，`StatSheet.FinalValue(key)` 自动汇总所有 Buff 的 Modifier，避免手动追踪。

3. **配置表驱动一切**：
   - 技能、怪物、装备、词缀、天赋、掉落表 — 全部来自外部数据（见第四部分）。
   - 代码中不硬编码任何数值。

### 1.3 渲染与性能基线

- **渲染**：Godot MultiMesh（`MultiMeshInstance2D`）绘制同屏怪物 Sprite，一次 Draw Call 渲染最多 1023 个同类单位。
- **碰撞**：`Area2D` 做伤害判定（不是 `RigidBody2D`），避免物理引擎压力。Godot 的 Physics Server 在 2D 下性能优异，800 个 Area2D 的 overlap 检测仍可跑 60fps。
- **弹幕管理**：使用对象池（`ProjectilePool`），预分配 2000 个投射物实例，复用而非频繁实例化。

---

## 二、Stage 1：核心战斗 MVP（1-3 个月）

> **目标**：拿出一个可交互的战斗场景 — 玩家用 1 个技能杀 1 种怪，掉 1 件带随机词缀的装备，穿上后数值生效。能跑 60fps。

### 2.1 MVP 范围边界（严格砍到最少）

| 必须实现 | 明确不做 |
|---------|---------|
| 1 个玩家角色 + 四向移动 + 攻击动画 | 多职业、外观定制 |
| 1 个主动技能（如火球：投射物+爆炸AOE） | 技能树、宝石连接 |
| 1 种怪物 + 基础寻路/追击 + 死亡动画 | 怪物词缀、Boss AI |
| 1 条伤害公式链路（物/火/冰/电 + 护盾 + 抗性） | BUFF/Debuff、异常状态 |
| 1 件可掉落的随机词缀装备 | 背包系统、多部位 |
| 装备穿戴后面板数值变化 | 装备模型/外观改变 |
| 基础 UI：HP 条、技能图标、装备属性面板 | 天赋树、技能栏 |
| 同屏 200+ 怪物不掉帧 | 关卡生成、地形 |

### 2.2 MVP 技术实现 Checklist

#### 第 1 周：项目骨架搭建
- [ ] 创建 Godot 4.x C# 项目，配置 `.csproj` 引用
- [ ] 建立目录结构（见下方）
- [ ] 实现 `StatSheet` 类：`Dictionary<StatKey, float>` + `AddModifier`/`RemoveModifier`/`FinalValue`
- [ ] 定义 `StatKey` 枚举（先只定义 MVP 需要的 15-20 个 key）
- [ ] 玩家 `CharacterBody2D` + 八方向移动控制（键盘+鼠标指向）
- [ ] 相机跟随（`Camera2D` 挂玩家子节点，Smoothing enabled）

#### 第 2-3 周：战斗核心
- [ ] 实现 `DamagePipeline.Calculate()`：
  ```
  最终伤害 = 基础伤害 
           × (1 + 伤害加成%) 
           × 计算穿透后抗性 
           × 护盾吸收比例
  ```
- [ ] 实现 `SkillComponent`：持有技能列表，处理 CD/GCD
- [ ] 实现第一个技能 `Fireball`：
  - 投射物生成（对象池）
  - 直线飞行 + 碰撞检测（`Area2D`）
  - 爆炸 AOE（圆形范围伤害）
- [ ] 实现 `ProjectilePool`（2000 预分配，`Visible=false` 做回收）
- [ ] 怪物基础 AI：`NavigationAgent2D` 寻路 → 进入追击范围 → 近战攻击
- [ ] 怪物受击反馈（闪烁白帧 + 击退 + 血条变化）
- [ ] 玩家受击 + 死亡/重生

#### 第 4-5 周：装备与词缀
- [ ] 定义 `AffixTemplate` 数据结构：
  ```csharp
  class AffixTemplate {
      string Name;           // "燃烧的"
      AffixType Type;        // Prefix / Suffix
      EquipmentSlot Slot;    // 可出现的装备部位
      int Tier;              // 词缀等级 (T1-T10)
      int Weight;            // 随机权重
      Dictionary<StatKey, (float min, float max)> StatMods;  // 数值范围
      List<string> Tags;     // 标签组，用于互斥和联动
  }
  ```
- [ ] 实现 `EquipmentItem` 类：`BaseType + List<AffixInstance>`
- [ ] 实现装备随机生成器 `LootGenerator`：
  - 稀有度 Roll（普通 70%/魔法 25%/稀有 5%）
  - 词缀数量 Roll（魔法 1-2 条，稀有 3-6 条）
  - 词缀 Tier Roll（按权重表）
- [ ] 实现装备掉落（怪物死亡 → `LootGenerator.Generate()` → 生成 `Area2D` 可拾取物）
- [ ] 装备栏 UI：武器的 `TextureRect` 槽位 + 属性面板
- [ ] 穿戴/卸下装备时 `StatSheet` 自动更新（装备提供 `StatMod`，装备时 `AddModifier`，卸下时 `RemoveModifier`）

#### 第 6-8 周：打磨与性能
- [ ] 同屏压力测试：500 怪物 + 1000 投射物，Profile 到 60fps 以上
- [ ] 怪物批量渲染切换为 `MultiMeshInstance2D`
- [ ] 简易怪物 Spawner（按时间/波次出怪）
- [ ] 基础 HUD：HP 球、MP 球、技能快捷键图标、Buff 图标行
- [ ] 装备属性对比面板（当前装备 vs 地上装备）
- [ ] 音效钩子（攻击/受击/死亡/掉落）— 用免费临时音效即可
- [ ] 录屏 + 试玩 10 分钟，修明显 Bug

#### MVP 目录结构建议

```
project/
├── scenes/
│   ├── battle_world.tscn
│   ├── player.tscn
│   ├── monsters/
│   │   └── zombie.tscn
│   ├── projectiles/
│   │   └── fireball.tscn
│   └── ui/
│       ├── hud.tscn
│       └── equipment_panel.tscn
├── scripts/
│   ├── core/
│   │   ├── StatSheet.cs
│   │   ├── StatKey.cs
│   │   ├── DamagePipeline.cs
│   │   └── ObjectPool.cs
│   ├── combat/
│   │   ├── SkillComponent.cs
│   │   ├── SkillBase.cs
│   │   ├── FireballSkill.cs
│   │   └── Projectile.cs
│   ├── entities/
│   │   ├── PlayerController.cs
│   │   ├── MonsterAI.cs
│   │   └── HealthComponent.cs
│   ├── items/
│   │   ├── EquipmentItem.cs
│   │   ├── AffixTemplate.cs
│   │   ├── LootGenerator.cs
│   │   └── InventoryComponent.cs
│   └── data/
│       ├── AffixDatabase.cs      (加载 JSON/CSV)
│       └── MonsterDatabase.cs
├── data/                         (配置表目录)
│   ├── affixes.json
│   ├── monsters.json
│   ├── skills.json
│   └── equipment_bases.json
└── assets/
    ├── sprites/
    ├── sounds/
    └── fonts/
```

---

## 三、Stage 2：系统骨架搭建（MVP 之后）

### 3.1 攻坚顺序与理由

优先级从高到低排列，每一项说明"为什么是这个位置"。

#### 第 1 优先级：装备词缀系统完整化

**为什么最先**：
- MVP 已有时只有 1 件装备，你需要尽快验证"千件装备产出"的体验。
- 装备系统是 ARPG 的正反馈核心回路 — 杀怪→掉装→变强→杀更强的怪。这个回路必须先闭环。
- 词缀系统会倒逼你的配置管理流程（见第四部分），早做早发现问题。

**交付物**：
- 全 10 个装备槽位（头/身/手/脚/武器/副手/项链/戒指×2/腰带）
- 50+ 基础装备类型（不同底材：攻速、基础护盾、移动速度惩罚）
- 200+ 词缀（T1-T10 分布）、前缀/后缀分离
- 通货基础（至少"重铸石"和"点金石"：洗白/随机词缀）
- 装备过滤/排序基础 UI

#### 第 2 优先级：技能树（被动天赋星图）

**为什么在装备之后**：
- 被动技能树本质是"另一种装备"：提供 StatMod，走同样的 `AddModifier`/`RemoveModifier` 管道。
- 技术上与装备系统完全同构，可以在做完装备后复用同一套 `StatSheet` 基础设施。
- 但对玩家来说，天赋树是角色 Build 的灵魂 — 必须尽快验证"有了天赋点和装备后，数值是否能产生质变 Build"。

**交付物**：
- 星图数据结构（Node 有 ID、坐标、邻居列表、StatMod、Icon）
- 天赋树渲染（Godot `GraphEdit` 或自定义 `Control` 绘制）
- 天赋点分配/撤销（路径合法性检查 — 必须从已分配节点出发）
- 6 个起始位置 + 3 个外围核心大点（验证"跨区点天赋"体验）
- 重置机制（返点通货）

#### 第 3 优先级：怪物词缀与 AI 扩展

**为什么在技能树之后**：
- 怪物的"难"需要建立在玩家"Build 成形"的基础上。天赋树和装备系统不到位，你没有尺度去校准怪物强度。
- 怪物词缀的技术架构和装备词缀高度相似（都是 Modifier 系统），可以大量复用代码。

**交付物**：
- 怪物词缀系统：20+ 词缀（"迅捷"、"火焰强化"、"多重投射"、"分身"、"生命回复"、"死亡自爆"等）
- 稀有怪/精英怪识别（特殊边框 + 光效）
- AI 行为树扩展：远程怪、施法怪、逃跑怪、召唤怪
- 怪物词缀组合的显示与命名生成（"迅捷的·火焰强化·僵尸"）

#### 第 4 优先级：关卡生成

**为什么最后**：
- 关卡生成是"看起来最炫但正反馈最弱"的系统。玩家不会因为"地图生成算法很厉害"而留下来，只会因为"打怪掉装备很爽"而留下来。
- 技术上关卡生成独立性强，与核心战斗系统耦合度低，可以后置。
- 在没做好怪物和装备前，关卡生成只是空跑图，测不出好不好玩。

**交付物**：
- 基础 TileMap 模板拼接（预设房间模板 + 随机组合）
- 区域类型（室内/室外/洞穴）+ 对应的 TileSet
- 障碍物/可破坏物生成
- 关卡目标与出口（杀 Boss→开传送门→下一层）
- Minimap（小地图渲染）

### 3.2 Stage 2 整体时间预估

| 阶段 | 内容 | 预估周期 |
|------|------|---------|
| Stage 2.1 | 装备词缀完整化 | 1.5-2 个月 |
| Stage 2.2 | 被动天赋星图 | 1.5-2 个月 |
| Stage 2.3 | 怪物词缀+AI扩展 | 1-1.5 个月 |
| Stage 2.4 | 关卡生成 | 1-1.5 个月 |

Stage 2 总计约 **5-7 个月**（个人全职开发）。

---

## 四、数值与资产管理策略

### 4.1 配置管理铁律

> **代码是引擎，数据是燃料。任何出现在代码中的具体数值都是技术债。**

因此采用以下工作流：

```
Google Sheets (设计/调整)
        │
        ▼ (导出)
    CSV / JSON
        │
        ▼ (Godot 启动时加载)
    C# Runtime Data
        │
        ▼ (进入游戏)
    StatSheet / BuffContainer
```

### 4.2 Sheets 表结构设计

**核心原则**：一张表做一件事，列名即为运行时 Key。

#### 表 1：Stats 定义表
| StatKey | DisplayName | Category | DefaultValue | MinValue | MaxValue | IsPercentage |
|---------|-------------|----------|-------------|----------|----------|-------------|
| MaxLife | 最大生命 | Defense | 100 | 1 | 99999 | false |
| FireRes | 火焰抗性 | Defense | 0 | -200 | 90 | true |
| PhysDmgBonus | 物理伤害加成 | Offense | 0 | -100 | 9999 | true |

#### 表 2：装备底材表
| BaseID | Name | Slot | ImplicitStatKey | ImplicitValue | AtkSpeedBase | BaseArmour | BaseShield | DropLevel |
|--------|------|------|-----------------|---------------|-------------|------------|------------|-----------|
| rusted_sword | 生锈长剑 | Weapon | PhysDmgBonus | 20 | 1.3 | 0 | 0 | 1 |
| quilted_vest | 棉布外套 | Body | EvasionRating | 50 | 0 | 20 | 10 | 5 |

#### 表 3：词缀模板表
| AffixID | Name | Type | Slot | Tier | Weight | StatMod1_Key | StatMod1_Min | StatMod1_Max | TagGroup | ExclusiveGroup |
|---------|------|------|------|------|--------|-------------|-------------|-------------|----------|----------------|
| pref_fire_dmg_t1 | 燃烧的 | Prefix | Weapon | 1 | 1000 | FireDmgBonus | 10 | 25 | Fire | |
| pref_fire_dmg_t2 | 烈焰的 | Prefix | Weapon | 2 | 400 | FireDmgBonus | 26 | 40 | Fire | FireWeapon |

#### 表 4：技能宝石表
| SkillID | Name | ManaCost | Cooldown | DamageMultiplier | DamageType | ProjectileCount | AOE_Radius | SkillTags |
|---------|------|----------|----------|-----------------|------------|----------------|------------|-----------|
| fireball | 火球 | 8 | 0.85 | 1.0 | Fire | 1 | 80 | Fire, Projectile, Spell, AOE |

#### 表 5：怪物表
| MonsterID | Name | BaseHP | BaseAtk | MoveSpeed | ExpReward | DropTableID | AIBehavior |
|-----------|------|--------|---------|-----------|-----------|-------------|------------|
| zombie | 僵尸 | 50 | 8 | 80 | 20 | drop_common | MeleeChase |
| skeleton_archer | 骷髅弓箭手 | 35 | 12 | 60 | 30 | drop_common | RangedKite |

### 4.3 导出自动化

写一个简单的 Python 脚本（或 Godot EditorPlugin）做 Sheets → JSON 转换：

```python
# export_data.py
import gspread
import json

# 从 Google Sheets 读取
gc = gspread.service_account()
sh = gc.open("PoE-Like Data")

for worksheet_name in ["stats", "equipment_bases", "affixes", "skills", "monsters"]:
    ws = sh.worksheet(worksheet_name)
    records = ws.get_all_records()  # 自动取表头为 key
    with open(f"../data/{worksheet_name}.json", "w", encoding="utf-8") as f:
        json.dump(records, f, ensure_ascii=False, indent=2)
```

**运行时加载**（Godot C#）：

```csharp
public class DataManager {
    public Dictionary<string, AffixTemplate> Affixes;
    public Dictionary<string, MonsterTemplate> Monsters;
    
    public void LoadAll() {
        Affixes = JsonUtility.FromJson<List<AffixTemplate>>(
            File.ReadAllText("res://data/affixes.json")
        ).ToDictionary(a => a.AffixID);
        // ...
    }
}
```

### 4.4 数值平衡"硬规则"

1. **伤害公式写死在代码中**，但公式中的所有参数从配置表读取。改公式 = 改代码（极少发生）；调参数 = 改 Sheets（每天发生）。
2. **词缀之间通过 Tag 系统互斥**：`FireWeapon` 组内最多 1 条前缀。规则在生成代码中实现，但 Tag 定义在配置表中。
3. **怪物强度用"战力评分"做校验**：
   ```
   MonsterPower = HP × (1 + 减伤率) × 秒伤 × AI复杂度系数
   ```
   每次生成新怪物后自动跑脚本验证同等级怪物的 Power 偏差不超过 ±20%。
4. **Tier 分布严格指数递减**：T1 权重 1000，T2 权重 400，T3 权重 160，T4 权重 64... 保证极品率可控。
5. **绝对不做"多乘区"的无限放大**：所有同类加成做加法，不同类加成才做乘法，并在公式层面写死乘区数量上限。

---

## 五、避坑指南

### 5.1 第一大坑：美术资源黑洞

**症状**：花 3 个月画一个角色的 8 方向行走动画，结果游戏里根本看不清。

**现实**：ARPG 俯视角下，角色在屏幕上只有 64-96 像素高。你在 Aseprite 里逐帧抠的头发飘动细节，缩小后就是一坨色块。

**应对策略**：

1. **用色块/剪影做 Prototype，美术后置**。
   - MVP 阶段所有角色用 32×32 纯色方块 + 方向箭头表示。
   - 怪物用不同颜色和形状区分即可：红色圆形 = 近战怪，蓝色三角 = 远程怪。
   - 验证"好不好玩"不需要任何美术资源。

2. **买 Asset Pack，别自己画**。
   - itch.io、OpenGameArt、Unity Asset Store（可迁移到 Godot）有大量 2D 俯视角怪物/特效包。
   - 预算 $200-500 能买到涵盖 50+ 怪物类型的完整包。
   - **重点花在 VFX（技能特效）上**，因为玩家 80% 的视觉注意力在弹幕和爆炸。

3. **动画靠 Procedural，减少手工帧**。
   - 走路：代码做上下 Bob + 左右倾角，1 帧站立图即可模拟。
   - 攻击：Sprite 缩放+旋转+位移组成"挥砍"，不需要逐帧动画。
   - Godot 的 `AnimationPlayer` + `Tween` 可以拼出 80% 的动画需求。

### 5.2 第二大坑：性能优化死穴

**症状**：怪物一多就掉帧，于是疯狂加对象池、LOD、剔除...三个月后仍在优化，游戏还没内容。

**应对策略**：

1. **先测瓶颈，再优化 — 永远用 Profiler 说话**。
   ```python
   # Godot 内置 Profiler: Debugger → Profiler 面板
   # 关注指标：
   # - Process Time: Script 耗时 > 50% → 逻辑优化
   # - Physics Time: 碰撞耗时 > 30% → 减少碰撞体/简化形状
   # - Draw Calls: > 500 → 需要 MultiMesh 合批
   ```

2. **提前设定性能预算**：
   - 同屏怪物上限：500 只（超出直接限制生成，简单粗暴但有效）
   - 同屏投射物上限：1500 个（对象池硬上限，超限时 Oldest 先消失）
   - 粒子效果上限：100 个活跃 `GPUParticles2D`
   - 每个怪物的 AI Tick 间隔：近处 0.1s，远处 0.5s（按距离分频）

3. **MultiMesh 是 2D ARPG 的性能银弹**：
   - 所有同类型怪物共用 1 个 `MultiMeshInstance2D`。
   - 每个实例只更新 `Transform2D`（位置+旋转），不重新创建 Node。
   - Godot 4.x 的 MultiMesh 支持 per-instance color，可以用来做"稀有怪发光变色"。

4. **伤害判定用 Overlap 而非逐个碰撞**：
   ```csharp
   // 错误做法：1000 个投射物各自检测碰撞
   // 正确做法：
   var spaceState = GetWorld2D().DirectSpaceState;
   var query = new PhysicsShapeQueryParameters2D();
   query.Shape = new CircleShape2D() { Radius = aoeRadius };
   query.Transform = new Transform2D(0, explosionCenter);
   var results = spaceState.IntersectShape(query);
   // 一次查询拿到 AOE 范围内所有怪物，然后遍历发伤害
   ```

### 5.3 第三大坑：过早泛化

**症状**："我现在就做一个支持无限技能的通用框架，以后加技能就只用填表了。"

**为什么危险**：你会在 MVP 阶段花 80% 的时间写框架，最后发现框架支持的 90% 的功能，实际游戏根本不需要。更致命的是——你做出来的框架因为缺乏真实用例验证，结构本身可能就是错的。

**应对策略**：

- MVP 阶段就硬编码 1 个火球技能的字段，够用即可。加到第 5 个技能时，再抽象出 `SkillBase`。
- 加到第 3 种怪物时，再抽象出 `MonsterTemplate`。
- 加到第 10 种词缀时，再抽象出 `AffixTemplate` 的完整字段。
- **两次重复 = 照抄，三次重复 = 抽象。** 不要提前抽象。

### 5.4 第四大坑：忽视"手感"

**症状**：伤害公式完美、词缀系统精妙，但玩起来"飘"、"软"、"没有打击感"。

**这是 ARPG 的隐形杀手。** PoE 成功的秘密不只是天赋树，更是击中怪物时那一帧的画面反馈。

**MVP 阶段必须实现的"手感 Checklist"**：

- [ ] 受击时怪物 Sprite 闪白 2 帧（shader 参数 `modulate = white`）
- [ ] 受击时怪物后移 4-8px（代码 Tween）
- [ ] 击杀时怪物分裂/消散（粒子爆发）
- [ ] 屏幕震动（`Camera2D` offset Tween，幅度与伤害量正相关）
- [ ] 伤害数字弹出（`Label` 浮起 + 淡出，暴击加大加粗变色）
- [ ] 音效滞后补偿：攻击动作第 3 帧播放挥砍音效，而不是第 1 帧
- [ ] 冻结帧（Hit Stop）：高伤害命中时画面暂停 0.03-0.05 秒

这七项不改任何数值，但能让你和测试者的"好玩度"感知提升 300%。

### 5.5 第五大坑：不做自动化校验

**症状**：你手动改了 Sheets 里的一个值，忘了关联更新，进游戏发现某个 Boss 一刀秒杀玩家。你花了 3 小时排查，最后发现是某个词缀的 Tier 权重写成了 10000 而不是 1000。

**应对策略**：

写一个简单的 Data Validator 脚本（C# 或 Python），在每次加载数据后自动跑：

```csharp
public static class DataValidator {
    public static List<string> Validate(Database db) {
        var errors = new List<string>();
        
        // 1. 引用完整性：装备引用的词缀 ID 必须存在
        // 2. 数值边界：Tier 1 的 min_max 不能超过 Tier 2
        // 3. 权重总和：稀有度权重之和不能为 0
        // 4. 循环引用：A 词缀的 ExclusiveGroup 中不能包含自己
        // 5. 掉落表：所有怪物的 DropTableID 必须能找到对应表
        
        foreach (var affix in db.Affixes.Values) {
            if (affix.StatMod_Max < affix.StatMod_Min)
                errors.Add($"词缀 {affix.Name}: max < min");
            if (affix.Weight <= 0)
                errors.Add($"词缀 {affix.Name}: 权重为 0，永远不会出现");
        }
        
        return errors;
    }
}
```

在 Godot Editor 启动时自动调用，有任何 Error 直接弹窗阻断启动 — 强迫自己修好数据再进游戏。

---

## 附录：推荐工具链总览

| 用途 | 推荐工具 | 备选 |
|------|---------|------|
| 游戏引擎 | Godot 4.x (C#) | - |
| IDE | Rider / VS Code + C# Dev Kit | Visual Studio 2022 |
| 数值设计 | Google Sheets | Excel + Git LFS |
| 数据导出脚本 | Python (gspread) | C# EditorPlugin |
| 像素画 | Aseprite | Libresprite |
| 特效/VFX | Godot GPUParticles2D + 自制 | Effekseer |
| 音效 | BFXR.net (生成) + freesound.org | - |
| 版本控制 | Git + GitHub | GitLab |
| 项目管理 | Notion / Trello | GitHub Projects |
| 调试/Profile | Godot Debugger (内置) | - |
| 录屏/GIF | ScreenToGif (Windows) / Peek (Linux) | OBS |

---

> **最后一条忠告**：PoE 做了 10 年，几千人年投入。你有生之年做不出另一个 PoE。但你可以做出一个某个维度上比 PoE 更好的游戏 — 比如更好的手感、更好玩的某几个技能、更聪明的怪物 AI。找准你的游戏最独特的那个点，把 80% 的精力砸在那个点上，其余 20% 保持"够用就行"。独立开发的核心竞争力不是量，是锐度。
