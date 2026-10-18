# Faith Trader

A Claude plugin that manages the stocks in your Robinhood account using J. Welles Wilder's Parabolic SAR (from *New Concepts in Technical Trading Systems*, 1978), driven by a single setting per symbol: **faith**, an integer from 0 to 100.

- **Faith 0** is Wilder's system untouched: the parabola runs at his acceleration factor and a stop sells the whole position.
- **More faith** flattens the parabola, widens the stop, shrinks what a stop sells, and sets a recurring buy on every run.
- **Faith 100** never sells and buys 1 percent of the account on every run.
- **Crypto** held in the same account is buy-only: faith sets the size of each recurring buy and the plugin never sells a coin.

Every run computes each managed symbol's parabola from completed daily bars, takes the exits the system calls for, places the faith buys, and records everything in three Claude Docs that belong to you (Settings, State and Log). The only other setting is the **mode**: `advise-only` (the default) tells you what to do; `trade` places the orders.

## Requirements

- A paid Claude plan (Pro, Max, Team or Enterprise).
- The **Robinhood** connector, with an account that allows agent trading.
- The **Claude Docs** connector, where your settings, state and log live.
- Scheduled tasks, if you want a daily unattended run.

The plugin stops rather than guesses when any of these is missing, and walks you through connecting Robinhood on first use.

## Install

### In the Claude app

1. Open **Customize > Plugins**, click **Add**, then **Add marketplace**, then **Add from a repository**.
2. Enter `ThePonnz/claude-faith-trader`.
3. Select **Faith Trader** and click **Add**.

### In Claude Code

```
claude plugin marketplace add ThePonnz/claude-faith-trader
claude plugin install faith-trader@faith-trader-market
```

Or in one command inside a session (Claude Code v2.1.275 or later):

```
/plugin install faith-trader --marketplace ThePonnz/claude-faith-trader
```

## First run

Say something like "run faith" or "set up Faith Trader". The plugin checks your connections, shows the Robinhood account it may trade, and asks for your faith in each stock you hold. New traders start in `advise-only` mode; nothing is bought or sold until you switch to `trade`.

## Updates

```
claude plugin update faith-trader@faith-trader-market
```

Auto-update is off by default for third-party marketplaces; you can turn it on for this one in the plugin manager.

## Disclaimer

Faith Trader is a tool, not investment advice. In `trade` mode it places real market orders in your brokerage account on your standing instructions. Markets can gap through any stop. You are responsible for every order it places, for the faith values you set, and for reviewing its log. Use `advise-only` mode until you understand what it does. No warranty of any kind is given; see LICENSE.

## Layout

```
.claude-plugin/plugin.json        plugin manifest
.claude-plugin/marketplace.json   makes this repo its own marketplace
skills/faith-trader/SKILL.md      the skill, including its Python calculator
```
