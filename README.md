# QMT 量化交易系统

一个专业的 Python 量化交易系统，提供完整的回测和实盘交易功能。系统采用模块化设计，支持多种专业量化策略，配备友好的 Streamlit Web 界面。

## 核心特性

- **13 种专业策略**：从经典指标到机构级量化策略
- **双模式运行**：支持历史回测和实盘交易
- **Web 可视化界面**：基于 Streamlit 的专业交易终端
- **多数据源支持**：AKShare（免费）、TuShare、EastMoney
- **模块化架构**：易于扩展的策略开发框架
- **风险控制**：完善的交易风控系统

## 策略列表

### 经典指标策略

| 策略        | 说明             | 适用场景  |
| ----------- | ---------------- | --------- |
| `double_ma` | 双均线交叉策略   | 趋势跟踪  |
| `ma_trend`  | 多均线趋势策略   | 趋势跟踪  |
| `macd`      | MACD 动量策略    | 趋势/震荡 |
| `rsi`       | RSI 超买超卖策略 | 震荡市场  |
| `bollinger` | 布林带均值回归   | 震荡市场  |

### 专业量化策略

| 策略             | 说明                                         | 适用场景      |
| ---------------- | -------------------------------------------- | ------------- |
| `turtle`         | 海龟交易法则（Donchian 通道 + ATR 仓位管理） | 趋势跟踪      |
| `multi_factor`   | 多因子共振（趋势 + 动量 + 成交量）           | 综合选股      |
| `mean_reversion` | Z-Score 统计套利                             | 均值回归      |
| `kama`           | Kaufman 自适应均线                           | 趋势/震荡识别 |
| `vol_breakout`   | ATR 波动率突破                               | 突破交易      |
| `elliott_wave`   | 艾略特波浪 + 斐波那契回调                    | 波浪理论      |

### 其他策略

| 策略              | 说明           | 适用场景 |
| ----------------- | -------------- | -------- |
| `grid`            | 网格交易策略   | 震荡市场 |
| `volume_breakout` | 成交量突破策略 | 放量突破 |

## 快速开始

### 1. 安装依赖

```bash
# 创建虚拟环境（推荐）
python -m venv venv
venv\Scripts\activate  # Windows

# 安装依赖
pip install -r requirements.txt
```

### 2. 启动 Web 界面

```bash
# 完整版（推荐）- 包含所有 13 种策略和可视化
streamlit run ui_pro.py

# 轻量版 - 基础功能
streamlit run ui_simple.py
```

### 3. 使用界面

1. 访问 `http://localhost:8501`
2. 在左侧边栏选择策略和数据源
3. 调整策略参数
4. 点击"回测"运行历史回测
5. 点击"实盘"启动实时交易监控

## 项目结构

```
qmt-quant/
├── data/                    # 数据模块
│   ├── data_gateway.py      # 数据网关抽象基类
│   ├── tushare_gateway.py   # TuShare 数据源
│   ├── akshare_gateway.py   # AKShare 数据源
│   ├── eastmoney_gateway.py # EastMoney 数据源
│   ├── mock_gateway.py      # 模拟数据源
│   └── data_cache.py        # SQLite 数据缓存
│
├── strategies/              # 策略模块
│   ├── strategy_base.py     # 策略基类
│   ├── strategy_context.py  # 策略上下文
│   ├── strategy_engine.py   # 策略引擎
│   ├── backtester.py        # 回测引擎
│   ├── double_ma_strategy.py
│   ├── ma_trend_strategy.py
│   ├── macd_strategy.py
│   ├── rsi_strategy.py
│   ├── bollinger_strategy.py
│   ├── grid_strategy.py
│   ├── volume_breakout_strategy.py
│   ├── turtle_strategy.py       # 海龟交易法则
│   ├── multi_factor_strategy.py # 多因子共振
│   ├── mean_reversion_strategy.py
│   ├── kama_strategy.py         # Kaufman 自适应均线
│   ├── volatility_breakout_strategy.py
│   └── elliott_wave_strategy.py # 艾略特波浪
│
├── broker/                  # 交易模块
│   ├── broker_gateway.py    # 券商网关抽象基类
│   ├── miniqmt_gateway.py   # MiniQMT/模拟交易接口
│   └── risk_controller.py   # 风险控制模块
│
├── utils/                   # 工具模块
│   ├── config_manager.py    # 配置管理
│   ├── logger.py            # 日志系统
│   └── monitor.py           # 系统监控
│
├── config/                  # 配置文件
│   └── config.yaml
│
├── ui_pro.py                # Web 界面（完整版）
├── ui_simple.py             # Web 界面（轻量版）
├── main.py                  # 命令行入口
├── backtest_all_stocks.py   # 批量回测脚本
└── requirements.txt         # 依赖列表
```

## 策略开发指南

### 创建自定义策略

继承 `StrategyBase` 基类，实现 `initialize()` 和 `on_bar()` 方法：

```python
from strategies.strategy_base import StrategyBase
import pandas as pd


class MyStrategy(StrategyBase):
    def __init__(self, name: str = None):
        super().__init__(name)
        self._period = 20  # 策略参数

    def initialize(self):
        """策略初始化"""
        self._period = self.parameters.get('period', 20)
        self.set_variable('position', {})

    def on_bar(self, data: pd.DataFrame):
        """处理 K 线数据"""
        for symbol, df in data.items():
            if df.empty or len(df) < self._period:
                continue

            current_price = df['close'].iloc[-1]
            ma = df['close'].rolling(self._period).mean().iloc[-1]

            position = self.get_position(symbol)

            # 买入信号：价格上穿均线
            if current_price > ma and position.volume == 0:
                cash = self.get_cash()
                volume = int(cash * 0.95 / current_price / 100) * 100
                if volume >= 100:
                    self.buy(symbol, volume)
                    print(f"买入 {symbol} {volume}股 @ {current_price:.2f}")

            # 卖出信号：价格下穿均线
            elif current_price < ma and position.volume > 0:
                self.sell(symbol, position.volume)
                print(f"卖出 {symbol} {position.volume}股 @ {current_price:.2f}")
```

### 可用 API

```python
# 交易操作
self.buy(symbol, volume, price=None)       # 买入
self.sell(symbol, volume, price=None)      # 卖出
self.cancel_order(order_id)                # 撤单

# 账户信息
self.get_cash()                            # 可用资金
self.get_position(symbol)                  # 持仓信息
self.get_total_value()                     # 总资产

# 状态管理
self.set_variable(key, value)              # 设置持久化变量
self.get_variable(key, default)            # 获取持久化变量
self.parameters.get(key, default)          # 获取策略参数

# 工具方法
self.cross_above(s1, s2)                   # 判断上穿
self.cross_below(s1, s2)                   # 判断下穿
```

## 回测引擎

```python
from strategies.backtester import Backtester, BacktestConfig
from strategies.double_ma_strategy import DoubleMaStrategy

# 配置回测
config = BacktestConfig(
    initial_cash=1000000,
    commission_rate=0.0003,
    stamp_tax=0.001,
    slippage=0.001
)

# 运行回测
backtester = Backtester(strategy, config)
results = backtester.run(
    data_dict,  # {symbol: DataFrame}
    start_date='20230101',
    end_date='20231231'
)

# 查看结果
print(f"总收益率：{results['total_return']:.2%}")
print(f"年化收益：{results['annual_return']:.2%}")
print(f"最大回撤：{results['max_drawdown']:.2%}")
print(f"夏普比率：{results['sharpe_ratio']:.2f}")
```

## 配置说明

### 数据源配置

| 数据源      | 说明               | 费用   |
| ----------- | ------------------ | ------ |
| `akshare`   | 开源免费数据源     | 免费   |
| `tushare`   | 专业金融数据       | 积分制 |
| `eastmoney` | 东方财富实时行情   | 免费   |
| `mock`      | 模拟数据（测试用） | -      |

### 交易成本

| 项目   | 默认值 | 说明         |
| ------ | ------ | ------------ |
| 佣金   | 0.3‱   | 买卖双向收取 |
| 印花税 | 1‱     | 仅卖出收取   |
| 滑点   | 1‱     | 预估冲击成本 |

## 风险控制

系统内置风控模块，支持以下规则：

```python
from broker.risk_controller import RiskController

risk = RiskController()

# 日亏损限额
risk.add_daily_loss_limit(50000)  # 日亏 5 万停止

# 单只股票仓位限制
risk.add_position_limit(0.3)  # 单票不超过 30%

# 总仓位限制
risk.add_total_position_limit(0.95)  # 总仓不超过 95%

# 单笔委托限制
risk.add_order_volume_limit(5000)  # 单笔不超过 5000 股
```

## 技术栈

- **语言**: Python 3.8+
- **数据处理**: Pandas, NumPy
- **Web 框架**: Streamlit
- **图表库**: Plotly
- **数据源**: AKShare, TuShare, EastMoney
- **缓存**: SQLite

## 依赖安装

```bash
# 核心依赖
pip install numpy pandas pyyaml psutil

# 数据源（至少安装一个）
pip install akshare tushare

# Web 界面
pip install streamlit plotly
```

## 注意事项

1. **数据质量**: 免费数据源可能存在延迟或错误，实盘交易请考虑使用付费数据源
2. **回测局限**: 回测结果仅供参考，实盘需考虑滑点、冲击成本、流动性等因素
3. **交易风险**: 量化交易存在亏损风险，请谨慎使用
4. **合规要求**: 实盘交易需遵守相关法律法规和券商规定

## 许可证

MIT License

## 免责声明

本系统仅供学习研究使用，不构成任何投资建议。使用本系统进行交易产生的所有风险和损失由用户自行承担。
