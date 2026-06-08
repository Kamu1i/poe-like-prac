# PoE-Like ARPG 分步开发计划

> **使用说明**：每一步只做一件事。完成一步后打勾 `[x]`，遇到问题在该步记录备注。步骤之间尽量减少依赖，可跳步但不可合并。

---

## Phase 0：项目初始化与环境搭建

### Step 0.1 — 安装 Godot 4.x
- [ ] 从 [godotengine.org](https://godotengine.org) 下载 Godot 4.x（推荐 4.3+）的 .NET 版本（不是标准版）
- [ ] 确认文件名包含 `mono` 或 `net` 字样
- [ ] 安装到系统并启动一次，确认 Editor 正常打开

### Step 0.2 — 安装 .NET SDK
- [ ] 安装 .NET SDK 8.0（Godot 4.3+ 要求）
- [ ] 终端运行 `dotnet --version` 确认版本号 ≥ 8.0
- [ ] 运行 `dotnet --list-sdks` 确认 SDK 已注册

### Step 0.3 — 创建 Godot C# 项目
- [ ] 打开 Godot，点击"新建项目"
- [ ] 项目名称：`poe-like`（或自定义）
- [ ] 渲染器选择 `Forward+`（2D 也用这个，兼容性最好）
- [ ] 创建后关闭 Editor

### Step 0.4 — 配置 C# 项目
- [ ] 在项目根目录找到 `.csproj` 文件
- [ ] 确认 `<TargetFramework>` 为 `net8.0`（或对应版本）
- [ ] 在 Godot Editor 中点击"构建"→"生成 C# 解决方案"
- [ ] 确认无编译错误

### Step 0.5 — 配置代码编辑器
- [ ] 安装 VS Code + C# Dev Kit 扩展，或安装 JetBrains Rider
- [ ] 在 VS Code 中打开项目，确认语法高亮和 IntelliSense 正常工作
- [ ] 确认按 F5 或点击运行能从 IDE 启动 Godot

### Step 0.6 — 初始化 Git 仓库
- [ ] 项目根目录运行 `git init`
- [ ] 创建 `.gitignore` 文件，内容包含：
  - `.godot/`
  - `.import/`
  - `*.translation`
  - `export/`
  - `bin/`
  - `obj/`
  - `.vs/`
  - `.vscode/`
  - `*.user`
- [ ] 运行 `git add -A && git commit -m "init: empty Godot 4.x C# project"`
- [ ] （可选）在 GitHub 创建远程仓库并推送

---

## Phase 1：核心数据层 — StatSheet 与 StatKey

### Step 1.1 — 创建目录结构
- [ ] 在 `scripts/` 下创建子目录：`core/`、`combat/`、`entities/`、`items/`、`data/`、`ui/`
- [ ] 在 `scenes/` 下创建子目录：`monsters/`、`projectiles/`、`ui/`
- [ ] 创建 `data/` 目录（与 scripts 平级）
- [ ] 创建 `assets/sprites/`、`assets/sounds/`、`assets/fonts/` 目录
- [ ] 确认目录结构与 imp-plan.md 中的一致

### Step 1.2 — 定义 StatKey 枚举
- [ ] 创建文件 `scripts/core/StatKey.cs`
- [ ] 定义 `enum StatKey`，只包含 MVP 需要的 15-20 个 key：
  ```csharp
  public enum StatKey
  {
      // 防御
      MaxLife, MaxMana, MaxShield,
      Armour, EvasionRating,
      FireResistance, ColdResistance, LightningResistance, ChaosResistance,
      // 攻击
      PhysicalDamageBonus, FireDamageBonus, ColdDamageBonus,
      LightningDamageBonus, ChaosDamageBonus,
      AttackSpeed, CastSpeed, CriticalChance, CriticalMultiplier,
      // 资源
      ManaRegeneration, LifeRegeneration,
      // 移动
      MovementSpeed
  }
  ```
- [ ] 确认编译通过

### Step 1.3 — 实现 StatSheet 类的 AddModifier 方法
- [ ] 创建文件 `scripts/core/StatSheet.cs`
- [ ] 定义 `StatMod` 结构体：`{ StatKey Key; float Value; object Source; }`
- [ ] `StatSheet` 类内部维护 `private List<StatMod> _modifiers`
- [ ] 实现 `public void AddModifier(StatKey key, float value, object source)`
- [ ] 方法内部：`_modifiers.Add(new StatMod { Key = key, Value = value, Source = source })`
- [ ] 确认编译通过

### Step 1.4 — 实现 StatSheet 类的 RemoveModifier 方法
- [ ] 实现 `public void RemoveModifier(StatKey key, object source)`
- [ ] 方法内部：`_modifiers.RemoveAll(m => m.Key == key && m.Source == source)`
- [ ] 确认编译通过

### Step 1.5 — 实现 StatSheet 类的 RemoveAllFromSource 方法
- [ ] 实现 `public void RemoveAllFromSource(object source)`
- [ ] 方法内部：`_modifiers.RemoveAll(m => m.Source == source)`
- [ ] 这样做装备卸下时可以一步清除所有该装备提供的属性
- [ ] 确认编译通过

### Step 1.6 — 实现 StatSheet 类的 FinalValue 方法
- [ ] 实现 `public float FinalValue(StatKey key)`
- [ ] 方法内部：
  1. 取该 key 的 base value（从一个 `Dictionary<StatKey, float> _baseValues` 取，没有则返回 0）
  2. 遍历 `_modifiers.Where(m => m.Key == key)`，累加所有 value
  3. 返回 base + sum
- [ ] 确认编译通过

### Step 1.7 — 实现 StatSheet 的 BaseValue 存取
- [ ] 添加 `private Dictionary<StatKey, float> _baseValues = new()`
- [ ] 实现 `public void SetBaseValue(StatKey key, float value)`
- [ ] 实现 `public float GetBaseValue(StatKey key)`
- [ ] 确认编译通过

### Step 1.8 — 为 StatSheet 添加 OnStatChanged 事件
- [ ] 定义 `public event Action<StatKey> OnStatChanged`
- [ ] 在 `AddModifier` 和 `RemoveModifier` 末尾调用 `OnStatChanged?.Invoke(key)`
- [ ] 此事件供 UI 和 Buff 系统订阅，值变化时自动刷新
- [ ] 确认编译通过

---

## Phase 2：玩家角色与相机

### Step 2.1 — 创建战斗世界场景
- [ ] 在 `scenes/` 下创建 `battle_world.tscn`
- [ ] 根节点类型：`Node2D`，命名为 `BattleWorld`
- [ ] 添加一个 `ColorRect` 做临时地板，背景色设为深灰色，大小 2000×2000 像素
- [ ] 保存场景

### Step 2.2 — 创建玩家场景
- [ ] 在 `scenes/` 下创建 `player.tscn`
- [ ] 根节点类型：`CharacterBody2D`，命名为 `Player`
- [ ] 添加子节点 `CollisionShape2D`，Shape 选 `CircleShape2D`，半径 16px
- [ ] 添加子节点 `Sprite2D`，先用 Godot 内置 icon.svg 或 32×32 纯色方块占位

### Step 2.3 — 实现 PlayerController 的移动逻辑
- [ ] 创建文件 `scripts/entities/PlayerController.cs`
- [ ] 挂载到 Player 节点
- [ ] `_Ready()` 中缓存 `CharacterBody2D`、`Sprite2D` 引用
- [ ] `_PhysicsProcess(double delta)` 中读取输入：
  ```csharp
  var input = Input.GetVector("ui_left", "ui_right", "ui_up", "ui_down");
  var velocity = input * MoveSpeed;
  Velocity = velocity;
  MoveAndSlide();
  ```
- [ ] 默认 `MoveSpeed` 设为 200f
- [ ] 运行测试：确认能用 WASD/方向键移动 Player

### Step 2.4 — 实现鼠标朝向
- [ ] 在 `PlayerController._Process(double delta)` 中：
  ```csharp
  var mousePos = GetGlobalMousePosition();
  var direction = (mousePos - GlobalPosition).Normalized();
  Sprite.FlipH = direction.X < 0;
  ```
- [ ] 运行测试：Player 始终面向鼠标

### Step 2.5 — 实现相机跟随
- [ ] 在 Player 场景中添加子节点 `Camera2D`
- [ ] `Camera2D` 勾选 `Enabled`
- [ ] 设置 `Position Smoothing → Enabled = true`
- [ ] 设置 `Drag → Horizontal = 0.1`, `Drag → Vertical = 0.1`
- [ ] 运行测试：相机平滑跟随 Player 移动

### Step 2.6 — 设置相机边界
- [ ] 在 `Camera2D` 上设置 `Limit Left/Top/Right/Bottom`：
  - Left: 0, Top: 0, Right: 2000, Bottom: 2000
- [ ] 运行测试：相机不会超出世界范围

### Step 2.7 — 将 Player 添加到 BattleWorld
- [ ] 打开 `battle_world.tscn`
- [ ] 将 `player.tscn` 作为子节点拖入 BattleWorld
- [ ] Player 初始位置放在世界中央（1000, 1000）
- [ ] 运行测试：战斗世界 + 可移动的 Player + 相机跟随

---

## Phase 3：伤害计算管道

### Step 3.1 — 定义伤害类型枚举
- [ ] 创建文件 `scripts/core/DamageType.cs`
- [ ] 定义 `enum DamageType { Physical, Fire, Cold, Lightning, Chaos }`
- [ ] 确认编译通过

### Step 3.2 — 定义伤害输入结构体
- [ ] 创建文件 `scripts/core/DamageInput.cs`
- [ ] 定义 `struct DamageInput`：
  ```csharp
  public struct DamageInput
  {
      public float BaseDamage;          // 技能基础伤害
      public DamageType Type;           // 伤害类型
      public float AddedDamageBonus;    // 来自属性的加成比例
      public float PenetrationPercent;  // 穿透比例
      public object Attacker;           // 攻击者
  }
  ```
- [ ] 确认编译通过

### Step 3.3 — 定义伤害输出结构体
- [ ] 在 `scripts/core/DamageInput.cs` 同文件中定义 `struct DamageOutput`：
  ```csharp
  public struct DamageOutput
  {
      public float FinalDamage;
      public float DamageToShield;
      public float DamageToLife;
      public bool IsCritical;
      public float ResistanceAfterCalc;
  }
  ```
- [ ] 确认编译通过

### Step 3.4 — 实现 DamagePipeline 框架
- [ ] 创建文件 `scripts/core/DamagePipeline.cs`
- [ ] 实现 `public static DamageOutput Calculate(DamageInput input, StatSheet attackerStats, StatSheet defenderStats)`
- [ ] 方法内部先返回一个硬编码的 `DamageOutput`（用于测试编译）
- [ ] 确认编译通过

### Step 3.5 — 实现伤害加成计算（攻击方加成一侧）
- [ ] 在 `Calculate` 中实现：
  ```csharp
  StatKey bonusKey = input.Type switch {
      DamageType.Physical => StatKey.PhysicalDamageBonus,
      DamageType.Fire => StatKey.FireDamageBonus,
      DamageType.Cold => StatKey.ColdDamageBonus,
      DamageType.Lightning => StatKey.LightningDamageBonus,
      DamageType.Chaos => StatKey.ChaosDamageBonus,
      _ => StatKey.PhysicalDamageBonus
  };
  float damageBonus = attackerStats.FinalValue(bonusKey) / 100f;
  float amplified = input.BaseDamage * (1f + damageBonus);
  ```
- [ ] 确认编译通过

### Step 3.6 — 实现抗性计算（防御方抗性一侧）
- [ ] 在 `Calculate` 中继续实现抗性计算：
  ```csharp
  StatKey resKey = input.Type switch {
      DamageType.Fire => StatKey.FireResistance,
      DamageType.Cold => StatKey.ColdResistance,
      DamageType.Lightning => StatKey.LightningResistance,
      DamageType.Chaos => StatKey.ChaosResistance,
      _ => StatKey.FireResistance
  };
  float resistance = defenderStats.FinalValue(resKey) / 100f;
  float effectiveRes = Mathf.Max(-2f, Mathf.Min(0.9f, resistance - input.PenetrationPercent));
  float afterResist = amplified * (1f - effectiveRes);
  ```
- [ ] 注意：Physical 伤害应使用 `StatKey.Armour` 计算减伤
- [ ] 确认编译通过

### Step 3.7 — 实现护盾吸收计算
- [ ] 在 `Calculate` 中继续实现护盾吸收：
  ```csharp
  float currentShield = defenderStats.FinalValue(StatKey.MaxShield);
  float shieldRatio = 0.25f;
  float damageToShield = afterResist * shieldRatio;
  float damageToLife = afterResist - damageToShield;
  ```
- [ ] 确认编译通过

### Step 3.8 — 实现暴击判定
- [ ] 在 `Calculate` 中实现暴击：
  ```csharp
  bool isCritical = false;
  float critChance = attackerStats.FinalValue(StatKey.CriticalChance) / 100f;
  if (GD.Randf() < critChance) {
      float critMulti = attackerStats.FinalValue(StatKey.CriticalMultiplier) / 100f;
      damageToShield *= critMulti;
      damageToLife *= critMulti;
      isCritical = true;
  }
  ```
- [ ] 确认编译通过

### Step 3.9 — 组装最终 DamageOutput 并返回
- [ ] 在 `Calculate` 末尾组装结果：
  ```csharp
  return new DamageOutput {
      FinalDamage = damageToShield + damageToLife,
      DamageToShield = damageToShield,
      DamageToLife = damageToLife,
      IsCritical = isCritical,
      ResistanceAfterCalc = effectiveRes
  };
  ```
- [ ] 确认编译通过

### Step 3.10 — 手动验证 DamagePipeline 计算
- [ ] 在 `_Ready()` 中写临时测试代码：
  ```csharp
  var atk = new StatSheet();
  atk.SetBaseValue(StatKey.FireDamageBonus, 50);
  var def = new StatSheet();
  def.SetBaseValue(StatKey.FireResistance, 30);
  var input = new DamageInput { BaseDamage = 100, Type = DamageType.Fire };
  var result = DamagePipeline.Calculate(input, atk, def);
  GD.Print($"最终伤害: {result.FinalDamage}");
  // 预期：100 × 1.5 × (1 - 0.3) = 105
  ```
- [ ] 运行游戏，控制台输出验证无误后删除测试代码

---

## Phase 4：技能系统与第一个技能（火球）

### Step 4.1 — 定义技能基类 SkillBase
- [ ] 创建文件 `scripts/combat/SkillBase.cs`
- [ ] 定义核心字段（MVP 不提前抽象，只放火球需要的字段）：
  ```csharp
  public abstract class SkillBase
  {
      public string SkillName = "Unnamed";
      public float ManaCost = 0;
      public float Cooldown = 0.5f;
      public float DamageMultiplier = 1.0f;
      public DamageType DamageType = DamageType.Physical;
      public float RemainingCooldown = 0f;

      public abstract void Execute(Vector2 from, Vector2 to,
          StatSheet casterStats, Node worldParent);
  }
  ```
- [ ] 确认编译通过

### Step 4.2 — 创建 SkillComponent 管理技能列表
- [ ] 创建文件 `scripts/combat/SkillComponent.cs`
- [ ] 继承 `Node`
- [ ] 添加字段：`public List<SkillBase> Skills = new()`
- [ ] 在 `_Process` 中递减每个技能的 `RemainingCooldown`
- [ ] 确认编译通过

### Step 4.3 — 实现 SkillComponent.UseSkill 方法
- [ ] 实现 `public void UseSkill(int index, Vector2 from, Vector2 to, StatSheet stats, Node worldParent)`：
  ```csharp
  if (index >= Skills.Count) return;
  var skill = Skills[index];
  if (skill.RemainingCooldown > 0) return;
  skill.Execute(from, to, stats, worldParent);
  skill.RemainingCooldown = skill.Cooldown;
  ```
- [ ] 确认编译通过

### Step 4.4 — 将 SkillComponent 挂到 Player 场景
- [ ] 打开 `player.tscn`
- [ ] 添加子节点 `Node`，命名为 `SkillComponent`
- [ ] 挂载 `SkillComponent.cs` 脚本
- [ ] 在 `PlayerController._Ready` 中缓存 `SkillComponent` 引用

### Step 4.5 — 创建火球投射物场景
- [ ] 在 `scenes/projectiles/` 下创建 `fireball.tscn`
- [ ] 根节点类型：`Area2D`，命名为 `Fireball`
- [ ] 添加子节点 `CollisionShape2D`，Shape 选 `CircleShape2D`，半径 8px
- [ ] 添加子节点 `Sprite2D`，用红色圆形占位
- [ ] 保存场景

### Step 4.6 — 创建投射物脚本
- [ ] 创建文件 `scripts/combat/Projectile.cs`
- [ ] 继承 `Area2D`
- [ ] 定义核心字段：
  ```csharp
  public Vector2 Direction;
  public float Speed = 600f;
  public float MaxDistance = 800f;
  public float DamageMultiplier = 1f;
  public DamageType DamageType;
  public StatSheet CasterStats;
  public float AOERadius = 80f;
  private Vector2 _startPos;
  private bool _hasExploded = false;
  ```
- [ ] `_Ready()` 中记录 `_startPos = GlobalPosition`
- [ ] 确认编译通过

### Step 4.7 — 实现 Projectile 飞行逻辑
- [ ] 在 `Projectile._PhysicsProcess` 中：
  ```csharp
  GlobalPosition += Direction * Speed * (float)delta;
  if (GlobalPosition.DistanceTo(_startPos) > MaxDistance) Explode();
  ```
- [ ] 确认编译通过

### Step 4.8 — 实现 Projectile 碰撞检测与爆炸
- [ ] 连接 `Area2D` 的 `body_entered` 信号，调用 `Explode()`
- [ ] 实现 `Explode()` 方法：
  ```csharp
  private void Explode() {
      if (_hasExploded) return;
      _hasExploded = true;
      var space = GetWorld2D().DirectSpaceState;
      var query = new PhysicsShapeQueryParameters2D();
      query.Shape = new CircleShape2D() { Radius = AOERadius };
      query.Transform = new Transform2D(0, GlobalPosition);
      var results = space.IntersectShape(query);
      foreach (var result in results) {
          var collider = result["collider"].As<Node>();
          // 伤害处理在后续步骤中完善
      }
      QueueFree();
  }
  ```
- [ ] 确认编译通过

### Step 4.9 — 实现 FireballSkill 具体技能类
- [ ] 创建文件 `scripts/combat/FireballSkill.cs`
- [ ] 继承 `SkillBase`，构造函数中设置：
  ```csharp
  SkillName = "火球";
  ManaCost = 8;
  Cooldown = 0.85f;
  DamageMultiplier = 1.0f;
  DamageType = DamageType.Fire;
  ```
- [ ] 重写 `Execute`：从 PackedScene 实例化 Fireball，设置 Direction、CasterStats、DamageType，添加到 worldParent
- [ ] 确认编译通过

### Step 4.10 — 在 Player 中绑定技能触发（鼠标左键）
- [ ] 在 `PlayerController` 中添加：
  ```csharp
  private StatSheet _stats;
  public override void _Ready() {
      _skills = GetNode<SkillComponent>("SkillComponent");
      _skills.Skills.Add(new FireballSkill());
      _stats = new StatSheet();
  }
  public override void _Input(InputEvent @event) {
      if (@event is InputEventMouseButton mb
          && mb.ButtonIndex == MouseButton.Left && mb.Pressed) {
          _skills.UseSkill(0, GlobalPosition, GetGlobalMousePosition(),
              _stats, GetParent());
      }
  }
  ```
- [ ] 运行测试：点击鼠标左键发射红色圆形投射物，向鼠标方向飞行

---

## Phase 5：投射物对象池

### Step 5.1 — 创建通用 ObjectPool 泛型类
- [ ] 创建文件 `scripts/core/ObjectPool.cs`
- [ ] 泛型类 `ObjectPool<T>` where T : Node：
  ```csharp
  public class ObjectPool<T> where T : Node
  {
      private PackedScene _scene;
      private Node _parent;
      private Stack<T> _available = new();
      private int _maxSize;
      // 构造函数：scene, parent, preAlloc, maxSize
      // CreateNew()：实例化、设 Visible=false、AddChild、Push
  }
  ```
- [ ] 确认编译通过

### Step 5.2 — 实现 ObjectPool.Get() 方法
- [ ] 若 `_available.Count == 0`，调用 `CreateNew()`
- [ ] Pop 一个实例，`Visible = true`，返回
- [ ] 确认编译通过

### Step 5.3 — 实现 ObjectPool.Return() 方法
- [ ] `obj.Visible = false`、`obj.SetProcess(false)`
- [ ] 若未超 `_maxSize`：Push 回 `_available`；否则 `QueueFree`
- [ ] 确认编译通过

### Step 5.4 — 创建 ProjectilePool 管理类
- [ ] 创建文件 `scripts/combat/ProjectilePool.cs`，继承 `Node`
- [ ] 维护 `Dictionary<string, ObjectPool<Projectile>>`（key = 技能 ID）
- [ ] 实现 `Initialize(string skillId, PackedScene scene, int preAlloc)`
- [ ] 实现 `GetProjectile(string skillId)` / `ReturnProjectile(string skillId, Projectile p)`
- [ ] 确认编译通过

### Step 5.5 — 修改 Projectile 支持对象池回收
- [ ] 添加 `OnReturnToPool()` 方法：重置 `_hasExploded`、重新启用 Process
- [ ] 爆炸后不 `QueueFree()`，改为通知 `ProjectilePool.Return()`
- [ ] 确认编译通过

### Step 5.6 — 修改 FireballSkill 使用对象池
- [ ] 不再 `scene.Instantiate`，改为从 `ProjectilePool.GetProjectile("fireball")` 获取
- [ ] ProjectilePool 实例挂在 BattleWorld 上，通过依赖注入获取引用
- [ ] 确认编译通过

### Step 5.7 — 配置 ProjectilePool 预分配并测试
- [ ] 在 BattleWorld 场景中添加 `ProjectilePool` 子节点
- [ ] 预分配火球 200 个
- [ ] 运行测试：连射火球，确认对象复用正常，无实例泄漏

---

## Phase 6：怪物系统

### Step 6.1 — 创建 MonsterData 配置类
- [ ] 创建文件 `scripts/data/MonsterData.cs`
- [ ] 定义可序列化 `class MonsterData`：
  ```csharp
  public class MonsterData {
      public string MonsterID;
      public string Name;
      public float BaseHP;
      public float BaseAttack;
      public float MoveSpeed;
      public int ExpReward;
      public string DropTableID;
      public string AIBehavior;
  }
  ```
- [ ] 确认编译通过

### Step 6.2 — 创建第一个怪物配置 JSON
- [ ] 创建文件 `data/monsters.json`
- [ ] 包含 1 条僵尸数据：
  ```json
  [{
      "MonsterID": "zombie",
      "Name": "僵尸",
      "BaseHP": 50,
      "BaseAttack": 8,
      "MoveSpeed": 80,
      "ExpReward": 20,
      "DropTableID": "drop_common",
      "AIBehavior": "MeleeChase"
  }]
  ```
- [ ] 确保 JSON 合法

### Step 6.3 — 创建 MonsterDatabase 加载器
- [ ] 创建文件 `scripts/data/MonsterDatabase.cs`
- [ ] 实现静态方法 `Load(string path)`：读取 JSON → 反序列化 → 返回 `Dictionary<string, MonsterData>`
- [ ] 确认编译通过

### Step 6.4 — 创建怪物场景
- [ ] 在 `scenes/monsters/` 下创建 `zombie.tscn`
- [ ] 根节点类型：`CharacterBody2D`，命名为 `Zombie`
- [ ] 添加子节点 `CollisionShape2D`（CircleShape2D，半径 14px）
- [ ] 添加子节点 `Sprite2D`（绿色矩形占位，32×32）
- [ ] 添加子节点 `NavigationAgent2D`

### Step 6.5 — 创建 HealthComponent
- [ ] 创建文件 `scripts/entities/HealthComponent.cs`，继承 `Node`
- [ ] 定义字段：`CurrentHP`、`MaxHP`、`CurrentShield`、`MaxShield`
- [ ] 定义事件：`OnDamaged(float amount, bool isCritical)`、`OnDied()`、`OnHealed(float amount)`
- [ ] 实现 `TakeDamage(DamageOutput dmg)` 方法：护盾先承伤，再扣血，HP≤0 触发 OnDied
- [ ] 确认编译通过

### Step 6.6 — 将 HealthComponent 挂到怪物
- [ ] 给 `zombie.tscn` 添加 `HealthComponent` 子节点并挂载脚本
- [ ] 在 `_Ready` 中设置 MaxHP=50, CurrentHP=50

### Step 6.7 — 创建 MonsterAI 基架
- [ ] 创建文件 `scripts/entities/MonsterAI.cs`，继承 `Node`
- [ ] 定义状态枚举 `enum AIState { Idle, Chase, Attack, Dead }`
- [ ] `_Ready` 中缓存 `CharacterBody2D`、`NavigationAgent2D`、`HealthComponent`、`Sprite2D`
- [ ] 确认编译通过

### Step 6.8 — 实现 MonsterAI 追逐逻辑
- [ ] `_PhysicsProcess` 中：
  ```csharp
  if (_state != AIState.Chase && _state != AIState.Attack) return;
  _navAgent.TargetPosition = _target.GlobalPosition;
  if (_navAgent.IsNavigationFinished()) {
      _state = AIState.Attack;
      return;
  }
  var nextPos = _navAgent.GetNextPathPosition();
  var dir = (nextPos - GlobalPosition).Normalized();
  _body.Velocity = dir * MoveSpeed;
  _body.MoveAndSlide();
  _sprite.FlipH = dir.X < 0;
  ```
- [ ] 确认编译通过

### Step 6.9 — 实现 MonsterAI 警戒与追击范围
- [ ] 添加 `DetectionRadius = 300f`、`AttackRange = 40f`
- [ ] `_Process` 中检测玩家距离，切换 Idle/Chase/Attack 状态
- [ ] 距离 > DetectionRadius × 1.5 时退回 Idle
- [ ] 确认编译通过

### Step 6.10 — 实现 MonsterAI 攻击时序
- [ ] 添加 `_attackCooldown = 1.5f` 和 `_attackTimer`
- [ ] 在 Attack 状态下，`_attackTimer` 归零时对 Target HealthComponent 造成伤害
- [ ] 死亡时 `_state = AIState.Dead`，停止一切逻辑
- [ ] 确认编译通过

### Step 6.11 — 创建简易 MonsterSpawner
- [ ] 创建文件 `scripts/entities/MonsterSpawner.cs`，继承 `Node`
- [ ] 实现 `SpawnMonster(string id, Vector2 pos)`：加载场景 → 实例化 → 设置位置 → 添加到世界
- [ ] 确认编译通过

### Step 6.12 — 在 BattleWorld 中测试怪物生成
- [ ] 在 `BattleWorld._Ready` 中生成 5 只僵尸
- [ ] 运行测试：僵尸 Idle，Player 靠近后追逐并攻击

---

## Phase 7：玩家受击、死亡与重生

### Step 7.1 — 给 Player 添加 HealthComponent
- [ ] 给 `player.tscn` 添加 `HealthComponent` 子节点，挂载脚本
- [ ] 设置 MaxHP=200, CurrentHP=200
- [ ] 在 PlayerController 中缓存 HealthComponent 引用

### Step 7.2 — 实现玩家受击
- [ ] 在 MonsterAI 攻击逻辑中，查询 Target HealthComponent 并调用 TakeDamage
- [ ] 先用简化版 DamageOutput（不经过 DamagePipeline，直接填 FinalDamage）
- [ ] 运行测试：站在僵尸旁边挨打，HP 下降

### Step 7.3 — 实现玩家死亡
- [ ] 订阅 `HealthComponent.OnDied`
- [ ] 死亡时：禁用移动输入、Sprite 变灰/变红、显示"按 R 重生"
- [ ] 确认编译通过

### Step 7.4 — 实现玩家重生
- [ ] 监听 R 键：HP 回满、位置回出生点、重新启用移动
- [ ] 运行测试：死亡→按 R→重生

---

## Phase 8：受击反馈与手感（Game Feel）

> ⚠️ 这 7 步是"好玩度"的关键，不要跳过。

### Step 8.1 — 怪物受击闪白
- [ ] 在 `HealthComponent.OnDamaged` 中设置 `Sprite2D.Modulate = Colors.White`
- [ ] 等待 0.05 秒后恢复原色（Tween 或 Timer）
- [ ] 运行测试：击中怪物，怪物闪白两帧

### Step 8.2 — 怪物受击击退
- [ ] 在 `HealthComponent.OnDamaged` 中获取攻击方向
- [ ] 用 Tween 将怪物沿受击方向后退 6px，耗时 0.1s
- [ ] 运行测试：击中怪物，怪物轻微后移

### Step 8.3 — 怪物死亡粒子效果
- [ ] 在 `HealthComponent.OnDied` 中生成 `GPUParticles2D` 爆炸效果
- [ ] 粒子生命周期 0.5 秒，然后 QueueFree
- [ ] 运行测试：击杀怪物，粒子爆发

### Step 8.4 — 屏幕震动
- [ ] 在 Camera2D 所在脚本添加 `Shake(float intensity)` 方法
- [ ] 用 Tween 做一个随机 offset 后再归零
- [ ] 暴击或大伤害时调用
- [ ] 运行测试：受击时屏幕轻微晃动

### Step 8.5 — 伤害数字弹出
- [ ] 创建 `DamageNumber` 场景：`Label` 节点
- [ ] 脚本实现：显示伤害数字 → 上浮 40px + 淡出（0.6s）→ QueueFree
- [ ] 暴击时字体更大、颜色更红
- [ ] 在 HealthComponent.OnDamaged 中实例化 DamageNumber
- [ ] 运行测试：击中怪物，数字跳出

### Step 8.6 — 攻击音效滞后补偿
- [ ] 从 freesound.org 或 bfxr.net 下载临时音效文件
- [ ] 存放于 `assets/sounds/`
- [ ] 在投射物生成的 0.05 秒后播放音效（有动画时应在第 3 帧播放）
- [ ] 运行测试：听到攻击音效

### Step 8.7 — 冻结帧（Hit Stop）
- [ ] 实现 `HitStop(float duration)` 方法：
  ```csharp
  Engine.TimeScale = 0.1f;
  await ToSignal(GetTree().CreateTimer(duration * 0.1f), "timeout");
  Engine.TimeScale = 1f;
  ```
- [ ] 暴击或高伤害时调用 `HitStop(0.04f)`
- [ ] 运行测试：暴击命中时画面瞬间"卡"一下

---

## Phase 9：装备系统

### Step 9.1 — 定义 EquipmentSlot 枚举
- [ ] 创建文件 `scripts/items/EquipmentSlot.cs`
- [ ] 定义 `enum EquipmentSlot { Weapon, Offhand, Head, Body, Hands, Feet, Belt, Amulet, Ring1, Ring2 }`
- [ ] 确认编译通过

### Step 9.2 — 定义 AffixType 枚举
- [ ] 在同文件或单独文件定义 `enum AffixType { Prefix, Suffix }`
- [ ] 确认编译通过

### Step 9.3 — 定义 AffixTemplate 数据结构
- [ ] 创建文件 `scripts/items/AffixTemplate.cs`
- [ ] 定义字段：AffixID、Name、Type、Slot、Tier、Weight、StatMod1_Key、StatMod1_Min、StatMod1_Max、TagGroup、ExclusiveGroup
- [ ] 确认编译通过

### Step 9.4 — 定义 AffixInstance 运行实例
- [ ] 定义 `class AffixInstance`：
  ```csharp
  public class AffixInstance {
      public AffixTemplate Template;
      public float RolledValue;
      public StatKey GetStatKey() => Enum.Parse<StatKey>(Template.StatMod1_Key);
  }
  ```
- [ ] 确认编译通过

### Step 9.5 — 定义 EquipmentBase 底材数据
- [ ] 定义 `class EquipmentBase`：BaseID、Name、Slot、ImplicitStatKey、ImplicitValue 等字段
- [ ] 确认编译通过

### Step 9.6 — 实现 EquipmentItem 类
- [ ] 创建文件 `scripts/items/EquipmentItem.cs`
- [ ] 定义字段：`EquipmentBase Base`、`List<AffixInstance> Affixes`、`string Rarity`
- [ ] 实现 `GetAllStatMods()` 返回底材固定属性 + 所有词缀属性
- [ ] 确认编译通过

---

## Phase 10：掉落与拾取系统

### Step 10.1 — 创建装备底材 JSON 配置
- [ ] 创建文件 `data/equipment_bases.json`
- [ ] 至少包含 3 种底材（武器/身体/戒指），每种含完整字段
- [ ] 确保 JSON 合法

### Step 10.2 — 创建词缀 JSON 配置（20 条）
- [ ] 创建文件 `data/affixes.json`
- [ ] 至少 20 条词缀，覆盖 Prefix/Suffix、不同 Tier、不同 Slot
- [ ] 确保 JSON 合法

### Step 10.3 — 创建 AffixDatabase 加载器
- [ ] 创建文件 `scripts/data/AffixDatabase.cs`
- [ ] `Load(string path)` 从 JSON 加载所有词缀模板
- [ ] 存为 `public static Dictionary<string, AffixTemplate> All`
- [ ] 确认编译通过

### Step 10.4 — 实现 LootGenerator（稀有度 Roll）
- [ ] 创建文件 `scripts/items/LootGenerator.cs`
- [ ] 实现 `RollRarity()`：Normal 70% / Magic 25% / Rare 5%
- [ ] 确认编译通过

### Step 10.5 — 实现 LootGenerator（词缀数量 Roll）
- [ ] 实现 `RollAffixCount(string rarity)`：
  - Normal: 0、Magic: 1-2、Rare: 3-6
- [ ] 确认编译通过

### Step 10.6 — 实现 LootGenerator（词缀按权重随机选择）
- [ ] 实现 `RollAffix(EquipmentSlot slot, AffixType type, List<string> excludeGroups)`
- [ ] 筛选符合 slot/type 且不在 excludeGroups 中的词缀，按 Weight 加权随机
- [ ] 返回选中的 AffixTemplate
- [ ] 确认编译通过

### Step 10.7 — 实现 LootGenerator（完整生成流程）
- [ ] 实现 `EquipmentItem Generate(Vector2 dropPos, int monsterLevel)`：
  1. 底材池筛选（DropLevel ≤ monsterLevel），随机选一个
  2. RollRarity → RollAffixCount → RollAffix（含 ExclusiveGroup 去重）
  3. 每个词缀在 Min-Max 之间 Roll 实际值
  4. 返回完整 EquipmentItem
- [ ] 确认编译通过

### Step 10.8 — 创建掉落物场景
- [ ] 创建 `scenes/items/dropped_item.tscn`
- [ ] 根节点：`Area2D`，命名为 `DroppedItem`
- [ ] 子节点：`Sprite2D`（按稀有度变色）+ `CollisionShape2D`
- [ ] 脚本持有 `public EquipmentItem Item` 字段
- [ ] 连接 `body_entered` 信号

### Step 10.9 — 实现掉落物生成（怪物死亡触发）
- [ ] 在 `HealthComponent.OnDied` 中调用 `LootGenerator.Generate()`
- [ ] 将 EquipmentItem 挂到 DroppedItem 场景实例
- [ ] 在怪物死亡位置生成掉落物
- [ ] 运行测试：击杀怪物，地上出现装备

### Step 10.10 — 实现玩家拾取
- [ ] 给 Player 添加拾取范围 `Area2D`
- [ ] 或在 PlayerController 中监听 `area_entered`
- [ ] 拾取时 `DroppedItem.QueueFree()`，EquipmentItem 加入玩家背包
- [ ] 运行测试：走过掉落物，装备被拾取

---

## Phase 11：装备 UI 与穿戴

### Step 11.1 — 创建装备面板场景
- [ ] 创建 `scenes/ui/equipment_panel.tscn`
- [ ] 根节点：`Control`，命名为 `EquipmentPanel`
- [ ] 添加武器槽位 `TextureRect` + 属性对比区域 `RichTextLabel`
- [ ] 默认隐藏，按 C 键切换显示
- [ ] 保存场景

### Step 11.2 — 实现装备槽显示
- [ ] 装备放入槽位：显示对应图标（临时用色块代替）
- [ ] 空槽位：灰色背景 + 装备类型文字（"武器"）
- [ ] 确认编译通过

### Step 11.3 — 实现属性对比面板
- [ ] 鼠标悬停装备槽：显示该装备提供的属性列表
- [ ] 拾取地上装备时：显示"当前装备 vs 地上装备"属性对比
- [ ] 增加用绿色标注，减少用红色
- [ ] 确认编译通过

### Step 11.4 — 实现装备穿戴逻辑
- [ ] 在 Player 添加 `InventoryComponent`（管理已装备物品）
- [ ] `Equip(EquipmentSlot slot, EquipmentItem item)`：
  1. 若槽位已有装备，先 Unequip
  2. 将 item 的所有 StatMod 通过 StatSheet.AddModifier 注册
  3. 更新 UI 槽位显示
- [ ] 确认编译通过

### Step 11.5 — 实现装备卸下逻辑
- [ ] `Unequip(EquipmentSlot slot)`：
  1. 调用 `StatSheet.RemoveAllFromSource(equippedItem)` 移除所有属性
  2. 将装备从槽位移除，更新 UI
- [ ] 确认编译通过

### Step 11.6 — 实现装备面板快捷键
- [ ] 在 `_Input` 中监听 C 键，切换 EquipmentPanel.Visible
- [ ] 运行测试：按 C 打开/关闭装备面板

---

## Phase 12：基础 HUD

### Step 12.1 — 创建 HUD 场景
- [ ] 创建 `scenes/ui/hud.tscn`
- [ ] 根节点：`CanvasLayer`，命名为 `HUD`
- [ ] 添加子节点：HP 球（`TextureProgressBar`）、MP 球、技能图标槽
- [ ] 保存场景

### Step 12.2 — 实现 HP/MP 球
- [ ] 订阅 HealthComponent 的 HP/MP 变化
- [ ] 更新 `TextureProgressBar.Value`
- [ ] HP 红色、MP 蓝色
- [ ] 运行测试：受击时 HP 球减少

### Step 12.3 — 实现技能图标与 CD 遮罩
- [ ] 在 HUD 底部添加技能槽 `TextureRect`
- [ ] 显示火球技能图标（临时红色方块）
- [ ] CD 中显示半透明遮罩旋转动画
- [ ] 运行测试：使用技能后图标进入 CD 动画

### Step 12.4 — 将 HUD 添加到 BattleWorld
- [ ] 在 `battle_world.tscn` 中添加 HUD 场景为子节点
- [ ] CanvasLayer 保证 UI 渲染在最上层
- [ ] 运行测试：HUD 始终可见

---

## Phase 13：压力测试与性能优化

### Step 13.1 — 设置性能预算常量
- [ ] 创建文件 `scripts/core/PerformanceBudget.cs`
- [ ] 定义常量：
  ```csharp
  public static class PerformanceBudget {
      public const int MaxMonsters = 500;
      public const int MaxProjectiles = 1500;
      public const int MaxParticles = 100;
      public const float NearAITickRate = 0.1f;
      public const float FarAITickRate = 0.5f;
  }
  ```
- [ ] 确认编译通过

### Step 13.2 — 实现 MonsterSpawner 数量上限
- [ ] 维护当前存活怪物计数
- [ ] SpawnMonster 前检查：若 ≥ MaxMonsters，跳过生成
- [ ] 怪物死亡时计数 -1
- [ ] 确认编译通过

### Step 13.3 — 实现 AI Tick 按距离分频
- [ ] MonsterAI 维护 `_tickTimer`
- [ ] 距离 < 500px → TickRate 0.1s；距离 ≥ 500px → TickRate 0.5s
- [ ] `_Process` 中只有 timer 归零才执行 AI 逻辑
- [ ] 确认编译通过

### Step 13.4 — 探索 MultiMeshInstance2D 批量渲染（可选）
- [ ] 在 BattleWorld 中创建 `MultiMeshInstance2D`
- [ ] 设置 Instance Count 为 500
- [ ] 代码中更新实例 Transform2D 而非操作独立 Node
- [ ] 注意：先保留独立 Node 方案，此步在压测不过时再做

### Step 13.5 — 执行同屏压力测试
- [ ] 修改 MonsterSpawner 生成 500 只僵尸
- [ ] 使用火球反复攻击
- [ ] 打开 Godot Debugger → Profiler，记录 FPS 和各耗时项
- [ ] 若 FPS < 60，分析瓶颈

### Step 13.6 — 根据 Profiler 数据做第一轮优化
- [ ] Script Process > 50%：检查循环、减少每帧运算
- [ ] Physics > 30%：减少碰撞体、用 IntersectShape 替代逐对碰撞
- [ ] Draw Calls > 500：启用 MultiMesh 合批
- [ ] 再次测试直到 500 怪物 + 500 投射物 60fps

---

## Phase 14：MVP 录制与 Bug 修复

### Step 14.1 — 录制 10 分钟试玩视频
- [ ] 使用 OBS / ScreenToGif / Peek 录制
- [ ] 覆盖完整流程：移动、杀怪、掉装、拾取、穿装、看面板、死亡重生
- [ ] 保存录像文件

### Step 14.2 — 自测记录 Bug 清单
- [ ] 看录像，逐帧标注所有异常
- [ ] 每个 Bug 单独建修复任务

### Step 14.3 — 修复已知 Bug
- [ ] 逐一修复，每修完一个重新验证
- [ ] 修复完成后重新录 10 分钟确认无新 Bug

### Step 14.4 — 提交 MVP 里程碑
- [ ] `git add -A && git commit -m "milestone: MVP complete"`
- [ ] 推送远程，备份项目

---

## Phase 15：Stage 2.1 — 装备系统完整化

### Step 15.1 — 创建全 10 个装备槽位 UI
- [ ] 在 EquipmentPanel 中添加全部 10 个槽位
- [ ] 每个槽位有文字标签标明部位名称
- [ ] 确认编译通过

### Step 15.2 — 为每类装备槽补充底材（30+ 种）
- [ ] 在 `equipment_bases.json` 中为每个槽位添加至少 3 种底材
- [ ] 涵盖护盾向/闪避向/伤害向不同倾向
- [ ] 确认 JSON 合法

### Step 15.3 — 扩展 InventoryComponent 支持全槽位
- [ ] `Dictionary<EquipmentSlot, EquipmentItem>` 管理 10 个槽位
- [ ] Equip/Unequip 支持所有槽位
- [ ] 确认编译通过

### Step 15.4 — 扩展词缀池至 200 条
- [ ] 在 `affixes.json` 中补充词缀至 200 条
- [ ] T1-T10 Weight 按指数递减（1000 → 400 → 160 → 64 → ...）
- [ ] 运行 DataValidator 校验

### Step 15.5 — 实现装备底材基础属性差异
- [ ] 不同底材的 BaseArmour/BaseShield/AtkSpeedBase 有显著差异
- [ ] EquipmentPanel 中显示底材名称和基础属性
- [ ] 确认生成装备时底材属性正确附加

### Step 15.6 — 实现重铸石通货
- [ ] 创建 `CurrencyOfScouring`：清空装备所有词缀，稀有度变为 Normal
- [ ] 在装备面板添加"使用重铸石"按钮
- [ ] 运行测试：使用重铸石清空词缀

### Step 15.7 — 实现点金石通货
- [ ] 创建 `CurrencyOfAlchemy`：Normal → Rare + 随机 4 条词缀
- [ ] 限制：只能对 Normal 装备使用
- [ ] 运行测试：Normal 装备变 Rare

### Step 15.8 — 实现装备排序/过滤 UI
- [ ] 排序按钮：按稀有度、按部位、按等级
- [ ] 过滤下拉框：只看武器/只看稀有
- [ ] 确认排序和过滤正确

---

## Phase 16：Stage 2.2 — 被动天赋星图

### Step 16.1 — 定义天赋节点数据结构
- [ ] 创建文件 `scripts/combat/PassiveNode.cs`
- [ ] 定义字段：NodeID、Name、Position、NeighborIDs、StatMods、IconPath、IsKeystone、IsStartingPoint
- [ ] 确认编译通过

### Step 16.2 — 定义天赋树状态管理
- [ ] 创建文件 `scripts/combat/PassiveTreeState.cs`
- [ ] 维护 `HashSet<string> AllocatedNodes` 和 `int AvailablePoints`
- [ ] 实现 `CanAllocate(string nodeID)`：检查邻居中是否有已分配节点
- [ ] 实现 `Allocate(string nodeID)`：标记已分配 + StatMods 加入 StatSheet
- [ ] 确认编译通过

### Step 16.3 — 实现 Unallocate 的连通性校验
- [ ] 撤销节点后对剩余已分配节点做 BFS，确保仍与某起始点连通
- [ ] 若有节点孤立，拒绝撤销
- [ ] 确认编译通过

### Step 16.4 — 设计并写入 6 个起始节点
- [ ] 设计 6 个职业倾向起始位置：力、敏、智、力敏、力智、敏智
- [ ] 每个起始点含 1 个起始节点 + 3 个周边小节点
- [ ] 写入 `data/passive_tree.json`

### Step 16.5 — 设计并写入 3 个 Keystone
- [ ] 设计：鲜血魔法、元素超载、混沌免疫
- [ ] 放在远离起始点的位置
- [ ] 写入 `data/passive_tree.json`

### Step 16.6 — 设计并写入 50+ 普通被动节点
- [ ] 分布在各路径上，属性小幅加成（+8 生命、+5% 火伤等）
- [ ] 确保多路线可达 Keystone
- [ ] 写入 `data/passive_tree.json`

### Step 16.7 — 实现天赋树渲染（自定义 Control）
- [ ] 创建 `scripts/ui/PassiveTreePanel.cs`，继承 `Control`
- [ ] `_Draw()` 中绘制连线（已分配绿色/未分配灰色）
- [ ] 绘制节点圆形（已分配金色/未分配灰色），Keystone 更大
- [ ] 确认渲染正确

### Step 16.8 — 实现天赋节点点击交互
- [ ] `_InputEvent` 检测点击位置是否有节点
- [ ] 未分配 + CanAllocate → 分配 + 消耗 1 点
- [ ] 已分配 → 撤销 + 返还 1 点
- [ ] 确认交互正确

### Step 16.9 — 实现天赋树拖拽与缩放
- [ ] 鼠标中键/右键拖拽平移视口
- [ ] 鼠标滚轮缩放（0.3x ~ 2.0x）
- [ ] 运行测试：自由浏览星图

### Step 16.10 — 实现返点通货
- [ ] 创建 `CurrencyOfRegret`：消耗 1 个，返还 1 个天赋点
- [ ] 右键已分配节点 → 消耗后悔石 → 撤销
- [ ] 确认返还点数正确

### Step 16.11 — 整合天赋树到 HUD
- [ ] HUD 添加"天赋树"按钮（或按 P 键）
- [ ] 点击弹出 PassiveTreePanel
- [ ] 运行测试：完整流程 — 升级得天赋点 → 点天赋 → 属性生效

---

## Phase 17：Stage 2.3 — 怪物词缀与 AI 扩展

### Step 17.1 — 创建怪物词缀模板（MonsterAffixTemplate）
- [ ] 创建文件 `scripts/items/MonsterAffixTemplate.cs`
- [ ] 定义字段：AffixID、Name、Weight、StatMods、SpecialBehavior、VisualEffect
- [ ] 确认编译通过

### Step 17.2 — 创建怪物词缀 JSON 配置（20+ 条）
- [ ] 创建文件 `data/monster_affixes.json`
- [ ] 数值型：迅捷、火焰强化、硬化等
- [ ] 行为型：死亡自爆、分身、多重投射、生命回复、反伤
- [ ] 确认 JSON 合法

### Step 17.3 — 实现怪物词缀生成器
- [ ] 稀有怪 2-4 条词缀，按 Weight 加权随机，互斥检查
- [ ] 生成后将 StatMods 加入怪物 StatSheet
- [ ] 确认编译通过

### Step 17.4 — 实现稀有/精英怪视觉识别
- [ ] 稀有怪：黄色名字 + 特殊边框
- [ ] 精英怪：橙色名字 + 光环效果
- [ ] 名字下方显示词缀列表
- [ ] 确认视觉区分明显

### Step 17.5 — 实现怪物命名生成
- [ ] 格式：`"{词缀1}·{词缀2}·{baseName}"`
- [ ] 例如："迅捷的·火焰强化·僵尸"
- [ ] 确认命名正确

### Step 17.6 — 创建 AI 行为抽象基类
- [ ] 创建文件 `scripts/entities/AIBehavior.cs`
- [ ] 定义 `Enter()`、`Execute(float delta)`、`Exit()` 抽象方法
- [ ] 确认编译通过

### Step 17.7 — 重构现有 AI 为 MeleeChaseBehavior
- [ ] 将 MonsterAI 逻辑迁移到 `MeleeChaseBehavior`，继承 AIBehavior
- [ ] 确认行为不变

### Step 17.8 — 实现远程风筝 AI（RangedKiteBehavior）
- [ ] 玩家靠近则后退、距离合适时射击、CD 控制
- [ ] 复用 ProjectilePool 生成投射物
- [ ] 确认编译通过

### Step 17.9 — 实现施法者 AI（CasterBehavior）
- [ ] 召唤小怪、放 AOE 地面技、给周围怪物上 Buff
- [ ] 添加施法条显示
- [ ] 确认编译通过

### Step 17.10 — 实现逃跑 AI（FleeOnHitBehavior）
- [ ] 受击后概率逃跑、跑一段后停下、几秒后恢复正常
- [ ] 确认编译通过

### Step 17.11 — 实现"死亡自爆"词缀行为
- [ ] OnDied 检查词缀 → 延迟 1s → AOE 伤害
- [ ] 显示红色闪烁预警圈
- [ ] 运行测试：击杀自爆怪，注意躲避

### Step 17.12 — 实现"分身"词缀行为
- [ ] HP < 50% 触发，生成 2 个 HP 减半的相同怪物
- [ ] 分身不带"分身"词缀防止无限分裂
- [ ] 运行测试：攻击分身怪，分裂

---

## Phase 18：Stage 2.4 — 关卡生成

### Step 18.1 — 设计房间模板数据结构
- [ ] 创建文件 `scripts/levels/RoomTemplate.cs`
- [ ] 定义字段：RoomID、Width、Height、TileData、DoorPositions、RoomType
- [ ] 确认编译通过

### Step 18.2 — 用 TileMap 编辑器制作 5 种房间模板
- [ ] 空旷房间、走廊型、柱子房间、Boss 房间、起始房间
- [ ] 每个模板导出可序列化数据
- [ ] 保存模板

### Step 18.3 — 实现房间随机组合算法
- [ ] 创建文件 `scripts/levels/MapGenerator.cs`
- [ ] 实现 `Generate(int floorNumber)`：
  1. 从入口房间开始
  2. 在未占用门洞上随机连接新房间
  3. 达目标数量后最后一间设为 Boss 房
  4. 确保无断路
- [ ] 确认编译通过

### Step 18.4 — 实现 TileMap 铺装
- [ ] 将房间布局转换为 `TileMapLayer.SetCell()` 调用
- [ ] 墙壁 Tile 在 TileSet 中配置碰撞
- [ ] 确认编译通过

### Step 18.5 — 创建 3 种区域 TileSet
- [ ] 室内（石砖+石墙）、室外（草地+树石）、洞穴（暗色+钟乳石）
- [ ] 每种 TileSet 在 TileMap 中正确渲染
- [ ] 确认无误

### Step 18.6 — 实现关卡目标与出口
- [ ] Boss 房间生成 Boss 怪物 + 特殊血条
- [ ] Boss 死亡后生成传送门（Area2D + 光效）
- [ ] 玩家站上传送门按 F → 加载下一层
- [ ] 运行测试：清层 → 杀 Boss → 进下一层

### Step 18.7 — 实现 Minimap
- [ ] 创建 Minimap 场景：SubViewportContainer + SubViewport + 俯视 Camera
- [ ] 已探索区域可见、未探索灰色
- [ ] 显示玩家白点、Boss 红点、出口绿点
- [ ] 小地图放 HUD 右上角

### Step 18.8 — 实现关卡难度递进
- [ ] 每层怪物等级 +1
- [ ] 每 5 层切换区域类型
- [ ] 每 10 层特殊事件层
- [ ] 确认难度曲线平滑

---

## Phase 19：数据管道与自动化工具

### Step 19.1 — 搭建 Google Sheets 数据表
- [ ] 创建 "PoE-Like Data" 文件
- [ ] 按表结构创建 5 个工作表：stats、equipment_bases、affixes、skills、monsters
- [ ] 每张表填入 MVP 阶段数据

### Step 19.2 — 写 Python 导出脚本
- [ ] 创建 `tools/export_sheets.py`
- [ ] 使用 `gspread` 连接 Google Sheets
- [ ] 遍历 5 个工作表，导出为 JSON 到 `data/` 目录
- [ ] 终端运行验证导出成功

### Step 19.3 — 验证导出 JSON 格式一致性
- [ ] 检查 JSON key 与 StatKey 枚举匹配
- [ ] 检查所有 ID 引用一致
- [ ] 手动修正不匹配项

### Step 19.4 — 写 DataValidator（引用完整性）
- [ ] 创建文件 `scripts/data/DataValidator.cs`
- [ ] 检查：装备底材 Slot 枚举存在、词缀 Slot 有对应底材、DropTableID 存在
- [ ] 返回错误列表

### Step 19.5 — 写 DataValidator（数值边界）
- [ ] 检查：StatMod_Min < Max、相邻 Tier 范围不重叠、Weight > 0
- [ ] 返回错误列表

### Step 19.6 — 写 DataValidator（逻辑一致性）
- [ ] 检查：同 ExclusiveGroup 内 Slot 一致、无循环引用
- [ ] 返回错误列表

### Step 19.7 — 游戏启动时自动运行 DataValidator
- [ ] DataManager.LoadAll() 末尾调用 Validate()
- [ ] 有错误则弹窗阻断启动
- [ ] 运行测试：故意写错 JSON 值，确认阻断生效

---

## Phase 20：最终检查与交付

### Step 20.1 — 完整走通全流程（3 遍）
- [ ] 创建角色/进游戏 → 杀怪 → 掉装 → 穿装 → 升天赋 → 通关一层 → 进下一层
- [ ] 走 3 遍，记录不顺滑之处

### Step 20.2 — 修复残留问题
- [ ] 修复流程测试中发现的所有问题
- [ ] 确认无崩溃、无软锁、无数据异常

### Step 20.3 — 写 README 开发文档
- [ ] 记录环境搭建步骤
- [ ] 记录数据导出流程
- [ ] 记录关键架构决策和原因

### Step 20.4 — 提交最终版本
- [ ] `git add -A && git commit -m "feat: complete Stage 2 - full ARPG skeleton"`
- [ ] 打 tag `v0.2.0`，推送到 GitHub

---

> **统计**：总计约 **120 步**，覆盖从空项目初始化到完整 ARPG 骨架的全部过程。
>
> **建议工作节奏**：每天完成 2-4 步（取决于复杂度），保持可持续的进度。
>
> **每完成一个 Phase 后**：提交一次 Git，记录实际耗时，用于后续估算更准确。
