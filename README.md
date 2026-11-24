# livebot

A live trend-following trading bot for **OKX perpetual swaps** (isolated margin, market orders). It pulls 1-hour candles from OKX, enters on strong ADX trends confirmed by DI, RSI and the 200-period SMA, and manages exits with ATR-based stop-loss / take-profit plus a DI-flip early exit. Trade results are logged to CSV and a weekly PnL summary is emailed.

> ⚠️ **This bot trades with real money.** All OKX clients are created with `flag="0"` (live trading). Use OKX demo trading (`flag="1"`) or a small `POS_SIZE` until you trust it.

---

## How it works

The bot runs an infinite loop (about once per second). Each iteration it builds a market snapshot and steps a small finite-state machine:

```
FLAT ──(entry signal)──▶ IN_TRADE ──(TP / SL / DI flip)──▶ COOLDOWN ──(timer expires)──▶ FLAT
```

### Market snapshot (`build_market_context`)
From the last 300 one-hour OKX candles for `SYMBOL`:
- **ADX(14)**, **DI+**, **DI−** and the previous candle's ADX
- **SMA(200)** and the latest close
- **RSI(14)**
- **ATR(14)**, Wilder/EWM style (matches TradingView)
- Current mark price of the open position

### Entry (`FLAT` state)
Entries are only checked in the first 30 seconds of each hour (on candle close), and at most once per candle.

| Signal | Conditions |
|---|---|
| **Long** (`buy`)  | ADX > 25, DI+ > DI−, RSI < 70, close > SMA200 |
| **Short** (`sell`) | ADX > 25, DI− > DI+, RSI > 30, close < SMA200 |

Orders are sent with `place_trade(side)` as isolated-margin market orders of size `POS_SIZE`.

### Exit (`IN_TRADE` state)
- **Stop-loss:** 1.5 × ATR from the entry price.
- **Take-profit:** a multiple of the stop distance, scaled by the current ADX:

  | ADX | TP distance |
  |---|---|
  | ≤ 18 | 1 × SL |
  | 18 – 20 | 1.5 × SL |
  | 20 – 25 | 3 × SL |
  | 25 – 30 | 2 × SL |
  | 30 – 40 | 3 × SL |
  | ≥ 40 | 4 × SL |

- **DI-flip exit:** if the opposite DI takes over while ADX ≥ 22 and falling, a counter increases; after 3 flips the position is closed early.

Closing orders are `reduceOnly` market orders. After each close the bot checks that the position is really gone on the exchange.

### Risk controls & cooldown
| Setting | Value |
|---|---|
| `MAX_TRADES_PER_DAY` | 6 |
| `DAILY_LOSS_LIMIT` | 3% (sum of losing trades' % PnL) |
| `WIN_COOLDOWN_HOURS` | 6 |
| `LOSS_COOLDOWN_HOURS` | 6 |

Daily counters reset at UTC midnight.

### Persistence & reporting
- **State** is saved atomically to `/home/ubuntu/livebot/state_<BOT_NAME>.json` after every step and reloaded on restart, so the bot picks up an open trade after a crash or reboot.
- **Trade log** is appended to `pnl_log.csv` (in the working directory): `timestamp, side, entry, exit, pnl`.
- **Weekly report:** every Sunday between 00:00 and 00:05 UTC, a summary (trade count, win rate, average and total PnL, plus the trade list) is emailed through Gmail SMTP.

---

## Requirements

- Python 3.9+
- An OKX account with an API key that has **trade** permission
- A Gmail account with an [App Password](https://support.google.com/accounts/answer/185833) (for the weekly email)

Install dependencies:

```bash
pip install pandas numpy requests ta python-okx joblib torch
```

> `torch` and `joblib` are imported but not currently used by the strategy; you can remove those imports if you want a lighter install.

---

## Configuration

Everything about the trade is set with environment variables:

| Variable | Description | Example |
|---|---|---|
| `OKX_API_KEY` | OKX API key | |
| `OKX_API_SECRET` | OKX API secret | |
| `OKX_API_PASSPHRASE` | OKX API passphrase | |
| `SYMBOL` | OKX instrument ID | `AVAX-USDT-SWAP` |
| `POS_SIZE` | Order size in contracts | `1` |
| `BOT_NAME` | Name used for the state file (default `default`) | `avax` |
| `SENDER_EMAIL` | Gmail address the weekly report is sent from | `you@gmail.com` |
| `RECEIVER_EMAIL` | Where the weekly report goes | `you@gmail.com` |
| `GMAIL_APP_PASSWORD` | Gmail App Password for `SENDER_EMAIL` | |

If any of the three email variables is missing, the bot keeps trading and just skips the weekly email.

Example `.env` file (already ignored by `.gitignore`):

```bash
OKX_API_KEY=xxxxxxxx
OKX_API_SECRET=xxxxxxxx
OKX_API_PASSPHRASE=xxxxxxxx
SYMBOL=AVAX-USDT-SWAP
POS_SIZE=1
BOT_NAME=avax
SENDER_EMAIL=you@gmail.com
RECEIVER_EMAIL=you@gmail.com
GMAIL_APP_PASSWORD=xxxxxxxxxxxxxxxx
```

Never commit real credentials to the repo; keep them in `.env` or your server's environment.

---

## Running

```bash
mkdir -p /home/ubuntu/livebot      # state file location
set -a && source .env && set +a
python goinglivenow.py
```

To keep it running after you log out, use `tmux`, `screen`, `nohup`, or a `systemd` service:

```bash
nohup python goinglivenow.py > bot.log 2>&1 &
```

You can run several bots on different symbols on the same machine; give each one a different `BOT_NAME` so they keep separate state files.

---

## Project structure

```
livebot/
├── goinglivenow.py   # the bot: data, indicators, strategy, order handling, FSM, main loop
├── .gitignore
└── README.md
```

### Key functions in `goinglivenow.py`

| Function | Purpose |
|---|---|
| `preprocess`, `rsi`, `adx_ma_close` | Fetch candles from OKX and compute indicators |
| `get_account_bal` | Available account balance |
| `get_current_price_TPSL`, `get_entry_price`, `get_exit_price` | Mark, entry and last fill prices |
| `position_exists` | Check whether a position is open on OKX |
| `place_trade` | Open a position (market order) |
| `TP_long`, `SL_long`, `TP_short`, `SL_short` | Close a position (reduce-only market order) |
| `calculate_sl_tp`, `check_close_condition` | ATR-based exit logic |
| `fsm_flat`, `fsm_in_trade`, `fsm_cooldown`, `fsm_step` | State machine |
| `save_state`, `load_state`, `save_pnl_log` | Persistence |
| `send_weekly_report` | Weekly PnL email |

---

## Disclaimer

This software is provided for educational purposes. Crypto derivatives trading carries significant risk of loss. You are solely responsible for any trades the bot places.
