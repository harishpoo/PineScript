//@version=5
strategy(
    "Crypto & Gold EMA+ADX+RSI Strategy v6 ($10 Auto-Risk + Optional Auto-Trade)",
    overlay=true,
    initial_capital=10,
    default_qty_type=strategy.cash,
    default_qty_value=0,
    pyramiding=1,
    max_labels_count=500
)

// =====================
// USER INPUT: Auto Trade Toggle
// =====================
autoTrade = input.bool(true, "Enable Automatic Trading")

// =====================
// SYMBOL SETTINGS
// =====================
cryptoSymbols = ["BTCUSD", "ETHUSD", "BNBUSDT"]
goldSymbols   = ["XAUUSD"]

isCrypto = array.includes(cryptoSymbols, syminfo.ticker)
isGold   = array.includes(goldSymbols, syminfo.ticker)

// =====================
// GOLD SESSION FILTER
// =====================
sessionActive = true
if isGold
    sessionActive := not na(time(timeframe.period, "0600-1800:1234567"))

// =====================
// INDICATORS
// =====================
ema9   = ta.ema(close, 9)
ema20  = ta.ema(close, 20)
ema50  = ta.ema(close, 50)
ema200 = ta.ema(close, 200)

rsi   = ta.rsi(close, 14)
adx   = ta.adx(14)
atr   = ta.atr(14)
volMA = ta.sma(volume, 20)

// =====================
// DYNAMIC PARAMETERS
// =====================
atrMultiplierTP = isCrypto ? 2.0 : 1.5
atrMultiplierSL = 1.0
adxThreshold    = isCrypto ? 20 : 25
emaGapFactor    = isCrypto ? 0.3 : 0.2

emaGap   = math.abs(ema50 - ema200)
sideways = adx < adxThreshold or emaGap < atr * emaGapFactor

// =====================
// LONG / SHORT CONDITIONS
// =====================
longBase =
    ema50 > ema200 and
    close > ema50 and
    ema9 > ema20 and
    rsi > 50 and
    volume > volMA and
    close > ema9

shortBase =
    ema50 < ema200 and
    close < ema50 and
    ema9 < ema20 and
    rsi < 50 and
    volume > volMA and
    close < ema9 and close < open

longCondition  = longBase and adx > adxThreshold and adx > adx[1] and not sideways and sessionActive
shortCondition = shortBase and adx > adxThreshold and adx > adx[1] and not sideways and sessionActive

longReEntry  = longBase and close <= ema20 and not sideways and sessionActive
shortReEntry = shortBase and close >= ema20 and not sideways and sessionActive

// =====================
// AUTO-RISK & DYNAMIC LEVERAGE
// =====================
startingCapital = 10.0
riskPerTrade    = 1.0
baseLeverage    = 0.1

currentLeverage = baseLeverage * (strategy.equity / startingCapital)

// ATR-based SL & TP
longSL  = close - atr * atrMultiplierSL
shortSL = close + atr * atrMultiplierSL

longTP  = close + atr * atrMultiplierTP
shortTP = close - atr * atrMultiplierTP

// Risk distance & position size
longRiskDistance  = math.abs(close - longSL)
shortRiskDistance = math.abs(close - shortSL)

longPositionSize  = (riskPerTrade / longRiskDistance) * currentLeverage
shortPositionSize = (riskPerTrade / shortRiskDistance) * currentLeverage

// =====================
// EXECUTION (OPTIONAL AUTO-TRADE)
// =====================
if autoTrade
    // LONG
    if longCondition
        strategy.entry("LONG", strategy.long, qty=longPositionSize)
    if longReEntry
        strategy.entry("LONG RE", strategy.long, qty=longPositionSize)

    // SHORt
    if shortCondition
        strategy.entry("SHORT", strategy.short, qty=shortPositionSize)
    if shortReEntry
        strategy.entry("SHORT RE", strategy.short, qty=shortPositionSize)

// EXIT
if strategy.position_size > 0
    strategy.exit("LONG EXIT", stop=longSL, limit=longTP)
if strategy.position_size < 0
    strategy.exit("SHORT EXIT", stop=shortSL, limit=shortTP)

// =====================
// PLOTS
// =====================
plot(ema9,   color=color.yellow, title="EMA 9")
plot(ema20,  color=color.orange, title="EMA 20")
plot(ema50,  color=color.blue,   title="EMA 50")
plot(ema200, color=color.red,    title="EMA 200")

// =====================
// LIVE LABELS ON CHART
// =====================
var label longLabel  = na
var label shortLabel = na

if longCondition or longReEntry
    label.delete(longLabel)
    longLabel := label.new(bar_index, high, 
      text="LONG\nEntry: " + str.tostring(close, "#.2f") +
           "\nSL: " + str.tostring(longSL, "#.2f") +
           "\nTP: " + str.tostring(longTP, "#.2f") +
           "\nQty: " + str.tostring(longPositionSize, "#.6f") +
           "\nLev: " + str.tostring(currentLeverage, "#.2f"),
      color=color.lime, style=label.style_label_up, textcolor=color.black, size=size.small)

if shortCondition or shortReEntry
    label.delete(shortLabel)
    shortLabel := label.new(bar_index, low, 
      text="SHORT\nEntry: " + str.tostring(close, "#.2f") +
           "\nSL: " + str.tostring(shortSL, "#.2f") +
           "\nTP: " + str.tostring(shortTP, "#.2f") +
           "\nQty: " + str.tostring(shortPositionSize, "#.6f") +
           "\nLev: " + str.tostring(currentLeverage, "#.2f"),
      color=color.red, style=label.style_label_down, textcolor=color.white, size=size.small)

// =====================
// ALERTS
// =====================
alertcondition(longCondition, title="LONG ALERT", message="LONG {{ticker}} @ {{close}} | SL/TP via ATR")
alertcondition(shortCondition, title="SHORT ALERT", message="SHORT {{ticker}} @ {{close}} | SL/TP via ATR")
alertcondition(longReEntry, title="RE-ENTRY LONG", message="LONG RE-ENTRY {{ticker}} @ EMA20")
alertcondition(shortReEntry, title="RE-ENTRY SHORT", message="SHORT RE-ENTRY {{ticker}} @ EMA20")

// =====================
// SIGNAL STATUS TABLE
// =====================
var table t = table.new(position.top_right, 3, 2, border_width=1)

trendText =
     ema50 > ema200 ? "BULLISH" :
     ema50 < ema200 ? "BEARISH" : "FLAT"

signalText =
     longCondition ? "LONG" :
     shortCondition ? "SHORT" :
     sideways ? "NO TRADE" : "WAIT"

table.cell(t, 0, 0, "TREND", bgcolor=color.gray)
table.cell(t, 1, 0, "SIGNAL", bgcolor=color.gray)
table.cell(t, 2, 0, "ADX", bgcolor=color.gray)

table.cell(t, 0, 1, trendText)
table.cell(t, 1, 1, signalText)
table.cell(t, 2, 1, str.tostring(adx, "#.##"))
