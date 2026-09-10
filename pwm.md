# PWM（RT584 / RT58x）周邊文件

| 項目 | 內容 |
| --- | --- |
| 周邊 | PWM（PWM0 ~ PWM4，5 組獨立 PWM 模組）+ TIMER0/1/2 內建的 PWM 輸出功能 |
| RM 章節 | **Ch22 PWM**；timer 側見 **Ch13 Timer §PWM Function** |
| 驗證表 sheet | `RT584_MPA_IC_peripheral_verification_status_20241008.xlsx` → **PWM** |
| Examples | `examples/peripheral/pwm/` 共 **4 個 project**；`examples/peripheral/timer/timer-pwm`、`timer-pwm-button` 另外 2 個 |
| Driver | [pwm.c](components/platform/soc/rt584/rt584_driver/Src/pwm.c) / [pwm.h](components/platform/soc/rt584/rt584_driver/Inc/pwm.h) / [pwm_reg.h](components/platform/soc/rt584/rt584_driver/Inc/pwm_reg.h) |
| HOSAL | [hosal_pwm.c](components/platform/hosal/rt584_hosal/Src/hosal_pwm.c) / [hosal_pwm.h](components/platform/hosal/rt584_hosal/Inc/hosal_pwm.h) |
| 輸出多工 | `SYSCTRL->soc_pwm_sel`（offset 0x4C），5 個 2-bit 欄位 |
| 相關文件 | [timer.md](timer.md)、[pwm-zerocross-dimmer.md](pwm-zerocross-dimmer.md) |

> ⚠ **「判定條件」一律留 `待實測`** —— 未實際上機跑過的不推測。
> ⚠ RM 章號由 docx Heading-1 順序推算，章名可信、章號請對一次實體 TOC。

---

## 快速上手：只要記三件事

**先看這一節就好。** 後面 §1 之後都是需要時再翻的參考，一次讀完會被數學淹死。

九成的需求（調光、調速、調功率、發聲）用 **Timer PWM** 就夠，而它只有三個數字：

```
prescale        →  一格有多久
timeout_ticks   →  幾格算一個週期   → 決定頻率
threshold       →  前幾格是高電位   → 決定 duty
```

`prescale = 16` 的話：`32 MHz ÷ 16 = 2 MHz`，**一格 = 0.5 µs**。之後只要問兩個問題：

1. 我要的一個週期是幾微秒？除以 0.5 → 填 `timeout_ticks`
2. 其中要亮多久？同樣除以 0.5 → 填 `threshold`

| 我要 | 週期 | `timeout_ticks` | `threshold`（50%） |
| --- | --- | --- | --- |
| 100 Hz | 10000 µs | 20000 | 10000 |
| 1 kHz | 1000 µs | 2000 | 1000 |
| 10 kHz | 100 µs | 200 | 100 |

**調亮度只改 `threshold`**，頻率不用動：25% 就填 `timeout_ticks / 4`。

程式碼就這樣（完整可跑版本見 §4.6）：

```c
hosal_timer_pwm_config_tick_t tick;
tick.timeload_ticks = 0;
tick.timeout_ticks  = 2000;   /* 1 kHz */
tick.threshold      = 1000;   /* 50% */
tick.phase          = 0;
hosal_timer_pwm_start(RT_TIMER0, tick);   /* 可重複呼叫改參數，不用先 stop */
```

**什麼時候才需要讀後面那一大串？** 只有一種情況：你要**硬體自己播一整串會變化的 duty**
（呼吸燈、燈條動畫、CPU 完全不介入）。那是 PWM 模組的 sequence controller，
只有它做得到，代價就是 §1.3~§1.6 那些格式與位元打包。

---

## 0. PWM 是要拿來幹嘛的

PWM = Pulse Width Modulation，脈寬調變。**週期固定，只改「一個週期裡有多久是高電位」**（duty，佔空比）。

```
duty 25%   ┌─┐     ┌─┐     ┌─┐
           │ │     │ │     │ │
        ───┘ └─────┘ └─────┘ └───

duty 75%   ┌───┐   ┌───┐   ┌───┐
           │   │   │   │   │   │
        ───┘   └───┘   └───┘   └─
        |<-- 週期 -->|
```

為什麼要這樣做？因為**數位腳只會輸出 0 或 1，做不出「一半的電壓」**。但如果切換得比負載的反應速度快很多，負載感受到的就是平均值。所以 PWM 是「用數位腳做出類比效果」最便宜的方法 —— 不用 DAC，不用外部電路。

常見用途：

| 用途 | 怎麼用 | 頻率量級 |
| --- | --- | --- |
| **LED 調光** | duty 就是亮度。眼睛的積分速度慢，看到的是平均亮度 | > 200 Hz（低於此會看到閃爍） |
| **馬達轉速 / 風扇** | duty 就是等效電壓。線圈電感自己會把方波平滑掉 | 幾 kHz ~ 20 kHz（避開人耳） |
| **加熱器 / 電源** | duty 就是功率比例 | 幾百 Hz ~ 幾十 kHz |
| **伺服馬達（SG90 之類）** | 這種是**看脈寬絕對值**不是看比例：週期固定 20 ms，脈寬 1.0~2.0 ms 對應角度 | 50 Hz 固定 |
| **蜂鳴器 / 簡單音調** | 這種**是看頻率**，duty 固定 50%，改頻率就改音高 | 幾百 Hz ~ 幾 kHz |
| **紅外線遙控載波** | 38 kHz 載波 + 開關調變（不過 RT584 有專用的 IRM 模組） | 38 kHz |
| **AC 相位切割調光（TRIAC）** | 這種**是看延遲時間**：過零點後延遲多久才發觸發脈衝 = 亮度。詳見 [pwm-zerocross-dimmer.md](pwm-zerocross-dimmer.md) | 100 / 120 Hz |

所以「PWM 拿來幹嘛」其實有三種完全不同的用法，看你在乎的是哪個量：

1. **在乎 duty 比例** → 調光、調速、調功率（最常見）
2. **在乎脈寬絕對值** → 伺服馬達
3. **在乎頻率** → 發聲、產生載波

RT584 的 PWM 模組**三種都能做**，但介面設計明顯是為第 1 種、而且是為「LED 燈條效果」最佳化的（後面 §1.4 的 sequence controller 就是為此存在）。

---

## 1. 原理

### 1.1 全景：5 個輸出通道，每個通道有 4 個可能的來源

這是理解 RT584 PWM 最關鍵的一張圖，也是 SDK 文件裡沒講的部分：

```
  PWM0 模組 ─┐
  TIMER0    ─┤
  TIMER1    ─┼──► [soc_pwm_sel.pwm0_src_sel] ──► pwm0 輸出 ──► [pin mux MODE_PWM0] ──► 任一支 GPIO
  TIMER2    ─┘

  PWM1 模組 ─┐
  TIMER0    ─┤
  TIMER1    ─┼──► [soc_pwm_sel.pwm1_src_sel] ──► pwm1 輸出 ──► [pin mux MODE_PWM1] ──► 任一支 GPIO
  TIMER2    ─┘

  ... pwm2 / pwm3 / pwm4 同理
```

`SYSCTRL->soc_pwm_sel`（[sysctrl_reg.h:123-133](components/platform/soc/rt584/rt584_driver/Inc/sysctrl_reg.h#L123-L133)）有 5 個 2-bit 欄位：

| 欄位 | bits | 值 |
| --- | --- | --- |
| `pwm0_src_sel` | [1:0] | 0 = PWM 模組（reset 值）／1 = TIMER0／2 = TIMER1／3 = TIMER2 |
| `pwm1_src_sel` | [3:2] | 同上 |
| `pwm2_src_sel` | [5:4] | 同上 |
| `pwm3_src_sel` | [7:6] | 同上 |
| `pwm4_src_sel` | [9:8] | 同上 |

⚠ **只有 TIMER0/1/2 在這個多工器上。** 俗稱的 timer3/4（`SLOWTIMER0/1` = RM 的
TIMER32K0/1）連 PWM 暫存器都沒有，不是 SDK 少寫。證據見 [timer.md §1.0](timer.md)。

值 1/2/3 = timer 是從 [timer.c:466-484](components/platform/soc/rt584/rt584_driver/Src/timer.c#L466-L484) 讀出來的（`= timer_id + 1`）。值 0 = PWM 模組是**推論**：`pwm.c` 從頭到尾沒碰過這顆暫存器，而 PWM 模組的 4 個 example 用 reset 值就能出波形。→ 列入 §7 待確認。

**兩個重要推論：**

1. **最多同時 5 路 PWM 輸出**，不管來源是 PWM 模組還是 timer —— 因為瓶頸在這 5 個輸出通道，不是在來源數量。
2. **同一個通道不能兩個來源一起用。** 你如果 `timer_pwm_open()` 開了 pwm0_enable，就把 pwm0 通道搶去指向那顆 timer，PWM0 模組再怎麼設都出不來。反之亦然。

輸出腳位**不限定**：`pin_set_mode(pin, MODE_PWM0)` 對 pin 0~31（除 10/11 是 SWD）都成立（[sysctrl.c:347-375](components/platform/soc/rt584/rt584_driver/Src/sysctrl.c#L347-L375)），會順便設成輸出、關掉 pull。

### 1.2 PWM 模組 vs Timer PWM：差在哪

| | **PWM 模組**（PWM0~4） | **Timer PWM**（TIMER0/1/2） |
| --- | --- | --- |
| 數量 | 5 組 | 3 顆 timer，每顆可同時驅動多個輸出通道 |
| 計數器寬度 | **15-bit**（3 ~ 32767） | **32-bit** |
| 時脈來源 | `SystemCoreClock`（跟著 CPU 頻率跑） | PERI clock **固定 32 MHz** |
| 分頻 | ÷1 ~ ÷256（3-bit enum）或 8-bit `ck_div_val` | 3-bit enum（÷1~÷1024）或 10-bit `prescale` |
| duty 怎麼設 | 記憶體裡的 element（可一整串自動播） | 暫存器 `thd` 一個值 |
| 波形序列 | **有**，sequence controller + xDMA，可自動播一整串 duty | **沒有**，只能 CPU 自己改 |
| 中斷 | 有（Pwm0~4_IRQn = 32~36），7 種來源 | 有，timer 自己的 timeout 中斷 |
| 適合 | LED 呼吸燈 / 燈條效果 / 要一長串變化的 | 單純固定 duty、要長週期、或要 32-bit 解析度的 |
| 缺點 | 15-bit 計數器上限低；driver 一堆 bug（§6） | 沒有序列功能，每次改都要 CPU 進去；佔掉一顆 timer |

**選擇建議**

- 要「設一次就不管、固定 duty」→ **Timer PWM** 更簡單，而且 32-bit 計數器不會爆。
- 要「一長串 duty 自動變化」（呼吸燈、燈條動畫）→ 只有 **PWM 模組** 做得到。
- 要「低頻長週期」（例如 1 Hz）→ **Timer PWM**。PWM 模組最低約 `32MHz ÷ 256 ÷ 32767 ≈ 3.8 Hz`。
- 要「執行中頻繁改參數」→ 兩者都要 CPU 介入，差別不大；見 `timer-pwm-button` 範例。

### 1.3 PWM 模組的時脈鏈

```
SystemCoreClock ──► [ck_div 或 ck_div_val] ──► pwm_tick ──► [15-bit counter, 0 ~ cnt_end] ──► 比較 thd ──► pwm_o
                       ÷1 ~ ÷256                                                                │
                                                                                        phase 決定極性
```

兩級結構，**兩個東西都會影響頻率**：

1. **分頻器** `cfgX_ck_div` [11:8]，enum 0~8 = ÷1、÷2、÷4 … ÷256（`pwm_clk_div_t`）
2. **計數上限** `cnt_end`，範圍 3 ~ 32767（`PWM_COUNT_END_VALUE_MIN/MAX`）

輸出頻率 = `SystemCoreClock / (1 << ck_div) / (cnt_end + 1)`

⚠ **注意 `SystemCoreClock` 不是固定 32 MHz** —— PWM 分頻是接在 CPU 時脈上，[pwm.c 的 `pwm_set_frequency()`](components/platform/soc/rt584/rt584_driver/Src/pwm.c) 直接用 `SystemCoreClock / (1 << cfg_ck_div)`。這跟 timer 的 PERI clock 固定 32 MHz 不一樣，改 CPU 頻率時 PWM 頻率會跟著跑掉。

⚠ **兩個分頻器互斥**：`cfgX_ck_div` [11:8]（reset 0）「只有在 `ck_div_val` = 0 時有效」；`cfgX_ck_div_val` [31:24]（reset 0）「只有 > 0 時有效」。後者**在 `pwm_set_t` 裡根本沒宣告**（落在 `RVD4`），所以 SDK 只能用前者。目前能運作是因為 `pwm_init_fmt0/1` 整個 word 寫下去、順手把 `ck_div_val` 清成 0 —— 巧合，不是保證（§6 P11）。

### 1.4 兩種取樣格式（element 格式）

PWM 模組的 duty 不是寫暫存器，而是**寫一個 32-bit 的 element**，由 xDMA 餵進去。有兩種打包格式：

**Format 1**（`PWM_DMA_SMP_FMT_1`）—— 一個 element 一個週期，**週期長度自己帶**

```
 bit  31            16 15    14           0
     ┌────────────────┬──┬────────────────┐
     │   cnt_end      │ph│      thd       │
     └────────────────┴──┴────────────────┘
```
```c
#define PWM_FILL_SAMPLE_DATA_MODE1(phase, thd, cnt_end)  ((cnt_end << 16) | (phase << 15) | (thd))
```

- `cnt_end` [31:16]：這個週期的計數上限 → **每個 element 可以有不同的週期**（可變頻率）
- `ph` [15]：極性
- `thd` [14:0]：高電位（或低電位）的計數值

**Format 0**（`PWM_DMA_SMP_FMT_0`）—— 一個 element 兩個門檻，週期由暫存器 `count_end_val` 統一決定

```
 bit  31 30           16 15 14            0
     ┌──┬──────────────┬──┬───────────────┐
     │p2│     thd2     │p1│     thd1      │
     └──┴──────────────┴──┴───────────────┘
```
```c
#define PWM_FILL_SAMPLE_DATA_MODE0(val0, val1, val2)  ((val0 << 31) | (val2 << 16) | (val0 << 15) | (val1))
```

⚠ 這個巨集有問題：`val0` 同時塞進 bit31 和 bit15（兩個 phase 綁在一起、不能分別設），而參數名 `val0/val1/val2` 完全看不出對應關係。註解說 `Mode0: val1=THD1, val2=THD2`，但 phase 沒有獨立參數。→ §6

`phase`：
- `PWM_PHASE_POSITIVE`（0）= 先高 `thd` 個 count，再低
- `PWM_PHASE_NEGATIVE`（1）= 先低，再高 ← 相位切割調光要的是這個

### 1.5 Sequence Controller（這是 PWM 模組真正的價值）

每個 PWM 模組有**兩條序列**：**R-SEQ** 和 **T-SEQ**，各自有獨立的 xDMA（`pwm_rdma0_*` / `pwm_rdma1_*`）。

```
      記憶體裡的 element 陣列
      ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
R-SEQ │ e0│ e1│ e2│ e3│ e4│ e5│ e6│ e7│ e8│ e9│  element_num = 10
      └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
        每個 element 播 repeat_num 次
        整串播完等 delay_num
        然後照 seq_order 跳到 T-SEQ 或重播
```

每條序列三個參數（`pwm_seq_para_t`）：

| 欄位 | 意義 |
| --- | --- |
| `rdma_addr` | element 陣列在記憶體的起始位址 |
| `element_num` | 這串有幾個 element |
| `repeat_num` | **每個** element 重複播幾次（拉長每一階的時間） |
| `delay_num` | 整串播完之後等多久（24-bit） |

搭配的整體參數（`pwm_config_t`）：

| 欄位 | 意義 |
| --- | --- |
| `seq_num` | `PWM_SEQ_NUM_1` = 只用一條／`PWM_SEQ_NUM_2` = 兩條都用 |
| `seq_order_1st` / `seq_order_2nd` | 先播 R 還是先播 T（`PWM_SEQ_ORDER_R` / `_T`） |
| `seq_mode` | `NONCONTINUOUS` = 播完停／`CONTINUOUS` = 播完接著再來 |
| `play_cnt` | 整體播幾輪，**0 = 無限** |
| `triggered_src` | 誰來啟動這個 PWM，見下 |
| `counter_mode` | 目前 header 只定義了 `PWM_COUNTER_MODE_UP` |
| `invert` | 輸出波形反相 |

**這是為 LED 效果設計的**：把「亮度 0 → 100 → 0」寫成一串 element，設 `CONTINUOUS` + `play_cnt = 0`，CPU 設完就可以完全不管，硬體自己播呼吸燈。這是 timer PWM 做不到的事。

### 1.6 Trigger（`cfg_pwm_ena_trig`）

`cfgX_pwm_ena_trig` [14:12]，reset 值 3'd7：

| 值 | 意義 |
| --- | --- |
| 0 ~ 4 | 「當 PWM0 ~ PWM4 的 sequence 播完時」觸發本通道 |
| 其他（含 reset 的 7） | 自我觸發（`PWM_TRIGGER_SRC_SELF`） |

所以 `pwm_trigger_src_t` 的來源**只有 PWM0~PWM4 和自己**，沒有外部腳、沒有 timer。這是為**多通道接力**設計的：PWM0 播完換 PWM1、PWM1 播完換 PWM2 …… 做跑馬燈或多色漸變。

⚠ 註記說「若 `cfgX_seqx_cnt` 是 65535，則在 PWMX 第一個 sequence 播完時觸發」。
⚠ 觸發時機是**sequence 播完**，不是每個 PWM 週期。
⚠ `triggered_src` 只有 `pwm_multi_init()` 走得到，4 個 example 全都填 `SELF` → 這條路徑**完全沒有 example 覆蓋、沒人驗過**。

### 1.7 PWM 的「gate function」是什麼（不是外部閘門）

常被誤解。相關兩個位元：

- `cfgX_ck_ena`：「Enable pwm **gated clock**」
- `cfgX_pwm_ck_free`：「0 = Auto gated／1 = Free run」

這是**時脈閘控，為省電而存在** —— 沒事的時候把 PWM 的時脈關掉。**不是**可以拿外部訊號來閘住輸出的功能。客戶問「PWM 的 gate function 能不能同步外部訊號」，答案是不能，理由就在這裡。

### 1.8 中斷

每個 PWM 模組一個 IRQ：`Pwm0_IRQn` ~ `Pwm4_IRQn` = **32 ~ 36**（[mcu.h:87-91](components/platform/soc/rt584/rt584_driver/Inc/mcu.h#L87-L91)）。

`pwm_int_t` 7 個來源：

| 位元 | 名稱 | 意義 |
| --- | --- | --- |
| 0 | `rdma0_int` | R-SEQ 的 DMA 搬完 |
| 1 | `rdma0_err_int` | R-SEQ 的 DMA 出錯 |
| 2 | `rdma1_int` | T-SEQ 的 DMA 搬完 |
| 3 | `rdma1_err_int` | T-SEQ 的 DMA 出錯 |
| 4 | `rseq_done_int` | R-SEQ 播完 |
| 5 | `tseq_done_int` | T-SEQ 播完 |
| 6 | `rtseq_done_int` | R+T 兩條都播完 |

### 1.9 低功耗

`lpm_enable_timer_pwm()` 會把 `peri3_off` 設成 `false`，讓 PWM / timer 在 **sleep** 中活著（[lpm.c](components/platform/soc/rt584/rt584_driver/Src/lpm.c)）。`hosal_timer_pwm_start()` 內部就會自己呼叫它。

**deep sleep 會把 PERI1/2/3 全部斷電**，PWM 一定停。而唯一能在 deep sleep 存活的 TIMER32K0/1 **既沒有 capture 也沒有 PWM**，所以「深睡中還要維持 PWM 輸出」在 RT584 上做不到。

---

## 2. 暫存器層在做什麼

每個 PWM 模組一塊獨立的暫存器區（[pwm_reg.h:222-261](components/platform/soc/rt584/rt584_driver/Inc/pwm_reg.h#L222-L261)）：

| offset | 名稱 | 用途 |
| --- | --- | --- |
| 0x00 | `pwm_ctl0` | enable / clock enable / reset / RDMA enable / RDMA reset |
| 0x04 | `pwm_ctl1` | 中斷清除、中斷遮罩 |
| **0x08** | **`pwm_set0`** | **主控制字**：`pwm_set_t` 那堆 bitfield（seq order / seq 數 / seq mode / dma 格式 / counter mode / cnt trig / dma auto / `ck_div` / `ena_trig`） |
| 0x0C ~ 0x28 | `pwm_set1` ~ `pwm_set8` | count end value、play count、R/T-SEQ 的 repeat / delay 數 |
| 0x40 ~ 0x5C | `pwm_rdma0_*` | **R-SEQ 的 xDMA**：位址、長度、狀態（`_r0`/`_r1` 唯讀） |
| 0x60 ~ 0x7C | `pwm_rdma1_*` | **T-SEQ 的 xDMA**，同上 |

控制位元的巨集（[pwm.h:144-148](components/platform/soc/rt584/rt584_driver/Inc/pwm.h#L144-L148)）：

```c
#define PWM_ENABLE_PWM  (0x01UL << PWM_CFG0_PWM_ENA_SHFT)
#define PWM_ENABLE_CLK  (0x01UL << PWM_CFG0_CK_ENA_SHFT)
#define PWM_RESET       (0x01UL << PWM_CFG0_PWM_RST_SHFT)
#define PWM_RDMA_ENABLE (0x01UL << PWM_CFG0_PWM_RDMA0_CTL0_SHFT)
#define PWM_RDMA_RESET  (0x01UL << PWM_CFG0_PWM_RDMA0_CTL1_SHFT)
```

**Timer 側**只多兩個東西（見 [timer.md §1.4](timer.md)）：
- `timer->cap_en.bit.timer_pwm_en` = 1 開啟 PWM 輸出
- `timer->thd`（門檻）+ `timer->pha.bit.pha`（起始相位）

Timer PWM 一共只有這三個旋鈕加上原本的 `timeload` / `timeout`，比 PWM 模組簡單非常多。

---

## 3. API 怎麼用

### 3.1 PWM 模組 —— HOSAL 層

單通道最短路徑（4 行）：

```c
hosal_pwm_dev_t pwm_dev;
pwm_dev.config.id        = HOSAL_PWM_ID_0;
pwm_dev.config.frequency = 16000;
pwm_dev.config.pin_out   = 20;
hosal_pwm_init_fmt1(&pwm_dev);
hosal_pwm_fmt1_duty(pwm_dev.config.id, 50);   /* 內部會自己 pwm_start() */
```

| API | 用途 |
| --- | --- |
| `hosal_pwm_pin_conifg(id, pin)` | 設輸出腳（名字拼錯，sic） |
| `hosal_pwm_init_fmt1(dev)` / `_fmt0(dev)` | 單通道初始化，格式 1 / 格式 0 |
| `hosal_pwm_fmt1_duty(id, duty)` / `_fmt0_duty` | 設 duty（0~100%），**內部會呼叫 `pwm_start()`** |
| `hosal_pwm_fmt1_count(id, count)` / `_fmt0_count` | 直接給 count 值而非百分比，**內部也會 `pwm_start()`** |
| `hosal_pwm_multi_init(dev)` | 多序列初始化（唯一能設 `triggered_src` 的路徑） |
| `hosal_pwm_multi_fmt1_duty(id, dev, element, duty)` | 填第 `element` 個 element 的 duty |
| `hosal_pwm_multi_fmt0_duty(id, dev, element, thd1, thd2)` | 格式 0 版，兩個門檻 |
| `hosal_pwm_multi_fmt1_count` / `_fmt0_count` | 同上但給 count |
| `hosal_pwm_start(id)` / `hosal_pwm_stop(id)` | 啟動 / 停止（⚠ `stop` 會把 element 清 0，見 §6 P4） |
| `hosal_pwm_ioctl(dev, ctl, arg)` | 其他所有設定，見下表 |

### 3.2 `hosal_pwm_ioctl` 的 26 個命令 —— 只有 16 個真的有實作

| # | 命令 | 有實作? |
| --- | --- | --- |
| 1 | `HOSAL_PWM_SET_COUNT_MODE` | ✅ |
| 2 | `HOSAL_PWM_SET_FRQUENCY` | ✅ |
| 3 | `HOSAL_PWM_SET_DELAY_NUMBER` | ✅ |
| 4 | `HOSAL_PWM_SET_REPEAT_NUMBER` | ✅ |
| 5 | `HOSAL_PWM_SET_PLAY_NUMBER` | ❌ **沒有 case** |
| 6 | `HOSAL_PWM_SET_TSEQ_ADDRESS` | ❌ **沒有 case** |
| 7 | `HOSAL_PWM_SET_RSEQ_ADDRESS` | ❌ **沒有 case** |
| 8 | `HOSAL_PWM_GET_COUNT_MODE` | ❌ **沒有 case** |
| 9 | `HOSAL_PWM_GET_FRQUENCY` | ✅（但 driver getter 壞的，P1） |
| 10 | `HOSAL_PWM_GET_DUTY` | ❌ **沒有 case** |
| 11 | `HOSAL_PWM_GET_COUNT` | ✅（同 P1） |
| 12 | `HOSAL_PWM_GET_DELAY_NUMBER` | ✅（同 P1） |
| 13 | `HOSAL_PWM_GET_REPEAT_NUMBER` | ✅（同 P1） |
| 14 | `HOSAL_PWM_GET_PLAY_NUMBER` | ❌ **沒有 case** |
| 15 | `HOSAL_PWM_GET_TSEQ_ADDRESS` | ❌ **沒有 case** |
| 16 | `HOSAL_PWM_GET_RSEQ_ADDRESS` | ❌ **沒有 case** |
| 17 | `HOSAL_PWM_SET_CLOCK_DIVIDER` | ✅ |
| 18 | `HOSAL_PWM_GET_PHASE` | ✅（同 P1） |
| 19 | `HOSAL_PWM_SET_PHASE` | ✅ |
| 20 | `HOSAL_PWM_SET_COUNT_END_VALUE` | ✅ |
| 21 | `HOSAL_PWM_SET_DMA_FORMAT` | ✅ |
| 22 | `HOSAL_PWM_GET_INVERT` | ✅（同 P1） |
| 23 | `HOSAL_PWM_SET_INVERT` | ✅ |
| 24 | `HOSAL_PWM_DISABLE_INTERRUPT` | ❌ **沒有 case** |
| 25 | `HOSAL_PWM_ENABLE_INTERRUPT` | ❌ **沒有 case** |
| 26 | `HOSAL_PWM_REGISTER_CALLBACK` | ✅ |

→ **10 個宣告了但沒實作**，呼叫會靜默失敗（不是回錯誤碼，是 switch 走到底什麼都不做）。§6 P7。

### 3.3 PWM 模組 —— driver 層

| API | 備註 |
| --- | --- |
| `pwm_init_fmt1(id, freq)` / `pwm_init_fmt0(id, freq, cnt_end)` | ⚠ 內部硬寫 `PWM_CLK_DIV_1`（P8） |
| `pwm_set_frequency(id, freq)` / `pwm_get_frequency(id, *out)` | getter 壞（P1） |
| `pwm_clock_divider(id, div)` | ⚠ 先解參考再檢查 id（P3） |
| `pwm_set_pahse` / `pwm_get_pahse`（sic，拼錯） | getter 壞 |
| `pwm_get_count` / `pwm_set_counter_mode` / `pwm_set_counter_end_value` | |
| `pwm_set_dma_format(id, fmt)` | |
| `pwm_fmt1_duty` / `pwm_fmt0_duty` / `pwm_fmt1_count` / `pwm_fmt0_count` | 全部內部呼叫 `pwm_start()` |
| `pwm_multi_init(cfg, freq)` | 唯一能設 `triggered_src` 的入口 |
| `pwm_multi_fmt1_duty` / `_fmt0_duty` / `_fmt1_count` / `_fmt0_count` | |
| `pwm_set_repeat_number` / `pwm_get_repeat_number` | getter 壞 |
| `pwm_set_delay_number` / `pwm_get_delay_number` | ⚠ setter 也有 bug（P2）；getter 壞 |
| `pwm_set_dma_element` / `pwm_get_dma_element` | **`pwm_get_dma_element` 是唯一寫對的 getter** |
| `pwm_set_inveter` / `pwm_get_inveter`（sic） | getter 壞 |
| `pwm_start(id)` / `pwm_stop(id)` | ⚠ `stop` 會清掉使用者的 buffer（P4） |
| `pwm_register_callback_function(id, cb)` | |

### 3.4 Timer PWM —— HOSAL 層

只有 4 支：

```c
hosal_timer_pwm_config_mode_t cfg;
cfg.counting_mode = HOSAL_TIMER_UP_COUNTING;
cfg.int_en        = HOSAL_TIMER_INT_ENABLE;
cfg.mode          = HOSAL_TIMER_PERIODIC_MODE;
cfg.oneshot_mode  = HOSAL_TIMER_ONE_SHOT_DISABLE;
cfg.prescale      = HOSAL_TIMER_PRESCALE_16;
cfg.user_prescale = 0;
cfg.pwm0_enable   = 1;              /* 這五個決定搶哪幾個輸出通道 */
cfg.pwm1_enable   = 0;
cfg.pwm2_enable   = 0;
cfg.pwm3_enable   = 0;
cfg.pwm4_enable   = 0;
hosal_timer_pwm_open(RT_TIMER0, cfg);

hosal_timer_pwm_config_tick_t tick;
tick.timeload_ticks = 0;
tick.timeout_ticks  = 2000;         /* 週期 */
tick.threshold      = 1000;         /* 門檻 → duty 50% */
tick.phase          = 0;
hosal_timer_pwm_start(RT_TIMER0, tick);
```

| API | 備註 |
| --- | --- |
| `hosal_timer_pwm_open(id, cfg)` | ⚠ 只有 `state == CLOSED` 才成功；會強制把時脈源設成 PMU |
| `hosal_timer_pwm_start(id, tick)` | **可重複呼叫改頻率／duty，不用先 stop**（`state` 還是 OPEN 就直接寫暫存器）；內部會 `lpm_enable_timer_pwm()` |
| `hosal_timer_pwm_stop(id)` | 清 `timer_pwm_en` + `en` |
| `hosal_timer_pwm_close(id)` | ⚠ **不會把 `soc_pwm_sel` 還原**，見 §6 P12 |

頻率算法：`輸出頻率 = 32MHz / prescale / timeout_ticks`，duty = `threshold / timeout_ticks`。

---

## 4. 逐 example

### 4.1 `pwm_fmt0_duty`

| 項目 | 內容 |
| --- | --- |
| 路徑 | [examples/peripheral/pwm/pwm_fmt0_duty](examples/peripheral/pwm/pwm_fmt0_duty) |
| 行數 | 59 |
| 設定 | PWM0、16 kHz、`count_end_val = 3000`、GPIO20、格式 0 |
| 做什麼 | `hosal_dma_init()` → `hosal_pwm_init_fmt0()` → 註冊 callback → `hosal_pwm_fmt0_duty(id, 50)` → `while(1)` |
| 判定條件 | **待實測** |
| 觀察 | 沒有呼叫 `hosal_pwm_start()`（靠 `fmt0_duty` 內部啟動）；`status` 變數宣告了沒用到 |

### 4.2 `pwm_fmt1_duty`

| 項目 | 內容 |
| --- | --- |
| 路徑 | [examples/peripheral/pwm/pwm_fmt1_duty](examples/peripheral/pwm/pwm_fmt1_duty) |
| 行數 | 53 |
| 設定 | PWM0、16 kHz、GPIO20、格式 1（`count_end_val` 不用管） |
| 做什麼 | 同上，最後多呼叫一次 `hosal_pwm_start()` |
| 判定條件 | **待實測** |
| 觀察 | `hosal_pwm_fmt1_duty()` 內部已經 `pwm_start()` 了，外面又 `hosal_pwm_start()` 一次 → 重複 |

### 4.3 `pwm_multi_fmt0_duty`

| 項目 | 內容 |
| --- | --- |
| 路徑 | [examples/peripheral/pwm/pwm_multi_fmt0_duty](examples/peripheral/pwm/pwm_multi_fmt0_duty) |
| 行數 | 102 |
| 設定 | PWM0、R-SEQ + T-SEQ 各 10 個 element、`SEQ_NUM_2`、`CONTINUOUS`、`play_cnt = 0`（無限）、`CLK_DIV_1`、`TRIGGER_SRC_SELF`、`count_end_val = 3000`、GPIO20 |
| 做什麼 | `hosal_pwm_multi_init()` → 兩個 for 迴圈各填 10 個 element（duty 從 10 每次 +5 / 從 30 每次 +5）→ `hosal_pwm_start()` |
| 判定條件 | **待實測** |
| 觀察 | ⚠ printf 印 `dma_smp_fmt : HOSAL_PWM_DMA_SMP_FMT_1`，程式碼實際設 `FMT_0` → **log 跟程式不一致** |

### 4.4 `pwm_multi_fmt1_duty`

| 項目 | 內容 |
| --- | --- |
| 路徑 | [examples/peripheral/pwm/pwm_multi_fmt1_duty](examples/peripheral/pwm/pwm_multi_fmt1_duty) |
| 行數 | 104 |
| 設定 | 同 4.3 但格式 1、`count_end_val = 0`（格式 1 不用）、`seq_order_1st = T`、`2nd = R` |
| 做什麼 | 同 4.3 |
| 判定條件 | **待實測** |

### 4.5 `timer-pwm`（584 only）

| 項目 | 內容 |
| --- | --- |
| 路徑 | [examples/peripheral/timer/timer-pwm](examples/peripheral/timer/timer-pwm) |
| 設定 | TIMER0→PWM0(GPIO4)；TIMER1→PWM1(GPIO5)+PWM2(GPIO20)；TIMER2→PWM3(GPIO21)+PWM4(GPIO30)。TIMER0/1 prescale 16、TIMER2 prescale 32，全部 `timeout=2000` / `thd=1000`（duty 50%） |
| 做什麼 | 設完三顆 timer 就進 sleep 無限迴圈 |
| 判定條件 | **待實測** |
| 觀察 | ⚠ PWM0 打在 **GPIO4 = EVB 的 KEY4**（見 [lpm/sleep/main.c:20-27](examples/peripheral/lpm/sleep/sleep/main.c#L20-L27)），量測要外接；⚠ 設完就睡，沒示範任何執行中的參數變更 |

### 4.6 `timer-pwm-button`（584 only，新增）

| 項目 | 內容 |
| --- | --- |
| 路徑 | [examples/peripheral/timer/timer-pwm-button](examples/peripheral/timer/timer-pwm-button) |
| 設定 | TIMER0→PWM0(**GPIO20 = EVB LED**)、prescale 16（tick = 2 MHz / 0.5 µs）；按鈕 GPIO0（KEY0）rising edge + 100K 上拉 + debounce 1024 slow clocks |
| 做什麼 | 每次按鈕在 100 / 500 / 1k / 2k / 5k / 10k Hz 之間循環，duty 固定 50%。中斷只設 flag，主 task 每 20 ms 處理後才 `hosal_timer_pwm_start()` 並印 log |
| 判定條件 | **待實測** |
| 用途 | 示範**執行中由 GPIO 中斷改 PWM 參數** —— 客戶 AC 調光案要的就是這條路徑（把按鈕換成過零脈衝） |

---

## 5. 驗證表對照

`RT584_MPA_IC_peripheral_verification_status_20241008.xlsx` → sheet **PWM**。依 [coverage matrix](RT584_peripheral_coverage_matrix.md) 記錄：

| 測項 | 對應 example | 狀態 |
| --- | --- | --- |
| PWM Mode | pwm_fmt1_duty / pwm_multi_fmt1_duty | 有 example |
| Register Mode | pwm_fmt0_duty / pwm_multi_fmt0_duty | 有 example |
| Trigger | pwm_multi_fmt0_duty / pwm_multi_fmt1_duty | ⚠ 但兩個 example 都填 `SELF`，**PWM→PWM 接力沒被覆蓋** |
| xDMA | 4 個 example 都會用到 | 沒有專屬驗證 |
| Clock Divider | — | ❌ **無 example**（4 個都用 `CLK_DIV_1`） |
| Interrupt | — | ❌ **無專屬 example**（callback 只 printf） |
| Sequence Controller Mode | — | ❌ **無專屬 example** |

Timer 側的 PWM 測項見 [timer.md §5](timer.md) 第 8 項（對應 `timer-pwm`）。

**缺口總結**

- **4 個 example 在功能上高度重複**：都是「init → 設固定 duty → start → `while(1)`」。
- 完全沒有 example 動過：`ck_div`、`invert`、`phase = NEGATIVE`、`repeat_num` / `delay_num`（都填 0）、`play_cnt` ≠ 0、`NONCONTINUOUS`、`SEQ_NUM_1`、`triggered_src` ≠ SELF、任何中斷實際做事。
- 完全沒有 example 示範：**執行中改頻率或改 duty**（`timer-pwm-button` 是第一個，且走 timer 而非 PWM 模組）。
- `PWM_FILL_SAMPLE_DATA_MODE1` / `MODE0` 這兩個巨集**在任何 example 裡都沒出現過**，等於 element 手動打包完全沒示範。

---

## 6. Code Review findings

### 🔴 P1 —— 6 個 getter 全壞，13 個呼叫點

```c
uint32_t pwm_get_frequency(uint32_t id, uint32_t* get_frequency) {
    ...
    *(uint32_t*)&get_frequency = m_pwm_handle[id].frequency;   /* ← 寫到參數本身，不是寫到 caller */
```

`&get_frequency` 是那個**指標變數自己的位址**（在 stack 上），不是它指向的地方。等於改了一個馬上要消失的局部變數，caller 拿到的永遠是未初始化的值。

受影響：`pwm_get_frequency`:248、`pwm_get_pahse`:305、`pwm_get_count`:321、`pwm_get_repeat_number`:825/827/831/833、`pwm_get_delay_number`:881/883/887/889、`pwm_get_inveter`:907。

正確寫法是 `*get_frequency = ...`。`pwm_get_dma_element` 是唯一寫對的。

### 🔴 P2 —— `pwm_set_delay_number` 寫錯暫存器

```c
} else if (pwm_set_cfg.bit.cfg_seq_two_sel == PWM_SEQ_NUM_1) {
    if (... == PWM_SEQ_ORDER_R)      { pwm->pwm_set5 = dly_number; }
    else if (... == PWM_SEQ_ORDER_T) { pwm->pwm_set0 = dly_number; }   /* 應該是 pwm_set8 */
```

`pwm_set0` 是**主控制字**。這一行會把整個 PWM 設定（seq 模式、dma 格式、`ck_div`、`ena_trig`……）覆蓋成一個 delay 數值。不只是設定沒生效，是把 PWM 設壞。

### 🔴 P3 —— `pwm_clock_divider` 先解參考才檢查參數

```c
uint32_t pwm_clock_divider(pwm_id_t id, pwm_clk_div_t pwm_clk_div) {
    pwm_t* pwm = m_pwm_handle[id].pwm;              /* ← 越界讀 */
    pwm_set_cfg = *(pwm_set_t*)&pwm->pwm_set0;      /* ← 越界解參考 */
    if (id > PWM_ID_MAX) { return STATUS_INVALID_PARAM; }
```

兩個問題：檢查在使用之後；而且 `>` 應該是 `>=`（`PWM_ID_MAX` = 5 本身就已經越界，`m_pwm_handle[5]` 超出 `[5]` 陣列）。

### 🟡 P4 —— `pwm_stop()` 會清掉使用者的 buffer

`pwm_stop()` 把使用者傳進來的 R-SEQ / T-SEQ element 陣列**內容清成 0**，而且在停止路徑裡還做了 `|= PWM_RDMA_ENABLE`。後果：`stop` 之後不能單純 `start` 回去，一定要重填 element。

這也是 [pwm-zerocross-dimmer.md](pwm-zerocross-dimmer.md) §3.4 那段程式碼為什麼 `hosal_pwm_stop()` 後面一定要接 `hosal_pwm_fmt1_count()` 的原因。

### 🟡 P5 —— `PWM_FILL_SAMPLE_DATA_MODE0` 巨集把兩個 phase 綁在一起

```c
#define PWM_FILL_SAMPLE_DATA_MODE0(val0,val1,val2)  ((val0 << 31) | (val2 << 16) | (val0 << 15) | (val1))
```

`val0` 同時填 bit31（phase2）和 bit15（phase1），無法分別設定；參數名 `val0/val1/val2` 也完全看不出語意。且所有參數都沒加括號，傳入運算式會出錯。

### 🟡 P6 —— 命名拼錯，已經進了公開 API

`pwm_set_pahse` / `pwm_get_pahse`（phase）、`pwm_set_inveter` / `pwm_get_inveter`（inverter）、`hosal_pwm_pin_conifg`（config）、`pwm_init_fmt1(uint32_t id, uint32_t freqency)`（frequency）、`HOSAL_PWM_SET_FRQUENCY`（frequency）。改動會破壞相容性，但至少該加正確拼字的 alias。

### 🟡 P7 —— `hosal_pwm_ioctl` 26 個命令只實作 16 個

見 §3.2 表。缺的 10 個呼叫後**靜默什麼都不做**，也不回錯誤碼。至少 `default:` 要回 `STATUS_INVALID_PARAM`。

### 🟡 P8 —— `pwm_init_fmt0/1` 硬寫 `PWM_CLK_DIV_1`

初始化把 `ck_div` 寫死成 ÷1，`pwm_config_t.clk_div` 在單通道路徑上被忽略。要改分頻必須 init 之後再呼叫一次 `pwm_clock_divider()` 或 `HOSAL_PWM_SET_CLOCK_DIVIDER`。這一點在任何文件或 example 裡都沒講。

### 🟡 P9 —— `pwm_set_frequency` 在格式 0 + POSITIVE 時把 `full_count` 設成 0

```c
if (m_pwm_handle[id].phase == PWM_PHASE_POSITIVE) {
    if (pwm_set_cfg.bit.cfg_pwm_dma_fmt == PWM_DMA_SMP_FMT_0) {
        m_pwm_handle[id].full_count = 0;
    } else { m_pwm_handle[id].full_count -= 1; }
}
```

`full_count = 0` 之後 `pwm_fmt0_duty()` 算 `duty * full_count / 100` 一律得 0。看起來是想寫別的東西寫錯了 —— 需要對 RM 確認格式 0 下 `full_count` 到底該是什麼。

### 🟡 P10 —— `pwm_fmt*_duty` / `pwm_fmt*_count` 內部偷偷 `pwm_start()`

「設 duty」和「啟動」被綁在一起，呼叫者無法只改 duty 而不重啟。也導致 `pwm_fmt1_duty` example 裡出現重複的 `hosal_pwm_start()`。

### 🟡 P11 —— `ck_div_val` [31:24] 從沒宣告，目前能運作是巧合

`pwm_set_t` 的 [31:24] 落在 `RVD4:12` 裡，`ck_div_val` 完全沒有宣告，SDK 無法使用 8-bit 精細分頻。而 `ck_div` 之所以有效，是因為 `pwm_init_fmt*` 整個 word 寫下去、順手把 `ck_div_val` 清成 0。如果哪天有人改成 read-modify-write 或有其他程式碼碰過 [31:24]，`ck_div` 會靜默失效。

### 🟡 P12 —— `timer_pwm_close()` 不還原 `soc_pwm_sel`（新增）

`timer.c` 是**整個 repo 裡唯一寫 `SYSCTRL->soc_pwm_sel` 的地方**：

```c
if (cfg.pwm0_enable) { SYSCTRL->soc_pwm_sel.bit.pwm0_src_sel = timer_id + 1; }
```

但 [`timer_pwm_close()`](components/platform/soc/rt584/rt584_driver/Src/timer.c#L538-L553) 只清 `timer->control.reg`，**沒有把 mux 設回 0**。後果：一旦用過 timer PWM，那個輸出通道就永久指向那顆 timer，之後改用 PWM 模組會**完全沒有輸出而且沒有任何錯誤提示**，除非重開機。

同理 `pwm.c` 從來不設 `soc_pwm_sel` = 0，所以 PWM 模組也不會把通道搶回來。

**建議**：`timer_pwm_close()` 把用過的通道還原成 0；或 `pwm_init_*` 主動把自己那個通道設成 0。

### 🟢 P13 —— example 的 log 跟程式不一致

`pwm_multi_fmt0_duty` 印 `HOSAL_PWM_DMA_SMP_FMT_1` 但設 `FMT_0`。`pwm_fmt0_duty` 的 `status` 變數宣告未使用。`pwm_fmt1_duty` 的註解 `//using format 1 the count end value don't care` 被複製到 `pwm_fmt0_duty` 的 frequency 那行，位置錯了。

---

## 7. 待確認 / 待實測清單

| # | 項目 | 為什麼要確認 |
| --- | --- | --- |
| V1 | `soc_pwm_sel` 值 **0 是否等於「PWM 模組」** | §1.1 是推論；決定 §6 P12 的修法 |
| V2 | 同一輸出通道被 timer 搶走後，PWM 模組是否真的完全無輸出 | 驗證 P12 的實際後果 |
| V3 | `soc_pwm_sel` 的 reset 值 | 若非 0 則 §1.1 整段要改 |
| V4 | PWM 模組實際最低 / 最高頻率 | `32MHz ÷ 256 ÷ 32767 ≈ 3.8 Hz` 是算的，沒量過 |
| V5 | `SystemCoreClock` 改變時 PWM 頻率是否真的跟著跑掉 | 影響所有換 CPU 頻率的專案 |
| V6 | `ck_div_val` [31:24] 是否真的存在且可用 | 若可用則值得補進 `pwm_set_t` |
| V7 | `cfg_pwm_cnt_trig` [5]「register mode 要設 1」到底做什麼 | repo 裡沒有任何程式碼寫過這個位元 |
| V8 | 單一 element 是否算「一個 sequence」（影響 `ena_trig` 觸發時機） | 決定 PWM→PWM 接力能不能做逐週期的事 |
| V9 | `ena_trig` 接力的傳遞延遲有多少 | 沒有任何文件或 example |
| V10 | `seq_dly` [23:0] 的單位是 PWM tick 還是 element | 相位延遲的替代方案 |
| V11 | 環狀接力（PWM0↔PWM1 互相觸發）能不能自己啟動、會不會死鎖 | 高價值但完全未驗證 |
| V12 | `repeat_num` / `delay_num` 的實際行為 | 4 個 example 全填 0 |
| V13 | 7 個中斷來源各自的觸發時機 | callback 只 printf，沒人驗過哪個位元會亮 |
| V14 | sleep 中 PWM 是否真的持續輸出 | `lpm_enable_timer_pwm()` 的效果沒量過 |

> 以上全部 **待實測**。實驗條件限制：目前只能用 GPIO 跳線當刺激，沒有調光電路、沒有市電過零訊號。

---

## 附錄：本家族的 CI 覆蓋

`.github/config/{dev,qa}/<chip>/basic.json`：

| example | RT584H | RT584HA4 | RT584L | RF1301 | RT581/2/3 |
| --- | --- | --- | --- | --- | --- |
| pwm_fmt0_duty | ✅ | ✅ | ✅ | ✅ | ✅ |
| pwm_fmt1_duty | ✅ | ✅ | ✅ | ✅ | ✅ |
| pwm_multi_fmt0_duty | ✅ | ✅ | ✅ | ✅ | ✅ |
| pwm_multi_fmt1_duty | ✅ | ✅ | ✅ | ✅ | ✅ |
| timer-pwm | ✅ | ✅ | ✅ | ✅ | ❌ 584 only |
| timer-pwm-button | ✅ | ✅ | ✅ | ✅ | ❌ 584 only |

（PWM 模組的 4 個 example 在 coverage matrix 標記為 `58x+584`，實際 config 檔清單請以各 example 目錄下的 `default-*.config` 為準。）
