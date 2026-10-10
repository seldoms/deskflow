# Windows 端部署说明

配套 Mac 端自编译版本（commit `fb5d1dcea`）使用。Mac 端已装好 `~/Applications/Deskflow.app`。

## 方式一：官方安装包 + 中文语言包（推荐，10 分钟搞定）

1. 到 https://github.com/deskflow/deskflow/releases 下载最新的 Windows 安装包（`.msi` 或 `.exe`）并安装。
   > 建议与 Mac 端版本保持一致（1.27.x），跨版本协议兼容性没保证。

2. 把本目录的 `deskflow_zh_CN.qm` 复制到安装目录的 `translations\` 子目录（默认
   `C:\Program Files\Deskflow\translations\`）。需要管理员权限。

3. 再从 Qt 官方语言包里补一份 `qt_zh_CN.qm` 放进同一目录——没有它界面主体是中文，
   但「确定/取消」这类标准按钮仍是英文。最快的获取方式：
   ```powershell
   # 用 aqtinstall 直接抓 Qt 的中文语言包（无需装 Qt）
   pip install aqtinstall
   aqt install-qt windows desktop 6.8.3 win64_msvc2022_64 -m qttranslations -O C:\Temp\qt
   # 然后把 C:\Temp\qt\6.8.3\msvc2022_64\translations\qt_zh_CN.qm 拷到
   # C:\Program Files\Deskflow\translations\
   ```

4. 首次启动后如语言没变，在 设置 → Language 里选 简体中文 并重启程序。

## 方式二：源码编译（想连 Windows 端也用你的仓库）

前置：Visual Studio 2022（含 C++ 桌面开发）、CMake 3.24+、Qt 6.7+（在线安装器装到 `C:\Qt`）。

```powershell
git clone https://github.com/seldoms/deskflow.git
cd deskflow
# 把 Qt 的 cmake 与 bin 目录加入 PATH（按实际版本调整）
$env:Path = "C:\Qt\6.8.3\msvc2022_64\bin;C:\Qt\6.8.3\msvc2022_64\lib\cmake;$env:Path"
cmake -S . -B build
cmake --build build --config Release
```

其余依赖（OpenSSL 等）vcpkg 会自动拉取。编译产物在 `build\bin\`。

## 两端组网步骤

1. **Mac 端（接键鼠的这台）当 server**：打开 Deskflow → 选「服务器」→ 在布局网格里把
   新电脑拖到 Mac 旁边（对应两台显示器的实际相对位置，随时可拖动调整）。
2. **Windows 端当 client**：选「客户端」，填 Mac 的 IP（服务端界面会显示）。
3. Mac 端弹出新客户端提示时点接受（或在服务器设置里手动添加）。
4. Windows 防火墙首次会弹窗，**两个网络（专用+公用）都放行**；端口 24800，TLS 默认开启。
5. Mac 端首次运行必须在 系统设置 → 隐私与安全性 → **辅助功能** 里勾选 Deskflow，
   否则核心进程直接退出（代码硬校验）。

## 已知注意点

- Windows 端若要以管理员权限操作某些窗口，Deskflow 本身也得以管理员运行。
- 若两端键盘布局来回跳，关掉客户端的 languageSync 选项。
- 蓝牙键鼠只配对 Mac 即可；Windows 那台无法用这套键鼠唤醒（蓝牙没连它）。
