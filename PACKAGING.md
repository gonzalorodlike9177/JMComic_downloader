# 桌面打包说明

四个平台共用同一套代码：`scripts/jmcore.py`（引擎）+ `webui/`（界面，浏览器里的本地页面）。
界面不依赖任何 GUI 工具包，所以打包只是 Python + jmcomic。

## 通用

```bash
python -m pip install jmcomic pyinstaller

python gui/build.py            # 本平台，默认单文件
python gui/build.py --onedir   # 输出文件夹：启动更快，推荐
python gui/build.py --console  # 保留控制台，能直接看到界面地址
python gui/build.py --verify   # 打包后自动跑一次真实下载自检，推荐
python gui/build.py --icon app.ico
python gui/build.py --android  # 只打印 Android 构建说明
```

| 平台 | 产物 |
| --- | --- |
| Windows | `dist/jmcomic-downloader/jmcomic-downloader.exe`（`--onedir`） |
| Linux | `dist/jmcomic-downloader/jmcomic-downloader` |
| macOS | `dist/jmcomic-downloader.app` |

macOS 的包只能在 macOS 上构建：PyInstaller 不能交叉编译。

`--verify` 调的是产物里的 `--selftest`：真实下载一章并把结果写进
`%TEMP%/jmcomic-downloader-selftest.log`。打包版是 `--windowed`、没有 stdout，所以自检
结果一定落文件，`--verify` 就是读这个文件判成败。

> 打包前请先退出正在运行的旧程序。Windows 会锁住 `dist` 里的文件，PyInstaller 清理失败会
> 留下半截的二进制，运行时报 `Failed to execute script ... unhandled exception`。

## Windows

```powershell
python gui/build.py --onedir
```

未签名，首次运行会有 SmartScreen 提示（更多信息 → 仍要运行）。双击会自动打开浏览器；
没有就用 `--url-file` 拿地址：

```powershell
.\dist\jmcomic-downloader\jmcomic-downloader.exe --url-file "$env:TEMP\jm-url.txt"
Get-Content "$env:TEMP\jm-url.txt"
```

## Linux

```bash
python gui/build.py --onedir
```

单文件模式启动时要解压到 `/tmp`，若 `/tmp` 挂载为 `noexec` 会直接失败，因此建议 `--onedir`。
自动打开浏览器需要 `xdg-open`（`sudo apt install xdg-utils`）；缺了也能用 `--url-file`
或 `--no-browser`。

### AppImage

```bash
mkdir -p AppDir/usr/bin
cp -r dist/jmcomic-downloader/* AppDir/usr/bin/

cat > AppDir/jmcomic-downloader.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=JMComic 下载器
Exec=jmcomic-downloader
Categories=Network;FileTransfer;
Terminal=true
EOF

wget https://github.com/AppImage/appimagetool/releases/download/continuous/appimagetool-x86_64.AppImage
chmod +x appimagetool-x86_64.AppImage
./appimagetool-x86_64.AppImage AppDir jmcomic-downloader-x86_64.AppImage
```

### .deb

```bash
sudo apt install -y ruby-dev build-essential
sudo gem install fpm
python gui/build.py --onedir
fpm -s dir -t deb -n jmcomic-downloader -v 1.0.0 --depends xdg-utils \
    dist/jmcomic-downloader/=/opt/jmcomic-downloader/
```

## macOS

脚本在 macOS 上自动使用 `--onedir`（`.app` 本身就是目录结构）。

未签名的 `.app` 在别人电脑上会被 Gatekeeper 拒绝，用户需要执行一次：

```bash
xattr -dr com.apple.quarantine /Applications/jmcomic-downloader.app
```

或右键 → 打开 → 再点打开。

```bash
# dmg
hdiutil create -volname "JMComic" -srcfolder dist/jmcomic-downloader.app \
    -ov -format UDZO dist/jmcomic-downloader.dmg

# 签名 + 公证（需要 Apple 开发者账号，$99/年）
codesign --deep --force --options runtime \
    --sign "Developer ID Application: Your Name (TEAMID)" dist/jmcomic-downloader.app
xcrun notarytool submit dist/jmcomic-downloader.dmg \
    --apple-id "you@example.com" --team-id TEAMID --password "app-specific-pw" --wait
xcrun stapler staple dist/jmcomic-downloader.dmg
```

架构：在 Intel Mac 上构建的只能跑 Intel，Apple Silicon 上构建的只能跑 ARM；通吃需要在
`build.py` 里加 `--target-arch universal2`（体积翻倍）。

## CI

`.github/workflows/desktop.yml` 有四个 job：

| Job | 内容 |
| --- | --- |
| `smoke` | 字节编译、引擎不依赖 GUI 工具包的断言、网页资源与 CLI JSON 契约检查 |
| `linux` / `windows` / `macos` | 各自构建并跑**冻结产物**的端到端测试 |

打 tag 时三个平台的产物会自动挂到 Release（Android 由 `android.yml` 负责）。
