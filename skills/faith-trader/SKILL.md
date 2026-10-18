---
name: "faith-trader"
description: "Manage a trader's Robinhood stock positions with J. Welles Wilder's Parabolic SAR driven by one per-symbol setting, faith (0 to 100), which flattens the curve, widens the stop, scales what a stop sells and sets a recurring faith buy; crypto is buy-only, with faith setting only the size of each buy. Places orders through the Robinhood connector or advises only. Use this skill whenever the user mentions Faith Trader, 'run faith', a faith check, a trading check or trading run, Wilder or Parabolic SAR stops on Robinhood, wants to set or change their faith in a stock or coin or their trading mode, wants to slowly accumulate a stock or crypto they believe in, asks whether to sell or buy more of something they hold on Robinhood, wants to drop a symbol or resume after a halt, wants a daily trading check scheduled, wants to set up, connect or check Robinhood for trading, or is moving over from Wild Trader. Replaces wild-trader. Trigger even when the user does not name the skill."
---

# Faith Trader

Faith Trader runs J. Welles Wilder's Parabolic Time/Price System (from *New Concepts in Technical Trading Systems*, 1978) over the stocks in a trader's agent-enabled Robinhood account, with one setting per stock: **faith**, an integer from 0 to 100. There is no risk level and no volatility setting. Faith does four things at once: it flattens the parabola, it gives the stock more room below its high before a sell, it shrinks how much is sold when that stop is hit, and it sets how much is bought on every run. Faith 0 is Wilder's system untouched and sells the whole position at his stop. Faith 100 never sells and buys 1 percent of the account on every run. Crypto held in the same account is buy-only: a coin's faith sets the size of its recurring buy and the skill never sells it (see "Crypto: buy-only").

Each run first checks that Robinhood and Claude Docs are ready, guiding the trader through setup when Robinhood is not (see "Initialization"). It then computes every managed symbol's parabola from completed daily bars, takes the exits the system calls for, places the recurring faith buys, and records everything in three Claude Docs. The only other setting is the **mode**: `advise-only` (the default for a new trader) tells the trader what to do, `trade` places the orders. In trade mode nothing is asked per order: exits go out because they only reduce risk, and faith buys go out because setting a faith value is the trader's standing instruction to make them.

The skill never has a buy and a sell working on the same symbol. When the system's sell and the faith buy land on the same run, they are netted into one order for the difference. It never uses margin and never counts money already reserved for queued orders (buys are capped at the lower of cash and buying power), never touches an order the trader placed themselves, and stops rather than guesses when something it needs is missing. Robinhood does not allow short selling, so Wilder's "stop and reverse" becomes "stop, sell the faith-scaled part, and watch for the bullish flip."

Faith Trader replaces Wild Trader but takes nothing from it. Every trader is onboarded fresh on the first run (see "First run").

## Requirements

- Robinhood's own trading connector, the server at `agent.robinhood.com`. Its tools are named after the trader's connection, so never assume a prefix: `mcp__Robinhood__*` in the Claude app, and in Claude Code most likely `mcp__robinhood-trading__*` or `mcp__claude_ai_Robinhood__*`. This file calls them by their bare names (`get_accounts` and so on). When the connector is not ready, follow "Initialization": an attended run guides the trader through setup.
- Claude Docs (`batch`, `guide` and `update`, plus `read` and `create` loaded through tool search; `mcp__Claude_Docs__*` in the Claude app). It is checked in "Initialization" too. Never fall back to memory or local files, since the settings must follow the trader across sessions.
- The Artifact tool's `list` action, used to find the trader's Faith Trader docs (docs are artifacts).
- A shell with Python 3 for the calculator at the end of this file. Without one, compute by hand from the formulas in that section and say in the report that the numbers were hand-computed.
- Scheduled runs use the scheduled-task tools (`create_trigger` and friends), never in-session cron tools.

## Initialization: check the connections first

Every run starts here (run step 0), and so do the "Set up Robinhood" and "Schedule" commands. Initialization reads and never changes anything: no order, no cancel, no doc write. When everything is ready it says nothing and the run goes on. The `get_accounts` result and the docs guide loaded here are reused by steps 1 and 2, so a ready run makes no extra call.

Claude cannot add a connector to a trader's account and cannot sign in to Robinhood for them. Setting up means finding out exactly what is missing, putting the connect card or the exact steps in front of the trader, waiting, checking once, and carrying on.

1. **Decide attended or unattended** (see "Unattended runs"). An unattended run never asks anything here.
2. **Find the Robinhood tools.** Look for a connection whose name contains `robinhood` (any case) and that offers `get_accounts`, `get_portfolio`, `get_equity_positions`, `get_equity_historicals`, `get_equity_quotes`, `review_equity_order` and `place_equity_order`. Connector tools are usually deferred, so look for them with tool search before deciding they are absent.
3. **Probe.** Call `get_accounts`. An account whose `agentic_allowed` is true is one this agent may trade. Never show that field name to the trader.
4. **Check Claude Docs.** Its tools must be present (search for them too) and `guide(items=["topic.index","topic.editing"])` must answer.
5. **Name the state** from the table and act on it. Check both Robinhood and Claude Docs before asking the trader for anything, and report everything that is missing in one message.

| State | How to tell | Attended run |
| --- | --- | --- |
| Ready | The tools are found and `get_accounts` returns an account the agent may trade | Go on to step 1 without comment |
| Off for this chat | No tools. The connector list shows Robinhood with `connected` true and `enabledInChat` false. Claude Code: `claude mcp list` shows it disabled for this project | Ask the trader to turn it on: the + button at the lower left of the message box, then Connectors, then Robinhood (Claude Code: `/mcp`, then enable it). Say that a new chat may be needed afterward. Then check once |
| On, but not in this conversation | No tools, although the connector list shows `connected` true and `enabledInChat` true. A chat looks like this for a while after the connector was turned on while it was open | Say that Robinhood is connected but this conversation cannot see it yet. The quick way on is a new chat (Claude Code: a new session) and "run faith" there. The tools can also come back here on their own after some minutes, so the trader may instead ask again later in this chat |
| Sign-in lapsed | The connector list shows Robinhood with `connected` false. Claude Code: "Needs authentication" | Ask the trader to connect again: Customize, then Connectors, then Connect on Robinhood (Claude Code: `/mcp`, select the server, authenticate). Then check once |
| Not installed | No tools, and Robinhood is not in the connector list. Claude Code: not in `claude mcp list` | Follow "Setting up Robinhood" |
| Cannot connect | The connection is there, but the call fails for a reason none of the rows above explains. Claude Code: "Failed to connect" | Give the error as it came, suggest Robinhood's own remedy (disconnect the connector and connect it again), check once, and stop if it still fails |
| No account for the agent | `get_accounts` answers, but no account is one the agent may trade | Follow "An account for the agent" |
| Unknown | No connector list is available, or it gives no status (`connected` is null) | Ask the trader whether Robinhood is connected in this app, then follow the matching row |

**Never read the state from an error message.** In the Claude app a call to a connector that is only turned off for the chat fails with `MCP server "Robinhood" needs authentication`, although the sign-in is fine. The connector list decides: `connected` false is a lapsed sign-in, `enabledInChat` false is off for this chat, and both true with no tools means this conversation has not caught up yet: a new chat, or a later try here.

**Where to look.** In the Claude app the connector list is a tool (`ListConnectors`), and the directory is searched with `SearchMcpRegistry` and shown to the trader with `SuggestConnectors`. Names can differ between apps, so find them with tool search. Use `claude mcp list` and `claude mcp get <name>` only in Claude Code, which means no connector list tool exists and the shell is the trader's own computer. In a cloud session of the Claude app that command reports no servers even when Robinhood is connected, and a server added there would vanish with the session.

**Robinhood's own server only.** When the app shows the connection's address (a directory entry sometimes does, and `claude mcp get <name>` does), it must be on `agent.robinhood.com`. When no address is shown, the tool set in step 2 is the test. A connection called Robinhood at another address, or without those tools, is not Robinhood's own server: do not use it, say so, and offer to set up the official one. Never suggest any other Robinhood MCP server. If two connections qualify, use the one that answered `get_accounts` for the whole run and never mix them.

**After the trader says a step is done,** check once: search for the tools again, then call `get_accounts`. If the tools are still not visible, the state is "On, but not in this conversation". Do not keep re-checking in the same reply; check again when the trader asks again.

**An unattended run** in any state but Ready stops here. It asks nothing, reads nothing from the account, touches no doc, and ends with the "not started" report (see "Report format").

**When setup ends in Ready** in this conversation, carry on with the run: first-run onboarding for a new trader, a normal run otherwise. The "Set up Robinhood" and "Schedule" commands do not start a run.

### Setting up Robinhood

Before the trader connects, say once what connecting grants, in Robinhood's own terms: the agent can read all of the trader's Robinhood accounts, positions, balances and order history, and it can place trades only in the account Robinhood opens for the agent. Say also that this is a custom connector to Robinhood's own server at `agent.robinhood.com`, which Anthropic does not verify. Never ask for a Robinhood username, password or code in chat: the trader signs in through Robinhood's own window.

Claude app (web or desktop):

1. Search the connector directory for Robinhood. If Robinhood's own connector is listed, show its connect card and go to step 4.
2. Otherwise ask the trader to open Customize, then Connectors, select "+ Add", then "Add custom connector". Robinhood's help page words the same path as Settings, then Connectors, then Add custom connector; give both.
3. Name it `Robinhood`, paste `https://agent.robinhood.com/mcp/trading`, continue, and choose to sign in now.
4. Sign in to Robinhood in the window that opens and follow Robinhood's onboarding, which opens the account for the agent.
5. Check that Robinhood is on for the chat (the + button, then Connectors), then tell Claude it is done.

On a Team or Enterprise plan only an owner can add a custom connector: tell a member to ask their owner, and then to select Connect on it under Customize, then Connectors. On the free plan a trader can have one custom connector; say that Robinhood will use it.

Claude Code:

1. Show this command and run it only after a yes: `claude mcp add --transport http --scope user robinhood-trading https://agent.robinhood.com/mcp/trading`
2. Ask the trader to type `/mcp`, select `robinhood-trading` and authenticate in the browser. Claude cannot do this part.
3. Robinhood's onboarding opens the account for the agent.
4. Check once. If the tools are not visible, ask the trader to restart Claude Code and say "run faith".

A trader who already has Robinhood connected in the Claude app, and who is signed in to Claude Code with the same Claude account, needs no setup in Claude Code: connectors from the account are carried over, though not under an API key.

When a menu does not match these steps, point the trader to Robinhood's own instructions: https://robinhood.com/us/en/support/articles/onboarding-an-external-agent/

### An account for the agent

Robinhood lets the agent trade in one account, opened for the agent during onboarding. When no account is tradable, say: "Robinhood is connected, but it has no account this agent can trade in." Do not say that an account needs to be enabled, because an account can be open to a different agent. Then give the steps:

1. Robinhood's onboarding opens by itself after the agent is authenticated. If it was closed early, disconnect the connector and connect it again to bring it back.
2. Robinhood's conditions: a primary individual investing account in good standing, and at most 10 self-directed individual accounts including this one.
3. Move money into the new account. Faith buys spend only that account's cash, so an empty account gives "no spendable cash" on every run.
4. When the trader says it is done, call `get_accounts` again.

### When Claude Docs is missing

There is nothing to set up: Claude Docs is part of the Claude app, not a connector the trader adds. Say that Claude Docs is not available in this session and that the skill keeps settings nowhere else. Suggest a new chat in the Claude app, or in Claude Code signing in with the Claude account, since account connectors are not loaded under an API key. Then stop.

### The setup check

When the trader asks to set up, connect or check Robinhood, run this section attended and stop. Report one line per requirement and finish any setup that is needed. Place nothing and write nothing.

```
Faith Trader setup check - 2026-10-11
Robinhood: connected and on in this chat. Trading account: Agentic ••••1234.
Claude Docs: available.
Ready. Say "run faith" to start a run.
```

## Faith and what it drives

Let f = faith / 100. Everything below is implemented by the calculator; this section is what to tell a trader who asks why.

- **Acceleration factor.** Maximum AF = 0.20 x (1 - f) and step = maximum / 10. Faith 0 gives Wilder's own 0.02 step and 0.20 maximum. More faith flattens the curve, and at faith 100 the SAR never moves. A leg starts with AF = one step and adds a step on each new extreme.
- **Wilder's SAR, long side.** SAR(next) = SAR + AF x (EP - SAR), where EP is the highest high of the leg, never above today's or yesterday's low.
- **Freeze.** The SAR is never lowered. While the price sits under Wilder's SAR but above the stop below, the long stays open and the SAR holds its level, so a slow slide still reaches the stop. At faith 0 this never applies.
- **The stop** used for selling is `EP - (EP - SAR) / (1 - f)`. Faith multiplies the room between the leg's high and Wilder's SAR by 1 / (1 - f): faith 50 doubles it, faith 75 makes it four times, faith 90 ten times. At faith 100, and whenever the formula gives zero or less, there is no stop and the system cannot sell the symbol. The stop never moves down within a leg; without that rule a new high would loosen it, because the high jumps before the SAR catches up.
- **What a stop-out sells:** shares held x (1 - f). Faith 0 sells everything, faith 50 half, faith 97 three percent. The rest is held through the bearish leg, and nothing more is sold until a later long leg hits its own stop.
- **Bearish side** (tracking only): Wilder's bearish parabola, unscaled, starting from the ended long's EP and using the same faith-scaled AF. When a completed bar's high touches the bearish SAR the system is long again, with SAR = the bearish leg's lowest low. Because the curve flattens with faith, a high-faith symbol that does get stopped out needs the price back near its old high before the flip.
- **Faith 100 is always long.** A bearish symbol whose faith is set to 100 flips to long at once.
- **Faith buy.** On every run, each symbol with faith above zero whose parabola is long gets a buy of 1 percent of account value x f (faith 100 buys 1 percent of the account, faith 50 half a percent). It is a dollar-based market order for fractional shares, capped by spendable cash (the lower of cash and buying power, so money reserved for queued orders is never counted), never under the broker's 1 USD minimum, at most one per symbol per trading session, and the symbol does not have to be held already. While the parabola is bearish the faith buy is paused; it resumes on the run that sees the bullish flip in a completed bar. When spendable cash cannot cover every faith buy, the highest faith is served first, stocks and coins alike.
- **Netting.** On the run that takes an exit, that session's faith buy (if not already placed and not blocked) is netted against the system's sell. Sell value = shares x (1 - f) x bid. If the sell is larger by 1 USD or more, sell the difference in shares; if the buy is larger by 1 USD or more, buy the difference in dollars; within 1 USD, no order. Either way the exit counts as taken, the faith buy counts as done, and the parabola is bearish from that run.

| Faith | AF step / maximum | Room | Stop when EP is 100.00 and SAR 92.00 | Sold at a stop | Faith buy per run |
| --- | --- | --- | --- | --- | --- |
| 0 | 0.02 / 0.20 | x1 | 92.00 | 100% of shares | none |
| 25 | 0.015 / 0.15 | x1.33 | 89.33 | 75% | 0.25% of account |
| 50 | 0.01 / 0.10 | x2 | 84.00 | 50% | 0.50% |
| 75 | 0.005 / 0.05 | x4 | 68.00 | 25% | 0.75% |
| 90 | 0.002 / 0.02 | x10 | 20.00 | 10% | 0.90% |
| 97 | 0.0006 / 0.006 | x33.33 | no stop | 3% | 0.97% |
| 100 | 0 / 0 | no stop | no stop | nothing | 1.00% |

Resting stop orders exist only at faith 0. A symbol with faith 1 to 99 has a faith buy on every run while it is long, and a buy and a resting sell never sit together, so its stop is checked when a run happens and the exit is placed then. With one run before the open each day, a stop touched during the session is acted on the next morning. Say this plainly whenever a trader asks how protected a position is.

A symbol is managed only if it has a row in the Faith table. A stock or coin with no faith value is never acted on, because a value cannot be assumed: faith 0 would mean selling all of a stock at Wilder's stop. The skill asks instead. Every attended run asks for the missing values before it does anything else, and an unattended run lists them under "Needs you" (run step 3). A row whose faith is `off` is one the trader told the skill to ignore, and it is not asked about again.

## What a run does with each symbol

The calculator returns one `signal` per symbol:

| Signal | Meaning | Faith buy | Sell |
| --- | --- | --- | --- |
| `hold` | Long and held, price above the stop (or no stop at this faith) | Yes | No. At faith 0 in trade mode, refresh the resting stop |
| `accumulate` | Long, not held, faith above zero | Yes, it starts the position | No |
| `sell` | The long ended on this run: a newly completed bar touched the stop, or the live price is at or under it | Netted | (1 - f) of the shares, netted into `net_order` |
| `out` | Bearish. Any shares still held stay held on faith | Paused | No |
| `flip_pending` | Bearish, but the live price is back above the bearish SAR; today's completed bar confirms it on the next run | Paused | No |
| `resumed` | A completed bar flipped the system back to long | Yes | No |
| `watch` | Faith 0 and not held (or a coin at faith 0): nothing to do but report | No | No |
| `buy_only` | A coin with faith above zero. There is no parabola and no stop for it | Yes | Never |

A sell fires once per stop-out, on the run that first sees the long end. If one run's bars contain a stop-out and a later flip back to long, nothing is sold and both are reported. When the exit comes from the live price before the day's bar is complete, the calculator flips the stored state to bearish at once (bearish SAR = the long's EP, EP = the live price) and the completed bar is applied to it on the next run.

The system can stay long, and keep buying, with the price well under Wilder's own SAR (`under_wilder_sar`). That is the room faith grants; report it as a fact, not a problem.

## Crypto: buy-only

The Robinhood connector gives a live quote for crypto but no price history, so no parabola and no stop can be computed for a coin. A coin with a faith value is therefore handled as buy-only.

- **Faith buy only.** On every run a coin with faith above zero gets the same faith buy as a stock: 1 percent of account value x f, as a dollar-based market order, capped by spendable cash and by crypto buying power, never under 1 USD, at most one per coin per Eastern calendar day. Crypto trades around the clock, so the order fills when it is placed, weekends included.
- **Never sold.** A coin has no stop, no exit, no netting and no resting order. The skill never sells a coin and never cancels a crypto order. Faith 0 lists the coin without buying it, and `off` ignores it. The circuit breaker blocks coin buys like any other buy; the day-trade guard does not apply to crypto.
- **Symbols.** A coin is stored and shown by its pair symbol, `currency.code` plus `-USD` (`BTC-USD`), never by the bare code, which can collide with a stock ticker (`BTC` is also an ETF). When the trader names a bare code that could be either, take the one they hold and say which it was; if that does not settle it, ask.
- **Quantities** are stated in units of the coin ("0.00742184 BTC"), never as shares.
- **Account.** Crypto tools take the account's `rhs_account_number` from `get_accounts`, not `account_number` and never `rhc_account_number`. If the account has no crypto access, or a crypto call fails, carry on with the stocks and say so in the report.
- **Fees.** Robinhood can charge a fee on a crypto order. `preview_crypto_order` shows it; report it with the buy.
- **State.** A coin has a Faith row and a Faith buys row. It has no Symbols row and no Exits row, because there is no parabola state to keep.

**The advice that goes with every coin's faith value.** Whenever a coin's faith is asked for or set (onboarding, the missing-faith question in run step 3, or "set faith"), say all three of these, every time, before taking or confirming the value:

1. Crypto is buy-only here: Robinhood gives the skill no crypto price history, so a coin has no Wilder curve and no stop.
2. The skill will never sell the coin. Faith only sets how much of it is bought on each run.
3. Because nothing is ever sold, choose a smaller faith value than you would give a stock.

Then put the value in money: the buy per run at today's account value and what 30 runs add up to. For example: "BTC-USD is buy-only. Faith Trader has no price history for crypto, so it never sells a coin and faith only sets the size of each buy. Because nothing is ever sold, I recommend a smaller faith value than you would give a stock. On this account faith 10 buys about 5.00 USD per run, about 150 USD over 30 runs; faith 50 buys about 25.00 USD per run." The value the trader then gives is the one used; give the advice once per question and do not argue it.

## Persistence: three Claude Docs per trader

Each trader has three docs, found by exact title. Load the docs guide once per run before touching them: `guide(items=["topic.index","topic.editing"])`. Find them with `Artifact(action="list", type="Docs", limit=50)`; if that call rejects `type`, use `Artifact(action="list", scope="mine", limit=50)`. The doc id is the trailing id of each URL. Read a doc with `read(ref={"object":"project","id":"<doc id>"})` to get its body id (`files[0].content.id`), then `read(ref={"object":"node","id":"<body id>"}, engine="prose", container={"kind":"project","id":"<doc id>"})` for the blocks, their ids, their `h` hashes and the `rev`.

Rewrite a table with a guarded `replace` of the whole table block: target the table's full block id, send `ifHash` (the table's `h`) and `ifRev` (the `rev` of that read), `"as":"markdown"`, and re-send every row. The guard is what stops a concurrent edit from being silently overwritten; if it is refused, re-read and merge rather than forcing. An empty table keeps a single row reading `none`.

**Faith Trader Settings** holds the trader's choices. The last four Faith columns are derived from the faith value and rewritten with it. A coin's row reads `buy-only`, `no stop` and `never sold`, followed by its faith buy.

```markdown
# Faith Trader Settings

<?claude block asof?> · <?claude block me?>

Faith is the only setting per stock or coin. Change a value here or tell Claude to change it.

| Setting | Value |
| --- | --- |
| Mode | advise-only |
| Updated | 2026-10-05 |

## Faith

Faith 0 is Wilder's system untouched. More faith flattens the curve, widens the stop, sells less at a stop and buys more on every run. Faith 100 never sells. Crypto is buy-only: a coin's faith sets the size of each buy and the coin is never sold. A stock or coin that is not listed is not managed.

| Symbol | Faith | AF step / max | Room | Sold at a stop | Faith buy per run |
| --- | --- | --- | --- | --- | --- |
| NVDA | 100 | 0 / 0 | no stop | nothing | 1.00% of account |
| TSLA | 97 | 0.0006 / 0.006 | x33.33 | 3% | 0.97% of account |
| BTC-USD | 10 | buy-only | no stop | never sold | 0.10% of account |
```

**Faith Trader State** is machine-maintained working memory. Accounts are shown by their last four digits only (the full number comes from `get_accounts` each run). Rewrite each table whole after every run.

```markdown
# Faith Trader State

<?claude block asof?> · <?claude block me?>

Maintained by Faith Trader on every run. Do not edit by hand; tell Claude to drop a symbol or to resume instead.

## Symbols

| Symbol | Account | Held | Avg cost | Faith | Side | SAR | AF | EP | Stop | Since | Last bar |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |

## Skill orders

| Symbol | Account | Order id | Kind | Stop | Qty | Placed | Status |
| --- | --- | --- | --- | --- | --- | --- | --- |

## Exits

| Symbol | Account | Signal date | Stop | System sell | Faith buy netted | Net order | Order id | Status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |

## Faith buys

| Symbol | Account | Last buy date | Amount | Order id | Status |
| --- | --- | --- | --- | --- | --- |

## Circuit breaker

| Item | Value |
| --- | --- |
| Status | active |
| Consecutive losing exits | 0 |
| Halted on | none |
| Reason | none |

## Account value history

| Date | Account | Total value |
| --- | --- | --- |
```

The Symbols row is the calculator's `state` written back exactly as returned (SAR, EP and Stop to four decimals, AF to six, Stop `none` when there is none), because the next run feeds it in again and rounding it would stall a slow curve. Side is `long` or `short` (the side the system is on, whether or not shares are held). Faith is the value the stored state was computed with, which is how the next run notices a change. Avg cost is the position's `average_buy_price`, kept so an exit can be judged a win or a loss after the shares are gone. When the calculator returns `leg_predates_backfill`, write Since as `before <date>`.

The Exits table keeps one row per symbol for its latest stop-out: Status is `advised` (advise-only), `open`, `filled`, `no order` (netted to nothing), `blocked` (a review alert stopped it) or `reported only` (an exit that predates the skill). The Faith buys table keeps the last faith buy per symbol; Last buy date is the trading session the order fills in (today while the market is open or before the open, otherwise the next trading day), which is what keeps a weekend run and a Monday run from both buying. For a coin it is the Eastern calendar date of the buy, since crypto fills at any hour. Status is `open`, `filled`, `closed` or `netted`.

**Faith Trader Log** is append-only, newest first. Insert each entry with `insert` after the `## Entries` heading block (`"side":"after"`, `"as":"markdown"`).

```markdown
# Faith Trader Log

<?claude block asof?> · <?claude block me?>

Every run, signal, order and fill, newest first.

## Entries
```

Create a missing doc with one `batch` call: `container = {"kind":"project","create":{"name":"Faith Trader Settings","doc":{"blocks":{"asof":{"type":"date","value":"<today>"},"me":{"type":"mention","user":"me"}},"markdown":"<template>"}}}` and `batch = []`. Open it afterward with `Artifact(action="open", url=<link from the ack>)` when the trader is present.

### First run: onboard

If Faith Trader Settings does not exist and the trader is present, onboard. Nothing is imported: the docs of any other skill, Wild Trader's included, are not read or copied.

1. From the `get_accounts` result of run step 0, take the first account this agent may trade, in the order the connector lists them. Step 0 has already made sure there is one.
2. Call `get_portfolio` for that account and show its buying power (`buying_power.buying_power`, in `buying_power.display_currency`), with the account's nickname when it has one and its number masked to the last four digits. For example: "Agentic ••••1234: buying power 3,534.34 USD." This is for the trader's information. The skill's own buys are capped at the lower of `cash` and that buying power, so when the two figures differ, show both and say in a few words that faith buys can spend only the lower one.
3. Explain faith in three sentences, call `get_equity_positions` and `get_crypto_positions`, and ask in one numbered message for the mode (advise-only or trade) and a faith value (0 to 100, or `off` to leave it alone) for each held stock, each stock with a buy order of the trader's own waiting, and each held coin. Give the crypto advice from "Crypto: buy-only" with every coin.
4. Create the three docs, write the first log entry with the buying power that was shown, and continue the run.

If the trader still has the wild-trader skill, say once that it should be removed, because two skills buying the same stocks on every run would double each faith buy. Only the trader can remove a skill.

On every later attended run, any stock or coin that still has no faith value is asked about before anything else is done (run step 3).

### Unattended runs

A run is unattended when it was started by a scheduled task or the trader has said they will not be around. In an unattended run: never ask questions; if Robinhood or Claude Docs is not ready, stop at step 0 with the "not started" report; if a Faith Trader doc is missing, stop and report that setup has not been done (onboarding needs the trader); do not ask for missing faith values, but leave each stock or coin without one untouched and list it under "Needs you"; finish with the run report as the final output. Exits and faith buys still go out in trade mode, since neither needs a decision.

## Run procedure

Follow these steps in order on every run. Steps 3 through 11 repeat per agent-enabled account.

0. **Initialize.** Follow "Initialization: check the connections first": decide attended or unattended, find the Robinhood tools, call `get_accounts`, check Claude Docs, and set up or stop when something is not ready. Go on only from Ready.

1. **Docs.** Attended or unattended was decided in step 0, and the docs guide was loaded there. Find and read the three docs. Onboard or stop as described above. If State lacks a table or column this file describes, add it silently.

2. **Accounts.** Use the `get_accounts` result from step 0; do not call it again. Work only with accounts this agent may trade (`agentic_allowed` is true; the nickname does not matter). Step 0 has already dealt with the case where there is none.
   - **Different account.** If Faith Trader State already has rows and none of them carries the last four digits of an account this agent may trade, this connection trades a different account than the one the docs were built on. A second connection, or a connector that was removed and added again, can be given an account of its own. Attended: show both accounts by their last four digits and ask whether to go on with this one before doing anything else. Unattended: stop and list it under "Needs you".

3. **Portfolio, positions and missing faith.**
   - Call `get_portfolio`. `total_value` is the account value, crypto included. Faith buys may spend at most the lower of `cash` and `buying_power.buying_power`: `cash` on its own still counts money reserved for queued orders, and buying power on its own can include margin. A coin's buy is also held to `crypto_buying_power.buying_power` when the connector returns it. Hand all three figures to the calculator in step 7, which applies the cap; never work it out by hand.
   - Call `get_equity_positions` (`quantity` is the position, `shares_available_for_sells` is what can be sold right now, `average_buy_price` is the cost basis) and `get_crypto_positions` with the `rhs_account_number` (`quantity` is in units of the coin; the pair symbol is `currency.code` plus `-USD`).
   - Append today's `total_value` to the account value history if today has no row; keep the last 15 rows per account.
   - **Missing faith.** List every symbol that has no row in the Faith table: each held stock, each held coin, and each stock that `get_equity_positions` returns with quantity 0 while a buy order of the trader's own is waiting (confirm with `get_equity_orders(account_number, symbol=...)`: an open buy is one in state `new`, `queued`, `confirmed`, `unconfirmed` or `partially_filled`; a quantity 0 row with no open buy is a stock that was sold, so skip it). A row set to `off` is not missing. In an attended run, ask for all of them in one numbered message before doing anything else, and wait for the answer. Say what the choices are (0 to 100; 0 puts a whole stock position under Wilder's stop; `off` leaves the symbol alone for good; "skip" decides later), say that in trade mode this run starts acting on the values given, and give the crypto advice from "Crypto: buy-only" with every coin. Write each answer to the Faith table with its derived columns, log it, and carry on with the run using the new values. A skipped symbol is left untouched and listed under "Needs you". In an unattended run, do not ask: leave them untouched and list each under "Needs you".

4. **Reconcile.**
   - Skill orders with Status `open`: call `get_equity_orders(account_number, order_id=...)`. `filled`: a faith 0 stop was hit; log the fill, write an Exits row, and count it as a losing exit if `average_price` is under the stored Avg cost (add one to Consecutive losing exits) or reset the count to 0 otherwise. The bar that touched the stop flips the parabola in step 7, and any fractional share left over comes back as a `sell` signal there. `cancelled`, `rejected`, `failed`, `voided` or expired: mark the row `closed`. Still open: leave it for step 9.
   - Faith buys and Exits with Status `open`: mark them `filled` or `closed` from the order state and log the fill. A coin's faith buy is looked up with `get_crypto_orders(rhs_account_number, order_id=...)`; crypto spells the cancelled state `canceled`.
   - Positions against State: a symbol that left the positions without a skill fill was sold by the trader; set Held `no` and keep the row. A managed symbol with faith above zero keeps accumulating when it is long, so say that its faith buys continue until faith is set to 0 or the symbol is dropped. A stock in the Faith table with no Symbols row is new and is backfilled in step 7, whether or not it is held. A coin never gets a Symbols row.
   - Day-trade guard: note every symbol sold in this account today (a skill fill, or a filled sell in `get_equity_orders` with `created_at_gte` today). Unless `total_value` is 25,000 USD or more, those symbols get no buy today.

5. **Market data.** Call `get_equity_historicals` with `interval: "day"`, up to 10 symbols per call. A new symbol needs about 375 calendar days back so that 250 completed bars are available (fewer is fine for a younger stock; five is the minimum). A symbol already in State needs only the bars from a week before its Last bar. Call `get_equity_quotes` for every symbol: the live price is `last_trade_price`, or `last_non_reg_trade_price` if its timestamp is newer; use `bid_price` and `ask_price`, dropping either when it is zero. If the most recent trading day's bar is missing, proceed with what exists and say so in the report. Coins need no bars: call `get_crypto_quotes` with the pair symbols and the `rhs_account_number`, and use `mark_price` as the live price and `ask_price` as the ask.

6. **Circuit breaker.** With the history rows, compare today's `total_value` to the most recent value from a prior trading day (daily) and to the oldest value within the last five trading days (weekly). Halt when the daily drop is 5 percent or more, the weekly drop is 10 percent or more, or Consecutive losing exits reaches 3. Record Status `halted`, the date and the reason. A halt blocks buys only; sells always continue. A withdrawal can look like a drop, so name the cause when you can. The trader lifts a halt by saying resume.

7. **Compute.** Put the calculator below in a file once per run. When this SKILL.md is readable on disk, extract the code block with a short script rather than retyping it. Then run `python3 faith_trader_calc.py --selftest`, which must print `selftest ok`; anything else means the file was mistyped, so write it again before going on. Pipe one JSON object per account through it:

   ```json
   {"account": {"total_value": 5001.91, "cash": 3534.34, "buying_power": 3300.00, "crypto_buying_power": 3300.00},
    "today": "2026-10-05", "today_bar_complete": false, "buys_blocked": null,
    "symbols": [
     {"symbol": "TSLA", "faith": 97, "backfill": true, "bars": [["2025-10-06", 440.75, 453.55, 436.69, 453.25]],
      "quote": {"last": 371.42, "bid": 371.43, "ask": 371.80}, "shares_held": 1.0, "avg_cost": 375.54},
     {"symbol": "AMD", "faith": 50, "bars": [["2026-10-01", 160.2, 161.0, 156.9, 157.4], ["2026-10-02", 157.0, 159.3, 155.8, 158.1]],
      "state": {"side": "long", "sar": 152.6, "af": 0.04, "ep": 164.0, "stop": 141.2, "since": "2026-09-12", "last_bar": "2026-10-02", "faith": 50},
      "quote": {"last": 158.1, "bid": 158.0, "ask": 158.2}, "shares_held": 10, "avg_cost": 149.3,
      "faith_buy_done_today": false, "buys_blocked": null},
     {"symbol": "BTC-USD", "asset": "crypto", "faith": 10, "quote": {"last": 86650.0, "ask": 86700.0},
      "units_held": 0.00742184, "faith_buy_done_today": false}]}
   ```

   - `account` carries `total_value`, `cash`, `buying_power` (the portfolio's `buying_power.buying_power`) and `crypto_buying_power` (`crypto_buying_power.buying_power`; leave it out when the connector omits it). The calculator caps the faith buys at the lower of cash and buying power, highest faith first, and returns that starting figure as `spend_cap`.
   - The `bars` lists above are cut short; a backfill sends about 250 bars and a continuing symbol a week or so.
   - `today` is today's date in Eastern time, and `today_bar_complete` is true only on a trading day after 4:00 PM Eastern. The calculator drops interpolated bars and, unless that flag is true, today's unfinished bar, which would corrupt the parabola.
   - `bars` may be pasted as the connector returns them (`begins_at`, `open_price`, `high_price`, `low_price`, `close_price`) or written compactly as `[date, open, high, low, close]`, oldest first. Copy the prices exactly. If the result carries `bar_warnings`, a bar was mistyped: fix it and run again before acting.
   - A new symbol gets `"backfill": true`. Every other symbol gets `state` from its Symbols row, bars that start at or before its Last bar, and the current `faith` from the Faith table; when the two faith values differ the calculator applies the change (AF capped at the new maximum, stop recomputed from scratch).
   - `shares_held` is the position's `quantity`. `faith_buy_done_today` is true when the Faith buys row already carries this session's date (today on a trading day before 4:00 PM Eastern, otherwise the next trading day). Top-level `buys_blocked` carries the circuit-breaker reason when halted; a symbol's own `buys_blocked` carries the day-trade guard.
   - A coin gets `"asset": "crypto"`, its pair symbol, `faith`, `quote` (`last` is the `mark_price`, `ask` the `ask_price`), `units_held` and `faith_buy_done_today` (true when its Faith buys row carries today's Eastern date). It takes no bars and no state.

   Read back, per symbol: `state`, `signal`, `stop`, `wilder_sar`, `flips`, `faith_buy`, and when present `sell`, `net_order`, `resting_stop`, `preexisting_exit`, `under_wilder_sar`, `faith_changed`, `leg_predates_backfill` and `error`. A symbol that returns `error` is reported and skipped; the rest of the run goes on. A coin returns only `signal` (`buy_only`, or `watch` at faith 0) and `faith_buy`.

8. **Exits** (`signal` is `sell`). The calculator gives `sell` (the system's quantity and value) and `net_order` (the one order to place after netting the faith buy).
   - Trade mode: at faith 0, first cancel the symbol's open skill stop so its shares are free. Then `review_equity_order` and `place_equity_order` the `net_order`: a `sell` is `type` market with `quantity` (fractional shares allowed), a `buy` is `type` market with `dollar_amount`, both `regular_hours` and `gfd`; `none` places nothing. Write the Exits row. When a faith buy was netted (`faith_buy.netted`), also write the Faith buys row for this session with Status `netted`, so step 10 does not buy again.
   - Advise-only: place nothing. State the exit, the netting and the exact order, and write the Exits row with Status `advised`.
   - Count the exit once: `sell.losing_exit` true adds one to Consecutive losing exits, false resets it to 0. Skip this when step 4 already counted the same stop-out from a filled resting stop.
   - `preexisting_exit` (a first backfill that ends bearish while shares are held): the exit happened before the skill managed the symbol. Place nothing, write the Exits row with Status `reported only`, and list it under "Needs you" with the quantity it would sell.

9. **Resting stops.** Only for `resting_stop` (faith 0, long, held).
   - Trade mode: cancel the symbol's open skill order if today's is still open (only ids recorded in Skill orders; yesterday's day order has already expired), then review and place a `sell`, `stop_market`, `gfd`, `regular_hours` order for `whole_shares` at `stop_price`, and record it as `open`. Under one whole share no stop is possible; say so and give the level. Trade mode is the trader's standing confirmation for replacing the skill's own stop.
   - Any symbol with faith above zero that still has an open skill stop (left from when its faith was 0): cancel it and mark the row `closed`.
   - Advise-only: give the stop level and the order the trader would place, and offer a Robinhood price alert at the stop (`create_alert`, `price_below`); create it only on a yes in this conversation.

10. **Faith buys.** For every symbol whose `faith_buy.amount_usd` is above zero and not netted: in trade mode, review and place a `buy`, `market`, `regular_hours`, `gfd` order with `dollar_amount` (no `quantity`) and a fresh UUID `ref_id`, then write the Faith buys row with the session date and Status `open`. No confirmation is asked, and the report says the buy was placed on the standing faith instruction. In advise-only mode, state the amount and the order. A `skip_reason` (parabola bearish, already bought, buys blocked, no spendable cash, under the 1 USD minimum) is reported in a few words. When a buy is skipped for no spendable cash, or comes out smaller than `wanted_usd`, while cash is above buying power, say that the rest of the cash is reserved for queued orders. An order placed while the market is closed queues for the next open.
    - A coin's faith buy uses the crypto tools: in trade mode call `preview_crypto_order` and then `place_crypto_order` with the `rhs_account_number`, the pair `symbol`, `side` buy, `type` market, `dollar_amount` and a fresh UUID `ref_id` (no `quantity` and no `time_in_force`). It fills at once at any hour, so write the Faith buys row with today's Eastern date and the state the order came back with, and report the fee the preview showed.

11. **Persist.** Rewrite the Symbols, Skill orders, Exits, Faith buys, Circuit breaker and Account value history tables from the calculator output and the orders placed. Update the `Updated` row in Settings only when a setting changed. Insert one log entry for the run.

12. **Report** in the format below. In chat, mask account numbers to their last four digits.

## Order rules (apply to every order)

- Always `review_equity_order` before placing, and read its alerts; a buying-power, halt or pattern-day-trader alert means stop, mark the row `blocked`, and tell the trader rather than pushing through. For a coin the same step is `preview_crypto_order`, and a validation error there is treated the same way.
- `place_equity_order` and `place_crypto_order` get a fresh UUID `ref_id` per logical order, re-sent unchanged only on a transport retry.
- Stock orders are `regular_hours` and `gfd`. Market and stop orders cannot run in other sessions; placed while the market is closed they queue for the next open, which is what a pre-open run wants.
- Fractional and dollar-based orders are market orders only. Stop orders need whole shares.
- Round every price to cents. A sell stop must sit below the current price; if it does not, the calculator returns `sell` instead.
- Never cancel an order the skill did not place. The only cancellable ids are those in Skill orders.
- Buys are capped at the lower of `cash` and buying power, so money reserved for queued orders is never counted and margin is never used. The calculator already shrinks each faith buy to what is left after the higher-faith ones; never substitute a different amount.
- Crypto is buy-only. The system never sells a coin, never places a stop on one and never cancels a crypto order. State a coin's quantity in units of the coin, never as shares.
- One order per symbol per run. A buy and a sell on the same symbol in the same run means the netting was skipped; go back to `net_order`.
- If a trader asks to sell, buy or move a stop outside the system, do it through the same review-then-place path and log it as a manual instruction, saying plainly that it departs from the system. Selling a coin is always such an instruction, because the system never does it.

## Trader commands

- **Run** ("run faith", "faith check", "trading check", "check my positions"): the run procedure.
- **Set faith for SYMBOL to N**: validate an integer 0 to 100, rewrite the symbol's Faith row with its derived columns, log the change, and say what it means in one line each: the curve, the room, the share sold at a stop, and the faith buy per run. Before running, check the effect: if lowering faith puts the live price at or under the new stop, the next run sells, so show the trader that sale and get a yes before a trade-mode run places it. Then run. Setting faith to 0 stops the faith buys and puts the whole position under Wilder's stop. For a coin, give the crypto advice from "Crypto: buy-only" first, every time; then write its row as buy-only and state the buy per run in USD. A coin has no curve, room or stop to describe and no sale to check.
- **Set mode to trade / advise-only**: rewrite Settings and log it. Switching to trade mode should be followed by a run.
- **Take the exit on SYMBOL**: for an exit that was reported or advised and not placed. Run the calculator with `"take_exit": true` on that symbol, show the order, and place it through the usual path. No faith buy is netted, because the parabola is bearish.
- **Drop SYMBOL**: set its Faith to `off`, remove its Symbols row, cancel its open skill stop if any, and log it (a coin has neither). The skill no longer buys, sells or reports the symbol, and does not ask about it again.
- **Resume**: set Circuit breaker Status to `active`, reset Consecutive losing exits to 0, log it.
- **Status**: read the docs and summarize mode, faith values, sides and stops without placing anything.
- **Set up Robinhood** ("set up Robinhood", "connect Robinhood", "check my connection"): run "Initialization" on its own, attended, and stop without starting a run. See "The setup check".
- **Schedule**: first run "Initialization", and create no task until Robinhood and Claude Docs are ready. Then create a scheduled task with the scheduled-task tools. Suggest weekdays shortly before the 9:30 AM Eastern open, written in the trader's time zone with a `CRON_TZ=` prefix. Task prompt: "Run the faith-trader skill for my Robinhood account. This is an unattended scheduled run: follow the skill's unattended rules, do not ask questions, and finish with the run report." Tell the trader that the task's own approval setting decides whether its orders go through with nobody present, and in trade mode recommend they confirm that setting is what they want. Say also that a scheduled run starts in a fresh session and can use only what is connected at that moment; if Robinhood is not on then, the run reports "not started".

## Report format

Use this shape every run, one block per account. Keep it to what changed and what needs the trader. The figures here are invented to show the layout.

```
Faith Trader run - 2026-10-05 8:52 AM ET - trade mode
Account ••••1234: value 5,001.91 USD, cash 3,534.34 USD, buying power 3,300.00 USD (faith buys can spend 3,300.00 USD). Circuit breaker active.

NVDA   held 3.2145 sh | faith 100: never sold | long | faith buy 50.02 USD at market placed (1.00% of account)
TSLA   held 1 sh | faith 97: room x33.33, sells 3% | long, price under Wilder's SAR 426.07 | no stop at this faith | faith buy 48.52 USD placed
AMD    held 10 sh | faith 50: room x2, sells 50% | long since 09-12 | stop 141.20 (Wilder's SAR 152.60) | faith buy 25.01 USD placed
INTC   held 8 sh | faith 50 | STOP HIT 10-02, bid 30.10 | system sell 4 sh (120.40 USD) less faith buy 25.01 USD = sold 3.1691 sh at market
PLTR   held 2.5 sh | faith 60 | system OUT since 09-24, holding on faith | faith buys paused until the flip above 187.40
F      held 20 sh | faith 0 | long since 09-20 | stop 11.42 | stop-market sell 20 sh @ 11.42 placed
BTC-USD held 0.00742184 BTC | faith 10: buy-only, never sold, no stop | faith buy 5.00 USD at market placed (0.10% of account), fee 0.05 USD

Needs you:
1. SOFI is held but has no faith value. Give it one (0 to 100) or say "drop SOFI".
2. DOGE-USD is held but has no faith value. Crypto is buy-only and never sold, so choose a small value, or say "drop DOGE-USD".
Data note: 10-02 daily bar not yet posted; levels use bars through 10-01 and the live quote.
```

Say "Needs you:" with a numbered list when there is a missing faith value, an exit that was reported and not placed, a blocked order, or a different account than before, and "Needs you: nothing." when there is none. Show cash and buying power on the account line every run; when buying power is the lower figure, the difference is money reserved for queued orders or not yet settled, and faith buys cannot use it. In advise-only mode write "advised" where the example says "placed" or "sold". Never present a recommendation as a certainty; this is a mechanical system the trader chose, not a forecast. Write money as "1,000.65 USD" rather than with a leading dollar sign so the figures survive copying between tools.

A run that stops at step 0 uses this short form and nothing else. Word item 1 for the state that was found:

```
Faith Trader run - 2026-10-12 8:52 AM ET - not started
Needs you:
1. Robinhood is connected to your Claude account but is not on in this session. Open a chat and say "set up Robinhood".
Nothing was read from the account, no order was placed and no doc was changed.
```

An attended run that had to set something up adds one line to its report and to its log entry naming the state that was found and fixed. A run that was ready adds nothing.

## Wilder math and the calculator

Formulas the calculator implements, for hand computation or checking, with f = faith / 100:

- AF maximum = 0.20 x (1 - f), step = maximum / 10, both to six decimals.
- Long side: exit when a bar's low is at or under the stop in force (at the stop, or at the open on a gap under it). Otherwise raise EP and AF on a new high, then SAR(next) = max(SAR, min(SAR + AF x (EP - SAR), today's low, yesterday's low)).
- Stop = EP - (EP - SAR) / (1 - f); none at faith 100 or when the result is zero or less; equal to the SAR at faith 0. Stop in force = the highest stop so far in the leg.
- Reversal to bearish: SAR = the long's EP (not below the last two highs), EP = the bar's low, AF = one step, no acceleration that day. Bearish side: lower EP and raise AF on a new low, SAR(next) = max(SAR + AF x (EP - SAR), today's high, yesterday's high). Flip to long when a bar's high touches the SAR; the new long has SAR = the bearish EP (not above the last two lows), EP = the bar's high, AF = one step.
- A new symbol is backfilled over the last 250 completed daily bars, seeded long on the fifth bar with SAR = the lowest low and EP = the highest high of the first five. At faith 0 the seed washes out and the result is Wilder's current view; at high faith the symbol is simply long.
- Faith buy = account value x 1% x f, capped by the spendable cash left after higher-faith buys, skipped under 1 USD. Spendable cash = the lower of cash and buying power, never below zero. A coin's buy is also held to crypto buying power, and every buy draws both figures down.
- A coin has the faith buy and nothing else: no bars, no SAR, no stop and no sell.
- Exit: sell shares x (1 - f); net against the faith buy as described under "Netting".

Write this file as `faith_trader_calc.py` and run `python3 faith_trader_calc.py < input.json`.

```python
#!/usr/bin/env python3
"""Faith Trader calculator: JSON in on stdin, JSON out on stdout.

Input, one object per account:
  {"account": {"total_value", "cash", "buying_power", "crypto_buying_power"}, "today": "YYYY-MM-DD",
   "today_bar_complete": false, "buys_blocked": null or "reason", "symbols": [ <symbol>, ... ]}
A single symbol object with "account" inside it is also accepted. Faith buys spend at most the lower of cash and
buying_power (a coin: also at most crypto_buying_power), so money reserved for queued orders is never counted.

Stock: symbol, faith (0..100), bars, and either "backfill": true (a new symbol) or
  "state": {"side","sar","af","ep","stop","since","last_bar","faith"} (bars after last_bar are applied;
  the bars sent must include last_bar itself).
  Optional: quote {"last","bid","ask"}, shares_held, avg_cost, faith_buy_done_today, buys_blocked, take_exit.
Coin: "asset": "crypto", symbol (the pair, such as BTC-USD), faith, quote {"last","ask"}, units_held and
  faith_buy_done_today. A coin is buy-only: it takes no bars and no state, has no stop and is never sold.
Bars are daily, oldest first, as {"date","open","high","low","close"}, as the Robinhood connector returns them
({"begins_at","open_price","high_price","low_price","close_price","interpolated"?}), or as [date, open, high, low, close].
Interpolated bars are dropped, and so is any bar dated today or later unless today_bar_complete is true.

Output: {"results": [...], "spend_cap", "cash_after_buys"}. A stock result has params, state (write it back to the
Symbols row), flips, signal (hold, accumulate, sell, out, flip_pending, resumed, watch) and, when they apply,
faith_buy, sell, net_order, resting_stop, preexisting_exit, leg_predates_backfill, bar_warnings or error.
A coin result has only params, signal (buy_only, or watch at faith 0), held, live_price and faith_buy."""
import json, sys

AF_BASE_MAX = 0.20     # Wilder's maximum acceleration factor, used whole at faith 0
AF_STEPS = 10          # the step is one tenth of the maximum (Wilder's 0.02 against 0.20)
FAITH_BUY_PCT = 1.0    # percent of account value bought per run at faith 100, scaled by faith / 100
MIN_ORDER_USD = 1.0    # Robinhood's minimum for a dollar-based order; also the netting dead band
BACKFILL_BARS = 250    # a new symbol is backfilled over this many completed daily bars
SEED_BARS = 5          # the backfill is seeded long on the fifth bar


def r2(x):
    return None if x is None else round(x + 1e-9, 2)


def r4(x):
    return None if x is None else round(x, 4)


def af_params(faith):
    """Faith flattens the curve: maximum AF = 0.20 x (1 - faith / 100), step = maximum / 10."""
    af_max = round(AF_BASE_MAX * (1 - faith / 100.0), 6)
    return round(af_max / AF_STEPS, 6), af_max


def scaled_stop(sar, ep, faith):
    """The stop used for selling: Wilder's SAR pushed away from the leg's high (EP) so the room below the
    high is multiplied by 1 / (1 - faith / 100). None means no stop (faith 100, or the formula at zero or less)."""
    if faith >= 100:
        return None
    if faith <= 0:
        return sar
    s = ep - (ep - sar) / (1 - faith / 100.0)
    return s if s > 0 else None


def ratchet(prev, new):
    """The stop in force never moves down within a long leg."""
    if new is None:
        return prev
    if prev is None:
        return new
    return max(prev, new)


def norm_bars(raw, today=None, today_complete=False):
    out = {}
    for b in raw:
        if isinstance(b, (list, tuple)):
            d, o, h, l, c = b[:5]
        else:
            if b.get("interpolated"):
                continue
            d = b.get("date") or b.get("begins_at")
            o = b.get("open", b.get("open_price"))
            h = b.get("high", b.get("high_price"))
            l = b.get("low", b.get("low_price"))
            c = b.get("close", b.get("close_price"))
        d = str(d)[:10]
        if today and (d > today or (d == today and not today_complete)):
            continue
        out[d] = {"date": d, "open": float(o), "high": float(h), "low": float(l), "close": float(c)}
    return [out[d] for d in sorted(out)]


def bar_warnings(bars):
    """Dates whose prices cannot be right (low above open or close, or high below them): usually a copying slip."""
    return [b["date"] for b in bars
            if b["low"] > min(b["open"], b["close"]) + 1e-9 or b["high"] < max(b["open"], b["close"]) - 1e-9]


def start_short(bars, j, prior_ep, step):
    """Wilder's reversal: the bearish SAR is the ended long's extreme point, not below the last two highs."""
    b, p = bars[j], bars[j - 1] if j > 0 else bars[j]
    return {"side": "short", "sar": max(prior_ep, b["high"], p["high"]), "af": step, "ep": b["low"],
            "stop": None, "since": b["date"]}


def start_long(bars, j, prior_ep, step, faith):
    """Wilder's reversal: the new long's SAR is the bearish leg's lowest low, not above the last two lows."""
    b, p = bars[j], bars[j - 1] if j > 0 else bars[j]
    sar = min(prior_ep, b["low"], p["low"])
    return {"side": "long", "sar": sar, "af": step, "ep": b["high"], "stop": scaled_stop(sar, b["high"], faith),
            "since": b["date"]}


def simulate(bars, k, st, faith):
    """Apply bars k+1 .. end to state st (its sar and stop are the levels in force for bar k+1).
    Returns the final state (levels in force for the next session) and the flips seen."""
    step, af_max = af_params(faith)
    flips = []
    for j in range(k + 1, len(bars)):
        b, p = bars[j], bars[j - 1]
        if st["side"] == "long":
            if st["stop"] is not None and b["low"] <= st["stop"]:
                px = min(st["stop"], b["open"])  # a gap under the stop exits at the open
                flips.append({"date": b["date"], "to": "short", "price": r2(px)})
                st = start_short(bars, j, st["ep"], step)
                continue  # no acceleration on the reversal day
            if b["high"] > st["ep"]:
                st["ep"] = b["high"]
                st["af"] = min(st["af"] + step, af_max)
            sar = min(st["sar"] + st["af"] * (st["ep"] - st["sar"]), b["low"], p["low"])
            st["sar"] = max(st["sar"], sar)  # freeze: the SAR is never lowered while faith holds the long open
            st["stop"] = ratchet(st["stop"], scaled_stop(st["sar"], st["ep"], faith))
        else:
            if b["high"] >= st["sar"]:
                px = max(st["sar"], b["open"])
                flips.append({"date": b["date"], "to": "long", "price": r2(px)})
                st = start_long(bars, j, st["ep"], step, faith)
                continue
            if b["low"] < st["ep"]:
                st["ep"] = b["low"]
                st["af"] = min(st["af"] + step, af_max)
            st["sar"] = max(st["sar"] + st["af"] * (st["ep"] - st["sar"]), b["high"], p["high"])
    return st, flips


def spend_caps(acct):
    """What faith buys may spend: the lower of cash and buying power, so money reserved for queued orders is
    never counted and margin is never used. Coins are also held to crypto buying power. Returns (cap, coin cap)."""
    if not acct:
        return 0.0, 0.0
    cap = float(acct.get("cash") or 0)
    if acct.get("buying_power") not in (None, ""):
        cap = min(cap, float(acct["buying_power"]))
    cap = max(0.0, cap)
    coin = cap
    if acct.get("crypto_buying_power") not in (None, ""):
        coin = max(0.0, min(cap, float(acct["crypto_buying_power"])))
    return cap, coin


def buy_amount(want, cash_avail, done_today, blocked, due=True):
    """Size one faith buy: the wanted amount shrunk to the spendable cash left, or 0 and the reason it is skipped."""
    if not due:
        return 0.0, "parabola bearish: faith buys paused until the bullish flip"
    if done_today:
        return 0.0, "already bought today"
    if blocked:
        return 0.0, "buys blocked: %s" % blocked
    if cash_avail < MIN_ORDER_USD:
        return 0.0, "no spendable cash"
    amount = r2(min(want, cash_avail))
    if amount < MIN_ORDER_USD:
        return 0.0, "under the 1 USD minimum"
    return amount, None


def run_crypto(inp, acct, cash_avail, blocked_all):
    """A coin is buy-only. The connector has no crypto price history, so there is no parabola and no stop:
    faith sets the size of the faith buy and nothing else, and the result never carries a sell."""
    faith = max(0, min(100, int(inp.get("faith", 0) or 0)))
    f = faith / 100.0
    units = float(inp.get("units_held") or inp.get("shares_held") or 0)
    q = inp.get("quote") or {}
    last = float(q["last"]) if q.get("last") not in (None, "") else None
    ask = float(q["ask"]) if q.get("ask") not in (None, "") and float(q["ask"]) > 0 else last
    out = {"symbol": inp.get("symbol"), "asset": "crypto", "faith": faith,
           "params": {"faith_buy_pct_of_account": round(FAITH_BUY_PCT * f, 4)},
           "signal": "buy_only" if faith > 0 else "watch", "held": units > 1e-12,
           "live_price": None if last is None else round(last, 8)}
    cash_used = 0.0
    if faith > 0 and acct:
        want = float(acct["total_value"]) * FAITH_BUY_PCT / 100.0 * f
        amount, reason = buy_amount(want, cash_avail, inp.get("faith_buy_done_today"),
                                    inp.get("buys_blocked") or blocked_all)
        fb = {"wanted_usd": r2(want), "amount_usd": amount, "skip_reason": reason, "type": "market",
              "dollar_based": True, "netted": False}
        if amount and ask:
            fb["est_units"] = round(amount / ask, 8)  # before the fee Robinhood shows at preview
        out["faith_buy"] = fb
        cash_used = amount
    return out, cash_used


def run_symbol(inp, acct, cash_avail, today, today_complete, blocked_all):
    faith = max(0, min(100, int(inp.get("faith", 0) or 0)))
    f = faith / 100.0
    step, af_max = af_params(faith)
    out = {"symbol": inp.get("symbol"), "faith": faith,
           "params": {"af_step": step, "af_max": af_max,
                      "room_multiple": None if faith >= 100 else round(1 / (1 - f), 2),
                      "sell_fraction": round(1 - f, 4), "faith_buy_pct_of_account": round(FAITH_BUY_PCT * f, 4)}}
    bars = norm_bars(inp.get("bars") or [], today, today_complete)
    backfill = bool(inp.get("backfill")) or "state" not in inp
    faith_changed = False
    if backfill:
        bars = bars[-BACKFILL_BARS:]
        if len(bars) < SEED_BARS:
            out["error"] = "need at least %d completed daily bars to start a symbol" % SEED_BARS
            return out, 0.0
        k = SEED_BARS - 1
        sar = min(b["low"] for b in bars[:SEED_BARS])
        ep = max(b["high"] for b in bars[:SEED_BARS])
        st = {"side": "long", "sar": sar, "af": step, "ep": ep, "stop": scaled_stop(sar, ep, faith),
              "since": bars[k]["date"]}
    else:
        s = inp["state"]
        idx = [i for i, b in enumerate(bars) if b["date"] <= s["last_bar"]]
        if not idx:
            out["error"] = "bars must start at or before the state's last_bar (%s)" % s["last_bar"]
            return out, 0.0
        k = max(idx)
        st = {"side": s["side"], "sar": float(s["sar"]), "af": float(s["af"]), "ep": float(s["ep"]),
              "stop": None if s.get("stop") in (None, "", "none") else float(s["stop"]), "since": s["since"]}
        old_faith = int(s.get("faith", faith))
        if old_faith != faith:
            faith_changed = True
            st["af"] = min(st["af"], af_max)
            if st["af"] <= 0 and af_max > 0:
                st["af"] = step
            if st["side"] == "short" and faith >= 100:  # faith 100 has no stop, so it is always long
                st = {"side": "long", "sar": min(st["ep"], bars[k]["low"]), "af": 0.0, "ep": bars[k]["high"],
                      "stop": None, "since": bars[k]["date"]}
            if st["side"] == "long":  # a faith change recomputes the stop from scratch (no ratchet across it)
                st["stop"] = scaled_stop(st["sar"], st["ep"], faith)
    final, flips = simulate(bars, k, st, faith)
    if bar_warnings(bars):
        out["bar_warnings"] = bar_warnings(bars)
    if backfill and not flips:  # the long was already running when the backfill window opened
        out["leg_predates_backfill"] = True

    shares = float(inp.get("shares_held") or 0)
    held = shares > 1e-9
    q = inp.get("quote") or {}
    last = float(q["last"]) if q.get("last") not in (None, "") else None
    bid = float(q["bid"]) if q.get("bid") not in (None, "") and float(q["bid"]) > 0 else last
    ask = float(q["ask"]) if q.get("ask") not in (None, "") and float(q["ask"]) > 0 else last

    turned_bearish = (not backfill) and final["side"] == "short" and bool(flips) and flips[-1]["to"] == "short"
    turned_bullish = (not backfill) and final["side"] == "long" and bool(flips) and flips[-1]["to"] == "long"
    live_breach = bool(final["side"] == "long" and final["stop"] is not None and last is not None
                       and last <= final["stop"])
    stop_hit = final["stop"] if live_breach else None
    if live_breach:  # the long ends now; today's completed bar is applied to this bearish state on the next run
        day = today or bars[-1]["date"]
        flips.append({"date": day, "to": "short", "price": r2(last), "live": True})
        final = {"side": "short", "sar": final["ep"], "af": step, "ep": last, "stop": None, "since": day}
    elif turned_bearish:
        stop_hit = flips[-1]["price"]
    flip_pending = bool(final["side"] == "short" and last is not None and last >= final["sar"])
    preexisting = bool(backfill and final["side"] == "short" and held and faith < 100 and not live_breach)
    take_exit = bool(inp.get("take_exit")) and final["side"] == "short" and held and faith < 100
    sell_now = bool(held and faith < 100 and (turned_bearish or live_breach or take_exit))

    if sell_now:
        signal = "sell"
    elif not held and faith == 0:
        signal = "watch"
    elif final["side"] == "short":
        signal = "flip_pending" if flip_pending else "out"
    elif turned_bullish:
        signal = "resumed"
    else:
        signal = "hold" if held else "accumulate"

    out["state"] = {"side": final["side"], "sar": r4(final["sar"]), "af": round(final["af"], 6),
                    "ep": r4(final["ep"]), "stop": r4(final["stop"]), "since": final["since"],
                    "last_bar": bars[-1]["date"], "faith": faith}
    out.update({"signal": signal, "held": held, "flips": flips, "faith_changed": faith_changed,
                "wilder_sar": r2(final["sar"]), "stop": r2(final["stop"]) if final["side"] == "long" else None,
                "last_close": r2(bars[-1]["close"]), "live_price": r2(last), "live_breach": live_breach,
                "flip_pending": flip_pending,
                "under_wilder_sar": bool(final["side"] == "long" and last is not None and last < final["sar"])})

    # Faith buy: 1 percent of account value x faith / 100, only while the parabola is long,
    # capped by the spendable cash left, never under the broker minimum, once per trading day.
    cash_used = 0.0
    blocked = inp.get("buys_blocked") or blocked_all
    fb = None
    if faith > 0 and acct:
        want = float(acct["total_value"]) * FAITH_BUY_PCT / 100.0 * f
        due = final["side"] == "long" or (sell_now and not take_exit)
        amount, reason = buy_amount(want, cash_avail, inp.get("faith_buy_done_today"), blocked, due)
        fb = {"wanted_usd": r2(want), "amount_usd": amount, "skip_reason": reason, "type": "market",
              "dollar_based": True, "market_hours": "regular_hours", "netted": False}
        if amount and ask:
            fb["est_shares"] = round(amount / ask, 6)
        out["faith_buy"] = fb

    # Exit: sell (1 - faith / 100) of the shares, netted against today's faith buy into one order.
    if sell_now:
        px = bid or last or bars[-1]["close"]
        sell_shares = round(shares * (1 - f), 6)
        sell_value = sell_shares * px
        buy_amt = fb["amount_usd"] if fb and not fb["skip_reason"] else 0.0
        out["sell"] = {"shares": sell_shares, "price_used": r2(px), "value_usd": r2(sell_value),
                       "stop_hit": r2(stop_hit), "reason": "taken at the trader's request" if take_exit else
                       ("live price at or under the stop" if live_breach else "a completed bar touched the stop")}
        if inp.get("avg_cost") not in (None, ""):
            out["sell"]["losing_exit"] = bool(px < float(inp["avg_cost"]))
        diff = sell_value - buy_amt
        if buy_amt <= 0:
            net = {"side": "sell", "type": "market", "quantity": sell_shares} if sell_shares > 0 else {"side": "none"}
        elif diff >= MIN_ORDER_USD:
            net = {"side": "sell", "type": "market", "quantity": min(sell_shares, round(diff / px, 6)),
                   "value_usd": r2(diff)}
        elif diff <= -MIN_ORDER_USD:
            net = {"side": "buy", "type": "market", "dollar_amount": r2(-diff)}
            cash_used = r2(-diff)
        else:
            net = {"side": "none", "note": "the system sell and the faith buy are within 1 USD of each other"}
        net["market_hours"] = "regular_hours"
        out["net_order"] = net
        if fb and buy_amt > 0:
            fb["netted"] = True
    elif fb and not fb["skip_reason"]:
        cash_used = fb["amount_usd"]

    if preexisting:
        px = bid or last or bars[-1]["close"]
        out["preexisting_exit"] = {"since": final["since"], "sell_shares_if_taken": round(shares * (1 - f), 6),
                                   "value_usd": r2(round(shares * (1 - f), 6) * px),
                                   "note": "the backfill dates this exit before the skill managed the symbol; "
                                           "report it under Needs you and place nothing unless the trader says so"}

    # Faith 0 has no faith buy, so it keeps Wilder's resting stop (whole shares only).
    if faith == 0 and held and final["side"] == "long" and final["stop"] is not None and not sell_now:
        whole = int(shares + 1e-9)
        rs = {"type": "stop_market", "stop_price": r2(final["stop"]), "whole_shares": whole,
              "fractional_remainder": round(shares - whole, 6)}
        if whole < 1:
            rs["note"] = "under one whole share: no resting stop possible; sell at market when a run finds the price at or under the stop"
        out["resting_stop"] = rs
    return out, cash_used


def selftest():
    """Catches a mistyped copy of this file. Run: python3 faith_trader_calc.py --selftest"""
    acct = {"total_value": 5001.91, "cash": 3534.34}
    drop = [["d1", 100, 100, 100, 100], ["d2", 100, 100, 100, 100], ["d3", 50, 50, 10, 10]]

    def case(faith, shares, bid, stop):
        st = {"side": "long", "sar": 92.0, "af": 0.01, "ep": 100.0, "stop": stop, "since": "d0",
              "last_bar": "d2", "faith": faith}
        return run_symbol({"faith": faith, "bars": drop, "state": st, "shares_held": shares,
                           "quote": {"last": bid, "bid": bid, "ask": bid}}, acct, acct["cash"], None, False, None)[0]

    a, b, c = case(50, 10, 100.0, 84.0), case(97, 1, 380.0, 385.0), case(0, 4, 50.0, 92.0)
    up = norm_bars([[d, 10 + i, 11 + i, 9 + i, 10.5 + i] for i, d in enumerate("abcdefgh")] + [["i", 12, 12, 9, 9.5]])
    st, flips = simulate(up, 4, {"side": "long", "sar": 9.0, "af": 0.02, "ep": 15.0, "stop": 9.0, "since": "e"}, 0)
    dip = norm_bars([["p", 98, 99, 97, 98], ["q", 96, 97, 93, 94], ["r", 100, 110, 99, 109]])
    held, _ = simulate(dip, 0, {"side": "long", "sar": 95.0, "af": 0.01, "ep": 100.0, "stop": 90.0, "since": "p"}, 50)
    coins = run_account({"account": {"total_value": 10000.0, "cash": 500.0, "buying_power": 120.0,
                                     "crypto_buying_power": 60.0},
                         "symbols": [{"symbol": "LOW-USD", "asset": "crypto", "faith": 50, "quote": {"last": 10.0}},
                                     {"symbol": "TOP-USD", "asset": "crypto", "faith": 100, "quote": {"last": 10.0}}]})
    low, top = coins["results"]
    checks = [
        held["side"] == "long" and held["sar"] == 95.0 and held["stop"] == 90.0 and held["ep"] == 110.0,
        af_params(0) == (0.02, 0.2), af_params(97) == (0.0006, 0.006), af_params(100) == (0.0, 0.0),
        r2(scaled_stop(92.0, 100.0, 50)) == 84.0, r2(scaled_stop(92.0, 100.0, 75)) == 68.0,
        scaled_stop(92.0, 100.0, 97) is None, scaled_stop(92.0, 100.0, 0) == 92.0,
        a["signal"] == "sell" and a["faith_buy"]["amount_usd"] == 25.01 and a["net_order"].get("quantity") == 4.7499,
        b["sell"]["shares"] == 0.03 and b["net_order"].get("dollar_amount") == 37.12,
        c["net_order"].get("quantity") == 4.0 and "faith_buy" not in c,
        flips == [{"date": "i", "to": "short", "price": 10.4}],
        st["side"] == "short" and st["sar"] == 18.0 and st["ep"] == 9.0 and st["af"] == 0.02,
        spend_caps({"cash": 2884.35, "buying_power": 2650.01}) == (2650.01, 2650.01),
        spend_caps({"cash": 100.0, "buying_power": 900.0}) == (100.0, 100.0),
        coins["spend_cap"] == 120.0 and coins["cash_after_buys"] == 60.0,
        top["signal"] == "buy_only" and top["faith_buy"]["wanted_usd"] == 100.0 and low["faith_buy"]["wanted_usd"] == 50.0,
        top["faith_buy"]["amount_usd"] == 60.0 and top["faith_buy"]["est_units"] == 6.0,
        "sell" not in top and "net_order" not in top and "state" not in top,
        low["faith_buy"]["amount_usd"] == 0.0 and low["faith_buy"]["skip_reason"] == "no spendable cash",
    ]
    bad = [i for i, ok in enumerate(checks) if not ok]
    print("selftest ok" if not bad else "selftest FAILED at checks %s: this file was mistyped, write it again" % bad)
    return 1 if bad else 0


def run_account(inp):
    batch = "symbols" in inp
    syms = inp["symbols"] if batch else [inp]
    acct = inp.get("account")
    cap, coin_cap = spend_caps(acct)
    start = cap
    today, complete, blocked = inp.get("today"), bool(inp.get("today_bar_complete")), inp.get("buys_blocked")
    # spendable cash goes to the highest faith first and shrinks after each buy, stock or coin
    order = sorted(range(len(syms)), key=lambda i: -max(0, min(100, int(syms[i].get("faith", 0) or 0))))
    results = [None] * len(syms)
    for i in order:
        s = syms[i]
        try:
            if s.get("asset") == "crypto":
                results[i], used = run_crypto(s, s.get("account") or acct, min(cap, coin_cap), blocked)
            else:
                results[i], used = run_symbol(s, s.get("account") or acct, cap, s.get("today") or today,
                                              bool(s.get("today_bar_complete")) or complete, blocked)
        except Exception as e:  # one bad symbol must not sink the run
            results[i], used = {"symbol": s.get("symbol"), "error": "%s: %s" % (type(e).__name__, e)}, 0.0
        cap, coin_cap = max(0.0, cap - used), max(0.0, coin_cap - used)
    if batch:
        return {"results": results, "spend_cap": r2(start), "cash_after_buys": r2(cap)}
    return results[0]


def main():
    if len(sys.argv) > 1 and sys.argv[1] == "--selftest":
        sys.exit(selftest())
    json.dump(run_account(json.load(sys.stdin)), sys.stdout, indent=1)


if __name__ == "__main__":
    main()
```

Worked checks. Reference row: EP 100.00 and SAR 92.00 give a stop of 84.00 at faith 50, 68.00 at faith 75, 20.00 at faith 90, and none at faith 97 or 100. Faith buys on an account worth 5,001.91 USD: 50.02 USD at faith 100, 48.52 USD at faith 97, 25.01 USD at faith 50. Netting on that account: faith 50 with 10 shares and a 100.00 bid sells 5 shares (500.00 USD) against a 25.01 USD faith buy, so `net_order` is a sell of 4.7499 shares; faith 97 with 1 share and a 380.00 bid sells 0.03 shares (11.40 USD) against a 48.52 USD faith buy, so `net_order` is a buy of 37.12 USD; faith 0 with 4 shares sells all 4 and has no faith buy. Cash: three new symbols at faith 100, 60 and 40 on an account worth 10,000.00 USD with 120.00 USD of cash get 100.00, 20.00 and nothing ("no spendable cash"), and they get exactly the same with 500.00 USD of cash and 120.00 USD of buying power, because the lower figure is the cap. Cash of 2,884.35 USD with buying power of 2,650.01 USD gives a cap of 2,650.01 USD. Crypto on the account worth 5,001.91 USD: a coin gets a 5.00 USD buy at faith 10, 25.01 USD at faith 50 and nothing at faith 1 (under the 1 USD minimum), and no coin result ever carries a sell. Live check on 2026-10-04 with a year of TSLA daily bars: at faith 0 the calculator gives 11 stop-outs and is bearish since 2026-09-28; at faith 50 it gives 3 stop-outs and is long with a stop of 340.05; at faith 97 it gives none, is long with no stop, and reports the price under Wilder's SAR of 426.07.

## Tone and responsibility

This skill executes a mechanical system the trader chose and configured. Describe what the system says and what was done; do not dress it up as a prediction or as financial advice. When data is stale, a bar is missing, or a guard blocked something, say so plainly in the report. When something cannot be done (an order blocked at review, a connection that will not come up after setup, Claude Docs missing), stop and explain rather than improvise. A Robinhood connection that is not set up yet is not such a case in an attended run: guide the trader through "Initialization".