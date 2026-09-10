# RT58x / RT584 Peripheral 對照表

> 產生日期：2026-09-10　·　狀態：**對照表完成，判定條件全部待實測**

三份來源的交叉對照：

| 來源 | 檔案 | 內容 |
|---|---|---|
| Example | `examples/peripheral/` | 22 家族 / **109 project** / 629 chip-config |
| 驗證表 | `D:\workdir\584_doc\RT584_MPA_IC_peripheral_verification_status_20241008.xlsx` | 21 sheet / **120 測項**（119 Pass、1 Fail） |
| RM | `D:\workdir\584_doc\RT584_Reference_Manual_rev-1.00.docx` | **30 章**（rev 1.00） |

⚠ **RM 章號是由 docx 的 Heading-1 出現順序推算**（Introduction = Ch1），Word 自動編號無法從檔案直接讀出。章名可信，章號請對一次實體 TOC 再定案。

⚠ 「判定條件」欄位一律留 `待實測` — 未實際跑過的東西不推測。

---

## 1. 主對照表（109 project）

晶片欄縮寫：`58x` = RT581/582/583，`584` = RF1301/RT584H/RT584HA4/RT584L。

| # | Project | 周邊 | 晶片 | 驗證表測項 | RM 章 | 判定條件 | 備註 |
|---|---|---|---|---|---|---|---|
| 1 | [aux-deepsleep](examples/peripheral/aux-comp/aux-deepsleep) | AUX Comparator | 584 | AUX_BOD · Deepsleep Wakeup with/without 32K | Ch26 AUX Comparator | 待實測 |  |
| 2 | [aux-deepsleep-counter](examples/peripheral/aux-comp/aux-deepsleep-counter) | AUX Comparator | 584 | AUX_BOD · Deepsleep Wakeup + Counter Mode | Ch26 AUX Comparator | 待實測 | 兩測項合併 |
| 3 | [aux-deepsleep-level](examples/peripheral/aux-comp/aux-deepsleep-level) | AUX Comparator | 584 | **無對應測項** | Ch26 AUX Comparator | 待實測 | AUX_BOD sheet 無 level 觸發測項 |
| 4 | [aux-normal](examples/peripheral/aux-comp/aux-normal) | AUX Comparator | 584 | AUX_BOD · Interrupt | Ch26 AUX Comparator | 待實測 |  |
| 5 | [aux-normal-counter](examples/peripheral/aux-comp/aux-normal-counter) | AUX Comparator | 584 | AUX_BOD · Counter Mode | Ch26 AUX Comparator | 待實測 |  |
| 6 | [aux-sleep](examples/peripheral/aux-comp/aux-sleep) | AUX Comparator | 584 | AUX_BOD · Sleep Wakeup | Ch26 AUX Comparator | 待實測 |  |
| 7 | [aux-sleep-counter](examples/peripheral/aux-comp/aux-sleep-counter) | AUX Comparator | 584 | AUX_BOD · Sleep Wakeup + Counter Mode | Ch26 AUX Comparator | 待實測 | 兩測項合併 |
| 8 | [bod-deepsleep](examples/peripheral/bod-comp/bod-deepsleep) | BOD Comparator | 584 | AUX_BOD · Deepsleep Wakeup with/without 32K | Ch27 BOD Comparator | 待實測 | sheet 未區分 AUX/BOD |
| 9 | [bod-deepsleep-counter](examples/peripheral/bod-comp/bod-deepsleep-counter) | BOD Comparator | 584 | AUX_BOD · Deepsleep Wakeup + Counter Mode | Ch27 BOD Comparator | 待實測 | sheet 未區分 AUX/BOD |
| 10 | [bod-deepsleep-level](examples/peripheral/bod-comp/bod-deepsleep-level) | BOD Comparator | 584 | **無對應測項** | Ch27 BOD Comparator | 待實測 | AUX_BOD sheet 無 level 觸發測項 |
| 11 | [bod-normal](examples/peripheral/bod-comp/bod-normal) | BOD Comparator | 584 | AUX_BOD · Interrupt | Ch27 BOD Comparator | 待實測 | sheet 未區分 AUX/BOD |
| 12 | [bod-normal-counter](examples/peripheral/bod-comp/bod-normal-counter) | BOD Comparator | 584 | AUX_BOD · Counter Mode | Ch27 BOD Comparator | 待實測 | sheet 未區分 AUX/BOD |
| 13 | [bod-sleep](examples/peripheral/bod-comp/bod-sleep) | BOD Comparator | 584 | AUX_BOD · Sleep Wakeup | Ch27 BOD Comparator | 待實測 | sheet 未區分 AUX/BOD |
| 14 | [bod-sleep-counter](examples/peripheral/bod-comp/bod-sleep-counter) | BOD Comparator | 584 | AUX_BOD · Sleep Wakeup + Counter Mode | Ch27 BOD Comparator | 待實測 | sheet 未區分 AUX/BOD |
| 15 | [comparator](examples/peripheral/comp/comparator) | Comparator (RT58x) | 58x | **無對應測項** | — (RT584 RM 無此章（RT58x 專屬 comparator）) | 待實測 | RT584 驗證表無 RT58x comparator；AUX_BOD sheet 僅涵蓋 584 |
| 16 | [crypto_aes128_cbc](examples/peripheral/crypto/crypto_aes128_cbc) | Crypto Engine | 58x+584 | CRYPTO · AES | Ch28 Crypto Engine | 待實測 |  |
| 17 | [crypto_aes128_ctr](examples/peripheral/crypto/crypto_aes128_ctr) | Crypto Engine | 58x+584 | CRYPTO · AES | Ch28 Crypto Engine | 待實測 |  |
| 18 | [crypto_aes128_decrypt](examples/peripheral/crypto/crypto_aes128_decrypt) | Crypto Engine | 58x+584 | CRYPTO · AES | Ch28 Crypto Engine | 待實測 |  |
| 19 | [crypto_aes128_encrypt](examples/peripheral/crypto/crypto_aes128_encrypt) | Crypto Engine | 58x+584 | CRYPTO · AES | Ch28 Crypto Engine | 待實測 |  |
| 20 | [crypto_aes192_ctr](examples/peripheral/crypto/crypto_aes192_ctr) | Crypto Engine | 58x+584 | CRYPTO · AES | Ch28 Crypto Engine | 待實測 |  |
| 21 | [crypto_aes256_decrypt](examples/peripheral/crypto/crypto_aes256_decrypt) | Crypto Engine | 58x+584 | CRYPTO · AES | Ch28 Crypto Engine | 待實測 |  |
| 22 | [crypto_aes256_encrypt](examples/peripheral/crypto/crypto_aes256_encrypt) | Crypto Engine | 58x+584 | CRYPTO · AES | Ch28 Crypto Engine | 待實測 |  |
| 23 | [crypto_ccm](examples/peripheral/crypto/crypto_ccm) | Crypto Engine | 58x+584 | CRYPTO · CCM | Ch28 Crypto Engine | 待實測 |  |
| 24 | [crypto_ctr_drbg](examples/peripheral/crypto/crypto_ctr_drbg) | Crypto Engine | 584 | CRYPTO · CTR_DRBG | Ch28 Crypto Engine | 待實測 |  |
| 25 | [crypto_curve_c25519_1](examples/peripheral/crypto/crypto_curve_c25519_1) | Crypto Engine | 58x+584 | CRYPTO · ECC | Ch28 Crypto Engine | 待實測 | sheet 未細分曲線 |
| 26 | [crypto_curve_c25519_2](examples/peripheral/crypto/crypto_curve_c25519_2) | Crypto Engine | 58x+584 | CRYPTO · ECC | Ch28 Crypto Engine | 待實測 | sheet 未細分曲線 |
| 27 | [crypto_ecc_gf2m_b163_1](examples/peripheral/crypto/crypto_ecc_gf2m_b163_1) | Crypto Engine | 58x+584 | CRYPTO · ECC | Ch28 Crypto Engine | 待實測 | sheet 未細分曲線 |
| 28 | [crypto_ecc_gf2m_b163_2](examples/peripheral/crypto/crypto_ecc_gf2m_b163_2) | Crypto Engine | 58x+584 | CRYPTO · ECC | Ch28 Crypto Engine | 待實測 | sheet 未細分曲線 |
| 29 | [crypto_ecc_gfp_192](examples/peripheral/crypto/crypto_ecc_gfp_192) | Crypto Engine | 58x+584 | CRYPTO · ECC | Ch28 Crypto Engine | 待實測 | sheet 未細分曲線 |
| 30 | [crypto_ecc_gfp_192_ecdh](examples/peripheral/crypto/crypto_ecc_gfp_192_ecdh) | Crypto Engine | 58x+584 | CRYPTO · ECC | Ch28 Crypto Engine | 待實測 | sheet 未細分曲線 |
| 31 | [crypto_ecc_gfp_p256](examples/peripheral/crypto/crypto_ecc_gfp_p256) | Crypto Engine | 584 | CRYPTO · ECC / ECC_SRAM | Ch28 Crypto Engine | 待實測 |  |
| 32 | [crypto_ecc_gfp_p256_add](examples/peripheral/crypto/crypto_ecc_gfp_p256_add) | Crypto Engine | 584 | CRYPTO · ECC / ECC_SRAM | Ch28 Crypto Engine | 待實測 |  |
| 33 | [crypto_ecc_gfp_p256_ecdh](examples/peripheral/crypto/crypto_ecc_gfp_p256_ecdh) | Crypto Engine | 58x+584 | CRYPTO · ECC / ECC_SRAM | Ch28 Crypto Engine | 待實測 |  |
| 34 | [crypto_ecdsa](examples/peripheral/crypto/crypto_ecdsa) | Crypto Engine | 58x+584 | CRYPTO · ECDSA | Ch28 Crypto Engine | 待實測 |  |
| 35 | [crypto_ecjpake](examples/peripheral/crypto/crypto_ecjpake) | Crypto Engine | 58x+584 | CRYPTO · ECJPAKE | Ch28 Crypto Engine | 待實測 |  |
| 36 | [crypto_hkdf](examples/peripheral/crypto/crypto_hkdf) | Crypto Engine | 58x+584 | **無對應測項** | Ch28 Crypto Engine | 待實測 | CRYPTO sheet 無 HKDF 測項 |
| 37 | [crypto_hmac](examples/peripheral/crypto/crypto_hmac) | Crypto Engine | 58x+584 | **無對應測項** | Ch28 Crypto Engine | 待實測 | CRYPTO sheet 只有 HMAC_DRBG，無純 HMAC |
| 38 | [crypto_hmac_drbg](examples/peripheral/crypto/crypto_hmac_drbg) | Crypto Engine | 584 | CRYPTO · HMAC_DRBG | Ch28 Crypto Engine | 待實測 |  |
| 39 | [crypto_misc](examples/peripheral/crypto/crypto_misc) | Crypto Engine | 584 | CRYPTO · MISC | Ch28 Crypto Engine | 待實測 |  |
| 40 | [crypto_pbkdf2](examples/peripheral/crypto/crypto_pbkdf2) | Crypto Engine | 584 | CRYPTO · PBKDF2 | Ch28 Crypto Engine | 待實測 |  |
| 41 | [crypto_sm2](examples/peripheral/crypto/crypto_sm2) | Crypto Engine | 584 | CRYPTO · SM2 | Ch28 Crypto Engine | 待實測 |  |
| 42 | [crypto_sm3](examples/peripheral/crypto/crypto_sm3) | Crypto Engine | 584 | CRYPTO · SM3 | Ch28 Crypto Engine | 待實測 |  |
| 43 | [crypto_sm4](examples/peripheral/crypto/crypto_sm4) | Crypto Engine | 584 | CRYPTO · SM4 | Ch28 Crypto Engine | 待實測 |  |
| 44 | [dma_interrupt](examples/peripheral/dma/dma_interrupt) | DMA | 58x+584 | DMA · Interrupt | Ch9 DMA | 待實測 |  |
| 45 | [dma_link_list](examples/peripheral/dma/dma_link_list) | DMA | 58x | **無對應測項** | Ch9 DMA | 待實測 | DMA sheet 無 link-list 測項；且本例僅 RT58x |
| 46 | [dma_polling](examples/peripheral/dma/dma_polling) | DMA | 58x+584 | DMA · Memory to Memory Transfer | Ch9 DMA | 待實測 |  |
| 47 | [flash](examples/peripheral/flash/flash) | Flash Controller | 58x+584 | FLASH · Flash Information / Read / Write / Erase Function | Ch8 Flash Control | 待實測 |  |
| 48 | [flash_bp_protect](examples/peripheral/flash/flash_bp_protect) | Flash Controller | 584 | FLASH · Flash Status / Security Register Control | Ch8 Flash Control | 待實測 | BP 位元在 status reg |
| 49 | [flash_dataset](examples/peripheral/flash/flash_dataset) | Flash Controller | 58x+584 | **無對應測項** | Ch8 Flash Control | 待實測 | EnhancedFlashDataset 是 SDK 元件，非 IC 驗證項 |
| 50 | [flash_wnbytes](examples/peripheral/flash/flash_wnbytes) | Flash Controller | 58x+584 | FLASH · Read Function / Write Function | Ch8 Flash Control | 待實測 | byte/page 讀寫 |
| 51 | [gpio-input](examples/peripheral/gpio/gpio-input) | GPIO | 58x+584 | GPIO · Input | Ch17 GPIO Control | 待實測 |  |
| 52 | [gpio-interrupt](examples/peripheral/gpio/gpio-interrupt) | GPIO | 58x+584 | GPIO · Interrupt | Ch17 GPIO Control | 待實測 |  |
| 53 | [gpio-output](examples/peripheral/gpio/gpio-output) | GPIO | 58x+584 | GPIO · Output | Ch17 GPIO Control | 待實測 |  |
| 54 | [gpio-wakeup-deep-power-down](examples/peripheral/gpio/gpio-wakeup-deep-power-down) | GPIO | 584 | GPIO · Deep Power down Wakeup | Ch17 GPIO Control | 待實測 |  |
| 55 | [gpio-wakeup-deep-sleep](examples/peripheral/gpio/gpio-wakeup-deep-sleep) | GPIO | 58x+584 | GPIO · Deep Sleep Wakeup | Ch17 GPIO Control | 待實測 |  |
| 56 | [gpio-wakeup-sleep](examples/peripheral/gpio/gpio-wakeup-sleep) | GPIO | 58x+584 | GPIO · Sleep Wakeup | Ch17 GPIO Control | 待實測 |  |
| 57 | [i2c-master](examples/peripheral/i2c/i2c-master) | I2C | 58x+584 | I2CM · Status / Control / Command / Interrupt / EEPROM | Ch20 I2C Master | 待實測 | 4 個測項只有 1 個 example |
| 58 | [i2c-slave](examples/peripheral/i2c/i2c-slave) | I2C | 584 | I2CS · Status / Control / Command / Interrupt | Ch21 I2C Slave | 待實測 |  |
| 59 | [i2s-loopback](examples/peripheral/i2s/i2s-loopback) | I2S | 58x+584 | I2SM · Loopback / Xdma / Interrupt / Clock / Format | Ch23 I2S | 待實測 |  |
| 60 | [i2s-mic](examples/peripheral/i2s/i2s-mic) | I2S | 58x+584 | **無對應測項** | Ch23 I2S | 待實測 | I2SM sheet 無 MIC 輸入測項（Rx only 部分涵蓋於 Tranceiver） |
| 61 | [irm-nec](examples/peripheral/irm/irm-nec) | IRM | 584 | IRM · NEC (+ Control / Interrupt) | Ch24 IRM | 待實測 |  |
| 62 | [irm-rc6](examples/peripheral/irm/irm-rc6) | IRM | 584 | IRM · RC6 (+ Control / Interrupt) | Ch24 IRM | 待實測 |  |
| 63 | [irm-sirc](examples/peripheral/irm/irm-sirc) | IRM | 584 | IRM · SIRC (+ Control / Interrupt) | Ch24 IRM | 待實測 |  |
| 64 | [lpm_deep_sleep](examples/peripheral/lpm/lpm_deep_sleep) | Low Power Mode | 58x+584 | LPM · Deepsleep | Ch5 Low Power Modes | 待實測 |  |
| 65 | [lpm_power_down](examples/peripheral/lpm/lpm_power_down) | Low Power Mode | 584 | LPM · Deepsleep Powerdown | Ch5 Low Power Modes | 待實測 |  |
| 66 | [lpm_sleep](examples/peripheral/lpm/lpm_sleep) | Low Power Mode | 58x+584 | LPM · Sleep | Ch5 Low Power Modes | 待實測 |  |
| 67 | [sleep](examples/peripheral/lpm/sleep) | Low Power Mode | 58x+584 | LPM · Sleep | Ch5 Low Power Modes | 待實測 | ⚠ 不在 dev CI；見備註 4 |
| 68 | [otp_rand_number](examples/peripheral/otp/otp_rand_number) | OTP / PUF / TRNG | 584 | TRNG_PUF_OTP · OTP/PUF/randnumber | Ch11 PUFrt | 待實測 | ⚠ 該測項狀態 = Fail |
| 69 | [otp_read](examples/peripheral/otp/otp_read) | OTP / PUF / TRNG | 584 | TRNG_PUF_OTP · OTP/PUF/randnumber | Ch11 PUFrt | 待實測 | ⚠ 該測項狀態 = Fail |
| 70 | [otp_write](examples/peripheral/otp/otp_write) | OTP / PUF / TRNG | 584 | TRNG_PUF_OTP · OTP/PUF/randnumber | Ch11 PUFrt | 待實測 | ⚠ 該測項狀態 = Fail；OTP 不可逆 |
| 71 | [pwm_fmt0_duty](examples/peripheral/pwm/pwm_fmt0_duty) | PWM | 58x+584 | PWM · Register Mode / xDMA | Ch22 PWM | 待實測 | fmt0 = 取樣格式 0 |
| 72 | [pwm_fmt1_duty](examples/peripheral/pwm/pwm_fmt1_duty) | PWM | 58x+584 | PWM · PWM Mode / xDMA | Ch22 PWM | 待實測 | fmt1 = 取樣格式 1 |
| 73 | [pwm_multi_fmt0_duty](examples/peripheral/pwm/pwm_multi_fmt0_duty) | PWM | 58x+584 | PWM · Register Mode / Trigger | Ch22 PWM | 待實測 | 多通道 |
| 74 | [pwm_multi_fmt1_duty](examples/peripheral/pwm/pwm_multi_fmt1_duty) | PWM | 58x+584 | PWM · PWM Mode / Trigger | Ch22 PWM | 待實測 | 多通道 |
| 75 | [qspi_access_flash](examples/peripheral/qspi/qspi_access_flash) | (Q)SPI | 58x+584 | QSPI · SPI Flash | Ch19 QSPI | 待實測 |  |
| 76 | [qspi_dma_loopback](examples/peripheral/qspi/qspi_dma_loopback) | (Q)SPI | 58x+584 | QSPI · Loopback | Ch19 QSPI | 待實測 | SPI0 master ↔ SPI1 slave |
| 77 | [qspi_dma_master](examples/peripheral/qspi/qspi_dma_master) | (Q)SPI | 58x+584 | QSPI · Control / Interrupt | Ch19 QSPI | 待實測 | DMA master |
| 78 | [qspi_dma_slave](examples/peripheral/qspi/qspi_dma_slave) | (Q)SPI | 58x+584 | QSPI · Control / Interrupt | Ch19 QSPI | 待實測 | DMA slave |
| 79 | [qspi_epaper](examples/peripheral/qspi/qspi_epaper) | (Q)SPI | 584 | QSPI · LCD Function | Ch19 QSPI | 待實測 | sheet 寫 LCD，example 是 e-paper |
| 80 | [qspi_pio_master](examples/peripheral/qspi/qspi_pio_master) | (Q)SPI | 58x+584 | QSPI · Control / Status | Ch19 QSPI | 待實測 | PIO master |
| 81 | [qspi_pio_slave](examples/peripheral/qspi/qspi_pio_slave) | (Q)SPI | 58x+584 | QSPI · Control / Status | Ch19 QSPI | 待實測 | PIO slave |
| 82 | [rtc-event](examples/peripheral/rtc/rtc-event) | RTC Timer | 58x+584 | RTC · Interrupt | Ch16 RTC Timer | 待實測 |  |
| 83 | [rtc-everytime](examples/peripheral/rtc/rtc-everytime) | RTC Timer | 58x+584 | RTC · Timer / Counters | Ch16 RTC Timer | 待實測 |  |
| 84 | [rtc-matchtime](examples/peripheral/rtc/rtc-matchtime) | RTC Timer | 58x+584 | RTC · Interrupt | Ch16 RTC Timer | 待實測 | alarm 比對 |
| 85 | [rtc-wakeup-deep-sleep](examples/peripheral/rtc/rtc-wakeup-deep-sleep) | RTC Timer | 58x+584 | RTC · Deep Sleep Wakeup | Ch16 RTC Timer | 待實測 |  |
| 86 | [rtc-wakeup-sleep](examples/peripheral/rtc/rtc-wakeup-sleep) | RTC Timer | 58x+584 | RTC · Sleep Wakeup | Ch16 RTC Timer | 待實測 |  |
| 87 | [sadc](examples/peripheral/sadc/sadc) | SADC | 58x+584 | SADC · Sample Rate / Scan Mode / Ouput Resolution / Oversample Rate / xDMA / Interrupt | Ch25 AUX ADC | 待實測 | 1 個 example 對 6~7 個測項 |
| 88 | [sleep](examples/peripheral/sleep) | Low Power Mode | 58x+584 | LPM · Sleep | Ch5 Low Power Modes | 待實測 | ⚠ **孤兒目錄**，永遠不會被編譯；見備註 4 |
| 89 | [slow-timer-freerun-downcount](examples/peripheral/slow-timer/slow-timer-freerun-downcount) | 32K Timer | 58x+584 | 32K_Timer · Free Run | Ch14 Slow-clock Timer | 待實測 | 下數 |
| 90 | [slow-timer-freerun-upcount](examples/peripheral/slow-timer/slow-timer-freerun-upcount) | 32K Timer | 584 | 32K_Timer · Free Run | Ch14 Slow-clock Timer | 待實測 | 上數 |
| 91 | [slow-timer-oneshot](examples/peripheral/slow-timer/slow-timer-oneshot) | 32K Timer | 584 | **無對應測項** | Ch14 Slow-clock Timer | 待實測 | 32K_Timer sheet 無 one-shot 測項 |
| 92 | [slow-timer-periodic](examples/peripheral/slow-timer/slow-timer-periodic) | 32K Timer | 58x+584 | 32K_Timer · Periodic | Ch14 Slow-clock Timer | 待實測 |  |
| 93 | [slow-timer-repeat-delay](examples/peripheral/slow-timer/slow-timer-repeat-delay) | 32K Timer | 58x+584 | 32K_Timer · Repeat Delay Interrupt / Status Interval | Ch14 Slow-clock Timer | 待實測 |  |
| 94 | [slow-timer-user-prescale](examples/peripheral/slow-timer/slow-timer-user-prescale) | 32K Timer | 584 | 32K_Timer · Prescale | Ch14 Slow-clock Timer | 待實測 |  |
| 95 | [swi](examples/peripheral/swi/swi) | Software IRQ | 58x+584 | SWI · SW IRQ Enable / Interrupt / Data | Ch12 Software IRQ | 待實測 | 1 個 example 對 3 個測項 |
| 96 | [timer-capture](examples/peripheral/timer/timer-capture) | Timer | 584 | Timer · Capture / Capture Interrupt Status | Ch13 Timer | 待實測 |  |
| 97 | [timer-freerun-downcount](examples/peripheral/timer/timer-freerun-downcount) | Timer | 58x+584 | Timer · Free Run | Ch13 Timer | 待實測 | 下數 |
| 98 | [timer-freerun-upcount](examples/peripheral/timer/timer-freerun-upcount) | Timer | 584 | Timer · Free Run | Ch13 Timer | 待實測 | 上數 |
| 99 | [timer-oneshot](examples/peripheral/timer/timer-oneshot) | Timer | 584 | **無對應測項** | Ch13 Timer | 待實測 | Timer sheet 無 one-shot 測項 |
| 100 | [timer-periodic](examples/peripheral/timer/timer-periodic) | Timer | 58x+584 | Timer · Periodic / Interrupt Interval | Ch13 Timer | 待實測 |  |
| 101 | [timer-pwm](examples/peripheral/timer/timer-pwm) | Timer | 584 | Timer · PWM | Ch13 Timer | 待實測 | Timer 內建 PWM 輸出，非 PWM 模組 |
| 102 | [timer-user-prescale](examples/peripheral/timer/timer-user-prescale) | Timer | 584 | Timer · Prescale | Ch13 Timer | 待實測 |  |
| 103 | [uart1_dma_it_loopback](examples/peripheral/uart/uart1_dma_it_loopback) | UART | 58x+584 | UART · xDMA / Interrupt | Ch18 UART | 待實測 |  |
| 104 | [uart1_dma_loopback](examples/peripheral/uart/uart1_dma_loopback) | UART | 58x+584 | UART · xDMA | Ch18 UART | 待實測 |  |
| 105 | [uart1_hwflow_loopback](examples/peripheral/uart/uart1_hwflow_loopback) | UART | 58x+584 | UART · (UART1/UART2) HW Flow Control | Ch18 UART | 待實測 |  |
| 106 | [uart1_loopback](examples/peripheral/uart/uart1_loopback) | UART | 58x+584 | UART · BuadRate / Interrupt | Ch18 UART | 待實測 |  |
| 107 | [uart1_uart2_loopback](examples/peripheral/uart/uart1_uart2_loopback) | UART | 58x+584 | UART · xDMA (UART1_TX↔UART2_RX) | Ch18 UART | 待實測 |  |
| 108 | [wdt-interrupt](examples/peripheral/wdt/wdt-interrupt) | Watchdog Timer | 58x+584 | WDT · Interrupt | Ch15 Watchdog Timer | 待實測 |  |
| 109 | [wdt-reset](examples/peripheral/wdt/wdt-reset) | Watchdog Timer | 58x+584 | WDT · Reset | Ch15 Watchdog Timer | 待實測 |  |

---

## 2. 有周邊、但沒有 example

以 RM 章節 + driver 原始檔為母體，找出 `examples/peripheral/` 完全沒覆蓋的部分。

| RM 章 | 周邊 / 功能 | 有無 driver | 有無驗證測項 | 有無 example |
|---|---|---|---|---|
| Ch3 | Clocks（時脈來源切換、PLL、RC/XTAL 校正） | `sysctrl.c` (1215 行) | 無 | **無** |
| Ch4 | Resets（reset cause 判讀） | `sysfun.c` | 無（RTC sheet 內附帶一項） | **無** |
| Ch6 | Security Controller（Flash/RAM/周邊 secure 分區） | `sysctrl.c` 部分 | 無（DMA sheet 提到 secure world） | **無** |
| Ch7 | COMM Subsys（M0+ 子系統、host interface） | `comm_subsystem_drv.c` | 無 | **無** |
| Ch10 | xDMA（周邊專用 DMA） | 各周邊 driver 內建 | 散在 I2SM/SADC/PWM/UART sheet | **無獨立 example** |
| Ch11 | PUF / TRNG（PUFrt 中 OTP 以外的部分） | `trng.c`, `hosal_trng.c` | TRNG_PUF_OTP（**Fail**） | **無**（只有 otp_* 三個） |
| Ch29 | Cache Controller | 無獨立 driver | 無 | **無** |
| — | PinMux（GPIO 多工切換） | `gpio.c` / `sysctrl.c` | PinMux · PinMux | **無** |
| — | DWT（cycle counter） | `dwt.c` (173 行) | 無 | **無** |

**小結**：RM 30 章中，`Ch1 Introduction`、`Ch2 Host CPU`、`Ch30 Revision History` 屬說明性章節；
其餘 27 章有 example 覆蓋的是 20 章，**7 章（Clocks / Resets / Security Controller / COMM Subsys / xDMA / PUFrt 之 PUF+TRNG / Cache）完全沒有 example**。

---

## 3. 有測項、但沒有對應 example

驗證表 120 項中，找不到單一 example 直接對應的（多半是「一個 example 涵蓋多測項」或「完全沒測到」）。

| Sheet | 測項 | 現況 |
|---|---|---|
| Timer | Status Interval | `timer-periodic` 只驗中斷，未驗「關中斷、輪詢 status」路徑 |
| 32K_Timer | Status Interval | 同上，`slow-timer-repeat-delay` 部分涵蓋 |
| WDT | Lock | 無 example（暫存器 lock/unlock） |
| WDT | Kick | 無獨立 example（寫 0xA5A5 reload） |
| RTC | Counters | 無獨立 example（ms/s/min/hour/day/month/year 全欄位讀寫） |
| RTC | Clock Division | 無 example（divisor 31999 → 1Hz） |
| RTC | Watchdog Reset / System Reset | 無 example（reset 後 counter 不清除） |
| RTC | Scratchpad | 無 example（scratchpad 暫存器讀寫） |
| GPIO | Debounce | 無 example |
| I2CM | Status / Control / Command / Interrupt | 4 項全靠 `i2c-master` 一個 example |
| I2CM | EEPROM | `i2c-master` 是否接 EEPROM 待確認 |
| I2CS | Status / Control / Command / Interrupt | 4 項全靠 `i2c-slave` 一個 example |
| I2SM | Clock / Tranceiver / Format / Sample / Interrupt | 5 項全靠 `i2s-loopback` |
| SADC | Test Mode | 無 example（SADC 測試值注入） |
| SADC | Sample Rate / Scan Mode / Resolution / Oversample / xDMA / Interrupt | 6 項全靠 `sadc` 一個 example |
| PWM | Clock Divider / Interrupt / xDMA / Sequence Controller Mode | 4 項無專屬 example |
| FLASH | Operation Pattern Match | 無 example（寫 RABT 0x52415254 啟動） |
| FLASH | Flash Suspend | 無 example（program/erase 中存取 flash） |
| FLASH | Internal Flash | 無 example（程式寫入內部 flash） |
| UART | Line Status | 無 example（DR / overrun / parity / framing / break） |
| UART | TRx FIFO Flush | 無 example（FIFO reset、trigger level） |
| UART | Sleep Wake up | 無 example（含低速時脈 921.6K / 38.4K 收資料喚醒） |
| QSPI | Manual CS | 無 example |
| QSPI | Interrupt / Status | 散在既有 example 中，無專屬驗證 |
| LPM | （3 項） | `lpm_*` 三個 example 對應，但 `lpm/sleep` 與 `sleep/sleep` 疑似重複 |
| PinMux | PinMux | 無 example |
| TRNG_PUF_OTP | OTP/PUF/randnumber | **唯一 Fail 項**；example 只有 otp_read/write/rand_number |

---

## 4. 有 example、但沒有對應測項

| Project | 周邊 | 原因 |
|---|---|---|
| [aux-deepsleep-level](examples/peripheral/aux-comp/aux-deepsleep-level) | AUX Comparator | AUX_BOD sheet 無 level 觸發測項 |
| [bod-deepsleep-level](examples/peripheral/bod-comp/bod-deepsleep-level) | BOD Comparator | AUX_BOD sheet 無 level 觸發測項 |
| [comparator](examples/peripheral/comp/comparator) | Comparator (RT58x) | RT584 驗證表無 RT58x comparator；AUX_BOD sheet 僅涵蓋 584 |
| [crypto_hkdf](examples/peripheral/crypto/crypto_hkdf) | Crypto Engine | CRYPTO sheet 無 HKDF 測項 |
| [crypto_hmac](examples/peripheral/crypto/crypto_hmac) | Crypto Engine | CRYPTO sheet 只有 HMAC_DRBG，無純 HMAC |
| [dma_link_list](examples/peripheral/dma/dma_link_list) | DMA | DMA sheet 無 link-list 測項；且本例僅 RT58x |
| [flash_dataset](examples/peripheral/flash/flash_dataset) | Flash Controller | EnhancedFlashDataset 是 SDK 元件，非 IC 驗證項 |
| [i2s-mic](examples/peripheral/i2s/i2s-mic) | I2S | I2SM sheet 無 MIC 輸入測項（Rx only 部分涵蓋於 Tranceiver） |
| [slow-timer-oneshot](examples/peripheral/slow-timer/slow-timer-oneshot) | 32K Timer | 32K_Timer sheet 無 one-shot 測項 |
| [timer-oneshot](examples/peripheral/timer/timer-oneshot) | Timer | Timer sheet 無 one-shot 測項 |

---

## 5. 備註 / 已知問題

1. **驗證表只涵蓋 RT584 MPA IC**。RT581/582/583 的 67 個 chip-config 在這份表裡沒有對應母體，
   `comp/comparator`、`dma/dma_link_list` 這兩個 RT58x 專屬 example 因此完全無測項。
2. **AUX_BOD 是同一張 sheet**，5 個測項沒有區分 AUX Comparator 與 BOD Comparator，
   但 example 有 aux-comp（7 個）與 bod-comp（7 個）兩套共 14 個。測項顆粒度明顯低於 example。
3. **`*-deepsleep-level` 兩個 example（aux/bod）在驗證表找不到對應**，需確認是後加的功能還是漏測。
4. **`examples/peripheral/sleep/` 是孤兒目錄，永遠不會被編譯**（已查證）：
   - `config/build_project.config:159` → `default "sleep" if PER_SLEEP`
   - `config/examples/basic_examples.config:896` → `source "examples/peripheral/lpm/sleep/Kconfig"`
   - 根 `CMakeLists.txt:41` 只有 `examples/peripheral/lpm/${CONFIG_BUILD_PROJECT}` 這一條，
     沒有任何一行指向 `examples/peripheral/sleep`
   - 結論：`PER_SLEEP` 一律解析到 `examples/peripheral/lpm/sleep/`。
     頂層 `examples/peripheral/sleep/`（含自己的 `CMakeLists.txt`、`Kconfig`、7 個 `.config`、
     `rtos/` 與 `sleep/` 兩個子目錄）是死碼。**建議刪除或補上 CMake 掛載**。
5. **`components/platform/hosal/hosal/` 是死目錄**：只有 `CMakeLists.txt.in` 與 `CMakeLists.txt_b`，
   全 repo 無任何引用，但裡面有一整套 `Include/hosal_*.h`。屬 code review 待清理項。
6. 驗證表 Summary 記 120 項全數已測、119 Pass / 1 Fail（TRNG_PUF_OTP）。
   **這是 IC 驗證的結果，不等於 SDK example 的結果** — 兩者驗的是不同層（暫存器 vs. HOSAL API）。

---

## 6. dev CI 沒有編譯的 example

以 `.github/config/dev/<chip>/*.json` 為母體比對：
629 個 chip-config 中 **600 個在 dev CI，29 個不在**（6 個 project）。

| Project | 缺的晶片 | 說明 |
|---|---|---|
| `bod-comp/bod-sleep-counter` | 全部 4 顆 | 同組其他 6 個都在 CI，只漏這一個 → 疑似漏加 |
| `crypto/crypto_curve_c25519_2` | RT581/582/583 | 584 系列有進 CI，58x 漏加 |
| `flash/flash_bp_protect` | 全部 4 顆 | 需實體 flash 保護測試，可能刻意排除 |
| `qspi/qspi_epaper` | 全部 4 顆 | 需外接 e-paper 面板，可能刻意排除 |
| `lpm/sleep` | 全部 7 顆 | 見備註 4 |
| `sleep（頂層）` | 全部 7 顆 | 孤兒目錄，見備註 4 |

前兩項（`bod-sleep-counter`、`crypto_curve_c25519_2`）看起來是單純漏加，
補進 `.github/config/dev/<chip>/basic.json` 與 `crypto.json` 即可。

