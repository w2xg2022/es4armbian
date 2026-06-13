# es4armbian: EmulationStation for Armbian v1.0

基于 EmuELEC [EmulationStation](https://github.com/EmuELEC/emuelec-emulationstation) 的二次开发版本，目标是让 EmulationStation 前端可以在 **Armbian** 系统上独立运行，而不依赖完整的 EmuELEC 发行版环境。


## 1. 仓库来源与定位

- 代码来源：EmuELEC 项目中的 `emulationstation` 子模块（`es-core` / `es-app` / `locale` 等），版本基于 EmuELEC 当前主线。
- 修改目的：EmuELEC 原版 EmulationStation 假设运行在完整的 EmuELEC 系统镜像之上（依赖 `/emulationstation`、`/emuelec/configs/emuelec.conf` 等 EmuELEC 专属路径和服务）。本项目将其移植到普通的 **Armbian aarch64** 环境中，作为独立的游戏前端 / launcher 使用，并逐步替换为 es4armbian / Armbian 品牌。
- 设备测试环境：RK3566 MD1000 一体机主板，运行 Armbian（**Debian 13 / trixie**），内核 `6.18.33-ophub`（aarch64）。

## 2. 主要修改内容

- **编译适配**：必须使用 `-DGLES=OFF -DGLES2=ON -DENABLE_EMUELEC=1 -DCEC=OFF` 编译，否则因头文件/渲染相关宏不匹配导致编译错误（详见 4.2）。
- **语言切换修复**：`Paths.cpp`/`main.cpp` 改为通过 `Settings`（`~/.emulationstation/es_settings.cfg`）持久化和读取语言设置，使中文等语言可正常切换并在重启后保留。
- **菜单大小写统一**：`MenuComponent::addEntry` 也调用 `toUpper`，使主菜单、退出菜单等图标项与其他菜单的大写风格保持一致。
- **中文翻译补全与术语统一**：补全简体/繁体中文数百条缺失词条，统一同一英文术语（如 `SETTINGS`）在不同菜单中的译法，避免“设定/设置/配置”混用。
- **启动/退出画面（Splash）**：实现独立的启动画面、退出画面开关并支持渲染退出画面；修复两个开关在“平台设置”中无法保存、重启后被重置为默认值的问题（`Settings.cpp` 的 `settings_dont_save` 误排除了相关字段）。
- **品牌重做**：将实际生效的启动画面 `resources/logo.png` 重绘为 es4armbian / ARMBIAN 风格（1920x1080，白到灰渐层背景，灰色 EMULATIONSTATION 字样，沿用原版排版位置）。
- **菜单与功能调整**：移除 SSH 开关、修正 PLATFORM SETTINGS 图标、将网络/蓝牙设定迁移并改用 `batocera-wifi`/`batocera-bluetooth`，支持手动配对蓝牙设备。
- **系统层面适配（部署必需，非代码改动）**：非 root 用户重启/关机依赖 `polkitd` + `pkexec`；需安装 `libsdl2-mixer-2.0-0` 等运行依赖。

## 3. 运行环境（可无需 X11）

- EmulationStation 基于 SDL2 渲染，除了 X11，SDL2 本身也支持 `KMSDRM` / `fbdev` 等视频驱动，理论上可以直接在没有 X 服务器（即 Armbian 的纯 console / `multi-user.target`）的环境下运行，从而减少一层 X11 的资源开销。
- **当前测试环境**仍然通过 `startx` 启动 Xorg 后再运行 `emulationstation`（`DISPLAY=:0`），主要是为了方便用 `xdotool` / `scrot` 等 X11 工具进行调试和截图验证，**并非运行的硬性要求**。
- 若要切换为无 X11 的 `KMSDRM` 模式，需要将 `SDL_VIDEODRIVER` 设为 `kmsdrm`（或留空让 SDL2 自动探测），并直接以具备 `/dev/dri`、`/dev/input` 访问权限的用户启动 `emulationstation`，跳过 `startx`/`xinit`。该模式尚未在本项目中正式验证，作为后续优化方向记录于此。

## 4. 编译方法

### 4.1 编译环境

推荐使用与目标设备同架构（aarch64）的 Docker 容器或 chroot 环境（例如基于 **Debian 13 / trixie**（`arm64v8/debian:trixie`）的容器），避免交叉编译带来的兼容性问题。

### 4.2 获取源码与配置编译

```bash
git clone https://github.com/w2xg2022/es4armbian.git emuelec-es
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

### 4.3 部署到设备

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

## 5. 云端编译（GitHub Actions）

仓库内置 `.github/workflows/build.yml`，推送到 `main` 分支或手动触发（workflow_dispatch）时，会在 `arm64v8/debian:trixie` 容器中以 aarch64 原生方式编译，无需自备 Docker/aarch64 环境：

- 自动安装编译依赖并以 4.2 所述的 `-DGLES=OFF -DGLES2=ON -DCEC=OFF -DENABLE_EMUELEC=1` 参数构建。
- 构建完成后将 `emulationstation`、`resources/`、`locale/` 打包为 `emulationstation-armbian-aarch64.zip`，并发布到 [Releases](../../releases) 的 `latest` 标签，按 4.3 说明覆盖部署即可，无需重新编译。

## 6. 授权 License

本项目沿用上游 EmulationStation / EmuELEC 的 **MIT License**（见 [LICENSE.md](LICENSE.md)，原始版权声明 `Copyright (c) 2014 Alec Lofquist`）。

- 本仓库的所有修改同样以 MIT License 发布，不额外附加限制条款。
- 各第三方库（SDL2、PugiXML、RapidJSON、FreeImage、libcurl、VLC 等）的版权和授权信息见 [CREDITS.md](CREDITS.md)，使用/分发时请一并遵守。
- 原版 EmuELEC EmulationStation 的说明文档保留在 [README_EmuELEC.md](README_EmuELEC.md) 中作为参考。
