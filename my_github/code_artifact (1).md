# 🚗 AI Line-Following RoboCar — 智慧巡軌車

> **AMB82-Mini × Gemma4-E2B Edge AI 智慧巡軌機器人**

<p align="center">
  <img src="images/robocar_concept.png" alt="AI Line-Following RoboCar Concept" width="750"/>
</p>

---

## 📖 Introduction（簡介）

**AI Line-Following RoboCar** 是一台基於 **Realtek AMB82-Mini** 開發板的智慧巡軌車，整合 **Google Gemma4-E2B** Edge AI 模型，實現自主循跡、交通號誌辨識、語音互動與遠端監控。

巡軌車使用 5 路紅外線感測器搭配 PID 控制演算法進行精準循跡，同時透過板載攝影機執行影像辨識、RTSP 即時串流，使用者可透過 Edge AI App 或直接語音與車輛互動。

### 🌟 主要特色

- 🔍 **視覺 + IR 融合循跡**：攝影機影像與紅外線感測器雙重定位
- 🧠 **Edge AI 號誌辨識**：即時辨識停止、轉彎、減速等交通號誌
- 🎤 **語音互動**：直接對車下達語音指令
- 📱 **遠端控制**：透過 Edge AI App + Gemma4-E2B 進行 Agentic 操控
- 📹 **即時串流**：RTSP 720p/1080p 車載攝影機影像
- 🔊 **語音播報**：播放狀態提示音效與 TTS 語音

---

## 🏗️ System Architecture（系統架構圖）

### 硬體架構圖

```
                    ┌───────────────────────┐
                    │    Edge AI App        │
                    │  (Phone / PC)         │
                    │ ┌───────────────────┐ │
                    │ │  Gemma4-E2B Model │ │
                    │ │  (Agentic Agent)  │ │
                    │ └────────┬──────────┘ │
                    └──────────┼────────────┘
                               │ WiFi (HTTP/WS)
                    ┌──────────▼────────────┐
                    │     AMB82-Mini        │
                    │  ┌────┐ ┌────┐ ┌───┐ │
                    │  │CAM │ │MIC │ │SD │ │
                    │  └──┬─┘ └──┬─┘ └─┬─┘ │
                    │     │      │     │   │
                    │  ┌──▼──────▼─────▼─┐ │
                    │  │   MCU + NPU     │ │
                    │  │ (500MHz, 0.4T)  │ │
                    │  └──┬──────┬───────┘ │
                    └─────┼──────┼─────────┘
                          │      │
              ┌───────────┤      ├───────────┐
              │           │      │           │
         ┌────▼────┐ ┌────▼──┐ ┌─▼────┐ ┌───▼────┐
         │ IR ×5   │ │Motor │ │ SPK  │ │HC-SR04│
         │Sensors  │ │Driver│ │      │ │(選配) │
         └─────────┘ └──┬───┘ └──────┘ └───────┘
                        │
                   ┌────▼────┐
                   │  2×DC   │
                   │ Motors  │
                   └─────────┘
```

### 軟體分層架構

```
┌─────────────────────────────────────────────────────┐
│                Application Layer                     │
│  ┌─────────────┐ ┌─────────────┐ ┌───────────────┐  │
│  │ Edge AI App │ │ Voice Cmd   │ │ RTSP Viewer   │  │
│  │ (Gemma4)   │ │ Interface   │ │ (VLC/Custom)  │  │
│  └──────┬──────┘ └──────┬──────┘ └───────┬───────┘  │
└─────────┼───────────────┼────────────────┼──────────┘
          │ WiFi          │ Local          │ RTSP
┌─────────▼───────────────▼────────────────▼──────────┐
│               AMB82-Mini Middleware                  │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐ ┌─────────┐ │
│  │ HTTP API │ │ Audio    │ │ Video   │ │ AI Inf. │ │
│  │ Server   │ │ Engine   │ │ Stream  │ │ Engine  │ │
│  └──────────┘ └──────────┘ └─────────┘ └─────────┘ │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│                Hardware Abstraction                   │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────────┐  │
│  │ GPIO / │ │ PWM    │ │ ADC    │ │ I2C / SPI    │  │
│  │ IRQ    │ │ Motor  │ │ IR Sen │ │ SD Card      │  │
│  └────────┘ └────────┘ └────────┘ └──────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## ⚙️ Features Detail（功能細節）

### 1. 自動循跡系統 (Line-Following System)

- **感測器**：5 路 TCRT5000 紅外線反射式感測器，排列於車頭底部
- **演算法**：PID 控制器（Proportional-Integral-Derivative）
  - 根據感測器偏移量計算誤差
  - 動態調整左右馬達轉速實現平滑轉彎
- **精度**：支援 $90^\circ$ 直角彎、S 型彎道、交叉路口

```
感測器排列：
    [S1] [S2] [S3] [S4] [S5]
         ───黑線軌道───
         
偏移計算：
  error = (-2*S1) + (-1*S2) + (0*S3) + (1*S4) + (2*S5)
  
PID輸出：
  output = Kp*error + Ki*integral(error) + Kd*(d_error/dt)
```

### 2. AI 號誌辨識 (Traffic Sign Recognition)

- **模型**：YOLOv4-Tiny / YOLOv7-Tiny (TFLite 量化版)
- **支援號誌**：
  - 🔴 紅燈 → 停車
  - 🟢 綠燈 → 通行
  - ⬅️ 左轉標誌 → 路口左轉
  - ➡️ 右轉標誌 → 路口右轉
  - 🛑 停止標誌 → 停車 3 秒
  - ⚠️ 減速標誌 → 降速 50%
- **推論速度**：~10 FPS ($320 \times 320$ input, AMB82-Mini NPU)

### 3. 語音互動 (Voice Interaction)

| 語音命令 | 動作 |
|---------|------|
| 「開始」/「Start」 | 開始循跡 |
| 「停止」/「Stop」 | 停車 |
| 「加速」/「Faster」 | 提升基礎速度 |
| 「減速」/「Slower」 | 降低基礎速度 |
| 「狀態」/「Status」 | 語音播報目前圈數、速度、電量 |
| 「拍照」/「Capture」 | 拍攝當前畫面並儲存至 SD 卡 |

### 4. RTSP 即時串流 (Live Video Streaming)

- **協議**：RTSP over WiFi
- **解析度**：720p ($1280 \times 720$) / 1080p ($1920 \times 1080$) 可選
- **編碼**：H.264 硬體編碼
- **檢視方式**：VLC Player / 自製 App 內嵌播放器
- **串流地址**：`rtsp://<AMB82-IP>:554/stream`

### 5. Edge AI Agentic 控制 (Gemma4-E2B Integration)

透過手機或 PC 上運行的 Gemma4-E2B 模型，使用者可以用自然語言下達複雜指令：

```
使用者: 「跑兩圈，如果看到紅色障礙物就停下來拍照」

Gemma4-E2B 解析:
{
  "actions": [
    {"type": "set_laps", "count": 2},
    {"type": "enable_detection", "target": "red_obstacle"},
    {"type": "on_detect", "do": ["stop", "capture_photo"]},
    {"type": "start_tracking"}
  ]
}
```

---

## 📌 Pin Configuration（腳位配置）

| AMB82-Mini Pin | 功能 | 連接元件 |
|---------------|------|---------|
| GPIO 0 (PWM) | 左馬達速度 | L298N ENA |
| GPIO 1 | 左馬達方向 IN1 | L298N IN1 |
| GPIO 2 | 左馬達方向 IN2 | L298N IN2 |
| GPIO 3 (PWM) | 右馬達速度 | L298N ENB |
| GPIO 4 | 右馬達方向 IN3 | L298N IN3 |
| GPIO 5 | 右馬達方向 IN4 | L298N IN4 |
| ADC 0 | IR 感測器 S1 | TCRT5000 #1 |
| ADC 1 | IR 感測器 S2 | TCRT5000 #2 |
| ADC 2 | IR 感測器 S3 | TCRT5000 #3 |
| ADC 3 | IR 感測器 S4 | TCRT5000 #4 |
| ADC 4 | IR 感測器 S5 | TCRT5000 #5 |
| GPIO 6 | 超音波 Trig | HC-SR04 |
| GPIO 7 | 超音波 Echo | HC-SR04 |
| Camera | 攝影機模組 | JXF37 |
| MIC | 內建麥克風 | — |
| SD Card | SD 卡槽 | MicroSD |

> ⚠️ 實際腳位請依 AMB82-Mini 官方 pinout 文件調整

---

## 🛠️ Development Environment（開發環境）

- **IDE**：Arduino IDE 2.x
- **Board Package**：Ameba Pro2 (AMB82-Mini)
- **AI Framework**：TensorFlow Lite Micro
- **Edge AI Model**：Google Gemma4-E2B (手機/PC 端)
- **Programming Language**：C/C++ (Firmware), Python (AI App)
- **Version Control**：Git + GitHub

---

## 🚀 Getting Started（快速開始）

### 1. 硬體組裝

```bash
# 底盤組裝步驟
1. 安裝 TT 馬達至底盤
2. 固定萬向輪
3. 安裝 AMB82-Mini 於底盤上層
4. 連接 L298N 馬達驅動板
5. 安裝 5 路 IR 感測器模組（距地面 1~2cm）
6. 接線（參照 Pin Configuration）
7. 安裝電池座與穩壓模組
```

### 2. 韌體燒錄

```bash
# 安裝 Arduino IDE Board Package
# Arduino IDE → Preferences → Additional Board Manager URLs:
# https://github.com/nicholasgodfreychua/RealtekAmebaArduino/raw/master/package_realtek_amebapro2_index.json

# 選擇 Board: AMB82-Mini (RTL8735B)
# 上傳 firmware/main.ino
```

### 3. 執行

```bash
# 1. 插入 SD 卡（含 AI 模型與音效）
# 2. 開機
# 3. 連接同一 WiFi 網路
# 4. 開啟 Edge AI App 或使用 VLC 連線 RTSP
# 5. 語音或 App 指令啟動巡軌
```

---

## 📁 Project Structure（專案結構）

```
AI-LineFollower-RoboCar/
├── README.md                    # 本文件
├── PROPOSAL.md                  # 計畫書
├── firmware/                    # AMB82-Mini 韌體
│   ├── main.ino                # 主程式
│   ├── line_tracker.h          # 循跡模組
│   ├── motor_control.h         # 馬達控制
│   ├── camera_stream.h         # 攝影機串流
│   ├── voice_cmd.h             # 語音命令
│   ├── ai_inference.h          # AI 推論
│   └── wifi_api.h              # WiFi API Server
├── app/                         # Edge AI App (Python)
│   ├── gemma_agent.py          # Gemma4-E2B Agent
│   ├── robot_api.py            # 車輛 API Client
│   └── ui.py                   # 使用者介面
├── models/                      # AI 模型
│   ├── traffic_sign.tflite     # 號誌辨識模型
│   └── keyword_spotting.tflite # 關鍵字辨識模型
├── audio/                       # 音效檔案
│   ├── start.mp3
│   ├── stop.mp3
│   ├── lap_complete.mp3
│   └── sign_detected.mp3
├── images/                      # 圖片資源
│   └── robocar_concept.png     # 概念圖
├── docs/                        # 文件
│   ├── wiring_diagram.png      # 接線圖
│   └── pinout_reference.md     # 腳位參考
└── tests/                       # 測試
    ├── test_motor.ino           # 馬達測試
    ├── test_ir_sensor.ino       # IR 感測器測試
    └── test_camera.ino          # 攝影機測試
```

---

## 📊 Demo（展示）

### 預期展示內容
1. 📹 巡軌車在黑線軌道上自動循跡的影片
2. 🚦 號誌辨識 → 自動停車/轉彎的演示
3. 🎤 語音命令控制啟動/停止
4. 📱 Edge AI App 遠端操控與 RTSP 串流畫面
5. 📊 PID 參數調整對循跡效果的比較

---

## 📝 License

MIT License

---

## 🙏 Acknowledgments

- [Realtek Ameba IoT](https://www.amebaiot.com/) — AMB82-Mini 開發板
- [Google AI](https://ai.google.dev/) — Gemma4-E2B Edge AI Model
- [rkuo2000/AI-plush-giraffe](https://github.com/rkuo2000/AI-plush-giraffe) — 範例計畫參考
- 老師的指導與課程資源