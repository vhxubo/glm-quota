# GLM Coding Plan 额度查询

输入 GLM Coding Plan 的 API Key，即可查看额度用量的纯静态单页。无后端、无依赖、无构建、零第三方请求（字体使用系统等宽字体栈）。

## 使用

- 直接双击打开 `index.html` 即可（`file://` 下已验证可用），或起个静态服务：
  ```bash
  python3 -m http.server 8747
  # 打开 http://127.0.0.1:8747/
  ```
- 粘贴 API Key（z.ai 或智谱开放平台的 API Keys 页面可找到），点击「查询用量」。
- 可选：记住 Key（仅存 localStorage）、每 60 秒自动刷新、切换接口站点。

## 展示内容

| 区块 | 来源接口 | 说明 |
|---|---|---|
| 额度窗口进度条：5 小时窗口 + 本周 | `GET /api/monitor/usage/quota/limit` | `limits[]` 中两个 `type === 'TOKENS_LIMIT'` 条目，靠 `unit`/`number` 区分：`unit:3, number:5` = 5 小时窗口，`unit:6, number:1` = 本周。字段：`percentage`、`total`（token 上限）、`currentValue`（已用，缺省时按 `total × percentage/100` 推导）、`nextResetTime`（重置时间戳，显示为完整日期时间 + 倒计时，如「重置 2026-10-13（周二）22:00 · 36 小时 0 分钟后」）。响应 `data.level` 为套餐等级，显示在面板右上角 |
| 其他限额（如 MCP） | 同上 | `type === 'TIME_LIMIT'` 即 MCP 月度调用；其余未知类型以原始名称渲染为进度条，副行显示 `nextResetTime` 的重置时间与倒计时 |
| 今日 / 近 7 天 / 近 30 天调用量 | `GET /api/monitor/usage/model-usage?startTime=…&endTime=…` | `totalUsage.totalModelCallCount`、`totalUsage.totalTokensUsage`；时间格式 `YYYY-MM-DD HH:mm:ss`（本地时间，URL 编码）。每张卡片带迷你比例条（该时段 tokens 占近 30 天的比例，30 天卡恒为满格作标尺），副行附平均 tok/次 |
| 模型消耗折线图（今日 / 近 7 天 / 近 30 天） | `GET /api/monitor/credit-usage/usage-detail?usageType=MODEL&startTime=…&endTime=…`；失败时回退旧 `GET /api/monitor/usage/model-usage` | 时间横轴取 `data.x_time`（当日按小时 `HH:mm`、7/30 天按天 `MM-DD`，即官方「当日/7日/30日」口径）。序列自动发现：优先嵌套 `series[]`，其次与 `x_time` 等长的数值平行数组（如 `cachedTokens`/`uncachedTokens`/`outputTokens`/`modelCallCount`，未知键名以原始键名展示）。内联 SVG：网格线 + y 轴 1/2/5 取整、x 轴 ≤7 个刻度、点标记（≤40 桶）、单序列面积填充；图例可点击开关系列（至少保留一个），多序列时每项带合计；悬停显示竖向参考线 + 深色 tooltip 列出各序列值。该时段无时间序列时显示占位 |
| MCP 工具用量（环形图 + 条形图） | `GET /api/monitor/usage/tool-usage?…`（30 天） | 左侧为内联 SVG 环形图（中心显示 30 天 MCP 调用总数，扇区为各工具占比），右侧为带色块的条形行。数据取 `totalUsage` 的 `totalNetworkSearchCount`（联网搜索）/ `totalWebReadMcpCount`（网页读取）/ `totalZreadMcpCount`（Zread 读取），若接口新增其他 `*Count` 字段也会以原始键名展示；`quota/limit` 中 `TIME_LIMIT` 的 `usageDetails[]`（`modelCode` + `usage`）属另一统计口径，以灰色条形行追加（`xx（按工具）`），不并入环形图以免重复计数 |
| 原始接口数据 | 全部 | 页面底部折叠面板，展示各接口未经处理的完整 JSON，便于核对页面未使用的字段（如重置卡、积分等是否存在） |

> 注：若接口只返回一个 token 窗口（无周限制），则只显示 5 小时窗口一行；两者都没有时显示占位提示。

## 接口与鉴权

- 站点：`https://api.z.ai`（国际）/ `https://open.bigmodel.cn`（中国），路径相同，均已开启 CORS（浏览器可直连）。
- 认证头依次尝试 `Authorization: Bearer <key>` 和 `Authorization: <key>`，401 时自动换另一种重试。
- 注意：业务错误也返回 HTTP 200，需按响应体 `success` / `code` 判断，如 `{"code":401,"msg":"token expired or incorrect","success":false}`。

## 部署（GitHub Pages）

仓库自带 Action（`.github/workflows/deploy.yml`），推送到 `main` 即自动部署：

1. 在 GitHub 新建仓库，名称建议就用 `glm-quota`；
2. 推送：
   ```bash
   git remote add origin git@github.com:<你的用户名>/glm-quota.git
   git push -u origin main
   ```
3. 仓库 Settings → Pages → Build and deployment → Source 选 **GitHub Actions**（首次部署前需要设置一次）；
4. 访问 `https://<你的用户名>.github.io/glm-quota/`。

之后每次 push 到 `main` 自动重新部署，也可在 Actions 页手动触发（workflow_dispatch）。

## 测试

`test.html` 是本地联调页：通过 iframe 加载主页面并 mock `fetch`，无需真实 Key 即可验证渲染与错误路径。

```bash
test.html?mode=ok      # mock 成功数据（双窗口 + MCP 限额 + 模型明细）
test.html?mode=err401  # mock 401
test.html?mode=high    # mock 92.8% / 88.1% 高用量（红色进度条）
test.html?mode=fivonly # mock 只返回 5 小时窗口（周限制隐藏）
test.html?mode=real401 # 不 mock，用假 Key 打真实接口验证 CORS/401
```

## 安全

- **离线与防外传由 CSP 强制**：页面内置 `Content-Security-Policy`——`connect-src` 仅允许 `api.z.ai` 和 `open.bigmodel.cn` 两个官方域名，`default-src 'none'` 禁止加载任何外部脚本/样式/字体/图片/框架，`form-action 'none'` 禁止表单提交。即使代码被篡改，浏览器也会拦截向其他地址的任何请求。
- Key 只出现在 `Authorization` 请求头中，且只发往你在下拉框选择的官方域名；不进 URL、无统计脚本、无其他任何第三方请求（唯一的外部 URL 引用就是这两个接口常量，可全文搜索验证）。
- Key 默认只在页面内存中；勾选「记住 Key」才写入 localStorage（本机浏览器，不参与云同步）。
- **`file://` 打开时的注意点**：Chrome 把所有本地页面视为同一存储空间，勾选记住后 key 可被你本地打开的其他 HTML 读取。本地文件场景建议不勾选，或改用 `http://127.0.0.1` 访问（按源隔离）。
- 接口返回的错误信息、限额类型、模型名等字符串均经 HTML 转义后插入，防止响应污染注入脚本。
- **部署工作流零第三方**：`.github/workflows/deploy.yml` 只使用 GitHub 官方 `actions/*` 仓库的四个 action，且全部固定到 commit SHA（防 tag 篡改）；`upload-pages-artifact` 的产物只进入 GitHub Pages 自有存储，不存在向第三方上传的环节。

## 隐私

页面不收集任何数据；Key 只保存在本机 localStorage（需勾选「记住 Key」），请求由浏览器直连官方接口。
