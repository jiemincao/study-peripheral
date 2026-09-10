# RT584：用 GPIO 過零訊號同步 PWM（AC 調光 / phase-cut）

> 起因：客戶 Brandon G 詢問「AC light dimmer，PWM 輸出要跟進來 GPIO 的 zero-crossing
> 訊號同步。hardware timer 有 capture compare、PWM 暫存器有 gate function，這兩個能不能用？」
>
> 本文先把「硬體到底有沒有這條路」查清楚，再給出可行的做法與誤差來源。
> **我們手上沒有調光電路，本文所有時間數字都是從暫存器定義推算，尚未上機量測**，
> 需要實測的項目統一列在 §6。

---

## 1. 結論先講

| 問題 | 答案 | 依據 |
|---|---|---|
| PWM 能不能由 GPIO 硬體直接觸發？ | **不能** | `cfgX_pwm_ena_trig` 觸發來源只有 PWM0~PWM4 與 SELF |
| PWM 的 "gate function" 是不是外部閘控？ | **不是**，是省電用的時脈閘控 | `cfgX_ck_ena` = "Enable pwm **gated clock**"、`cfgX_pwm_ck_free` = "0: Auto gated / 1: Free run" |
| Timer capture 能不能啟動 / 重置 PWM？ | **不能**，只會鎖存計數值並發中斷 | RM Ch13；`T0_CAP_EN` 只有 `chN_capture_en` |
| 有沒有任何 GPIO → 周邊的硬體觸發路徑？ | **沒有**。GPIO_OMUX 只有輸出方向的 mux（0x06~0x0A = PWM0~PWM4），輸入側只有 GPIO 自己的邊緣中斷 | 全暫存器表掃描 |
| 那到底怎麼做？ | **GPIO 邊緣中斷 → ISR 內重啟 PWM 一次性波形**。PWM 自己的 15-bit 計數器負責產生「延遲後導通」的波形，軟體只負責每半週期重新對齊一次 | 見 §3 |

也就是說：**同步這件事一定會經過一次 CPU 中斷**，做不到純硬體。
但因為 PWM 的計數器會自己走完整個半週期，CPU 只在每個過零點做一次「對齊」，
中斷延遲是一個**固定偏移量 + 抖動**，固定偏移可以在軟體裡補回來（見 §5）。

### 1.1 順帶查證：timer3 / timer4

若客戶用的是 timer3 / timer4（SDK 內名稱 `TIMER32K0` / `TIMER32K1`，暫存器前綴 `T3_` / `T4_`）：

- 參考時脈是 **32K，不是 40K**（MemoryMap sheet：TIMER0/1/2 = 32M，TIMER32K0/1 = 32K）
- **完全沒有 capture、也沒有 PWM 功能**。TIMER32K0 整張暫存器表只有
  `T3_LOAD / T3_VALUE / T3_CONTROL / T3_CLEAR / T3_REPEAT_DELAY / T3_PRESCALE / T3_EX_VAL`，
  搜尋 capture / pwm / thd / pha 為 0 筆
- tick = 30.5 µs，60 Hz 半週期只切得出約 273 階
- 唯一跟時序有關的額外功能 `int_repeat_delay` 註明 "for sleep mode only"

→ **要做調光只能用 TIMER0/1/2（32 MHz）或 PWM 周邊。**這點要先跟客戶確認清楚。

---

## 2. 訊號流向

```
  AC 市電
    │
    ▼
  光耦合過零偵測器 ── 每個半週期一個脈波（60Hz→120Hz，50Hz→100Hz）
    │
    ▼
  GPIO input（邊緣中斷）        ← 這是「輸入」
    │
    ▼  ISR：重啟 PWM
  PWM 計數器（自走 15-bit）
    │
    ▼
  PWM output ── TRIAC gate     ← 這是「輸出」
```

TRIAC 的行為：被 gate 觸發後導通，**到下一個過零點自己熄滅**。
所以「過零後延遲多久才觸發」＝亮度。延遲越短越亮。

---

## 3. 建議做法：GPIO ISR + PWM 一次性波形

### 3.1 為什麼用 PWM 周邊而不用 timer 的 PWM

- Timer PWM（`value_thd` + `pha`）也做得到，但 `timer_pwm_open()` 會把該 timer 的
  時脈來源永久切到 PMU，而 `timer_pwm_close()` 不會還原（見 `timer.md` §6 findings）
- PWM 周邊的 element 一次就把「週期」跟「門檻」都帶進去，重填一個 32-bit 字即可改亮度
- 驅動層雖然全部走 xDMA，但單一 element 的情況實際上只是 DMA 搬 1 個字，成本很低

### 3.2 時脈與解析度

```
PWM clock = SystemCoreClock / (1 << cfg_ck_div)
```

**注意這是跟著 CPU 時脈走的，不是固定 32 MHz**（`pwm_set_frequency()` 直接用
`SystemCoreClock`）。以 `SystemCoreClock = 32 MHz` 計：

| ck_div | PWM clock | tick | 60Hz 半週期(8.333ms) counts | 50Hz 半週期(10ms) counts |
|---|---|---|---|---|
| 1 | 32 MHz | 31.25 ns | 266,656 ✗ 超過上限 | 320,000 ✗ |
| 8 | 4 MHz | 250 ns | 33,332 ✗ | 40,000 ✗ |
| **16** | **2 MHz** | **0.5 µs** | **16,666 ✓** | **20,000 ✓** |
| 32 | 1 MHz | 1 µs | 8,333 ✓ | 10,000 ✓ |

`cfgX_pwm_cnt_end` 是 **15-bit，有效範圍 3 ~ 32767**，所以 **120 Hz / 100 Hz 必須用
ck_div ≥ 16**。建議 **ck_div = 16**，0.5 µs 解析度、50/60 Hz 都不會溢位。

#### 兩級結構：ck_div 與 PWM 自己的 counter

```
SystemCoreClock ─►[ ck_div 或 ck_div_val ]─► PWM engine clock ─►[ 15-bit counter 0..cnt_end ]─► 比 thd ─► pwm_out
                    ↑ 二選一，見下表                                ↑ 這就是「PWM 自己的 timer」
```

`ck_div` 給刻度、`cnt_end` 給格數，兩者相乘才是週期。counter 只有 15-bit，所以低頻一定得靠
`ck_div` 把刻度放大。

**分頻器有兩顆，二選一**（暫存器表原文）：

| 欄位 | bits | reset | 說明 |
|---|---|---|---|
| `cfgX_ck_div` | [11:8] | 0 | enum：0=÷1、1=÷2、2=÷4、3=÷8、4=÷16、5=÷32、6=÷64、7=÷128、8=÷256、其他=÷256。**"only valid when `cfgX_ck_div_val`=0"** |
| `cfgX_ck_div_val` | [31:24] | 0 | 任意分頻值，**"only valid when >0"** |

這跟 **timer 的 prescaler 是一模一樣的二選一設計**（`control.prescale` 3-bit enum vs 獨立的
10-bit `prescale` 暫存器，後者 >0 時前者失效）——同一個 IP 設計慣例，但**兩者是完全獨立的
方塊，PWM 的 `ck_div` 與 timer 的 `prescale` 沒有任何關係**。關鍵差別在參考時脈：

| | 參考時脈 | 分頻欄位 |
|---|---|---|
| TIMER0/1/2 | **固定 32M**（MemoryMap sheet 明寫） | `prescale` / `control.prescale` |
| TIMER32K0/1 | **固定 32K** | `timer3_prescale` / `control.timer_prescale` |
| PWM0~4 | **`SystemCoreClock`**（`pwm_set_frequency()` 直接讀） | `cfgX_ck_div` / `cfgX_ck_div_val` |

→ CPU 跑 48 MHz 時，同樣設 ÷16，timer 還是 2 MHz，**PWM 會變成 3 MHz**。

⚠️ 驅動的 `pwm_set_t` bitfield 只宣告 `cfg_ck_div : 4`，`ck_div_val` 落在 `RVD4`（[31:20] 保留區）
裡，從來沒有被寫過。目前之所以沒出事，是因為 `pwm_init_fmt1()` / `pwm_init_fmt0()` 是
**整個字覆寫** `pwm->pwm_set0 = reg_config;`，順手把 `ck_div_val` 清成 0，`ck_div` 才會生效。
**這是巧合不是保證**——任何人把它改成 read-modify-write（例如 `pwm_clock_divider()` 就是
read-modify-write）都不會踩到，但若有人先寫了 `ck_div_val` 就會靜默失效。

⚠️ 驅動的 `pwm_init_fmt1()` 會把 `cfg_ck_div` 寫死成 `PWM_CLK_DIV_1` 再算 frequency，
所以**必須在 init 之後補做「先設 divider、再重設 frequency」兩步**，否則 full_count 會爆掉。

### 3.3 element 格式（Format 1）

```c
#define PWM_FILL_SAMPLE_DATA_MODE1(phase, thd, cnt_end)  \
        ((cnt_end << 16) | (phase << 15) | (thd))
```

| 欄位 | bits | 意義 |
|---|---|---|
| `cnt_end` | [31:16] | 這個 element 的計數上限（＝半週期） |
| `phase` | [15] | 0 = POSITIVE：前 `thd` 個 count 輸出高，之後低<br>1 = NEGATIVE：前 `thd` 個 count 輸出低，之後高 |
| `thd` | [14:0] | 門檻 |

**phase-cut 要的是 NEGATIVE**：過零後先低 `thd` 個 tick（＝觸發延遲），
然後拉高把 TRIAC gate 點著，直到半週期結束。

亮度 ↔ 延遲：

```
thd = half_cycle_counts * (100 - brightness_percent) / 100
```

`thd = 0` → 全亮；`thd = half_cycle_counts` → 全暗。

### 3.4 完整流程

```c
#include "hosal_gpio.h"
#include "hosal_pwm.h"
#include "hosal_dma.h"

#define ZC_PIN          0        /* 過零訊號進來的腳 */
#define GATE_PIN        20       /* PWM0 輸出 → TRIAC gate */
#define MAINS_HALF_HZ   120      /* 60Hz 市電；50Hz 改 100 */

static hosal_pwm_dev_t   pwm_dev;
static volatile uint32_t g_thd;          /* 目前的觸發延遲(counts) */
static uint32_t          g_half_counts;  /* 一個半週期的 counts */

/* ---- 每個過零脈波觸發一次 ---- */
static void zc_isr(uint32_t pin, void* param) {
    hosal_pwm_stop(HOSAL_PWM_ID_0);               /* 注意：會把 element 清 0 */
    hosal_pwm_fmt1_count(HOSAL_PWM_ID_0, g_thd);  /* 重填 element 並 pwm_start() */
}

void dimmer_set_brightness(uint8_t percent) {     /* 0 = 全暗, 100 = 全亮 */
    if (percent > 100) { percent = 100; }
    g_thd = (g_half_counts * (100 - percent)) / 100;
}

void dimmer_init(void) {
    hosal_gpio_input_config_t zc_cfg;

    hosal_dma_init();

    /* --- PWM0 --- */
    pwm_dev.config.id        = HOSAL_PWM_ID_0;
    pwm_dev.config.pin_out   = GATE_PIN;
    pwm_dev.config.frequency = MAINS_HALF_HZ;
    hosal_pwm_init_fmt1(&pwm_dev);

    /* init 把 ck_div 寫死成 1，這裡改成 16 再重算 full_count */
    pwm_dev.config.clk_div = HOSAL_PWM_CLK_DIV_16;
    hosal_pwm_ioctl(&pwm_dev, HOSAL_PWM_SET_CLOCK_DIVIDER, NULL);

    pwm_dev.config.phase = PWM_PHASE_NEGATIVE;    /* 先低後高 */
    hosal_pwm_ioctl(&pwm_dev, HOSAL_PWM_SET_PHASE, NULL);

    hosal_pwm_ioctl(&pwm_dev, HOSAL_PWM_SET_FRQUENCY, NULL);

    /* ⚠️ pwm_get_count() 目前是壞的（見 §7 P1），自己算 */
    g_half_counts = (SystemCoreClock / 16) / MAINS_HALF_HZ;  /* 32M/16/120 = 16666 */

    dimmer_set_brightness(50);

    /* --- 過零輸入 --- */
    zc_cfg.param        = NULL;
    zc_cfg.pin_int_mode = HOSAL_GPIO_PIN_INT_EDGE_RISING;
    zc_cfg.usr_cb       = zc_isr;
    hosal_pin_set_mode(ZC_PIN, MODE_GPIO);
    hosal_gpio_cfg_input_parameters(ZC_PIN, zc_cfg, true);
    /* 過零脈波不要開 debounce：debounce 走 32K slow clock，一格就 30.5us */
    hosal_gpio_debounce_disable(ZC_PIN);
    NVIC_SetPriority(Gpio_IRQn, 0);
    NVIC_EnableIRQ(Gpio_IRQn);
}
```

### 3.5 中斷優先權

過零 ISR 必須是系統裡最高優先權之一，否則被 BLE / Thread 的 RF 中斷擋住，
亮度會抖。ISR 內**不要 printf**。

---

## 4. 搭配 Timer capture 做量測與補償（選用）

TIMER0/1/2（32 MHz）的 input capture 可以把同一支過零腳接進來：

- `chN_capture_io_sel` 是 5-bit，**GPIO 0~31 任何一支都可以**
- 邊緣到達時把計數值鎖進 `chN_cap_value` 並發中斷
- 內建 deglitch，會濾掉短於 4 個 timer clock 的脈波
- **但它只鎖存、不會啟動 / 重置 / 閘控計數器**

用途：

1. **量測市電實際週期**（連續兩次 capture 相減），50/60 Hz 自動判別、頻率飄移自動追蹤
2. **量測光耦脈波寬度**（rising 用 ch0、falling 用 ch1），取中點當真正的過零時刻
3. **量測 ISR 抖動**：capture 值是硬體鎖的，跟軟體進 ISR 的時間差就是延遲

⚠️ 硬體上 `T0_CAP_EN` 的 `ch0_capture_en` / `ch1_capture_en` / `timer_pwm_en` 是三個獨立
bit，**同一顆 timer 可以同時 capture + PWM**；但目前驅動把 capture 與 PWM 拆成互斥的
兩組 API，做不到。（已記入 code review）


---

## 4.5 低功耗：睡眠時 PWM 還在不在？

| 模式 | PWM / TIMER0-2 | 說明 |
|---|---|---|
| **Sleep** | **還活著**（需自行開啟） | `lpm.c` 有 `lpm_enable_timer_pwm()`，設了之後 `peri3_off = false`，PERI3 電源保持，timer 與 PWM 在 sleep 下持續輸出 |
| **Deep sleep** | **全滅** | PERI1/2/3 全部斷電（除非用 `keep_on` 明確保留）；`cfg_ds_rco32k_off` 預設關掉 RCO32K，只有 RTC / AUX comp / BOD comp 會把它留著 |
| **Deep power down** | 全滅 | — |

所以「要睡覺就只能用 32K timer」這句話**只對 deep sleep 成立**；而 TIMER32K0/1
**本身就沒有 PWM 功能**，所以深睡期間不可能有調光輸出。

不過對本應用而言這多半是假議題：

- 過零脈波每 8.33 ms（60 Hz）就來一次，中間沒有值得睡的空檔；deep sleep 的喚醒延遲
  加上 32 MHz 時脈重新鎖定的時間，相位預算直接爆掉
- TRIAC 調光器是插在市電上的，通常不是電池裝置

合理的睡眠情境是「燈全關 → 進 deep sleep → GPIO / RTC 喚醒 → 重新 init PWM 開始調光」，
不需要 PWM 在睡眠中維持。若客戶真的要「睡覺同時還要調光」，那就只能用 sleep（不是
deep sleep）＋ `lpm_enable_timer_pwm()`，且要接受 PERI3 保持供電的功耗。**待實測其電流。**

---

## 5. 誤差來源（一定要跟客戶講）

| # | 來源 | 量級 | 能不能補 |
|---|---|---|---|
| 1 | GPIO 中斷進入延遲（NVIC + flash wait + 其他 ISR） | 數 µs ~ 數十 µs | 固定部分可補；被高優先權 ISR 搶走的部分不可補 |
| 2 | 光耦脈波寬度：真正的過零在**脈波中心**，不是邊緣 | 視光耦與限流電阻，可達數百 µs | 可補：用 timer capture 量 rising/falling 取中點 |
| 3 | PWM clock = `SystemCoreClock / 2^ck_div`，**CPU 時脈變動時 PWM 週期會跟著變** | 依時脈切換而定 | 要在時脈切換後重設 frequency |
| 4 | 內部 32 MHz 參考時脈誤差 vs 市電 50/60 Hz 本身的飄移 | ppm 級，但長時間累積 | 用 timer capture 每個週期重新量測即可 |
| 5 | 0.5 µs 量化（ck_div=16） | ±0.5 µs ≈ 60Hz 半週期的 0.006% | 可忽略 |
| 6 | `pwm_start()` 是否真的把計數器歸零，驅動沒有 reset API | **未知** | **必須實測**，見 §6 |

以 60 Hz 為例，半週期 8333 µs。若總誤差 50 µs，相當於相位誤差 **1.08°**、亮度誤差約 0.6%。
一般調光可接受，但**這個數字我們沒有量過**。

---

## 6. 必須上機驗證的項目

我們手上**沒有調光電路，也沒有市電過零訊號**，只能用跳線把一支 PWM/GPIO 輸出接到過零輸入腳
來產生乾淨的 120 Hz 刺激（`timer-capture` 範例就是用 GPIO22 輸出 → GPIO30/31 capture 這個手法）。

| # | 要驗證什麼 | 怎麼驗 | 結果 |
|---|---|---|---|
| V1 | `pwm_stop()` → `pwm_fmt1_count()` 之後計數器是否從 0 重新開始 | 示波器同時看刺激訊號與 PWM 輸出，看每個週期的上升緣位置是否固定 | 待實測 |
| V2 | 若 V1 不成立，是否需要額外寫 `pwm_ctl1 \|= PWM_RESET`（驅動沒有這個 API） | 直接寫暫存器比對 | 待實測 |
| V3 | ck_div=16 + frequency=120 算出來的 `full_count` 是否真的等於 16666 | 量 PWM 自走時的週期是否 8.333 ms | 待實測 |
| V4 | GPIO ISR 進入延遲與抖動 | 刺激訊號 vs ISR 內 toggle 一支 debug GPIO，示波器量 | 待實測 |
| V5 | `PWM_PHASE_NEGATIVE` 是否真的是「先低後高」 | 設 thd = 50% 看波形 | 待實測 |
| V6 | 過零腳關掉 debounce 後最短能認到多寬的脈波 | 逐步縮短刺激脈波寬度 | 待實測 |
| V7 | 同一顆 timer 同時開 capture + PWM（繞過驅動，直接寫 `T0_CAP_EN`） | 直接寫暫存器 | 待實測 |

---

## 7. 這次順帶發現的 PWM driver 缺陷（併入 code review）

| # | 位置 | 問題 | 嚴重度 |
|---|---|---|---|
| P1 | `pwm.c` 6 個 getter、共 13 處 | `*(uint32_t*)&get_xxx = value;` 寫到區域指標參數**本身的位址**，不是它指向的地方。`(uint32_t*)` 轉型把編譯器警告吃掉了。<br>受影響：`pwm_get_frequency`:248、`pwm_get_pahse`:305、`pwm_get_count`:321、`pwm_get_repeat_number`:825/827/831/833、`pwm_get_delay_number`:881/883/887/889、`pwm_get_inveter`:907。<br>只有 `pwm_get_dma_element` 是對的。 | 🔴 |
| P2 | `pwm_set_delay_number()`，SEQ_NUM_1 + ORDER_T 分支 | 寫到 `pwm->pwm_set0`（**設定暫存器**）而不是 `pwm_set8`，會把整個 PWM 設定洗掉 | 🔴 |
| P3 | `pwm_clock_divider()` | `pwm_t* pwm = m_pwm_handle[id].pwm;` 在 `if (id > PWM_ID_MAX)` **之前**就解參考；而且是 `>` 不是 `>=` | 🔴 |
| P4 | `pwm_stop()` | 會把使用者的 rseq/tseq buffer 全部寫 0，且在「停止」流程裡反而 `\|= PWM_RDMA_ENABLE`。stop→start 不能續播，一定要重填 | 🟡 |
| P5 | 驅動層 | 沒有任何 register mode API，一切都得走 xDMA；`cfgX_pwm_cnt_trig` 從頭到尾沒有任何程式碼寫過 | 🟡 |
| P6 | 驅動層 | 沒有暴露 PWM counter reset（`pwm_ctl1 \|= PWM_RESET` 只在 init 內部用），本文 §6 V2 就是被這個卡住 | 🟡 |
| P7 | `hosal_pwm_ioctl()` | header 宣告了 26 個 ioctl，switch 只實作 18 個。`SET_PLAY_NUMBER`/`SET_TSEQ_ADDRESS`/`SET_RSEQ_ADDRESS`/`GET_COUNT_MODE`/`GET_DUTY`/`GET_PLAY_NUMBER`/`GET_TSEQ_ADDRESS`/`GET_RSEQ_ADDRESS`/`DISABLE_INTERRUPT`/`ENABLE_INTERRUPT` 都掉進 `default: return -1` | 🟡 |
| P8 | `pwm_init_fmt1()` / `pwm_init_fmt0()` | 把 `cfg_ck_div` 寫死 `PWM_CLK_DIV_1`，`pwm_config_t.clk_div` 被忽略；使用者必須事後補呼叫 divider + frequency 兩步 | 🟡 |
| P9 | `pwm_set_frequency()` | FMT0 + PHASE_POSITIVE 時把 `full_count` 設成 **0**，之後 `pwm_fmt1_count()` 的夾限就失效了 | 🟡 |
| P10 | 全部 4 個 pwm example | `triggered_src` 一律 `SELF`，PWM→PWM 串接完全沒有被驗證過 | 🔵 |
| P11 | `pwm_set_t` bitfield | `cfgX_ck_div_val`（[31:24]，第二顆分頻器）沒有被宣告，落在 `RVD4` 保留區。`cfg_pwm_cnt_trig`（bit[5]）有宣告但全 repo 無人寫過 | 🔵 |

---

## 8. 給客戶的回覆草稿（英文）

> Hi Brandon,
>
> Short answer: **there is no hardware path from a GPIO input to the PWM block on RT584** —
> the synchronisation has to go through one CPU interrupt. Three clarifications on the
> features you mentioned:
>
> - The PWM "gate function" (`cfg_ck_ena` / `cfg_pwm_ck_free`) is a **clock gate for power
>   saving**, not an external enable input.
> - The PWM counter's own trigger source (`cfg_pwm_ena_trig`) can only be **PWM0..PWM4 or
>   itself** — there is no GPIO option.
> - The hardware timer's input capture **latches the counter value and raises an interrupt**;
>   it does not start, reset or gate anything.
>
> What we would suggest instead:
>
> 1. Feed the opto-isolated zero-cross pulse into a GPIO configured for an **edge interrupt**,
>    at the highest NVIC priority.
> 2. Configure the **PWM peripheral** in Format 1 with `ck_div = 16` (2 MHz, 0.5 µs per tick)
>    and one element whose `cnt_end` is one mains half-cycle (16666 counts @ 60 Hz,
>    20000 @ 50 Hz — note the counter is 15-bit, max 32767, so `ck_div = 1` will overflow)
>    and whose `phase` bit is set to **negative**, i.e. the output stays low for `thd`
>    ticks and then goes high. `thd` is your firing delay, so
>    `thd = half_cycle_counts * (100 - brightness%) / 100`.
> 3. In the zero-cross ISR, restart the PWM. The PWM counter then runs the whole half-cycle
>    on its own, so the CPU is only involved once per half-cycle and the interrupt latency
>    becomes a **fixed offset plus jitter** rather than a per-edge cost.
> 4. Optionally, capture the same zero-cross pin with **TIMER0/1/2** (32 MHz reference) on
>    channel 0 (rising) and channel 1 (falling). That gives you the true mains period and
>    the centre of the opto pulse, which is where the real zero crossing is — the edge is not.
>
> Please note: **we do not have an equivalent dimmer circuit here, so we have not been able
> to measure this against real mains.** The numbers above are derived from the register
> definitions, so some deviation is to be expected. In particular the following would need
> to be verified on your hardware:
>
> - the total fixed offset and jitter of the GPIO interrupt path in your firmware
>   (expect tens of microseconds; at 60 Hz, 50 µs ≈ 1.1° of phase error);
> - the width of your opto-coupler pulse, since the true zero crossing is at its centre;
> - that the PWM clock is `SystemCoreClock / 2^ck_div` — if your application changes the
>   CPU clock at runtime, the PWM period changes with it and must be reprogrammed.
>
> One more thing to confirm: **which timer are you planning to use?** If you meant
> "timer 3 / timer 4", those are the 32 kHz slow timers (`TIMER32K0` / `TIMER32K1`) and they
> have **neither input capture nor a PWM function** — only TIMER0/1/2 do.
>
> Happy to go deeper on any of these.

---

## 9. 待跟客戶確認

1. 用的是 TIMER0/1/2 還是 timer3/timer4（TIMER32K0/1）？
2. 市電是 50 Hz 還是 60 Hz？（決定 `cnt_end`）
3. 調光器是 TRIAC（leading-edge）還是 MOSFET（trailing-edge）？後者的波形極性相反
4. 光耦輸出的脈波寬度？（決定要不要做中點補償）
5. 需要幾階亮度？（決定 ck_div 能不能再放大以省電）
6. 韌體裡還有沒有跑 BLE / Thread？（決定過零 ISR 會不會被搶）
7. 有沒有低功耗需求？要「睡覺時仍持續調光」的話只能用 sleep + `lpm_enable_timer_pwm()`，deep sleep 下 PWM 會斷電（見 §4.5）
