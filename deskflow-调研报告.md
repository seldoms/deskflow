# Deskflow 调研报告

> 仓库：https://github.com/seldoms/deskflow
> 本地克隆：`/Users/seldoms/WorkBuddy/deskflow`（master，HEAD = `a7a51e00d`）
> 版本：`DESKFLOW_VERSION 1.27.0`

---

## 1. 这是什么

**Deskflow 是一套开源软件 KVM（键鼠共享）工具**：用一台电脑的键盘/鼠标，通过网络去操作旁边另一台电脑，鼠标可以「穿过」屏幕边缘跳到另一台机器上。

- 技术栈：C++20 + Qt 6.7+，CMake 构建，OpenSSL 3.0+（TLS）
- 支持平台：**macOS / Windows / Linux / BSD**（`src/lib/platform/` 下有 `OSXComputer.mm`、`MSWindowsComputer.cpp`、`XWindowsComputer.cpp`、`EiComputer.cpp`）
- 血缘：代码头部的版权声明还是 `Synergy App Ltd` / `Chris Schoeneman`，即 **Synergy / Barrier 这一脉的开源延续**
- 组成：`deskflow-gui`（图形界面）+ `deskflow-core`（后台核心，server/client 两种模式）+ `deskflow-daemon`（Windows 提权守护）
- 你的这个 clone 与上游 `deskflow/deskflow` 的 master 同步，最新提交只是注释拼写修正

> 顺带一提：仓库里 `docs/Readme.md` 的项目官网链接写成了 `github.com/deskflow/deskflo`（少了 w，上游的笔误）。

**结论：你要的功能就是它的主线功能，不用二次开发。**

---

## 2. 你的场景：Mac + Win，各接一台显示器，共用一套键鼠

### 2.1 能不能共用？——能

把接了物理键鼠的那台设为 **server**（服务端），另一台设为 **client**（客户端）。

在你的代码里可以对应到：

- `src/lib/server/Server.cpp` —— 服务端逻辑，负责抓取本机键鼠事件、判断光标是否越界、转发给客户端
- `src/lib/client/Client.cpp` + `ServerProxy.cpp` —— 客户端逻辑，接收事件并在本机「模拟」出按键和鼠标移动
- 默认端口 **24800**，**TLS 默认开启**并有指纹校验（首次连接需要人工确认）

### 2.2 每台电脑自己的显示器怎么算

Deskflow 把**一台电脑的整个桌面（它自己挂的几块屏取并集）当作一个「屏幕节点」**。代码证据：

- macOS：`OSXComputer.mm` 里遍历 `CGGetActiveDisplayList`，对每块屏取 `CGDisplayBounds` 求并集作为本机的 shape
- Windows：`MSWindowsComputer.cpp::updateComputerShape()` 用 `GetSystemMetrics(SM_XVIRTUALSCREEN / SM_CXVIRTUALSCREEN / ...)`

所以你「一台 Mac 一块屏、一台 Win 一块屏」是很简单的两节点布局；以后某台再接一块副屏，也只是那个节点的 shape 变大，不用改拓扑。

---

## 3. 分辨率变化兼容性 —— 兼容，而且是「运行期动态 + 按比例映射」两层保障

### 3.1 运行期能感知分辨率变化

- macOS 注册了 `CGDisplayRegisterReconfigurationCallback`；Windows 走虚拟屏幕 metrics 变化
- 变化后发出 `ComputerShapeChanged` 事件
- 客户端侧处理函数里直接写着日志：`Client::handleShapeChanged()` → `LOG_DEBUG("resolution changed")`，然后重新向服务端上报本机的 `x, y, w, h`
- 服务端侧 `Server::handleShapeChanged()` 会更新跳转坐标，如果改的正好是当前激活的机器，会立刻按新尺寸重算光标位置

也就是说：**中途改分辨率、拔插显示器、投屏切换，都不需要重启 Deskflow**。

### 3.2 跨屏坐标是「按比例」映射，不是按像素

这是最关键的一点，看 `Server.cpp`：

```cpp
float Server::mapToFraction(...) {      // 源屏幕边缘位置 → 0.0 ~ 1.0 的比例
    return (y - sy + 0.5f) / static_cast<float>(sh);
}
void Server::mapToPixel(...) {          // 比例 → 目标屏幕自己的像素坐标
    y = static_cast<int32_t>(f * sh) + sy;
}
```

含义：光标从源屏幕边缘的**百分比位置**出去，就在目标屏幕边缘的**同一百分比位置**进来。

**举例**：Mac 2560×1440 在左，Windows 1920×1080 在右。光标从 Mac 右边缘、高度 70% 处划出去 → 会落在 Windows 左边缘、高度 70% 处。分辨率不同**不会错位**。

### 3.3 唯一要注意的坑：高 DPI / Retina 与「物理高度不一致」

坐标换算用的是**各系统上报的桌面尺寸**，而不是物理屏幕的厘米数：

- macOS 用 `CGDisplayBounds`（Retina 内置屏上报的是**逻辑分辨率**，比如面板 2880×1800 而上报 1440×900）
- Windows 用 `SM_CXVIRTUALSCREEN`（DPI 感知模式下是物理像素）

因此当两侧「像素高度」的比值与「实际屏幕物理高度」的比值对不上时（典型场景：一台 16:10 的 Mac 笔记本屏 + 一台 16:9 的 1080p 显示器并排），光标纵向穿屏时会有轻微的「跳一下」的手感差异。

**解决办法**（见第 4 节）：在 server 配置文件里用 `links` 的**百分比区间**只对齐某一段边缘，而不是整条边。

> 实测补充：在本机（单台 2560×1440 非 Retina 显示器）跑 CoreGraphics 查询，`CGDisplayBounds` 与 `CGDisplayPixelsWide` 完全一致，说明普通显示器上不存在缩放偏差。

---

## 4. 两台电脑的相对位置能不能调？——能，两种方式

### 方式 A：图形界面拖拽（推荐，日常用这个）

服务端配置页里直接**拖动电脑图标**到网格上任意格子。相关代码：

- `src/lib/gui/dialogs/ServerConfigDialog.ui` 的提示原文：*"Configure the layout of your computer displays by dragging to where you want."*
- `src/lib/gui/widgets/ComputerSetupView.cpp` —— 实现了拖放（`setDragDropMode(DragDrop)`）
- 网格默认 **5 列 × 3 行**（`gridWidth=5`、`gridHeight=3`），服务端默认落在中心格（`serverDefaultIndex = 7`）
- `ServerConfig.cpp::operator<<` 会根据网格相邻关系自动生成 `section: links`（只会生成整边对齐，方向为 left/right/up/down）

### 方式 B：直接改服务端配置文件（能做精细对齐）

生成的配置长这样：

```
section: links
    macmini:
        right = winbox
    winbox:
        left  = macmini
end
```

配置语法文件：`docs/Configuration.md`。相比 GUI，手写配置多了 **边区间（range）** 能力：

```
section: links
    mac:
        right          = win            # 整条右边 → 整个左边
    win:
        left(50,100)   = mac(0,50)      # 只让 win 的左下半 对应 mac 的左上半
end
```

区间是 `(start, end)` 百分比，0 是边的左/上，100 是右/下。**这正好可以用来修掉 3.3 里说的分辨率/物理高度对不齐问题**——把两条边只对齐其中一段。

### 配套的调节选项（`docs/Configuration.md`）

| 选项 | 作用 |
|:---|:---|
| `switchDelay`（默认 250ms） | 光标要贴边停留多久才切屏，防误触 |
| `switchDoubleTap` | 改成「快速双击边缘」才切 |
| `switchCorner*` / `switchCornerSize` | 四角不触发切屏，防误触 |
| `relativeMouseMoves` | 锁定光标后用相对移动（对游戏友好），默认关闭 |
| `enableSwitchDelay` | 开关切屏延时 |

UI 上还有热键：`switchToScreen(名字)`、`switchInDirection(方向)`，以及 Scroll Lock 锁定光标到当前机器。

---

## 5. 蓝牙键鼠能用吗？——完全没问题

Deskflow 工作在**操作系统输入事件层**，与键鼠怎么连的（蓝牙 / USB / 2.4G 接收器）**无关**：

- macOS 侧用 CGEventTap / Accessibility 权限抓取和注入事件（`OSXComputer.mm`、`OSXKeyState.cpp`）
- Windows 侧用低级钩子（`MSWindowsHook.cpp`、`MSWindowsDesks.cpp`）

对蓝牙的实际影响：

1. **更省事**：用了 Deskflow，键鼠只需配对到 server 那一台，另一台靠网络收模拟事件——不用再按罗技 Easy-Switch 的 1/2/3 切换了。
2. **延迟不变**：蓝牙本身的延迟只体现在连接到 server 那台；跨到另一台是局域网跳转，通常几毫秒。
3. **唤醒限制**：蓝牙设备唤醒不了 client 那台电脑（因为蓝牙只连着 server）。睡眠唤醒要靠 client 本机键盘/电源键，或开 `preventSleep=true`。
4. **掉线即中断**：蓝牙键鼠在 Mac 上断连/休眠时，Windows 那台也收不到输入——因为事件源头断了。

---

## 6. 部署要点与坑

| 项 | 说明 |
|:---|:---|
| macOS 权限 | **必须给「辅助功能（Accessibility）」权限**。代码里是硬校验：`AXIsProcessTrusted()` 不过就直接抛错退出并提示去系统设置里授权。另外需要放行**本地网络**权限（`NSLocalNetworkUsageDescription`） |
| Windows | 想操作**以管理员权限运行**的窗口，Deskflow 也需要以管理员/服务方式运行（`deskflow-daemon`）；服务模式要求配置不放在安装目录 |
| 网络 | 两台在同一局域网，默认端口 24800；TLS 默认开启，首次连接要确认对端指纹 |
| 装机方式 | **建议直接用 deskflow.org 的官方安装包**，不必自己编译（编译需要 Qt 6.7+ / CMake 3.24+ / OpenSSL 3.0+，首次配置还要拉依赖，比较重） |
| 排查 | 日志等级调到 debug；`ComputerShapeChanged` / `resolution changed` 关键字可以直接确认分辨率识别情况 |

---

## 6.5 中文支持情况

分四个层面看，**都支持**。界面翻译已在本项目中补全至 100%。

### 6.5.1 界面中文 —— 已汉化完成（323/323）

- 翻译文件：`translations/deskflow_zh_CN.ts`，`LocalizedName` = **简体中文**
- 已进构建：`translations/CMakeLists.txt` 的 TRS 列表里包含 `deskflow_zh_CN.ts`
- 语言选择界面：设置对话框的 `comboLanguage` 由 `I18N::detectedLanguages()` 动态填充；`I18N::detectLanguages()` 里对 `zh` 做了特殊处理（**保留完整语言码 `zh_CN`**，不像其他语言那样截成两位），说明中文是被专门照顾的

**关于原始完成度的更正**：上游仓库里 `deskflow_zh_CN.ts` 有 119 条标记为 `type="unfinished"`，但其中 **113 条其实已经有中文译文**——只是没清除 `unfinished` 标记。真正没有译文的只有 **6 条**（集中在 MainWindow 与 QObject）。另外有一批机器翻译质量堪忧（如 `Exit` → 出口、`&Window` → 窗户、`Scale` → 规模）。

本项目已处理完毕：

| 项 | 数值 |
|:---|:---|
| 总字符串数 | **323** |
| 已翻译 | **323（100%）** |
| `unfinished` / 空译文 | **0** |
| 新补翻译 | 6 条 |
| 修正机器翻译 | 22 条 |
| 清除 `unfinished` 标记 | 111 条 |

`lrelease` 校验输出：`Generated 323 translation(s) (323 finished and 0 unfinished)`

产物与使用方式见 `zh-CN/安装说明.md`，语言包为 `zh-CN/deskflow_zh_CN.qm`。

> 注意：对话框上的「确定 / 取消」等标准按钮由 **Qt 自己的语言包**翻译，Deskflow 会去找 `qt_zh_CN.qm`（中文用完整语言码而不是 `zh`）。缺了它不影响界面主体，只是标准按钮保持英文。

### 6.5.2 中文输入法 —— 支持，代码里有专门处理

Deskflow 转发的是**按键事件**（key id + 修饰键掩码），不是文本，所以输入法在「光标当前所在的那台机器」上本地运行：

- 在 Windows 端打拼音 → Windows 自己的输入法处理，候选框出现在 Windows 上
- macOS 端在 `OSXKeyState.cpp:185-210` 有专门逻辑：如果当前输入源是**输入法（没有 key layout data，比如简体拼音）且不可选**，就回退到 `TISCopyCurrentASCIICapableKeyboardLayoutInputSource()` 用 ASCII 布局做键位翻译——**这正是不让输入法干扰英文按键的处理**
- Windows 端用 `SendInput` 注入事件（`MSWindowsDesks.cpp:84-107`），对输入法友好

### 6.5.3 剪贴板中文 —— 支持

文本在网络上走 **UTF-8**。macOS 侧 `OSXClipboardTextConverter` 在 UTF-8 与系统编码之间转换，Windows 侧有 UTF-16 / HTML / 位图等转换器。中文复制粘贴正常。

### 6.5.4 键盘布局同步的小坑

客户端选项 `languageSync`（界面文案 *"Use server's keyboard language on this computer"*，**默认开启**），走协议 1.8 的 `kMsgDLanguageSynchronisation`（`LSYN`），会把 server 的**键盘布局列表**同步给 client。若 Mac 上频繁切换输入法/布局导致 Windows 端布局乱跳，可以在客户端配置里把它关掉。

> 另外：计算机名建议用 ASCII。文档里 `aliases` 明确要求「valid hostnames」。

---

## 7. 一句话结论

| 你的问题 | 结论 |
|:---|:---|
| Mac + Win 共用一套键鼠 | **可以**，这就是 Deskflow 的主功能；Mac 做 server（接键鼠），Win 做 client |
| 两块屏各接一台电脑 | **可以**，每台电脑的桌面被当成一个节点，按左右/上下连接 |
| 蓝牙键鼠 | **可以**，且只需要配对到 Mac 那台，省掉多设备切换 |
| 分辨率变化兼容 | **兼容**。运行期动态感知（`resolution changed`），跨屏按百分比映射；高 DPI/物理高度不一致时可用 `links` 的百分比区间做精细对齐 |
| 相对位置可调 | **可以**。GUI 拖拽（默认 5×3 网格）即可；要精细对齐就手写 `section: links` 带百分比区间 |
| 中文支持 | **已完整汉化（323/323）**。`deskflow_zh_CN.ts` 已补齐并重新编译为 `zh-CN/deskflow_zh_CN.qm`；输入法通过按键事件转发天然支持，macOS 侧有 IME 回退逻辑；剪贴板文本走 UTF-8 |

---

## 8. 本次汉化产物

| 路径 | 说明 |
|:---|:---|
| `translations/deskflow_zh_CN.ts` | 已就地补全的翻译源文件（323/323，0 unfinished），从源码编译即得中文界面 |
| `zh-CN/deskflow_zh_CN.qm` | 编译好的语言包，可直接丢进官方安装包使用 |
| `zh-CN/deskflow_zh_CN.ts` | 翻译源文件副本，供后续与上游合并 |
| `zh-CN/安装说明.md` | 各平台安装路径、Qt 语言包搭配、校对记录 |

**未汉化（有意保留）**：核心进程里数百条 `LOG_*` 日志字符串（C++ 硬编码，不属 Qt 翻译体系，保留英文便于提 issue 时被检索）、帮助文档正文 `docs/HelpMain.md`（文件名在代码里写死，改成中文会让英文用户失去英文帮助）、配置文件的选项名（属配置语法而非界面文案）。
