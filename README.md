# 五子棋 · Gobang

> Java 17 + JavaFX 21 桌面五子棋。三档 AI 人机对战、双人同屏、**远程联机**（房间码 / 局域网 / 直连）。
>
> 除了棋本身，它花了大量力气在**「手感」**上：猜先是一个真的随机动画，落子有物理下落与回弹，音效和背景音乐**全部由代码实时合成**——仓库里没有一个音频文件。

<p>
  <img alt="Java" src="https://img.shields.io/badge/Java-17-ED8B00?style=flat-square&logo=openjdk&logoColor=white">
  <img alt="JavaFX" src="https://img.shields.io/badge/JavaFX-21-FF6B35?style=flat-square">
  <img alt="Tests" src="https://img.shields.io/badge/测试-12%20个测试类-52C41A?style=flat-square">
  <img alt="Dep" src="https://img.shields.io/badge/音频资源-0%20个文件（全合成）-blueviolet?style=flat-square">
</p>

---

## 为什么这个五子棋不太一样

大部分 JavaFX 五子棋到「能下、能判胜负」就结束了。这个项目在三个地方多花了功夫：

| | 常见做法 | 这个项目 |
| --- | --- | --- |
| **音效** | 塞几个 mp3 进 resources | `Dsp` + `SynthWav` 用正弦波、噪声、包络**现场合成**，落子 / 胜利 / 按钮音各不相同，零版权问题、包体极小 |
| **落子** | 直接画一个圆 | `StoneAnimator` 做下落 + 压扁回弹，`ShakeEffect` 在关键手震屏，`Particles` 溅出粒子 |
| **联机** | 局域网 Socket 对下 IP | 三种方式并存，房间码走 Supabase 表轮询，**猜先用加密随机数保证双方都无法作弊** |

---

## 功能

### 对战模式

| 模式 | 说明 |
| --- | --- |
| **人机对战** | 三档 AI：简单 / 中等 / 困难（`EasyAI` / `MediumAI` / `HardAI`，共用 `ScoreEvaluator` 棋型评估） |
| **双人同屏** | 一台电脑轮流落子 |
| **远程联机** | 见下 |

### 三种联机方式

| 方式 | 适用 | 原理 |
| --- | --- | --- |
| **房间码** | 异地首选，零额外软件 | 4 位房号，通过 Supabase 表轮询交换落子（`SupaRestLink` + `RoomCodes`） |
| **局域网直连** | 同一个 WiFi | `LocalIp` 自动探测本机地址，一端监听、一端连接 |
| **手动填 IP** | 跨网段 / 端口转发 | 直接输入对端地址 |

### 其他

- **猜先仪式**：开局随机决定谁执黑，动画真实随机；联机模式下用 `GuessCrypto` 保证双方无法预知或篡改
- **胜利演出**：`VictorySequence` 连成五子后的高亮、粒子、音效序列
- **背景**：`ForestBackground` 程序生成的森林景深，不是贴图
- **设置面板**：音效 / 音乐开关、音量、主题（`SettingsStore` 持久化）
- **打包分发**：`package.ps1` 产出**捆绑 JRE 的免安装 zip**，对方不用装 Java

---

## 架构

```
org.example.gobang
├── Launcher.java / Main.java     入口（Launcher 用于 jpackage 打包）
├── model/                        棋盘与落子（纯数据）
│   ├── Board.java   GameState.java   Move.java   MoveOutcome.java
├── logic/                        规则与 AI（零 JavaFX 依赖）
│   ├── WinChecker.java           五连判定
│   ├── GameSession.java          一局的状态机
│   └── ai/
│       ├── AIStrategy.java       策略接口
│       ├── EasyAI / MediumAI / HardAI
│       ├── ScoreEvaluator.java   棋型打分（活四 / 冲四 / 活三…）
│       ├── AICandidate.java  Point.java
├── net/                          联机
│   ├── Protocol.java             消息协议
│   ├── Link.java / NetLink.java  传输抽象
│   ├── GuessCrypto.java          猜先的加密随机
│   ├── LocalIp.java              本机地址探测
│   └── supa/                     Supabase 房间链路
│       ├── SupaRestLink.java  RoomCodes.java  SupaConfig.java  JsonKit.java
├── fx/                           表现层
│   ├── MenuView / GameView / NetLobbyView / SettingsPanel
│   ├── GuessDialog / RemoteGuessDialog
│   ├── StoneAnimator / Particles / ShakeEffect / VictorySequence
│   ├── ForestBackground   Theme   Ui
└── audio/                        纯代码合成音频
    ├── Dsp.java                  振荡器、包络、噪声、滤波
    ├── SynthWav.java             合成结果写成 WAV PCM
    ├── SoundManager.java         SFX
    ├── MusicManager.java         BGM
    ├── SoundType.java            音效枚举
    └── SettingsStore.java        音量 / 开关持久化
```

**分层原则和 [game-2048](https://github.com/sqyCoder/game-2048) 一致**：`model` / `logic` / `net` 三层完全不依赖 JavaFX，所以 AI、胜负判定、协议、加密都能在无界面环境里跑测试。

---

## 构建与运行

环境：JDK 17+、Maven 3.9+

```bash
git clone https://github.com/sqyCoder/gobang.git
cd gobang

mvn javafx:run     # 运行
mvn test           # 全量测试
```

### 打包给朋友（免安装）

```powershell
powershell -ExecutionPolicy Bypass -File package.ps1            # 含测试
powershell -ExecutionPolicy Bypass -File package.ps1 -SkipTests # 跳过测试
```

产物 `dist/Gobang-win64.zip`：**捆绑 JRE**，对方无需安装 Java 或 IDE。
Windows 首次运行若弹 SmartScreen，点「更多信息 → 仍要运行」即可。

---

## 联机指南

### 房间码（推荐，异地首选）

1. 一方点 **「生成房间码」** → 得到 4 位房号（如 `K7XQ`）
2. 把房号发给朋友
3. 对方点 **「加入房间」** 输入房号 → 双方自动进入猜先

> 房间码走 Supabase，需要配置 `src/main/resources/supabase.properties`（表结构见 `sql/supabase-gobang.sql`）。
> 没配置也能玩——局域网直连和手动 IP 完全不依赖外部服务。

### 局域网直连

一方点「创建房间」，另一方在同一 WiFi 下点「扫描」或直接点选自动探测到的地址。

### 手动填 IP

跨网段或做了端口转发时用，直接输入 `IP:端口`。

---

## 测试

12 个测试类，分四组：

| 组 | 测试类 | 覆盖 |
| --- | --- | --- |
| 规则 | `WinCheckerTest`、`BoardTest`、`SessionTest` | 五连判定（横竖斜、边界、六子算不算）、落子合法性、一局状态机 |
| AI | `AITest` | 三档 AI 的棋型识别与必应手 |
| 网络 | `ProtocolTest`、`GuessCryptoTest`、`RoomCodesTest`、`JsonKitTest`、`LoopbackGameIT`、`SupaRestLinkIT` | 协议编解码、猜先随机不可预测、房间码生成、JSON 容错、**本机回环打完整一局** |
| UI | `LobbySmokeTest`、`UiSmokeTest`、`BoardSnapshotProbeTest` | 界面能起来、棋盘渲染快照 |

```bash
mvn test
```

> `LoopbackGameIT` 是最有意思的一个：在本机起两个端点，让它们自己下一整局，验证协议不会在真实时序下卡死。这类集成测试比单测更容易抓到「本地过、联机挂」的问题。

---

## 已知限制

- **AI 只有三档，没有棋力标定**。困难档能看 2 步内的棋型，打不过认真下的人。
- **房间码依赖 Supabase 免费额度**，且是轮询实现（有几百毫秒延迟），不是长连接。
- **只在 Windows 上验证过**，macOS / Linux 需要自己改 `package.ps1` 的打包部分。
- **字体用的是霞鹜文楷**（OFL 协议，随仓库附带），换字体需要改 `Ui`。
- 没有观战、没有复盘、没有存档续局。

---

## License

个人项目，代码可自由参考。随附的 LXGW WenKai 字体遵循 SIL OFL 1.1（见 `src/main/resources/fonts/OFL-LICENSE.txt`）。
