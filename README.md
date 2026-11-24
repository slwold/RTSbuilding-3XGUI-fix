# RTSbuilding-3XGUI-fix

[RTS Building: Build From Above](https://github.com/slwold/RTSbuilding-ae2terminal) 的修复版，基于上游 `1.1.8-beta2`（`68b73ef8`），面向 **Minecraft 1.21.1 + NeoForge 21.1.238**。

**版本：`1.1.8-beta2-fix`**（tag 同名）

只修一个 bug：**工作流面板在非 1:1 GUI 缩放下内容显示不完整**。代码改动为 2 个文件、各 1 处调用。

---

## 修复：工作流面板缩放适配

**现象**：工作流面板只有 MC GUI 缩放 = 3 且 RTS UI Scale = 3 时显示完整；其他组合内容被裁剪。RTS 默认 UI Scale = 2、MC 常用 3，所以开箱即坏。把 RTS 调到 3 虽正常，但 UI 过大无法使用。

**根因**：RTS 有固定 UI 缩放系统，`renderScale = RTS 档位 / MC GUI 缩放`。

- 两者相等时 `renderScale = 1`，不进虚拟缩放帧，一切正常；
- 不等时（默认 2/3 ≈ 0.667）进入缩放帧，**所有坐标与裁剪区域都必须乘 `renderScale`**。

项目已有统一入口 `BuilderScreen.enableRtsScissor()` → `RtsGuiScaleCoordinator.enableScissor()`，会乘上 `renderScale`，外层窗口裁剪都走它。但两个工作流面板的**内层列表滚动裁剪**直接调用了原生 `graphics.enableScissor()`，漏了这一步——外层裁剪被放大、内层没放大，列表内容因此被裁掉。

出问题的位置：

| 文件 | 行 |
|---|---|
| `RtsWorkflowPanel.java` | 242（主面板列表） |
| `RtsWorkflowDetailPanel.java` | 158（详情文本） |

**改动**：

```java
// 改前：原生调用，漏乘 renderScale
graphics.enableScissor(x1, y1, x2, y2);
// 改后：走缩放感知入口，自动适配任何缩放档位
this.screen.enableRtsScissor(graphics, x1, y1, x2, y2);
```

**效果**：RTS 保持默认 2 倍、MC 3 倍即可正常显示，无需调到 3 倍。`renderScale = 1` 时行为完全不变，不影响原本正常的场景。

**验证**：`compileJava` / `build` 通过。

> 该问题自上游 `20c67d65`（1.1.8-beta）起一直存在，发布版与当前源码这部分字节相同，只是特定缩放组合才触发。

---

## 相对上游的全部改动

| 文件 | 改动 |
|---|---|
| `RtsWorkflowPanel.java` | 内层裁剪改走 `enableRtsScissor` |
| `RtsWorkflowDetailPanel.java` | 同上 |
| `gradle.properties` | `mod_version` → `1.1.8-beta2-fix` |
| `.gitignore` | 忽略 `/build/`、`/.gradle/` |
| `README.md` | 本文件 |

无新增或删除任何类文件。

---

## 构建

JDK 21，沿用上游 Gradle 配置：

```bash
./gradlew build
```

本机因代理无法访问 `maven.neoforged.net`（TLS 握手失败），改用本地 maven 仓库 + init script 绕过，并跳过依赖 `DevLaunch` 的测试任务（`-x test -x performanceDebt -x uiCoreTest -x uiKitTest`）。这是本机网络环境问题，与代码无关。

---

## 安装

1. 取 `build/libs/rtsbuilding-neoforge-1.21.1-1.1.8-beta2-fix.jar`；
2. 放入实例 `mods/`；
3. 确保只保留一个 RTS jar——旧版改 `.jar.disabled` 或删除，否则重复 modid 会导致 NeoForge 拒绝加载。

---

## License

源码及非媒体文件为 LGPL-3.0-only。原始 RTS Building 贴图与音频依 [LICENSE-ASSETS](LICENSE-ASSETS) 保留所有权利，详见 [ASSET-LICENSES.md](ASSET-LICENSES.md)。Copyright (C) 2026 JerryLunar。
