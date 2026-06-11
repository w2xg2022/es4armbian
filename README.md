# es4armbian

基于 [EmuELEC](https://github.com/EmuELEC/EmuELEC) 的 EmulationStation（`emulationstation2` / es-core + es-app）二次开发版本，目标是让 EmulationStation 前端可以在 **Armbian (Debian 12 Bookworm, aarch64)** 系统上独立运行，而不依赖完整的 EmuELEC 发行版环境。

> 本仓库不再以与 EmuELEC 上游保持兼容/可合并为目标，会根据 Armbian 环境和实际使用需求自由调整 UI、行为和翻译。

## 1. 仓库来源与定位

- 代码来源：EmuELEC 项目中的 `emulationstation` 子模块（`es-core` / `es-app` / `locale` 等），版本基于 EmuELEC 当前主线。
- 修改目的：EmuELEC 原版 EmulationStation 假设运行在完整的 EmuELEC 系统镜像之上（依赖 `/emulationstation`、`/emuelec/configs/emuelec.conf` 等 EmuELEC 专属路径和服务）。本项目将其移植到普通的 **Armbian + Debian 12** 环境中，作为独立的游戏前端 / launcher 使用。
- 设备测试环境：RK3566 (MD1000)，运行 Armbian (Debian 12 Bookworm)，aarch64。

## 2. 主要修改内容

### 2.1 编译适配（GLES2 / Armbian 工具链）

- 必须使用以下 cmake 选项编译，否则会出现头文件 / 渲染相关编译错误：
  - `-DGLES=OFF -DGLES2=ON`：设备只提供 GLES2/3 头文件，若开启 `-DGLES=ON` 会定义 `RENDERER_OPENGLES_10`，导致 `Renderer.cpp` 出现 `#elif` 语法错误及 `glBegin` 等未声明错误。
  - `-DENABLE_EMUELEC=1`：开启 `_ENABLEEMUELEC` 宏，否则 `CloudSaves.cpp`（`loadGridAndCenter`、`EmulatorFeatures::cloudsave` 等）编译失败。
  - `-DCEC=OFF`：设备不需要 CEC 支持。

### 2.2 语言切换修复（中文界面可正常切换并记忆）

EmuELEC 原版在非 EmuELEC 系统镜像下，语言设置无法保存、也不会在启动时被读取，具体原因和修复：

- `Paths.cpp`：原版在 `_ENABLEEMUELEC` 下硬编码 `mSystemConfFilePath = "/emuelec/configs/emuelec.conf"`，该文件在 Armbian 上不存在，导致 `SystemConf::get/set` 形同虚设。
  - 修改为：该文件不存在时 `mSystemConfFilePath = ""`，使 `SystemConf::get/set` 回退到 `Settings`，最终持久化到 `~/.emulationstation/es_settings.cfg` 的 `Language` 字段。
- `main.cpp`（`setLocale()`）：原版从未读取已保存的语言偏好，启动时永远传空字符串给 `EsLocale::init`。
  - 修改为：启动时先读取 `SystemConf::getInstance()->get("system.language")`，再传给 `EsLocale::init(savedLanguage, path)`。

> 注意：`SystemConf::get/getBool` 在 `mSystemConfFile` 为空时会直接调用 `Settings::get/getBool`，**完全忽略 EmuELEC 原版的 `defaults` 默认值表**（例如 `kodi.enabled` 默认应为 `true`，但会被读成 `false`，导致依赖默认值显示的菜单项被隐藏）。如遇到“某选项默认应该显示却被隐藏”的问题，优先检查这个机制。

### 2.3 选单大小写显示规则统一

- EmulationStation 选单文字的大小写由代码渲染时的 `Utils::String::toUpper()` 决定，与 `.po` 翻译文件里 `msgstr` 的原始大小写无关。
- 原版中：
  - `MenuComponent::addRow` / `addWithDescription` / `addSwitch`（如「平台设置」子菜单）会调用 `toUpper`，英文字母统一显示为大写；
  - `MenuComponent::addEntry`（主菜单、退出/QUIT 子菜单等带图标的项目）**没有**调用 `toUpper`，导致「重新启动EmulationStation」「启动RetroArch」等条目大小写与其他菜单不一致。
- 修改：在 `addEntry` 中也调用 `Utils::String::toUpper(name)`，使全部菜单统一为大写英文风格。

### 2.4 中文翻译补全与术语统一

- 补全简体中文 (`zh_CN`) / 繁体中文 (`zh_TW`) 翻译文件 `locale/lang/{zh_CN,zh_TW}/LC_MESSAGES/emulationstation2.po` 中大量缺失的词条（云存档、启动画面 Splash 设置、按键重映射、Wiimote、外部存储挂载、分辨率边框调整等新增功能相关文案）。
- 术语统一：
  - 繁体中文界面「平台設置」统一改为「平台設定」，与其他设置菜单（游戏设定、系统设定等）用词一致。
  - 语言选择菜单中「正體中文」改为更常用的「繁體中文」（影响所有语言界面下的语言选择列表显示）。
- 维护一份完整的英文 / 简体中文 / 繁体中文对照表 `all_translations.csv`，便于后续校对和补译。

### 2.5 其他系统层面注意事项（非代码改动，但部署必需）

- ES 以非 root 的 `game` 账号运行，「重启 / 关机」通过 `systemctl reboot` / `systemctl poweroff`（经 D-Bus 调用 `systemd-logind`）。Debian 12 上 `policykit-1` 已被拆分，需要安装 `polkitd` + `pkexec`（而不是已不存在的 `policykit-1`），否则非 root 用户的重启/关机请求会被 `logind` 拒绝，表现为“只重启了 EmulationStation 但系统没重启”。
- 运行依赖：`libsdl2-mixer-2.0-0` 等（`ldd` 检查缺失库）。

## 3. 编译方法

### 3.1 编译环境

推荐使用与目标设备同架构（aarch64）的 Docker 容器或 chroot 环境（例如基于 Armbian/Debian 12 的容器），避免交叉编译带来的兼容性问题。

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

### 3.3 部署到设备（Armbian, 192.168.8.182 等）

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
