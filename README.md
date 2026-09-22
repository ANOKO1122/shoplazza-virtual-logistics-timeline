# shoplazza-virtual-logistics-timeline

店匠（Shoplazza）**订单详情页**的物流展示模块。一段可以直接粘贴进主题模板的 Liquid 片段，含配套的内联 JS / CSS。

- 订单**未发货**时 → 显示「Order Progress」订单进度时间轴
- 订单**已发货**时 → 显示真实物流查询（自动探测运单号 + 按承运商路由）
- 全程顺带把承运商品牌名 `Yanwen` 从浏览器翻译里保护起来

> 目前这套代码已经在店铺上使用。文件名保持中文，方便和店匠后台/主题模板对照。

## 文件说明

| 文件 | 说明 |
| --- | --- |
| `代码.txt` | 模块全部代码（Liquid 判断 + 两个内联 `<script>` + CSS），整段粘贴到订单详情页模板 |
| `操作步骤.docx` | 后台操作步骤（粘贴位置、保存、验证），含截图 |

## 功能拆解

### 1. Yanwen 品牌名保护

店匠的物流组件是**异步渲染**的，而且品牌名会被 Chrome / 浏览器翻译改写成中文（"燕文"），导致和 TRACK718 的承运商匹配、以及用户对单号的理解都对不上。

- 把物流区域里出现的 `Yanwen` 文本替换成 `<span class="notranslate vt-carrier-brand" translate="no" lang="en">`，并给该行打上 `data-vt-yanwen-*` 标记
- 用 `MutationObserver` 盯着整个文档，组件晚渲染、重渲染、被整体 `innerHTML` 覆盖都能重新保护
- 顺便把运单号缓存到 `window.VTYanwenTrackingIds`，给下面的路由用
- 对外暴露 `window.VTRefreshYanwen()`，可同步刷新缓存

### 2. 虚拟物流时间轴（未发货阶段）

条件：`financial_status == 'paid'` 且 `fulfillment_status` 属于 `blank / initialled / waiting`。

渲染 7 个固定节点（Payment Successful → Order Confirmed → Items Processing → Quality Inspection → Packaging → Awaiting Carrier Pickup → Shipped）：

- 时间 = 订单时间（优先 `placed_at`，回退 `created_at`）+ 写死的分钟偏移（约 0 / 12h / 24h / 72h / 4d / 6d / 10d）
- 分钟和秒的偏移用 `order.id` 哈希做种子的伪随机数 → **同一订单每次刷新时间固定不变**
- 只输出已经"到点"的节点，最新完成的最上面
- 时间是按访客浏览器本地时区格式化的

### 3. 真实物流查询（已发货阶段）

条件：`fulfillment_status` 属于 `shipped / partially_shipped`。

- 店匠服务端不给 `tracking_number`，所以页面加载后轮询 DOM 抓单号：① 店匠按钮的 `tracking-id` 属性；② `Tracking number: XXXX` 文本节点；③ 物流区域外再兜底扫一次
- 路由：命中 Yanwen 标记，或前缀是 `BST / GZX / LR / ST / WBD / XY` → **TRACK718**；其他 → **17TRACK**
- 探测失败就显示提示，让用户手动粘贴单号查询
- 两个平台的外部脚本和样式都是**首次真正查询时才按需加载**，加载失败会给出提示
- 查询结果用跨域 iframe 渲染，高度按视口动态限制（350–560px）

## 使用

见 `操作步骤.docx`。简单说：

1. 店匠后台 → 店铺设计 / 主题编辑 → 找到**订单详情页**模板
2. 把 `代码.txt` 的全部内容粘贴到物流信息区域对应的位置并保存
3. 用一笔已付款订单看时间轴；用一笔已发货订单看运单号自动探测与查询

## 注意事项

- **时间轴上的时间是虚构的**：完成时间和"已完成"状态都不是真实物流事件，用来填补"已付款但还没发货"的空窗期。如果站点所在的平台/地区对物流时效展示有合规要求，请自行评估。
- 查询依赖第三方 `static.track718.net` 与 `www.17track.net`，运单号会发送给这两个平台。
- TRACK718 / 17TRACK 的脚本按需加载，用户首次点查询会多 100–300ms 等待。
- 时间按访客浏览器本地时区显示，不同地区用户看到的时刻不同。
- 页面级监听只对物流区域内的 DOM 变化做重扫；如果店匠改了物流组件的 class 名（`.shipment-package-container` / `.shipment-package` / `id^="shipment-tracking-"`），需要同步调整代码里的选择器常量。

## 说明

代码以 Liquid 片段形式分发，没有构建步骤，也不依赖任何 npm 包。本仓库只做版本留存与分发。
