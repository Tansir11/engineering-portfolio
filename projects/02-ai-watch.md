[← 返回项目集](../README.md)

# ESP32-S3 AI 语音对话手表

> **周期**　2026.03–2026.09
>
> **个人负责**　手表端触屏界面与语音交互功能开发。
>
> **成果**　中国国际大学生创新大赛三等奖、“汇创青春”市级二等奖、“挑战杯”校内赛二等奖。

## 项目目标

使用微雪 ESP32-S3-Touch-LCD-1.46 触屏开发板搭建手表原型，让用户通过触屏启动录音，在屏幕上查看识别文字和 AI 回复，并听到语音播报。界面使用 LVGL 与 SquareLine Studio，语音通过 STT、DeepSeek 和 TTS 三个接口串联。

## 完整交互流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant W as 手表
    participant S as STT 服务
    participant L as DeepSeek
    participant T as TTS 服务
    U->>W: 触屏发起对话并录音
    W->>S: 上传 WAV 录音
    S-->>W: 识别文本
    W-->>U: 显示识别文字
    W->>L: 提示词与用户文本
    L-->>W: 文字回复
    W-->>U: 显示回复文字
    W->>T: 提交回复文本
    T-->>W: 合成音频
    W-->>U: 通过 I²S 输出并播放
```

### 从语音到回复

1. **采集。** 由录音模块读取麦克风数据，采用 16 kHz、16 bit 单声道格式并组织 WAV 缓冲，交给 STT 接口。
2. **生成。** 把识别结果作为用户消息，与系统提示词一并发给 DeepSeek。工程里的提示词要求使用简洁中文，并限制回复篇幅，便于手表显示和朗读。
3. **呈现。** 回调更新聊天气泡，将模型返回的文字提交给 TTS，合成后的音频经 I²S 输出。

## 软件架构

```mermaid
flowchart TD
    UI["LVGL 页面与触屏事件"] --> CHAT["AI_Chat 对话调度"]
    CHAT --> REC["MIC_Recorder 录音与 WAV 缓冲"]
    REC --> STT["STT 语音识别"]
    STT --> AI["DeepSeek 文字回复"]
    AI --> TTS["TTS_Speaker 下载与播放"]
    TTS --> OUT["I²S 音频输出"]
    STT --> UI
    AI --> UI
    NET["WiFi_Manager 网络请求"] -.-> STT
    NET -.-> AI
    NET -.-> TTS
```

### 运行入口与状态机

`main.cpp` 依次初始化外设、显示、LVGL、界面、网络以及录音和对话模块。主循环调用 `Lvgl_Loop()` 和 `AI_Chat_Loop()`；独立的 `Driver_Loop` 任务处理电源键、时钟、电池等外设。

对话状态按 `IDLE → RECORDING → STT_WAITING → AI_WAITING → TTS → DONE → IDLE` 推进。录音阶段更新采样，识别和生成阶段执行网络请求，播报阶段调用 `TTS_Update()`，播报完成后回到待机。

## 个人工作与实现思路

### 触屏交互

使用 LVGL 和 SquareLine Studio 搭建 12 个页面，编写页面切换与对话显示逻辑。生成页面集中在 `squareline_ui/`，聊天界面通过回调接收用户文字、AI 回复和运行状态。中文显示由字体资源和 `ui_FontLoader.cpp` 配合。

### 对话流程管理

一次对话会依次占用麦克风、网络和音频输出。将每个阶段放进对话状态机，根据识别结果、请求结果和播报状态决定下一步；请求或录音失败时给出状态反馈并退出当前流程。录音设置最大时长，TTS 的连接、响应头、音频下载和播放阶段分别处理超时。

### 录音缓冲与音频输出

录音数据由 `MIC_Recorder.cpp` 分配到 PSRAM。云端返回文字后，`TTS_Speaker.cpp` 负责获取合成音频并推进 I²S 播放，让文本生成和音频输出形成完整交互。

## 原工程目录

```text
watch/
├── platformio.ini          # PlatformIO 与 Arduino 构建配置
├── lv_conf.h               # LVGL 配置
├── sdkconfig.defaults      # 底层配置
├── include/                # 模块头文件
└── src/
    ├── main.cpp            # 初始化、主循环和外设任务
    ├── AI_Chat.cpp         # 对话状态机、STT 和 DeepSeek 请求
    ├── MIC_Recorder.cpp    # 麦克风采样与 WAV 缓冲
    ├── TTS_Speaker.cpp     # TTS 获取、超时和 I²S 播放
    ├── WiFi_Manager.cpp    # 网络连接和 HTTP 请求封装
    ├── Emotion_Manager.cpp # 情绪分析
    ├── LVGL_Driver.cpp     # LVGL 显示与触摸桥接
    ├── Display_SPD2010.cpp / Touch_SPD2010.cpp
    ├── ui_FontLoader.cpp   # 中文字体加载
    └── squareline_ui/
        ├── ui.c / ui.h     # UI 初始化及对象声明
        ├── screens/        # 页面定义
        ├── fonts/          # 字体
        └── images/         # 图片资源
```

上表列出与界面及语音对话直接相关的模块，原工程还包括电池、RTC、陀螺仪等外设模块。

## 技术与成果

| 项目 | 实现 |
| :-- | :-- |
| 硬件 | 微雪 ESP32-S3 触屏开发板 |
| 开发 | C/C++、PlatformIO、Arduino |
| 交互 | LVGL、SquareLine Studio、12 个页面 |
| 语音 | 16 kHz 录音、STT、DeepSeek、TTS、I²S |
| 缓冲 | PSRAM 录音缓冲 |

已完成触摸界面与 AI 语音对话演示。本文按工程中的功能与代码组织说明实现。
