---
name: index-eval-trading
description: 指数评价与交易体系私有 Skill：对单一指数（如中证红利、纳斯达克100、沪深300 等）执行"取数→指标体系→策略回测→交易方案"全链路。当用户要求拉取指数历史行情、建立估值/行情指标体系、回测估值择时策略、或生成"买入持有为核心 + 估值风控降仓"交易方案时使用。数据源为腾讯财经（行情）+ 东方财富妙想 MCP（估值交叉验证），遵循"LLM 读数、代码算数"原则：所有数字由确定性脚本计算。
---

# 指数评价与交易体系

## 定位
本 Skill 把指数投资研究的方法论固化为可复用脚本链，四步覆盖完整链路：

```
Step 1 取数   fetch_index_data.py  腾讯财经日频拉取 + 清洗标准化
Step 2 指标   compute_metrics.py   行情/趋势/风险/估值四层指标
Step 3 回测   run_backtest.py      买入持有/估值择时/阶梯仓位/趋势
Step 4 方案   generate_plan.py     生成交易方案文档
```

核心铁律：**LLM 只做解读、复盘、报告；数字与回测全部由脚本确定性地计算**，禁止用模型推算收益/回撤。

## 工作目录约定
- 每个指数一套工作目录，推荐结构：
  ```
  <workdir>/
  ├── raw/<key>_daily.csv            # Step1 原始数据
  ├── standardized/<key>_daily_std.csv  # Step1 标准化
  ├── metrics/<key>_pe_monthly.json  # 估值（外部获取后转录，格式见下）
  ├── metrics/<key>_dy_monthly.json  # 股息率（可选）
  ├── metrics/*_metrics_summary.csv  # Step2 指标
  ├── backtest/nav.csv perf.csv yearly.csv  # Step3 回测
  └── TRADING_PLAN.md                # Step4 方案
  ```

## Step 1 · 取数（腾讯财经）
```bash
python scripts/fetch_index_data.py --code sh000922 --name 中证红利 --ccy CNY \
    --start 2015-01-01 --end <最新> --out raw/csi_dividend_daily.csv
```
- 常用代码：`sh000922`=中证红利、`us.NDX`=纳斯达克100、`us.IXIC`=纳指综合、`us.DJI`=道琼斯、`sh000300`=沪深300、`sh000905`=中证500、`sh000016`=上证50。
- **注意**：`us.NDX` 必须带点（`usNDX` 返回 1 条不可用）；代码区分大小写。
- 脚本自动绕过环境代理（本地到腾讯接口可直连），分页上限 640 条/次自动翻页。
- 输出原始 CSV + 标准化 CSV（含 `pct_chg`、`nav` 基期=1、`index_name`、`ccy`）。

## Step 2 · 指标体系（需估值数据）
先用东方财富妙想 MCP 获取估值（`mx_index_block_finance_data` 查 PE/PB/股息率及月度历史），转录为 JSON 后计算：
```bash
python scripts/compute_metrics.py --csv standardized/<key>_daily_std.csv \
    --pe-json metrics/<key>_pe_monthly.json --out metrics/<key>_metrics_summary.csv
```
- 行情/趋势/风险指标从日频价格计算；估值指标（PE 当前值、全历史分位、E/P、ERP）需 `--pe-json`。
- 输出指标汇总 CSV + 估值快照 JSON（`*_snapshot.json`，供 Step4 用）。
- 估值 JSON 两种兼容格式（任选其一）：
  ```json
  {"months":["2026-09","2026-08"],"pe_ttm":[8.803,8.717],"latest_rf":1.684}
  {"data":[{"month":"2026-09","pe_ttm":8.803},...]}
  ```

## Step 3 · 回测
```bash
python scripts/run_backtest.py --csv standardized/<key>_daily_std.csv \
    --pe-json metrics/<key>_pe_monthly.json --dy-json metrics/<key>_dy_monthly.json \
    --start 2017-01-01 --out-dir backtest/
```
- 自动运行：基准(买入持有)、A(PE阈值6/9)、PLAN_阶梯(PE分位三档)、D(MA250趋势)。
- 口径：T+1 生效、现金收益 0、`--dy-json` 提供则含息近似（月股息率/100/250）。
- 阶梯阈值默认 60/80 分位、中间仓位 70%、底仓 30%，可用 `--th-lo/--th-hi/--w-mid/--w-low` 调整。
- 输出 nav.csv（净值）/ perf.csv（绩效矩阵）/ yearly.csv（分年度）。

## Step 4 · 交易方案
```bash
python scripts/generate_plan.py --index-name 中证红利 --code 000922.SH \
    --snapshot metrics/<key>_metrics_summary_snapshot.json \
    --backtest-perf backtest/perf.csv --out TRADING_PLAN.md
```
- 生成"买入持有为核心 + 估值风控降仓"方案，含当前信号仓位建议与回测绩效表。
- 方案要点（方法论默认）：常态 100% 持有；PE 分位 60-80% 降 70%、≥80% 降 30% 底仓（**永不空仓**，空仓会损失股息）；月度调仓、低频执行。

## 已验证的踩坑记录（勿重复尝试）
- iFinD MCP（`ifind-finance-data`）**不支持指数与美股**，仅 A 股个股；`000922` 会被误解析为"佳电股份"。
- yfinance→Yahoo 超时（curl 28）；东财 push2his 接口本地被代理拦截（RemoteDisconnected）；新浪 K 线接口返回 2019 旧数据。**均不可用**。
- 腾讯接口会间歇返回 **501（WAF 拦截）**：必须带完整浏览器头（Referer/Accept/UA，脚本已内置）并退避重试；单次请求失败不代表断源，重试即可。
- 东方财富妙想 MCP：`mx_index_block_finance_data` 查指数估值/行情；`mx_us_finance_data` 查美股；`mx_macro_data` 查 10 年期国债收益率（ERP 用）。调用前须按工具约束先 `tool_search` 取回 schema。

## 质量与可复现要求
- 每个数字标注来源与口径（价格/含息、T+1、现金0、无风险利率）。
- 回测必须避免未来函数：月末信号下月生效、PE 分位用 expanding 窗口。
- 交付含：脚本、数据、报告（指标/回测/方案三件套）、可视化（净值曲线、估值分位图）。
- 估值数据月度更新；行情可增量重拉。
