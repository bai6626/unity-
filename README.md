# Valorant 复刻练习

用 Unity 复刻《Valorant》的核心玩法：第一人称射击 + 技能系统。

> 目标不是做一个更好的游戏，而是搞清楚"一个能上场的 FPS 角色到底由哪些零件组成"。

## 当前进度

- [x] 第一人称移动（WASD + 跳跃，Rigidbody 驱动，无惯性急停手感）
- [x] 第一人称视角（身体转身 / 相机俯仰分离，俯仰限位 ±80°）
- [x] 技能框架（冷却数据、释放入口、协程驱动）
- [x] 三个技能：`E` 冲刺、`Q` 升空、`C` 烟雾弹
- [ ] 技能冷却 UI
- [ ] 射击与命中判定
- [ ] 假人 Bot
- [ ] 技能数值 ScriptableObject 化

## 操作

| 按键 | 行为 |
| --- | --- |
| `W` `A` `S` `D` | 移动 |
| `Space` | 跳跃 |
| 鼠标 | 视角 |
| `E` | 冲刺 |
| `Q` | 升空 |
| `C` | 烟雾弹 |

## 开发日志

| # | 标题 | 内容 |
| --- | --- | --- |
| 01 | [能跑、能跳、能放技能的第一人称角色](docs/blog/01-player-controller-and-skills.md) | Rigidbody 移动、视角分离、技能冷却框架，以及五个真实的坑 |

## 代码评审与重构示范

| # | 标题 | 内容 |
| --- | --- | --- |
| 01 | [代码评审：玩家控制与技能](docs/review/01-code-review-player-and-skills.md) | 6 个必须先修的 bug、职责拆分的三条判据、12 个文件的重构示范 |
| — | [重构示范代码](Assets/RefactorDemo~/README.md) | `Assets/RefactorDemo~/`，Unity 会忽略带 `~` 的文件夹，所以不影响现有场景 |

## 环境

- Unity **2022.3.62f1c1**
- 渲染管线：Built-in
- 输入：旧版 Input Manager（`Input.GetAxis` / `GetKeyDown`）

## 运行

1. 用 Unity 2022.3.62f1c1（或同小版本）打开本目录
2. 打开 `Assets/1.unity`
3. Play

## 目录

```
Assets/
├─ 1.unity                 场景
├─ move.cs                 水平移动 / 跳跃 / 地面判定
├─ frist_person_view.cs    鼠标视角
├─ skills.cs               技能与冷却
├─ smoke bomb.prefab       烟雾弹弹体
docs/blog/                 开发日志
```
