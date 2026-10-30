# 🚗 AI Line-Following RoboCar — 智慧巡軌車計畫書

> **使用 AMB82-Mini 晶片** 🔧

## 📋 Project Overview（計畫概述）

本計畫旨在設計一台搭載 **Realtek AMB82-Mini** 微控制器與 **Google Gemma4-E2B Edge AI** 模型的智慧巡軌車（AI Line-Following RoboCar）。巡軌車透過攝影機擷取路面影像，結合紅外線感測器陣列實現高精度循跡；同時整合語音互動、物件辨識與遠端操控功能，打造一台具備 **Edge AI 推論能力** 的自主巡軌機器人。

### 🎯 核心目標
1. **精準循跡**：利用 IR 感測器 + 攝影機視覺融合，實現穩定的軌道跟隨
2. **智慧辨識**：透過 Edge AI 辨識交通號誌（停止、轉彎、減速等），自主做出反應
3. **語音互動**：支援語音命令控制（啟動、停止、報告狀態）
4. **遠端監控**：透過 RTSP 即時串流，在手機或電腦上監看車輛視角

---

## 🏗️ System Architecture（系統架構）

### 硬體架構

```
┌─────────────────────────────────────────────────┐
│                AMB82-Mini (主控制器)               │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Camera   │  │ Mic/     │  │  WiFi/BLE     │  │
│  │ (JXF37)  │  │ Speaker  │  │  2.4G/5G      │  │
│  └────┬─────┘  └────┬─────┘  └───────┬───────┘  │
│       │              │                │          │
│  ┌────▼──────────────▼────────────────▼───────┐  │
│  │         Arm v8M MCU (500MHz)               │  │
│  │         NPU 0.4 TOPS                       │  │
│  │         128MB DDR2                         │  │
│  └────┬──────────┬──────────┬─────────────────┘  │
│       │          │          │                    │
└───────┼──────────┼──────────┼────────────────────┘
        │          │          │
   ┌────▼───┐ ┌────▼───┐ ┌───▼────────┐
   │IR Line │ │Motor   │ │ SD Card    │
   │Sensor  │ │Driver  │ │ (Models/   │
   │Array   │ │(L298N) │ │  Audio)    │
   └────────┘ └────┬───┘ └────────────┘
                   │
              ┌────▼────┐
              │ 2 x DC  │
              │ Motors   │
              │ + Wheels │
              └─────────┘
```

### 軟體架構

```
┌──────────────────────────────────────────┐
│           Edge AI App (Phone/PC)          │
│  ┌──────────────────────────────────────┐│
│  │  Gemma4-E2B Model (Agentic Agent)   ││
│  │  - Text/Voice → Function Call       ││
│  │  - Image Understanding              ││
│  │  - Decision Making                  ││
│  └──────────────┬───────────────────────┘│
└─────────────────┼────────────────────────┘
                  │ WiFi / BLE
┌─────────────────▼────────────────────────┐
│          AMB82-Mini Firmware              │
│  ┌────────────┐ ┌──────────────────────┐ │
│  │ Line Track │ │ RTSP Video Stream    │ │
│  │ Controller │ │ (720p/1080p)         │ │
│  ├────────────┤ ├──────────────────────┤ │
│  │ Motor PWM  │ │ Audio I/O (TTS/STT) │ │
│  │ Controller │ │ MP3 Playback         │ │
│  ├────────────┤ ├──────────────────────┤ │
│  │ IR Sensor  │ │ AI Inference         │ │
│  │ Reader     │ │ (TFLite/YOLO)        │ │
│  └────────────┘ └──────────────────────┘ │
└──────────────────────────────────────────┘
```

---

## 🤖 Interactive Modes（互動模式比較）

### Mode 1：Edge AI App + Gemma4-E2B 遠端互動

| 項目 | 說明 |
|------|------|
| **操作方式** | 使用者透過手機/電腦上的 Edge AI App，以文字或語音發出指令 |
| **AI 處理** | Gemma4-E2B 在手機/PC 端運行，負責理解語意並產生 function call |
| **通訊方式** | WiFi (HTTP REST API / WebSocket) 連接 AMB82-Mini |
| **延遲** | 中等（含 AI 推論 + 網路傳輸） |
| **適用場景** | 複雜指令（「巡完第三圈後停下來拍照」）、遠端監控、教學展示 |
| **優勢** | 可處理複雜語意、支援多模態輸入、可顯示即時影像 |

### Mode 2：直接對車說話（On-Device 語音互動）

| 項目 | 說明 |
|------|------|
| **操作方式** | 使用者直接對巡軌車說話（例如：「開始」「停」「轉彎」） |
| **AI 處理** | AMB82-Mini 內建麥克風收音 → 簡易關鍵字辨識 or 傳送至 Edge AI 處理 |
| **通訊方式** | 本地處理（關鍵字偵測）或 WiFi 傳送音檔 |
| **延遲** | 低（關鍵字偵測）/ 中（需傳送至 Edge AI） |
| **適用場景** | 快速命令（啟動/停止/加速/減速）、免持操作 |
| **優勢** | 免持手機、直覺操作、低延遲反應 |

---

## 🔧 Hardware Components（硬體元件清單）

| 元件 | 型號/規格 | 用途 |
|------|-----------|------|
| 主控板 | **AMB82-Mini** (RTL8735B) | 主控制器，AI推論、影像擷取、WiFi/BLE通訊 |
| 攝影機 | JXF37 (1080p) | 路面拍攝、交通號誌辨識 |
| IR 感測器 | TCRT5000 x 5 (五路循跡) | 紅外線循跡感測 |
| 馬達驅動 | L298N / TB6612FNG | 雙輪直流馬達驅動 |
| 直流馬達 | TT Motor x 2 | 驅動輪 |
| 底盤 | 二輪智慧車底盤 | 車體結構（含萬向輪） |
| 電源 | 18650 鋰電池 x 2 + 穩壓模組 | 供電 |
| SD 卡 | MicroSD 16GB+ | 存放 AI 模型、音效檔案 |
| 喇叭 | 小型 8$\Omega$ 喇叭 | 播放語音提示、MP3 音效 |
| 麥克風 | 板載麥克風 (AMB82-Mini 內建) | 語音命令收音 |
| 超音波感測器 | HC-SR04 (選配) | 前方障礙物偵測 |

---

## 📦 Feature List（功能清單）

### ✅ 核心功能（必做）

- [x] **自動循跡**：5 路 IR 感測器偵測黑線軌道，PID 控制演算法實現平穩跟隨
- [x] **攝影機即時串流**：RTSP 720p 影像串流至手機/電腦
- [x] **交通號誌辨識**：利用 TFLite 模型辨識紅燈停/綠燈行/轉彎標誌
- [x] **WiFi 遠端控制**：HTTP API 提供 start/stop/speed/direction 控制端點
- [x] **語音命令**：支援「開始」「停止」「加速」「減速」基本關鍵字辨識

### 🎯 進階功能（選做）

- [ ] **Edge AI Agentic 控制**：整合 Gemma4-E2B 模型，支援自然語言指令解析
- [ ] **路口決策**：在 T 字路口或十字路口自動選擇方向（可由語音或 AI 指定）
- [ ] **圈數計數**：自動記錄巡軌圈數，達到指定圈數後自動停車
- [ ] **障礙物閃避**：超音波感測器偵測前方障礙物，自動減速或停車
- [ ] **語音播報**：巡軌車透過喇叭播報目前狀態（如「已完成第 2 圈」「偵測到停止標誌」）
- [ ] **競速模式**：記錄單圈最快時間，PID 參數動態調整以最佳化速度

---

## 🔄 System Workflow（系統工作流程）

### 巡軌車啟動流程

```
1. 開機 → AMB82-Mini 初始化
2. 載入 SD 卡中的 AI 模型與音效檔
3. 連接 WiFi → 啟動 RTSP Server
4. 等待啟動指令（語音 / App / 按鈕）
5. 進入巡軌模式：
   a. IR 感測器讀取 → PID 計算 → 馬達 PWM 輸出
   b. 攝影機每 N 幀進行號誌辨識
   c. 偵測到號誌 → 執行對應動作（停車/轉彎/減速）
   d. 語音/App 命令即時中斷處理
6. 收到停止指令 → 減速停車 → 語音播報完成
```

### 與 Edge AI App 互動流程

```
使用者: 「幫我跑三圈然後停下來」
    ↓
Gemma4-E2B (手機端): 
  解析意圖 → function_call: {action: "run_laps", count: 3, then: "stop"}
    ↓
WiFi API → AMB82-Mini:
  設定圈數=3 → 開始巡軌 → 計數 → 到達3圈 → 停車
    ↓
AMB82-Mini → App:
  回報：「已完成3圈，用時45秒，已停車」
```

---

## 📅 Development Timeline（開發時程）

| 週次 | 工作項目 | 預期成果 |
|------|---------|---------|
| Week 1-2 | 硬體組裝 & 基礎測試 | 底盤組裝完成、馬達可正反轉 |
| Week 3-4 | IR 循跡功能開發 | 基本循跡成功（5路感測器 + PID） |
| Week 5-6 | 攝影機串流 & 號誌辨識 | RTSP 串流正常、可辨識基本號誌 |
| Week 7-8 | WiFi 遠端控制 & 語音命令 | App 可遠端控制、語音關鍵字可用 |
| Week 9-10 | Edge AI 整合 & 進階功能 | Gemma4-E2B 整合、Agentic 控制 |
| Week 11-12 | 系統測試 & 優化 & 展示準備 | 全功能整合測試、展示影片製作 |

---

## 📚 References（參考資料）

1. [AMB82-Mini 官方文件 (Ameba Arduino)](https://www.amebaiot.com/en/amebapro2-amb82-mini-arduino-getting-started/)
2. [Google Gemma4-E2B Model Card](https://ai.google.dev/gemma)
3. [TensorFlow Lite for Microcontrollers](https://www.tensorflow.org/lite/microcontrollers)
4. [PID 控制演算法教學](https://en.wikipedia.org/wiki/PID_controller)
5. [RTSP Streaming with AMB82-Mini](https://www.amebaiot.com/en/amebapro2-arduino-multimedia-rtsp/)
6. [rkuo2000/AI-plush-giraffe 範例](https://github.com/rkuo2000/AI-plush-giraffe)