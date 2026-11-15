# HANDOFF · dsh-chat-columns（给接手的 agent）

> 1.0.0-beta.2。读者是人类，但这份是给**下一个人/agent**看的：只写「怎么接、别踩什么、坏了去哪找」。
> 用户视角的说明在 `README.md`，两边不要互相复制。

## 1. 它是什么

一个 DSH Web 客户端插件：把会话正文按四种「读法」组织（原生 / 报纸 / 无缝 / 复制）。
只有浏览器半边（`lib/client.js`），服务端 `index.js` 是空壳，只为让包出现在插件清单里。

## 2. 目录

```
src/                 源码（改这里，不是 lib/）
  index.js           插件入口：apply / mount / unmount、模式注册、__dshc 调试 API
  controller.js      控制器 UI（右上角胶囊 + 面板）、模式/列数/实现档切换
  adapter.js         宿主适配层：唯一允许碰宿主 DOM 细节的地方
  flow.js            消息流抽象：识别消息、收集装饰节点、变更订阅
  indexer.js         消息索引：id/序号/类型
  math.js            纯函数：列数、装箱、逻辑位置投影（有单测）
  newspaper.js       报纸模式
  seamless.js        无缝模式（竖向）：粘滞视口 + 逻辑窗口 + 自己换算滚动
  paged.js           纸带引擎：横向 / 翻页
  mirror.js          复制模式：克隆 + 净化 + N 栏独立滚动
  native-flow.js     原生流的包装/还原（切模式时用）
  native-mirror.js   原生流镜像
  layout.css         所有样式，作用域限定在 data-dshc-* 下
  tokens.css         设计令牌
lib/                 构建产物（build 生成，不要手改）
scripts/             build / test / check / smoke / install
docs/                宿主契约审计、无缝模型说明
```

## 3. 跑起来

```bash
npm run verify     # build + test + check + smoke，提交前必须全绿
```

装进 profile：

```bash
node scripts/install.mjs --profile web      # 建链接 + 写 bundles + 备份
```

**改了 `src/` 一定要 `npm run build`，然后重启 `dsh-web`。**
`lib/client.js` 在启动时被读进内存并按 rev 缓存，只刷新浏览器页面还是旧字节。

## 4. 宿主契约

- 目标宿主：DSH Web **0.2.1-alpha.2**。宿主差异**全部收敛在 `adapter.js`**，其它模块不许直接查宿主 DOM。
- 判断主会话：`[data-slot="main"] [data-conversation-scroll]`；右栏 / 子智能体的 embedded 会话必须排除。
- 内容结构：`[data-chat-flow]` 是列容器，行是 `[data-chat-flow-key]`。
- 「加载更早」是**按钮**，锚点补偿绑在它的 onClick 上——只能 `button.click()`，不能直接调服务。

## 5. 不变量（违反就会坏）

1. **不搬 React 节点**：不能把宿主节点移动到别处，否则 keyed seat 会重复/错乱。
2. **不克隆带交互状态的 DOM 当可交互副本**：克隆没有监听器，只能是只读。
3. **一切 DOM 操作必须可逆**：`unmount` 后宿主必须回到初始状态（smoke 会验）。
4. **Observer / 定时器 / 监听器必须能释放**（smoke 验「没有遗留定时器」）。
5. **插件的样式只能写在 `data-dshc-*` 作用域下**，不许污染宿主选择器（`check.mjs` 会拦）。

## 6. 已知缺陷的代码位置

| 现象 | 位置 | 性质 |
|---|---|---|
| 复制模式只读 | `src/mirror.js:102` `sanitized()` + `adapter.sanitize` | 结构使然 |
| 交互档置灰 | `src/controller.js:349-360`（`implBlocked`） | 等宿主 `renderSlot` |
| 跨缝选中会断 / Ctrl+F 两份 | `src/seamless.js:5-10` 模型注释 | 模型使然 |
| 代码块被切开 / 长行软换行 | `src/layout.css:403-420` | 设计代价 |
| 列底留白（消息不跨列） | `src/seamless.js:201`、`:231` | 设计代价 |
| 超高消息内部滚动 | `src/seamless.js:233-236` | 设计代价 |
| 宿主锚点与插件补偿可能抖 | `src/seamless.js:125`、`:280` | **待真机验证** |
| 复制模式 N 份 DOM | `src/mirror.js:56` | 设计代价 |
| 超宽表格被压 | `src/layout.css:404` | 设计代价 |
| 宿主「加载更早」被收起 | `src/flow.js:90` | 设计取舍 |

## 7. 调试

浏览器控制台：

```js
__dshc.diagnose()        // 一行摘要：版本 / 模式 / 主会话 / 列数 / 各层状态
__dshc.getState()        // 结构化状态
__dshc.setMode('seamless')
__dshc.unmount()         // 手动卸载，验证可逆
```

出问题先要 `__dshc.diagnose()` 和完整报错，别只要一句「坏了」。

## 8. 踩过的坑

1. **改了 `src/` 忘记 build**：页面看着没变化，其实是 `lib/client.js` 没更新。
2. **build 了没重启 `dsh-web`**：同上，启动时读缓存。
3. **手改 `node_modules` 软链**：pnpm 一 reconcile 就把手改的链接删掉，插件直接不加载。要用 `install.mjs` 或正规 `link:` 依赖。
4. **profile 里留两代包名**（`dsh-chat-columns` 和 `dsh-chat-columns-3`）：两个插件抢同一块 DOM，启动报 `required plugins did not activate`。升级时把旧的从 deps 和 bundles 里删干净。
5. **`git push` 到 github.com 经常断**：兜底用 Git Data API（blob → tree → commit → PATCH ref），这个环境一直能用。
