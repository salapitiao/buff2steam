# buff2steam

一个用来分析「网易 BUFF → Steam 余额」搬砖收益的 AI 技能（Skill）。

G胖的钱包余额从哪来最划算？答案是：从 BUFF 用现金买皮肤，挂到 Steam 市场卖掉，7 折多换成一堆余额。这个 skill 让你的 AI 助手自动算清楚每一件饰品到底能"折"出多少。

## 它能做什么

- 抓取 BUFF 每件饰品的**最低在售价**（免登录，不需要 BUFF 账号）
- 对照 **Steam 市场挂牌美元价**，扣掉 15% 卖方手续费
- 按当日美元汇率折算成人民币，算出每件物品的**收益率**并排序
- 自动标注供给紧张的物品，提示吃货困难 / Steam 侧有价无市的风险

跑一次大概长这样：

| 物品 | BUFF 价(¥) | Steam($) | 扣费折算(¥) | 收益率 |
|---|---|---|---|---|
| Recoil Case | 1.50 | 0.37 | 2.12 | +41% |
| Clutch Case | 2.85 | 0.70 | 4.01 | +41% |
| AWP Asiimov (FT) | 704.50 | 155.54 | 890.44 | +26% |

箱子类收益率普遍 +30%~40%，中端皮肤 +10%~25%；而信仰刀和高端手套反而是倒挂的（Steam 溢价太高，买不如直接买余额）。

## 安装

把 `SKILL.md` 放进你的 AI 助手的技能目录即可，例如 Claude Code：

```bash
git clone https://github.com/salapitiao/buff2steam.git
mkdir -p ~/.claude/skills
cp buff2steam/SKILL.md ~/.claude/skills/buff2steam/SKILL.md
```

然后对助手说「帮我分析一下 BUFF 转 steam 的搬砖收益」就会触发。

## 怎么用

1. AI 会先确认当日美元中间价
2. 拉 csgo-trader 扩展的 4 万条物品 ID 映射表，解析你要分析的物品
3. 在浏览器内免登录调 BUFF 的公开接口查实时价
4. 输出按收益率排序的表格 + 风险提示

## PS（必读）

- **Steam 卖出所得是钱包余额，不可提现**。这是"7 折现金换余额"的充值通道，不是套利，想清楚了再上车。
- BUFF 买入后有 7~8 天交易冷却，期间价格可能波动。
- Steam 挂牌价 ≠ 成交价，高价挂单可能长期无人接盘。

## License

MIT
