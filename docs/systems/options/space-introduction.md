# Options Space Introduction

> Source: migrated from `reference/docs/options-space-introduction.md` on
> 2026-07-14. This is general background, not a current system specification.

## Introduction

Options trading is a very different prospect than individual equities/futures or other strategies that involve only a handful of instruments. Each trading style has its own challenges (neither one is simple) but the key differences with options, namely: scale, speed, safety and flexibility make for a very different type of trading system. As we know the current approach is to trade through an extensible 3rd party system. The extensibility is a boon here as it allows us to slowly decouple things and do it ourselves, something we are already doing from the pricing and modelling side. This is actually quite a common path for options trading firms, it offers the easiest point of entry into a complex set of systems, allowing you to get to the market quickly whilst in parallel you develop your own systems. If you imagine the average standalone 3rd party system you could peel things off in this order: pricing/models, basic strategies, complex strategies, instrument management, parameter management, trade/position services, UI, back-end SOD/EOD systems – although this could be sliced a dozen different ways.

### Latency

A quick word about latency (you can't talk about options systems without mentioning latency). There are of course some strategies that are more sensitive than others but on the whole there is a latency *minimum* you need to achieve in order to get the fills you want or to adequately protect your resting liquidity. The latency required varies across the different exchanges and who you are up against (competing firms). The big futures exchanges tend to have the best technology on the back-end and therefore require the greatest investment as a participant however in many cases its who is fastest (and smartest) to the edge of the network, and a mature options firm will share its technology across markets so if they are competitive on CME (probably the fairest/fastest exchange) then they will apply this tech across the board. We're getting into exchange technology micro-structure here which is discussed more later on but roughly, for tick-to-order latencies (i.e. MD or event received in our network → order sent) you need to be 10-20us to be moderately competitive, sub 5us to be in the higher tiers and sub 1us to be in the Ultra Low Latency space. It is possible to achieve 5us in software but probably no quicker, everything else must be done in hardware (FPGA).

Trading options is not just about building an options execution system. In a way the 'easy' bit is to develop a system that can listen to native market data, process it in a strategy, make decisions and then execute natively - these are solved problems that many people have done. Where things get tougher, the tail-end of the process, is in the data – analysing the home market and understanding its micro-structure, understanding the relevant correlations with other instruments, fine-tuning the trading strategy, continuing to develop the valuation (fair value) and much more. This is harder to estimate but I wanted to mention it as it often gets forgotten, ==it's the unpredictable side of the business and time that should be accounted for==.

---

## Tenets of a Good Options Market Making System

What makes a good Options trading system? I think it can be broken down into 4 key areas:

### Scale
- A strategy or system component can work fine at small scale (single desk, small number of products) but will inevitably have scaling issues.
- You can often (hopefully if you system is designed this way!) throw hardware at the problem, but this has limitations and brings other challenges (e.g. managing large groups of hardware/copies of a system).
- Inbound data rates grow year-by-year, more complex strategies need more data.
- Number of instruments and their correlations also grow over time and as trading becomes more complex.
- Internal data such as logs, recorded exchange data tend to grow exponentially

### Speed
- In a lot of cases, not being 1st or 2nd means there isn't much left for the rest.
- Entire stack needs to be optimised, from the exchange cross connect onwards.
- Use of latest high-performance hardware, servers and switches.
- Use of kernel bypass methodology and technology.
- Not just CPUs, FPGAs for MD and execution, GPUs for pricing.

### Safety
- Large amounts of trading capital at risk.
- Need holistic approach to risk and trading limits.
- Including reliable day-to-day 'safeties' like FPGA-enabled bulk-cancelling.

### Flexibility
- Trading strategies come and go, lots of IP in a trading engine that can be leveraged.
- The system must have a flexible architecture and appropriate tooling to allow new strategies to be deployed quickly, and to tweak existing ones without drawn out development processes.

---

## Options Strategy Types

This list is by no means exhaustive and is really meant to give a general overview of the types of strategy needed in the average OMM business. Also worth pointing out I've focused on the more electronic/low-latency side of things since non-LL strategies are as numerous as they are disparate. Also worth pointing out that the descriptions are quite high-level, there are a huge number of variations and bells & whistles that can be applied to each, either to suit a particular business case or some subtlety at an exchange.

### Mass Quoting/Quote Streaming
- Mass Quotes are a unique message type usually only available to registered market makers that allow you to submit one or more 2-sided quotes. The messages are often dealt with in the matching engine preferentially and in some cases have extra functionality available such as MM/delta protection. When supplying large amounts of liquidity across unique instruments the ability to insert/update/delete en masse is invaluable. For a description of how CME handles MQ, look here: https://www.cmegroup.com/confluence/display/EPICSANDBOX/Mass+Quotes
- Broadly speaking there are two reasons to stream quotes:
  - **Compliance obligations**
    - What do you have to give in return for access to the MQ message? Usually its a guarantee to provide liquidity, often for a set number of instruments for a set period of the trading day. If you breach these obligations there can be hefty penalties
    - Therefore it's important to *always* be quoting the required spread, it's usually quite wide so there is little chance of execution, but you have to maintain the prices as the reference will constantly change
  - **Active/Tight**
    - This is for risk taking/active trading, i.e. you would happily trade at these levels
    - Since these quotes are going to be tighter (and more likely to be filled) then we need to update them much more frequently, control our exposure and employ tools and techniques to use our bandwidth optimally

### Quote Response
- A special type of message that a MM will respond to when another participant requests a quote for an instrument that currently has no market
- Often there are time limits on how quickly to respond and you could have allocation rules that are time sensitive (i.e. the quicker you respond, the more of any fills you get)

### Quote Protection
- Trading a complex like S&P (CME ES Options & Futures, SPY Options on 6+ exchanges, CBOE SPX Options) could extend to over 100k individual instruments. An active and competitive market maker could have an (aggressive) quote if a 1/3 or even 1/2 of these. During a big market move you could be significantly exposed so the ability to move your quotes, or more probably cancel them outright, is essential. ==Considerable effort goes into an extremely fast quote protection system and is an area where exchange micro-structure plays a big part.==

### Takeout/Electronic Eye
- An automated strategy to capture edge when the underlying or option price changes. Effectively you are picking off other peoples quotes before they can re-price. There is a lot of variation and subtleties here but there are two main flavours:
  - **Single option**
    - The quoted price of a single option (or combo) changes and puts the available edge within our threshold, our Takeout machine will then send an IOC to try and take that price
  - **Underlying move**
    - ==The underlying/base price changes which causes a knock-on effect to the derived option prices meaning a large number of strikes could now be mis-priced and within our edge threshold (i.e. worth buying or selling)==
    - This is usually the more lucrative kind of change but its computationally more complex than single-option since a base price change means you need to re-compute all the derived prices. There are however tricks to speed this process up such as pre-computing prices

### Auto Hedging
- Simple strategy that again can have lots of nuance or parameters, most obvious example is Delta hedging but there are others

---

## Other

### Quote Prioritisation
- Often done by Delta mainly because of limited resources allocated by the exchange to participants, e.g. if you need to change thousands of quotes, even with bulk messages, the quotes at the end could take a long time to be sequentially processed. Also it makes sense to prioritise those quotes which are likely to be more competitive

### Automation
- When trading a large number of complex products you can build up a big inventory which could become difficult to manage as parameters (e.g. vol) change and you also may develop inventory hot (or cold) spots as you trade in and out during the day
- At a basic level, quoting and takeout are based on offsets from the current market, by adding extra automation into your offset generation you can begin to remove some of the repetitive human element from Market Making e.g. ==by automatically re-evaluating offsets after parameter changes, vol/market moves, when trades happen and other factors from correlated events==
- Ideally this logic is baked directly in to your strategies (for performance) although that can become difficult if you have multiple, disparate quoting engines, in which case you need some kind of centralised service to monitor inputs such as underlying, vol, trades, and then output offset changes. This is just one model though, it could conceivably be accounted for in your vol model as well.

---

## Options System Design

Options systems share many characteristics with other prop systems, the differences are usually related to scale (number of instruments). Given the performance and scale requirements it is recommended to move away from a monolithic setup and make extensive use of services, to abstract away what you can from the pure algo & execution elements of our trading. This usually points you towards a 'Trading Engine' setup where a limited set of functions are performed in a high-performance, stand-alone application that can scale horizontally ad infinitum.

I've tried a number of different ways of showing a whole options eco system diagrammatically but it's not easy (and I'm no artist). Essentially it's trying to show a clean (ish) logical view of how data is shared between the major components. I broke out MD and instrument data as 'layers in the onion' as it is commonly used data across almost all services, we could also do this with prices and maybe even trades although the diagram needs to remain tidy and readable.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SYSTEM ARCHITECTURE                          │
│                        (Work in Progress)                           │
│                                                                     │
│   ·····> External data format (e.g. exchange)                       │
│   ───── Internal data format                                        │
│                                                                     │
│                          ┌──────────┐                               │
│                          │ Exchange │                                │
│                  ·····>  └──────────┘  <─────                       │
│              Reference Data    │    Orders,                         │
│              Market Data       │    Requests                        │
│              Order/Trades      │                                    │
│              Other Events      ▼                                    │
│                                                                     │
│   ┌────────────┐                          ┌─────────────────┐       │
│   │ MD Service │····>                     │ Trading Engine n │       │
│   └────────────┘                          └─────────────────┘       │
│         │                                                           │
│         ▼                                                           │
│   ┌─────────────┐    ┌────┐                                        │
│   │ Instruments │    │ UI │                                        │
│   └─────────────┘    └────┘                                        │
│         │   ┌──────┐                                                │
│         │   │Inst  │                                                │
│         │   │ DB   │                                                │
│         ▼   └──────┘                                                │
│   ┌────┐                                                            │
│   │ UI │    ┌──────────┐                                           │
│   └────┘    │ Param DB │                                           │
│             └──────────┘                                            │
│   ┌────────────┐              ┌────┐                               │
│   │ Parameters │              │ UI │                                │
│   └────────────┘              └────┘                                │
│         │                                                           │
│         ▼                ┌─────────────────┐    ┌───────────┐      │
│   ┌─────────┐            │                 │    │ Custom UI/│      │
│   │ Pricing │──────────> │ Trading Engine  │    │ Excel/RTD │      │
│   └─────────┘            │    (central)    │    └───────────┘      │
│                          │                 │                        │
│                          └─────────────────┘                        │
│                                 │                                   │
│   ┌────────────────┐    ┌────┐  │   ┌──────┐                       │
│   │ Risk/Portfolio │    │ UI │  │   │Trade │                       │
│   └────────────────┘    └────┘  │   │ DB   │                       │
│         ▲                       │   └──────┘                        │
│         │                       ▼                                   │
│         │              ┌──────────────────┐    ┌────┐              │
│         └──────────────│ Trade / Position │    │ UI │              │
│                        └──────────────────┘    └────┘              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Things to Note

- **Exchange**
  - The focal point for trading and the 'gateway' in/out of our systems. Usually two separate 'channels', market data (info that all participants get) and order execution (our connection for sending orders and receiving related data)
  - Broadly we receive Securities information, Market data, (Market) order and trade information, and other events on the market data channel
  - And our own order/trade info via the order execution connection

- **Trading engine** - this is described in more detail below
  - Note that multiple engines could need to talk to each other, e.g. to more directly/quickly share information such as trades (rather than via a trade/position server)

- **MD Service**
  - An essential service that handles market data events for everything except the engine (which does it natively)
  - This service and the engine will share some code, namely the decoder

- **Instrument**
  - This service could either take a direct feed from the exchange, or receive its instruments from the MD service
  - Another core/essential service that transforms and normalises exchange reference data into our format, and stores in a DB
  - It handles ALL queries related to instruments, from other services and for humans via a UI (or API)

- **Parameter Server**
  - A standalone application/UI/DB stack for handling parameters.
  - Parameter is quite vague but intentionally so since almost any application requires configuration and this can be at multiple levels: application, product, strategy, even outright option/line.
  - Therefore flexibility is key here so a relational DB is probably not the best idea. New parameters will come and go frequently also
  - Key-value pairs are probably the best approach, but this needs good management

- **Pricing Engine**
  - Theoreticals are needed both inside and outside the engine
    - INSIDE for making trading decisions
    - OUTSIDE for communicating the prices to humans (UIs, scripts) and other services (Risk)
  - You could achieve this a few different ways:
    1. Calculate inside the engine and broadcast out
    2. Calculate twice, inside and outside the engine
    3. Calculate outside, inject IN to the engine and make available to other services
  - 1) is acceptable but puts an I/O burden on the engine and where you have multiple engines trading the same instruments, would be wasteful
  - 2) also acceptable but at a system-wide level it wastes computation and potentially exposes you to the situation where theos in the engine are different to those outside (if the two are configured incorrectly)
  - 3) is preferable (in my opinion!) to calculate the bulk of our theoretical prices *outside* the engine and then share this with the wider system as well as with the engine itself
    - This requires some careful engineering to not create a performance issue
    - i.e. you don't want the engine waiting for an outside service to update theos during a market move
    - ==One method is to calculate a 'cube' of theoretical prices that have pre-computations for market (and parameter moves). Put another way, I already know ahead of time what my price will be within a reasonable underlying range, and same for volatility moves. This cube could be multi-dimensional if you want an extrapolation of other parameters.==
    - This is possible in software but GPUs are probably the best option, and there are a lot of open source and off-the-shelf implementations that can be harnessed

- **Risk/Portfolio Service**
  - A service to track risk across all our inventory *and* open orders
  - At an individual instrument level it should be possible to identify the various greek exposures
  - These can then be netted in myriad ways e.g. by portfolio (which could be a desk, or a product, or a group of products), or at the firm level
  - The attached API should be flexible and extensible since risk tools will be ~~an active area of development~~
  - A function to run shocks/scenarios is also necessary for day-to-day use. e.g. this would allow an application that can show desk level risks (delta, gamma, theta etc) over time and as vol or some underlying price moves

- **Trade/Position**
  - A conceptually simple service that stores all our trade information
  - From this baseline you can compute *position*
  - Given the sanctity of trade information, particular attention should be paid to securing/backing-up/retaining this data. We can never lose trades (either those in-flight from an engine, or historical), and the information captured must be accurate (price/quantity/time etc)

- **UIs**
  - Many services have UIs so humans can interact with the data without directly touching the DB
  - The UI will use an API that can also be directly used by humans, for example in a Python environment, allowing quants/researchers to query instrument/MD etc
  - In most cases it's recommended to use a web-UI. The technology is fast moving and true rapid development is possible. The one exception could be the market making application itself since this is often required to display large amounts of frequently updating information (option chains) across multiple screens. In which case a 'fat' UI might be necessary.

- **DBs**
  - Discussed in great detail elsewhere but at a high-level ==the DB technology should be abstracted away from the services, be fault tolerant and highly available==
  - To be truly fault tolerant and HA, this requires functions in the applications themselves. i.e. just using the built-in DB HA/failover tech isn't enough if the app isn't aware

---

## Trading Engines

Most OMM systems are organised around a central trading 'engine' as seen above. This is where the trading strategies or business logic reside and where decisions get made/executions happen with supporting systems connecting and sending data to/from the engine. The architectural challenge becomes deciding what logic goes in the engine and what can be abstracted out, usually following the mantra 'less is more' - put simply:

### Basic Trading Engine Data Flow

```
                        ┌───────────┐
                        │ Exchange  │
                        │   (☁️)    │
                       ╱└───────────┘╲
                     ╱↗                ╲↘
                   ╱                     ╲
    ┌───────────┐╱                         ╲┌──────────────┐
    │ Strategy  │         Basic Trading     │  MD / User   │
    │           │◄────    Engine        ────►│  Events      │
    └───────────┘         Data Flow         └──────────────┘
           ↑                                       │
           │                                       ↓
    ┌───────────┐                           ┌──────────────┐
    │  Order    │◄──────────────────────────│  Valuation / │
    │ Execution │                           │  Pricing     │
    └───────────┘                           └──────────────┘
```

*The circular data flow: Exchange → MD/User Events → Valuation/Pricing → Order Execution → Strategy → Exchange*

An **engine** is a core component performing critical functions. In this case its a piece of software that is highly optimised around receiving events at a high rate AND low latency in a concurrent environment, broadly an engine needs to:

- Process and distribute events to multiple concurrent recipients (i.e. strategies)
- Minimise latency and maximise throughput (often these are conflicting)
- Continue to handle input events even if processing or waiting on other events
- ==In general you need to optimise for the bursty/extreme events, usually at open/close or because of some other factors - this is usually when the market presents the most opportunities and the slower/less smart players will be either out of the market, extremely wide or just slow==

### Exchange Access

- **MD decoders**
  - Options trading is not normally limited to a few markets, inevitably as the business scales you connect to more and more markets
  - Also exchanges don't stand still and are often upgrading and changing their infrastructure
  - Over time this adds up to a fairly considerable amount of development time in the market access space so it's advisable to structure your code for maximum flexibility and re-use
  - I think this is difficult to achieve above 50%, given the differences across the exchanges, even with something like the Nasdaq family, that has 4 different options markets, all sharing tech, there are still differences that require special case code
  - The key lesson here is not to underestimate the effort required in building a decoder, and don't assume commonalities until you've looked deeper, things like Recovery can cause technical headaches, for example

- **OE encoders**
  - Similar story to above but harder as the differences are more stark across exchanges
  - There are always some common elements though, which aren't always clear when you start so again, flexibility is key
  - *NOTE: In areas where performance is of the upmost concern, this often conflicts with flexibility and re-use, so an engineering/business decision will need to be made in those cases*

- **Book building**
  - An area of obvious standardisation since we want all our downstream components to be agnostic as possible to the upstream source
    - i.e. our mass quoter shouldn't care if its quoting CME, or HKFE or...
  - Different exchanges supply books in different ways (e.g. Market By Price, or Market By Order) but there are only so many ways this is achieved
  - Aim for 100% general solution but unlikely to be possible since there are often variations in exchange rules
  - ==But even so, we should try to generalise those rules and remove any special cases from the code, just use configuration==
    - e.g. lets say most exchanges expect a two-sided pair when sending quotes, but some allow single-sided, this would be generalised in the code and activated/de-activated through config

### Concurrency

- Modern servers have a high core count and this continues to increase (with more modest increases in clock speed)
- Both CPU count and outright clock speed is important
  - No substitute for outright speed, for getting a single task completed quickly
  - Multiple concurrent cores give you an opportunity to distribute tasks and achieve concurrency
- A threading model that doesn't lock would be the gold standard here
- Suggest investigating the **Disruptor** pattern (https://lmax-exchange.github.io/disruptor/)
  - Although originally invented for Java this can be applied in C++
  - It requires some special configuration on the hardware but this is not difficult

### Valuation & Pricing

- Simple market data prices are not enough to make trading decisions, that is the market/public view of the asset - what is our view?
- Broadly this can be thought of as two elements:
  - The **Valuation**/Base price - this is the value we give to the underlying asset, e.g. the Future
    - *Influenced by:* bid/offer spread of target asset, volumes of BBO and the depth, recent trades, correlated products
  - The **Theo price** - this is the derived price of the option from the valuation
    - *Influenced by:* valuation, strike, term, volatility, dividends, rates

- **Internal/External theo pricing**
  - ==Theoretical prices could be computed outside the engine, in something like a GPU which is capable of massively parallel operations==
  - It would then deliver a multi-dimensional block of theo prices to the engine
  - Which would be stored in memory and accessed as required by strategies without having to wait for a refresh
    - e.g. as the underlying price changes, the engine can just lookup the relevant option price
    - NOTE: Certain events cause a full pricing refresh

- **Valuation development**
  - The bulk of the valuation effort is actually an analysis or research project, involving lots of backtesting (and outside the scope of this page)
  - The code itself is relatively simple, once a good, extensible and flexible Valuation framework has been developed
  - Some examples of valuation inputs outside of basic calculations like VWAP:
    - Composite - a valuation for an asset that takes prices from other assets, e.g. there might be a statistically significant correlation between the HSI and the price of Pork Futures
    - Exchange events - some exchanges deliver information in such a way that can be used as a predictor for an upcoming market data event
      - e.g. it is very well known (publicly) that CME delivers order fills to the executing parties BEFORE the public market data, such knowledge could be used to act quicker on certain events

### Strategy

- **Strategy library**
  - A sub-section of the engine that is structured to help easy strategy dev, samples/templates around basic liquidity taking and liquidity providing strats
  - Can be built/maintained separately (accounting for potentially sensitive IP)
- ==One strategy tends to reside on one core, so there is no queuing==

### Other Engine Components

- **Limit**
  - Limit layer
  - Standardised and can be built outside the engine for testing and validation

- **Logging**
  - High performance binary logging system
  - Dynamic tweaking of log levels, usually operate in a low or zero-logging environment
    - Can set to a deeper log level through C&C
  - System-wide data capture system with enriched data (e.g. correlation IDs in messages or other meta data)
    - Some troubleshooting can take place without looking at application logs
    - There is a bare minimum needed though, usually at start up and when sending orders

- **C&C - Command & Control**
  - View of health of the engine(s)
  - Gets very complex when you have lots of engines, so need a good UI
  - Allows introspection of certain elements live, e.g. interrogate the engine for various statistics
  - Also can change debug level, very useful in troubleshooting when running in a zero/light logging mode

---

## FPGA

Software engines have their limits when it comes to performance, there is only so fast you can get a bit of information from a network switch, along a cable, through the NIC, into the program, be processed, and then send an order back out in reverse. This information is slightly out of date but the fastest a SolarFlare card can process data in/out (this is with zero actual processing so just the time in and out), is around **1.5us**. I believe there will always be a place for software given the development timelines (rapid), the availability of qualified people (high) and its flexibility (very high). FPGAs have a place in a hybrid model where they do simplified or boiled-down elements of the trading engine extremely quickly, the challenge becomes in isolating those tasks and integrating them with the software engine. Some examples of common FPGA use cases in a trading system:

- **Quote protection**
  - Often the first strategy developed, this is quite simple a very quick way of cancelling your quotes when some conditions are met (e.g. market (or correlated markets) moves by X)
  - Many of the elements developer here can be used in more complex strategies (processing MD and building a book, sending a message back to the exchange)

- **Takeout**
  - See above sections for more details on these strategies but this is fairly simple to reason about. The FPGA would be pre-loaded with a series of 'if this, then that' type instructions
    - We would take our theoretical prices for all relevant strikes and turn these into simple instructions for the FPGA
    - The FPGA would then just monitor the market prices for those instruments, and if they reach or exceed our threshold, it would send an order (probably IOC)

- **Risk limits**
  - Given the inherent speed of an FPGA it can be used as an inline (i.e. on the critical path) risk/limit engine, assessing orders as they flow through the system and blocking/allowing
  - This is probably still too much latency in the ULL space though so limits are often baked-in to the pre-loaded decisions on the card

- **Market Data/feed handler**
  - Already mentioned as a basic building block for many (Although not all!) strategies, this data could then be broadcast to other systems, or be used to shorten the latency in the Software engine

### Keep it Simple!

If you try and implement a full quoting engine in an FPGA it will either a) take years to do, or b) be too slow, or c) just not work. It's key to understand your end-to-end architecture and that of the exchange to identify which parts of the strategy can be pulled out of the software engine and into the FPGA. The strategy then needs to be kept 'fed' by the software, a two-way relationship where the software engine keeps the decision points updated in the card and the card sends feedback regarding market events. One of the engineering challenges is extracting the 'fast bit' from a strategy, implementing it in the FPGA and then integrating it with the software

### You Can't Beat Physics

A couple of things to consider when thinking about ULL systems - 10GHz networks operate at a wire bitrate of ~322MHz, the speed of light is roughly 2.99x10⁸ metres/sec (in a vacuum, in copper its 2.3x10⁸ metres/sec, in fibre its 2.0x10⁸ metres/sec). Interesting tidbit, a foot (~30cm) of wire is roughly 1ns in latency (https://americanhistory.si.edu/collections/search/object/nmah_692464).

A well developed FPGA will process information at wire speed with minimal interruptions. If you imagine the shortest possible distance between the exchange cross connect and a switch, and then the same again to the server and so-on, it gives you an idea of the minimums with a ULL setup. There are still advances being made in information egress (ingress is much simpler), in devices such as Muxers (https://www.arista.com/en/products/7130-meta-mux and https://www.cisco.com/c/en/us/products/switches/nexus-3550-series/index.html) and also in fibre optics (hollow core cables) so expect this to slowly tick down nano-by-nano.

### Development

FPGA development timelines tend to be longer because of the differences in the process. It's not a case of: write code, compile code, look for outcomes. I wont go into detail but there are many steps, each of which can really extend a development timeline, especially when aiming for ULL, some more info here: https://www.fpgatutorial.com/fpga-build-process/

There are broadly 3 ways to approach the implementation of FPGA into your engine:

1. **Completely bespoke**, i.e. entirely inhouse using something like Verilog and VHDL
   a. A variation here is you could purchase a 3rd party MAC/PHY or TCP/IP stack (although generally the accepted view is that doing it all yourself gives the best results)

2. **In-house but using high-level languages** such as C opr C++ used in an environment that produces FPGA code
   a. This is a fast-maturing space although I have personally not used any of them
   b. Generally its accepted this wont be as fast as native but it will be faster than pure software
   c. e.g. Open CL or something like https://www.xilinx.com/applications/data-center/financial-technology/accelerated-algorithmic-trading.html

3. **Off-the-shelf**
   a. There are a number of 3rd party offerings that generally skew towards MD
   b. In theory you are quicker to market, but they are never as fast as home-grown and of course you are locked-in
   c. e.g.
      - https://www.enyx.com/nxfeed/
      - https://novasparks.com/ticker-plant/
      - https://www.raptorfintech.com/raptor

---

## Exchange Technology Micro-structure

In the context of trading systems, technology microstructure refers to the art/science of gaining a deep understanding of how an exchange/counterparty operates both at the software/protocol level and with hardware. Some of this information is made readily available, some of it you have to ask, or work out. Some examples where you can potentially gain an edge:

### Market Data Feeds

- Usually an exchange will have multiple sources for market data, this might be obvious by the IP address or it may be hidden
- Since these different sources will be on different hardware and have some differences in the path they follow on the exchange side, to your network, they may experience different load characteristics
- Therefore you may receive MD for instrument A quicker on a different source
- General point, remember data is handed to you sequentially over a line, so don't oversubscribe! i.e. dont request too much market data down a single link or you will induce delays

### Order Gateways

- A similar situation to MD, and very dependent on exchange architecture
- Put simply, at different moments, different order endpoints will be under different pressure (due to market activity)
- So you either need to dynamically adjust to this, and send your orders to the least loaded gateway (if you can elicit this, it's not always possible), or spread your orders out to increase the chances (sometimes sending more than 100% of your quantity)
- Some exchanges have 'throttles' which is another factor to be considered when distributing your orders

### Order Messages

- Depending on the type of messaging used you can gain modest latency improvements by only sending exactly what you need, the bare minimum required by the exchange
- It's worth talking about 'bit time' (https://en.wikipedia.org/wiki/Bit_time) here for a second
  - As previously discussed, speed of light in fibre is approximately 2.0x10⁸ m/s. Doesn't matter if you're using 1G, 10G or 25G - this is a constant
  - What is different is the speed at which bits can be 'clocked' on to the wire:
    - For 1G, a single bit of data will take 1ns
    - ==For 10G, it's 0.1ns - so if you find 10 bits of data you don't need to send - you just saved a nanosecond. This might be the difference in you hitting/missing==

---

## Infrastructure

Not to be overlooked in a low-latency environment. You could have the fastest software on earth but that would mean very little if you had a slow path in your network, or your host system is not tuned properly. Some high-level items worthy of note, when thinking about your infrastructure:

### End-to-end Latency Visibility

- Have easy, clean data for monitoring latencies
- Report/highlight regressions

### Local Network

A mixture of different technologies is necessary for the lowest latency and highest throughput:

*This is a simplified view missing some details, and there are variations (e.g. you could use a L2 switch as a L1 switch, also I've ignored the complexities around dealing with BGP)*

**Legend:**
```
────────►  Market Data
════════►  Orders
━━━━━━━━►  MD & Orders
```

```
                                    ┌─────────────────┐
                                    │   L2 Mux        │◄────────┐
                                    │   Switch        │         │
                                    └────────┬────────┘         │
                                             │                  │
                                             ▼                  │
┌───────────┐                       ┌─────────────────┐   ┌─────┴─────────┐
│           │◄═════════════════════►│   L1 Switch     │   │  LL Trading   │
│ Exchange  │───────────────────────►                 │   │    Server     │
│           │━━━━━━━━━━━━━━━━━━━━━━━►└────────┬────────┘   └───────────────┘
└───────────┘                                │
                                             │
                                             ▼
                                    ┌─────────────────┐   ┌───────────────┐
                                    │   L3 Switch     │◄━━►│ Non-LL Trading│
                                    │                 │   │    Server     │
                                    └────────┬────────┘   └───────────────┘
                                             │
                                             ▼
                                    ┌─────────────────┐   ┌───────────────┐
                                    │   Tap/Agg       │───►│ Data capture  │
                                    │   Switch        │   │    server     │
                                    └─────────────────┘   └───────────────┘
```

### Servers

- **NIC tuning**
  - Whether you're using Solarflare or Mellanox or whatever, they have tuning parameters that need to be set correctly based on your workload (some parameters are skewed towards latency, some towards throughput, for example)

- **C/P states**
  - By default, a server off the shelf from Dell or HP will have lots of smart power-saving functionality turned on. We don't care about power saving, and these functions cause jitter - turn them off

- **Hyper threading**
  - More cores == good, right? No, again this causes jitter and increased latency, so should be disabled

### Testing

- **Labs for testing new hardware**
  - New switches, NICs, cables, servers - these come and go, you need an environment where these can be easily tested, with repeatable tests that give an outcome that can be used for comparison
  - Can also be used for testing software improvements
