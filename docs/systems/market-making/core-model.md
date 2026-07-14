# Core Model

> Source: migrated from `reference/docs/core-model.md` on 2026-07-14. This is
> reusable market-making knowledge, not the contract for any active repository.

## Background

This document outlines the logic behind market making and how it automates the execution process.

## Ideas

The idea of implementing a market making strategy is to accumulate position in good levels while minimising our market footprint and giving away minimal information.

---

## Types of Execution

Execution can be summarised into two main groups:

| Type | Order Type | Aggressor | Applies to Execution | Trigger |
|------|------------|-----------|---------------------|---------|
| Hitting | Immediate-or-Cancel (IOC) | Ourselves | Hidden Quoter | 1. A change in our fair value (either change in underlying price or change in the vol surface) that make any existing order in the orderbook become attractive, i.e. we would like to trade against the order<br>2. A change in the order book (new orders come in) and we would like to trade against it |
| Quoting | Limit Order | Counterparty | Quoter | 1. For continuous quoting, our order will be kept on the exchange order book and let other market participants to trade against our order<br>2. For axe quoting, we flash our quote (limit order) on one or a few instruments for a few seconds (usually 1-sided instead of 2-sided) |

---

## Market Making Parameters

### 1/ Width

A few key risk associated to an option to add up for the width (theoretical edge) that one requires to enter an option position:

| Type | Description |
|------|-------------|
| Fee | All relevant fee to enter a option position, e.g. exchange fee, fee paid to PB / brokers |
| Premium | Usage of the credit line from paying upfront premium or posting margin to the exchange, less relevant if option margin account is settled in future style |
| Strike | Risk of having big strikes position which will grow bigger when we approach expiry |
| Delta | Potential loss in delta space given we have to reach out to other markets (options of the same/different names, underlying markets etc), i.e. hedging at a worse level compared to the underlying valuation we see on the trade |
| Roll | Any corporate action / change of implied dividend that affect the forward price offset, separated from delta cost despite both of them are expressed in delta as risk associated is very different |
| Vega / Option Count (Opt#) | Risk of losing in volatility space from the movement of the volatility surface |
| Local | Another kind of vega credit but focus on the part of the curve rather than only vega itself, mainly use for acknowledging any exposure on the part of the curve |

As the associated risk profile is different, different base credit (cost) should be assigned to different expiries and names.

A rough express of the width of one unit of a particular instrument can be:

```
Fee + (Premium Cost * Premium) + Strike Cost + Local Cost + Base Delta Cost * f(Delta) + Base Roll Cost * g(Delta) + h(Opt#)
```

where base cost means the cost we charge if the instrument have one unit of that particular exposure (1 Delta Option, ATM options which have 1 opt#)

f(), g(), and h() determines what percentage of the base cost we should charge to this option (e.g. f(0.5) = 0.6 means that the function is designed such that for a 0.5 delta option we would like to charge 60% of what we would charge a 1 Delta option)

---

### 2/ Price Offset / Lean

A vanilla market maker that aims at capturing bid-ask spread would like to keep the position as flat as possible, i.e. lean to reduce risk exposure in the option space whenever he has one.

- **Opt#**: We will lean to flatten out our vega position by expiry. Overall exposure will be reduced if we have opposite position across expiries but we will still lean to flatten out all expiries (although less eager than being on the same direction in all expiries)
- **Strike**: We will lean to clean up strikes near the ATM forward when we are approaching expiry. Eagerness will be determined by how the options will be settled and magnitude of unnecessary pnl variance due to expiration.
- **Roll**: We will lean to trade out the roll exposure especially when we are approaching earnings and there is more uncertainty in amount of div / possible change in ex-div date.
- **Local**: We will lean to trade out that delta bucket if we have strong exposure in nearby / consecutive strikes.
- **Delta**: we are less eager to reduce our delta exposure as it can be easily hedged in the underlying market. But worth looking into it once the liquidity pool is big enough for us to hedge delta using options.

Usually the magnitude of lean in any of the risk should be in terms of the cost itself, i.e. what % of the cost should I lean with the current exposure to that risk. One have to define the function of magnitude of lean against the limit usage ratio.

E.g. our limit is 100 opt# and we now long 100 opt#. We should be leaning 100% (depends on the function) of the opt# credit. If our width is 3 (-3 @ +3) and our opt# credit is 1, assuming we have no other exposure, our lean should be -1 (leaning to sell) and our new market should be (-4 @ +2)

Strike and roll lean is more on a instrument / expiry basis, i.e. when looking at the risk, exposure in one expiry is local and did not affect other expiry.

However for opt# risk, they are partially offsetting each other as we see the relationship of vol movement between expiries.

That should be a function of (to be completed):

- Ratio of Time to expiry (1 month to expiry option should be more correlated to the 2 month to expiry option than the one expires today)

---

### 3/ Size

Size should be a function of the following factors:

- Maximum risk limit of the entire portfolio
- maximum amount of risk (delta & opt#) one would like to take in one single trade / one single level
- Desired risk exposure in one single trade (which is in opt# / vega space and also relevant curve risk)

---

## Trader's Control / Discrete Decision

The parameterization of the algorithm should ensure minimal intervention. However, some of the elements might be hard to systematically addressed and sometimes requires trader to intervene.

- **Delta Mult**: additional base delta cost to be applied, to address higher hedging cost (usually from sharp underlying move or thin underlying order book)
- **Opt# Mult**: additional base opt# cost to be applied, to address increase of vol of vol (requires more edge to enter a vega position as vol surface is volatile than usual)
- **Size Mult**: option to size down on the entire name, usually happens when you found yourselves overtrade and it allows you to size down temporality and keep yourselves in the market while re-calibrating the parameters
- **Change in desired position**: usually done by the strategy engine but traders might have trading ideas / see opportunities on screen

---

## Market Making as a Mean of Execution

Although the above logic is usually applied to vanilla market making strategy which the ideal portfolio is to be as clean as possible, it is also very useful to accumulate position if one have a view on.

We could make use of the similar logic to accumulating position, e.g. internally flipping 100 opt# from the main portfolio to a sub portfolio (usually called the manual portfolio) while the script is only pointing at the main portfolio. We are still flat but what the script is seeing is we are short 100 opt# and would try to cover that by leaning to buy option until we are flat. So by keeping the manual portfolio with our desired position, we can make use of the market making technique to accumulate position. A robust logic of how to understand the exposure we currently have and how it affects our lean on different options in different names can automate the execution process without constant manual intervention to capture certain flow of the market.

One should not separate market making as a separate strategy from any positional strategy unless what they are looking for is a vanilla market making strategy. Market making can be run in line with the positional strategy that you adjust your offset so that you have bias to buy or sell but (almost) always willing to take on the opposite side given the right price (as a core belief of everything has a price).

---

## Important Input to the Whole Trading Cycle

- Greek profile for each instrument (~~Good to have it in a way that we generate a value (greek / fair value or any other thing related to that particular option) upon submitting a request on the internal server, but good for now if its stored in memory / db only~~) - Now live greek per instrument is obtained by API created by Vincent
- The Greek for the current portfolio (this should be prioritized as that would affect our offset in all instruments) (~~Waiting for the TBricks Portfolio exporter from Wei~~ Done)
- An engine that keeps generating a desired position (An input from the prop / position taking strategy)
- ~~Construction of a portfolio with an artificial sub portfolio for us to flip position around (Looking to implement with Tbricks, by register an opposite position in test portfolio (short in test if you wanna get long) while the script is pointing to the entire portfolio)~~ Done using excel interface for manual adj, going forward can make use of position adjustment trade register in TBricks
- Locality of the option chain: how close each option are to each other, should be able to approximate that by moving underlying price to that strike and observe the new vega profile of that and the other options.

---

## Parameter to be Pushed to TBricks

On an ideal situation that we calculate almost everything out of Tbricks (including portfolio greek, greek per instrument):

On a per context per instrument basis:

- Bid Absolute Offset
- Ask Absolute Offset
- Quote volume

For now we should have a set of the above parameters for each instrument:

- Hitting / Hidden quote
- Continuous Quoting
- Axe Quoting

---

## Parameters for the First Iteration of the MM Script

For the first iteration of the MM script (which we have yet to pull necessary input from TBricks / calculated on my own):

### Credit:

- base cost (Fee, Premium, Strike, Delta, Roll, Optct)
- base cost param (Delta, Optct)

### Lean / Offset:

- Limit (Strike, Roll, Optct)
- Limit usage param (Strike, Roll, Optct)
- Current Greek Exposure (Optct, Roll, Strike, for optct we have pre-calculate the adjusted exposure after taking the calendar offset into account)

### Size:

- ATM desired size
- size curve param
- limit on trade delta
- limit on trade optct
- max size limit
