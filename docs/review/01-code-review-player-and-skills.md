# 代码评审 #1：玩家控制与技能（`move.cs` / `frist_person_view.cs` / `skills.cs`）

> 评审对象：Unity 2022.3.62f1c1，Built-in 渲染管线，旧版 Input Manager
> 代码量：3 个脚本，368 行（`move.cs` 80 / `frist_person_view.cs` 38 / `skills.cs` 250）
> 示范重构：`Assets/RefactorDemo~/Scripts/`（12 个文件，已用 Unity 2022.3 程序集编译通过，0 错误 0 警告）

---

## 0. 一句话结论

**方向对、底子薄、债很集中。**

方向对：你选的两条路都是对的 —— 用 `rb.velocity` 覆盖而不是 `AddForce` 做移动（FPS 的正确解法），
以及把"冷却"从三个技能里抽出来做成一份数据。这两件事很多做了半年 FPS 的人都还没想明白。

债很集中：`skills.cs` 一个类背了 5 件事，`move.cs` 用 `public` 字段当接口给外面改，
还有 6 处"不报错但会让你怀疑手感"的逻辑问题。这些不是"再写点功能就好了"的问题，
它们会随着技能变多按乘法恶化。

| 维度 | 分数 | 说明 |
| --- | --- | --- |
| 能跑 | 8 / 10 | 三条技能链路都是通的，手感有可调参数 |
| 逻辑正确性 | 5 / 10 | 6 个 bug，其中 3 个是"浏览器上看不出来，换台机器就变"的时间/单位问题 |
| 结构 | 4 / 10 | 1 个类 5 个职责；状态靠 `public` 字段互改；技能之间没有裁判 |
| 命名与格式 | 4 / 10 | 类名小写、`frist`/`smock` 拼错、类名与文件名不一致、字段命名混用两种风格 |
| 可维护性 | 3 / 10 | 想改数值要读 250 行；Inspector 里填的东西被 `Awake` 覆盖 |
| 工程卫生 | 4 / 10 | 没有 git 仓库、场景没进 Build Settings、一个文件 GBK、残留默认场景 |

---

## 1. 先说你在做对的三件事

写评审容易只写坏的，但这三条真的值得记住，后面重构的时候别把它们丢了：

1. **移动用速度覆盖，不用力。** `move.cs:55-57` 每物理帧把水平速度直接写成目标速度，
   `y` 单独保留给重力和跳跃。这是 FPS 的教科书做法，也是"急停"手感的来源。
2. **视角的 yaw / pitch 分离。** 身体转 yaw、相机只转 pitch，这个结构对了。
   很多人第一版是让相机跟着俯仰把胶囊体也转过去，后面再想改就全乱了。
3. **冷却只存一个 `lastUseTime`，其余算出来。** 不存"剩余冷却"这种会飘的状态，
   就不会出现"计时器和 UI 对不上"。顺手还留了 `CooldownProgress` 给以后的 UI 用。

另外 `README.md` 和 `docs/blog/01-...md` 这两个文件的存在，比代码本身更能说明你在认真学 ——
大部分人的"学习项目"连一句说明都没有。

---

## 2. 必须先修的 6 个 bug

按"症状有多难查"排序，不是按代码行数。

### 2.1 鼠标灵敏度乘了 `Time.deltaTime`（`frist_person_view.cs:23-24`）

```csharp
float mouseX = Input.GetAxis("Mouse X") * mouseSensitivity * Time.deltaTime;
```

`Input.GetAxis("Mouse X")` 返回的**已经是**"这一帧鼠标移动了多少"（一个增量）。
再乘 `deltaTime` 等于把时间乘了两遍。

它不会让你"感觉变慢"，它会让**灵敏度随帧率变化**：60fps 乘 0.0167，144fps 乘 0.0069 ——
同一台机器换块高刷屏，转身速度只剩一半多。这类 bug 在你自己的机器上永远测不出来。

**判断规则（记住这一条就够）：鼠标增量不乘 `deltaTime`；键盘长按（移动、旋转、加速）才乘。**

### 2.2 冲刺速度乘了 `Time.deltaTime`（`skills.cs:99`）

```csharp
Vector3 targetVelocity = Dir * dashSpeed * Time.deltaTime;
move.rb.velocity = targetVelocity;
```

`velocity` 的单位是**米/秒**，`dashSpeed * deltaTime` 算出来是**米/帧**。
把一个"每帧位移"塞进"每秒速度"的位置，实际速度只有设计值的 1/60。

这就是为什么场景里 `dashSpeed` 被调到 **650**（正常应该是 25~40），
而手感还是"被轻轻推了一下"。**数值异常大，往往是在给单位错误打补丁。**

### 2.3 地面判定用了"每帧重置的布尔值"（`move.cs:59-67` + `69-79`）

```csharp
void FixedUpdate()
{
    // ...
    if (jumpRequested && isGrounded) { /* 跳 */ }
    isGrounded = false;            // 每帧先清
}
void OnCollisionStay(Collision col) { /* 之后才置 true */ }
```

`OnCollisionStay` 是在物理步**之后**回调的，所以 `FixedUpdate` 里读到的是上一轮的判定。
站在稍陡的坡上、或者贴墙时法线 `y` 掉到 0.5 以下，就会隔几帧"假摔"一次 ——
玩家感受到的是**站在地上偶尔跳不起来**。

改法不是"再置一次 true"，而是**记时间戳**：

```csharp
private float groundedUntil;
bool IsGrounded => Time.fixedTime < groundedUntil;

void OnCollisionStay(Collision c)
{
    for (int i = 0; i < c.contactCount; i++)
        if (c.GetContact(i).normal.y >= minGroundNormalY)
        { groundedUntil = Time.fixedTime + 0.1f; return; }
}
```

顺手白送一段**土狼时间**（离开平台边缘 0.1 秒内还能起跳）。玩家注意不到，去掉就会被骂。
另外 `foreach (ContactPoint c in col.contacts)` 每次访问都会分配数组，而这是每个物理步都调的回调 ——
换成 `contactCount` + `GetContact(i)` 就不分配了。

### 2.4 冲刺中途被打断，玩家永远动不了（`skills.cs:96-108`）

```csharp
move.canMove = false;
while (timer < dashDuration) { ... }
move.canMove = true;      // ← 这一行不一定执行得到
```

禁用物体、`StopCoroutine`、协程抛异常 —— 任何一种情况下 `canMove` 永远停在 `false`。
这是一个**会让玩家彻底玩不下去**的 bug，但它可能潜伏很久才被触发。

改法：`try / finally`。协程被 `StopCoroutine` 或物体被禁用时，`finally` 依然会执行。

### 2.5 `Awake` 覆盖了 Inspector 里填的技能名，而且名字和行为是错位的（`skills.cs:59-61`）

```csharp
skills[0].skillName = "dash";
skills[1].skillName = "smoke";     // ← 但 CastSkill2 是升空
skills[2].skillName = "suspend";   // ← 但 CastSkill3 是烟雾弹
```

场景里你实际填的是 `Dash / Floating / smock`（`1.unity`），运行后全被改成上面这三个 ——
于是 Inspector 里的数据变成了一份**假的真相**，而且第 1、2 槽的名字和行为对不上。

这件事的教训不是"名字写错了"，而是：**同一个事实不要存两份。**
技能名应该只属于技能自己（或干脆用类名）。

### 2.6 `SkillController` 和 `move` 之间没有校验，缺引用就直接 NRE

`skills.cs` 里 `move`、`fpv`、`smock` 三个 `public` 引用，任何一个没拖就
`NullReferenceException`，堆栈指在 `CastSkill1` 里面（`skills.cs:87` 起），第一次遇到根本不知道为什么。

改法：`Awake` 里检查一遍，缺什么直接说清楚缺什么、怎么补，然后把组件禁用掉。
**宁可第一帧红字报错，也不要运行时莫名其妙崩。**

---

## 3. 结构问题：三个脚本各自背了几件事

### 3.1 `skills.cs` 一个类装了 5 件事

| 职责 | 变更频率 | 现状 |
| --- | --- | --- |
| 冷却数据模型（`Skill` 内部类） | 几乎不变 | 混在里面 |
| 按键映射（`Update` 里三个 `if`） | 改绑定就变 | 混在里面 |
| 冲刺行为 | 调手感时天天变 | 混在里面 |
| 升空行为 | 调手感时天天变 | 混在里面 |
| 烟雾弹 + 飞行 + 碰撞 + 膨胀 | 调手感时天天变 | 混在里面 |
| 给 UI 用的 5 个查询方法 | 做 UI 时才变 | 混在里面 |

判断"该不该拆"的标准不是行数，而是**这堆东西会不会因为不同的原因、在不同的时候被改**。
上面这张表里，前两行和后面四行的变化节奏完全不同 —— 这就是必须拆的信号。

结果就是你改一次冲刺手感，要在一份 250 行的文件里翻到第 100 行；
而这份文件里还躺着另外两个技能的代码，随时可能被手滑改到。

### 3.2 用 `public` 字段当接口，导致"谁都能改我的状态"

`move.cs` 对外暴露了 `canMove`、`rb`、`moveDir` 三个 `public` 字段，
`skills.cs` 直接拿它们做三件事：

```csharp
move.canMove = false;                       // 抢控制权
move.rb.velocity = targetVelocity;          // 直接改刚体
Vector3 Dir = move.moveDir;                 // 读内部缓存
```

问题的本质不是"`public` 不好"，而是**没有任何地方规定谁能改什么**：

- `canMove` 谁都能设成 `false`，也谁都能忘了设回来（见 2.4）；
- 冲刺和升空**同时**在写 `rb.velocity`，最后谁写谁赢，这种行为不会报错，只会让手感"说不清"；
- `moveDir` 是 `move.cs` 在 `FixedUpdate` 里算的缓存，却被外部当成"玩家输入方向"来读，
  两个脚本的时间基准被悄悄绑在一起。

正确的做法不是把字段改成 `private` 了事，而是**给状态找一个唯一的主人**：
速度归 `PlayerMotor` 所有，别人要改就"申请"和"归还"，由主人负责合成最终结果。
示范代码里就是这么做的（`TryBeginPlanarControl` / `EndPlanarControl` 会校验 owner）。

### 3.3 技能之间没有裁判（你已经在日志里记下了这条）

现在按下 `E` 后立刻按 `Q`，两个协程同时在跑，都在写 `rb.velocity`。**这是结构问题，不是 bug** ——
缺的是一个"同一时间只允许一个技能"的规则（最低成本版本），再往上是状态机。

### 3.4 视角脚本挂错了物体

`frist_person_view` 挂在 **Main Camera** 上，但它干的是"转身体 + 转相机"两件事，
而相机是**根物体**（`1.unity` 里 `m_Father: {fileID: 0}`），不是玩家的子物体。
所以你只能用 `TransformPoint` 每帧把相机搬到玩家旁边。

标准做法是让相机成为玩家的子物体（`localPosition = (0, 4.5, 1)`），脚本只写 `localRotation`。
好处：不用算位置、玩家被缩放时不会错（`TransformPoint` 会把缩放乘进偏移里）、
以后加蹲下/受击位移也只需要改 `localPosition`。

### 3.5 玩家身上有两个碰撞体

`1.unity` 里 `jett` 同时挂着：

- `BoxCollider`：size `(1, 7.6, 1)`，center `(0, 1.5, 0)`
- `CapsuleCollider`：radius `1.8`，height `7.5`，center `(0, 1.5, 0)`

盒子被胶囊完全包住，只是多出一份接触点、多一次碰撞计算、多一种地面判定的噪声来源。
删掉盒子。

---

## 4. "该不该一个功能拆成一个小脚本" —— 我的答案

**不是。** 这是 Unity 初学者最容易走偏的地方，比"一个类写 800 行"更常见。

拆分的唯一目的是**降低每次修改要付出的代价**。所以判据只有三条：

1. **有没有独立的变化原因？** 这段代码会不会因为"另一件事"而需要改？
   （技能数值一天改十次，冷却规则一个月改一次 → 拆）
2. **它是否独占一份状态？** 这份状态是不是只有它该改？
   （速度只能有一个人写 → 拆出一个 owner）
3. **名字能不能一句话说清？** 说不清就说明混了。
   （`SkillController` 是"技能入口"还是"三个技能的实现"？说不清 → 拆）

反过来，下面这些是**为了文件数而拆**，只有成本没有收益：

- 把 `moveDir` 的计算单独拆成 `DirectionCalculator.cs`（它一辈子只被一个人用）；
- 把 `speed` 拆成 `SpeedSettings.cs`（配置就是配置，跟着用它的组件走）；
- 给每个技能再套一层 `DashConfig` + `DashHandler` + `DashRunner`。

**量化一下代价**：一次跨文件跳转约等于 3~5 秒的注意力切换。
一个 300 行的项目拆成 20 个文件，读一遍要看 20 次，其中 12 次是 10 行的空壳 —— 这是负收益。

**我的建议**：按"变化原因"拆，拆完每个文件应该能填满这张表的一行，
而且**每个文件都得能回答"如果我删掉它，哪件事会坏"**。答不出来的文件就是在制造噪音。

示范代码 12 个文件，就是按这条线切的（每个 30~213 行，最长的 `PlayerMotor.cs` 也只有 213 行）。
它比原来多 9 个文件，但每多一个文件你都说得清"它为什么必须单独存在"。

---

## 5. 命名、格式与规范

逐条对照，左边是你现在的问题，右边是标准做法。

| 现在 | 问题 | 应该 |
| --- | --- | --- |
| `class move` | 类名小写，和 Unity 的 `Move`/`Input` 风格冲突 | `PlayerMotor` / `PlayerMovement` |
| `class frist_person_view` | `frist` 是 `first` 的拼写错误；下划线命名 | `PlayerLook` / `FirstPersonCamera` |
| `skills.cs` 里装 `class SkillController` | **MonoBehaviour 的文件名必须和类名一致**，否则在 Add Component 里选不到 | 文件名 = 类名 |
| `smock`、`smockMag`、`smockMagSpeed` | `smoke` 的拼写错误，且带 `smock` 的五个字段全都要跟着错 | 全文替换成 `smoke*` |
| `MaxFlyTime` | 字段用 PascalCase（这是属性/方法的风格） | `maxFlyTime` |
| `public float speed` | 公共字段当接口用，外部能随便改 | `[SerializeField] private` + 只读属性 |
| `jumpforce` | 缺词界，应 `jumpForce` 或更准确 `jumpSpeed` | `jumpSpeed`（直接设速度，见下） |
| `lift` / `smockMag` | 看不出单位和含义 | `liftSpeed`（米/秒）、`expandMultiplier` |
| `using JetBrains.Annotations;` | 没用到的 using（`skills.cs` 里还留着一个 `using Unity.VisualScripting;`） | 删掉；开启 IDE 的"未使用 using"提示 |
| `Start()` 里 `GetComponent` | 别的脚本的 `Awake` 可能马上要用这个引用 | 挪到 `Awake` |
| 每个技能 `Debug.Log` 在 `while` 循环里 | 每帧一条日志，控制台刷屏 + 性能 | 只在状态切换（开始/结束）打一次，或用条件编译 |
| 魔数 `0.01f` / `1.2f` / `1.0f` | 不知道是什么 | 提到有名字的字段，或加注释说明单位 |
| 5 个 `GetXxx(index)` 各自写一遍下标检查 | 重复代码 | 抽一个 `IsValidIndex`，或让 `GetCooldown` 返回对象 |
| `skills.cs` 是 **GBK** 编码 | 换台机器/换编辑器中文注释全乱（我这边读出来就是乱码） | 全工程 UTF-8 |
| 没有命名空间 | 以后装个包也叫 `Player` 就撞名 | `namespace ValorantDemo.Player`（顶层按项目名） |

**格式上还差一样东西：`.editorconfig`。**
它能让"缩进、换行、编码、`using` 排序"从"每次记得"变成"编辑器自动"。
我在项目根目录放了一份，直接就能用。

---

## 6. 工程层面的四个坑（和代码无关，但会咬人）

1. **没有 git 仓库**（`git log` 报 `not a git repository`）。
   `README.md` 最后一行写着"对应代码提交 `feat: ...`"，但仓库不存在 —— 说明记录和现实脱节了。
   `Assets/RefactorDemo~` 这种"我想保留但不想被 Unity 编译"的写法，本身就是 git 时代的习惯。
   **这是最该先补的一件事**：现在这份 368 行，一旦改坏了没有任何退路。
2. **`ProjectSettings/EditorBuildSettings.asset` 里 `m_Scenes: []`** —— 场景没进 Build Settings，打包会报"没有场景"。
3. **`Assets/Scenes/SampleScene.unity` 是残留的默认场景**，里面还挂着旧的 `move` 组件但没有视角和技能。
   打开它 Play 会得到一个不能转视角的角色 —— 这就是"两个真相"的另一种形式。
4. **资产平铺在 `Assets/` 根目录**：三个脚本、两个材质、两个预制体、两个场景全在一起。
   超过 10 个资产之后，`Assets/Scripts`、`Assets/Prefabs`、`Assets/Scenes`、`Assets/Materials` 是必须的。

---

## 7. 示范代码在哪，怎么一步步迁过去

位置：**`Assets/RefactorDemo~/Scripts/`**

> 为什么带一个 `~`：Unity 会忽略以 `~` 结尾的文件夹（也忽略以 `.` 开头的）。
> 这样示范代码就躺在项目里、你能在资源管理器里直接对着看，但**不会参与编译**，
> 不会和现有的 `move` / `frist_person_view` / `SkillController` 打架，也不会让你现在能跑的场景坏掉。
> 想正式启用，把文件夹名改成 `RefactorDemo`（去掉 `~`）即可。

文件清单（12 个，每个都只有一个"变化原因"）：

```
Scripts/
├─ Player/
│  ├─ PlayerInput.cs             全工程唯一允许出现 Input. 的地方；把硬件输入翻译成意图
│  ├─ GroundDetector.cs          只回答"现在算不算贴地"（时间戳 + 土狼时间）
│  ├─ PlayerMotor.cs             唯一写 rigidbody.velocity 的地方；速度的所有者
│  ├─ PlayerLook.cs              yaw 归身体、pitch 归相机；灵敏度不乘 deltaTime
│  └─ PlayerAbilityController.cs 技能入口：收按键、判冷却、保证同一时间只有一个技能
└─ Abilities/
   ├─ AbilityBase.cs             技能基类（冷却 + 一个协程入口）
   ├─ PlayerAbilityContext.cs    把马达和视角打包传给技能，技能不自己找引用
   ├─ AbilityCooldown.cs         纯 C# 冷却计时器，不依赖 Time.time，可单测
   ├─ DashAbility.cs             E
   ├─ LiftAbility.cs             Q
   ├─ SmokeAbility.cs            C（只负责"扔出去"）
   └─ SmokeProjectile.cs         弹体自己的一生：飞、撞停、膨胀、消散
```

### 迁移路线（每一步都能单独跑起来，别一次改完）

**第 0 步 · 先搞版本控制**（10 分钟）
`git init`，确认 `.gitignore` 里有 `Library/ Temp/ obj/ Logs/ UserSettings/`，提交一次。

**第 1 步 · 只拆两个"独立性强"的**（半天）
把 `frist_person_view.cs` 换成 `PlayerLook.cs`，把贴地判定换成 `GroundDetector.cs`，
`move.cs` 里剩下的事情先不动（把 `isGrounded` 换成问 `GroundDetector` 就行）。
这一步能立刻验证两件事：视角不再随帧率变、斜坡上不再"假摔"。

**第 2 步 · 引入 `PlayerMotor`，把 `move.cs` 干掉**（半天）
`PlayerMotor` 接管速度，`move.cs` 的 `canMove`/`rb`/`moveDir` 三个 `public` 字段一起消失。
此时技能还没改，会编译不过 —— 所以第 2 步和第 3 步要连着做。

**第 3 步 · 拆技能**（一天）
`skills.cs` → `PlayerAbilityController` + 三个 `*Ability` + `SmokeProjectile`。
这一步顺便把 2.2 / 2.4 / 2.5 三个 bug 一起修掉（它们都在被拆掉的那段代码里）。

**第 4 步 · 收尾**（1 小时）
删掉 `Assets/Scenes/SampleScene.unity`、把 `Assets/1.unity` 加进 Build Settings、
`git commit` 一次，然后在 README 的"代码债清单"里划掉相应行。

### 场景里要一次性改的四处

1. `Main Camera` 拖到 `jett` 下面当子物体，`localPosition` 设成 `(0, 4.5, 1)`。
2. 删掉 `jett` 上的 `BoxCollider`（只留 `CapsuleCollider`）。
3. `Camera` 组件建议把 `Clipping Planes.Near` 稍微加大（0.3 左右），避免枪模型穿模。
4. `smoke bomb.prefab` 根物体上挂 `SmokeProjectile`，`cloud` 字段拖子物体 `Sphere`。

---

## 8. 验收清单

改完之后，逐条在 Play 里过一遍。每条都对应上面某个具体问题：

- [ ] 站在斜坡上连续按空格 20 次，每一次都能跳起来（对应 2.3）
- [ ] 把 Game 视图的分辨率/帧率调成 30 和 144，转 180° 需要划过的鼠标距离一致（对应 2.1）
- [ ] 冲刺距离固定：对着墙冲，不管从多远开始冲，位移都一样（对应 2.2）
- [ ] 冲刺中途按 Esc / 切换场景 / 启用禁用玩家，冲刺结束后仍能正常移动（对应 2.4）
- [ ] 连按 `E Q C`，同一时间只有一个技能生效（对应 3.3）
- [ ] 升空时按住 `W`，能边升边往前飞（不是被冻住）
- [ ] 扔出烟雾弹后立刻冲刺，烟雾照常膨胀、消散，不会跟着玩家走（对应 3.3 的变种）
- [ ] Console 里跑 1 分钟没有任何一条日志（对应"循环里打日志"）
- [ ] Inspector 里改技能冷却，运行时真的按新值生效（对应 2.5）

---

## 9. 下一步该做什么（按性价比排序）

1. **补 git + 上面 6 个 bug** —— 半天，收益最大。
2. **技能冷却 UI**：`AbilityCooldown.Progress01` 直接喂 `Image.fillAmount`，三个图标 + 数字倒计时。
   这一步会立刻暴露"UI 需要一份稳定的数据源"，而你现在的 `GetCooldownProgress` 已经准备好了。
3. **射击与命中判定**：`Raycast` + 弹孔 + 简单伤害。有了伤害，技能才有意义。
4. **一个会动不会还手的假人**：没有靶子，所有手感都验证不了。这比再做一个技能重要得多。
5. **数值外置成 ScriptableObject**：等你开始"为了调平衡反复改代码"的时候再做，
   现在做只是把数字从一个地方搬到另一个地方。

---

## 10. 参考

- [Unity Manual — Rigidbody Physics](https://docs.unity3d.com/Manual/RigidbodyPhysics.html)
- [Unity Scripting API — Input.GetAxis](https://docs.unity3d.com/ScriptReference/Input.GetAxis.html)
- [Unity Scripting API — Rigidbody.SweepTest](https://docs.unity3d.com/ScriptReference/Rigidbody.SweepTest.html)
- [Unity Manual — Coroutines](https://docs.unity3d.com/Manual/Coroutines.html)
- [Unity Manual — Special folder names](https://docs.unity3d.com/Manual/SpecialFolders.html)（`~` 结尾被忽略的依据）
- [C# Coding Conventions](https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions)
