# Timer（RT584 / RT58x）周邊文件

| 項目 | 內容 |
| --- | --- |
| 周邊 | Timer（TIMER0 / TIMER1 / TIMER2，3 顆 32-bit fast timer） |
| RM 章節 | **Ch13 Timer**（Block Diagram / Functional Description: Pre-scaler、Interrupt、Operational Modes、PWM Function、Input Capture / Registers） |
| 驗證表 sheet | `RT584_MPA_IC_peripheral_verification_status_20241008.xlsx` → **Timer**（8 個測項，皆 Pass） |
| Examples | `examples/peripheral/timer/` 共 **7 個 project** |
| Driver | [timer.c](components/platform/soc/rt584/rt584_driver/Src/timer.c) / [timer.h](components/platform/soc/rt584/rt584_driver/Inc/timer.h) |
| HOSAL | [hosal_timer.c](components/platform/hosal/rt584_hosal/Src/hosal_timer.c) / [hosal_timer.h](components/platform/hosal/rt584_hosal/Inc/hosal_timer.h) |
| 相關但獨立的家族 | `examples/peripheral/slow-timer/`（SLOWTIMER0/1，32kHz）、`examples/peripheral/pwm/`（獨立 PWM 周邊，非 timer PWM）→ 見 [pwm.md](pwm.md) |

> ⚠ **「判定條件」一律留 `待實測`** — 未實際上機跑過的東西不推測。實測後再回填本文件。
> ⚠ RM 章號由 docx Heading-1 出現順序推算（Introduction = Ch1），章名可信、章號請對一次實體 TOC。

本文件是 **22 個周邊家族文件的格式原型**。後續每個家族沿用同一節次結構（§1 原理 → §2 暫存器 → §3 API → §4 逐 example → §5 驗證表對照 → §6 code review → §7 待確認）。

---

## 1. 原理（RM Ch13 摘要）

Timer 是 **32-bit 上/下數計數器**，前面串一級 **10-bit prescaler**（分頻 1~1024）。

```
 timer clock ──► [prescaler]──► timerTick ──► [32-bit counter] ──► 比較 expired_value ──► intReq
   (PERI /                          │                                                      │
    RCO1M /                         └──► cap_value（輸入邊緣鎖存）                          └──► pwm_o（value_thd / pha）
    PMU)
```

四個功能區塊：

### 1.1 Pre-scaler
- `control.prescale`（3-bit）提供 8 種常用分頻；`prescale`（獨立的 10-bit 暫存器，driver 叫 `user_prescale`）提供任意分頻。
- **driver 的規則是二選一**：`user_prescale != 0` 時寫 10-bit 暫存器、並把 `control.prescale` 留 0；`user_prescale == 0` 時才用 `control.prescale`（[timer.c:95-100](components/platform/soc/rt584/rt584_driver/Src/timer.c#L95-L100)）。
- **`TIMER_PRESCALE_*` 的編碼不是遞增的**，務必用 macro 不要填數字：

  | macro | 分頻 | 暫存器值 |
  | --- | --- | --- |
  | `TIMER_PRESCALE_1` | ÷1 | 0 |
  | `TIMER_PRESCALE_16` | ÷16 | 1 |
  | `TIMER_PRESCALE_256` | ÷256 | 2 |
  | `TIMER_PRESCALE_2` | ÷2 | 3 |
  | `TIMER_PRESCALE_8` | ÷8 | 4 |
  | `TIMER_PRESCALE_32` | ÷32 | 5 |
  | `TIMER_PRESCALE_128` | ÷128 | 6 |
  | `TIMER_PRESCALE_1024` | ÷1024 | 7 |

- PERI clock = **32 MHz**。已由 `RT584MPA_register_table.xlsx` 的 **MemoryMap sheet** 直接證實：TIMER0/1/2 標註 "Reference clock **32M**"、TIMER32K0/1 標註 "Reference clock **32K**"；與所有 example 註解反推的結果一致（÷16 → 2 MHz，1000000 ticks = 500 ms ✓；÷32 → 1 MHz，8000000 ticks = 8 s ✓）。
- **仍待實測：這個 32 MHz 是不是固定值、會不會隨 CPU AHB 頻率（16/24/32/36/40/48/64 MHz）變動。**參考點：PWM 周邊的時脈**確定會跟著跑**（`pwm_set_frequency()` 直接用 `SystemCoreClock`），所以不能假設 timer 也是固定的。
- `user_prescale = N` 的實際分頻比由 example 反推是 **÷(N+1)**（N=15 → 5 s ✓、N=159 → 50 s ✓、N=31 → 10 s ✓）。**待實測確認。**

### 1.2 Operational Modes
| mode | 計到 expired_value 時 | 用途 |
| --- | --- | --- |
| Free-running (`mode=0`) | 中斷後**繼續數**（down count 且 expired=0 → wrap 到 0xFFFFFFFF） | 自由計時、取當前值 |
| Periodic (`mode=1`) | 中斷後**重載 load 值** | 週期性 tick |
| One-shot (`oneshot_en=1`) | 中斷後**停在 expired value** | 單次延遲 |

`up_count` 選上/下數。一般用法：
- **down count**：`timeload_ticks` = 起始值、`timeout_ticks` = 0（數到 0 中斷）。
- **up count**：`timeload_ticks` = 0、`timeout_ticks` = 目標值（數到目標中斷）。

### 1.3 Interrupt
`intReq` 在 counter == `timer_expired_value` 時觸發；寫 `clear` 暫存器清除；`int_status` 可讀狀態。目前值由 `value` 暫存器讀出。

### 1.4 PWM Function
定位是**低功耗下的簡單波形產生**（不是全功能 PWM，全功能請看獨立的 `pwm` 周邊）。只有三個旋鈕：
- `timer_pwm_en`：開關
- `value_thd`（threshold）：波形變化的邊界
- `pha`（phase）：**`pha=0` → `value >= value_thd` 時輸出 high；`pha=1` → `value >= value_thd` 時輸出 low**

pwm_o 預設 low，且比 counter 值**晚 1 個 timerTick**。5 條 PWM 輸出（PWM0~PWM4）由 `SYSCTRL->soc_pwm_sel` 決定由哪顆 timer 驅動。

### 1.5 Input Capture
2 個獨立 channel，各自有輸入腳、設定與狀態：
- `chN_capture_io_sel`：選哪支 GPIO 當輸入
- `chN_capture_edge`：正緣 / 負緣
- `chN_deglich_en`：**去彈跳，小於 4 個 timer clock 的脈衝會被濾掉**
- 中斷觸發後讀 `chN_cap_value` 取得鎖存的計數值

---

## 2. 暫存器層在做什麼

以 `timer_open()` 為例（[timer.c:70-111](components/platform/soc/rt584/rt584_driver/Src/timer.c#L70-L111)）：

```c
timer->control.reg = 0;                      /* 先整個關掉 */
do { get_timer_enable_status(id,&en); } while(en);   /* 等 HW 真的停 */
timer->clear = 1;                            /* 清中斷 */
/* prescale 二選一 */
timer->control.bit.up_count   = cfg.counting_mode;
timer->control.bit.one_shot_en= cfg.oneshot_mode;
timer->control.bit.mode       = cfg.mode;
timer->control.bit.int_enable = cfg.int_en;
```

`timer_start()` → `timer_load()` + `control.bit.en = 1`；`timer_load()` 寫 `load = timeload_ticks - 1`、`expried_value = timeout_ticks`，然後**busy-wait 到 `value` 真的等於 `load`**（[timer.c 內](components/platform/soc/rt584/rt584_driver/Src/timer.c)，見 §6 findings D）。

ISR（`timer0_handler` 等）只做兩件事：寫 `TIMERn->clear = 1`、呼叫使用者 callback。

---

## 3. HOSAL API 怎麼用

HOSAL 層**幾乎是 1:1 pass-through**，`HOSAL_TIMER_*` 巨集就是 `TIMER_*` 的別名。

| HOSAL | Driver | 說明 |
| --- | --- | --- |
| `hosal_timer_init(id, cfg, cb)` | `timer_open` | 設定 + 註冊 callback |
| `hosal_timer_start(id, tick_cfg)` | `timer_start` | 載入 ticks 並啟動 |
| `hosal_timer_stop(id)` | `timer_stop` | `en = 0` |
| `hosal_timer_reload(id, tick_cfg)` | `timer_load` | 重載 |
| `hosal_timer_finalize(id)` | `timer_close` | 關閉、狀態回 CLOSED |
| `hosal_timer_current_get(id, &v)` | `timer_current_get` | 讀目前值 |
| `hosal_timer_capture_init/start/stop/finalize` | `timer_capture_*` | Capture |
| `hosal_timer_chN_capture_value_get / _int_status` | 同名 | 讀鎖存值 / 中斷狀態 |
| `hosal_timer_pwm_open/start/stop/close` | `timer_pwm_*` | PWM |

標準流程：

```c
hosal_timer_config_t cfg = {
    .counting_mode = HOSAL_TIMER_DOWN_COUNTING,
    .int_en        = HOSAL_TIMER_INT_ENABLE,
    .mode          = HOSAL_TIMER_PERIODIC_MODE,
    .oneshot_mode  = HOSAL_TIMER_ONE_SHOT_DISABLE,
    .prescale      = HOSAL_TIMER_PRESCALE_32,
    .user_prescale = 0,
};
hosal_timer_tick_config_t tick = { .timeload_ticks = 1000000, .timeout_ticks = 0 };

hosal_timer_init(0, cfg, my_cb);
NVIC_EnableIRQ(Timer0_IRQn);      /* ★ HOSAL 不會幫你開 NVIC，必須自己開 */
hosal_timer_start(0, tick);
```

**注意事項**
- `NVIC_EnableIRQ()` 一定要 app 自己呼叫，`hosal_timer_init()` 不做。
- `cfg` 是 struct by value，未初始化欄位是垃圾值 → 每個欄位都要填（所有 example 都是逐欄位填的，照抄即可）。
- **HOSAL 沒有 slow timer（SLOWTIMER0/1）的包裝**，driver 有 `slowtimer_*` 一整套但 hosal 層缺；`slow-timer` 家族的 example 是直接呼叫 driver。見 §6 finding K。
- HOSAL 沒有 `timer_int_status_get()` 的對應 API。

---

## 4. 逐 example

7 個 project：程式碼在 `examples/peripheral/timer/<name>/<name>/main.c`，但 `default-<CHIP>-EVB.config` 在**上一層** `examples/peripheral/timer/<name>/`，皆支援 7 顆晶片（RT581/582/583/RF1301/RT584H/RT584HA4/RT584L）除 capture / pwm（584 系列 only）。

編譯（**編譯燒錄由使用者執行**）：
```bash
cmake -DCUSTOM_CONFIG_DIR="examples/peripheral/timer/timer-periodic/default-RT584H-EVB.config" -S . -B build_timer -G Ninja
cmake --build build_timer -j16
```

### 4.1 timer-periodic

| | |
| --- | --- |
| 在測什麼 | Periodic 模式 + down counting，三顆 timer 不同週期同時跑 |
| 設定 | T0: ÷16, load 1000000 → 500 ms；T1: ÷32, load 1000000 → 1 s；T2: ÷32, load 4000000 → **4 s（註解寫 8s，錯，見 finding L）** |
| 現象 | 三顆 timer 的 callback 各自週期性 printf `Timer<n>, value:<v>` |
| 接線/儀器 | 只需 UART console |
| 驗證表測項 | Timer / **Periodic**、Interrupt Interval、Status Interval |
| RM | Ch13 §Operational Modes、§Interrupt |
| **判定條件** | **待實測** |

### 4.2 timer-freerun-upcount

| | |
| --- | --- |
| 在測什麼 | Free-running + up counting，`timeload=0` / `timeout=目標值` |
| 設定 | T0: ÷16, timeout 1000000 → 500 ms；T1: ÷32, timeout 1000000 → 1 s；T2: ÷32, timeout 8000000 → 8 s |
| 現象 | 中斷後**繼續數不重載**，callback 印出的 value 會持續累加 |
| 接線/儀器 | 只需 UART console |
| 已知問題 | `timerN_done` 只被設 0、從未設 1 → `main()` 的 finalize 分支是 dead code（finding M） |
| 驗證表測項 | Timer / **Free Run**、Interrupt Interval |
| RM | Ch13 §Operational Modes |
| **判定條件** | **待實測** |

### 4.3 timer-freerun-downcount

| | |
| --- | --- |
| 在測什麼 | Free-running + down counting，`timeload=起始值` / `timeout=0` |
| 設定 | T0: ÷16, load 1000000 → 500 ms；T1: ÷32, load 1000000 → 1 s；T2: ÷32, load 8000000 → 8 s |
| 現象 | 數到 0 中斷後 **wrap 到 0xFFFFFFFF 繼續數**（RM 明文） → callback 印出的 value 會是接近 0xFFFFFFFF 的大數 |
| 接線/儀器 | 只需 UART console |
| 已知問題 | 同 4.2，`timerN_done` 是 dead code |
| 驗證表測項 | Timer / **Free Run** |
| RM | Ch13 §Operational Modes（"wrap around to 0xFFFFFFFF"） |
| **判定條件** | **待實測** |

### 4.4 timer-oneshot

| | |
| --- | --- |
| 在測什麼 | One-shot vs 非 one-shot 的對照（只用 T0/T1） |
| 設定 | T0: ÷16, load 1000000, **ONE_SHOT_ENABLE** → 500 ms 一次；T1: ÷32, load 1000000, **ONE_SHOT_DISABLE** → 1 s 持續 |
| 現象 | T0 只印一次就停在 expired value；T1 持續。兩個 callback 都設 `timerN_done = 1`，`main()` 收到後 `hosal_timer_finalize()` + `NVIC_DisableIRQ()` |
| 接線/儀器 | 只需 UART console |
| 驗證表測項 | **無對應測項** ⚠（Timer sheet 8 項沒有 one-shot） |
| RM | Ch13 §Operational Modes（`oneshot_en`） |
| **判定條件** | **待實測** |

> ⚠ 這個 example 是 §4 「有 example、沒有測項」清單的一員 → 驗證表應補一項 Timer / One-shot。
> ⚠ 另外 fast timer 的 ISR **沒有**在 one-shot 時清 `en`，slow timer 的 ISR 有（finding I）→ 實測時請確認 T0 停下來是 HW 自己停的。

### 4.5 timer-user-prescale

| | |
| --- | --- |
| 在測什麼 | 10-bit `user_prescale` 取代 3-bit `prescale` |
| 設定 | 三顆都 up counting / free-run / timeout 10000000；`user_prescale` = 15 / 159 / 31 → 註解宣稱 5 s / 50 s / 10 s（即 ÷16 / ÷160 / ÷32） |
| 現象 | 三顆不同週期 printf |
| 接線/儀器 | 只需 UART console（50 s 那顆要等） |
| 陷阱 | `cfg.prescale` 仍填 `PRESCALE_16` 但**會被忽略**（driver 是 if/else 二選一） |
| 驗證表測項 | Timer / **Prescale** |
| RM | Ch13 §Pre-scaler |
| **判定條件** | **待實測**（重點：實測分頻比到底是 ÷N 還是 ÷(N+1)） |

### 4.6 timer-capture（584 only）

| | |
| --- | --- |
| 在測什麼 | 輸入邊緣捕捉 + capture 中斷狀態 |
| 設定 | T0：一般 timer，週期翻轉 **GPIO22** 當測試訊號源<br>T1：capture，`timeout 4000000`，**ch0 = GPIO30**，正緣<br>T2：capture，`timeout 0xFFFFFFF`，**ch0 = GPIO30 正緣 + ch1 = GPIO31 負緣** |
| 接線 | **GPIO22 (輸出) → 接到 GPIO30 和 GPIO31**；或外部訊號產生器打進 GPIO30/31 |
| 儀器 | 邏輯分析儀 / 示波器（確認 GPIO22 波形與捕捉值一致）；最低限度只要杜邦線回接 + UART console |
| 已知問題 | `timer_cap_handler1` 連續呼叫兩次 `hosal_timer_ch0_capture_int_status`，第二次應為 ch1（finding O） |
| 驗證表測項 | Timer / **Capture**、**Capture Interrupt Status** |
| RM | Ch13 §Input Capture |
| **判定條件** | **待實測** |

### 4.7 timer-pwm（584 only）

| | |
| --- | --- |
| 在測什麼 | timer 簡易 PWM，且**在 sleep 模式下仍要能輸出** |
| 腳位 | GPIO4=PWM0、GPIO5=PWM1、GPIO20=PWM2、GPIO21=PWM3、GPIO30=PWM4（`hosal_pin_set_mode`） |
| 設定 | T0 → PWM0（÷16, period 2000, thd 1000, **pha 0**）<br>T1 → PWM1+PWM2（÷16, period 2000, thd 1000, **pha 1**）<br>T2 → PWM3+PWM4（÷32, period 2000, thd 1000, pha 1） |
| 預期波形 | 皆 50% duty；T0/T1 頻率相同（32MHz÷16÷2000 = 1 kHz），T2 為其一半（500 Hz）；**pha 0 與 pha 1 相位相反** |
| 低功耗 | `main()` 設 GPIO0 為 wakeup source、進入 `HOSAL_LPM_SLEEP` 後在 while 迴圈反覆 sleep → **PWM 應在 sleep 期間持續輸出** |
| 接線/儀器 | **示波器或邏輯分析儀必要**（量 5 支腳的頻率/duty/相位）；量電流可一併驗低功耗 |
| 已知問題 | `hosal_timer_pwm_start()` 呼叫 `lpm_enable_timer_pwm()` 但全 repo 無人呼叫 `lpm_disable_timer_pwm()`（finding J）；`timer_pwm_open()` 會偷偷把時脈切成 PMU 且不還原（finding G/H） |
| 驗證表測項 | Timer / **PWM** |
| RM | Ch13 §PWM Function |
| **判定條件** | **待實測** |

---

## 5. 驗證表對照

`RT584_MPA_IC_peripheral_verification_status_20241008.xlsx` → sheet **Timer**，8 項全 Pass：

| # | 測項 | 對應 example | 備註 |
| --- | --- | --- | --- |
| 1 | Free Run | timer-freerun-upcount / timer-freerun-downcount | 表上未分上/下數 |
| 2 | Periodic | timer-periodic | |
| 3 | Prescale | timer-user-prescale | 表上未分 3-bit / 10-bit |
| 4 | Interrupt Interval | timer-periodic / timer-freerun-* | |
| 5 | Status Interval | timer-periodic | 需讀 `int_status`，example 沒讀（HOSAL 也沒 API）→ 見 finding K |
| 6 | Capture | timer-capture | 584 only |
| 7 | Capture Interrupt Status | timer-capture | 584 only |
| 8 | PWM | timer-pwm | 584 only |

**缺口**
- 有 example 沒測項：**timer-oneshot**
- 有測項但 example 覆蓋不足：**Status Interval**（沒有 example 讀 `int_status`）

---

## 6. Code Review findings

分級：🔴 = 功能性 bug、🟡 = 健壯性/可維護性、🔵 = 討論題。

### Driver（`rt584_driver/Src/timer.c`）

**🔴 A. 邊界檢查全部 off-by-one（30 處）**
```c
#define MAX_NUMBER_OF_TIMER  (3)
timern_t *base[MAX_NUMBER_OF_TIMER] = {TIMER0, TIMER1, TIMER2};
if (timer_id > MAX_NUMBER_OF_TIMER) {      /* 應為 >= */
    return STATUS_INVALID_PARAM;
}
timer = base[timer_id];                     /* timer_id == 3 → base[3] 越界 */
```
全檔 **21 處 fast timer + 9 處 slow timer，零處使用 `>=`**。`timer_id == 3` 會讀到堆疊上陣列後面的垃圾值當成暫存器位址寫下去。
→ 建議：全部改 `>=`，並抽成 `#define TIMER_ID_INVALID(id) ((id) >= MAX_NUMBER_OF_TIMER)`。

**🔴 B. `timer_clear_int()` 是 `timer_close()` 的逐字複製**
```c
uint32_t timer_clear_int(uint32_t timer_id) {
    ...
    timer->control.reg = 0;
    timer_cfg[timer_id].timer_callback = NULL;
    timer_cfg[timer_id].state = TIMER_STATE_CLOSED;
    return STATUS_SUCCESS;
}
```
函式名說「清中斷」，實際上是「關閉 timer 並丟掉 callback」，而且**完全沒碰 `clear` 暫存器**。`slowtimer_clear_int()` 同樣。
→ 建議：改成 `timer->clear = 1;` 就好，或直接刪掉這個 API（目前無人呼叫）。

**🔴 C. `timer_capture_stop()` 把 timer 打開了**
```c
uint32_t timer_capture_stop(uint32_t timer_id) {
    ...
    timer->control.bit.en = 1;              /* 應為 0 */
    timer->cap_en.bit.ch0_capture_en = 0;
    timer->cap_en.bit.ch1_capture_en = 0;
```
stop 卻設 `en = 1`。capture 是關了，但計數器被啟動。
→ 建議：改 `= 0`。

**🟡 D. `timer_load()` 沒有 zero-guard，且 busy-wait 無上限**
```c
timer->load = (timeload_ticks - 1);         /* timeload_ticks == 0 → 0xFFFFFFFF */
timer->expried_value = timeout_ticks;
while ( !((timer->load == timer->value) && (timer->load == (timeload_ticks - 1))) ) {}
```
- up counting 的正常用法就是 `timeload_ticks = 0`（4.2/4.5/4.7 全部這樣傳），此時 `load` 被寫成 0xFFFFFFFF。up count 從 0xFFFFFFFF 開始數是否符合預期？**待實測**。
- while 迴圈沒有 timeout，若 HW 不如預期就是死當（無 WDT 時要人工 reset）。
→ 建議：`timer->load = timeload_ticks ? timeload_ticks - 1 : 0;`；busy-wait 加上重試上限並回 `STATUS_TIMEOUT`。

**🟡 E. `*_open()` 的 `do{...}while(en_status)` 同樣無上限**（`timer_open` / `timer_capture_open` / `timer_pwm_open` / `slowtimer_open` 四處）。

**🟡 F. 未使用變數**：`timer_ch0_capture_value_get()` / `timer_ch1_capture_value_get()` 各有一個沒用到的 `uint32_t value;`。

**🔴 G. `timer_pwm_open()` 會偷偷把時脈源切成 PMU，且沒人還原**
```c
/* select PMU_CLK */
if (timer_id == 0) SYSCTRL->sys_clk_ctrl1.bit.timer0_clk_sel = TIMER_CLOCK_SOURCEC_PMU;
...
```
`timer_open()` **從不設定 clock source**（沿用現況）。所以順序若是 `pwm_open(0)` → `pwm_close(0)` → `timer_open(0)`，這顆 timer 會靜悄悄跑在 PMU clock 而不是 32 MHz PERI，所有時間計算全錯，而且沒有任何錯誤碼。
→ 建議：(1) `timer_pwm_close()` 還原 clock sel；(2) 或把 clock source 提升成 `timer_config_mode_t` 的一個欄位，讓 `timer_open()` 也明確設定。

**🟡 H. `timer_pwm_close()` 沒清乾淨**：留下 `cap_en.bit.timer_pwm_en` 與 `SYSCTRL->soc_pwm_sel` 的 mux 設定 → 下一顆 timer 想接同一支 PWM 腳會被卡住。

**🔵 I. fast timer 的 ISR 不處理 one-shot，slow timer 的會**
```c
void timer0_handler(void) {
    TIMER0->clear = 1;
    if (timer_cfg[0].timer_callback) timer_cfg[0].timer_callback(0);
}
void slowtimer0_handler(void) {
    if (SLOWTIMER0->control.bit.one_shot_en) SLOWTIMER0->control.bit.en = 0;   /* fast 沒有 */
    SLOWTIMER0->clear = 1; ...
}
```
問題：one-shot 停止是 HW 自己做的，還是要靠 SW 清 `en`？RM 寫 "timer counter stop when the timer expired"，聽起來是 HW 行為 → 那 slow timer ISR 那行是多餘的；若是 SW 行為 → fast timer 的 one-shot 是壞的。**兩套 driver 行為不一致，至少有一邊是錯的。實測 timer-oneshot 時可以直接驗出來。**

### HOSAL（`rt584_hosal/Src/hosal_timer.c`）

**🔴 J. `lpm_enable_timer_pwm()` 有開無關**
```c
uint32_t hosal_timer_pwm_start(uint32_t timer_id, hosal_timer_pwm_config_tick_t cfg) {
    rval = timer_pwm_start(timer_id, cfg.timeload_ticks, cfg.timeout_ticks,
                           cfg.threshold, cfg.phase);
    lpm_enable_timer_pwm();
    return rval;
}
```
`lpm_disable_timer_pwm()`（宣告於 [lpm.h:297](components/platform/soc/rt584/rt584_driver/Inc/lpm.h#L297)，定義於 [lpm.c:181](components/platform/soc/rt584/rt584_driver/Src/lpm.c#L181)）**全 repo 無任何呼叫點**。PWM 一旦開過，這個低功耗抑制旗標就永遠拿不掉。
→ 建議：`hosal_timer_pwm_stop()` / `hosal_timer_pwm_close()` 呼叫 `lpm_disable_timer_pwm()`（需先確認多顆 timer 同時用 PWM 時要不要 refcount）。

**🟡 K. HOSAL 層的覆蓋不完整且無附加價值**
- 211 行裡唯一的行為差異就是 J 那一行，其餘 100% pass-through，只多做一次 struct copy。
- **缺 slow timer 整套**（driver 有 `slowtimer_open/load/start/stop/close/clear_int/int_status_get/current_get`，hosal 一個都沒有）→ `slow-timer` 家族的 example 只能直接呼叫 driver，違反「app 應對 HOSAL 寫」的架構原則。
- 缺 `timer_int_status_get()` 的包裝 → 驗證表的 "Status Interval" 測項在 HOSAL 層無法做。
→ 討論：要嘛把 HOSAL 補齊（含 slowtimer、int_status），要嘛承認 timer 不需要 HOSAL 層。目前是最糟的中間狀態。

### Examples

| | 問題 | 位置 |
| --- | --- | --- |
| 🟡 L | 註解說 8 s，實際 4000000 ÷ (32MHz/32) = **4 s** | timer-periodic `cfg2` |
| 🟡 M | `timerN_done` 只設 0 從不設 1 → `main()` 的 finalize 分支是 dead code | timer-freerun-upcount、timer-freerun-downcount、timer-user-prescale（timer-oneshot 是對的） |
| 🟡 N | callback（ISR context）裡直接 `printf()` | 全部 7 個 example |
| 🔴 O | `timer_cap_handler1` 呼叫兩次 `hosal_timer_ch0_capture_int_status`，第二次應該是 ch1，但後面卻去讀 ch1_value | timer-capture |
| 🟡 P | 傳 `timeload_ticks = 0`，觸發 finding D 的 0xFFFFFFFF | timer-freerun-upcount、timer-user-prescale、timer-pwm |
| 🟡 Q | timer-capture 混用 driver 巨集（`TIMER_UP_COUNTING`、`TIMER_PRESCALE_32`）而非 `HOSAL_TIMER_*`，其餘 example 都用 HOSAL 版 | timer-capture |
| 🔵 R | 三個 free-run/prescale example 的 `main()` while(1) 完全相同且都是 dead code，可整併 | timer-freerun-*、timer-user-prescale |

---

## 7. 待確認 / 待實測清單

實測時請一併回答以下問題，回填到本文件的「判定條件」欄：

1. ~~PERI clock 是不是 32 MHz？~~ **已由暫存器表 MemoryMap sheet 證實為 32M**（TIMER32K0/1 為 32K）。**剩下待實測的是：會不會隨 CPU AHB 頻率變動？** PWM 周邊確定會跟著 `SystemCoreClock` 變，timer 未知。
2. `user_prescale = N` 的分頻比是 ÷N 還是 ÷(N+1)？
3. `user_prescale = 0` 能不能表示 ÷1？（driver 的 if/else 讓它不可能）
4. one-shot 停止是 HW 還是 SW 行為？（finding I）
5. `timeload_ticks = 0` 在 up counting 下，`load` 被寫成 0xFFFFFFFF 是否影響行為？（finding D）
6. free-run down count 且 expired = 0 時，是否真的 wrap 到 0xFFFFFFFF？
7. capture deglitch 的門檻是否真的是 4 個 timer clock？
8. PWM `pha` 的極性是否符合 RM 描述（pha=0 → value ≥ thd 輸出 high）？
9. PWM 在 `HOSAL_LPM_SLEEP` 下是否持續輸出？電流是多少？
10. `timer_capture_stop()` 的 `en = 1`（finding C）在實機上造成什麼現象？
11. 硬體上 `T0_CAP_EN` 的 `ch0_capture_en` / `ch1_capture_en` / `timer_pwm_en` 是三個獨立 bit，**同一顆 timer 應該可以同時 capture + PWM**，但 driver 把兩者做成互斥的 API。實機上直接寫暫存器同時開啟是否可行？

> 相關文件：[pwm-zerocross-dimmer.md](pwm-zerocross-dimmer.md) — 客戶 AC 調光案，
> 內含 timer capture / PWM 周邊的實際應用推導、TIMER32K0/1（timer3/timer4）無 capture 無 PWM 的查證，
> 以及 PWM driver 的 10 項缺陷。

---

## 附錄：本家族的 CI 覆蓋

`.github/config/dev/<chip>/peripheral.json`：7 個 project × 7 顆晶片中，capture / pwm 僅 584 系列有 config。CI 缺口統計見 [RT584_peripheral_coverage_matrix.md](docs/peripheral/RT584_peripheral_coverage_matrix.md) §6。
