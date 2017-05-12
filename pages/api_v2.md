Table of Contents
=================

  * [Overview](#overview)
  * [Trading API](#trading-api)
    * [Code Structure](#code-structure)
    * [Global data and methods](#global-data-and-methods)
    * [Instrument object](#instrument-object)
    * [Core Modules](#core-modules)
      * [Logger](#logger)
      * [Plot](#plot)
      * [Alerts](#alerts)
      * [Lodash library](#lodash-library)
    * [Modules](#modules)
      * [Ta-lib](#ta-lib)
      * [Params](#params)
      * [Trading](#trading)
      * [Datasources](#datasources)
      * [Margin Trading](#margin-trading)



## Overview
Welcome to **Cryptotrader.org API**. We aim to provide an API that allows developers to write 
fully featured trading algorithms. Our automated trading platform can backtest and execute trading scripts coded in CoffeeScript on historical data.



---
## Trading API

### Code Structure
Each script has to implement the following two methods:
##### init
Initialization method called before trading logic starts. Put any initizalition here.
    
**Example:**

    init: ->
      context.treshold = 0.21  
      ...  

##### handle
Called on each tick according to the tick interval that was set (e.g 1 hour)


**Example:**

    handle: ->
      ...
      # pick default instrument
      instrument = data.instruments[0]
      debug "Price: #{instrument.price}"




### Global data and methods

##### @config
global object that contains an instance configration

  - market - primary market on which the instance currently running on 
  - pair - primary instrument pair 
  - interval - primary interval

##### context
object that holds current script state

##### data
provides access to current trading environment
- data.at - current time in milliseconds
      
- data.instruments - array of instrument objects (see Instrument object below). *Currenly only single instrument object is supported*
 

##### storage 
the object is used to persist variable data in the database that is essential for developing reliable algorithms. In the case if an instance is interrupted unexpectedly, the storage object will be restored from the database to the last state. Do not use to store complex objects such as classes and large amounts for data (> 64kb).

**Example:**

          handle: ->
            ...
            if orderId
              storage.lastBuyPrice = instrument.price

##### sleep(ms)
  Pauses code execution for specified number of milliseconds	

##### stop
  The method immediately stops script execution and updates the log with the resulting balance.
  
##### onRestart
  This hook method is called upon restart

**Example**

  onRestart: ->
   
      debug "Restart detected"

##### onStop
This hook method is called when the bot stops


**Example:**

    onStop: ->  
    	debug "The bot will be stopped"

### Instrument object
The object that provides access to current trading data and technical indicators.

- OHLCV candlestick data. Includes the following array data

  - open
  - low
  - high
  - close
  - volumes
  
**Example:**

    instrument = data.instrument[0]
    debug instrument.high[instrument.high.length-1] # Displays current High value

##### market 
  the market/exchange this instrument is traded on

##### interval
  trading interval in minutes 

##### price 
  current price

##### volume
  current volume





### Core Modules 
---

#### Logger

##### debug(msg), info(msg), warn(msg)
Logs message with specified log level 

---
#### Plot


##### plot(series)
  Records data to be shown on the chart as line graphs. Series is a dictionary object that includes key and value pairs.

**Example:**

    # draw moving averages on the chart
    plot
        short: instrument.ema(10)
        long: instrument.ema(21)

##### plotMark(marks)
  Similar to plot method, but adds a marker that displays events on the charts.

**Example:**

    high = _.last instrument.high
    if high > maxPrice
      plotMark
          "new high": high


##### setPlotOptions(options)
  This method allows to configure how series data and markers will be plotted on the chart. Currently the following options are supported:
  
  - color (in HTML format. e.g #eeeeee or 'darkblue')
  - lineWidth (default: 1.5)
  - secondary (default: false) specifies whether the series should plotted to secondary y-axis

**Example:**
  
    init: ->
      ...
      setPlotOptions
        sell:
          color: 'orange'
        EMA:
          color: 'deeppink'
          lineWidth: 5

---
#### Alerts


##### sendEmail(text) [only Live mode]

Sends an email message to your e-mail. An e-mail address must be valid and verified.

**Example:**

	# simple price alert
	handle: ->
	  ...
	  if @data.instruments[0].price >= 100
	    sendEmail 'The price hit $100'
	  
---

#### Lodash library
  The binding to Lodash library (http://lodash.com/) which has many helper functions
  
**Example:**

    debug _.max(instrument.high.splice(-24)) # Prints maximum high for last 24 periods

### Modules 

Non-core modules shoud be imported using **require** directive before they can be used.

---

#### Ta-lib
  The interface to Ta-lib library (http://ta-lib.org/), which contains 125+ indicators for technical analysis.
  [TA-Lib Functions (Index)](/talib)
  

**Example**

    talib = require 'talib'
    # Calculate SMA(10) using ta-lib API
    instrument = data.instruments[0]
    value = talib.SMA
      startIdx: 0 
      endIdx: instrument.close.length-1
      inReal: instrument.close
      optInTimePeriod: 10
      
---
      

#### Params 
The module allows to create bots that take user input at runtime.

##### add title, defaultValue

In order to pass parameters to your strategy, put **add** method call to the top of the code, like in the example below:


	params = require 'params'
    STOP_LOSS = params.add 'Stop Loss',100 # default value is 100
    MARKET_ORDER = params.add 'Market Order',false # will be displayed as checkbox
    MODE = params.add 'a) Low risk b) Aggressive','a' # can be a string value
    
    
##### addOptions title, options, defaultValue

Adds a control that provides a menu of options


	params = require 'params'
    ORDER_TYPE = params.addOptions 'Order type',['Market','Limit','Iceberg'],'Limit' # default value is 'Limit'
---

#### Trading

Provides an interface to base trading on any exchage supported by the platform

##### Portfolios object
The **portfolios** object gives access to information about funds on all exchanges the algorithm has access to. During live trading this data gets updated automatically before handle method is called and after orders are traded.
**@portfolio** is a link to primary markets's portfolio 
  

**Example:**
       
    trading = require "trading"
    
    handle: ->
      instrument = @data.instruments[0]
      info "Cash: #{@portfolios[instrument.market].positions[instrument.curr()].amount"
      info "Assets: #{@portfolios[instrument.market].positions[instrument.asset()].amount"





##### buy(instrument,type,[amount],[price],[timeout])
This method executes a purchase of specified asset. 

- type - 'limit' or 'market'
- amount - order amount
- price - order price
- timeout - specifies the length of time in seconds an order can be outstanding before being canceled. 


**Example:**

    trading = require "trading"
    
    handle: ->
      instrument = @data.instruments[0]
      cash = @portfolio.positions[instrument.curr()].amount
      #  1/2 of cash
      if trading.buy instrument, 'limit', cash / 2 / instrument.price, instrument.price
        debug 'BUY order traded'

##### sell(instrument,type,[amount],[price],[timeout])
This method executes a sale of specified asset. 

**Example:**

    trading = require "trading"
    
    handle: ->
      instrument = @data.instruments[0]
      # check if 
      if @portfolio.positions[instrument.asset()].amount > 0
        if trading.sell instrument
          debug 'SELL order traded'


**Advanced orders**

This set of functions gives more control over how orders are being processed:

##### addOrder(order)

Submits a new order and returns an object containing information about order

The order parameter is an object contaning:
- instrument - current instrument
- side - order side "buy" or "sell"
- type - 'stop' or 'limit' or 'market'
- amount - order amount
- price - order price

**Returns:**

- id - unique orderId. Note that id can be missing if the order was filled instantly.
- active - true if the order is still active
- cancelled - true if the order was cancelled
- filled - true if the order was traded

The engine automatically tracks all active orders and peridically update their statuses.

**Example:**

    	...
		order = trading.addOrder 
  		instrument: instrument
  		side: 'buy'
  		type: 'limit'
  		amount: amount
  		price: instrument.price * 1.05
        if order.id
        	debug "Order Id: #{order.id}"
        else
        	debug "Order fulfilled"

##### getActiveOrders()
Returns the list of currently open orders

##### getOrder(orderId)

Returns an order object by given id.

##### cancelOrder(order)

Cancels an order.

##### getTicker(instrument)  (only Live mode)

Returns live ticker data. The object includes two properties: buy and sell that represent current best bid and ask prices respectively.
In backtesting mode the buy and sell are set to the current price.

**Example:**

    trading = require "trading"
    
    handle: ->
      instrument = data.instruments[0]
      ## Get best ask price
      ticker = trading.getTicker instrument
      bestAskPrice = ticker.sell
      if trading.buy instrument,@portfolio.positions[instrument.curr()].amount / instrument.price,instrument.price,bestAskPrice
        debug 'BUY order traded'


##### getOrderBook(instrument) (only Live mode)

Allows to access market depth data for current market. The function returns an object that contains 'asks' and 'bids' fields, each of which is an array of [price,amount] elements representing orders in the orderbook.

**Example:**

	# find the price for which 1 BTC can be bought.

    orderBook = trading.getOrderBook instrument
    volume = 1
    if orderBook
        # logs the whole order book
        for key in ['asks','bids']
            debug "#{key}: #{orderBook[key].join(',')}"
        sum = 0
        for o in orderBook.asks
            sum += o[1]
            if sum >= volume
                debug "#{volume} BTC can be bought for #{o[0]}"
                break
    else
     	debug "orderbook isn't available" 
Note that in backtesting mode the method returns a null value.



---

#### Datasources 

Enables access to up to 5 additional trading instruments on different markets and trading intervals. The primary instrument, which the instance is running on, is added by default and always the first element in @data.instruments array and not needed to be added explicitly.


##### add(market, pair, interval, size=100)

Adds and returns a new instrument object that will be preloaded with *size* ticks (up to 500) upon the initialization of the program. This method can be used only at the top level of the script.
Currently, up to 5 additional instruments are allowed.

##### get(market, pair, interval)

Gets the instrument object with specified configuration that was added at the initialization stage

**Example:**

	trading = require 'trading'
	ds  = require 'datasources'

	ds.add 'poloniex', 'xmr_btc', '1h'
	ds.add 'poloniex', 'eth_btc', '4h'
	ds.add 'poloniex', 'eth_btc', '1d'

	TIMEOUT = 60


	 
	handle: ->
	    eth = ds.get 'poloniex', 'eth_btc', 5 # primary instrument
	    xmr = ds.get 'poloniex', 'xmr_btc', '1h'
	    eth4h = ds.get 'poloniex', 'eth_btc', '4h'
	
	    for ins in @data.instruments
	        debug "#{ins.id}: price #{ins.price} volume: #{ins.volume}"

	    xmrTicker = trading.getTicker xmr
	    xmrAmount = @portfolios.poloniex.positions.xmr.amount
	    unless xmrAmount > 0.1
	  	    trading.addOrder 
		        instrument: xmr
		        type: 'limit'
		        side: 'buy'
		        amount: 1
		        price: xmrTicker.buy
		        timeout: TIMEOUT
---


#### Margin Trading 

This module enables leveraged trading on Poloniex and Bitfinex

##### getMarginInfo(instrument)
Returns your trading wallet information for margin trading:
- margin_balance - the BTC (Poloniex) or USD (Bitfinex) value of all your trading assets
- tradable_balance - Your tradable balance in BTC or USD (the maximum size you can open on leverage for this pair)

**Example:**

    mt = require "margin_trading"
    
    handle: ->
      instrument = data.instruments[0]
      info = mt.getMarginInfo instrument
      debug "price: "margin balance: #{info.margin_balance} tradeable balance: #{info.tradable_balance}"

##### getPosition(instrument)
Returns the active position for specified instrument 

    mt = require "margin_trading"
    
    handle: ->
      instrument = data.instruments[0]
	  pos = mt.getPosition instrument
      if pos
        debug "position: #{pos.amount} @#{pos.price}"

##### closePosition(instrument)
Closes open position

##### buy(instrument,type,[amount],[price],[timeout])
This method executes a purchase of specified asset. 

- type - limit, stop (Bitfinex) or market (Bitfinex) 
- amount - order amount
- price - order price
- timeout - specifies the length of time in seconds an order can be outstanding before being canceled. 


**Example:**

    mt = require "margin_trading"
    
    handle: ->
      instrument = data.instruments[0]
      info = mt.getMarginInfo instrument
      ## Open long position 
      if mt.buy instrument,info.tradable_balance/instrument.price,instrument.price
        debug 'BUY order traded'

##### sell(instrument,type,[amount],[price],[timeout])
This method executes a sale of specified asset. 

**Example:**

    mt = require "margin_trading"
    
    handle: ->
      instrument = data.instruments[0]
      info = mt.getMarginInfo instrument
      ## Open short position 
      if mt.sell instrument,info.tradable_balance/instrument.price,instrument.price
        debug 'SELL order traded'


**Advanced orders**

This set of functions gives more control over how orders are being processed:

##### addOrder(order)

Submits a new order and returns an object containing information about order

The order parameter is an object contaning:
- instrument - current instrument
- side - order side "buy" or "sell"
- type - 'limit' 
- amount - order amount
- price - order price

**Returns:**

- id - unique orderId. Note that id can be missing if the order was filled instantly.
- active - true if the order is still active
- cancelled - true if the order was cancelled
- filled - true if the order was traded

The engine automatically tracks all active orders and peridically update their statuses.

**Example:**

    	...
		stopOrder = mt.addOrder 
  		instrument: instrument
  		side: 'buy'
  		type: 'limit'
  		amount: amount
  		price: instrument.price * 1.05
        if stopOrder.id
        	debug "Order Id: #{stopOrder.id}"
        else
        	debug "Order fulfilled"

##### getActiveOrders()
Returns the list of currently open orders

##### getOrder(orderId)

Returns an order object by given id.

##### cancelOrder(order)

Cancels an order.

##### getTicker(instrument)  (only Live mode)

Returns live ticker data. The object includes two properties: buy and sell that represent current best bid and ask prices respectively.
In backtesting mode the buy and sell are set to the current price.

**Example:**

    mt = require "margin_trading"
    
    handle: ->
      instrument = data.instruments[0]
      info = mt.getMarginInfo instrument
      ## Get best ask price
      ticker = mt.getTicker instrument
      bestAskPrice = ticker.sell
      if mt.buy instrument,info.tradable_balance/instrument.price,bestAskPrice
        debug 'BUY order traded'


##### getOrderBook(instrument) (only Live mode)

Allows to access market depth data for current market. The function returns an object that contains 'asks' and 'bids' fields, each of which is an array of [price,amount] elements representing orders in the orderbook.

**Example:**

	# find the price for which 1 BTC can be bought.

    orderBook = mt.getOrderBook instrument
    volume = 1
    if orderBook
        # logs the whole order book
        for key in ['asks','bids']
            debug "#{key}: #{orderBook[key].join(',')}"
        sum = 0
        for o in orderBook.asks
            sum += o[1]
            if sum >= volume
                debug "#{volume} BTC can be bought for #{o[0]}"
                break
    else
     	debug "orderbook isn't available" 
Note that in backtesting mode the method returns a null value.





