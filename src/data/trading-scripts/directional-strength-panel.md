---
title: "Directional Strength Panel"
description: "A multi-symbol dashboard displaying directional momentum using RSI of Moving Averages."
fullDescription: "This dashboard allows traders to monitor the directional strength of up to 6 different assets simultaneously. It uses the RSI of a user-selected Moving Average (such as ALMA, SMA, or EMA) to normalize momentum values between 0 and 100. The panel features a heatmap-style color coding to quickly identify overbought and oversold conditions across the watched portfolio."
image: "/directional-strength-panel.png"
---

```pine
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © fareidzulkifli

//@version=4
study(title="Directional Strength Panel", shorttitle="Directional Strength Panel", overlay = true) 

// ---------------------------
// Parameters
// ---------------------------
grp1         = "Table Settings",    grp2    = "Symbol List",     grp3 = "MA Settings"
var lbl_list = array.new_string(),  mom_val = array.new_float()
tips         = "Write the name for label in the second box"
tips2        = "Applicable only for ALMA"
tips3        = "Check to display MA on chart"

in_theme    = input("Dark",             title = "Chart Theme",                      group=grp1, options=["Light", "Dark"])
in_size     = input("Small",            title = "Table Size",                       group=grp1, options=["Small", "Large"])
// -------
ma_typ      = input("ALMA",             title="MA Type",      type=input.string,    group=grp3, options = ["ALMA","SMA","EMA","WMA","RMA","VWMA", "Hull", "SuperSmoother"])
ma_len      = input(20,                 title="MA Length",    type=input.integer,   group=grp3)
offset      = input(0.6,                title="  Offset",     type=input.float,     group=grp3, tooltip=tips2)
sigma       = input(6,                  title="  Sigma",      type=input.float,     group=grp3, tooltip=tips2)
disp        = input(true,               title="Display MA",   type=input.bool,      group=grp3, tooltip=tips3)
// -------
ticker_1    = input('BINANCE:BTCUSDT',  title="Ticker 1",     type=input.symbol,    group=grp2, inline="Ticker 1")                      
ticker_2    = input('BINANCE:ETHUSDT',  title="Ticker 2",     type=input.symbol,    group=grp2, inline="Ticker 2")
ticker_3    = input('BINANCE:BNBUSDT',  title="Ticker 3",     type=input.symbol,    group=grp2, inline="Ticker 3")
ticker_4    = input('BINANCE:XRPUSDT',  title="Ticker 4",     type=input.symbol,    group=grp2, inline="Ticker 4")
ticker_5    = input('BINANCE:ADAUSDT',  title="Ticker 5",     type=input.symbol,    group=grp2, inline="Ticker 5")
ticker_6    = input('BINANCE:DOGEUSDT', title="Ticker 6",     type=input.symbol,    group=grp2, inline="Ticker 6")

if(barstate.isfirst)                                                                                                                        //Index
	array.push(lbl_list, input("BTC",       title="",             type=input.string,    group=grp2, inline="Ticker 1", tooltip=tips))       //0
	array.push(lbl_list, input("ETH",       title="",             type=input.string,    group=grp2, inline="Ticker 2", tooltip=tips))       //1
	array.push(lbl_list, input("BNB",       title="",             type=input.string,    group=grp2, inline="Ticker 3", tooltip=tips))       //2
	array.push(lbl_list, input("XRP",       title="",             type=input.string,    group=grp2, inline="Ticker 4", tooltip=tips))       //3
	array.push(lbl_list, input("ADA",       title="",             type=input.string,    group=grp2, inline="Ticker 5", tooltip=tips))       //4
	array.push(lbl_list, input("DOGE",      title="",             type=input.string,    group=grp2, inline="Ticker 6", tooltip=tips))       //5
// ---------------------------
// Variables
// ---------------------------
var t_position  = position.bottom_right
var col_text    = in_theme  == "Dark" ? color.white : color.black

// -------
var PTable  = table.new(position = t_position, columns = 7, rows = 65)
var dash    = "■", var lim = "-------------"
var row_max = in_size == "Small" ? 20 : 40      // Number of 'dash'
var row_mul = 100/row_max

// ---------------------------
// Functions
// ---------------------------
//Supersmoother 2-pole
f_ss(_src, _length) =>
    s_a1    = exp(-sqrt(2) * math.pi / _length)
    s_b1    = 2 * s_a1 * cos(sqrt(2) * math.pi / _length)
    s_c3    = -pow(s_a1, 2)
    s_c2    = s_b1
    s_c1    = 1 - s_c2 - s_c3
    ss      = 0.0
    ss     := s_c1 * _src + s_c2 * nz(ss[1], _src[1]) + s_c3 * nz(ss[2], _src[2])

// Multiple Moving Average
get_MA(typ, len, _ofs, _sig) =>
    float maVal = na
    if (typ == "EMA")
        maVal := ema(close, len)
    if (typ == "SMA")
        maVal := sma(close, len)
    if (typ == "WMA")
        maVal := wma(close, len)
    if (typ == "RMA")
        maVal := rma(close, len)
    if (typ == "VWMA")
        maVal := vwma(close, len)
    if (typ == "Hull")
        maVal := hma(close, len)
    if (typ == "ALMA")
        maVal := alma(close, len, _ofs, _sig)
    if (typ == "SuperSmoother")
        maVal := f_ss(close, len)
    maVal

// Function to retrieve value for drawing
get_data(_sym, _typ, _len, _ofs, _sig)=>
    security(_sym, timeframe.period, rsi(get_MA(_typ, _len, _ofs, _sig), 14))    // Value should be between 0-100

// Function to update the color of each row
f_Fill(_table, column_i, _title, _val) =>
    for row_i=row_max to 1
        x1   = _val/row_mul, x2 = row_max - x1
        _col = na(_val) ? #00000000 : row_i == row_max ? #FF3333 : x2 > row_i-1 ? #00000000 : color.from_gradient(row_i*row_mul, 10, 100, #00CC00, #FF3333)
        table.cell_set_text_color(_table, column_i, row_i, _col)
        
    table.cell_set_text(_table, column_i, row_max+2, na(_val)? "n/a" : tostring(_val, "0.0"))

// ---------------------------
// Calculate
// ---------------------------
// push value for each ticker to array (same sequence with lbl_list)                Index
array.push(mom_val, get_data(ticker_1, ma_typ, ma_len, offset, sigma))              // 0
array.push(mom_val, get_data(ticker_2, ma_typ, ma_len, offset, sigma))              // 1
array.push(mom_val, get_data(ticker_3, ma_typ, ma_len, offset, sigma))              // 2
array.push(mom_val, get_data(ticker_4, ma_typ, ma_len, offset, sigma))              // 3
array.push(mom_val, get_data(ticker_5, ma_typ, ma_len, offset, sigma))              // 4
array.push(mom_val, get_data(ticker_6, ma_typ, ma_len, offset, sigma))              // 5

// ---------------------------
// Fill Table
// ---------------------------
if(barstate.isfirst)
    // Update first column with 25/50/75/100 levels
    for idx=0 to 3
        table.cell(PTable, 0, (idx*(row_max/4))+1, tostring(100-(25*idx)) + " -", height = 1, text_color = color.new(col_text, 30), text_size = size.tiny, text_halign = text.align_right)
    
    //fill the whole thing 1 time, each subsequent update will only update the color
    for col_i=1 to 6
        _lbl = array.get(lbl_list, col_i-1)
        table.cell(PTable, col_i, 0, lim, height=1.2, width = 2.5, text_color = col_text, text_size = size.tiny, text_halign = text.align_center, text_valign=text.align_center)
        for row_i=row_max to 1
            bg_col = row_i == ((2*(row_max/4))+1) ? color.new(col_text, 95) : #00000000
            table.cell(PTable, col_i, row_i, dash, height = 1.2, width = 0,   text_color = #00000000, text_size = size.normal, text_halign = text.align_center, bgcolor=bg_col)
        table.cell(PTable, col_i, row_max+1, lim,  height=1,     width = 2.5, text_color = col_text,  text_size = size.tiny,   text_halign = text.align_center, text_valign=text.align_center)
        table.cell(PTable, col_i, row_max+2, "",                 width = 0,   text_color = col_text,  text_size = size.small,  text_halign = text.align_center)
        table.cell(PTable, col_i, row_max+3, lim,  height=0.5,   width = 2.5, text_color = col_text,  text_size = size.tiny,   text_halign = text.align_center)
        table.cell(PTable, col_i, row_max+4, _lbl,               width = 2.5, text_color = col_text,  text_size = size.tiny,   text_halign = text.align_center)

if barstate.islast
    //Fill color based on calculated value
    for i=1 to 6
        f_Fill(PTable, i, array.get(lbl_list, i-1),   array.get(mom_val, i-1))

//Plot MA Line
plot(disp ? get_MA(ma_typ, ma_len, offset, sigma) : na, title="Moving Average")

// plot(array.size(lbl_list), title="Lbl_list Size")
// plot(array.size(mom_val), title="mom_val Size")


```
