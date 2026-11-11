# Grok Memecoin Momentum Bot (Solana)

An automatic trading bot for Solana memecoins that uses **Grok** (xAI's API) to decide what to buy.

> **Grok on its own cannot trade.** The Grok chat app has no wallet and can't sign transactions.
> This bot is a Python program that runs on **your computer** (or a cheap server). It collects the
> market data, runs safety checks, asks Grok for a verdict through the API, and makes the trades
> itself through Jupiter.

> ⚠️ **Read this first.** Memecoins are among the riskiest things you can trade. Most go to zero,
> rug pulls are common, and no bot or AI can predict them reliably. This bot can and will lose
> money. Only use money you can afford to lose completely. Never paste your private key or seed
> phrase into Grok, ChatGPT, Discord, or anywhere else; it belongs only in your local `.env` file.
> Not financial advice.

---

## How it works

```
 every 5 min                                                   every 30 s
 ┌──────────────────────┐                                      ┌────────────────────────┐
 │ 1. Find trending     │  GeckoTerminal trending pools,       │ Check open positions   │
 │    Solana tokens     │  DexScreener boosts/profiles         │  • stop-loss  -20%     │
 ├──────────────────────┤                                      │  • take-profit +50%    │
 │ 2. Momentum filters  │  liquidity, volume, buy/sell ratio,  │    (sell half)         │
 │                      │  1h/6h trend, not over-pumped, age   │  • trailing stop 25%   │
 ├──────────────────────┤                                      │    from the peak       │
 │ 3. Safety checks     │  mint + freeze authority disabled,   │  • max hold 24h        │
 │                      │  RugCheck report, simulated          └────────────────────────┘
 │                      │  buy→sell through Jupiter            Mechanical rules. Grok
 ├──────────────────────┤  (catches honeypots & sell taxes)    cannot override them.
 │ 4. Ask Grok          │  gets all the data + searches X,
 │                      │  replies BUY/SKIP + confidence
 ├──────────────────────┤
 │ 5. Buy via Jupiter   │  only if BUY and confidence ≥ 70
 └──────────────────────┘
```

Account-level protections: 3% of equity per trade (hard cap 10%), max 4 open positions,
max 8 buys a day, a **daily loss limit of 10%** (no new buys for the rest of the UTC day), a
24h cooldown before buying the same coin again, and an SOL reserve for fees.

The prompt sent to Grok is in **`grok_prompt.txt`**. You can edit it without touching the code.

---

## Setup (about 15 minutes)

### 1. Install Python
Install **Python 3.10 or newer** from <https://www.python.org/downloads/>.
On Windows, tick **"Add Python to PATH"** in the installer.

### 2. Download the bot and install its packages
Open a terminal (Windows: *Command Prompt*; Mac: *Terminal*) in the `memecoin-bot` folder:

```bash
python -m venv .venv
# Windows:
.venv\Scripts\activate
# Mac / Linux:
source .venv/bin/activate

pip install -r requirements.txt
```

### 3. Get your keys
| What | Where | Needed for |
|---|---|---|
| **xAI API key** (Grok) | <https://console.x.ai>. Add a few dollars of credit. API billing is separate from X Premium. | always |
| **Jupiter API key** (free) | Jupiter's developer portal (linked from <https://developers.jup.ag>) | optional, gives higher rate limits |
| **Solana RPC URL** (free) | e.g. Helius, QuickNode or Triton free tier | live mode (the public RPC works but is slow) |
| **A new Solana wallet** | Create a **fresh** account in Phantom/Solflare just for the bot. Export its private key. | live mode only |

### 4. Create your `.env` file
Copy `.env.example` to a new file named `.env` and fill in at least `XAI_API_KEY`.
Every setting is explained in the file.

### 5. Run it in paper mode first
```bash
python bot.py
```
`DRY_RUN=true` is the default. The bot uses real market data and real Jupiter quotes, but
**fake money** (`PAPER_START_SOL=2`). Let it run for **at least several days**. Then check:

- `trades_paper.csv`: every buy and sell with profit or loss
- `bot.log`: everything the bot did, including Grok's reasoning for each coin
- `python bot.py --status`: current balance and open positions

### 6. Go live (only if paper results are good)
1. Send your trading budget (plus ~0.05 SOL for fees) to the **new** bot wallet.
2. In `.env` set `SOLANA_PRIVATE_KEY=...`, `RPC_URL=...` and `DRY_RUN=false`.
3. Run `python bot.py` and type `LIVE` to confirm.

---

## Controls

| Do this | To |
|---|---|
| Create a file named `STOP` in the bot folder | pause new buys (exits keep running). Delete it to resume. |
| `Ctrl + C` | quit. Open positions are saved and managed again when you restart. |
| `python bot.py --status` | see balance and open positions |
| `python bot.py --sell-all` | sell every open position immediately |
| `python bot.py --once` | run a single cycle (good for testing your setup) |

Paper and live data are kept separately (`state_paper.json` / `state_live.json`).
**Don't delete `state_live.json` while you hold positions.** The bot uses it to know what it owns.

---

## Settings for a $100–$1,000 account

The defaults in `.env.example` are set for this range:

| Setting | Default | Meaning |
|---|---|---|
| `POSITION_SIZE_PCT` | 3 | 3% of equity per trade (5 SOL account → 0.15 SOL per trade) |
| `MAX_OPEN_POSITIONS` | 4 | at most ~12% of the account in memecoins at once |
| `STOP_LOSS_PCT` | 20 | sell all at -20% |
| `TAKE_PROFIT_PCT` / `TAKE_PROFIT_SELL_PCT` | 50 / 50 | at +50% sell half |
| `TRAILING_STOP_PCT` | 25 | after that, sell the rest if it drops 25% from its peak |
| `DAILY_MAX_LOSS_PCT` | 10 | stop buying for the day after -10% |
| `MIN_CONFIDENCE` | 70 | Grok must be at least this sure |

Gains and losses are measured by what you'd **actually get selling right now** (a real Jupiter
quote, including slippage). A new position therefore shows about -1 to -3% right after buying.
A stop-loss can fill worse than -20% if a coin crashes between checks, and a full rug pull
can mean -100% on that trade. That is why each position is kept small.

---

## Costs
- **Grok API**: each verdict is one API call (more tokens when X search is on). Limit spend with
  `MAX_GROK_CALLS_PER_SCAN` and `MAX_GROK_CALLS_PER_DAY`, and see current prices at <https://x.ai/api>.
  Most coins are rejected by the free filters and safety checks before Grok is ever asked.
- **Solana fees**: network and priority fees per swap, plus ~0.002 SOL rent for each new token account.
- **Jupiter**: Jupiter may charge a small swap fee; see their docs.

## Troubleshooting
- **`HTTP 401` from Grok**: wrong `XAI_API_KEY` or no credits.
- **Grok model error / "Responses API refused"**: the model name may have changed. Check
  <https://docs.x.ai> and update `GROK_MODEL`.
- **Lots of `429` or slow scans**: get a free Jupiter API key and a private RPC URL.
- **"Scan: ... 0 passed momentum filters"**: normal in quiet markets. Set `DEBUG=true` to see why
  each coin was rejected. Loosen the filters only after paper testing.
- **Every coin is "rejected by safety check: RugCheck danger"**: look at which risk names appear.
  You can ignore specific ones with `RUGCHECK_IGNORE_RISKS`, but understand what each means first.

## Files
| File | What it is |
|---|---|
| `bot.py` | the bot |
| `grok_prompt.txt` | the instructions sent to Grok with every coin |
| `.env.example` | settings template → copy to `.env` |
| `requirements.txt` | Python packages |
