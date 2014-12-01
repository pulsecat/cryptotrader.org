## Overview
Welcome to **Cryptotrader.org API**. We aim to provide an API that allows developer to write 
fully featured trading algorithms. Our automated trading platform can backtest and execute trading scripts coded in CoffeeScript language on historical data.


## Trading API

Each script has to implement the following two methods:

#### init(context)
Initialization method called before a simulation starts. Put any initizalition here.
  - context
    An object is passed to every method. Use **context** to store global variables and constants. 
    
**Example:**

    init: (context)->
      context.treshold = 0.21  
      ...  

#### serialize(context)
**Deprecated in favor of storage object (see below). The serialize method functionality is still supported for backward compatibility.**

#### handle(context,data,storage)
Called on each tick according to the tick interval that was set (e.g 1 hour)

  - context 
    Object that holds current script state

  - data 
    Allows to access current trading environment
    - data.at - current time in milliseconds
      
      **Example:**
          `debug new Date(data.at) # Logs current timestamp`

    - data.instruments - array of instrument objects (see Instrument object below) **Currenly only single instrument object is supported**
    - data.portfolio - Portfolio object (see below)  

  - storage
      The object is used to persist variable data in the database that is essential for developing reliable algorithms. In the case if an instance is interrupted unexpectedly, the storage object will be restored from the database to the last state. Do not use to store complex objects such as classes and large amounts for data (> 64kb).

      **Example:**

          handle: (context,data,storage)->
            ...
            if buy instrument
              storage.lastBuyPrice = instrument.price

**Example:**

    handle: (contex,data)->
      ...
      # pick default instrument
      instrument = data.instruments[0]
      # sell everything if current price is below 
      # volume weightened average price for latest 5 periods
      if instrument.price > instrument.vwap(5) 
        sell instrument

#### finalize: (context)-> (only Simulation mode)
An optional handler that will be called upon a simulation completes

**Example**

    init: (context) ->
      context.volume = 0

    handle: (contex,data)->
      context.volume += data.btc_usd.volume

    finalize: (contex)->
      debug "Total volume: #{context.volume}"


### Global methods and interfaces

#### Portfolio 
  The portfolio is a dictionary object with positions information.

**Example**

    btcAmount = portfolio.positions['btc'].amount
    usdAmount = portfolio.positions['usd'].amount
  

#### buy(instrument,[amount],[price],[timeout])
This method simulates/executes a purchase of specified asset. If **amount** parameter is missing or null, spends all funds available.

**Limit Orders**

if *price* is specified, the method will submit an order to buy at a specified price or better.
*timeout* parameter allows to limit the length of time in seconds an order can be outstanding before being canceled. 

if a buy was executed portfolio structure will be updated and [order] object is returned. 

**Example:**

    instrument = data.instruments[0]
    ## Spend half of cash for asset
    if buy instrument,portfolio.positions[instrument.curr()].amount / 2 * instrument.price
      debug 'BUY order traded'

#### sell(instrument,[amount],[price],[timeout])
This method simulates/executes selling specified asset. If **amount** parameter is missing or null, sell everything.

**Limit Orders**

if *price* is specified, the method will submit an order to sell at a specified price or better.
*timeout* parameter allows to limit the length of time in seconds an order can be outstanding before being canceled. 

if a sell was executed portfolio structure will be updated and [order] object is returned. 

**Example:**

    instrument = data.instruments[0]
    # Submit a limit order to sell everything at 5% below current price with order timeout set to 60 sec.
    if sell instrument,null,instrument.price * 0.95,60
      debug 'SELL order traded'

The orders created by buy/sell function are *auto orders* and get automatically cancelled by the engine when the next tick occurs.
If you need more control over how orders are placed/processed or create multple orders, see Advanced Orders section below.

#### debug(msg), info(msg), warn(msg)
Logs message with specified log level 
   
#### _
  The binding to Underscore library (http://underscorejs.org/) which has many helper functions
  
**Example**

    debug _.max(data.btc_usd.high.splice(-24)) # Prints maximum high for last 24 periods

#### plot(series)
  Records data to be shown on the chart as line graphs. Series is a dictionary object that includes key and value pairs.

**Example**

    # draw moving averages on the chart
    plot
        short: instrument.ema(10)
        long: instrument.ema(21)

#### plotMark(marks)
  Similar to plot method, but adds a marker that displays events on the charts.

**Example**

    high = _.last instrument.high
    if high > maxPrice
      plotMark
          "new high": high


#### setPlotOptions(options)
  This method allows to configure how series data and markers will be plotted on the chart. Currently the following options are supported:
  
  - color (in HTML format. e.g #eeeeee or 'darkblue')
  - lineWidth (default: 1.5)
  - secondary (default: false) specifies whether the series should plotted to secondary y-axis

**Example**
  
    init: (context)->
      ...
      setPlotOptions
        sell:
          color: 'orange'
        EMA:
          color: 'deeppink'
          lineWidth: 5


#### talib
  The interface to Ta-lib library (http://ta-lib.org/), which contains 125+ indicators for technical analysis.
  [TA-Lib Functions (Index)](/talib)
  

**Example**

    # Calculate SMA(10) using ta-lib API
    instrument = data.btc_usd
    value = talib.SMA
      startIdx: 0 
      endIdx: instrument.close.length-1
      inReal: instrument.close
      optInTimePeriod: 10


### Instrument object
The object that provides access to current trading data and technical indicators.

- OHLCV candlestick data. Includes the following array data

  - open
  - low
  - high
  - close
  - volumes
  
**Example:**

    instrument = data.btc_usd
    debug instrument.high[instrument.high.length-1] # Displays current High value

#### market 
  the market/exchange this instrument is traded on

#### period
  trading period in minutes 

#### price 
  current price

#### volume
  current volume

#### ticker
  provides access to live ticker data and has two properties: ticker.buy and ticker.sell
  gets updated each time before handle method is called or after an order is traded

#### vwap(period)
  Volume-weightened average price for the given period

#### ema(period)
  Calculates EMA value over specified period, [ta-lib] 

  This function is a shortcut for the following code:

    instrument = data.btc_usd
    result = talib.EMA
      startIdx: 0 
      endIdx: instrument.close.length-1
      inReal: instrument.close
      optInTimePeriod: period
    value = _.last(result)

#### macd(fastPeriod,slowPeriod,signalPeriod)
  Moving Average Convergence/Divergence indicator [ta-lib]

  **Return value:**
  An object with the following properties:

    - macd: <MACD>
    - signal: <Signal>
    - histogram: <Histogram>

  This function is a shortcut for the following code:

    results = talib.MACD
      startIdx: 0
      endIdx: instrument.close.length-1
      inReal: instrument.close
      optInFastPeriod: fastPeriod
      optInSlowPeriod: slowPeriod
      optInSignalPeriod: signalPeriod
    if results.outMACD.length
      values =
        macd: _.last results.outMACD
        signal: _.last results.outMACDSignal
        histogram: _.last results.outMACDHist

  **Example:**

    result = data.instruments[0].macd(10,21,8)
    if result
      debug "MACD=#{result.macd} Signal=#{result.signal} Histogram=#{result.histogram}"

### Advanced orders 
This set of functions gives more control over how orders are being processed:

####addOrder(instrument, type, amount, price, timeout)

Submits a new order with given type ("buy" or "sell") and returns an object containing information about order:

  - id - unique orderId. Note that id can be missing if the order was filled instantly.
  - active - true if the order is still active
  - cancelled - true if the order was cancelled
  - filled - true if the order was traded

The engine automatically tracks all active orders and update their statuses before a new tick or when a timeout occurs.

####getOrder(orderId)

Returns an order object by given id.

####cancelOrder(order)

Cancels an order

### Alerts
  Alerts functionality allows email messages to be sent from your code. 

#### sendEmail(message) [Live mode]
  Sends an email message to your e-mail. An e-mail address must be valid and verified. 

  **Example:** 

    # simple price alert
    handle: (contex,data)->
      ...
      if data.btc_usd.price >= 100
        sendEmail 'The price hit $100'
  
#### sendSMS(message) [Live mode] [Pro and VIP users]
  Sends a SMS message to the mobile number specified in the profile settings.
  The messages are limited to 160 characters.

  **Example:** 

    # simple price alert
    handle: (contex,data)->
      ...
      if sell instrument
        sendSMS "SOLD at #{instument.ticker.buy}"

### User interaction

The functionality allows to create bots that take some user input at runtime. 

####askParam title, defaultValue

In order to pass parameters to your strategy, add askParam method call to the top of the code, like in the example below:

    STOP_LOSS = askParam 'Stop Loss',100 # default value is 100
    MARKET_ORDER = askParam 'Market Order',false # will be displayed as checkbox
    MODE = askParam 'a) Low risk b) Aggressive','a' # can be a string value


Also check the implementation of Stop-Loss script that takes user's input:

https://cryptotrader.org/strategies/KWbNeR3CcMQp8SHkH


##Links

  - [TA-Lib : Technical Analysis Library](http://ta-lib.org)
  - [TA-Lib Functions (Index)](/talib)
