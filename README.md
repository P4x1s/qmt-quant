# QMT 量化交易系统

一个模块化、可扩展的 Python 量化交易系统，模拟 miniQMT 的核心功能，支持 A 股市场的全自动实盘交易。

## 项目结构

```
qmt-quant/
├── data/                   # 数据模块
│   ├── __init__.py
│   ├── data_gateway.py     # 数据网关抽象基类
│   ├── tushare_gateway.py  # TuShare 数据源实现
│   ├── akshare_gateway.py  # AKShare 数据源实现
│   └── data_cache.py       # SQLite 数据缓存
│
├── strategies/             # 策略模块
│   ├── __init__.py
│   ├── strategy_base.py    # 策略基类
│   ├── strategy_context.py # 策略上下文
│   ├── strategy_engine.py  # 策略引擎
│   ├── backtester.py       # 回测引擎
│   ├── double_ma_strategy.py   # 双均线策略示例
│   ├── grid_strategy.py        # 网格策略示例
│   └── momentum_strategy.py    # 动量轮动策略示例
│
├── broker/                 # 交易模块
│   ├── __init__.py
│   ├── broker_gateway.py   # 券商网关抽象基类
│   ├── miniqmt_gateway.py  # MiniQMT 接口实现
│   └── risk_controller.py  # 风险控制模块
│
├── utils/                  # 工具模块
│   ├── __init__.py
│   ├── config_manager.py   # 配置管理
│   ├── logger.py           # 日志系统
│   └── monitor.py          # 系统监控
│
├── config/                 # 配置文件
│   └── config.yaml         # 系统配置
│
├── examples/               # 示例代码
├── tests/                  # 单元测试
├── logs/                   # 日志文件
├── cache/                  # 数据缓存
│
├── main.py                 # 主程序入口
└── requirements.txt        # 依赖列表
```

## 快速开始

### 1. 安装依赖

```bash
# 创建虚拟环境（推荐）
python -m venv venv
venv\Scripts\activate  # Windows

# 安装依赖
pip install -r requirements.txt
```

### 2. 配置系统

编辑 `config/config.yaml` 文件：

```yaml
# 选择数据源
data:
  source: akshare  # 或 tushare

# 选择交易模式
broker:
  type: simulated_miniqmt  # 模拟交易
  # type: miniqmt  # 实盘交易

# 策略配置
strategy:
  name: double_ma
  parameters:
    short_period: 5
    long_period: 20
```

### 3. 运行回测

```bash
# 使用默认配置运行回测
python main.py --mode backtest --start-date 20230101 --end-date 20231231
```

### 4. 运行实盘（模拟）

```bash
python main.py --mode live
```

## 核心功能

### 1. 数据模块

支持多数据源切换：

```python
from data.akshare_gateway import AKShareGateway

# 初始化数据网关
gateway = AKShareGateway()
gateway.connect()

# 获取历史数据
df = gateway.get_history_data(
    symbol='000001.SZ',
    start_date='20230101',
    end_date='20231231',
    frequency='1d'
)

# 获取实时行情
quotes = gateway.get_realtime_quotes(['000001.SZ', '600036.SH'])
```

### 2. 策略开发

继承 `StrategyBase` 创建你的策略：

```python
from strategies.strategy_base import StrategyBase
import pandas as pd

class MyStrategy(StrategyBase):
    def initialize(self):
        # 设置策略参数
        self.set_parameters(
            short_period=5,
            long_period=20
        )

    def on_bar(self, data: pd.DataFrame):
        for symbol, df in data.items():
            # 计算指标
            ma5 = df['close'].rolling(5).mean().iloc[-1]
            ma20 = df['close'].rolling(20).mean().iloc[-1]

            # 交易逻辑
            position = self.get_position(symbol)

            if ma5 > ma20 and position.volume == 0:
                self.buy(symbol, 100)
            elif ma5 < ma20 and position.volume > 0:
                self.sell(symbol, position.volume)
```

### 3. 回测引擎

```python
from strategies.strategy_context import StrategyContext
from strategies.backtester import Backtester, BacktestConfig

# 创建策略和上下文
context = StrategyContext()
context.subscribe('000001.SZ')
context.add_history_data('000001.SZ', historical_df)

strategy = MyStrategy()

# 配置回测
config = BacktestConfig(
    initial_cash=1000000,
    commission_rate=0.0003,
    slippage=0.001
)

# 运行回测
backtester = Backtester(strategy, context, config)
result = backtester.run('20230101', '20231231')

# 查看结果
print(result.summary())
```

### 4. 风险控制

```python
from broker.risk_controller import RiskController, RiskRule, RiskRuleType

# 创建风控控制器
controller = RiskController()

# 添加自定义规则
controller.add_rule(RiskRule(
    rule_id='my_rule',
    rule_type=RiskRuleType.ORDER_LIMIT,
    name='单笔委托限制',
    params={'max_volume': 5000, 'max_value': 500000}
))

# 检查订单
passed, violations = controller.check_order(
    symbol='000001.SZ',
    direction='buy',
    volume=1000,
    price=20.0
)

if not passed:
    print(f"订单被拒绝：{violations}")
```

### 5. 实盘交易（MiniQMT）

```python
from broker.miniqmt_gateway import MiniQMTGateway

# 配置 MiniQMT
config = {
    'account_id': 'your_account',
    'path': 'C:\\Program Files\\国金证券\\qxtrade',
    'port': 58610
}

gateway = MiniQMTGateway(config)
gateway.connect()

# 下单
order = gateway.buy('000001.SZ', 100, 20.0)

# 查询持仓
positions = gateway.get_positions()

# 查询账户
account = gateway.get_account_info()
```

## 系统配置说明

### 数据源配置

| 参数               | 说明                         | 默认值  |
| ------------------ | ---------------------------- | ------- |
| data.source        | 数据源类型 (akshare/tushare) | akshare |
| data.cache.enabled | 启用缓存                     | true    |
| data.tushare.token | TuShare API token            | -       |

### 交易配置

| 参数                | 说明             | 默认值            |
| ------------------- | ---------------- | ----------------- |
| broker.type         | 券商类型         | simulated_miniqmt |
| broker.initial_cash | 模拟账户初始资金 | 1000000           |

### 风控配置

| 参数                            | 说明             | 默认值 |
| ------------------------------- | ---------------- | ------ |
| risk.enabled                    | 启用风控         | true   |
| risk.daily_loss_limit           | 日亏损限额       | 50000  |
| risk.max_position_percent       | 单只股票最大仓位 | 0.3    |
| risk.max_total_position_percent | 总仓位上限       | 0.95   |

## 策略开发指南

### 策略生命周期

1. `initialize()` - 策略初始化，设置参数
2. `on_bar(data)` - 处理 K 线数据
3. `on_tick(tick)` - 处理 Tick 数据（可选）
4. `on_order(order)` - 订单状态更新（可选）
5. `on_trade(trade)` - 成交回调（可选）

### 可用方法

```python
# 交易方法
self.buy(symbol, volume, price)      # 买入
self.sell(symbol, volume, price)     # 卖出
self.cancel_order(order_id)          # 撤单

# 信息查询
self.get_position(symbol)            # 获取持仓
self.get_cash()                      # 获取可用资金
self.get_total_value()               # 获取总资产

# 参数和变量
self.set_parameters(key=value)       # 设置参数
self.get_parameter(key, default)     # 获取参数
self.set_variable(key, value)        # 设置变量
self.get_variable(key, default)      # 获取变量

# 工具方法
self.cross_above(s1, s2)             # 判断上穿
self.cross_below(s1, s2)             # 判断下穿
```

## 注意事项

1. **数据源**: AKShare 完全免费，但数据质量可能不如付费源；TuShare 需要 token，但数据更稳定
2. **回测**: 回测结果仅供参考，实盘需考虑滑点、冲击成本等因素
3. **实盘**: 使用 MiniQMT 实盘交易前，请确保：
   - 已安装 miniQMT 客户端
   - 已开通量化交易权限
   - 充分了解交易风险
4. **合规**: 实盘交易需遵守相关法律法规和监管要求

## 技术栈

- **Python**: 3.8+
- **数据处理**: Pandas, NumPy
- **数据源**: AKShare, TuShare
- **交易接口**: xtquant (MiniQMT)
- **缓存**: SQLite
- **配置**: YAML
- **日志**: logging

## 许可证

MIT License

## 风险提示

量化交易存在风险，本系统仅供学习研究使用，实盘交易需谨慎。
使用本系统进行的任何交易，风险由用户自行承担。
