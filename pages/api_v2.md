
Table of Contents
=================

    * [Overview](#overview)
    * [Trading API](#trading-api)
          * [init](#init)
          * [handle](#handle)
      * [Global data and methods](#global-data-and-methods)
          * [context](#context)
          * [data](#data)
          * [storage](#storage)
          * [stop](#stop)
          * [onRestart](#onrestart)
          * [onStop](#onstop)
      * [Instrument object](#instrument-object)
          * [market](#market)
          * [period](#period)
          * [price](#price)
          * [volume](#volume)
      * [Core Modules](#core-modules)
        * [Logger](#logger)
          * [debug(msg), info(msg), warn(msg)](#debugmsg-infomsg-warnmsg)
        * [Plot](#plot)
          * [plot(series)](#plotseries)
          * [plotMark(marks)](#plotmarkmarks)
          * [setPlotOptions(options)](#setplotoptionsoptions)
        * [_ [lodash library]](#_-lodash-library)
      * [Modules](#modules)
        * [Ta-lib](#ta-lib)
        * [Params](#params)
          * [add title, defaultValue](#add-title-defaultvalue)
        * [Trading (trading)](#trading-trading)
          * [Portfolio object](#portfolio-object)
          * [buy(instrument,type,[amount],[price],[timeout])](#buyinstrumenttypeamountpricetimeout)
          * [sell(instrument,type,[amount],[price],[timeout])](#sellinstrumenttypeamountpricetimeout)
          * [addOrder(order)](#addorderorder)
          * [getActiveOrders](#getactiveorders)
          * [getOrder(orderId)](#getorderorderid)
          * [cancelOrder(order)](#cancelorderorder)
          * [getTicker(instrument)  (only Live mode)](#gettickerinstrument--only-live-mode)
          * [getOrderBook(instrument) (only Live mode)](#getorderbookinstrument-only-live-mode)
        * [Bitfinex Margin Trading (bitfinex/margin_trading)](#bitfinex-margin-trading-bitfinexmargin_trading)
          * [getMarginInfo(instrument)](#getmargininfoinstrument)
          * [getPosition(instrument)](#getpositioninstrument)
          * [closePosition(instrument)](#closepositioninstrument)
          * [buy(instrument,type,[amount],[price],[timeout])](#buyinstrumenttypeamountpricetimeout-1)
          * [sell(instrument,type,[amount],[price],[timeout])](#sellinstrumenttypeamountpricetimeout-1)
          * [addOrder(order)](#addorderorder-1)
          * [getActiveOrders](#getactiveorders-1)
          * [getOrder(orderId)](#getorderorderid-1)
          * [cancelOrder(order)](#cancelorderorder-1)
          * [linkOrder(orderA,orderB)](#linkorderorderaorderb)
          * [getTicker(instrument)  (only Live mode)](#gettickerinstrument--only-live-mode-1)
          * [getOrderBook(instrument) (only Live mode)](#getorderbookinstrument-only-live-mode-1)



## Overview
Welcome to **Cryptotrader.org API**. We aim to provide an API that allows developers to write 
fully featured trading algorithms. Our automated trading platform can backtest and execute trading scripts coded in CoffeeScript on historical data.



---
## Trading API

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

##### period
  trading period in minutes 

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

#### _ [lodash library]
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


	params = require 'params
    STOP_LOSS = params.add 'Stop Loss',100 # default value is 100
    MARKET_ORDER = params.add 'Market Order',false # will be displayed as checkbox
    MODE = params.add 'a) Low risk b) Aggressive','a' # can be a string value
    
---

#### Trading (trading)

Provides an interface to base trading on any exchage supported by the platform

##### Portfolio object
The portfolio object gives access to information about funds on the exchange. During live trading this data gets updated automatically before handle method is called and after orders are traded.

**Example:**
       
    trading = require "trading"
    
    handle: ->
      instrument = @data.instruments[0]
      info "Cash: #{@portfolio.positions[instrument.curr()].amount"
      info "Assets: #{@portfolio.positions[instrument.asset()].amount"





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

Submits a new order and returens an object containing information about order

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

##### getActiveOrders
Returns the list of currently open orders

##### getOrder(orderId)

Returns an order object by given id.

##### cancelOrder(order)

Cancels an order.

##### getTicker(instrument)  (only Live mode)

Returns live ticker data. The object includes two properties: buy and sell that represent current best bid and ask prices respectively.
In backtesting mode the buy and sell are set to the current price.

**Example:**

    mt = require "bitfinex/margin_trading"
    
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



#### Bitfinex Margin Trading (bitfinex/margin_trading)

This module enables leveraged trading on Bitfinex

##### getMarginInfo(instrument)
Returns your trading wallet information for margin trading:
- margin_balance - the USD value of all your trading assets
- tradable_balance - Your tradable balance in USD (the maximum size you can open on leverage for this pair)

**Example:**

    mt = require "bitfinex/margin_trading"
    
    handle: ->
      instrument = data.instruments[0]
      info = mt.getMarginInfo instrument
      debug "price: "margin balance: #{info.margin_balance} tradeable balance: #{info.tradable_balance}"

##### getPosition(instrument)
Returns the active position for specified instrument 

    mt = require "bitfinex/margin_trading"
    
    handle: ->
      instrument = data.instruments[0]
	  pos = mt.getPosition instrument
      if pos
        debug "position: #{pos.amount} @#{pos.price}"

##### closePosition(instrument)
Closes open position

##### buy(instrument,type,[amount],[price],[timeout])
This method executes a purchase of specified asset. 

- type - 'stop' or 'limit' or 'market'
- amount - order amount
- price - order price
- timeout - specifies the length of time in seconds an order can be outstanding before being canceled. 


**Example:**

    mt = require "bitfinex/margin_trading"
    
    handle: ->
      instrument = data.instruments[0]
      info = mt.getMarginInfo instrument
      ## Open long position 
      if mt.buy instrument,info.tradable_balance/instrument.price,instrument.price
        debug 'BUY order traded'

##### sell(instrument,type,[amount],[price],[timeout])
This method executes a sale of specified asset. 

**Example:**

    mt = require "bitfinex/margin_trading"
    
    handle: ->
      instrument = data.instruments[0]
      info = mt.getMarginInfo instrument
      ## Open short position 
      if mt.sell instrument,info.tradable_balance/instrument.price,instrument.price
        debug 'SELL order traded'


**Advanced orders**

This set of functions gives more control over how orders are being processed:

##### addOrder(order)

Submits a new order and returens an object containing information about order

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
		stopOrder = mt.addOrder 
  		instrument: instrument
  		side: 'buy'
  		type: 'stop'
  		amount: amount
  		price: instrument.price * 1.05
        if order.id
        	debug "Order Id: #{stopOrder.id}"
        else
        	debug "Order fulfilled"

##### getActiveOrders
Returns the list of currently open orders

##### getOrder(orderId)

Returns an order object by given id.

##### cancelOrder(order)

Cancels an order.

##### linkOrder(orderA,orderB)
Links orders so when one of them is cancelled or closed, the other one will be cancelled automatically.

##### getTicker(instrument)  (only Live mode)

Returns live ticker data. The object includes two properties: buy and sell that represent current best bid and ask prices respectively.
In backtesting mode the buy and sell are set to the current price.

**Example:**

    mt = require "bitfinex/margin_trading"
    
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

   




