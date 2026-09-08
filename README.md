# Fedora KDE OBS 插件云编译 Monorepo

这是 `noame19/fedora-kde-obs-plugin` —— 一个把多个 OBS 插件集中管理 + 在 GitHub Actions 上自动编译为 Fedora 44 匹配版本的单一仓库（monorepo）。

## 这个仓库是什么

本机的 OBS Studio 跑在 Fedora 44 KDE 上，系统自带的 obs-studio / ffmpeg / onnxruntime 都是当前发行版版本。OBS 插件的源码不能直接在本机编译（要装一堆 -devel 包，会污染环境），所以我们用 GitHub Actions：

1. **触发**：你在浏览器上点 `Run workflow`
2. **跑在云端**：GitHub 的 Ubuntu runner + `fedora:44` 容器，编译产物 ABI 100% 匹配你本机
3. **下载**：编译完产出 `obs-<name>-fedora44-x86_64.tar.gz`，你下载下来
4. **安装**：解压到 `~/.config/obs-studio/plugins/<name>/`，重启 OBS 就生效

## 目录结构

```
fedora-kde-obs-plugin/
├── .github/workflows/
│   ├── build-obs-ffmpeg-afilter.yaml     ← 编译 obs-ffmpeg-afilter 的 workflow
│   └── build-obs-backgroundremoval.yaml  ← 编译 obs-backgroundremoval 的 workflow
└── plugins/
    ├── obs-ffmpeg-afilter/               ← 插件 1 的源码（fork 自 sorayuki）
    │   ├── CMakeLists.txt
    │   ├── src/obs-ffmpeg-afilter.cpp
    │   ├── data/locale/                  ← i18n 文件（zh-CN.ini 等）
    │   └── ...
    └── obs-backgroundremoval/            ← 插件 2 的源码（fork 自 royshil）
        ├── CMakeLists.txt
        ├── src/
        ├── data/locale/                  ← i18n 文件
        └── ...
```

每个插件独立成目录，独立编译，不互相依赖。添加新插件 = 在 `plugins/` 下新增一个子目录 + 在 `.github/workflows/` 下新增一个 workflow。

## 已收录的插件

| 插件名 | 仓库子目录 | 用途 | 触发 workflow |
|---|---|---|---|
| **obs-ffmpeg-afilter** | `plugins/obs-ffmpeg-afilter` | 把 FFmpeg 的全部音频滤镜（80+ 个）暴露给 OBS 使用 | Actions → Build obs-ffmpeg-afilter for Fedora Linux 44 → Run workflow |
| **obs-backgroundremoval** | `plugins/obs-backgroundremoval` | 人像背景移除 / 替换（基于 ONNX 模型） | Actions → Build obs-backgroundremoval for Fedora Linux 44 → Run workflow |

## 怎么用

### 编译一个插件

1. 打开 https://github.com/noame19/fedora-kde-obs-plugin
2. 点顶栏 **Actions**
3. 左侧选对应插件的 workflow（如 `Build obs-ffmpeg-afilter for Fedora Linux 44`）
4. 右侧 **Run workflow** → **Run workflow**
5. 等 5-8 分钟编译完，点进这次 run，往下翻到 **Artifacts** 区
6. 下载 `obs-<name>-fedora44-x86_64.tar.gz`

### 安装到本机

下载下来的 tar.gz 是 `obs-<name>/{bin/64bit/<name>.so, data/locale/, data/...}` 布局，直接对应 OBS 的用户级插件目录约定：

```bash
# 解压到一个临时目录看一眼内容
mkdir -p /tmp/obs-plugin-install
tar -xzf obs-ffmpeg-afilter-fedora44-x86_64.tar.gz -C /tmp/obs-plugin-install

# 确认里面是 obs-ffmpeg-afilter/{bin,data}/
ls /tmp/obs-plugin-install/obs-ffmpeg-afilter/

# 拷贝到 OBS 用户级插件目录
mkdir -p ~/.config/obs-studio/plugins
cp -r /tmp/obs-plugin-install/obs-ffmpeg-afilter ~/.config/obs-studio/plugins/

# 重启 OBS
```

obs-backgroundremoval 还需要装 ML 依赖才能加载（ONNX Runtime）：

```bash
sudo dnf install onnxruntime-devel
```

## 添加新插件

1. 把上游源码（直接快照，不用 .git 历史）放到 `plugins/<new-plugin>/`
2. 参考 `.github/workflows/build-obs-backgroundremoval.yaml`，写一个 `build-<new-plugin>.yaml`：
   - 触发方式：`workflow_dispatch`
   - 容器：`fedora:44`
   - 装对应的 `-devel` 依赖
   - `cd plugins/<new-plugin> && cmake -S . -B build -G Ninja ... && cmake --build build ... && cmake --install build --component UserPlugin --prefix /tmp/stage/<new-plugin>`
   - 打包 `tar -C /tmp/stage -czf /tmp/<new-plugin>-fedora44-x86_64.tar.gz <new-plugin>`
   - 上传 artifact
3. 把这个文件 `git commit` 并 `git push`

## 关键设计要点

- **不用 Ubuntu 容器 + obsproject PPA**：那条路会让编译产物的 FFmpeg soname（6.x）和本机（8.x）对不上。**fedora:44 容器**直接用发行版的 ffmpeg 8.x，ABI 天然一致。
- **`cmake --install ... --component UserPlugin`** 是上游 OBS 插件的标准安装方式，会自动把 `data/locale/` 装到正确位置；obs-backgroundremoval 用这条线。obs-ffmpeg-afilter 上游没有这个 component 定义，所以走手工 `cp -r` —— 但要小心定位到 `*/obs/obs-plugins/<name>/data` 子目录再拷贝，避免出现 `<plugin>/data/data/` 这种错位。
- **ABI 验证用 `ldd`** 而不是 `ldconfig -p | grep`：容器里 ld.so.cache 可能没刷新，`ldd` 是动态链接器直接跑，更接近 OBS 实际加载插件时的行为。
