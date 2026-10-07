---
name: buff2steam
description: 分析网易 BUFF → Steam 市场的"现金换 Steam 钱包余额"搬砖收益。抓取 BUFF 最低在售价与 Steam 挂牌美元价，按扣 15% 手续费和实时汇率计算每件物品的收益率并排序。触发词：BUFF转steam、steam余额充值、饰品搬砖、buff steam差价。
---

# BUFF → Steam 余额搬砖分析

目标：给出"从网易 BUFF 用人民币买入饰品，在 Steam 市场卖出变钱包余额"的收益率排序表。

核心公式（全部按同一货币口径折算）：

```
实收(CNY) = Steam挂牌价(CNY) ÷ 1.15        # Steam 卖方实收 = 挂牌价/1.15（总费用约15%）
收益率    = 实收(CNY) / BUFF最低价(CNY) − 1
收益(CNY) = 实收(CNY) − BUFF最低价(CNY)
```

- 汇率默认 6.75，分析时先用 web 搜索确认当日中间价
- 这是"现金 → Steam 钱包"的折价充值，不是可提现套利，输出时必须提醒

## ⚠️ 头号陷阱：BUFF 的 steam_price 不可信（实测虚高 ~7%）

2026-10 实测：BUFF 接口返回的 `steam_price` / `steam_price_cny` 是其内部参考价，**系统性高于 Steam 实际挂牌**（15 个箱子交叉验证，比值 0.86~1.02、均值 ≈0.93）。例：Revolution Case BUFF 报 $0.24(¥1.61)，实际 Steam 最低挂牌 ¥1.46（$0.21）。直接用会把收益率虚高 5~15 个百分点（如 +27% 报成 +42%）。

**必须用真实 Steam 价**，来源 csgoskins.gg（可直连、与 Steam 盘口吻合）：
- 箱子 URL 规律：小写、空格→`-`，如 `https://csgoskins.gg/items/revolution-case`。用 WebFetch 抓，prompt 要求输出 `Steam=$X.XX, offers=N`（offers=在售件数，用于供给判断）。
- **皮肤/刀具 URL 规律不同，直接拼 slug 会 404**（`ak-47-redline-field-tested` 实测 404，站内搜索框跳转的同名 URL 也 404）。不要猜 URL；没拿到实价时用「BUFF steam_price × 0.93」估算并在表格里标注"估算"（±7% 波动足以翻转个位数收益率结论），下单前逐件人工核对 Steam 盘口。
- curl 直连 csgoskins.gg 被 Cloudflare 拦（返回"Just a small moment"挑战页），只能用 WebFetch；WebFetch 并发 >1 会报 user concurrency limit exceeded，**只能串行、每次 1~2 个**。
- Steam 盘口里买单价（buy order）才是"秒成交"价：实际操作是挂到买单价卖进深度。用户贴盘口（价格/数量/买单深度）时，直接用其数据重算，优先级最高。

## 数据获取（均已验证可用，无需登录 BUFF）

1. **物品名 → BUFF goods_id 映射**：抓取 csgo-trader 扩展的公开映射表（4 万余条，含全部箱子/皮肤/刀具/手套）：
   `https://raw.githubusercontent.com/gergelyszabo94/csgo-trader-extension/master/extension/src/utils/static/buffIds.json`
   返回 `{"AK-47 | Redline (Field-Tested)": 33960, ...}`，key 为英文 market_hash_name。
   ⚠️ 本机 curl 直连 raw.githubusercontent.com 会截断（curl exit 18/56，下载 357KB 而实际 2.2MB），下载后务必 `python -c "import json; json.load(...)"` 校验；改用 jsdelivr 镜像稳定：`https://cdn.jsdelivr.net/gh/gergelyszabo94/csgo-trader-extension@master/extension/src/utils/static/buffIds.json`

2. **单物品价格**（无需登录，需带 `X-Requested-With: XMLHttpRequest` 头，同源 cookie）：
   `https://buff.163.com/api/market/goods/sell_order?game=csgo&goods_id=<id>&page_num=1&sort_by=default&mode=&allow_tradable_cooldown=1`
   响应中：
   - `data.items[0].price` → BUFF 最低在售价（CNY 字符串）
   - `data.total_count` → 在售总量（供给紧张度）
     ⚠️ 实测该值在 91 处截断：充足供给的物品全部显示 91，无法区分 91 和 9 万。只有当 total_count < 91（如 34）时才是真实的小供给信号，可以据此标注 ⚠️；等于 91 时不要当作"只有 91 件在售"来解读。
   - `data.goods_infos["<id>"].steam_price` → ⚠️ BUFF 内部参考价，虚高 ~7%，不可直接当 Steam 挂牌价（见上）
   - `data.goods_infos["<id>"].steam_price_cny` → 同上，仅可作交叉验证参考

   注意：`/api/market/goods`（市场列表/搜索）需要登录，不要用它；sell_order 是免登录的。

3. **网络环境**：steamcommunity.com 直连会被墙，不要去抓 Steam 官方接口；公共 CORS 代理（allorigins/codetabs/r.jina.ai）在本机也基本不通。正确做法是在浏览器内（ZCode in-app browser，可直接打开 buff.163.com）用 `tab.playwright.evaluate(fetch...)` 调上述接口。
   - 请求频率限制：连续约 20 个请求后必被限流，返回 `访问频率过高` 或 HTML 错误页。实测 1.1s 间隔撑不过 24 个连发，2.5s 间隔稳定通过；**从第一个请求就按 2.5s 间隔，不要先快后慢再补救**。
   - 被限流后无需换接口，加大间隔重试即可恢复。
   - 单次 evaluate 有约 32 秒上限，每批按 2.5s 间隔只能跑 ~10 个物品，超过就分批（每批一次独立的 node_repl 调用）。
   - 每次浏览器 node_repl 调用都是新内核，必须重新跑 control-browser 的 bootstrap（setupBrowserRuntime），否则报 `agent is not defined`；tab 丢失时先 `browser.tabs.list()` 再 `tabs.get(id)`。

4. **汇率**：用 web 搜索"美元人民币汇率 今日"取中间价。

## 输出要求

按收益率降序的表格，列：物品名、BUFF 价(¥)、Steam 挂牌($)、扣费折算(¥)、收益率。并补充：

2. 收益率与单件绝对利润分开点评（低价箱比例高但单件赚得少）；**供给量同样是收益能否兑现的硬约束**——Steam 在售量极大（如 50 万件）说明挂单价会持续被压，只能卖进买单深度，点评里要点明
3. 在售量过小（<30）的物品标注 ⚠️，提示 BUFF 侧可能吃货困难或 Steam 侧有价无市
4. 表中每行标注数据置信度：Steam 价为实测（csgoskins.gg 或用户给的盘口）还是估算（×0.93），估算行的个位数收益率不可作为决策依据
5. 必须提醒：Steam 卖出所得为钱包余额不可提现；BUFF 买入后有 7~8 天交易冷却；汇率波动风险
6. 若用户要"只看某类"（如只要武器箱），候选清单可以直接给箱子名单：Recoil、Kilowatt、Dreams & Nightmares、Snakebite、Revolution、Horizon、CS20、Falchion、Fracture、Prisma、Prisma 2、Chroma 1-3、Gamma 1-2、Shattered Web、Clutch、Spectrum、Shadow、Operation Breakout、Glove、Danger Zone、Recoil 等
