# GIRP Reinforcement Learning Project Specification

> 本文件整理目前已討論完成的設計方向，並保留後續可以直接補充與決策的位置。  
> 原則：**先把規格討論清楚，再開始大量實作，避免後續重構與走回頭路。**

---

# 0. 專案目標

本專題目標是把既有 GIRP 遊戲改造成可由 Python 強化學習模型控制的 RL Environment。

不重新製作 GIRP，而是直接利用目前已有的：

- ActionScript
- Box2D Physics
- Player / Toehold / PlayState 遊戲邏輯
- 原始固定地圖
- 既有勝利 / 死亡 / Reset 流程

最終系統：

```text
GIRP / ActionScript / Box2D
        ↓
取得 Observation
        ↓
TCP Socket
        ↓
Python Gymnasium Environment
        ↓
PPO / MaskablePPO
        ↓
產生 Action
        ↓
TCP Socket
        ↓
GIRP 執行 Action
        ↓
Box2D Physics
        ↓
Reward / Done / New Observation
        ↓
回傳 Python
```

第一階段先使用原始固定 GIRP 地圖。

當模型可以穩定到達終點後，再加入：

- Small Randomization
- Seed-based Map Generation
- Unseen Seed Evaluation

---

# 1. 整體開發 Pipeline

```text
Phase 1
GIRP ↔ Python Socket
        ↓
Phase 2
Observation Extraction
        ↓
Phase 3
Action Execution
        ↓
Phase 4
Reward / Done / Reset
        ↓
Phase 5
Random Agent Test
        ↓
Phase 6
Gymnasium Environment
        ↓
Phase 7
PPO Baseline
        ↓
Phase 8
TensorBoard / CSV / Checkpoint
        ↓
Phase 9
Training Speed Optimization
        ↓
Phase 10
Multiple Environments
        ↓
Phase 11
Action Masking
        ↓
Phase 12
固定地圖穩定到頂
        ↓
Phase 13
Randomization
        ↓
Phase 14
Generalization Evaluation
```

---

# 2. GIRP ↔ Python 通訊

目前暫定：

```text
Python = Server
GIRP = Client
```

使用：

```text
localhost TCP Socket
127.0.0.1
```

第一版使用 JSON，方便 Debug。

之後如果效能不足，再考慮：

```text
Binary Float Array
```

## 2.1 需要支援的訊息

至少包含：

```text
HELLO
RESET
ACTION
STATE
REWARD
DONE
```

## 2.2 第一個 MVP

先不要直接接 PPO。

先完成：

```text
Reset
→ State
→ Random Action
→ New State
→ Reward
→ Death / Success
→ Reset
```

---

# 3. Observation Space

## 3.1 核心原則

Observation 必須是 **Physics-aware Observation**。

原因：

GIRP 並不是單純「選最近的岩點」。

Agent 必須有能力利用：

- Swing
- Momentum
- Linear Velocity
- Angular Velocity
- Joint Configuration

去抓原本靜止狀態下碰不到的岩點。

---

## 3.2 Player Physics State

目前建議至少包含：

### Torso / Chest

```text
position x
position y

linear velocity x
linear velocity y

angle
angular velocity
```

### Left Hand

```text
relative position x
relative position y

linear velocity x
linear velocity y
```

### Right Hand

```text
relative position x
relative position y

linear velocity x
linear velocity y
```

---

## 3.3 Joint State

候選：

```text
left shoulder angle
right shoulder angle

left elbow angle
right elbow angle

left shoulder angular velocity
right shoulder angular velocity

left elbow angular velocity
right elbow angular velocity
```

必要時之後再加入：

```text
hip
knee
neck
```

---

## 3.4 Grip State

至少包含：

```text
left hand grabbed?
right hand grabbed?

left hand current target
right hand current target
```

---

## 3.5 Nearby Holds

不建議只提供 World Position。

優先使用：

```text
hold.x - leftHand.x
hold.y - leftHand.y

hold.x - rightHand.x
hold.y - rightHand.y
```

每個 Hold 可考慮提供：

```text
relative x / y from left hand
relative x / y from right hand

available?
disabled?
currently occupied?
```

---

## 3.6 Observation Size

目前估計：

```text
Player State
約 20～30 values

Nearby Hold × N
約 5～6 values / hold
```

如果：

```text
N = 8
```

總 Observation 大約：

```text
60～80 dimensions
```

對 PPO MLP 是合理大小。

---

## 3.7 暫時不優先加入

Acceleration 不列為第一版必要 Observation。

原因：

```text
Position
Velocity
Angle
Angular Velocity
```

已能描述大部分 Dynamic State。

Acceleration 在碰撞與 Constraint 下容易產生 Spike / Noise。

---

# 4. Observation 待決定事項

> **後續討論時直接填寫這一區。**

## 4.1 Nearby Hold 數量

候選：

```text
N = 6
N = 8
N = 10
```

**最終決定：**

```text
TODO:
```

---

## 4.2 Nearby Hold 排序方式

候選：

```text
Distance from Hand
Height
Reachability
Combined Score
```

**最終決定：**

```text
TODO:
```

---

## 4.3 Padding

當畫面中不足 N 個 Hold 時：

```text
Zero Padding?
Special Invalid Flag?
```

**最終決定：**

```text
TODO:
```

---

## 4.4 Normalization

需要決定：

```text
Position Normalization
Velocity Normalization
Angle Normalization
Hold Relative Position Normalization
```

**最終決定：**

```text
TODO:
```

---

# 5. Action Space

目前已確定：

**不讓 PPO 直接學 A～Z。**

因為字母只是 GIRP 當下對岩點的控制 Mapping。

真正有意義的是：

```text
抓哪一個 Hold
```

因此 PPO 應該操作：

```text
Hold Index
```

GIRP 端可以直接：

```text
_player.addTarget(toehold)
```

不需要經過：

```text
選 Hold
→ 查 Letter
→ 模擬 Keyboard
→ Keyboard Handler
→ addTarget()
```

---

# 6. Action Space 下一個核心問題

Agent 不只需要決定：

```text
抓哪個 Hold
```

還要決定：

```text
哪隻手抓
什麼時候抓
什麼時候放手
什麼時候等待
```

這對利用 Momentum 非常重要。

---

## 6.1 候選方案 A：單一 Discrete Action

例如 Nearby Hold 數量為 8：

```text
0  = No-op

1  = Left Grab Hold 0
2  = Left Grab Hold 1
...
8  = Left Grab Hold 7

9  = Right Grab Hold 0
10 = Right Grab Hold 1
...
16 = Right Grab Hold 7

17 = Left Release
18 = Right Release
```

優點：

```text
容易 Debug
容易 Mask
容易接 MaskablePPO
```

目前比較傾向此方案。

---

## 6.2 候選方案 B：MultiDiscrete

例如：

```text
hand:
0 = left
1 = right

operation:
0 = no-op
1 = grab
2 = release

target:
0 ~ N-1
```

優點：

```text
結構比較直觀
```

缺點：

```text
Action Combination 可能產生很多 Invalid Action
Masking 較複雜
```

---

# 7. Action Space 待決定事項

> **下一輪討論建議從這裡開始。**

## 7.1 Discrete vs MultiDiscrete

**目前傾向：**

```text
Discrete
```

**最終決定：**

```text
TODO:
```

---

## 7.2 No-op 是否必要

No-op 可以讓 Agent：

```text
等待擺盪
等待速度增加
等待最佳抓取 Timing
```

**目前傾向：**

```text
需要
```

**最終決定：**

```text
TODO:
```

---

## 7.3 Release 是否分左右手

候選：

```text
Left Release
Right Release
```

**最終決定：**

```text
TODO:
```

---

## 7.4 同一時間能否同時操作兩手

候選：

```text
每個 RL Step 只允許一個 Hand Action
```

或：

```text
左右手可以同時 Action
```

**最終決定：**

```text
TODO:
```

---

# 8. Action Masking

預計加入 Action Masking。

例：

```text
Hold 0 可抓
Hold 1 太遠
Hold 2 disabled
Hold 3 可抓
```

Agent 不需要浪費 Exploration 在：

```text
Hold 1
Hold 2
```

可能採用：

```text
sb3-contrib MaskablePPO
```

需要 Mask 的情況：

```text
Hold 太遠
Hold disabled
Hold 不存在
該手已經抓住相同 Hold
其他不合法狀態
```

## 8.1 Action Mask 待決定

```text
TODO:
- Reachability Threshold
- 已抓住岩點是否允許重新選
- Release 何時合法
- No-op 是否永遠合法
```

---

# 9. Reward Design

第一版 Reward 應保持簡單。

核心：

```text
reward = new_height - old_height
```

另外可考慮：

```text
成功抓到新的較高 Hold
→ Small Positive Reward

Death
→ Large Negative Reward

Reach Top
→ Large Positive Reward
```

需要避免：

```text
反覆抓同一 Hold 刷 Reward
```

---

# 10. Reward 待決定事項

## 10.1 Height Reward

```text
TODO:
```

## 10.2 Grab Reward

```text
TODO:
```

## 10.3 Death Penalty

```text
TODO:
```

## 10.4 Finish Reward

```text
TODO:
```

## 10.5 Time Penalty

```text
TODO:
```

## 10.6 Reward Hacking 防止策略

```text
TODO:
```

---

# 11. Episode / Done

目前 Done 條件：

```text
掉入水中
成功到頂
超過最大時間
```

未來考慮：

```text
長時間高度沒有進展
→ Early Termination
```

## 11.1 Episode 待決定

```text
Maximum Episode Duration:
TODO:

No-progress Timeout:
TODO:
```

---

# 12. Reset

需要確保：

```text
Player
Box2D World
Grip State
Target State
Reward State
Timer
Previous Height
```

都能正確 Reset。

Reset 方案仍需確認：

```text
重新建立 PlayState

vs

只 Reset Player / World
```

**最終方案：**

```text
TODO:
```

---

# 13. Frame Skip / Action Repeat

不打算讓 PPO 每個 Physics Frame 都做一次 Decision。

候選：

```text
frame_skip = 2
frame_skip = 4
frame_skip = 5
```

流程：

```text
Action
→ Physics Step × N
→ New Observation
→ Next Action
```

優點：

```text
減少 Socket Overhead
減少 NN Inference
減少 Python Overhead
```

缺點：

```text
過大可能失去抓取 Timing
```

**最終值需要實驗決定。**

---

# 14. Training Mode / Demo Mode

## 14.1 Training Mode

```text
Rendering OFF
Sound OFF
UI OFF
Physics ON
Socket ON
Faster-than-real-time
```

## 14.2 Demo Mode

```text
Rendering ON
Normal Speed
Loaded Trained Model
```

---

# 15. Multiple Environments

原則：

```text
1 GIRP Instance = 1 RL Environment
```

不是：

```text
一個 GIRP 被多個 Thread 同時控制
```

而是：

```text
PPO
├─ Env 1 ↔ GIRP #1
├─ Env 2 ↔ GIRP #2
├─ Env 3 ↔ GIRP #3
└─ Env 4 ↔ GIRP #4
```

可能使用：

```text
SubprocVecEnv
multiprocessing
```

預計 Benchmark：

```text
1 Env
2 Env
4 Env
8 Env
16 Env
```

記錄：

```text
Steps / second
Episodes / hour
CPU usage
RAM usage
Wall-clock Training Time
```

---

# 16. Training Logging

工具：

```text
TensorBoard
CSV
```

至少記錄：

```text
Episode Reward
Episode Length
Average Height
Max Height
Success Rate
Fall Rate
Average Grab Count

Policy Loss
Value Loss
Entropy
KL Divergence
Learning Rate

Steps/sec
Episodes/hour
```

重點：

不能只看 Reward。

至少同時觀察：

```text
Reward
Average Height
Success Rate
```

---

# 17. Checkpoint / Rollback

不能只存 Final Model。

例如：

```text
checkpoints/
├── 050k.zip
├── 100k.zip
├── 150k.zip
├── 200k.zip
└── ...
```

另外保存：

```text
best_model.zip
```

Best Model 建議依據：

```text
Evaluation Success Rate
Average Height
```

而不是只有 Training Reward。

---

# 18. Experiment Versioning

每一次 Run：

```text
runs/
└── run_001/
    ├── config.yaml
    ├── metrics.csv
    ├── tensorboard/
    ├── checkpoints/
    ├── best_model.zip
    └── notes.md
```

`config.yaml` 至少包含：

```text
algorithm
learning_rate
gamma

observation_version
action_version
reward_version

frame_skip
number_of_envs

map_seed
randomization_settings
```

---

# 19. Git Versioning

```text
Git
→ Code Version

Checkpoint
→ Model Version

Config
→ Experiment Settings

TensorBoard / CSV
→ Training Result
```

建議 Milestone：

```text
v0.1 Socket Works
v0.2 Observation Works
v0.3 Action Works
v0.4 Reset / Reward / Done Works
v0.5 Random Agent Works
v0.6 PPO Baseline
v0.7 Training Optimization
v0.8 Multi-Env
v0.9 Randomization
```

---

# 20. Fixed Map → Randomization

目前已決定：

**不在第一階段做 Random Map。**

先讓 PPO 在原始固定 GIRP 地圖：

```text
穩定到頂
```

再加入 Randomization。

---

# 21. 「固定地圖成功」定義

候選標準：

```text
每次 Evaluation = 20 Episodes

連續 3 次 Evaluation

Success Rate ≥ 90%
```

才進入 Randomization。

**最終標準：**

```text
TODO:
```

---

# 22. Randomization Roadmap

## Stage 1

```text
Original Fixed Map
```

## Stage 2

```text
Letter Seed
Initial Angle
Initial Velocity
```

## Stage 3

```text
Small Hold Position Jitter
```

例如：

```text
x += random(-5, +5)
y += random(-3, +3)
```

## Stage 4

```text
Seed-based Procedural Map
```

## Stage 5

```text
Unseen Seed Evaluation
```

---

# 23. Procedural Map 原則

不能只是：

```text
x = random
y = random
```

需要 Constrained Generation。

保證：

```text
岩點不超出地圖範圍
相鄰岩點距離合理
至少有一條可解路徑
難度不能完全失控
```

真正困難的不是 Random 本身，而是：

```text
可解性
+
合理難度
```

---

# 24. Generalization Evaluation

未來：

```text
Training Seeds
例如 1 ~ 100
```

```text
Evaluation Seeds
例如 101 ~ 120
```

Evaluation Seeds 不給 PPO 訓練。

用來測試：

```text
Agent 是真的學會攀爬

還是

只記住固定路線
```

---

# 25. 預計 Ablation Study

## Physics-aware Observation

比較：

```text
Model A
Position Only
```

vs

```text
Model B
Position
+ Linear Velocity
+ Angular Velocity
```

比較指標：

```text
Training Speed
Average Height
Success Rate
Long-distance Grab Success
```

研究問題：

> 動態物理資訊是否能提升 RL Agent 利用慣性與擺盪完成遠距離抓取的能力？

---

# 26. Training Speed Optimization Roadmap

優先順序：

```text
1. Rendering OFF
2. Sound / UI OFF
3. Faster-than-real-time Physics
4. Frame Skip
5. Multiple Environments
6. Action Masking
7. Early Termination
8. Curriculum Learning
9. Observation Simplification
10. Binary Socket Protocol
11. Headless Physics Backend
```

最後一項：

```text
Headless Physics Simulator
```

只有真的需要更高 Throughput 時才考慮。

---

# 27. Curriculum Learning（Optional）

如果完整 GIRP 太難：

```text
Stage 1
短距離 / 低高度

Stage 2
中距離

Stage 3
完整地圖
```

或：

```text
Easy Hold Distance
→ Normal Hold Distance
→ Original GIRP
```

目前不是第一優先。

---

# 28. 尚未完成的設計問題

後續按照以下順序，一題一題討論：

- [ ] Action Space 最終定義
- [ ] Action Mask 規則
- [ ] Reward Function
- [ ] Done / Early Termination
- [ ] Reset Strategy
- [ ] Observation Hold Sorting / Padding / Normalization
- [ ] Socket Protocol Message Format
- [ ] Frame Skip
- [ ] Training Mode Implementation
- [ ] Multi-Environment Architecture
- [ ] Checkpoint / Evaluation Frequency
- [ ] Fixed-map Success Threshold
- [ ] Randomization Parameters
- [ ] Final Experiments / Ablation Studies

---

# 29. 下一個討論問題

## Question 1 — Action Space

要回答：

```text
1. Discrete 還是 MultiDiscrete？
2. 左右手如何表示？
3. Grab 如何表示？
4. Release 如何表示？
5. No-op 是否必要？
6. 一個 Step 能否同時操作左右手？
7. Action Mask 如何配合？
8. 如何讓 Agent 可以等待 Swing / Momentum Timing？
```

### 討論結果

```text
TODO:
```

### 最終規格

```text
TODO:
```

---

# 30. 自由補充區

> 這裡就是你們之後可以直接新增想法、問題、風險、實驗的地方。

## Idea 1

```text
TODO:
```

## Idea 2

```text
TODO:
```

## Idea 3

```text
TODO:
```

## New Requirement

```text
TODO:
```

## Engineering Risk

```text
TODO:
```

## Experiment Idea

```text
TODO:
```

---

# 31. Open Questions Log

| ID | Question | Status | Decision |
|---|---|---|---|
| Q01 | Action Space | Open | TODO |
| Q02 | Action Mask | Open | TODO |
| Q03 | Reward Design | Open | TODO |
| Q04 | Episode Done | Open | TODO |
| Q05 | Reset | Open | TODO |
| Q06 | Observation Details | Open | TODO |
| Q07 | Socket Protocol | Open | TODO |
| Q08 | Frame Skip | Open | TODO |
| Q09 | Multi-Environment | Open | TODO |
| Q10 | Randomization | Later | Fixed Map First |

---

# 32. Decision Log

> 每次討論完成一個重要決策，就在這裡新增一筆，避免之後忘記為什麼當初這樣設計。

| Date | Decision | Reason |
|---|---|---|
| TODO | Physics-aware Observation | Agent 需要利用 Momentum / Swing |
| TODO | Fixed Map First | 先確認 RL Pipeline 正確 |
| TODO | Randomization After Stable Success | 避免過早增加環境複雜度 |
| TODO | Hold-based Actions Instead of A-Z | Letter Mapping 本身沒有學習價值 |
| TODO | Socket Bridge | GIRP 與 Python 解耦 |

---

# 33. Meeting / Discussion Notes

> 每次組員討論完，可以直接貼在這裡，再整理到正式規格。

## Discussion 001

**Date:**

```text
TODO:
```

**Topic:**

```text
TODO:
```

**Ideas:**

```text
TODO:
```

**Decision:**

```text
TODO:
```

**Need More Research:**

```text
TODO:
```

---

# 34. Change Log

| Version | Date | Change |
|---|---|---|
| v0.1 | TODO | Initial specification based on current discussion |
