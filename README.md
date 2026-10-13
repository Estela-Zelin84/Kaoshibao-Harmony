# 考试宝 · 鸿蒙 Web 封装应用（HAP）

一个仿照 **DeepSeek 1.1.4 HAP** 结构与封装思路的鸿蒙应用：用 ArkTS 的 `<Web>` 组件把"考试宝"网站（或内置离线题库）包进一个原生 App 里。这就是所谓 "Web 封装 / H5 套壳" 的 HAP。

## 应用介绍

**考试宝 · 鸿蒙版**是一款专为鸿蒙（HarmonyOS / OpenHarmony）设备打造的轻量刷题工具，目的只有一个——**让在鸿蒙上用考试宝更顺手**。

考试宝官方的鸿蒙原生客户端体验并不理想：界面臃肿、广告与浮窗多、启动还要绕一圈才进得去。这个应用直接把考试宝的网页端用原生 WebView 套进一个干净的 App 里，并做了针对性的减法与优化：

- **打开即用**：启动直达登录页，登录后自动停留在应用内，免去来回跳转的麻烦。
- **精简页面**：自动隐藏下载 App 浮窗、页脚、合作伙伴墙、AI 推广条、营销数字和各类弹窗遮罩，只保留导航、学科分类、搜索与题库内容。
- **专注刷题**：界面更干净、干扰更少，把屏幕留给真正要做的题。
- **离线兜底**：内置一份离线题库网页，断网也能随手刷几道，完全免登录。

一句话：**不是另一个考试宝，而是更清爽、更好用的考试宝入口。**

> 参考来源：你提供的 `DeepSeek_1.1.4.hap` 本质上就是用 `WebviewController` 把 `https://chat.deepseek.com` 包进 `<Web>` 组件的 WebView 应用，本工程完全复刻了它的工程结构与封装方式。

**本工程已用 OpenHarmony 5.0.0 Public SDK（API 12）+ 命令行 hvigor 工具链成功构建出未签名 HAP。**

---

## 工程结构

```
KaoshibaoWrapper/
├── AppScope/                      # 应用级配置与图标
│   ├── app.json5                  # bundleName / 版本 / 应用名
│   └── resources/base/...
├── entry/                         # 入口模块（真正打包进 HAP 的内容）
│   ├── src/main/
│   │   ├── ets/
│   │   │   ├── entryability/EntryAbility.ets     # Ability 入口
│   │   │   └── pages/Index.ets                   # ★ WebView 封装页（核心）
│   │   ├── resources/base/...                   # 字符串/颜色/图标/路由
│   │   └── resources/rawfile/quiz/index.html     # 内置离线题库网页
│   ├── module.json5               # 模块配置 / 权限 / 页面路由
│   └── build-profile.json5
├── build-profile.json5            # 构建配置（未签名：无 signingConfigs）
├── hvigorfile.ts / entry/hvigorfile.ts
├── hvigor/hvigor-config.json5
├── oh-package.json5 / local.properties
└── build.sh                       # 一键命令行构建脚本
```

---

## 环境要求（命令行构建，免 DevEco / 免签名）

| 工具 | 版本 | 说明 |
|------|------|------|
| Node.js | 18 / 20 / 22 | 运行 hvigor |
| OpenHarmony SDK | 5.0.0.71（API 12，Public SDK） | 组件需放在**版本化目录**下 |
| hvigor + 插件 | 6.26.8 | 已装入本工程 `node_modules` |

### ⚠️ SDK 目录必须是"版本化"布局

hvigor 6.x 校验 SDK 时会把组件目录拼成 `sdk.dir/<apiVersion>/<component>`，因此 SDK 必须解压成如下结构（`sdk.dir` 指向 `linux` 这一层）：

```
<你的 sdk 根>/linux/
└── 12/                         ← 注意这层版本号目录（apiVersion=12）
    ├── ets/
    ├── js/
    ├── previewer/
    ├── toolchains/
    └── native/                 ← Native 组件也必须存在（否则报 license 未接受）
```

每个组件目录下需保留 `oh-uni-package.json`。若你的 SDK 是"平铺"解压（没有 `12/` 这一层），用 `mv ets js previewer toolchains native 12/` 调整即可。

> 更低版本的 hvigor 会报 `00308018 The SDK management mode has changed`，本质就是组件目录层级不匹配。

---

## 构建未签名 HAP（命令行）

直接用**工程内置的本地 hvigor** 构建（不要用全局 `hvigorw` 符号链接，否则会出现双重 hvigor 实例导致 `00302013 root node not available`）：

```bash
cd KaoshibaoWrapper
export OHOS_SDK_HOME=/opt/ohos-sdk/linux      # 指向 SDK 的 linux/ 目录
export PATH="$OHOS_SDK_HOME/toolchains:$PATH"
# 用本地 hvigor 单实例，避免双实例冲突
node node_modules/@ohos/hvigor/bin/hvigor.js assembleHap --mode module -p product=default --no-daemon
```

或用附带的一键脚本：

```bash
./build.sh
```

构建成功后产物在：

```
entry/build/default/outputs/default/entry-default-unsigned.hap
```

- 文件名带 `unsigned` 即**未签名** HAP（内部无 `META-INF` 签名目录）。
- `build-profile.json5` 的 `products.default` 故意**不引用 signingConfigs**，所以 hvigor 跳过 `SignHap` 步骤，直接产出未签名包。

### 安装 / 侧载

未签名 HAP 仅能安装到允许未签名应用的 OpenHarmony 设备 / 工程版镜像，例如：

```bash
hdc install entry-default-unsigned.hap
```

若设备要求签名，请改用 DevEco Studio 自动签名后重新构建（见下）。

---

## 在 DevEco Studio 中签名构建（可选）

1. 打开工程 → `File → Project Structure → Signing Configs` → 勾选 **Automatically generate signature**，登录华为账号后自动回填 `build-profile.json5` 的 `material`。
2. `Build → Build Hap(s)`，产物为已签名可安装包。
3. 当前 `module.json5` 用的是 OpenHarmony 公版字段（`virtualMachine: "ark"`、`deviceTypes: ["default","tablet"]`）。若切回 HarmonyOS NEXT 签名发布，把 `runtimeOS` 改回 `"HarmonyOS"` 并按 NEXT 规范补全 `runtime`/`compileMode` 等字段即可，封装逻辑无需改动。

---

## 自定义封装内容（改这两个常量即可）

打开 `entry/src/main/ets/pages/Index.ets`：

```ts
// 要封装的网址
const KAOSHIBAO_URL: string = 'https://www.kaoshibao.com/';

// true  -> 加载内置离线题库（resources/rawfile/quiz/index.html，无需联网）
// false -> 加载上面的远程考试宝网站
const USE_LOCAL_QUIZ: boolean = false;
```

- 换站点：改 `KAOSHIBAO_URL`。
- 完全离线：把 `USE_LOCAL_QUIZ` 设为 `true`，替换 `rawfile/quiz/index.html`（题库数组 `BANK` 可随意增删）。
- 改包名 / 应用名：改 `AppScope/app.json5` 的 `bundleName`、`label`，以及 `module.json5` 的 `label`。

### 启动页（登录界面）

默认 **打开应用先进入登录界面**：`START_AT_LOGIN = true` 时加载 `LOGIN_URL`（考试宝登录页 `https://www.kaoshibao.com/login/`）。登录后 cookie 留在 WebView 内，即可正常使用考试宝。

```ts
const LOGIN_URL: string = 'https://www.kaoshibao.com/login/';
const START_AT_LOGIN: boolean = true;   // false -> 打开直接进首页 KAOSHIBAO_URL
```

- 想打开直接进首页：把 `START_AT_LOGIN` 设为 `false`。
- 换登录页地址：改 `LOGIN_URL`（来源：官网首页里的 `href="/login/"`）。

### 网站精简（去掉没用的部分）

远程网站加载完成后会自动移除"没用的部分"——浮窗、页脚、合作伙伴墙、AI 推广条、营销数字、弹窗遮罩等，保留导航、学科分类、搜索与题库内容。

实现位置：`Index.ets` 顶部两个常量——

```ts
// 总开关：true=对远程网站做去冗余；内置离线题库不受影响
const TRIM_ENABLED: boolean = true;

// 要移除的元素（CSS 选择器）。想删更多/更少，改这个数组即可
const TRIM_SELECTORS: string[] = [
  '.right-float-window-box',  // 右侧"下载APP/咨询客服"浮窗
  'footer', '.footer-row',    // 页脚版权 / 快速导航
  '.hzhb',                    // "我们的客户"合作伙伴墙
  '.gaoxian',                 // 顶部 AI 能力推广横幅
  '.top-data',                // 顶部营销数据条（100亿+ 等）
  '.el-dialog__wrapper', '.el-popover', '.v-modal', '.el-overlay' // 弹窗 / 遮罩
];
```

原理：页面 `onPageEnd` 后用 `runJavaScript` 注入脚本，一次性移除 + `MutationObserver` 持续清理（应对延迟插入的弹窗/广告）。当前规则在考试宝首页实测命中 **26 个**冗余元素，未误伤任何核心功能。

> 注：该 OpenHarmony Public SDK 的 `Web` 组件不支持 `javaScriptOnDocumentStart`，故用 `runJavaScript` 方案；若站点改为纯自己的页面，把 `TRIM_ENABLED` 设为 `false` 即可。

### 权限说明

`module.json5` 已声明 `ohos.permission.INTERNET`（加载网页必需）和 `ohos.permission.GET_NETWORK_INFO`。用到定位、摄像头等需在此追加。

---

## 内置离线题库（考试宝风格 SPA）

`rawfile/quiz/index.html` 是一个开箱即用的纯前端刷题页：分类筛选、逐题作答、即时解析、进度条、得分统计、错题回顾、本地最佳成绩记录。交给 WebView 加载即为离线版"考试宝"，也可直接用浏览器打开预览。

---

## 踩坑记录（命令行构建 OpenHarmony Public SDK）

| 错误码 | 原因 | 解决 |
|--------|------|------|
| `00302013` root node not available | 全局 `hvigorw` 与本地插件加载出两套 hvigor 实例 | 用 `node node_modules/@ohos/hvigor/bin/hvigor.js` 本地单实例构建 |
| `00308018` management mode has changed | SDK 组件未放在 `sdk.dir/12/` 版本化目录 | 重组为 `12/{ets,js,previewer,toolchains,native}` |
| `00308018` license not accepted (Native) | 缺少 native 组件 | 解压并放入 `12/native` |
| `00303038` / `00303315` | `build-profile.json5` / `hvigor-config.json5` 字段与 modelVersion 不匹配 | 按本工程已调好的写法（`compileSdkVersion: 12` 数字、`nodeOptions` 放顶层） |
| `00303060` syscap 交集为空 | `deviceTypes` 含 SDK 不支持的 `phone`/`2in1` | 改为 `["default","tablet"]` |
| ArkTS 报错 `@kit.CoreBackupKit` 不存在 | 该 kit 在 OpenHarmony Public SDK 无 | 删除 backup 扩展 ability |
| ArkTS 报错 `MixedMode`/`CacheMode` 未导出 | 该 SDK 的 `@kit.ArkWeb` 版本较低 | 移除 `mixedMode`/`cacheMode` 调用与导入 |

---

## 注意事项

- 把第三方网站直接装入 WebView 并作为 App 发布，涉及版权与平台合规，请仅用于学习/自用或加载**你自己拥有**的内容。
- Release 包记得在 `Index.ets` 中把 `setWebDebuggingAccess(true)` 改为 `false`。
