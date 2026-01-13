---
title: "Non-Rescaled RSI"
description: "A variation of the Relative Strength Index with custom scaling and gradient visualization."
fullDescription: "This indicator presents a variation of the classic RSI where the Relative Strength (RS) component is calculated using a non-rescaled approach. It features extensive customization options, including a 'Ratio Cap' to manage extreme values and a sophisticated gradient coloring system (CSS1-CSS4) that visually represents momentum intensity. It also includes dynamic overbought/oversold lines and detailed labels."
image: "/non-rescaled-rsi.png"
---

```pine
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © fareidzulkifli

//@version=4
study("Non-Rescaled RSI", overlay=false)

section1        = input(false,  "═════════ RSI ════════")
src             = input(close,  title="Price Source")
rsilen          = input(14,     title="Length", minval=1)
ob              = input(70,     title="Overbought Line", minval=1, maxval=100)
os              = input(30,     title="Overbought Line", minval=1, maxval=100)
section2        = input(false,  "═════════ Additional Setting ════════")
displaylbl      = input(true,   title="Display RSI Label")
cap             = input(20,     title="Ratio Cap", minval=2, maxval=30)
rsicol          = input("CSS3", title="RSI Color", options=["CSS1","CSS2","CSS3","CSS4"])

// ----------
// Function
// ----------
get_rs(val)=> res = (100/(100-val))-1 //--> rsi = 100 - (100 / (1 + up / down)) <==> up/down = (100/(100-rsi))-1

// ----------
// Calculate
// ----------
rsi  = rsi(src, rsilen)
up   = rma(max(change(src), 0), rsilen)
down = rma(-min(change(src), 0), rsilen)
rs   = up/down

prsi = 
  down == 0 ? cap         : 
  up   == 0 ? -cap        : 
  rs   == 1 ? 0.0         : 
  rs   >  1 ? min(rs,cap) : max(-1/rs,-cap) 

// ----------
// Gradient Color
// ----------
var css1=array.new_color(na), var css2=array.new_color(na), var css3=array.new_color(na), var css4=array.new_color(na)
if barstate.isfirst
    array.push(css1, #80000010), array.push(css1, #8000001f), array.push(css1, #8000002f), array.push(css1, #8000003f), array.push(css1, #8000004f), array.push(css1, #8000005f), array.push(css1, #8000006f), array.push(css1, #8000007f), array.push(css1, #8000008f), array.push(css1, #8000009f), array.push(css1, #800000af), array.push(css1, #800000cf), array.push(css1, #800000df), array.push(css1, #800000ef), array.push(css1, #800000ff), array.push(css1, #008000ff), array.push(css1, #008000ef), array.push(css1, #008000df), array.push(css1, #008000cf), array.push(css1, #008000bf), array.push(css1, #008000af), array.push(css1, #0080009f), array.push(css1, #0080008f), array.push(css1, #0080007f), array.push(css1, #0080006f), array.push(css1, #0080005f), array.push(css1, #0080004f), array.push(css1, #0080003f), array.push(css1, #0080002f), array.push(css1, #0080001f), array.push(css1, #00800010), 
    array.push(css2, #E50003), array.push(css2, #E30D00), array.push(css2, #E12100), array.push(css2, #DF3500), array.push(css2, #DD4800), array.push(css2, #DB5B00), array.push(css2, #D96E00), array.push(css2, #D78000), array.push(css2, #D59200), array.push(css2, #D3A400), array.push(css2, #D1B500), array.push(css2, #CFC600), array.push(css2, #C2CD00), array.push(css2, #AECB00), array.push(css2, #9AC900), array.push(css2, #87C700), array.push(css2, #74C500), array.push(css2, #61C300), array.push(css2, #4FC100), array.push(css2, #3DBF00), 
    array.push(css3, #E50000), array.push(css3, #D8020A), array.push(css3, #CC0414), array.push(css3, #C0071E), array.push(css3, #B40928), array.push(css3, #A80B32), array.push(css3, #9C0E3C), array.push(css3, #901046), array.push(css3, #841250), array.push(css3, #78155A), array.push(css3, #6C1764), array.push(css3, #601A6E), array.push(css3, #541C78), array.push(css3, #481E82), array.push(css3, #3C218C), array.push(css3, #302396), array.push(css3, #2425A0), array.push(css3, #1828AA), array.push(css3, #0C2AB4), array.push(css3, #002DBF), 
    array.push(css4, #002DBF), array.push(css4, #0D38C2), array.push(css4, #1A43C5), array.push(css4, #284EC9), array.push(css4, #3559CC), array.push(css4, #4364CF), array.push(css4, #506FD3), array.push(css4, #5D7AD6), array.push(css4, #6B85D9), array.push(css4, #7890DD), array.push(css4, #869BE0), array.push(css4, #93A6E4), array.push(css4, #A1B1E7), array.push(css4, #AEBCEA), array.push(css4, #BBC7EE), array.push(css4, #C9D2F1), array.push(css4, #D6DDF4), array.push(css4, #E4E8F8), array.push(css4, #F1F3FB), array.push(css4, #FFFFFF), 

clr = 
  rsicol=="CSS1" ? array.get(css1,round(rsi*0.29)) : 
  rsicol=="CSS2" ? array.get(css2,round(rsi*0.19)) : 
  rsicol=="CSS3" ? array.get(css3,round(rsi*0.19)) : 
  rsicol=="CSS4" ? array.get(css4,round(rsi*0.19)) : 
  na

// ----------
// Plot RS/OB/OS line
// ----------
hline(0)
var line line1 = na, line.delete(line1), line1 := line.new(x1 = bar_index, y1 = min(get_rs(ob),cap),     x2 = bar_index - 1, y2 = min(get_rs(ob),cap),     extend = extend.both, color = color.blue,  width = 1, style = line.style_dashed)
var line line2 = na, line.delete(line2), line2 := line.new(x1 = bar_index, y1 = max(-1/get_rs(os),-cap), x2 = bar_index - 1, y2 = max(-1/get_rs(os),-cap), extend = extend.both, color = color.blue,  width = 1, style = line.style_dashed)
var label lbl1 = na, label.delete(lbl1), lbl1  := displaylbl ? label.new(x=bar_index, y=prsi, xloc=xloc.bar_index, text ="RSI: "+tostring(rsi, "#.##")+"\nRelative\nStrength: "+tostring(prsi, "#.##"), color = #00000000, textcolor = clr, size = size.normal, style = label.style_label_left, textalign = text.align_left) : na
plot(prsi, title="RSI", style=plot.style_line, linewidth=2, color = clr)
```
