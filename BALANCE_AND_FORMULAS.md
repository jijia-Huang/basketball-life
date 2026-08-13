# Basketball Life：數值與公式規格

版本：規格 v1.0  
對應企劃：[GAME_DESIGN_DOCUMENT.md](GAME_DESIGN_DOCUMENT.md)

本文件是第一版數值的唯一來源。程式、玩家文字與測試若和企劃摘要衝突，以此文件為準。所有係數均已定案；平衡調整只能改表內常數，不得暗加例外。

## 1. 共通規則

### 1.1 單位與取整

- 身高 `H`：整數公分；體重 `W`：整數公斤；BMI 保留一位小數顯示，計算使用未格式化值。
- 能力：20–80；每次永久變動後立即 `clamp(20, 80)`。
- 比率：內部使用 0–1；畫面乘 100 後保留一位百分比。
- GP、GS、MIN、命中／出手、籃板、助攻、抄截、阻攻、失誤、犯規及薪資：完成整季計算後四捨五入為整數。
- MPG、PTS、REB、AST、STL、BLK、TOV、PF：由整季總數除以 GP，顯示一位小數，不反向參與計算。
- FG%、3P%、FT%、eFG%、TS%：由已取整的整季命中／出手重算，顯示一位百分比。
- `round` 為四捨五入、`floor` 為無條件捨去、`clamp(x,a,b)=max(a,min(b,x))`。
- 所有表格的區間皆含上下界；落在兩格交界時取較高一格。

### 1.2 種子與亂數消耗順序

沿用原作 `seedInit`、`R()`、`ri()`、`chance()`；不得使用 `Math.random()` 產生遊戲結果。每個年度固定依下列順序消耗 RNG：

1. 身高成長。
2. 體重訓練公斤數。
3. 訓練骰數量與點數。
4. 球季狀態。
5. 角色範圍內的 GP、MPG、GS 與教練信任波動。
6. 事件卡及結果。
7. 數據波動。
8. 負荷與傷病。
9. 交易、冠軍、獎項、合約與國家隊。

呈現層不得呼叫遊戲 RNG。增刪動畫、重繪、時間軸或分享圖不能改變結果。

### 1.3 聯盟常數

`L` 同時是該聯盟的平均能力與 SeasonImpact 基準；`Pace` 是每隊每場回合；`GameLength` 都是 48 分鐘。

| 代碼 | 聯盟 | Games | L | Pace | 平均 TS% | 平均 BVI | 強度係數 |
|---|---|---:|---:|---:|---:|---:|---:|
| TW | 台灣職籃 | 40 | 42 | 90 | .550 | 27 | .45 |
| JP2 | 日本二部 | 52 | 46 | 91 | .555 | 28 | .55 |
| JP1 | 日本一部 | 60 | 52 | 92 | .565 | 30 | .70 |
| EUD | 歐洲國內 | 40 | 50 | 88 | .560 | 29 | .65 |
| EUT | 歐洲頂級 | 50 | 57 | 86 | .575 | 32 | .90 |
| GL | G League | 50 | 53 | 101 | .570 | 31 | .65 |
| NBA | 美職籃 | 82 | 60 | 100 | .580 | 34 | 1.00 |

## 2. 球員生成與身材

### 2.1 初始身高

先擲 `R()` 選區間，再於區間內等機率取整數：

| 機率 | 身高 |
|---:|---:|
| 5% | 165–174 cm |
| 25% | 175–184 cm |
| 35% | 185–194 cm |
| 25% | 195–204 cm |
| 10% | 205–211 cm |

初始 BMI 為 `19.0 + floor(R()*51)/10`，即 19.0–24.0、每 0.1 一格。`W=round(BMI×(H/100)^2)`。

### 2.2 年度身高成長

每年完全獨立抽取，不預先生成最終身高。超過 230 cm 的部分捨棄。

| 年齡 | 可成長 cm | 對應機率（由 0 cm 起） |
|---:|---|---|
| 17 | 0–7 | 5%、10%、15%、20%、20%、15%、10%、5% |
| 18 | 0–5 | 10%、20%、25%、25%、15%、5% |
| 19 | 0–3 | 25%、35%、25%、15% |
| 20 | 0–2 | 50%、35%、15% |
| 21 | 0–1 | 75%、25% |
| 22+ | 0 | 100% |

`H_new=min(230,H_old+Growth)`。長高不自動增重；玩家可在同年體重訓練補回對抗，因此突然抽高可能暫時偏瘦。

### 2.3 體重訓練

| 選擇 | 原始變化 |
|---|---:|
| 增肌 | `ri(2,5)` kg |
| 維持 | 0 kg |
| 減脂 | `-ri(2,4)` kg |

合法體重界線：

```text
Wmin = ceil(18 × (H/100)^2)
Wmax = floor(33 × (H/100)^2)
Wnew = clamp(Wold + change, Wmin, Wmax)
BMI  = Wnew / (H/100)^2
```

身材派生值：

```text
Mass       = clamp(50 + (BMI - 24) × 5, 30, 80)
Mobility   = clamp(62 + (ATH - 50) × 0.45 - max(0, BMI - 25) × 3, 20, 80)
Size       = clamp(50 + (H - 198) × 1.3 + (BMI - 24) × 1.2, 20, 80)
Collision  = clamp((W - (H - 100) × 0.90) × 0.18, -4, 4)
```

`Collision` 加到籃下命中修正；`Mass`、`Mobility`、`Size` 只在本文件明列的公式中使用，不改面板能力。

## 3. 能力成長與潛力

### 3.1 初始能力和潛力

十項能力先各取 `ri(20,32)`。再從十項中無重複抽兩項：第一項 `+ri(0,6)`、第二項 `+ri(0,4)`。

洗牌十項後依序給潛力：

- 第 1 項：`ri(72,80)`。
- 第 2–3 項：各 `ri(64,74)`。
- 第 4–6 項：各 `ri(56,68)`。
- 第 7–10 項：各 `ri(46,62)`。

### 3.2 年度訓練

骰數機率：3 顆 35%、4 顆 40%、5 顆 20%、6 顆 5%。每顆為 `ri(1,6)`；天才為 `ri(4,6)`，大器晚成為 `ri(3,6)`。

玩家逐顆指定能力。點數先進該能力的蓄力槽，達成本才升 1，剩餘點保留：

| 現值 | 每升 1 級成本 |
|---:|---:|
| 20–49 | 1 |
| 50–59 | 1 |
| 60–69 | 2 |
| 70–74 | 3 |
| 75–79 | 5 |

現值已達個人潛力時，本級成本再乘 3。能力 80 時不能投入；事件溢出改為當季 `FormBonus +1/點`，整季最多 +3。

業餘階段（包含高中與大學）每季另給可自由分配的球季成長點：

- 個人 BVI `<12`、`12–15.99`、`16–19.99`、`20–23.99`、`24–27.99`、`28–31.99`、`32–35.99`、`≥36`，依序給 3–10 點。
- 球隊勝率 `<35%`、`35–44.99%`、`45–49.99%`、`50–54.99%`、`55–59.99%`、`60–64.99%`、`65–74.99%`、`≥75%`，依序給 3–10 點。
- 總評再給 `floor(BaseOVR / 10)` 點。

三項合計後一次逐點分配，套用同一成本表與潛力規則。冠軍不另重複加碼。

## 4. 動態位置

### 4.1 身材適配

位置目標：

| 位置 | 目標身高 | 目標 BMI |
|---|---:|---:|
| PG | 185 | 23 |
| SG | 193 | 24 |
| SF | 201 | 25 |
| PF | 208 | 26 |
| C | 216 | 27 |

```text
HeightFit(pos) = clamp(80 - abs(H - targetH[pos]) × 2, 20, 80)
WeightFit(pos) = clamp(80 - abs(BMI - targetBMI[pos]) × 6, 20, 80)
```

### 4.2 技能適配與位置分數

```text
PGSkill = .30 HANDLE + .30 PASS + .15 THREE + .10 PERD + .10 ATH + .05 STA
SGSkill = .25 THREE + .20 FIN + .15 HANDLE + .15 PERD + .10 MID + .10 ATH + .05 PASS
SFSkill = .20 FIN + .15 THREE + .15 PERD + .15 ATH + .10 HANDLE + .10 REB + .10 INTD + .05 PASS
PFSkill = .20 REB + .20 INTD + .15 FIN + .15 ATH + .10 MID + .10 THREE + .05 PERD + .05 STA
CSkill  = .25 INTD + .20 REB + .20 FIN + .10 STA + .10 ATH + .05 PASS + .05 MID + .05 PERD

PositionScore(pos) =
  .75 × Skill(pos)
+ .20 × HeightFit(pos)
+ .05 × WeightFit(pos)
```

分數保留原值比較，畫面才四捨五入。最高分為主位置；第二名與最高分差 `≤4.0` 時為副位置。完全同分依 PG→SG→SF→PF→C 取先者，確保重現。

### 4.3 身材對位修正

```text
BodyFit = .75 × HeightFit(主位置) + .25 × WeightFit(主位置)
BodyMatch = clamp(round((BodyFit - 50) / 10), -3, 3)
```

## 5. 打法適性與綜合能力

### 5.1 進攻適性分數

```text
持球組織 = .35 PASS + .30 HANDLE + .15 THREE + .10 ATH + .10 STA

單打得分 = .30 HANDLE + .25 MID + .20 FIN + .15 ATH + .10 STA

外線射手 = .55 THREE + .15 STA + .10 MID + .10 ATH + .10 HANDLE

切入攻框 = .40 FIN + .25 ATH + .15 HANDLE + .10 STA + .10 MID
           + Collision

低位中樞 = .35 FIN + .25 REB + .15 INTD + .10 MID + .10 STA + .05 PASS
           + clamp((H-200)×.20,-3,3) + Collision
```

### 5.2 防守適性分數

```text
外線領防 = .45 PERD + .25 ATH + .20 STA + .10 HANDLE
           + clamp((198-H)×.12,-2,2)

換防萬用 = .30 PERD + .25 INTD + .20 ATH + .15 STA + .10 REB
           + min(2, HeightFit(SF)/20 - 1)

協防游擊 = .35 ATH + .30 PERD + .20 INTD + .15 STA

籃板卡位 = .50 REB + .20 STA + .15 INTD + .15 Mass

護框中樞 = .50 INTD + .20 REB + .15 ATH + .15 Size
```

所有適性分數 `clamp(20,80)`。

| 分數 | 等級 | SeasonImpact 修正 | 該打法數據係數 |
|---:|---|---:|---:|
| 64–80 | S | +4 | 1.10 |
| 56–63.99 | A | +2 | 1.05 |
| 46–55.99 | B | 0 | 1.00 |
| 20–45.99 | C | −6 | .90 |

### 5.3 BaseOVR 與 SeasonImpact

```text
BaseOVR = round(
  .45 × 最高進攻適性
+ .35 × 最高防守適性
+ .10 × STA
+ .10 × ATH
)
```

球季狀態在每季訓練後抽一次：−3（5%）、−2（10%）、−1（15%）、0（40%）、+1（15%）、+2（10%）、+3（5%）。事件溢出的 `FormBonus` 加在此處後，總狀態仍 `clamp(-3,3)`。

負荷懲罰：0–39 為 0；40–59 為 −1；60–79 為 −3；80–99 為 −6；100 為 −10。

```text
SeasonImpact = clamp(
  BaseOVR
+ 所選進攻適性修正
+ 所選防守適性修正
+ BodyMatch
+ SeasonForm
− LoadPenalty,
  20, 90
)
```

## 6. 名單、角色、分鐘與球權

### 6.1 名單資格與美職籃特例

TW、JP2、JP1、EUD、EUT、GL 的一般簽約興趣要求 `SeasonImpact ≥ L-8`，但選秀權、既有保證約和母隊養成可覆蓋一次。NBA **沒有能力硬門檻**：只要仍有 NBA 合約便留在名單，再由 `d=SeasonImpact-L` 决定角色。

NBA 合約到期後的留隊興趣分：

```text
PerformanceTrend = clamp(BVI_current-BVI_previous,-10,10)

Retention =
  SeasonImpact
+ clamp((PotentialOVR-BaseOVR)×.35,0,8)
+ PerformanceTrend×.20
+ PositionNeed         // 種子整數 -3..+3
− max(0,Age-30)×1.2
− MajorInjuries×2
```

`Retention≥58` 至少一隊正式約；52–57.99 有 55% 正式約、否則雙向／訓練營；46–51.99 有 20% 雙向／訓練營；低於 46 只有 5% 訓練營。仍在保證約內不擲此判定。
首次進入 NBA、沒有前季 BVI 時，`PerformanceTrend=0`。

### 6.2 角色表

先依 `d` 取角色。若 25 歲以下且 `PotentialOVR-BaseOVR≥8`，只把 GP 比率下限提高 0.10，不提升 MPG；合約不提高角色。第六人是一般板凳或輪替中 `USG≥.22` 且教練信任為正時的顯示角色，不改原分鐘範圍。

| d | 角色 | GP 比率 | MPG | GS 比率 | 基礎 USG |
|---:|---|---|---:|---:|---:|
| ≤−16 | DNP／名單末端 | .10–.35 | 0–6 | 0 | .10 |
| −15～−10 | 板凳邊緣 | .30–.65 | 4–10 | 0–.03 | .13 |
| −9～−5 | 一般板凳 | .55–.85 | 10–18 | .02–.10 | .16 |
| −4～0 | 輪替球員 | .75–.95 | 18–25 | .10–.35 | .19 |
| 1～4 | 先發 | .82–.98 | 25–31 | .70–.95 | .22 |
| 5～8 | 球隊核心 | .86–1.00 | 31–35 | .92–1.00 | .27 |
| ≥9 | 超級球星 | .88–1.00 | 34–38 | .97–1.00 | .32 |

範圍內以一次 `R()` 線性插值。之後：

```text
Availability = clamp(1 - InjuryMissRate, 0, 1)
GP  = round(Games × GPRate × Availability)
MPG = clamp(roundTo1(RandomMPG + (STA-L)×.06 - LoadPenalty×.20),0,38)
GS  = min(GP, round(GP × RandomGSRate))
MIN = round(GP × MPG)
```

`InjuryMissRate` 在第 9 節定義。若 `GP=0`，所有數據為 0 且不具獎項資格。

### 6.3 使用率

進攻打法 USG 修正：持球組織 +.02、單打得分 +.05、外線射手 +.01、切入攻框 +.03、低位中樞 +.02。

```text
PrimaryWeapon = max(FIN,MID,THREE,HANDLE)
USG = clamp(
  RoleBaseUSG
+ StyleUSG
+ (PrimaryWeapon-L)×.001
+ SeasonForm×.005,
  .08,.40
)
```

## 7. 賽季數據引擎

### 7.1 球權拆解

```text
OnCourtPossessions = Pace × MPG / 48
UsedPossessions    = OnCourtPossessions × USG
```

打法基礎罰球率 `FTr`、每次出手失誤率 `TOVperFGA`：

| 進攻打法 | FTr | TOVperFGA |
|---|---:|---:|
| 持球組織 | .30 | .18 |
| 單打得分 | .34 | .13 |
| 外線射手 | .16 | .08 |
| 切入攻框 | .52 | .16 |
| 低位中樞 | .42 | .15 |

```text
FTrAdjusted = clamp(FTr + (FIN-L)×.002, .08,.70)
TOVAdjusted = clamp(
  TOVperFGA
− (HANDLE-L)×.002
+ max(0,USG-.25)×.30,
  .05,.30
)

FGA_pg = UsedPossessions / (1 + .44×FTrAdjusted + TOVAdjusted)
FTA_pg = FGA_pg × FTrAdjusted
TOV_pg = FGA_pg × TOVAdjusted
```

### 7.2 出手分布與命中率

| 進攻打法 | 籃下 | 中距 | 三分 |
|---|---:|---:|---:|
| 持球組織 | 35% | 20% | 45% |
| 單打得分 | 35% | 35% | 30% |
| 外線射手 | 15% | 10% | 75% |
| 切入攻框 | 60% | 15% | 25% |
| 低位中樞 | 65% | 30% | 5% |

每季一次命中波動 `ShotLuck=(R()+R()-1)×.02`，範圍 −.02～+.02；同一 ShotLuck 套用三區，避免無意義地消耗多次 RNG。

```text
RimSize = clamp((H-198)×.002 + Collision×.005,-.05,.07)

Rim% = clamp(
  .48 + (FIN-L)×.006 + (ATH-L)×.002 + RimSize
  + SeasonForm×.005 + ShotLuck,
  .35,.80
)

Mid% = clamp(
  .34 + (MID-L)×.005 + SeasonForm×.004 + ShotLuck,
  .25,.58
)

3P% = clamp(
  .30 + (THREE-L)×.0035 + SeasonForm×.003 + ShotLuck,
  .20,.48
)

FT% = clamp(
  .55 + (((MID+THREE)/2)-30)×.006 + SeasonForm×.002,
  .45,.95
)
```

以 `FGA_pg×區域比例` 得各區場均出手；乘命中率得命中。整季先乘 GP 再取整：

```text
RimA = round(FGA_pg × RimShare × GP)
MidA = round(FGA_pg × MidShare × GP)
TPA  = round(FGA_pg × ThreeShare × GP)
RimM = round(RimA × Rim%)
MidM = round(MidA × Mid%)
TPM  = round(TPA × 3P%)
FTA  = round(FTA_pg × GP)
FTM  = round(FTA × FT%)

FGA = RimA + MidA + TPA
FGM = RimM + MidM + TPM
PTS = 2×(RimM+MidM) + 3×TPM + FTM
TOV = round(TOV_pg × GP)
```

### 7.3 助攻、防守與籃板

打法加成（每 36 分鐘）：

| 項目 | 加成 |
|---|---|
| 持球組織 AST | +2.5 |
| 單打得分 AST | −0.5 |
| 低位中樞 AST | +0.5 |
| 外線領防 STL | +0.45 |
| 協防游擊 STL／BLK | +0.35／+0.30 |
| 籃板卡位 REB | +2.5 |
| 護框中樞 BLK／REB | +0.75／+0.7 |
| 換防萬用 STL／BLK | +0.15／+0.15 |

先算每 36 分鐘：

```text
AST36 = clamp(4 + (PASS-L)×.28 + (HANDLE-L)×.08 + StyleAST, .5,14)

REB36 = clamp(5 + (REB-L)×.22 + (H-198)×.08 + (W-95)×.025
              + StyleREB, 1,20)

STL36 = clamp(1 + (PERD-L)×.035 + (ATH-L)×.020 + StyleSTL, .2,3.5)

BLK36 = clamp(.5 + (INTD-L)×.050 + (H-200)×.040 + (ATH-L)×.010
              + StyleBLK, 0,5)

PF36 = clamp(2.6 + StylePF - (STA-L)×.015, 1,6)
```

`StylePF`：協防游擊 +0.6、護框中樞 +0.5、外線領防 +0.3、其他 0。

乘 `MIN/36` 並取整得到 AST、REB、STL、BLK、PF。籃板拆分：

```text
OREBShare = clamp(.22 + (低位中樞?.10:0) + (籃板卡位?.08:0)
                  - (外線射手?.08:0), .08,.45)
OREB = round(REB × OREBShare)
DREB = REB - OREB
```

### 7.4 百分比與不變量

```text
FG%  = FGM/FGA
3P%  = TPM/TPA
FT%  = FTM/FTA
eFG% = (FGM + .5×TPM)/FGA
TS%  = PTS / (2×(FGA + .44×FTA))
```

分母為 0 時顯示 `—`，內部值為 0。完成後必須滿足：GP≤Games、GS≤GP、MPG≤38、所有累積數據非負、FGM≤FGA、TPM≤TPA、FTM≤FTA、TPM≤FGM、TPA≤FGA。

36 PPG、16 RPG、13 APG、4 BPG 是平衡警戒線而非硬截斷；超過時 selftest 報告校準失敗，開發者應調整公式常數，不能只截斷顯示值而破壞投籃與總分一致性。極端值必須只由頂尖能力、S 適性、34–38 MPG 與正向球季狀態共同產生。

## 8. 球隊戰績、季後賽與冠軍

每支玩家所屬球隊每年抽一次 `TeamContext=ri(-18,18)`，代表不個別模擬的隊友、教練與傷病。轉隊後重新抽；季中交易時按效力分鐘較多的球隊結算。

```text
WinPct = clamp(
  .50
+ TeamContext/100
+ (SeasonImpact-L) × (MPG/48) × .006,
  .10,.85
)

Wins   = round(Games × WinPct)
Losses = Games - Wins
```

季後賽資格機率：

```text
P(playoffs) = clamp(20 + (WinPct-.40)×180, 5, 100)
```

若進季後賽，冠軍機率：

```text
BaseChamp = 100 / LeagueTeamCount
P(champion) = clamp(
  BaseChamp
+ (WinPct-.50)×120
+ (SeasonImpact-L)×.8×(MPG/36),
  1,65
)
```

聯盟隊數使用 TW 6、JP2 12、JP1 12、EUD 12、EUT 12、GL 30、NBA 30。冠軍與個人獎項各用獨立 `chance()`，依第 1.2 節順序消耗 RNG。

總冠軍 MVP 只在奪冠後判定：

```text
FinalsScore = BVI + (SeasonImpact-L)×.4 + SeasonForm
P(FinalsMVP) = clamp(15 + (FinalsScore-LeagueFinalsThreshold)×8,5,95)
```

`LeagueFinalsThreshold` 為該聯盟 MVP 門檻減 3；超過門檻 10 分保證獲獎。

## 9. 生涯負荷與傷病

### 9.1 季初恢復與紅區選擇

```text
Recovery = clamp(14 + (STA-50)×.20 - max(0,Age-32)×.5, 6,20)
LoadStart = clamp(PreviousLoad-Recovery,0,100)
```

若 `LoadStart≥80`，在模擬球季前選：

- 輪休：本季 GP 比率乘 .65、MPG −3，`LoadStart−20`。
- 帶傷硬打：不降 GP／MPG，當季 InjuryChance +18%，MajorShare +15 個百分點。
- 手術／完整復健：本季 GP=0、所有數據 0、`Load=20`；下一季無額外報銷。

### 9.2 球季負荷

打法負荷：單打 +2、切入 +4、低位 +3、外線領防 +3、換防 +2、協防 +2、籃板卡位 +1、護框 +3；持球組織與外線射手 +0。

```text
StyleLoad = 進攻負荷 + 防守負荷
BMILoad   = max(0,18.5-BMI)×2 + max(0,BMI-28)×2
AgeLoad   = max(0,Age-30)×.8
IntlLoad  = 當年國際賽 ? 5 : 0

LoadGain = clamp(
  6
+ MPG×.45
+ max(0,USG-.20)×40
+ StyleLoad
+ BMILoad
+ AgeLoad
+ IntlLoad
− (STA-50)×.15,
  4,45
)

LoadEnd = clamp(LoadStart+LoadGain,0,100)
```

### 9.3 傷病判定

```text
LoadRisk = LoadEnd<40 ? 0
         : LoadEnd<60 ? 4
         : LoadEnd<80 ? 12
         : LoadEnd<100 ? 28 : 45

InjuryChance = clamp(
  5 + LoadRisk
+ max(0,Age-30)×1.2
+ max(0,18.5-BMI)×3
+ max(0,BMI-30)×3
+ PreviousMajorInjuries×3
+ HardPlayBonus,
  2,85
)
```

`HardPlayBonus` 只在帶傷硬打時為 18。失敗則無傷且 `InjuryMissRate=0`。受傷後重大傷病占比：

```text
MajorShare = clamp(12 + max(0,LoadEnd-60)×.8
                   + PreviousMajorInjuries×5
                   + (帶傷硬打?15:0),10,70)
```

先擲是否受傷，再擲重大／小傷。

- 小傷：`InjuryMissRate=ri(8,25)/100`；40% 留下後遺症，從 ATH、STA、PERD、INTD 中合法項隨機 −1。
- 大傷：依下表抽取；當季數據總量和 GP、GS、MIN 一起乘 `(1-InjuryMissRate)`，比率不變。永久能力立即扣除並重算未來位置與 OVR。

| 傷病 | 權重 | 缺賽率 | 永久效果 | 復發標記 |
|---|---:|---:|---|---:|
| 嚴重腳踝傷 | 35 | 30–55% | ATH −2、PERD −1 | +2% 傷病率 |
| 膝韌帶／ACL | 30 | 70–100% | ATH −5、STA −2 | +5% |
| 阿基里斯腱 | 15 | 100% | ATH −7、FIN −3 | +7% |
| 背傷 | 20 | 45–100% | STA −4、INTD −2 | +5% |

切入攻框把 ACL／阿基里斯權重各 +5、嚴重腳踝 −10；低位或護框把背傷 +10、腳踝 −10。權重調整後重新正規化。

## 10. 選秀、合約與全球流動

### 10.1 PotentialOVR

把十項目前能力替換為各自潛力，重新計算所有打法適性和 BaseOVR，即得 `PotentialOVR`。身高、體重仍使用申報當下值。

### 10.2 美職籃選秀

```text
ProductionRating = clamp(50 + (BVI-LeagueAverageBVI)×1.2,20,80)
AgeBonus = 19歲:+4, 20歲:+2, 21歲:0, 22歲:-3
SizeRarity = clamp((BodyFit-50)×.10,-3,3)
ScoutNoise = ri(-4,4)

DraftScore =
  .45×SeasonImpact
+ .25×PotentialOVR
+ .20×ProductionRating
+ AgeBonus
+ SizeRarity
− MajorInjuries×2
− max(0,LoadEnd-70)×.08
+ ScoutNoise
```

依 DraftScore 抽結果：

| 分數 | 樂透 1–14 | 首輪 15–30 | 次輪 31–60 | 落選 |
|---:|---:|---:|---:|---:|
| ≥75 | 100% | 0 | 0 | 0 |
| 70–74.99 | 65% | 30% | 5% | 0 |
| 65–69.99 | 15% | 60% | 20% | 5% |
| 60–64.99 | 0 | 30% | 55% | 15% |
| 55–59.99 | 0 | 5% | 45% | 50% |
| <55 | 0 | 0 | 1% | 99% |

選中區段後另擲 `PickNoise=ri(-2,2)`：

```text
樂透順位 = clamp(round(14-(DraftScore-75))+PickNoise,1,14)
首輪順位 = clamp(round(30-(DraftScore-60)×1.5)+PickNoise,15,30)
次輪順位 = clamp(round(60-(DraftScore-55)×2)+PickNoise,31,60)
```

DraftScore 88 以上的樂透球員固定為狀元，不再套 PickNoise。

### 10.3 薪資

所有金額單位為「萬新台幣／年」。

| 聯盟 | BaseSalary |
|---|---:|
| TW | 150 |
| JP2 | 500 |
| JP1 | 2,500 |
| EUD | 1,800 |
| EUT | 7,000 |
| GL | 150 |
| NBA | 3,500 |

| 角色 | RoleMultiplier |
|---|---:|
| DNP／名單末端 | 1.00 |
| 板凳邊緣 | 1.05 |
| 一般板凳 | 1.25 |
| 輪替 | 1.80 |
| 第六人 | 2.60 |
| 先發 | 3.50 |
| 核心 | 12.0 |
| 超級球星 | 45.0 |

```text
AgeMarket = Age≤29 ? 1 : clamp(1-(Age-29)×.06,.55,1)
HealthMarket = clamp(1-MajorInjuries×.08,.60,1)
Salary = round(BaseSalary × RoleMultiplier × AgeMarket × HealthMarket)
```

長約年薪乘 .92、短約乘 1.12。長約 3–5 年、保障 80–100%；短約 1–2 年、保障 30–70%。年限由角色决定：DNP／邊緣 1 年、一般板凳／輪替 1–3 年、先發 2–4 年、核心／超級球星 3–5 年；32–33 歲最多 4 年、34–35 最多 3 年、36+ 最多 2 年。

美職籃首輪新秀 4 年、前兩年 100% 保障、後兩年球隊選項；年薪由順位在 3,500–35,000 萬間線性插值。次輪 1–3 年、保障 0–50%、年薪 3,000–5,000 萬。雙向合約 1 年、年薪 1,800 萬，最多計 50 場美職籃出賽；超過後須轉正式約或回 G League。

被裁買斷：`剩餘年薪×剩餘年數×保障比例` 一次加入生涯總薪資。遊戲不實作薪資帽、奢侈稅或球隊財務。

## 11. 獎項算法

### 11.1 通用指標

以下 PTS、REB、AST、STL、BLK、TOV 均以整季總量除 GP 的未格式化場均值計算，不使用畫面的一位小數；TS 為 0–1。

```text
BVI = PTS + 1.2×REB + 1.5×AST + 3×STL + 3×BLK - 1.2×TOV
      + (TS%-LeagueAverageTS%)×40

Attendance = clamp((GP/Games)×min(1,MPG/30),0,1)
TeamFactor = .75 + WinPct×.50
MVPScore = BVI × Attendance × TeamFactor

DefenseScore =
  2.5×STL + 2.5×BLK + .35×DREB
+ (PERD+INTD-2×L)×.12
+ DefenseStyleBonus
```

`DefenseStyleBonus`：外線領防 1.0、換防萬用 1.2、協防游擊 1.0、籃板卡位 .6、護框中樞 1.2。

機率函式：

```text
AwardP(score,threshold,scale) =
  score ≥ threshold+3×scale ? 100
  : clamp(100/(1+exp(-(score-threshold)/scale)),1,95)
```

資格不符時為 0%。符合時依「數據王→明星賽→年度隊→新人王／第六人／進步最快→DPOY→MVP→冠軍→Finals MVP」順序各擲一次。

### 11.2 基準表

| 聯盟 | 明星 BVI | 年度三隊 BVI | MVPScore | DPOY DefenseScore | 新人王 BVI | 第六人 BVI |
|---|---:|---:|---:|---:|---:|---:|
| TW | 30 | 34 | 38 | 10 | 27 | 28 |
| JP2 | 31 | 35 | 39 | 10.5 | 28 | 29 |
| JP1 | 34 | 38 | 42 | 11.5 | 31 | 32 |
| EUD | 33 | 37 | 41 | 11 | 30 | 31 |
| EUT | 36 | 40 | 45 | 12.5 | 33 | 34 |
| GL | 35 | 39 | 43 | 11 | 32 | 33 |
| NBA | 38 | 43 | 48 | 13 | 35 | 36 |

- 明星賽：`AwardP(BVI,明星BVI,3)`；資格 GP≥Games×.50 且 MPG≥18。TW、JP2、EUD、GL 也顯示明星賽。
- 年度隊：資格 GP≥Games×.65、MPG≥24。依序擲一隊 `threshold+8`、二隊 `threshold+4`、三隊 `threshold`，scale 都為 2.5；中一項後停止。
- 年度防守隊：用 DefenseScore，資格同年度隊；一隊門檻 `DPOY門檻-1`、二隊 `-3`，scale 1.5，中一項後停止。
- MVP：`AwardP(MVPScore,表列,3)`；資格 GP≥Games×.70、MPG≥28。
- DPOY：`AwardP(DefenseScore,表列,1.5)`；資格 GP≥Games×.65、MPG≥24。
- 新人王：第一個該聯盟賽季且 Age≤23；GP≥Games×.50、MPG≥16；`AwardP(BVI,表列,3)`。
- 第六人：`GS/GP<.50`、GP≥Games×.60、MPG≥18；`AwardP(BVI,表列,3)`。
- 進步最快：前季同聯盟 MPG≥12、本季 MPG≥18；`Improvement=BVI-current - BVI-previous`，`AwardP(Improvement,7,1.5)`。

### 11.3 數據王

資格：GP≥Games×.70、MPG≥24；抄截／阻攻王可放寬至 MPG≥20。

| 聯盟 | PTS | REB | AST | STL | BLK | scale（依序） |
|---|---:|---:|---:|---:|---:|---|
| TW | 24 | 12 | 8 | 2.0 | 2.2 | 1.5／1／.8／.25／.30 |
| JP2 | 23 | 11 | 7.5 | 1.9 | 2.1 | 同上 |
| JP1 | 24 | 11.5 | 8 | 2.0 | 2.2 | 同上 |
| EUD | 22 | 10.5 | 7 | 1.8 | 2.0 | 同上 |
| EUT | 23 | 11 | 7.5 | 1.9 | 2.1 | 同上 |
| GL | 27 | 12 | 8.5 | 2.1 | 2.3 | 同上 |
| NBA | 28 | 12.5 | 9 | 2.1 | 2.5 | 同上 |

每項使用 `AwardP(場均值,表列門檻,對應scale)`。因此 NBA 32.5 PPG、15.5 RPG、11.4 APG、2.85 STL 或 3.4 BLK 分別達保證線。

## 12. 國際賽

徵召分：

```text
NationalScore = SeasonImpact + LeagueStrength×8 - LoadPenalty
```

世界盃門檻 58、奧運 61；台灣與日本門檻各 −6，歐洲國家 −2，美國不修正。超過門檻必徵召；低 1–5 分依每差 1 分減少 18% 機率，低超過 5 分不徵召。

國際賽使用 EUT 的 L=57、Pace=86、Games=8 重跑數據引擎，MPG 為職業 MPG 的 .80，最低 8、最高 34，USG 不變。隊伍結果用 `chance()`：

```text
TeamIntl = NationalScore + ri(-8,8)
≥70 冠軍、65–69.99 亞軍、61–64.99 季軍、56–60.99 八強、<56 未晉級
```

賽會 MVP 只限獎牌球員：`AwardP(BVI,42,3)`，冠軍分數 +3。

## 13. 隱藏特質

| 特質 | 觸發 | 精確效果 |
|---|---|---|
| 天才 | 22 歲前訓練骰累積 5 次 6 | 訓練骰改 4–6 |
| 大器晚成 | 26 歲後首次 BaseOVR +5 | 訓練骰改 3–6 |
| 鐵人 | 連續 5 季 GP≥Games×.90 | InjuryChance 上限 10% |
| 玻璃人 | 25 歲前兩次大傷 | InjuryChance 下限 35% |
| 自律狂 | 連續 5 年選維持且無場外負面事件 | 衰退年齡延後 2 年、Recovery +3 |
| 學院派 | 完成四年大學且獲年度隊 | 25 歲前 InjuryChance −5%、SeasonForm 負值提高 1 |
| 大心臟 | 全力事件成功 5 次 | 全力成功率 +15%、成功 +4、失敗 −2 |
| 神主牌 | 同隊 10 年且 5 次明星賽 | 原隊薪資至少市場價 1.2 倍、名人堂 +250 |
| 微波爐 | 替補身份 3 季平均 ≥15 PPG | 第六人角色 USG +.03 |
| 球場指揮官 | 單季 ≥9 AST 且 AST/TOV≥3 | 持球組織適性 +4 |
| 防守大鎖 | 外線領防下入選防守一隊 | 外線領防適性 +4 |
| 禁區支柱 | 護框下獲 DPOY 或阻攻王 | 護框適性 +4 |
| 耐戰體質 | 連續 5 季 ≥30 MPG 且無大傷 | LoadGain ×.80 |
| 國際賽之鬼 | 5 次徵召或賽會 MVP | IntlLoad=0、國際賽 SeasonForm 最低 +1 |
| 浴火重生 | 大傷復出後獲年度獎 | 移除玻璃人下限，但保留永久能力損失 |
| 更衣室毒瘤 | 兩次公開抱怨或重大場外事件 | 交易率 +25%、Retention −5 |
| 氣氛大師 | 三次主動要求交易 | 交易率 +20%、合約保障上限 −20% |
| 外務纏身 | 連續兩次代言失敗 | 每季訓練骰 −1，最低 2 |
| 歷史級球星 | 首輪入選名人堂 | 結算標籤，無遊戲中效果 |

適性特質加成在適性分數 clamp 前加入；同類只套一次。

## 14. 衰退、退休與名人堂

一般球員 31 歲開始衰退；自律狂 33 歲。每年季初、訓練前扣能力：

| 有效年齡 | 全能力 |
|---:|---:|
| 31–33 | −1 |
| 34–35 | −2 |
| 36–37 | −3 |
| 38+ | `−(4 + Age-38)` |

48 歲強制退休。無合約球員會進行兩輪全球市場；兩輪都無報價時，原隊提供一年、標準年薪 70% 的保底合約，球員也可選擇退休。35 歲以上每次合約到期另提供主動退休。

### 14.1 生涯價值

每季：

```text
SeasonCareerValue = max(0,BVI-12) × GP × LeagueStrength / 6
```

榮譽分：MVP +500、DPOY +400、年度一／二／三隊 +220／150／100、防守一／二隊 +160／100、數據王 +150、明星賽 +60、新人王／第六人／進步最快 +100、總冠軍 +120、Finals MVP +300、奧運／世界盃金銀銅 +250／150／100、賽會 MVP +200、神主牌 +250。

```text
CareerScore = round(sum(SeasonCareerValue)+sum(HonorPoints))
```

| 分數 | 結果 |
|---:|---|
| ≥5,000 | 全球籃球名人堂 |
| 3,500–4,999 | 傳奇球星 |
| 2,300–3,499 | 明星球員 |
| 1,200–2,299 | 可靠輪替 |
| 500–1,199 | 邊緣球員 |
| <500 | 一頁過客 |

名人堂球員退休 5 年後投票。`CareerScore≥6,500` 首輪入選；否則入選年份為 `2+floor((6500-CareerScore)/300)`，clamp 2–6。得票率 `clamp(75+(CareerScore-5000)/100,75,99.1)`。

## 15. 完整驗算案例

以下案例把區間亂數固定為指定值，方便實作測試。未列出的事件、傷病與國際賽均視為未觸發；命中波動 `ShotLuck=0`，除案例 2 為 +.005。數字由本文件公式計算，顯示值已依第 1.1 節取整。

### 15.1 178 cm 持球後衛

條件：NBA、178 cm／75 kg／BMI 23.7；STA 66、ATH 72、FIN 68、MID 54、THREE 62、HANDLE 74、PASS 76、REB 38、PERD 63、INTD 30；選持球組織＋外線領防；SeasonForm +1、負荷懲罰 0。

```text
位置分：PG 70.3、SG 63.4、SF 56.3、PF 46.8、C 46.1 → PG
持球組織 74.9(S/+4)；外線領防 69.0(S/+4)
BaseOVR = round(.45×74.9+.35×69.0+.10×66+.10×72) = 72
BodyMatch=+2
SeasonImpact = 72+4+4+2+1 = 83
d = 83-60 = 23 → 超級球星
```

固定 GP 比率 .94、MPG 35、GS 比率 .98：GP 77、GS 75、MIN 2,695。`USG=.359`、場均在場回合 72.92、使用回合 26.18。

完整結果：1,523 FGA、586 FGM、685 3PA、212 3PM、481 FTA、346 FTM、1,730 PTS、75 REB、906 AST、134 STL、0 BLK、281 TOV。場均為 **22.5 PTS、1.0 REB、11.8 AST、1.7 STL、3.6 TOV，FG 38.5%、3P 30.9%、FT 71.9%、TS 49.9%**。

這名球員雖然 OVR 高，但投射只略高於 NBA 平均、出手又有 45% 是三分，因此效率不會被 OVR 自動灌高；他的價值主要來自組織。

### 15.2 201 cm 全能側翼與 MVP

條件：NBA、201 cm／99 kg／BMI 24.5；STA 72、ATH 70、FIN 72、MID 68、THREE 69、HANDLE 66、PASS 62、REB 64、PERD 71、INTD 67；選單打得分＋換防萬用；SeasonForm +2、ShotLuck +.005。

```text
位置分：SF 71.4、SG 68.5、PF 68.0、C 64.4、PG 63.0 → SF/SG
單打得分 68.9(S/+4)；換防萬用 71.3(S/+4)
最高進攻適性 71.7；最高防守適性 71.3
BaseOVR = 71；BodyMatch=+3
SeasonImpact = 71+4+4+3+2 = 84
d=24 → 超級球星
```

固定 GP 比率 .95、MPG 36、GS 比率 .99：GP 78、GS 77、MIN 2,808、USG .392。完整結果：1,737 FGA／782 FGM、521 3PA／178 3PM、632 FTA／496 FTM、2,238 PTS、485 REB、354 AST、135 STL、89 BLK、279 TOV。

場均為 **28.7 PTS、6.2 REB、4.5 AST、1.7 STL、1.1 BLK、3.6 TOV，FG 45.0%、3P 34.2%、FT 78.5%、TS 55.5%**，以未格式化場均計得 BVI 46.30。TeamContext +10 得 WinPct .708。

```text
Attendance = (78/82)×1 = .9512
TeamFactor = .75+.708×.50 = 1.104
MVPScore = 46.297×.9512×1.104 = 48.619
P(MVP) = 100/(1+exp(-(48.619-48)/3)) = 55.1%
```

若該種子的 MVP 骰為 41.3，因 41.3≤55.1，獲得 NBA MVP；若骰為 60 則落選。這讓同等級球季不會年年必拿，但分數達 57 時必定得獎。

### 15.3 218 cm 護框長人、DPOY 與阻攻王

條件：EUT、218 cm／122 kg／BMI 25.7；STA 68、ATH 60、FIN 76、MID 52、THREE 35、HANDLE 40、PASS 52、REB 78、PERD 48、INTD 77；選低位中樞＋護框中樞；SeasonForm 0。

```text
位置分：C 71.6、PF 65.3、SF 57.1、SG 47.7、PG 42.5 → C
低位中樞 78.1(S/+4)；護框中樞 74.8(S/+4)
BaseOVR=74；BodyMatch=+3
SeasonImpact=74+4+4+3=85
d=85-57=28 → 超級球星
```

固定 GP 比率 .92、MPG 33、GS 比率 .96：GP 46、GS 44、MIN 1,518、USG .359。場均為 **20.4 PTS、11.5 REB、1.6 AST、0.7 STL、2.7 BLK、3.2 TOV，FG 53.1%、3P 23.5%、FT 63.2%、TS 56.7%**，BVI 42.69。

其中 DREB 為 361/46=7.85：

```text
DefenseScore = 2.5×(31/46) + 2.5×(126/46) + .35×(361/46)
             + (48+77-2×57)×.12 + 1.2
             = 13.80
P(DPOY) = logistic(13.80,12.5,1.5) = 70.4%

P(阻攻王) = logistic(126/46,2.1,.30) = 89.4%
```

若該種子兩次獎項骰分別為 35 與 75，則同時取得 DPOY 與阻攻王。若阻攻達 3.0，等於門檻加三個 scale，改為保證得獎。

### 15.4 能力不足但仍在 NBA 的板凳邊緣人

條件：NBA、190 cm／86 kg／BMI 23.8；STA 45、ATH 47、FIN 43、MID 42、THREE 51、HANDLE 44、PASS 40、REB 40、PERD 48、INTD 34；選外線射手＋外線領防；SeasonForm −1。球員仍在首輪新秀保證約第二年，因此不做 Retention 判定。

```text
位置分：SG 53.3、PG 51.2、SF 48.5、PF 43.8、C 39.2 → SG/PG
外線射手 48.1(B/0)；外線領防 47.7(B/0)
BaseOVR=48；BodyMatch=+3
SeasonImpact=48+0+0+3-1=50
d=50-60=-10 → 板凳邊緣
```

固定 GP 比率 .48、MPG 7、GS 比率 .01：GP 39、GS 0、MIN 273、USG .126。完整投籃為 61 FGA／17 FGM、46 3PA／12 3PM、8 FTA／5 FTM，得到 51 分與 7 次失誤。

場均為 **1.3 PTS、約 0.2 REB、0.1 AST、0.2 STL、0 BLK、0.2 TOV，FG 27.9%、3P 26.1%、FT 62.5%**。這符合 2–10 MPG、0–5 PPG 的末端角色目標。

他因保證約繼續留隊，不因 OVR 被立即踢出 NBA；合約到期才用 Retention。其 GP、MPG 不符合任何獎項資格，所以所有獎項機率直接為 0%，不會因小樣本命中率或種子運氣爆冷得獎。

## 16. 實作自我檢查

`?selftest=1` 必須在 console 輸出單一 `Basketball Life selftest: PASS`，否則逐項列出失敗。至少斷言：

1. 同種子完整跑兩次，球員生成與年度結果 JSON 深比較相等。
2. 任何身高成長後 `H≤230`，BMI 訓練後介於 18–33。
3. 本節四個固定案例的位置、BaseOVR、SeasonImpact、GP、MPG 和主要數據與列值一致，總量允許四捨五入誤差 1。
4. 任何模擬結果滿足第 7.4 節不變量。
5. NBA SeasonImpact 20 的保證約球員仍可存在，角色為 DNP；合約到期才判斷 Retention。
6. 獎項資格不符時機率為 0，超過保證線時為 100。
7. 呈現函式重跑不改 RNG 狀態。
