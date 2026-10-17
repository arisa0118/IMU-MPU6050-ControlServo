# MPU6050 DMP 雙軸伺服雲台（Yaw + Pitch）

本專案使用 **Arduino、MPU6050 六軸慣性感測器與兩顆伺服馬達**，透過 MPU6050 內建的 Digital Motion Processor（DMP）取得姿態角度，讓雙軸雲台依照感測器的 **Yaw（偏航）** 與 **Pitch（俯仰）** 角度移動。

> 本 README 說明的是目前提供的**原始程式**，並非已修正的版本。程式仍會等待序列埠輸入才啟動 DMP，也保留可能阻塞的 FIFO 等待迴圈。

## 功能

- 使用 MPU6050 DMP 計算 Yaw、Pitch、Roll。
- 以 Yaw 控制水平旋轉伺服馬達，以 Pitch 控制俯仰伺服馬達。
- 兩顆馬達啟動時先移動至 90°。
- 將馬達輸出限制在 **45°～135°**。
- 約每 **20 ms** 更新一次馬達控制指令（有新 DMP 資料時）。
- 使用一階平滑濾波降低角度跳動。
- 透過序列監控視窗輸出 `ypr` 姿態角度，鮑率為 **115200**。
- 使用 Arduino 板載 LED 指示 DMP 資料處理活動。

## 所需硬體

- Arduino Uno 或 Nano（以下接線以傳統 ATmega328P 版本為例）
- MPU6050 感測器模組（例如 GY-521）
- 兩顆位置控制型伺服馬達
- 適合伺服馬達負載的外部穩壓電源
- 杜邦線及雙軸雲台機構

## 接線

### MPU6050 與 Arduino

| MPU6050 | Arduino Uno / 傳統 Nano | 用途 |
|---|---|---|
| VCC | 依模組規格接 3.3V 或 5V | 供電 |
| GND | GND | 共地 |
| SDA | A4 | I2C 資料 |
| SCL | A5 | I2C 時脈 |
| INT | D2 | DMP 資料中斷 |
| AD0 | GND（或保持模組預設低電位） | I2C 位址 `0x68` |

> MPU6050 晶片本身並非 5V 邏輯元件；是否可接 5V 取決於模組上的穩壓與電位轉換設計，請確認模組規格。若 AD0 接高電位，I2C 位址可能改為 `0x69`，需同步修改程式。

### 伺服馬達與 Arduino

| 功能 | 程式物件 | Arduino 訊號腳位 | 初始角度 |
|---|---|---|---|
| Pitch（上下俯仰） | `servoPitch` | **D6** | 90° |
| Yaw（左右水平旋轉） | `servoYaw` | **D7** | 90° |

伺服馬達的電源正極接外部穩壓電源正極，電源負極接外部電源 GND；**外部電源 GND 必須與 Arduino GND 共地**。不要直接用 Arduino 5V 腳位供應兩顆伺服馬達的工作電流。

## 軟體需求

- Arduino IDE
- Arduino `Servo` 函式庫
- Arduino `Wire` 函式庫
- Jeff Rowberg 的 [`I2Cdevlib`](https://github.com/jrowberg/i2cdevlib)，包含相容的 `I2Cdev.h` 與 `MPU6050_6Axis_MotionApps20.h`

請確保 `I2Cdev` 與 `MPU6050` 函式庫版本彼此相容。不同版本的 DMP 範例可能使用不同標頭檔或 API。

## 執行方式（原始程式）

1. 完成接線，先確認 MPU6050 可以正常通訊。
2. 在 Arduino IDE 選擇正確的開發板與序列埠，上傳 `.ino` 程式。
3. 開啟 **Serial Monitor**，設定鮑率 **115200**。
4. 程式顯示 `Send any character to begin DMP programming and demo:` 時，輸入任意字元並傳送。
5. 若 DMP 初始化成功，會顯示 `DMP ready! Waiting for first interrupt...`。
6. 轉動 MPU6050，觀察兩顆伺服馬達是否跟隨 Yaw 與 Pitch 移動。

**注意：** 原始程式包含 `while (!Serial.available())`，因此不會在完全沒有序列埠輸入的情況下自動開始 DMP 初始化；如果要通電即啟動，需移除該等待區塊。

## 控制原理

MPU6050 DMP 輸出弧度制的 `ypr[3]`：

| 索引 | 姿態 | 用途 |
|---|---|---|
| `ypr[0]` | Yaw | 控制水平旋轉 |
| `ypr[1]` | Pitch | 控制上下俯仰 |
| `ypr[2]` | Roll | 目前僅輸出至 Serial，不控制馬達 |

先將弧度轉換成角度：

```cpp
float pitchDeg = ypr[1] * 180.0 / M_PI;
float yawDeg   = ypr[0] * 180.0 / M_PI;
```

再以 90° 為馬達中心，計算目標角度：

```cpp
float targetPitch = 90 + pitchDirection * pitchDeg;
float targetYaw   = 90 + yawDirection * yawDeg;
```

目前方向設定：

```cpp
const int pitchDirection = 1;
const int yawDirection = -1;
```

方向正負號對應的是**伺服指令角度增加或減少**，實際視覺上的左右／上下方向取決於馬達安裝方式。若轉向相反，可切換 `1` 與 `-1`。

### 角度限制

```cpp
const int servoMin = 45;
const int servoMax = 135;

targetPitch = constrain(targetPitch, servoMin, servoMax);
targetYaw   = constrain(targetYaw, servoMin, servoMax);
```

### 平滑控制

```cpp
smoothPitch += 0.2 * (targetPitch - smoothPitch);
smoothYaw   += 0.2 * (targetYaw - smoothYaw);
```

這是一階指數平滑：係數越大，跟隨越快但可能較抖；係數越小，動作越平順但延遲較明顯。原始程式在取得 DMP 封包後，以 `millis()` 檢查是否已間隔至少 20 ms 才更新伺服指令。

## Serial Monitor 輸出

成功運作時，序列監控視窗會持續輸出類似：

```text
ypr    12.34    -5.67    1.23
```

依序為 **Yaw、Pitch、Roll**，單位是度。這些數字僅為格式示例，不代表實際量測結果。

## 常見問題與排除方式

### `MPU6050 connection failed`

代表 `mpu.testConnection()` 沒有通過。請先：

1. 檢查 VCC、GND、SDA、SCL 接線及模組供電規格。
2. 暫時斷開兩顆伺服馬達電源，排除供電干擾。
3. 使用 I2C Scanner 檢查是否找到 `0x68` 或 `0x69`。
4. 確認程式中的 MPU6050 I2C 位址與實際掃描結果一致。
5. 嘗試將 `Wire.setClock(400000)` 改成 `Wire.setClock(100000)`。

### `DMP Initialization failed (code 1)`

表示 DMP 初始化過程中的初始記憶體載入失敗。可能涉及 I2C 通訊、電源、函式庫版本或模組故障。若同時出現 `connection failed`，應**優先修復基本 I2C 連線**，不要先調整伺服馬達角度。

### 馬達運作一段時間後停止

原始程式包含兩處可能長時間阻塞的 FIFO 等待：

```cpp
while (!mpuInterrupt && fifoCount < packetSize) {
  // ...
}
```

```cpp
while (fifoCount < packetSize) fifoCount = mpu.getFIFOCount();
```

如果中斷遺失或資料不再產生，可能一直等待。建議後續改成非阻塞式 FIFO 讀取或加入逾時處理，並檢查 I2C 及電源穩定性。

### `FIFO overflow!`

代表 FIFO 資料緩衝區溢位。原始程式會呼叫 `mpu.resetFIFO()` 嘗試恢復。若頻繁發生，可減少序列輸出量、改善讀取邏輯，並檢查 I2C 通訊是否正常。

### Yaw 逐漸漂移

MPU6050 沒有磁力計，Yaw 無法長期維持可靠的絕對方向參考，使用一段時間後可能漂移。原始程式也**沒有記錄開機時的初始 Yaw 作為零點**。

## 已知限制

- 此程式屬於**姿態跟隨**，不是雲台防震補償控制。
- 原始程式必須收到 Serial 字元才開始初始化 DMP。
- 使用 DMP 中斷腳位 D2，接線不完整時可能停在等待狀態。
- FIFO 讀取包含阻塞式等待，長時間運作穩定性仍待改善。
- 使用原始範例的陀螺儀與加速度計 offset，應針對實際模組重新校正。
- Pitch 與 Yaw 直接使用 DMP 姿態角度，沒有做開機零點校準。
- 馬達的實際機構極限可能小於程式設定的 45°～135°，應先確認機械安全範圍。

## 程式來源與授權

本專案以 Jeff Rowberg 的 MPU6050 DMP / I2Cdevlib 範例為基礎，加入雙軸 Servo 控制、角度限制與平滑處理。

- 原始專案：<https://github.com/jrowberg/i2cdevlib>
- 原作者：Jeff Rowberg
- 原始範例及 I2Cdevlib 採 MIT License；保留原始程式碼中的著作權與授權聲明。若再散布程式，請一併保留適用的授權文件。
