# Android 构建说明

APK 用 python-for-android（p4a）构建。界面是应用内置的 WebView（本地 5000 端口），
不使用 Kivy / SDL2，所以没有原生图形栈。

## 构建

buildozer 只能在 Linux / macOS 上运行；Windows 请用 WSL2 或 GitHub Actions。

### GitHub Actions（最省事）

推一个 tag（`git tag v1.3.9 && git push origin v1.3.9`），或在仓库页面
**Actions → android → Run workflow**。APK 会自动挂到 Release，也可以在 run 页面下载
artifact `jmcomic-apk`。

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

## 配置要点

| 配置 | 值 | 说明 |
| --- | --- | --- |
| `p4a.bootstrap` | `webview` | 用 WebView 承载界面，因此不需要 Kivy |
| `android.api` / `minapi` | 34 / 24 | targetSdk 34，minSdk 24 |
| `android.archs` | `arm64-v8a, armeabi-v7a` | |
| `source.dir` | `webui` | 构建前由 `gui/build_android.py prepare` 暂存引擎 |
| `p4a.local_recipes` | `recipes` | 只含 `recipes/jmcomic/` |
| `p4a.branch` | `develop` | buildozer 跑的是它自己 git clone 的 p4a，升级 pip 里那个没用 |

`requirements` 里有三条不是随便写的：

- **`libwebp`** —— JM 的图片是 `.webp`，而 Pillow 只有在 p4a 构建顺序里含 `libwebp` 时才会
  编译 WebP 编解码器。缺了它，所有图片都会「下载成功但解码失败」。
  `gui/verify_apk.py` 在 CI 上直接读 APK 断言这一点，`tests/test_android_requirements.py`
  也会在它被删掉时报错。
- **`jmcomic` 走 `recipes/jmcomic/`** —— 上游的 PyPI 元数据硬依赖 `curl-cffi`，而 p4a 没有
  它的 recipe。该 recipe 用 `--no-deps` 安装 jmcomic，依赖由我们自己列全。
- **没有 `pyyaml`** —— 它是 C 扩展、没有 android wheel，会让 p4a 的依赖解析直接失败；
  jmcomic 只在惰性路径上用到 YAML，Android 端不会走到。

`curl_cffi` 在 Android 上不存在，`scripts/jmcore.py` 会注册一个桩模块满足
`import jmcomic`，并自动把 HTTP 后端切成 `requests`。

## 下载的文件在哪里

默认保存到应用的外部目录：

```
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

```
[jmcomic] main.py starting
[jmcomic] android=True backend=requests
[jmcomic] Pillow 11.3.0 webp=yes jpg=yes ...
[jmcomic] storage probe: /storage/emulated/0/Android/data/.../files/downloads (via ...) - browsable
```

> 已知的显示问题：APK 文件名和 `versionName` 里的版本号一直是 `1.0.0`（`buildozer.spec`
> 里写的是另一个值）。只影响显示，不影响功能。
