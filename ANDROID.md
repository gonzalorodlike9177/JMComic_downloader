# Android 构建说明

APK 用 python-for-android（p4a）构建。界面是应用内置的 WebView（本地 5000 端口），
不使用 Kivy / SDL2，所以没有原生图形栈。

## 构建

buildozer 只能在 Linux / macOS 上运行；Windows 请用 WSL2 或 GitHub Actions。

### GitHub Actions（最省事）

在仓库页面 **Actions → android → Run workflow**。APK 会自动挂到 Release，也可以在 run 页面下载 artifact `jmcomic-apk`。

哦对了你直接去Release下载也可以的

CI 缓存了 `~/.buildozer` 和 `.buildozer`，第二次起只要几分钟；改动 `buildozer.spec`
或 `recipes/` 会让项目缓存失效，那一轮是冷构建（约 20 分钟）。

### WSL2 / Linux / macOS

```bash
sudo apt install -y git zip unzip openjdk-17-jdk autoconf automake libtool pkg-config \
  zlib1g-dev libncurses-dev libtinfo6 cmake libffi-dev libssl-dev build-essential ccache
pip install buildozer "cython<3.0"

python gui/build_android.py debug     # 构建 -> bin/*.apk
python gui/build_android.py deploy    # 构建并安装到已连接的设备
python gui/build_android.py logcat    # adb logcat -s python:D
python gui/build_android.py clean
```

`prepare` / `unprepare` 只做源码暂存（把 `scripts/jmcore.py` 复制进 `webui/`，构建后自动删除）；
`--keep` 可以保留暂存结果。

APK 是 debug 签名，可直接 `adb install -r bin/*.apk`。若报
`INSTALL_FAILED_UPDATE_INCOMPATIBLE`（签名变了），先
`adb uninstall io.github.nannank0.jmcomicdownloader`。

> 构建时报 `build-tools folder not found` / `Aidl not found`：buildozer 自带的旧版
> cmdline-tools 在 JDK 17 下无法工作。把新版 cmdline-tools 放到
> `~/.buildozer/android/platform/android-sdk/cmdline-tools/latest`，
> 在 `tools/bin/sdkmanager` 建软链，然后删掉 `.buildozer/state.db` 重跑
> （CI 里就是这么做的）。需要 **JDK 17**。

## 下载的文件在哪里

默认保存到应用的外部目录：

```txt
手机存储/Android/data/io.github.nannank0.jmcomicdownloader/files/downloads/
```

文件管理器和 USB 都能看到（`内部存储/Android/data/...`）。构建时优先选它，并逐个候选目录
做真实写入测试；启动时还会把旧版本写在私有目录里的文件搬过来。
私有目录 `/data/user/0/<包名>/files/downloads` 只有 root 或 `adb run-as` 能读，不作为默认值。

## 排错

| 现象 | 处理 |
| --- | --- |
| 构建报 `curl_cffi` 相关错误 | 确认 `recipes/jmcomic/__init__.py` 存在，且 `p4a.local_recipes = recipes` |
| `No module named 'jmcore'` | 先跑 `python gui/build_android.py prepare` |
| App 启动即闪退 | `adb logcat -s python:D` 看 Python 报错 |
| 界面停在加载页 | 本地服务没起来（WebView 会一直重试 `localhost:5000`）；logcat 里应有 `[jmcomic] main.py starting` |
| 提示下载完成但找不到文件 | 看 logcat 的 `[jmcomic] storage probe:`，它写明实际目录及是否可浏览 |
| 图片**全部**下载失败、异常 `cannot identify image file` | 构建缺 WebP 支持：检查 `requirements` 里的 `libwebp`，以及 `[jmcomic] Pillow ... webp=` 是否为 `yes` |

启动时 logcat 会打印这些行，贴出来基本就能定位问题：

```txt
[jmcomic] main.py starting
[jmcomic] android=True backend=requests
[jmcomic] Pillow 11.3.0 webp=yes jpg=yes ...
[jmcomic] storage probe: /storage/emulated/0/Android/data/.../files/downloads (via ...) - browsable
```

> 版本号：APK 文件名和 `versionName` 都取自 `buildozer.spec` 里的 `version`，发布时把它改成和
> git tag 一致的值（v1.4.0 实测：文件名是 `jmcomicdownloader-1.4.0-...apk`，manifest 里
> `versionName` 也是 `1.4.0`）。注意改这个文件会让 CI 的项目缓存失效，那一轮是冷构建（约 20 分钟）。
