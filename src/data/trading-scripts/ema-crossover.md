---
title: "EMA Crossover Strategy"
description: "A simple Exponential Moving Average crossover strategy that plots buy and sell signals."
fullDescription: "This strategy utilizes two Exponential Moving Averages (EMAs) to identify trend direction and potential entry points. A buy signal is generated when the short-term EMA crosses above the long-term EMA, indicating a bullish trend. Conversely, a sell signal is triggered when the short-term EMA crosses below the long-term EMA, signaling a bearish trend. The lengths of both EMAs are customizable inputs."
image: "https://placehold.co/600x400/222/eee?text=EMA+Cross"
---

```pine
//@version=5
strategy("EMA Crossover", overlay=true)

shortLen = input.int(9, "Short EMA Length")
longLen = input.int(21, "Long EMA Length")

shortEMA = ta.ema(close, shortLen)
longEMA = ta.ema(close, longLen)

plot(shortEMA, color=color.blue, title="Short EMA")
plot(longEMA, color=color.red, title="Long EMA")

longCondition = ta.crossover(shortEMA, longEMA)
if (longCondition)
    strategy.entry("Long", strategy.long)

shortCondition = ta.crossunder(shortEMA, longEMA)
if (shortCondition)
    strategy.entry("Short", strategy.short)
```
