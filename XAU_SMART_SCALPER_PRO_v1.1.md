# XAU SMART SCALPER PRO v1.1

اندیکاتور اسکالپ طلای جهانی (XAUUSD) با صدور سیگنال روی تایم‌فریم ۵ دقیقه و تأیید روند از آخرین کندل بسته‌شده ۱۵ دقیقه.

## نکات استفاده
- چارت را روی تایم‌فریم **5m** قرار دهید.
- تأیید روند 15m از کندل بسته‌شده قبلی گرفته می‌شود.
- سیگنال پس از بسته‌شدن کندل 5m صادر می‌شود.
- کد زیر Pine Script نسخه 6 است.
- این فایل صرفاً نسخه Markdown کد موجود در مخزن است؛ منطق فایل Pine تغییر داده نشده و کامپایل آن در TradingView در این مرحله انجام نشده است.

## کد Pine Script

```pine
//@version=6
indicator("XAU SMART SCALPER PRO v1.1", overlay=true, max_labels_count=300, max_lines_count=100, max_boxes_count=100)

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// SETTINGS
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
group1 = "Signal Settings"
minScore    = input.int(65, "Minimum Signal Score", minval=50, maxval=95, group=group1)
maxDaily    = input.int(4, "Maximum Daily Signals", minval=1, maxval=10, group=group1)
cooldown    = input.int(6, "Signal Cooldown Bars", minval=1, group=group1)
requireHTF  = input.bool(true, "Require 15M Trend Alignment", group=group1)
oneTrade    = input.bool(true, "One Active Trade At A Time", group=group1)
useSession  = input.bool(true, "Use Session Filter", group=group1)
tradeSession = input.session("0700-1700", "Trading Session (chart timezone)", group=group1)

group2 = "Risk Management"
atrLen = input.int(14, "ATR Period", group=group2)
slMult = input.float(1.3, "SL ATR Multiplier", step=0.1, group=group2)
tp1R   = input.float(1.0, "TP1 Risk Reward", step=0.1, group=group2)
tp2R   = input.float(2.0, "TP2 Risk Reward", step=0.1, group=group2)

group3 = "Indicators"
rsiLen    = input.int(14, "RSI", group=group3)
adxLen    = input.int(14, "ADX", group=group3)
pivotLen  = input.int(3, "Market Structure Pivot", minval=2, group=group3)
volFactor = input.float(0.8, "Relative Volume Factor", step=0.1, group=group3)
fvgMinAtr = input.float(0.1, "Min FVG Size (x ATR)", step=0.05, group=group3)

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// CHART TIMEFRAME + 15M CONFIRMED TREND
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
is5m = timeframe.isminutes and timeframe.multiplier == 5

// Non-repainting: last CLOSED 15m bar values
[htfClose, htfEma50, htfEma200] = request.security(
     syminfo.tickerid, "15",
     [close[1], ta.ema(close, 50)[1], ta.ema(close, 200)[1]],
     lookahead=barmerge.lookahead_on)

htfBull = htfClose > htfEma50 and htfEma50 > htfEma200
htfBear = htfClose < htfEma50 and htfEma50 < htfEma200

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// INDICATORS
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ema9   = ta.ema(close, 9)
ema21  = ta.ema(close, 21)
ema50  = ta.ema(close, 50)
ema200 = ta.ema(close, 200)

vwap = ta.vwap(hlc3)

rsi = ta.rsi(close, rsiLen)
[macdLine, macdSignal, macdHist] = ta.macd(close, 12, 26, 9)
[diPlus, diMinus, adx] = ta.dmi(adxLen, adxLen)

atr = ta.atr(atrLen)

bbBasis = ta.sma(close, 20)
bbDev   = ta.stdev(close, 20) * 2
bbUpper = bbBasis + bbDev
bbLower = bbBasis - bbDev
bbWidth = bbBasis != 0 ? (bbUpper - bbLower) / bbBasis : 0.0
bbExpanding = bbWidth > bbWidth[1]

// Volume (tick volume on forex/gold). If feed has no volume, don't block signals.
volAvg = ta.sma(volume, 20)
volOk  = na(volume) or volume == 0 or volume >= volAvg * volFactor

emaBull = ema9 > ema21 and close > ema50
emaBear = ema9 < ema21 and close < ema50

vwapBull = close > vwap
vwapBear = close < vwap

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// PRICE ACTION
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
body        = math.abs(close - open)
upperWick   = high - math.max(open, close)
lowerWick   = math.min(open, close) - low
candleRange = math.max(high - low, syminfo.mintick)

bullEngulf = close > open and close[1] < open[1] and close >= open[1] and open <= close[1]
bearEngulf = close < open and close[1] > open[1] and close <= open[1] and open >= close[1]

bullPin = lowerWick > body * 2 and close > open and close > low + candleRange * 0.6
bearPin = upperWick > body * 2 and close < open and close < low + candleRange * 0.4

bullPA = bullEngulf or bullPin
bearPA = bearEngulf or bearPin

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// SMART MONEY STRUCTURE
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ph = ta.pivothigh(high, pivotLen, pivotLen)
pl = ta.pivotlow(low, pivotLen, pivotLen)

var float swingHigh = na
var float swingLow  = na

if not na(ph)
    swingHigh := ph
if not na(pl)
    swingLow := pl

bullSweep = not na(swingLow) and low < swingLow and close > swingLow
bearSweep = not na(swingHigh) and high > swingHigh and close < swingHigh

bullBos = not na(swingHigh) and close > swingHigh and close[1] <= swingHigh
bearBos = not na(swingLow) and close < swingLow and close[1] >= swingLow

bullRecentSweep = ta.barssince(bullSweep) <= 8
bearRecentSweep = ta.barssince(bearSweep) <= 8

bullSMC = bullRecentSweep or bullBos
bearSMC = bearRecentSweep or bearBos

// Fair Value Gap (with minimum size so tiny gaps are ignored)
bullFVG = low > high[2] and (low - high[2]) >= atr * fvgMinAtr and close[1] > open[1]
bearFVG = high < low[2] and (low[2] - high) >= atr * fvgMinAtr and close[1] < open[1]

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// FVG + ORDER BLOCK ZONES
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
var box bullFvgBox = na
var box bearFvgBox = na

if bullFVG
    if not na(bullFvgBox)
        box.delete(bullFvgBox)
    bullFvgBox := box.new(bar_index - 2, low, bar_index + 12, high[2],
         bgcolor=color.new(color.green, 88), border_color=color.new(color.green, 45))

if bearFVG
    if not na(bearFvgBox)
        box.delete(bearFvgBox)
    bearFvgBox := box.new(bar_index - 2, low[2], bar_index + 12, high,
         bgcolor=color.new(color.red, 88), border_color=color.new(color.red, 45))

var box bullOB = na
var box bearOB = na

if bullBos and close[1] < open[1]
    if not na(bullOB)
        box.delete(bullOB)
    bullOB := box.new(bar_index - 1, open[1], bar_index + 15, low[1],
         bgcolor=color.new(color.teal, 85), border_color=color.teal)

if bearBos and close[1] > open[1]
    if not na(bearOB)
        box.delete(bearOB)
    bearOB := box.new(bar_index - 1, high[1], bar_index + 15, open[1],
         bgcolor=color.new(color.orange, 85), border_color=color.orange)

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// SCORE: 0 TO 100
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
bullScore =
     (htfBull ? 20 : 0) +
     (bullSMC ? 20 : 0) +
     (bullPA ? 15 : 0) +
     (emaBull ? 8 : 0) +
     (vwapBull ? 7 : 0) +
     (rsi > 50 and rsi < 75 ? 5 : 0) +
     (macdLine > macdSignal ? 5 : 0) +
     (adx > 18 and diPlus > diMinus ? 10 : 0) +
     (volOk ? 5 : 0) +
     (bbExpanding ? 5 : 0)

bearScore =
     (htfBear ? 20 : 0) +
     (bearSMC ? 20 : 0) +
     (bearPA ? 15 : 0) +
     (emaBear ? 8 : 0) +
     (vwapBear ? 7 : 0) +
     (rsi < 50 and rsi > 25 ? 5 : 0) +
     (macdLine < macdSignal ? 5 : 0) +
     (adx > 18 and diMinus > diPlus ? 10 : 0) +
     (volOk ? 5 : 0) +
     (bbExpanding ? 5 : 0)

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// DAILY CONTROL
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
var int dailySignals = 0
var int lastSignalBar = -1000

newDay = timeframe.change("D")
if newDay
    dailySignals := 0

cooldownOk = bar_index - lastSignalBar >= cooldown
sessionOk  = not useSession or not na(time(timeframe.period, tradeSession))

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// SETUPS (conflict-safe: BUY and SELL never fire together)
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
buySetup  = bullSMC and (bullPA or bullFVG) and bullScore >= minScore and (not requireHTF or htfBull)
sellSetup = bearSMC and (bearPA or bearFVG) and bearScore >= minScore and (not requireHTF or htfBear)

buyPick  = buySetup and (not sellSetup or bullScore > bearScore)
sellPick = sellSetup and (not buySetup or bearScore > bullScore)

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// TRADE TRACKING STATE
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
var int   tradeDir   = 0      // 1 = long, -1 = short, 0 = none
var int   tradeStage = 0      // 0 = waiting TP1, 1 = TP1 hit (SL moved to entry)
var int   tradeBar   = -1
var float entryPx = na
var float slPx    = na
var float tp1Px   = na
var float tp2Px   = na

var int cntSL  = 0            // stopped out before TP1
var int cntTP1 = 0            // TP1 hit, then closed at breakeven
var int cntTP2 = 0            // TP2 hit

var line  slLine    = na
var line  tp1Line   = na
var line  tp2Line   = na
var label tradeLabel = na

// Evaluate the active trade on every closed bar AFTER the entry bar.
// Conservative rule: if SL and TP are both touched in the same bar, SL wins.
if tradeDir != 0 and barstate.isconfirmed and bar_index > tradeBar
    if not na(slLine)
        line.set_x2(slLine, bar_index)
    if not na(tp1Line)
        line.set_x2(tp1Line, bar_index)
    if not na(tp2Line)
        line.set_x2(tp2Line, bar_index)

    if tradeDir == 1
        if tradeStage == 0
            if low <= slPx
                cntSL += 1
                tradeDir := 0
            else if high >= tp1Px
                if high >= tp2Px
                    cntTP2 += 1
                    tradeDir := 0
                else
                    tradeStage := 1
                    slPx := entryPx
                    if not na(slLine)
                        line.set_y1(slLine, entryPx)
                        line.set_y2(slLine, entryPx)
        else
            if low <= slPx
                cntTP1 += 1
                tradeDir := 0
            else if high >= tp2Px
                cntTP2 += 1
                tradeDir := 0
    else
        if tradeStage == 0
            if high >= slPx
                cntSL += 1
                tradeDir := 0
            else if low <= tp1Px
                if low <= tp2Px
                    cntTP2 += 1
                    tradeDir := 0
                else
                    tradeStage := 1
                    slPx := entryPx
                    if not na(slLine)
                        line.set_y1(slLine, entryPx)
                        line.set_y2(slLine, entryPx)
        else
            if high >= slPx
                cntTP1 += 1
                tradeDir := 0
            else if low <= tp2Px
                cntTP2 += 1
                tradeDir := 0

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// FINAL SIGNALS
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
canSignal = is5m and barstate.isconfirmed and sessionOk and cooldownOk
     and dailySignals < maxDaily and (not oneTrade or tradeDir == 0)

buySignal  = canSignal and buyPick
sellSignal = canSignal and sellPick

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// SIGNAL LEVELS + DRAWING
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
if buySignal or sellSignal
    dailySignals += 1
    lastSignalBar := bar_index

    if not na(slLine)
        line.delete(slLine)
    if not na(tp1Line)
        line.delete(tp1Line)
    if not na(tp2Line)
        line.delete(tp2Line)
    if not na(tradeLabel)
        label.delete(tradeLabel)

    risk = atr * slMult

    entryPx := close
    slPx    := buySignal ? entryPx - risk : entryPx + risk
    tp1Px   := buySignal ? entryPx + risk * tp1R : entryPx - risk * tp1R
    tp2Px   := buySignal ? entryPx + risk * tp2R : entryPx - risk * tp2R

    tradeDir   := buySignal ? 1 : -1
    tradeStage := 0
    tradeBar   := bar_index

    slLine  := line.new(bar_index, slPx, bar_index + 20, slPx, color=color.red, width=2)
    tp1Line := line.new(bar_index, tp1Px, bar_index + 20, tp1Px, color=color.green)
    tp2Line := line.new(bar_index, tp2Px, bar_index + 20, tp2Px, color=color.lime, width=2)

    scoreText     = buySignal ? bullScore : bearScore
    directionText = buySignal ? "BUY" : "SELL"

    tradeLabel := label.new(bar_index, buySignal ? low : high,
         directionText + "\nScore: " + str.tostring(scoreText) +
         "\nEntry: " + str.tostring(entryPx, format.mintick) +
         "\nSL: " + str.tostring(slPx, format.mintick) +
         "\nTP1: " + str.tostring(tp1Px, format.mintick) +
         "\nTP2: " + str.tostring(tp2Px, format.mintick),
         style=buySignal ? label.style_label_up : label.style_label_down,
         color=buySignal ? color.green : color.red,
         textcolor=color.white)

    alert("XAUUSD " + directionText + " | Score: " + str.tostring(scoreText) +
         " | Entry: " + str.tostring(entryPx, format.mintick) +
         " | SL: " + str.tostring(slPx, format.mintick) +
         " | TP1: " + str.tostring(tp1Px, format.mintick) +
         " | TP2: " + str.tostring(tp2Px, format.mintick), alert.freq_once_per_bar_close)

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// CHART
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
plot(ema9,   "EMA 9",   color=color.yellow)
plot(ema21,  "EMA 21",  color=color.orange)
plot(ema50,  "EMA 50",  color=color.blue, linewidth=2)
plot(ema200, "EMA 200", color=color.gray)
plot(vwap,   "VWAP",    color=color.fuchsia, linewidth=2)

plotshape(buySignal, title="BUY", style=shape.labelup, location=location.belowbar,
     color=color.green, text="BUY", textcolor=color.white, size=size.small)

plotshape(sellSignal, title="SELL", style=shape.labeldown, location=location.abovebar,
     color=color.red, text="SELL", textcolor=color.white, size=size.small)

plotshape(bullSweep, title="Bullish Liquidity Sweep", style=shape.circle,
     location=location.belowbar, color=color.aqua, size=size.tiny)

plotshape(bearSweep, title="Bearish Liquidity Sweep", style=shape.circle,
     location=location.abovebar, color=color.orange, size=size.tiny)

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// DASHBOARD
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
var table dash = table.new(position.top_right, 2, 9, border_width=1)

f_cell(int c, int r, string t, color bg = #1e222d) =>
    table.cell(dash, c, r, t, bgcolor=bg, text_color=color.white, text_size=size.small)

if barstate.islast
    totalClosed = cntSL + cntTP1 + cntTP2
    winRate = totalClosed > 0 ? (cntTP1 + cntTP2) / totalClosed * 100.0 : 0.0
    trendText = htfBull ? "BULLISH" : htfBear ? "BEARISH" : "NEUTRAL"
    trendBg = htfBull ? color.green : htfBear ? color.red : color.gray
    tradeText = tradeDir == 1 ? "LONG" + (tradeStage == 1 ? " (BE)" : "") : tradeDir == -1 ? "SHORT" + (tradeStage == 1 ? " (BE)" : "") : "NONE"

    f_cell(0, 0, "XAU SMART PRO", color.navy)
    f_cell(1, 0, is5m ? "5M ACTIVE" : "USE 5M", is5m ? color.green : color.red)

    f_cell(0, 1, "15M Trend")
    f_cell(1, 1, trendText, trendBg)

    f_cell(0, 2, "BUY Score")
    f_cell(1, 2, str.tostring(bullScore) + "/100")

    f_cell(0, 3, "SELL Score")
    f_cell(1, 3, str.tostring(bearScore) + "/100")

    f_cell(0, 4, "Daily Signals")
    f_cell(1, 4, str.tostring(dailySignals) + "/" + str.tostring(maxDaily))

    f_cell(0, 5, "Active Trade")
    f_cell(1, 5, tradeText)

    f_cell(0, 6, "Closed Trades")
    f_cell(1, 6, str.tostring(totalClosed))

    f_cell(0, 7, "Win Rate")
    f_cell(1, 7, str.tostring(winRate, "#.#") + "%", winRate >= 50 ? color.green : color.maroon)

    f_cell(0, 8, "TP2 / TP1+BE / SL")
    f_cell(1, 8, str.tostring(cntTP2) + " / " + str.tostring(cntTP1) + " / " + str.tostring(cntSL))
```
