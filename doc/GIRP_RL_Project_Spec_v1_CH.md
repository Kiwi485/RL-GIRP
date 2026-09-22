# GIRP 強化學習專題規格書 v1.0

## 1. 專案概述

本專案目標為使用強化學習（Reinforcement Learning, RL）訓練 AI 控制 GIRP
攀爬遊戲。

AI 需要學習：

-   選擇攀爬目標
-   控制抓取與放開
-   利用遊戲物理特性移動
-   利用擺盪慣性（Swing Momentum）
-   學習收縮與放鬆的時機
-   自動完成攀爬路線

主要研究問題：

> 強化學習 Agent 是否能透過與 GIRP
> 物理環境互動，學習類似人類的攀爬策略？

------------------------------------------------------------------------

# 2. 系統架構

    Python PPO 強化學習 Agent
              |
              | TCP Socket + JSON
              |
    GIRP 遊戲環境
              |
              |
    Physics 物理模擬

## Python RL Agent 負責：

-   PPO 訓練
-   Action 決策
-   模型儲存
-   模型評估
-   Logging

## GIRP Environment 負責：

-   物理模擬
-   遊戲狀態取得
-   執行 Action
-   Reward 計算
-   Episode 管理

------------------------------------------------------------------------

# 3. Environment Reset 設計

## 決定

第一版使用：

    完整 PlayState 重建 Reset

流程：

    Episode 結束

    ↓

    刪除目前遊戲狀態

    ↓

    建立新的 PlayState

    ↓

    清除 RL 狀態

    ↓

    開始新的 Episode

原因：

第一階段優先確保：

-   每個 Episode 狀態獨立
-   避免舊 Joint、Velocity、Target 殘留
-   降低 Debug 難度

未來：

若 Reset 成為效能瓶頸，再考慮 Fast Reset。

------------------------------------------------------------------------

# 4. Observation State 設計

## Version 1

Observation：

    上半身 Physics State
    +
    8 個 Nearby Holds

------------------------------------------------------------------------

## Body State

包含：

-   位置 Position
-   速度 Velocity
-   角度 Angle
-   角速度 Angular Velocity

------------------------------------------------------------------------

## Hand State

左右手分開記錄：

狀態：

    FREE
    ASSIGNED
    LOCKED

包含：

-   目前目標 Hold
-   位置
-   速度

注意：

Agent 不直接控制左右手。

由 GIRP 原本邏輯自動決定使用哪隻手。

------------------------------------------------------------------------

## Arm State

記錄：

-   Joint Angle
-   Joint Velocity
-   Contract State

Contract：

    0 = Relaxed
    1 = Contracted

------------------------------------------------------------------------

# 5. Observation 未來版本

## Version 2

如果 Version 1 表現不足：

加入：

-   Hip
-   Knee
-   Feet
-   下半身速度
-   下半身角度

目的：

驗證完整身體資訊是否改善攀爬能力。

------------------------------------------------------------------------

# 6. Observation Normalization

## 決定

所有連續數值 Normalize 到：

    [-1,1]

包含：

-   Position
-   Velocity
-   Angular Velocity
-   Joint Angle
-   Relative Hold Position

Binary 狀態保持：

    0 / 1

包含：

-   Hand State
-   Contract State
-   Valid Hold Flag

原因：

不同 Physics 數值尺度差異大。

Normalize 可以提升 PPO 訓練穩定性。

------------------------------------------------------------------------

# 7. Hold 系統

使用 GIRP 原始 Hold。

不自行生成新的岩點。

RL 使用：

    Nearby Hold Slot

而不是直接使用原始 Hold ID。

------------------------------------------------------------------------

# 8. Nearby Hold 選擇策略

總數：

    8 個 Hold Slots

使用 Hybrid Strategy：

    Slot 0-3:
    最近的候選 Hold

    Slot 4-7:
    未來可能有價值的 Hold

Future Hold 評估概念：

    高度收益 - 距離成本

所有 Hold 會固定排序，避免不同 Episode 中 Slot 意義改變。

------------------------------------------------------------------------

# 9. Action Space 設計

## Version 1

使用：

    Discrete(14)

Action：

    0:
    No-op


    1-8:
    Target Nearby Hold Slot 0-7


    9:
    Release Left Hand


    10:
    Release Right Hand


    11:
    Contract


    12:
    Relax


    13:
    Cancel Target

------------------------------------------------------------------------

# 10. 左右手控制

Agent 不決定：

    Left Hand
    or
    Right Hand

Agent 只決定：

    Target Hold

GIRP 自動：

    判斷適合的手
    ↓

    執行抓取

原因：

保留原始遊戲機制，降低 Action 複雜度。

------------------------------------------------------------------------

# 11. Persistent Target 設計

Target 不會立即消失。

流程：

    Agent 選擇 Hold

    ↓

    Target 保持

    ↓

    Swing / Momentum

    ↓

    抓取成功

Target 持續直到：

1.  成功抓到
2.  被新的 Target 取代
3.  Agent 使用 Cancel Target
4.  Episode 結束

目的：

讓 AI 學習：

    Target
    ↓
    Swing
    ↓
    Momentum
    ↓
    Grab

------------------------------------------------------------------------

# 12. Contract / Relax 設計

使用 Persistent State。

例如：

    Contract

    No-op

    No-op

    Relax

代表：

保持收縮狀態一段時間，再放鬆。

目的：

讓 AI 自己學習：

-   收縮 timing
-   擺盪控制
-   慣性利用

------------------------------------------------------------------------

# 13. Action Mask

使用：

    Conservative Action Masking

只移除明確無效 Action：

例如：

-   不存在 Hold
-   Disabled Hold
-   Release 空手
-   沒有 Target 時 Cancel

不禁止：

-   遠距離 Hold
-   難以到達的 Hold
-   需要 Momentum 才能抓的 Hold

原因：

避免限制 AI 學習 Swing。

------------------------------------------------------------------------

# 14. Reward 設計

## Step Reward

第一版：

    reward =
    目前高度 - 前一時間高度

鼓勵：

    往上攀爬

------------------------------------------------------------------------

不加入：

-   Grab Reward
-   Contract Reward
-   Swing Reward

原因：

避免 Reward Hack。

------------------------------------------------------------------------

# 15. 成功 Reward

主要目標：

    登頂

成功時：

    Finish Bonus
    +
    Completion Time Bonus

越快完成成功攀爬，獲得越高獎勵。

------------------------------------------------------------------------

# 16. Death 與 Timeout

## Death

死亡：

-   Episode 結束
-   不額外扣分

原因：

死亡本身已造成：

-   Episode 結束
-   失去未來 Reward

------------------------------------------------------------------------

## Timeout

Episode 結束條件：

1.  成功登頂
2.  死亡
3.  No Progress Timeout
4.  Hard Time Limit

------------------------------------------------------------------------

# 17. Training 架構

## 初版

    1 個 GIRP Environment

    +

    1 個 PPO Agent

目的：

先確認整個 Pipeline 正確。

------------------------------------------------------------------------

## 未來

支援：

    Multiple GIRP Environment

增加：

-   Training speed
-   Experience collection

------------------------------------------------------------------------

# 18. Communication Protocol

使用：

    TCP Socket + JSON

流程：

GIRP：

    Game State
    ↓

    Observation

Python：

    Observation
    ↓

    PPO

    ↓

    Action

GIRP：

    Execute Action

    ↓

    Physics Step

    ↓

    Reward

------------------------------------------------------------------------

# 19. Logging 與 Evaluation

記錄：

-   Episode Reward
-   Average Height
-   Success Rate
-   Completion Time
-   Episode Length
-   Death Reason

視覺化：

-   Reward Curve
-   Success Curve
-   Height Progress
-   Completion Time

------------------------------------------------------------------------

# 20. Model Checkpoint

保存：

Regular Checkpoint：

    固定 Training Step 保存

Best Model：

依據：

-   最高 Success Rate
-   最佳 Completion Time

------------------------------------------------------------------------

# 21. Randomization

第一版不加入。

等固定地圖可以穩定完成後，再加入：

-   Hold Randomization
-   Physics Randomization
-   Initial State Randomization

目的：

提升泛化能力。

------------------------------------------------------------------------

# 22. Implementation Roadmap

## Phase 1

Environment Connection

完成：

-   GIRP Socket Server
-   Python Socket Client
-   JSON Protocol

## Phase 2

Observation Extraction

完成：

-   Physics State
-   Hand State
-   Hold Detection
-   Normalization

## Phase 3

Action Implementation

完成：

-   Target System
-   Contract System
-   Release System

## Phase 4

RL Training

完成：

-   PPO
-   Training Loop
-   Logging
-   Checkpoint

## Phase 5

Evaluation

評估：

-   Success Rate
-   Completion Time
-   Learned Strategy

------------------------------------------------------------------------

# 開發原則

    先做簡單版本

    ↓

    確認 AI 能學習

    ↓

    分析失敗原因

    ↓

    增加複雜度

不要在基礎版本未成功前增加：

-   更大的 Observation
-   更複雜 Action Space
-   Randomization
