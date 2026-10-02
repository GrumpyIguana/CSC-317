# Crypto Paper-Trading Bot — Technical Requirements

## 1. Objective

Build a small Python-based cryptocurrency paper-trading bot that can consume real market data, make trades according to a defined strategy, simulate orders against a fake portfolio, and record performance.

Version 1 must **not place real trades or have access to real funds**.

The goal is to create a reliable trading framework first. AI can be added later as an optional signal-generation layer.

## 2. Initial Scope

The bot should:

- Start with a configurable fake balance, defaulting to **$50 USD**
- Trade one configurable crypto pair initially, such as `SOL/USD` or `BTC/USD`
- Read current market prices from an exchange API
- Generate buy, sell, or hold decisions
- Simulate orders rather than submitting real orders
- Track cash and crypto balances
- Account for trading fees
- Record every trade
- Calculate portfolio performance
- Compare bot performance against simple buy-and-hold

Do not implement live trading in V1.

## 3. Technology

Use:

- Python 3.12+
- CCXT for exchange market data
- Pandas for historical data and analysis
- SQLite for persistent storage
- Pytest for testing
- Standard Python logging

Prefer simple, readable code over unnecessary abstractions.

## 4. Architecture

Use this general flow:

`Exchange Market Data`
→ `Market Data Service`
→ `Trading Strategy`
→ `Risk Manager`
→ `Paper Execution Engine`
→ `Portfolio`
→ `Trade Database`
→ `Performance Report`

Each component should be isolated so it can later be replaced independently.

## 5. Components

### Market Data Service

Responsibilities:

- Connect to the configured exchange through CCXT
- Retrieve current ticker prices
- Retrieve OHLCV candle data
- Normalize returned market data
- Handle API errors and rate limits
- Retry temporary failures

Configuration should include:

- Exchange name
- Trading pair
- Candle interval
- Lookback period

### Trading Strategy

Define a common strategy interface.

```python
class Strategy:
    def generate_signal(self, market_data) -> str:
        """
        Returns:
        BUY
        SELL
        HOLD
        """
```

Start with one simple deterministic strategy.

Recommended first strategy:

**Moving Average Crossover**

Example:

- Short moving average crosses above long moving average → BUY
- Short moving average crosses below long moving average → SELL
- Otherwise → HOLD

Strategy parameters should be configurable.

### Risk Manager

All trades must pass through the risk manager.

The strategy itself must never decide position size.

Rules should include:

- Maximum percentage of portfolio per trade
- Maximum total exposure
- Minimum cash reserve
- Optional stop-loss percentage
- Optional take-profit percentage
- Prevent buying without sufficient cash
- Prevent selling more crypto than the portfolio owns
- Prevent duplicate orders caused by repeated signals

Default example:

- Maximum 20% of portfolio per new trade
- No leverage
- No shorting
- No borrowing
- Spot trading only

### Paper Execution Engine

Simulate exchange execution.

Each simulated trade should account for:

- Current market price
- Configurable trading fee
- Optional simulated slippage
- Trade quantity
- Total order value

The execution engine updates the fake portfolio but does not contact any live-order endpoint.

### Portfolio Manager

Track:

- Cash balance
- Crypto quantity
- Average entry price
- Realized profit/loss
- Unrealized profit/loss
- Total portfolio value
- Percentage return

Portfolio state must survive application restarts.

## 6. Database

Use SQLite.

Suggested tables:

### trades

Fields:

- id
- timestamp
- exchange
- symbol
- side
- quantity
- market_price
- execution_price
- fee
- total_value
- strategy_name
- signal_reason

### portfolio_snapshots

Fields:

- timestamp
- cash_balance
- asset_quantity
- asset_price
- portfolio_value
- realized_pnl
- unrealized_pnl

### bot_runs

Fields:

- run_id
- start_time
- end_time
- strategy
- configuration
- starting_balance
- ending_balance

## 7. Logging

Log:

- Market-data requests
- Generated signals
- Rejected trades
- Executed paper trades
- API failures
- Portfolio updates
- Exceptions

Use rotating log files.

Never log API secrets.

## 8. Configuration

Use a configuration file such as:

`config.yaml`

Example:

```yaml
exchange: kraken
symbol: SOL/USD

paper_trading:
  starting_balance: 50
  fee_percent: 0.4
  slippage_percent: 0.1

strategy:
  name: moving_average
  short_window: 10
  long_window: 30

risk:
  max_trade_percent: 20
  max_exposure_percent: 80
  stop_loss_percent: 5
```

No strategy parameters should be hardcoded.

## 9. Simulation Modes

Support two modes.

### Historical Backtest

Run the strategy against historical OHLCV data.

Output:

- Starting portfolio value
- Ending value
- Percentage return
- Number of trades
- Winning trades
- Losing trades
- Maximum drawdown
- Fees paid

### Forward Paper Trading

Use live market prices but simulated money.

The bot should wake up on a configurable schedule, evaluate the strategy, and simulate trades.

Example:

```text
Every 5 minutes:
1. Download latest candles
2. Generate signal
3. Run risk checks
4. Simulate trade if approved
5. Save portfolio snapshot
```

## 10. Benchmark

The bot must compare itself against buy-and-hold.

This prevents the bot from appearing successful simply because the overall market rose.

## 11. Testing

Add unit tests for:

- Buy execution
- Sell execution
- Trading fees
- Slippage calculations
- Insufficient cash
- Insufficient crypto balance
- Position-size limits
- Strategy signals
- Portfolio P&L calculations
- Database persistence

Tests must not call real trading endpoints.

## 12. Safety Requirements

Version 1 must have **no live order execution code**.

Do not request or require exchange withdrawal permissions.

If API credentials are eventually required for market data:

- Read them from environment variables
- Never hardcode them
- Never store them in Git
- Never print them in logs

Create `.env.example` but do not create a real `.env` with credentials.

## 13. Project Structure

Suggested structure:

```text
trading_bot/
│
├── main.py
├── config.yaml
├── requirements.txt
├── README.md
│
├── bot/
│   ├── market_data.py
│   ├── strategy.py
│   ├── risk.py
│   ├── execution.py
│   ├── portfolio.py
│   ├── database.py
│   └── reporting.py
│
├── strategies/
│   └── moving_average.py
│
├── tests/
│   ├── test_execution.py
│   ├── test_portfolio.py
│   ├── test_risk.py
│   └── test_strategy.py
│
└── data/
    └── trading_bot.db
```

## 14. CLI

Provide commands such as:

```bash
python main.py backtest
python main.py paper
python main.py status
python main.py report
```

## 15. AI — Future Version

Do **not** include AI in the execution path for V1.

Later, implement an optional AI signal provider.

Possible inputs:

- Recent candles
- Momentum indicators
- Trading volume
- Volatility
- Market trend
- News sentiment

The AI should only produce a signal.

The deterministic risk manager must still decide whether that signal is allowed to become a trade.

AI must never:

- Bypass risk limits
- Modify account permissions
- Transfer funds
- Withdraw funds
- Change its own maximum position size

## 16. Acceptance Criteria

V1 is complete when:

1. The application can download real crypto prices.
2. A strategy can generate BUY, SELL, and HOLD signals.
3. Trades are simulated against a fake $50 portfolio.
4. Fees and slippage are included.
5. Risk limits prevent invalid trades.
6. Portfolio state persists between runs.
7. Trades are saved to SQLite.
8. A historical backtest can be run.
9. Forward paper trading can run continuously.
10. Performance can be compared against buy-and-hold.
11. Unit tests pass.
12. No real order-placement capability exists.

## Codex Implementation Instruction

Implement this project incrementally.

Start by creating the project structure and configuration system.

Then implement, in this order:

1. Market data
2. Portfolio
3. Paper execution
4. Risk manager
5. Strategy interface
6. Moving-average strategy
7. SQLite persistence
8. Backtesting
9. Forward paper trading
10. Reporting
11. Tests
12. README

After each stage, run the test suite before proceeding.

Do not implement live trading yet.

Favor straightforward Python and explicit logic over complex frameworks.
