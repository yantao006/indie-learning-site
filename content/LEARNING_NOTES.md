# How Poor Am I — 项目学习笔记

> 基于对 [howpoorami](https://github.com/yantao006/howpoorami) 项目的完整分析整理。
> 涵盖数据管道、统计算法、前端可视化技巧、以及与 AI Agent 协作的方法论。

---

## 目录

1. [项目全景](#1-项目全景)
2. [数据管道：从 API 到静态文件](#2-数据管道从-api-到静态文件)
3. [外部 API 详解](#3-外部-api-详解)
4. [数据处理：对齐与清洗](#4-数据处理对齐与清洗)
5. [核心算法：帕累托分布拟合百分位](#5-核心算法帕累托分布拟合百分位)
6. [收入→财富估算模型](#6-收入财富估算模型)
7. [前端架构：静态优先的数据流](#7-前端架构静态优先的数据流)
8. [可视化技巧](#8-可视化技巧)
9. [细节工程](#9-细节工程)
10. [与 AI Agent 协作：需求描述方法论](#10-与-ai-agent-协作需求描述方法论)

---

## 1. 项目全景

### 是什么

用户输入收入或净资产，计算其在全球及 30+ 个国家中的**财富百分位排名**，并通过多种图表展示财富不平等的结构。

### 技术栈

| 层 | 技术 |
|---|---|
| 框架 | Next.js 16 App Router，静态导出 |
| UI | React 19 + Tailwind CSS v4 |
| 图表 | visx（SVG 组合式）+ Framer Motion |
| 数据 | WID.world / OECD / World Bank / Forbes API |
| 部署 | Cloudflare Pages（纯静态） |
| 语言 | TypeScript 严格模式 |

### 核心设计原则

- **隐私优先**：所有计算在浏览器完成，用户数据不上传
- **静态优先**：数据在构建时处理，运行时无 API 调用
- **渐进增强**：无 JS 也能看到基本内容（SSR 初始值）

---

## 2. 数据管道：从 API 到静态文件

### 整体流程

```
外部 API (离线执行一次)
    ↓ pnpm data:fetch
scripts/fetch-all-data.mjs
    ↓ 写入
data/raw/*.json  ← 提交进 git，透明可审计
    ↓ Next.js 构建时 import
src/data/wealth-data.ts
    ↓ 模块加载时执行转换
COUNTRIES[], COUNTRY_MAP  ← 运行时直接使用的数据对象
```

### 数据来源

| 文件 | 来源 | 内容 |
|---|---|---|
| `wid-historical.json` | WID.world API | 各国 top1/top10/bottom50 财富份额，年度时序（最早至 1820 年） |
| `wealth-income-shares.json` | WID.world | 当前财富 & 收入分布份额 |
| `wid-detailed-shares.json` | WID.world | 细分百分位数据（top0.1%, top0.01%） |
| `country-metadata.json` | WID + World Bank | 人口、中位数/均值财富、基尼系数、货币 |
| `billionaires.json` | Forbes RTB API | 每国最富者 |
| `exchange-rates.json` | ECB Frankfurter API | 实时汇率 |
| `purchasing-power.json` | OECD / FRED | 工资、CPI、房价指数（2000 年为基准） |

### 关键设计：为什么把数据提交进 git

1. **可复现**：任何人 clone 后无需调 API 即可构建
2. **透明**：数据来源可追溯，利于发现错误
3. **稳定**：API 变更/限流不影响构建

---

## 3. 外部 API 详解

项目共使用 **5 个外部 API**，全部在 `scripts/fetch-all-data.mjs` 中调用，结果离线缓存为 `data/raw/*.json`。

---

### 3.1 WID.world API（核心数据源）

**用途**：财富/收入分布份额、历史时序、基尼系数、人均财富。覆盖 30+ 国家，数据追溯至 1820 年。

**Base URL**：
```
https://rfap9nitz6.execute-api.eu-west-1.amazonaws.com/prod
```

**认证**：Header `x-api-key`，使用 WID R 包内置的公共 key（开源项目共享，非保密）。

**请求格式**：
```
GET /countries-variables?countries=US,GB,FR&variables=shweal_p99p100_992_j,...
```

**变量命名规则**（关键，不看规则看不懂字段名）：

```
  shweal  _  p99p100  _  992  _  j
  ───┬───    ──┬────    ─┬─    ┤
     │         │         │     └─ j = 成年人（adult）
     │         │         └─ 992 = 净个人财富
     │         └─ p99p100 = 百分位区间（top 1%）
     └─ sh = share（份额），a = average（均值），t = threshold（门槛），g = gini
```

**项目使用的变量清单**：

| 变量名 | 含义 |
|---|---|
| `shweal_p99p100_992_j` | top 1% 财富份额 |
| `shweal_p90p100_992_j` | top 10% 财富份额 |
| `shweal_p0p50_992_j` | bottom 50% 财富份额 |
| `shweal_p99.9p100_992_j` | top 0.1% 财富份额 |
| `shweal_p99.99p100_992_j` | top 0.01% 财富份额 |
| `ahweal_p0p100_992_j` | 全体成年人均值财富（本币） |
| `thweal_p50p100_992_j` | 中位数财富门槛（本币） |
| `ghweal_p0p100_992_j` | 财富基尼系数 |
| `aptinc_p0p100_992_j` | 均值税前收入（本币） |
| `tptinc_p50p100_992_j` | 中位数税前收入门槛（本币） |
| `gptinc_p0p100_992_j` | 收入基尼系数 |
| `sptinc_p99p100_992_j` | top 1% 收入份额 |
| `sptinc_p90p100_992_j` | top 10% 收入份额 |
| `sptinc_p0p50_992_j` | bottom 50% 收入份额 |

**响应结构**（嵌套较深，需解析）：
```json
{
  "shweal_p99p100_992_j": [
    { "US": { "values": [{ "y": 2023, "v": 0.351 }, ...] } },
    { "GB": { "values": [{ "y": 2023, "v": 0.213 }, ...] } }
  ]
}
```

**解析逻辑**：
```js
function parseWIDResponse(data) {
  const result = {};
  for (const [varKey, countryBlocks] of Object.entries(data)) {
    for (const block of countryBlocks) {
      for (const [cc, cdata] of Object.entries(block)) {
        if (!result[cc]) result[cc] = {};
        for (const { y, v } of cdata.values || []) {
          result[cc][varKey][y] = v; // { "US": { "shweal_p99p100_992_j": { "2023": 0.351 } } }
        }
      }
    }
  }
}
```

**限流处理**：每批次 10 个国家，批次间等待 500ms：
```js
for (let i = 0; i < countries.length; i += 10) {
  const batch = countries.slice(i, i + 10);
  await fetchWID(batch, vars);
  if (i + 10 < countries.length) await new Promise(r => setTimeout(r, 500));
}
```

**特殊国家代码**：全球汇总用 `WO-MER`（world at market exchange rates），WID 特有，ISO 标准没有。

**输出文件**：
- `wid-historical.json` — 历史时序（全部年份）
- `wid-sub-percentile.json` — 细分百分位原始数据
- `wid-detailed-shares.json` — 从细分数据派生的 6 组份额（摘取最新年份）
- `wealth-income-shares.json` — 财富与收入份额（当前值）
- `country-metadata.json`（部分）— 均值/中位数/基尼系数

---

### 3.2 Forbes 富豪榜 API（via komed3/rtb-api）

**用途**：获取每个国家财富最多的人及其净资产，用于"你需要多少年才能赚到 XXX 的财富"对比功能。

**来源**：GitHub 上的开源镜像项目 [komed3/rtb-api](https://github.com/komed3/rtb-api)（MIT 协议），数据来自 Forbes Real-Time Billionaires。

**Base URL**：
```
https://cdn.statically.io/gh/komed3/rtb-api/main/api
```

**两步获取**：

**第一步**：拉取国家统计列表，找出每国最富者的 slug：
```
GET /stats/country/_list
```
响应为纯文本，每行格式：
```
us  <count>  <total_NW>  <top_slug>  <rank>  <top_NW_in_billions>
```

**第二步**：用 slug 拉取个人资料：
```
GET /person/{slug}/profile
```
响应 JSON：`{ name, source, netWorth, ... }`

**货币单位注意**：列表中净资产单位为**十亿美元**，代码中乘以 `1_000_000` 转为美元（实际是的确乘了百万，即单位是百万美元，然后乘以 1000000 得到美元）：
```js
netWorth: Math.round(stats.topNetWorth * 1_000_000)
// topNetWorth 单位是 billion USD → 乘以 1,000,000 → USD（实为 × 10^6，billion = 10^9，所以结果是 × 10^3 = thousands USD
// 实际代码：topNetWorth(亿美元) × 1_000_000 = 美元值（亿 = 10^8 → ×10^6 = 百万位...需看实际数量级）
```

**输出文件**：`billionaires.json`

---

### 3.3 World Bank API

**用途**：人口数据、CPI（消费者物价指数）、智利比索汇率备用。免费，无需认证。

**Base URL**：`https://api.worldbank.org/v2`

**三个 indicator 用途**：

| Indicator | 含义 | 用于 |
|---|---|---|
| `SP.POP.TOTL` | 总人口 | 计算各组实际人数（亿 × 份额%） |
| `FP.CPI.TOTL` | CPI 指数（2010=100） | 购买力图表，通货膨胀对比 |
| `PA.NUS.FCRF` | 官方汇率（本币/美元） | CLP（智利比索）备用汇率 |

**请求示例**：
```
# 多国人口（用分号分隔国家代码）
GET /country/US;GB;FR;DE/indicator/SP.POP.TOTL?date=2023:2024&format=json&per_page=500

# 全球人口（WLD 为全球代码）
GET /country/WLD/indicator/SP.POP.TOTL?date=2023:2024&format=json&per_page=5
```

**响应结构**：返回数组，`[0]` 为分页元信息，`[1]` 为数据数组：
```json
[
  { "page": 1, "pages": 1, "total": 4 },
  [
    { "country": { "id": "US" }, "date": "2023", "value": 334914895 },
    ...
  ]
]
```

**注意事项**：
- 不要用 `r.country.id` 判断国家——有时 World Bank 会返回区域代码（如 `"XU"` 代表欧盟）。用 `r.countryiso2code` 更可靠。
- `per_page=500` 防止分页导致数据截断。

**输出文件**：`country-metadata.json`（人口字段）、`purchasing-power.json`（CPI 序列）

---

### 3.4 OECD SDMX API

**用途**：获取各国年均工资数据（USD，固定价格 + PPP），用于"工资是否跟上生活成本"图表。

**格式**：SDMX 2.1 REST，返回特殊的 JSON 格式（`application/vnd.sdmx.data+json`）。

**请求示例**：
```
GET https://sdmx.oecd.org/public/rest/data/OECD.ELS.SAE,DSD_EARNINGS@AV_AN_WAGE,1.0/USA.A.USD_PPP_CST..
    ?startPeriod=1990&endPeriod=2024&dimensionAtObservation=AllDimensions
Headers: Accept: application/vnd.sdmx.data+json;version=2.0.0
```

URL 各段含义：
```
OECD.ELS.SAE          — 提供方.部门.数据集
DSD_EARNINGS@AV_AN_WAGE,1.0  — 数据集ID@指标,版本
/ USA.A.USD_PPP_CST.. — 国家.频率.货币单位..
```

**响应解析（SDMX JSON 格式较特殊）**：
```js
// 观测值的 key 是多个维度索引用":"连接的字符串，如 "0:0:1:0:5"
// 最后一个索引对应时间维度
const obs = data.data.dataSets[0].observations;
const timeDim = data.data.structures[0].dimensions.observation
  .find(d => d.id === "TIME_PERIOD");
const timeValues = timeDim.values; // [{ id: "1990" }, { id: "1991" }, ...]

for (const [key, val] of Object.entries(obs)) {
  const timeIdx = parseInt(key.split(":").at(-1));
  const year = timeValues[timeIdx].id;
  result[cc][year] = val[0]; // val 是数组，取第一个元素
}
```

**国家代码**：OECD 用 ISO 3166-1 alpha-3（三字母）：`USA`、`GBR`、`FRA`、`DEU`、`NLD`。

**输出文件**：`purchasing-power.json`（wages 序列）

---

### 3.5 Frankfurter API（ECB 汇率）

**用途**：获取所有货币相对 USD 的汇率，用于在货币之间换算财富值。数据来自欧洲中央银行（ECB）。

**无需认证，完全免费**。

**请求示例**：
```
GET https://api.frankfurter.app/latest?from=USD&to=GBP,EUR,JPY,CNY,...
```

**响应**：
```json
{
  "amount": 1,
  "base": "USD",
  "date": "2024-01-15",
  "rates": { "GBP": 0.789, "EUR": 0.921, "JPY": 148.5, ... }
}
```

**限制**：不支持 `CLP`（智利比索），该货币回退到 World Bank `PA.NUS.FCRF`：
```js
const frankfurterCurrencies = currencies.filter(c => c !== "CLP");
// CLP 单独从 World Bank 获取
```

**汇率使用方式**：所有货币计算统一以 USD 为中间值：
```js
// 本币 → USD
function toUSD(value, currency) {
  return value / exchangeRates[currency]; // rate = 本币/美元
}
// USD → 本币
function fromUSD(usdValue, currency) {
  return usdValue * exchangeRates[currency];
}
```

**输出文件**：`exchange-rates.json`

---

### 3.6 FRED API（圣路易斯联储）

**用途**：获取各国实际房价指数（BIS 数据，已 CPI 平减），用于"房价是否远超工资"图表。

**需要注册免费 API Key**：`https://fred.stlouisfed.org/`，设置环境变量 `FRED_API_KEY`。未设置时该数据跳过。

**请求示例**：
```
GET https://api.stlouisfed.org/fred/series/observations
    ?series_id=QUSR628BIS&api_key={KEY}&file_type=json&observation_start=1990-01-01
```

**各国 Series ID**（BIS 房价数据）：

| 国家 | Series ID |
|---|---|
| 美国 | `QUSR628BIS` |
| 英国 | `QGBR628BIS` |
| 法国 | `QFRR628BIS` |
| 德国 | `QDER628BIS` |
| 荷兰 | `QNLR628BIS` |

**注意**：数据为季度频率，代码只取每年 Q1（第一条）：
```js
if (!(year in result[cc])) { // 只保留每年第一次出现的值
  result[cc][year] = parseFloat(obs.value);
}
```

**输出文件**：`purchasing-power.json`（housePriceIndex 序列）

---

### 3.7 数据指数化处理（共用）

OECD 工资、World Bank CPI、FRED 房价，三条序列的绝对数值单位不同，无法直接对比。统一做**2000 年基准化（rebase）**：

```js
function rebaseIndex(yearValueMap, baseYear = 2000) {
  const baseVal = yearValueMap[String(baseYear)];
  const result = {};
  for (const [y, v] of Object.entries(yearValueMap)) {
    result[y] = Math.round((v / baseVal) * 1000) / 10; // 保留1位小数
  }
  return result;
  // 2000年 = 100, 2023年 = 156.3 表示涨了 56.3%
}
```

最终 `purchasing-power.json` 存储的是**指数值**，不是原始货币/价格数字。

---

### API 汇总表

| API | 认证 | 限流 | 覆盖国家 | 输出文件 |
|---|---|---|---|---|
| WID.world | x-api-key（公共key） | 需分批，间隔500ms | 30+ | wid-historical, shares, metadata |
| Forbes/komed3 | 无 | 间隔100ms | 30+ | billionaires.json |
| World Bank | 无 | 无明确限制 | 全球 | country-metadata, purchasing-power |
| OECD SDMX | 无 | 无明确限制 | 5个核心国 | purchasing-power |
| Frankfurter | 无 | 无明确限制 | 主要货币 | exchange-rates.json |
| FRED | API Key（免费） | 无明确限制 | 5个核心国 | purchasing-power |

---

## 4. 数据处理：对齐与清洗

### 问题背景

WID.world 三条历史系列（top1、top10、bottom50）**各自独立采集，有数据的年份不完全相同**：

```
top1    数据: 1820...1907, 1909...2024  (125年)
top10   数据: 1820...1907, 1909...2024  (125年)
bottom50数据: 1820...1906, 1908...2024  (123年，缺 1907/1909)
```

历史折线图需要三条线在**同一 X 轴年份点**都有值，否则折线断裂或长度不一致。

### 解决方案：两步处理

#### 第一步：selectPoints — 从目标年份中捞数据

不使用全部年份，只取关键历史节点：

```ts
const TARGET_YEARS = [
  1900, 1910, 1913, 1920, 1929, 1930, 1940, 1945,
  1950, 1960, 1970, 1980, 1985, 1990, 1995, 2000,
  2005, 2008, 2010, 2015, 2020, 2023
];
```

对每个目标年，按优先级查找：精确匹配 → 偏移 +1 → -1 → +2 → -2。

```ts
// 例如目标 1945，原始数据没有，找到 1946 代替
for (const delta of [1, -1, 2, -2]) {
  if (yearValueMap[String(target + delta)] !== undefined) {
    selected.set(target + delta, ...);
    break;
  }
}
```

同时强制追加**最新可用年份**，确保图表包含最新数据。

#### 第二步：alignSeries — 取三者交集

```ts
const commonYears = new Set(
  [...top10Years].filter(y => top1Years.has(y) && bottom50Years.has(y))
);
// 三条系列各自只保留 commonYears 里的点
```

**结果**：三条折线年份完全一致，可以共享同一 X 轴坐标系。

### 学习要点

> 时序数据对齐的核心思路：不强行插值补齐缺失点，而是**取共同有数据的时间点**，保证数据真实性。

---

## 5. 核心算法：帕累托分布拟合百分位

### 问题

WID 数据只给了几个"份额"数字（top1% 占多少，top10% 占多少），以及均值/中位数。
用户输入一个具体财富值，需要知道对应的百分位——但没有完整的分布曲线。

### 解法：用两个份额数推导整条尾部分布

#### 第一步：推导帕累托指数 α

财富尾部服从幂律分布（帕累托分布），两个份额的比值满足：

```
share(top1%) / share(top10%) = (1% / 10%)^(1 - 1/α)
```

用美国数据（top1=35%, top10=70%）代入：

```
ratio = 35% / 70% = 0.5
0.5 = 0.1^(1 - 1/α)
ln(0.5) = (1 - 1/α) × ln(0.1)

求解：α ≈ 1.43
```

```ts
function estimateParetoAlpha(top1Share: number, top10Share: number): number {
  const ratio = top1Share / top10Share;
  const exponent = Math.log(ratio) / Math.log(0.1);
  const alpha = 1 / (1 - exponent);
  return Math.max(1.1, Math.min(3.0, alpha)); // 限制在合理范围
}
```

**α 的含义**：α 越小 → 尾部越肥 → 财富越集中。美国 1.43 极度集中，北欧国家约 1.8~2.2。

#### 第二步：从 α 推导各门槛财富值

帕累托分布性质：某组人的平均财富 = 最低入门门槛 × α/(α-1)

```
bCoeff = α / (α-1)

top10% 平均财富 = mean × top10Share / 10%
p90 门槛 = top10平均财富 / bCoeff
```

美国示例：
```
bCoeff = 1.43 / 0.43 ≈ 3.33
top10均值 = 107,000 × 70% / 10% = $749,000
p90门槛  ≈ $749,000 / 3.33 ≈ $225,000
p99门槛  ≈ $3,745,000 / 3.33 ≈ $1,124,000
```

#### 第三步：建锚点表，分段线性插值

```ts
const anchors: [wealth, percentile][] = [
  [negFloor,  0    ],  // 负财富下限 → 0%
  [0,         negPct], // $0 → 负财富人群占比（美国约12.5%）
  [p50,       50   ],  // 中位数（直接来自WID，最准确）
  [p90,       90   ],  // 帕累托估算
  [p99,       99   ],
  [p999,      99.9 ],
  [p9999,     99.99],
];
```

用户财富 $500,000（美国）：落在 p90~p99 区间，线性插值：

```
fraction = (500,000 - 225,000) / (1,124,000 - 225,000) = 0.306
百分位 = 90 + 0.306 × 9 ≈ 92.75%
```

### 负财富处理

```ts
function estimateNegativeFraction(bottom50Share: number): number {
  if (bottom50Share < -5) return 0.25; // 25%的人净负债
  if (bottom50Share < 0)  return 0.18;
  if (bottom50Share < 3)  return 0.12;
  if (bottom50Share < 8)  return 0.08;
  return 0.05;
}
```

bottom50 份额越低（甚至为负），说明净负债人群越多。

### 学习要点

> - 帕累托拟合：**两个数字重建整条尾部分布**，这是处理有限统计数据的经典手段
> - p50（中位数）直接来自实测，不用估算，是整个分布的"锚点"
> - 负财富人群单独建模，不能用正态假设

---

## 6. 收入→财富估算模型

### 核心思路

年收入不直接等于财富，需要乘以**财富倍数**，倍数由多个人口统计因子共同决定：

```
估算财富 = 年收入 × 年龄系数 × 储蓄系数 × 负债系数 × 教育系数 × 就业系数 × 婚姻系数
         + 房产净值 + 投资账户 + 退休账户
```

### 各因子系数（来源：美联储 SCF 2022）

| 因子 | 典型区间 |
|---|---|
| 年龄（25岁 → 65岁） | 0.6 → 6.5 |
| 储蓄率（极低 → 极高） | 0.4 → 1.8 |
| 负债水平（低 → 极高） | 0.85 → 0.15 |
| 教育（无学历 → 博士） | 0.6 → 1.3 |
| 就业（失业 → 企业主） | 0.5 → 1.6 |
| 婚姻（离婚 → 已婚） | 0.7 → 1.15 |

### 不确定性区间

用户填写的因子越多，估算范围越窄：

```ts
// 递减收益曲线：前几个高影响因子比后面的更有价值
const progress = 1 - Math.pow(1 - filled / MAX_FACTORS, 1.5);
spread = 70% - (70% - 10%) × progress
// 0个因子: ±70%;  全部13个因子: ±10%
```

最终输出 `{ low, mid, high }` 三个财富值 → 对应三个百分位，展示为范围而非单点。

### 因子影响分析

```ts
// 分析每个因子对估算的方向性影响
computeFactorImpacts(factors) → FactorImpact[]
// 返回: [{ key:"age", direction:"up", reason:"Age 45: 中年积累期" }, ...]
// 用于在 UI 上展示"哪些因素在推高/压低你的财富估算"
```

### 学习要点

> - 不要给单点估算，给**范围**。范围随信息增加而收窄，更诚实也更有用
> - 递减收益曲线：第一个因子的信息增益远大于第十个，用 `Math.pow` 建模
> - 直接加总（房产净值）优于用系数换算，前者数据确定性更高

---

## 7. 前端架构：静态优先的数据流

### 数据流向

```
data/raw/*.json  (git 中的静态文件)
    ↓ TypeScript import（构建时）
src/data/wealth-data.ts  → 模块顶层执行转换，生成 COUNTRIES[]
    ↓
HomeClient.tsx  (单一 state 入口)
    ├── selectedCountry → ALL_COUNTRY_MAP[code]  获取国家对象
    ├── userPercentile  → 从 WealthInput 冒泡上来
    └── 传给所有图表组件 (country, userPercentile)
```

### 两个关键状态

```ts
const [selectedCountry, setSelectedCountry] = useState<AllCountryCode>("US");
const [userPercentile, setUserPercentile] = useState<number | null>(null);
```

- `selectedCountry` 驱动所有图表重新渲染（数据联动）
- `userPercentile` 只影响"用户位置标记"（高亮当前所在分组）

### 切换国家时的货币转换

```ts
// 不清空输入，而是把值按汇率换算到新货币
const newValue = fromUSD(toUSD(parsedValue, prevCurrency), country.currency);
setInputValue(String(Math.round(newValue)));
```

**设计决策**：保留用户输入的"感觉"（数字大小），而不是保留精确的 USD 等值。

### 地理位置检测

```ts
// 优先用时区（无需权限），fallback 用语言
function detectFromTimezone(): AllCountryCode | null {
  const tz = Intl.DateTimeFormat().resolvedOptions().timeZone;
  return TIMEZONE_TO_COUNTRY[tz] ?? null;
}
```

不请求 Geolocation API（需要用户授权），用 `Intl.DateTimeFormat` 获取时区，静默完成。

### 学习要点

> - 全局状态尽量少：只维护"输入"，派生值在用时计算，不进 state
> - 隐私敏感的检测用无权限的 API（时区 > 语言 > Geolocation）
> - 切换上下文时，保留用户已有输入（转换），而不是重置

---

## 8. 可视化技巧

### 7.1 响应式 SVG 容器：ResizeObserver 模式

所有 SVG 图表通过 render prop 注入尺寸，不自己处理响应式：

```tsx
// 使用方
<ResponsiveChart aspectRatio={16/9} minHeight={350} maxHeight={500}>
  {({ width, height }) => <MyChart width={width} height={height} />}
</ResponsiveChart>

// 内部：ResizeObserver 监听容器变化
new ResizeObserver(() => {
  const width = container.clientWidth;
  const height = clamp(width / aspectRatio, minHeight, maxHeight);
  setDimensions({ width, height });
});
```

**要点**：不用 CSS transform 缩放 SVG，而是重新计算所有坐标。宽高变化 → 重新 render → visx scale 自动映射新坐标。

---

### 7.2 双条对比布局：视觉不对称传达信息

```
Wealth ████████████████████████  35.1%   ← h-5, opacity 100%
People ███                         1%    ← h-3, opacity 40%
```

同一颜色，不同粗细和透明度：**粗细差本身就是在说财富与人口的不对称**。

```tsx
{/* 财富条：粗 */}
<div className="flex-1 h-5 rounded-full bg-white/5 overflow-hidden">
  <motion.div style={{ backgroundColor: seg.color }}
    animate={{ width: `${wealthBarPct}%` }} />
</div>
{/* 人口条：细+暗 */}
<div className="flex-1 h-3 rounded-full bg-white/5 overflow-hidden">
  <motion.div style={{ backgroundColor: seg.color }}
    className="opacity-40"
    animate={{ width: `${popBarPct}%` }} />
</div>
```

---

### 7.3 数据钻取缩放：过滤 + 重归一化

Zoom 不是缩放 SVG，而是换一组数据并重新归一化坐标：

```ts
// overview → 6个组
// top10    → 去掉 bottom50 和 middle40，只剩4组
// top1     → 只剩高净值的3组

const maxWealth = Math.max(...segments.map(s => s.wealthShare)); // 重新计算最大值
const wealthBarPct = (seg.wealthShare / maxWealth) * 100; // 归一化
```

Top 0.01% 在 overview 里宽度极窄，zoom 进 top1 后铺满全宽，不等比但信息量更大。

---

### 7.4 面积编码财富：等高矩形图

`WealthHoardingChart` 用矩形面积（而非颜色深浅或高度）编码财富份额：

```ts
// 宽度 = 财富份额，高度固定
// → 面积直接正比于财富占有量
const rectWidth = (wealthShare / totalShare) * innerWidth;
const rectHeight = innerHeight; // 所有矩形等高
```

**标签自适应显示**：矩形过窄时不显示文字，图表外独立渲染 legend：

```ts
const shouldShowLabel   = (rect: RectData) => rect.w >= 40;
const shouldShowPercent = (rect: RectData) => rect.w >= 60;
```

---

### 7.5 Tooltip 防溢出翻转

4 行代码，处理所有图表边缘碰撞：

```ts
const flipX = left + offset + tooltipWidth > containerWidth;
const x = flipX ? left - offset - tooltipWidth : left + offset;

const flipY = top + tooltipHeight > containerHeight;
const y = flipY ? Math.max(4, top - tooltipHeight) : top;
```

加上 `pointerEvents: none` 确保 tooltip 不拦截鼠标事件。这是**所有 SVG 图表 tooltip 的通用解法**。

---

### 7.6 数字滚动进场：IntersectionObserver + rAF

不用 `setInterval`，用两个浏览器原生 API 组合：

```ts
// 1. 检测元素进入视口（只触发一次）
new IntersectionObserver(([entry]) => {
  if (!entry.isIntersecting || hasAnimated.current) return;
  hasAnimated.current = true;

  // 2. 用 rAF 逐帧更新，三次方缓出
  function tick(now: number) {
    const progress = Math.min((now - startTime) / durationMs, 1);
    const eased = 1 - Math.pow(1 - progress, 3); // 先快后慢
    setCount(eased * end);
    if (progress < 1) requestAnimationFrame(tick);
  }
  setCount(0);
  requestAnimationFrame(tick);
}, { threshold: 0.1 });
```

SSR 防闪烁：`useState(end)` 初始值用真实数字，服务端 HTML 正确，客户端水合后再触发动画。

---

### 7.7 历史图事件标注层

折线图分层叠加，事件标注与数据共享同一 `xScale`：

```ts
// 数据层：AreaClosed + LinePath
// 标注层：垂直虚线 + 旋转文字
const events = COUNTRY_EVENTS[country.code] ?? DEFAULT_EVENTS;
events.map(({ year, label }) => {
  const xPos = xScale(year); // 与折线数据用同一比例尺
  return (
    <line x1={xPos} x2={xPos} y1={0} y2={innerHeight} strokeDasharray="4,3" />
    <text x={xPos} transform="rotate(-45)" ...>{label}</text>
  );
});
```

---

### 7.8 色彩系统：Paul Tol 色板

所有图表统一使用同一套颜色常量，跨组件编码一致：

```ts
const COLORS = {
  bottom50: "#44AA99", // 青绿
  middle40: "#88CCEE", // 浅蓝
  next9:    "#DDCC77", // 黄
  next09:   "#CC6677", // 玫红（警示色）
  next009:  "#AA4499", // 紫
  top001:   "#882255", // 深紫红（最高集中度）
};
```

**Paul Tol 色板**专为色盲友好设计，12 种颜色在 8 种色盲类型下均可区分。颜色语义化：越红越"集中/警示"。

---

## 9. 细节工程

### 8.1 `tabular-nums`：数字对齐

```html
<span class="tabular-nums">1,234,567</span>
```

等宽数字字体，防止数字更新时文字抖动。所有展示金额、百分比的地方都加。

### 8.2 `useMemo` 细粒度依赖

```ts
// 不用 country 对象作为依赖（引用变化会重算）
// 用具体字段
const shares = useMemo(
  () => getDetailedShares(country),
  [country.code, country.wealthShares.bottom50, country.wealthShares.top1]
);
```

避免父组件渲染导致不必要的图表重计算。

### 8.3 错误边界隔离

```tsx
<ErrorBoundary>
  <ResponsiveChart>
    {({ width, height }) => <ComplexChart ... />}
  </ResponsiveChart>
</ErrorBoundary>
```

每个图表独立包裹 ErrorBoundary，单个图表崩溃不影响整页。

### 8.4 动画按需加载

```tsx
// 不全量导入 framer-motion
import { LazyMotion, domAnimation, m } from "framer-motion";

<LazyMotion features={domAnimation}>
  <m.div animate={...} />
</LazyMotion>
```

`domAnimation` 只包含基础动画特性，比完整包小 ~30%。

### 8.5 货币计算统一走 USD

```ts
// 所有百分位计算在 USD 进行
const usdValue = toUSD(inputValue, country.currency);
const percentile = findPercentile(usdValue, country);

// 展示时再转回本地货币
const display = fromUSD(thresholdUSD, country.currency);
```

避免汇率污染算法逻辑，跨国比较时也不需要额外处理。

---

## 10. 与 AI Agent 协作：需求描述方法论

### 为什么直接描述功能行不通

对 Agent 说"做一个财富不平等网站，用户输入收入显示百分位"——它会给你能跑的玩具，但算法精度差、数据假、图表没交互。

**根本原因：需求里省略了所有"为什么这么设计"的隐含决策。**

---

### 分层描述法：五层逐步收敛

#### 第一层：数据契约（最先确定）

把数据的形状钉死，不让 Agent 猜：

```
需要 CountryData 类型：
- medianWealthPerAdult, meanWealthPerAdult (USD, 来自 WID)
- wealthShares: { top1, top10, middle40, bottom50 } (百分比)
- historicalWealthTop1: { year, share }[]，要求三条系列对齐到相同年份

数据获取方式：调 WID.world API，字段名格式为 shweal_p99p100_992_j
结果存为静态 JSON，提交进 git，不要运行时调 API
```

**为什么最先做**：算法和前端都依赖数据契约，先定型就不会返工。

#### 第二层：算法，附推导过程

```
已知: top1份额、top10份额、均值、中位数
目标: 财富值 → 百分位

推导:
1. ratio = top1/top10 = (0.1)^(1-1/α)，解出 α
2. bCoeff = α/(α-1)，门槛 = 该组均值 / bCoeff
3. 建7个锚点（含负值区域），线性插值

注意：bottom50份额可能为负，负财富人群需单独估算比例
```

给了推导，Agent 的实现就可验证。

#### 第三层：视觉映射规则

```
财富分布图，每组一行：
- 第一条: 宽度=wealthShare/max(wealthShares), 高度h-5（财富条）
- 第二条: 宽度=population/max(population), 高度h-3, opacity 40%（人口条）
- 相同颜色，差异只在粗细和透明度

三个缩放按钮(All/Top10%/Top1%)：
切换时过滤展示的组，并重新归一化两条进度条的最大值（不写死100%）
```

把"重新归一化"这个决策明确说出来。

#### 第四层：状态流

```
全局状态只有两个：selectedCountry / userPercentile

selectedCountry 变化时：
- 所有图表联动更新
- WealthInput 将已输入值按汇率转换，不清空

userPercentile 变化时（用户输入后计算）：
- WealthDistributionChart 高亮用户所在分组
- 其他图表不变
```

#### 第五层：技术约束与原因

```
- 静态导出，无服务器，数据在构建时 import
- SVG 图表用 visx（不用 recharts），因为需要自定义叠加事件标注层
- 响应式：用 ResizeObserver 监听容器宽度，重新计算坐标，不用 CSS transform 缩放
```

---

### 最容易被忽略的隐含决策

| 如果不说 | Agent 会做什么错误决策 |
|---|---|
| 负财富人群存在 | 百分位从 $0 计算，负资产全部归 0% |
| 切换国家时转换货币 | 清空输入框，体验断裂 |
| 重归一化坐标轴 | top0.01% 永远是细线，zoom 后无变化 |
| tooltip 防溢出翻转 | tooltip 被图表边缘裁剪 |
| 颜色跨组件统一 | 同数据组在不同图表用不同颜色，认知负担 |
| 历史数据需要对齐年份 | 三条折线 X 轴不一致，图表断裂 |
| 数字显示用 tabular-nums | 数字更新时文字抖动 |

---

### 实际协作节奏

```
你                              Agent
────────────────────────────────────────────
给数据结构契约           →    写 fetch 脚本 + TS 类型
确认类型                 →    写核心算法
给测试用例               →    验证/修正算法
给视觉映射规则           →    实现单个图表组件
浏览器看效果             →    调整
给状态流说明             →    组装页面，连接 state
给交互细节               →    完善 tooltip/zoom/动画
```

**每轮只聚焦一层**，不要混合"帮我写算法和做图表和处理响应式"——Agent 会每层都做浅。

---

### 一句话原则

> **把你脑子里"当然应该这样"的隐含决策，全部显式说出来。Agent 没有"当然"，只有你说了的。**

---

*整理自 howpoorami 项目代码分析 — 2026/04/18*
