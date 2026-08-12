# Supramark React Native Windows 运行准备指南

> 本文档记录在 react-native-windows (RNW) 应用上运行 Supramark 所需的全部准备工作。

## 前提条件

### 开发环境

| 工具 | 版本要求 | 说明 |
|------|---------|------|
| Windows | 10/11 | 必须在 Windows 上构建原生库 |
| Visual Studio | 2019+ (2022 推荐) | 需安装「Desktop development with C++」工作负载 |
| Node.js | 18+ | RNW 运行时 |
| Rust | stable | `rustup default stable` |
| CMake | 3.16+ | Graphviz 构建需要 |
| Git Bash / MSYS2 | 任意 | 执行 `.sh` 构建脚本 |

### Rust 交叉编译 target

```bash
rustup target add x86_64-pc-windows-msvc
# 如需 ARM64:
rustup target add aarch64-pc-windows-msvc
```

---

## 第一步：构建原生 Rust 库

每个图表引擎和 Markdown parser 都有一个 Rust → C ABI 的 native crate，需要在 Windows 上编译为 `.dll`。

### 1.1 Markdown Parser

```bash
cd crates/supramark-markdown
./scripts/build-windows.sh
# 产出: output/windows-x86_64/bin/supramark_markdown_native.dll
#       output/windows-x86_64/include/supramark_markdown.h

# Stage 到 RN 包:
cd packages/react-native
npm run prepare-native
# 产出: windows/Frameworks/{bin/*.dll, include/*.h}
```

### 1.2 Mermaid

```bash
cd crates/mermaid-little
./scripts/build-windows.sh
# 产出: output/windows-x86_64/bin/supramark_mermaid_native.dll

cd packages/react-native
npm run prepare-native
```

### 1.3 D2

```bash
cd crates/d2-little
./scripts/build-windows.sh
# 产出: output/windows-x86_64/bin/supramark_d2_native.dll

cd packages/react-native
npm run prepare-native
```

### 1.4 PlantUML

```bash
cd crates/plantuml-little
./scripts/build-windows.sh
# 产出: output/windows-x86_64/bin/supramark_plantuml_native.dll

cd packages/react-native
npm run prepare-native
```

### 1.5 Graphviz (Dot)

Graphviz 比较特殊 — 它编译 C 源码而非纯 Rust：

```bash
cd crates/graphviz-anywhere
./scripts/build-windows.sh
# 产出: output/windows-x86_64/bin/graphviz_api.dll
#       output/windows-x86_64/lib/graphviz_api_static.lib
#       output/windows-x86_64/include/graphviz_api.h

cd packages/react-native
npm run prepare-native
```

> **注意：** Graphviz 构建依赖 CMake + Visual Studio，且会编译完整的 Graphviz C 源码树。首次构建可能需要 10-20 分钟。

---

## 第二步：宿主项目配置

### 2.1 安装依赖

在宿主 RNW 项目的 `package.json` 中添加以下依赖：

```jsonc
{
  "dependencies": {
    // Supramark 核心
    "@supramark/core": "workspace:*",
    "@supramark/rn": "workspace:*",

    // 原生引擎（side-effect import 注册）
    "@supramark/markdown-native-rn": "workspace:*",
    "@actrium/supramark-mermaid-native-rn": "workspace:*",
    "@actrium/supramark-d2-native-rn": "workspace:*",
    "@actrium/supramark-plantuml-native-rn": "workspace:*",
    "@actrium/graphviz-anywhere-rn": "workspace:*",

    // 纯 JS 引擎的 peer 依赖（ECharts / Vega-Lite）
    "echarts": "^5.5.0",
    "vega": "^5.30.0",
    "vega-lite": "^5.21.0",

    // SVG 渲染（图表/公式的显示基础）
    "react-native-svg": ">=15.0.0",

    // RNW 平台
    "react-native-windows": ">=0.72.0"
  }
}
```

### 2.2 Metro 配置

宿主的 `metro.config.js` 需要以下关键配置（参考 `examples/react-native/metro.config.js`）：

```javascript
const path = require('path');
const { getDefaultConfig } = require('metro-config');

const config = getDefaultConfig(__dirname);
const workspaceRoot = path.resolve(__dirname, '../..');

// 1. monorepo 支持
config.watchFolders = [workspaceRoot];
config.resolver.nodeModulesPaths = [
  path.resolve(__dirname, 'node_modules'),
  path.resolve(workspaceRoot, 'node_modules'),
];

const defaultResolver = config.resolver.resolveRequest;

config.resolver.resolveRequest = (context, moduleName, platform) => {
  // 2. plugin-loader 路由：RN 必须走 native parser loader，不能走 wasm
  if (platform !== 'web' && /(^|\/)plugin-loader(-web)?(\.(js|ts))?$/.test(moduleName)) {
    return {
      filePath: path.resolve(workspaceRoot, 'packages/core/src/plugin-loader-rn.ts'),
      type: 'sourceFile',
    };
  }

  // 3. @supramark/core 的 RN 入口
  if (moduleName === '@supramark/core') {
    return {
      filePath: path.resolve(workspaceRoot, 'packages/core/src/index.rn.ts'),
      type: 'sourceFile',
    };
  }

  // 4. @supramark/core/rn 子路径
  if (moduleName === '@supramark/core/rn') {
    return {
      filePath: path.resolve(workspaceRoot, 'packages/core/src/index.rn.ts'),
      type: 'sourceFile',
    };
  }

  // 5. @supramark/engines/rn 子路径
  if (moduleName === '@supramark/engines/rn') {
    return {
      filePath: path.resolve(workspaceRoot, 'packages/engines/src/rn.ts'),
      type: 'sourceFile',
    };
  }

  // 6. wasm web 包 stub（RN 绝不加载 wasm）
  if (/^@(kookyleo|actrium)\/(d2|mermaid|plantuml)-little-web$|^@(kookyleo|actrium)\/graphviz-anywhere-web$/.test(moduleName)) {
    return {
      filePath: path.resolve(__dirname, 'stubs/empty.js'),
      type: 'sourceFile',
    };
  }

  // 7. .js → .ts 后缀重映射（ESM 风格导入）
  if ((moduleName.startsWith('./') || moduleName.startsWith('../')) && moduleName.endsWith('.js')) {
    const stripped = moduleName.slice(0, -3);
    try {
      return context.resolveRequest(context, stripped, platform);
    } catch {
      // fall through
    }
  }

  if (defaultResolver) {
    return defaultResolver(context, moduleName, platform);
  }
  return context.resolveRequest(context, moduleName, platform);
};

module.exports = config;
```

### 2.3 创建空 stub 文件

```bash
mkdir -p stubs
echo "module.exports = {};" > stubs/empty.js
```

### 2.4 应用入口注册原生引擎

在宿主 App 的入口文件（如 `index.js` 或 `App.tsx`）顶部，**在任何 `@supramark/*` import 之前**添加 side-effect import：

```typescript
// === 必须最先导入：注册原生引擎适配器 ===

// Markdown parser（parse() 依赖此注册）
import '@supramark/markdown-native-rn';

// 图表引擎（createReactNativeDiagramEngine 依赖这些注册）
import '@actrium/supramark-mermaid-native-rn';
import '@actrium/supramark-d2-native-rn';
import '@actrium/supramark-plantuml-native-rn';
// Dot/Graphviz 通过 @actrium/graphviz-anywhere-rn 自动注册

// === 然后导入应用代码 ===
import { Supramark } from '@supramark/rn';
// ...
```

### 2.5 React Native Windows 项目生成

如果宿主项目还没有 Windows 原生工程：

```bash
npx react-native-windows-init --overwrite
```

这会生成 `windows/` 目录，包含：
- `<AppName>/` — C# UWP 应用工程
- `<AppName>.sln` — Visual Studio 解决方案

---

## 第三步：构建和运行

### 3.1 构建 RNW 应用

```bash
# 通过 CLI（推荐）
npx react-native run-windows

# 或通过 Visual Studio
# 打开 windows/<AppName>.sln → 选择 x64 → Release → Build
```

### 3.2 启动 Metro

```bash
npx react-native start
```

### 3.3 运行应用

```bash
npx react-native run-windows
```

---

## 第四步：ECharts / Vega-Lite / MathJax 的注意事项

这三个引擎是纯 JS 实现，不需要原生模块，但在 Hermes 上有以下注意事项：

### ECharts

- ECharts 5.x 的 SVG renderer 使用 `CanvasGradient` 等 API 的 polyfill — Metro 需要正确配置
- 子路径导入 `echarts/core`、`echarts/charts` 等需要 Metro 支持 `exports` 字段（Metro 0.76+ 默认支持）
- 如果遇到 `Cannot read property 'xxx' of undefined`，检查 Metro 版本是否 >= 0.76

### Vega-Lite

- Vega 5.x 使用 `Proxy` 和 `Symbol` — Hermes 74+ 默认启用
- 如果 Hermes 版本较低，需要在 `babel.config.js` 中添加 polyfill：
  ```bash
  npm install react-native-proxy-polyfill
  ```
  ```javascript
  // 在入口最前面
  import 'react-native-proxy-polyfill';
  ```

### MathJax

- `mathjax-full` 使用 `liteAdaptor`（不依赖 DOM），理论上在 Hermes 上直接可用
- 如果遇到性能问题（公式渲染慢），考虑通过 host bridge 注入原生字体测量

---

## 故障排除

### "RN runtime requires native markdown parser adapter"

**原因：** `@supramark/markdown-native-rn` 未被 side-effect import。

**解决：** 确保在 App 入口最顶部添加 `import '@supramark/markdown-native-rn';`

### "The package '@supramark/markdown-native-rn' doesn't seem to be linked"

**原因：** Windows 原生模块未正确链接。

**解决：**
1. 确认 `prepare-native` 已运行，`windows/Frameworks/bin/` 下有 `.dll`
2. 确认 `react-native.config.js` 没有禁用 Windows autolinking
3. 运行 `npx react-native-windows-init --overwrite` 重新生成项目

### 图表显示空白或报 "render_error"

**原因：** 对应引擎的 DLL 未构建或未链接。

**解决：**
1. 确认对应引擎的 `build-windows.sh` 已执行
2. 确认 `prepare-native` 已 staging DLL 到 `windows/Frameworks/`
3. 检查 Visual Studio Output 窗口是否有链接错误

### Graphviz 报 "NOT_INITIALIZED"

**原因：** `graphviz_api.dll` 未构建或未部署。

**解决：**
1. 运行 `crates/graphviz-anywhere/scripts/build-windows.sh`
2. 运行 `crates/graphviz-anywhere/packages/react-native/scripts/prepare-native.js`
3. 确认 `crates/graphviz-anywhere/packages/react-native/windows/Frameworks/bin/graphviz_api.dll` 存在

### Metro 报 "Cannot resolve module '@supramark/markdown-web'"

**原因：** Metro 尝试加载 wasm 包。

**解决：** 确认 `metro.config.js` 中 `plugin-loader` 重定向规则存在（见第二步 2.2）

### Hermes 报 "Proxy is not defined" (Vega-Lite)

**解决：** 添加 Proxy polyfill（见第四步 Vega-Lite 部分）

---

## 架构概览

```
宿主 RNW App
├── metro.config.js          ← Metro 解析 + wasm stub
├── App.tsx
│   ├── import '@supramark/markdown-native-rn'    ← side-effect 注册
│   ├── import '@actrium/supramark-mermaid-native-rn'
│   ├── import '@actrium/supramark-d2-native-rn'
│   ├── import '@actrium/supramark-plantuml-native-rn'
│   └── <Supramark markdown="..." />              ← 渲染
│
├── windows/
│   └── <AppName>/
│       └── AutoLinkedNativeModules.g.h           ← RNW 自动链接
│           ├── SupramarkMarkdownNative            → supramark_markdown_native.dll
│           ├── SupramarkMermaidNative             → supramark_mermaid_native.dll
│           ├── SupramarkD2Native                  → supramark_d2_native.dll
│           ├── SupramarkPlantumlNative            → supramark_plantuml_native.dll
│           └── GraphvizNative                     → graphviz_api.dll
│
└── node_modules/
    ├── @supramark/rn/                             ← RN 渲染组件
    ├── @supramark/engines/                        ← 引擎路由
    ├── @supramark/markdown-native-rn/windows/     ← C++/WinRT 模块源码
    ├── @actrium/supramark-mermaid-native-rn/windows/
    ├── @actrium/supramark-d2-native-rn/windows/
    ├── @actrium/supramark-plantuml-native-rn/windows/
    └── @actrium/graphviz-anywhere-rn/windows/
```

## 各分支对应关系

| 分支 | 内容 |
|------|------|
| `feat/windows-base-markdown` | `@supramark/markdown-native-rn` Windows 支持 |
| `feat/windows-diagram-engines` | mermaid / d2 / plantuml native engines Windows 支持 |

Graphviz 的 Windows C++ 模块代码已存在于 `upstream/main`（`crates/graphviz-anywhere/packages/react-native/windows/`），只需在 Windows 机器上运行 `build-windows.sh` + `prepare-native` 即可。
