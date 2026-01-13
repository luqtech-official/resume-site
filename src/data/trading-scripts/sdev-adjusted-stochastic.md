---
title: "SDev Adjusted Stochastic"
description: "A dynamic Stochastic oscillator that adapts its lookback period based on market volatility."
fullDescription: "This adaptive indicator modifies the Stochastic oscillator's lookback period in real-time based on market volatility, measured by the normalized Standard Deviation. The concept, inspired by quantitative analyst Sofien Kaabar, aims to make the oscillator more responsive during high volatility (shortening the period) and smoother during low volatility (lengthening the period), reducing false signals in ranging markets."
image: "/sdev-adjusted-stochastic.png"
---

```pine
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © fareidzulkifli

// Description : This Stochastic variant will auto-adjust stochastic period based on volatility measured by standard deviation. 
// The idea behind it are in highly volatile market, %K period will be reduced to account for recent price range, 
// while in low volatility market, %K period will be increased to account less of the recent price range.

// This idea is based on one of medium article written by Sofien Kaabar with slight modification on the adjusting logic implementation. Any ideas to further improve this indicator are welcome :)

// Disclaimer:
// I always felt Pinescript is a very fast to type language with excellent visualization capabilities, so I've been using it as code-testing platform prior to actual coding in other platform.
// Having said that, these study scripts was built only to test/visualize an idea to see its viability and if it can be used to optimize existing strategy.
// While some of it are useful and most are useless, none of it should be use as main decision maker.

//@version=4
study("SDev Adjusted Stochastic", overlay = false, precision=2)

length  = input(14, title="SDev Period")
Dsmooth = input(3, title="D")
Ksmooth = input(3, title="Smooth")
src     = input(close, title="Source", type=input.source)

//************************************************************************************************************//
// Standard Deviation (Normalized)
//************************************************************************************************************//
ema     = ema(src,length)
sdev    = stdev(ema,length)
hsd     = highest(sdev,length)
lsd     = lowest(sdev,length)
n_sdev  = (sdev-lsd) / (hsd-lsd) *100


//************************************************************************************************************//
// Define Lookback Period 
//************************************************************************************************************//

get_period(x) => int val = round(100+3-x)

//************************************************************************************************************//
// Stochastic 
//************************************************************************************************************//

stochlength = get_period(n_sdev)==0 ? 3 : get_period(n_sdev)

hh          = highest(high,stochlength)
ll          = lowest(low,stochlength)
k           = sma(((src-ll) /(hh-ll) * 100),Ksmooth)
d           = sma(k,Dsmooth)

//************************************************************************************************************//
// Drawing 
//************************************************************************************************************//


plot(k, color=#0094FF, title="%K")
plot(d, color=#FF6A00, title="%D")
h0 = hline(80, "Upper Band", color=#606060)
h1 = hline(20, "Lower Band", color=#606060)
fill(h0, h1, color=#9915FF, transp=80, title="Background")
```
