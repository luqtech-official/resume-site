---
title: "Financial Highlights Fundamentals"
description: "Visualizes key fundamental metrics like Revenue, Net Income, and EPS directly on the price chart."
fullDescription: "This tool overlays critical fundamental data onto the chart, allowing traders to correlate price action with company performance. It supports various metrics including Revenue & Profit After Tax, Net Profit Margin, Gross Profit Margin, Earnings Per Share (EPS), and Dividends. Users can switch between Quarterly and Annual reports and customize the visualization style (bar or line charts) to suit their analysis needs."
image: "/financial-highlights-fundamentals.png"
---

```pine
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// REUSING THIS CODE: You are welcome to reuse this code without permission, including in closed-source publications, as long as proper credits are given :)
// Author: ©fareidzulkifli

// Disclaimer:
// Past performance is not an indicator of future results.
// My opinions are my own and do not constitute financial advice in any way whatsoever. 
// Nothing published by me constitutes an investment/trading recommendation, nor should any data or Content published by me be relied upon for any investment/trading activities.
// I strongly recommends that you perform your own independent research and/or speak with a qualified investment professional before making any financial decisions.

// Any ideas to further improve this indicator are welcome :)

//@version=4
study("Financial Highlights", overlay=false, format=format.volume, precision=2)

//--------------------------------------------
// Parameter
//--------------------------------------------
per1     = "Quarter", per2 = "Annual"
typ1     = "Revenue & PAT", typ2 = "Net Profit Margin", typ3 = "Gross Profit Margin", typ4 = "EPS", typ5 = "Dividend"
loc1     = "Period Ended", loc2 = "Published Date"

per      = input(per1, title="Reporting Period", options=[per1, per2])
type     = input(typ1, title="Report Type", options=[typ1, typ2, typ3, typ4, typ5])
xloc     = input(loc2, title="Plot QR Earnings On: ", options=[loc1, loc2])

txtSet   = input(false, "═════════ Text ════════")
s_txt    = input(true, title="Use Small Text")
lbl_col  = input(color.orange, 'Text Label Color', type=input.color)

lineSet  = input(false, "═════════ Line Chart ════════")
line_col = input(color.orange, 'Line Chart Color', type=input.color)

barSet   = input(false, "═════════ Bar Chart ════════")
rev_col  = input(#ffcf03, 'Revenue Bar Color', type=input.color)
pt1_col  = input(#009900, 'Positive PAT Bar Color', type=input.color)
pt2_col  = input(#ff0000, 'Negative PAT Bar Color', type=input.color)
div_col  = input(#004c99, 'Dividend Bar Color', type=input.color)

//--------------------------------------------
// Variables
//--------------------------------------------

float   line_chart      = na
float   v_rev           = na
float   v_profit        = na
string  s_rev           = na
string  s_profit        = na
string  rev_txt         = na
float   p_nta           = na
string  s_nta           = na
float   p_eps           = na
string  s_eps           = na
float   p_margin        = na
string  s_p_margin      = na
float   gp_margin       = na
string  s_gp_margin     = na
float   p_div           = na
string  s_div           = na
color   profitcolor     = na

var period              = per == per2 or type==typ5 ? "FY" : "FQ"
var earning_ticker      = "ESD_FACTSET:" + syminfo.prefix + ";" + syminfo.ticker + ";EARNINGS"
var bar_width           = timeframe.isweekly ? 15 : timeframe.ismonthly ? 10 : 20

//--------------------------------------------
// Functions
//--------------------------------------------

get_date()=>
    security(earning_ticker, "D", time, lookahead=true) //earning report date

get_fin(_id)=>
    financial(syminfo.tickerid, _id, period, barmerge.gaps_off)

plot_bar(_val, _color)=>
    var line body = na, var line top = na, var line bottom = na
    body    := line.new(x1 = bar_index,  y1 = 0, x2 = bar_index, y2 = _val, color = _color, width = bar_width)

plot_label(_val, _txt, _align)=>
    label txt_lbl = label.new(x = bar_index, y = _val, 
                                  text      = "", 
                                  textalign = _align=="left" ? text.align_left : text.align_center, 
                                  textcolor = lbl_col,
                                  color     = #00000000, 
                                  style     = _val>0 and type!=typ1 and type!=typ5 ? label.style_label_up : label.style_label_down, 
                                  size      = s_txt ? size.small : size.normal)
    label.set_y(txt_lbl, _val)
    label.set_text(txt_lbl, _txt)


//--------------------------------------------
// Main Calculations
//--------------------------------------------
rev     = financial(syminfo.tickerid, "TOTAL_REVENUE", period, barmerge.gaps_on)//

int report_date = na
if(barstate.isconfirmed)
    report_date := get_date()

var _per = type==typ5 ? per2 : per
if((((xloc==loc1 and _per==per1) or _per==per2) and rev) or (_per == per1 and xloc==loc2 and report_date==time))
    if(type==typ1) //Revenue & Profit After Tax
        v_rev           := get_fin("TOTAL_REVENUE")
        v_profit        := get_fin("NET_INCOME")
        s_rev           := tostring(v_rev/1000000, "#,###,###.#")
        s_profit        := tostring(v_profit/1000000, "#,###,###.#")
        rev_txt         := "Rev: "+s_rev+" M \nPAT: "+s_profit+" M"
        profitcolor     := v_profit>0 ? pt1_col : pt2_col
        
        if(v_rev>=v_profit)
            plot_bar(v_rev, rev_col)
            plot_bar(v_profit, profitcolor)
        else
            plot_bar(v_profit, profitcolor)
            plot_bar(v_rev, rev_col)
        plot_label(max(v_rev, v_profit), rev_txt, "left")
    
    else if(type==typ2) //Net Profit Margin
        p_margin    := get_fin("NET_INCOME")/get_fin("TOTAL_REVENUE") * 100
        s_p_margin  := tostring(p_margin, "#.00")+" %"
        line_chart  := p_margin
        plot_label(p_margin, s_p_margin, "center")
        
    else if(type==typ3) //Gross Profit Margin
        gp_margin   := get_fin("GROSS_PROFIT")/get_fin("TOTAL_REVENUE") *100
        s_gp_margin := tostring(gp_margin, "#.00")+" %"
        line_chart  := gp_margin
        plot_label(gp_margin, s_gp_margin, "center")
        
    else if(type==typ4) //Earnings Per Share (EPS)
        p_eps       := get_fin("EARNINGS_PER_SHARE")
        p_eps       := (p_eps ? p_eps : get_fin("NET_INCOME")/get_fin("BASIC_SHARES_OUTSTANDING")) *100 //exception handling
        s_eps       := tostring(p_eps, "0.00#")
        line_chart  := p_eps
        plot_label(p_eps, s_eps, "center")

    else if(type==typ5) //Dividend
        p_div       := get_fin("DPS_COMMON_STOCK_PRIM_ISSUE")
        s_div       := tostring(p_div, "0.00#")
        plot_bar(p_div, div_col)
        plot_label(p_div, s_div, "left")

//--------------------------------------------
// Drawings
//--------------------------------------------

hline(0, linestyle=hline.style_dotted)
plot(line_chart,  title="Value", color=line_col, transp=0, style=plot.style_circles, linewidth=3, join=true)
```
