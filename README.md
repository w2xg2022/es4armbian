# es4armbian

基于 EmuELEC [EmulationStation](https://github.com/EmuELEC/emuelec-emulationstation) 的二次开发版本，目标是让 EmulationStation 前端可以在 **Armbian** 系统上独立运行，而不依赖完整的 EmuELEC 发行版环境。


## 1. 仓库来源与定位

- 代码来源：EmuELEC 项目中的 `emulationstation` 子模块（`es-core` / `es-app` / `locale` 等），版本基于 EmuELEC 当前主线。
- 修改目的：EmuELEC 原版 EmulationStation 假设运行在完整的 EmuELEC 系统镜像之上（依赖 `/emulationstation`、`/emuelec/configs/emuelec.conf` 等 EmuELEC 专属路径和服务）。本项目将其移植到普通的 **Armbian aarch64** 环境中，作为独立的游戏前端 / launcher 使用。
- 设备测试环境：RK3566 MD1000 一体机主板，运行 Armbian（**Debian 13 / trixie**），内核 `6.18.33-ophub`（aarch64）。

## 2. 主要修改内容

### 2.1 编译适配（GLES2 / Armbian 工具链）

- 必须使用以下 cmake 选项编译，否则会出现头文件 / 渲染相关编译错误：
  - `-DGLES=OFF -DGLES2=ON`：设备只提供 GLES2/3 头文件，若开启 `-DGLES=ON` 会定义 `RENDERER_OPENGLES_10`，导致 `Renderer.cpp` 出现 `#elif` 语法错误及 `glBegin` 等未声明错误。
  - `-DENABLE_EMUELEC=1`：开启 `_ENABLEEMUELEC` 宏，否则 `CloudSaves.cpp`（`loadGridAndCenter`、`EmulatorFeatures::cloudsave` 等）编译失败。
  - `-DCEC=OFF`：设备不需要 CEC 支持。

### 2.2 语言切换修复（中文界面可正常切换并保存）

EmuELEC 原版在非 EmuELEC 系统镜像下，语言设置无法保存、也不会在启动时被读取，具体原因和修复：

- `Paths.cpp`：原版在 `_ENABLEEMUELEC` 下硬编码 `mSystemConfFilePath = "/emuelec/configs/emuelec.conf"`，该文件在 Armbian 上不存在，导致 `SystemConf::get/set` 形同虚设。
  - 修改为：该文件不存在时 `mSystemConfFilePath = ""`，使 `SystemConf::get/set` 回退到 `Settings`，最终持久化到 `~/.emulationstation/es_settings.cfg` 的 `Language` 字段。
- `main.cpp`（`setLocale()`）：原版从未读取已保存的语言偏好，启动时永远传空字符串给 `EsLocale::init`。
  - 修改为：启动时先读取 `SystemConf::getInstance()->get("system.language")`，再传给 `EsLocale::init(savedLanguage, path)`。

### 2.3 菜单大小写显示规则统一

- EmulationStation 菜单文字的大小写由代码渲染时的 `Utils::String::toUpper()` 决定，与 `.po` 翻译文件里 `msgstr` 的原始大小写无关。
- 原版中：
  - `MenuComponent::addRow` / `addWithDescription` / `addSwitch`（如「平台设置」子菜单）会调用 `toUpper`，英文字母统一显示为大写；
  - `MenuComponent::addEntry`（主菜单、退出/QUIT 子菜单等带图标的项目）**没有**调用 `toUpper`，导致「重新启动EmulationStation」「启动RetroArch」等条目大小写与其他菜单不一致。
- 修改：在 `addEntry` 中也调用 `Utils::String::toUpper(name)`，使全部菜单统一为大写英文风格。

### 2.4 中文翻译补全与术语统一

- 补全简体中文 (`zh_CN`) / 繁体中文 (`zh_TW`) 翻译文件 `locale/lang/{zh_CN,zh_TW}/LC_MESSAGES/emulationstation2.po` 中大量缺失的词条（涉及云存档、启动画面 Splash 设置、按键重映射、Wiimote、外部存储挂载、分辨率边框调整等较新的功能）。

- **术语统一原则**：同一英文术语在不同菜单中应翻译为同一个中文词，不依据各菜单的上下文各自直译，避免同义词混杂、降低界面一致性。

  - **概念举例**：英文 `SETTINGS` 在游戏设定 (GAMES SETTINGS)、系统设定 (SYSTEM SETTINGS)、声音设定 (SOUND SETTINGS) 等多个菜单标题中出现，应统一对应同一个中文词，而不是有的译成「设定」、有的译成「设置」、「配置」。

  - **简体中文 (zh_CN) 例子**：各菜单标题原本已统一使用「设定」（如「游戏设定」「系统设定」「声音设定」），本次补译延续此用词，未发现需要调整的不一致项。

  - **繁体中文 (zh_TW) 例子**：
    - 「平台設置」→「平台設定」：原本「平台设置」单独使用「設置」，与其余菜单（遊戲設定、使用者介面設定、控制器與藍牙設定、聲音設定、遊戲合輯設定、系統設定）的「設定」不一致，本次统一改为「平台設定」。
    - 语言选择菜单：「正體中文」→「繁體中文」。前者是「Traditional Chinese」的字面直译，后者是繁体中文使用者更常见、更自然的自称用语，此项修改对所有语言界面下的语言选择列表均生效（即英文界面切换语言时，繁体中文选项也会显示为「繁體中文」而非「正體中文」）。

### 2.5 运行环境（无需 X11）

- EmulationStation 基于 SDL2 渲染，除了 X11，SDL2 本身也支持 `KMSDRM` / `fbdev` 等视频驱动，理论上可以直接在没有 X 服务器（即 Armbian 的纯 console / `multi-user.target`）的环境下运行，从而减少一层 X11 的资源开销。
- **当前测试环境**仍然通过 `startx` 启动 Xorg 后再运行 `emulationstation`（`DISPLAY=:0`），主要是为了方便用 `xdotool` / `scrot` 等 X11 工具进行调试和截图验证，**并非运行的硬性要求**。
- 若要切换为无 X11 的 `KMSDRM` 模式，需要将 `SDL_VIDEODRIVER` 设为 `kmsdrm`（或留空让 SDL2 自动探测），并直接以具备 `/dev/dri`、`/dev/input` 访问权限的用户启动 `emulationstation`，跳过 `startx`/`xinit`。该模式尚未在本项目中正式验证，作为后续优化方向记录于此。

### 2.6 其他系统层面注意事项（非代码改动，但部署必需）

- ES 以非 root 的 `game` 账号运行，「重启 / 关机」通过 `systemctl reboot` / `systemctl poweroff`（经 D-Bus 调用 `systemd-logind`）。Debian trixie 上需要安装 `polkitd` + `pkexec`，否则非 root 用户的重启/关机请求会被 `logind` 拒绝，表现为“只重启了 EmulationStation 但系统没重启”。
- 运行依赖：`libsdl2-mixer-2.0-0` 等（`ldd` 检查缺失库）。

## 3. 编译方法

### 3.1 编译环境

推荐使用与目标设备同架构（aarch64）的 Docker 容器或 chroot 环境（例如基于 **Debian 13 / trixie**（`arm64v8/debian:trixie`）的容器），避免交叉编译带来的兼容性问题。

### 3.2 配置与编译

```bash
cd emuelec-es
mkdir -p build && cd build

cmake .. \
  -DGLES=OFF \
  -DGLES2=ON \
  -DCEC=OFF \
  -DENABLE_EMUELEC=1

make -j8
```

编译产物：

- `emulationstation2`（可执行文件，部署时建议命名/链接为 `emulationstation`）
- `locale/lang/*/LC_MESSAGES/emulationstation2.mo`（由 `.po` 文件通过 `make i18n` 相关目标生成，约处理 40 种语言，耗时较长）

> 若只修改了 `.cpp` / `.h` 源码（不涉及翻译文件），可以只重新链接相关目标，无需重新跑 i18n。若修改了 `.po` 文件或涉及 es-core 公共组件（如 `MenuComponent.cpp`），需要完整执行一次 `make -j8`，耗时约 25-30 分钟。

### 3.3 部署到设备

将以下文件复制到设备的 `/opt/emulationstation/` 目录：

```bash
/opt/emulationstation/emulationstation        # 可执行文件
/opt/emulationstation/resources/              # 资源文件
/opt/emulationstation/locale/                 # 翻译文件（容易遗漏）
```

并确保安装运行依赖：

```bash
apt-get install -y libsdl2-mixer-2.0-0 polkitd pkexec
```

赋予可执行权限后，以普通用户（如 `game`）启动 `emulationstation` 即可。

## 4. 授权 License

本项目沿用上游 EmulationStation / EmuELEC 的 **MIT License**（见 [LICENSE.md](LICENSE.md)，原始版权声明 `Copyright (c) 2014 Alec Lofquist`）。

- 本仓库的所有修改同样以 MIT License 发布，不额外附加限制条款。
- 各第三方库（SDL2、PugiXML、RapidJSON、FreeImage、libcurl、VLC 等）的版权和授权信息见 [CREDITS.md](CREDITS.md)，使用/分发时请一并遵守。
- 原版 EmuELEC EmulationStation 的说明文档保留在 [README_EmuELEC.md](README_EmuELEC.md) 中作为参考。
