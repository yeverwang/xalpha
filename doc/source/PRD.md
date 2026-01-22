# xalpha 产品需求文档 (PRD)

## 文档信息

| 项目 | 内容 |
|------|------|
| 产品名称 | xalpha |
| 版本号 | 0.12.2 |
| 作者 | refraction-ray |
| 文档版本 | 1.0 |
| 创建日期 | 2026-01-22 |
| 许可证 | MIT License |

---

## 1. 产品概述

### 1.1 产品定位

xalpha 是一款面向中国投资者的**基金投资全流程管理工具**，专注于场外基金的信息获取、投资账户管理、策略回测与可视化分析。产品以 Python 库的形式提供，适合定投型和网格型投资的概览与管理分析。

### 1.2 目标用户

| 用户类型 | 特征描述 | 核心需求 |
|---------|---------|---------|
| 个人投资者 | 有基金定投习惯，希望精确管理投资账户 | 交易记录管理、收益分析、持仓透视 |
| 量化研究员 | 使用 Python 进行量化分析 | 数据获取、策略回测、估值分析 |
| 金融开发者 | 开发金融应用系统 | API 调用、数据源集成、自动化交易提醒 |

### 1.3 核心价值主张

1. **一站式数据获取**：统一接口获取几乎所有市场产品的价格数据
2. **精确到分的账户管理**：完美模拟实盘交易，支持分红、折算等复杂场景
3. **丰富的可视化支持**：配合 Jupyter Notebook 提供直观的图表展示
4. **灵活的策略回测**：支持动态回测框架，验证投资策略有效性
5. **底层持仓穿透**：透视基金组合的股票细节和行业配置

---

## 2. 功能需求

### 2.1 功能架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           xalpha 功能架构                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐│
│  │   数据层     │  │   交易层     │  │   分析层     │  │   工具层     ││
│  │  universal   │  │    trade     │  │   toolbox    │  │   backtest   ││
│  │    info      │  │   multiple   │  │   evaluate   │  │    policy    ││
│  │  realtime    │  │   record     │  │  indicator   │  │    misc      ││
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘│
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                         基础设施层                                   ││
│  │  cons (常量/工具函数)  │  provider (数据源管理)  │  exceptions      ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 模块功能详述

#### 2.2.1 数据层 (Data Layer)

##### 2.2.1.1 universal 模块 - 通用数据获取器

**功能描述**：提供统一的历史日线数据和实时数据获取接口，支持多种数据源。

**核心函数**：

| 函数名 | 功能描述 | 输入参数 | 输出 |
|--------|---------|---------|------|
| `get_daily(code, start, end, prev)` | 获取历史日线数据 | 产品代码、起止日期 | pd.DataFrame |
| `get_rt(code)` | 获取实时行情数据 | 产品代码 | Dict |
| `get_bar(code, interval)` | 获取K线数据 | 产品代码、时间间隔 | pd.DataFrame |

**支持的数据源**：

| 代码格式 | 数据类型 | 数据源 | 示例 |
|---------|---------|-------|------|
| SH/SZ + 6位数字 | 沪深股票/ETF/可转债 | 雪球 | SH600000, SZ000001 |
| HK + 5位数字 | 港股 | 雪球 | HK00700 |
| 纯字母 | 美股 | 雪球 | AAPL, TSLA |
| F + 6位数字 | 场外基金净值 | 天天基金 | F501018 |
| T + 6位数字 | 基金累计净值 | 天天基金 | T501018 |
| M + 6位数字 | 货币基金 | 天天基金 | M000198 |
| XXX/CNY | 人民币汇率中间价 | 外汇局 | USD/CNY |
| indices/xxx | 全球指数 | 英为财情 | indices/germany-30 |
| commodities/xxx | 大宗商品 | 英为财情 | commodities/crude-oil |
| SP + ID | 标普指数 | 标普官网 | SP5475707 |
| ZZ + 代码 | 中证指数 | 中证官网 | ZZ000905 |
| sw- + 代码 | 申万行业 | 聚宽 | sw-801720 |
| peb- + 代码 | PE/PB估值 | 聚宽 | peb-SH000300 |

**技术特性**：
- 支持前复权(.A)、后复权(.B)、不复权(.N)数据
- 内置缓存机制，支持 memory/csv/sql 后端
- 支持 handler 钩子函数自定义数据处理
- 双重验证模式保障数据准确性

##### 2.2.1.2 info 模块 - 基金信息管理

**功能描述**：封装各类基金的基础信息和历史净值数据。

**核心类**：

| 类名 | 功能描述 | 核心属性 |
|------|---------|---------|
| `fundinfo` | 普通开放式基金 | price(净值表), rate(申购费), feeinfo(赎回费), name |
| `mfundinfo` | 货币基金 | price(万份收益累计), name |
| `cashinfo` | 虚拟货币基金 | 可自定义日利率 |
| `indexinfo` | 指数信息 | 指数收盘价 |

**fundinfo 详细功能**：

```python
class fundinfo:
    """
    核心属性:
    - code: str, 6位基金代码
    - name: str, 基金名称
    - rate: float, 申购费率(%)
    - price: pd.DataFrame, 净值表 [date, netvalue, totvalue, comment]
    - feeinfo: List[str], 赎回费率信息
    - segment: List[List], 赎回费分段规则
    
    核心方法:
    - shengou(value, date, fee): 计算申购的日期、现金流、份额
    - shuhui(share, date, rem, fee): 计算赎回的日期、现金流、份额
    - feedecision(days): 根据持有天数返回赎回费率
    - get_stock_holdings(year, season): 获取股票持仓明细
    - get_bond_holdings(year, season): 获取债券持仓明细
    - get_portfolio_holdings(date): 获取股债现金配比
    - get_industry_holdings(year, season): 获取行业持仓分布
    """
```

**特殊处理**：
- 支持基金分红(现金/再投资)
- 支持基金折算(份额变化)
- 支持中港互认基金(96开头)
- 支持增量更新与本地缓存

##### 2.2.1.3 realtime 模块 - 实时行情

**功能描述**：获取各类产品的实时价格和相关信息。

**返回数据结构**：

```python
{
    "name": str,        # 产品名称
    "current": float,   # 当前价格
    "percent": float,   # 涨跌幅(%)
    "current_ext": float,  # 盘后价格(可选)
    "currency": str,    # 计价货币
    "market": str,      # 市场(CN/HK/US)
    "time": str,        # 更新时间
}
```

---

#### 2.2.2 交易层 (Trade Layer)

##### 2.2.2.1 record 模块 - 交易记录

**功能描述**：管理投资者的交易账单记录。

**账单格式规范**：

| 列名 | 类型 | 说明 |
|-----|------|-----|
| date | datetime | 交易日期 |
| [基金代码] | float | 正数表示申购金额，负数表示赎回份额 |

**特殊标记**：
- `0.05` 后缀：切换分红方式(现金↔再投资)
- `-0.005`：按比例赎回(如 -0.005 = 100%清仓)
- 小数点后两位：自定义申购费率

**示例**：

```csv
date,501018,000198
2020-01-02,10000,5000
2020-03-15,-500,0
2020-06-01,5000.05,0
```

##### 2.2.2.2 trade 模块 - 交易处理

**功能描述**：基于 info 和 record 进行交易模拟和分析。

**核心类**：

| 类名 | 功能描述 | 适用场景 |
|------|---------|---------|
| `trade` | 场外基金交易 | 普通基金申购赎回 |
| `itrade` | 场内交易 | 股票、ETF、可转债 |

**trade 类核心功能**：

```python
class trade:
    """
    核心属性:
    - aim: info对象, 交易标的
    - cftable: pd.DataFrame, 现金流量表 [date, cash, share]
    - remtable: pd.DataFrame, 持仓情况表 [date, rem]
    
    核心方法:
    - dailyreport(date): 生成每日持仓报告
    - briefdailyreport(date): 快速获取关键指标
    - xirrrate(date, startdate): 计算内部收益率(XIRR)
    - unitcost(date): 计算单位持仓成本
    - v_tradevolume(freq): 可视化交易量
    - v_tradecost(start, end): 可视化成本与净值
    - v_totvalue(end): 可视化持仓总值变化
    """
```

**日报输出字段**：

| 字段 | 说明 |
|-----|------|
| 基金名称 | 标的名称 |
| 基金代码 | 标的代码 |
| 当日净值 | 最新净值 |
| 单位成本 | 持仓平均成本 |
| 持有份额 | 当前份额 |
| 基金现值 | 当前市值 |
| 基金总申购 | 累计申购金额 |
| 历史最大占用 | 最大投入资金 |
| 基金分红与赎回 | 累计赎回和分红 |
| 换手率 | 年化换手率 |
| 基金收益总额 | 总收益金额 |
| 投资收益率 | 收益率(%) |

##### 2.2.2.3 multiple 模块 - 组合管理

**功能描述**：管理多基金投资组合。

**核心类**：

| 类名 | 功能描述 |
|------|---------|
| `mul` | 基金组合管理(场外+场内) |
| `mulfix` | 固定总投入的组合管理 |
| `imul` | 纯场内交易组合 |

**mul 类核心功能**：

```python
class mul:
    """
    核心方法:
    - combsummary(date): 组合汇总报告
    - xirrrate(date, startdate): 组合整体XIRR
    - evaluation(start): 获取evaluate分析对象
    - get_stock_holdings(year, season): 底层股票持仓穿透
    - get_portfolio(date): 股债现金配比
    - get_industry(date): 行业配置分布
    - v_positions(): 可视化持仓占比
    - v_category_positions(): 可视化大类配置
    """
```

**组合穿透功能**：
- 穿透到底层股票持仓
- 计算等效股票仓位
- 分析行业集中度
- 识别重仓股票

---

#### 2.2.3 分析层 (Analysis Layer)

##### 2.2.3.1 toolbox 模块 - 估值工具箱

**功能描述**：提供历史估值分析和预测工具。

**估值分析类**：

| 类名 | 功能描述 | 数据来源 |
|------|---------|---------|
| `IndexPEBHistory` | 指数PE/PB历史 | 聚宽 |
| `FundPEBHistory` | 基金PE/PB历史 | 聚宽 |
| `SWPEBHistory` | 申万行业PE/PB | 聚宽 |
| `StockPEBHistory` | 个股PE/PB历史 | 雪球 |
| `TEBHistory` | 指数总盈利/净资产 | 聚宽 |

**PEBHistory 核心功能**：

```python
class PEBHistory:
    """
    核心属性:
    - df: pd.DataFrame, 历史PE/PB数据
    - pep: List, PE分位数列表(0-100, 每10%一档)
    - pbp: List, PB分位数列表
    
    核心方法:
    - percentile(): 打印历史分位对应值
    - current(y): 返回当前PE或PB估值
    - current_percentile(y): 返回当前历史百分位
    - summary(): 打印完整估值分析报告
    - v(y): 可视化历史估值走势
    """
```

**预测工具类**：

| 类名 | 功能描述 |
|------|---------|
| `QDIIPredict` | QDII基金净值预测 |
| `RTPredict` | 场内基金实时估算 |
| `CBCalculator` | 可转债定价分析 |
| `Compare` | 多标的收益对比 |
| `OverPriced` | 溢价率分析 |

**QDIIPredict 示例**：

```python
# 预测QDII基金当日净值
qa = xa.QDIIPredict("SH501018", positions=True)
qa.get_t0_rate()  # 返回T+0估算涨跌幅
qa.get_t1_rate()  # 返回T+1估算涨跌幅
qa.benchmark_test()  # 回测预测准确性
```

##### 2.2.3.2 evaluate 模块 - 基金评估

**功能描述**：对基金本身进行多维度评估分析。

**核心功能**：

```python
class evaluate:
    """
    核心方法:
    - correlation(): 基金相关性矩阵
    - v_correlation(): 可视化相关性
    - v_netvalue(): 可视化归一净值走势
    - v_category(): 可视化基金类型分布
    - analyse(): 收益风险分析
    """
```

##### 2.2.3.3 indicator 模块 - 技术指标

**功能描述**：计算各类技术分析指标。

**支持的指标**：

| 指标 | 说明 |
|-----|------|
| MA | 移动平均线 |
| MACD | 指数平滑移动平均线 |
| RSI | 相对强弱指标 |
| BOLL | 布林带 |
| KDJ | 随机指标 |

---

#### 2.2.4 工具层 (Tool Layer)

##### 2.2.4.1 backtest 模块 - 回测框架

**功能描述**：提供动态策略回测环境。

**核心类**：

```python
class BTE:
    """
    BackTestEnvironment - 回测环境基类
    
    核心属性:
    - start: datetime, 回测起始日期
    - end: datetime, 回测结束日期
    - totmoney: float, 初始资金
    - trades: Dict, 交易记录
    - g: GlobalRegister, 全局变量存储
    
    核心方法:
    - prepare(): 初始化函数(需重写)
    - run(date): 每日执行函数(需重写)
    - backtest(): 执行完整回测
    - buy(code, value, date): 买入操作
    - sell(code, share, date): 卖出操作
    - get_current_mul(): 获取当前组合
    - get_current_asset(date): 获取当前资产净值
    """
```

**内置策略类**：

| 类名 | 策略描述 |
|------|---------|
| `Scheduled` | 无脑定投策略 |
| `AverageScheduled` | 价值平均定投 |
| `ScheduledSellonXIRR` | 定投+收益率止盈 |

**自定义策略示例**：

```python
class MyStrategy(xa.backtest.BTE):
    def prepare(self):
        self.code = "F501018"
        self.threshold = 0.05  # 5%阈值
    
    def run(self, date):
        # 获取当前估值
        peb = xa.PEBHistory("SH000300")
        percentile = peb.current_percentile("pe")
        
        # 低估值买入
        if percentile < 30:
            self.buy(self.code, 10000, date)
        # 高估值卖出
        elif percentile > 70:
            current_share = self.trades[self.code].briefdailyreport(date).get("currentshare", 0)
            if current_share > 0:
                self.sell(self.code, current_share * 0.5, date)
```

##### 2.2.4.2 policy 模块 - 投资策略

**功能描述**：提供预定义的投资策略生成器。

**支持的策略**：

| 策略类型 | 说明 |
|---------|------|
| 定额定投 | 固定金额定期投入 |
| 价值平均 | 按目标市值调整投入 |
| 网格交易 | 价格区间内低买高卖 |

##### 2.2.4.3 provider 模块 - 数据源管理

**功能描述**：管理和切换数据源。

**核心功能**：

```python
# 设置聚宽数据源
xa.provider.set_jq_data(debug=True)

# 设置代理
xa.set_proxy("http://127.0.0.1:7890")

# 查看可用数据源
xa.show_providers()
```

---

### 2.3 数据流图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           数据流向图                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌──────────┐                                                         │
│   │ 外部数据源 │                                                         │
│   │ (雪球/天天 │                                                         │
│   │ 基金/英为 │                                                         │
│   │ 财情/聚宽)│                                                         │
│   └────┬─────┘                                                         │
│        │                                                               │
│        ▼                                                               │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐                       │
│   │ universal │────▶│   info   │────▶│  record  │                       │
│   │ get_daily │     │ fundinfo │     │ (用户账单)│                       │
│   │  get_rt   │     │ mfundinfo│     └────┬─────┘                       │
│   └──────────┘     └──────────┘          │                             │
│                                          ▼                             │
│                                    ┌──────────┐                        │
│                                    │  trade   │                        │
│                                    │ (交易模拟)│                        │
│                                    └────┬─────┘                        │
│                                         │                              │
│                          ┌──────────────┼──────────────┐               │
│                          ▼              ▼              ▼               │
│                    ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│                    │ multiple │  │ evaluate │  │ backtest │            │
│                    │ (组合管理)│  │ (基金评估)│  │ (策略回测)│            │
│                    └────┬─────┘  └──────────┘  └──────────┘            │
│                         │                                              │
│                         ▼                                              │
│                    ┌──────────┐                                        │
│                    │ toolbox  │                                        │
│                    │ (估值分析)│                                        │
│                    │ (可视化) │                                        │
│                    └──────────┘                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 非功能需求

### 3.1 性能需求

| 指标 | 要求 |
|-----|------|
| 数据获取响应时间 | 单次 API 调用 < 3s |
| 内存占用 | 单基金数据 < 10MB |
| 缓存命中率 | > 90% (使用本地缓存后) |
| 并发支持 | 支持多线程数据获取 |

### 3.2 兼容性需求

| 环境 | 要求 |
|-----|------|
| Python 版本 | >= 3.6 |
| 操作系统 | Windows/macOS/Linux |
| 运行环境 | 本地/Jupyter/量化云平台 |

### 3.3 依赖项

**核心依赖**：

| 包名 | 用途 |
|-----|------|
| pandas | 数据处理 |
| numpy | 数值计算 |
| requests | 网络请求 |
| beautifulsoup4 | HTML解析 |
| pyecharts | 可视化 |
| scipy | 科学计算 |

**可选依赖**：

| 包名 | 用途 |
|-----|------|
| jqdatasdk | 聚宽数据源 |
| sqlalchemy | SQL后端存储 |
| xlrd | Excel读取 |

### 3.4 数据安全

- 不存储用户敏感信息
- 本地缓存数据支持加密存储
- API Token 临时获取，不持久化

---

## 4. API 参考

### 4.1 快速入门

```python
import xalpha as xa

# 1. 获取基金信息
fund = xa.fundinfo("501018")
print(fund.name, fund.rate)

# 2. 获取历史数据
df = xa.get_daily("SH518880")  # 黄金ETF
rt = xa.get_rt("SH000300")     # 沪深300实时

# 3. 交易模拟
status = xa.record("path/to/trade.csv")
shipan = xa.mul(status=status)
shipan.summary()

# 4. 估值分析
peb = xa.PEBHistory("SH000300")
peb.summary()

# 5. QDII预测
qa = xa.QDIIPredict("SH501018", positions=True)
qa.get_t0_rate()

# 6. 可转债定价
cb = xa.CBCalculator("SH113577")
cb.analyse()
```

### 4.2 完整 API 列表

#### 数据获取

| API | 功能 |
|-----|------|
| `xa.get_daily(code, start, end, prev)` | 历史日线 |
| `xa.get_rt(code)` | 实时行情 |
| `xa.get_bar(code, interval)` | K线数据 |
| `xa.fundinfo(code)` | 基金信息 |
| `xa.mfundinfo(code)` | 货币基金信息 |
| `xa.indexinfo(code)` | 指数信息 |
| `xa.get_fund_holdings(code, year, season)` | 基金持仓 |

#### 交易管理

| API | 功能 |
|-----|------|
| `xa.record(path)` | 读取账单 |
| `xa.irecord(path)` | 读取场内账单 |
| `xa.trade(infoobj, status)` | 创建交易对象 |
| `xa.itrade(code, status)` | 创建场内交易对象 |
| `xa.mul(*trades, status)` | 创建组合 |
| `xa.mulfix(*trades, totmoney)` | 固定投入组合 |

#### 分析工具

| API | 功能 |
|-----|------|
| `xa.PEBHistory(code)` | 估值分析 |
| `xa.TEBHistory(code)` | 盈利分析 |
| `xa.QDIIPredict(code)` | QDII预测 |
| `xa.CBCalculator(code)` | 可转债定价 |
| `xa.Compare(*codes)` | 收益对比 |
| `xa.evaluate(*funds)` | 基金评估 |

#### 配置管理

| API | 功能 |
|-----|------|
| `xa.set_backend(backend, path, prefix)` | 设置缓存后端 |
| `xa.set_proxy(proxy)` | 设置代理 |
| `xa.provider.set_jq_data()` | 启用聚宽数据 |
| `xa.set_display(env)` | 设置显示模式 |
| `xa.set_holdings(module)` | 设置持仓配置 |

---

## 5. 使用场景

### 5.1 场景一：基金定投管理

**用户故事**：作为一个基金定投投资者，我希望能够精确追踪我的定投记录和收益情况。

**解决方案**：

```python
import xalpha as xa

# 读入交易账单
jiaoyidan = xa.record("my_trade.csv")

# 创建组合
shipan = xa.mul(status=jiaoyidan)

# 查看汇总报告
print(shipan.summary())

# 计算年化收益率
print(f"年化收益率: {shipan.xirrrate():.2%}")

# 可视化持仓
shipan.v_positions()
```

### 5.2 场景二：估值择时

**用户故事**：作为一个价值投资者，我希望能够根据指数估值水平决定买入时机。

**解决方案**：

```python
import xalpha as xa

# 获取沪深300估值
peb = xa.PEBHistory("SH000300")
peb.summary()

# 判断当前位置
if peb.current_percentile("pe") < 30:
    print("估值较低，可以考虑加仓")
elif peb.current_percentile("pe") > 70:
    print("估值较高，可以考虑减仓")
else:
    print("估值中等，维持现有仓位")
```

### 5.3 场景三：QDII基金净值预测

**用户故事**：作为一个QDII基金投资者，我希望能够在净值公布前预估当日净值。

**解决方案**：

```python
import xalpha as xa

# 设置持仓信息
from xalpha import holdings
xa.set_holdings(holdings)

# 预测纳斯达克100ETF
qa = xa.QDIIPredict("F513100", positions=True)
print(f"T+0 预估涨跌: {qa.get_t0_rate():.2%}")
print(f"T+1 预估涨跌: {qa.get_t1_rate():.2%}")
```

### 5.4 场景四：策略回测

**用户故事**：作为一个量化研究员，我希望能够回测我的投资策略。

**解决方案**：

```python
import xalpha as xa
import pandas as pd

class ValueAveragingStrategy(xa.backtest.BTE):
    def prepare(self):
        self.code = "F510300"  # 沪深300ETF联接
        self.monthly_target = 10000  # 每月目标市值增加
        self.target = 0
        self.date_range = pd.date_range(self.start, self.end, freq="MS")
        
    def run(self, date):
        if date in self.date_range:
            self.target += self.monthly_target
            current = self.get_current_asset(date)
            
            if self.target > current:
                self.buy(self.code, self.target - current, date)

# 运行回测
bt = ValueAveragingStrategy(
    start="2020-01-01", 
    end="2023-12-31",
    totmoney=500000
)
bt.backtest()

# 查看结果
result = bt.get_current_mul()
print(result.summary())
print(f"年化收益率: {result.xirrrate():.2%}")
```

---

## 6. 路线图

### 6.1 已完成功能 (v0.12.x)

- [x] 通用数据获取器
- [x] 基金信息管理
- [x] 交易模拟系统
- [x] 组合管理
- [x] 估值分析工具
- [x] QDII净值预测
- [x] 可转债定价
- [x] 动态回测框架
- [x] 多数据源支持

### 6.2 规划功能

- [ ] Web UI 界面
- [ ] 实时交易提醒推送
- [ ] 更多数据源接入
- [ ] 机器学习预测模型
- [ ] RESTful API 服务
- [ ] 移动端支持

---

## 7. 附录

### 7.1 术语表

| 术语 | 解释 |
|-----|------|
| PE | 市盈率 (Price-to-Earnings Ratio) |
| PB | 市净率 (Price-to-Book Ratio) |
| XIRR | 内部收益率 (Extended Internal Rate of Return) |
| QDII | 合格境内机构投资者 (Qualified Domestic Institutional Investor) |
| ETF | 交易型开放式指数基金 (Exchange Traded Fund) |
| LOF | 上市型开放式基金 (Listed Open-ended Fund) |
| NAV | 净资产价值 (Net Asset Value) |

### 7.2 参考资源

- 官方文档：https://xalpha.readthedocs.io/
- GitHub 仓库：https://github.com/refraction-ray/xalpha
- PyPI 页面：https://pypi.org/project/xalpha/
- 集思录 QDII 预测：https://www.jisilu.cn/data/qdii/

### 7.3 版本历史

| 版本 | 日期 | 主要更新 |
|-----|------|---------|
| 0.3.0 | - | 支持通用数据获取器 |
| 0.9.0 | - | 支持底层持仓穿透 |
| 0.12.2 | - | 当前稳定版本 |

---

*本文档基于 xalpha v0.12.2 源代码分析生成*
