# Supramark React Native Windows 兼容性分析

> 生成日期: 2026-08-12
> 目标: 在 react-native-windows (RNW) 应用上运行 Supramark

## 一、总览矩阵

| 模块 | iOS | Android | macOS | **Windows** | 说明 |
|------|:---:|:-------:|:-----:|:-----------:|------|
| Markdown 基础渲染 | ✅ | ✅ | ✅ | ⚠️ JS 层可用 | 原生 parser 无 Windows 绑定，fallback wasm |
| Mermaid | ✅ native | ✅ native | ✅ native | ❌ | 无 Windows 原生绑定 |
| Dot/Graphviz | ✅ native | ✅ native | ✅ native | ⚠️ 有 C++ 代码 | 二进制未构建，Frameworks 目录为空 |
| D2 | ✅ native | ✅ native | ✅ native | ❌ | 无 Windows 原生绑定 |
| PlantUML | ✅ native | ✅ native | ✅ native | ❌ | 无 Windows 原生绑定 |
| ECharts | ✅ JS | ✅ JS | ✅ JS | ⚠️ 未验证 | Hermes/wasm 兼容性未知 |
| Vega-Lite | ✅ JS | ✅ JS | ✅ JS | ⚠️ 未验证 | 同上 |
| MathJax (数学公式) | ✅ JS | ✅ JS | ✅ JS | ⚠️ JS 可跑 | mathjax-full 在 Hermes 未验证 |
| 代码高亮 | ✅ | ✅ | ✅ | ✅ | 纯 JS，无原生依赖 |
| GFM (表格/任务列表等) | ✅ | ✅ | ✅ | ✅ | 纯 RN 组件 |
| Footnote / Admonition / Definition List | ✅ | ✅ | ✅ | ✅ | 纯 RN 组件 |
| Emoji | ✅ | ✅ | ✅ | ✅ | 纯文本替换 |
| Map (react-native-maps) | ✅ | ✅ | ⚠️ | ❌ | react-native-maps 不支持 Windows（本阶段忽略） |
| HTML Page | ✅ | ✅ | ✅ | ✅ | 回调式，宿主自行处理 |
| 文本选区 (rn-selection) | ✅ | ✅ | ⚠️ | ❌ | iOS/Android 专属（本阶段忽略） |
| Vison 卡片 | ✅ | ✅ | ✅ | ⚠️ | 依赖 react-native-svg（本阶段忽略） |

---

## 二、详细问题分析

### P0 — 完全缺失，阻断使用

#### 1. Mermaid / D2 / PlantUML — 无 Windows 原生绑定

这三个图表引擎在 iOS/Android/macOS 上通过 `@actrium/supramark-{mermaid,d2,plantuml}-native-rn` 提供 Rust → C ABI 的原生 FFI 绑定。

**现状：**
- `crates/mermaid-little/packages/react-native/` — 只有 `ios/`、`android/`、`macos/` 目录，**无 `windows/`**
- `crates/d2-little/packages/react-native/` — 同上
- `crates/plantuml-little/packages/react-native/` — 同上
- 这三个包的 `package.json` 中 `files` 字段只列出 `ios, macos, android`，**不含 `windows`**
- 没有 `build-windows.sh`（只有 graphviz-anywhere 有）

**影响：** 这三个引擎在 Windows 上完全不可用。`createReactNativeDiagramEngine` 查找不到 native adapter → fallback 到 `engine.ts` 的 `renderMermaidSvg` → 尝试 `import('@actrium/mermaid-little-web')`（wasm）→ 在 Hermes 上 wasm 加载**大概率失败**。

**需要做的工作（每个引擎）：**
1. 为 Rust crate 编写 Windows x86_64 (MSVC) 交叉编译脚本 `build-windows.sh`
2. 创建 `packages/react-native/windows/` 目录，编写 C++/WinRT `ReactPackageProvider` + `NativeModule`（参考 graphviz-anywhere 的 `GraphvizModule.cpp`）
3. 更新 `package.json` 的 `files` 字段加入 `windows`
4. 更新 `peerDependenciesMeta` 加入 `react-native-windows`
5. 更新 `prepare-native.js` 脚本处理 Windows 二进制 staging

#### 2. react-native-svg 在 Windows 的验证

**现状：** `@supramark/rn` 的 `peerDependencies` 要求 `react-native-svg >= 13.0.0`，所有图表/数学公式的渲染都通过 `SvgXml` 组件。

`react-native-svg` 确实有 Windows 支持（由 react-native-windows 社区维护），但需要确认：
- 版本兼容性（需要 >= 15.x 才有完整的 Windows 支持）
- `SvgXml` 解析器的行为差异
- `foreignObject` 不被支持（代码已在 `svgUtils.ts` 中做了 `foreignObject → text` 转换）

#### 3. 文本选区 (rn-selection) — iOS/Android 专属（本阶段忽略）

`packages/renderers/rn-selection/src/` 中有 `Platform.OS !== 'android'` 硬编码判断，手柄渲染依赖 PanResponder + 原生坐标。Windows 上不可用。

---

### P1 — 有基础框架但不完整

#### 4. Dot/Graphviz — C++ 模块已写，但二进制未构建

**已有的：**
- `crates/graphviz-anywhere/packages/react-native/windows/GraphvizNative/` — C++/WinRT 模块代码完整（`GraphvizModule.cpp`、`ReactPackageProvider.cpp`）
- `crates/graphviz-anywhere/scripts/build-windows.sh` 存在
- `package.json` 的 `files` 包含 `windows`，`peerDependencies` 包含 `react-native-windows`
- JS 层 `src/index.ts` 的 `Platform.select` 已包含 `windows: ''` 分支

**缺失的：**
- `output/windows-x86_64/` 目录**为空** — Windows 二进制 (`graphviz_api.dll` / `graphviz_api.lib`) 从未被构建
- `windows/Frameworks/` 目录**为空** — `prepare-native.js` 无法 staging 产物
- RN Windows 项目的 autolinking 配置未验证

**影响：** Dot 图表引擎代码框架完备，但开箱即用会报 `NOT_INITIALIZED` 错误。需要先在 Windows 机器上跑 `build-windows.sh` 编译 Graphviz 源码。

#### 5. supramark-markdown 原生 parser — 无 Windows 绑定

**现状：**
- `crates/supramark-markdown/packages/react-native/` 的 `package.json` 只支持 `ios, macos, android`
- **无 `windows/` 目录**
- JS 入口 `src/index.ts` 的 `Platform.select` **没有 `windows` 分支**（只有 ios / android）

**影响：** 在 Windows 上 `@supramark/markdown-native-rn` 无法链接，fallback 到 `@supramark/markdown-web`（wasm）。如果 wasm 在 Hermes 上能跑，Markdown 基础渲染可用；如果不能，`parse()` 失败。

#### 6. ECharts / Vega-Lite — JS 路径但 Hermes 兼容性未验证

**现状：** 这两个引擎走 `js-chart-loaders.ts` → `import('echarts/core')` / `import('vega')`。

**潜在问题：**
- ECharts 5.x 的 SVG SSR 渲染在 Hermes 上**未经验证**
- Vega 5.x 依赖大量 ES 特性（`Proxy`、`Symbol` 等）
- Metro bundler 需要正确配置才能 tree-shake echarts 的子路径导入

#### 7. MathJax — mathjax-full 在 Hermes 未验证

**现状：** `packages/engines/src/mathjax/index.ts` 使用 `mathjax-full` 纯 JS 包，通过 `liteAdaptor`（无 DOM 依赖）。

`liteAdaptor` 不依赖 DOM，理论上是兼容的，但 `mathjax-full` 在 Hermes 上未验证。

---

### P2 — 低风险，基本可用

#### 8. Map (react-native-maps)

`react-native-maps` 不支持 Windows。但代码有 `try/catch` fallback 到 placeholder card，**不会崩溃**。本阶段忽略。

#### 9. 纯 RN 组件层

段落、标题、代码块、列表、表格、引用块、定义列表、脚注、admonition、emoji 等 — 全部使用核心 RN 组件（`Text`、`View`、`ScrollView`），在 Windows 上应该正常工作。

#### 10. Linking.openURL

`Supramark.tsx` 使用 `Linking.openURL` 打开链接。react-native-windows 支持 `Linking` API，应该可用。

---

## 三、SVG 渲染管线的 Windows 特定风险

| 处理逻辑 | 位置 | Windows 风险 |
|---------|------|-------------|
| CSS `<style>` 解析 → 属性内联 | `svgUtils.ts normalizeSvg:165-269` | 低 — 纯 JS 字符串操作 |
| `foreignObject → text` 转换 | `svgUtils.ts normalizeSvg:283-306` | 低 — 已规避了 foreignObject |
| `stripRootSvgSize` | `svgUtils.ts:346-353` | 低 — 纯正则 |
| `viewBox` 注入 | `DiagramNode.tsx:254-258` | 中 — 需验证 Windows SvgXml 的 preserveAspectRatio |
| `Dimensions.get('window')` | `DiagramNode.tsx:214` | 低 — Windows 支持 |

---

## 四、本阶段处理范围

**处理：**
1. 基础 Markdown 渲染（确保 parse + 纯文本组件在 Windows 上工作）
2. 图表引擎（graphviz 补全二进制构建、mermaid/d2/plantuml 创建 Windows 绑定、echarts/vega/mathjax 验证）

**忽略：**
- Map（react-native-maps）
- 文本选区（rn-selection）
- Vison 卡片
