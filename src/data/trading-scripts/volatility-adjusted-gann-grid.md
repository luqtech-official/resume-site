---
title: "Volatility Adjusted Gann Grid"
description: "A dynamic Gann Grid implementation that adjusts support and resistance levels based on market volatility (ATR)."
fullDescription: "Gann Square of 9 revolves around the idea that price moves in a geometrical pattern. This script solves the rigidity of traditional fixed Gann levels by iterating through multipliers to find a Gann number range that aligns with current market volatility (ATR). This ensures that the support and resistance grid remains relevant and tradable across different timeframes and market conditions, dynamically expanding or contracting as volatility changes."
image: "/volatility-adjusted-grid-gann.png"
---

```pine
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// @fareidzulkifli
// @version=4

study("Volatility Adjusted Grid [Gann]", shorttitle = "Volatility Adjusted Grid", overlay = true, precision=4)

//-----------------------
// Parameter
//-----------------------
source      = close
group       = "Style"
typ1        = "Solid",          typ2   = "Dashed",  typ3 = "Dotted"
ext1        = "Left",           ext2   = "None"
alert1      = "Once per Bar",   alert2 = "Once per Bar Close"

maxline     = input(15,                         title = "No. of S&R Lines",     minval=1,  maxval=20,            tooltip = "No. of each S & R\n20 means 20 Resistance and 20 Support")
atr_size    = input(1.5,                        title = "Range Size",           minval=.1, maxval=10., step=0.1, tooltip = "Smaller Number indicates tighter range\nMin Val=0.1\nMax Val = 10.0")
disp_rguide = input(true,                       title = "Range Guide",                                           tooltip = "Display Range Size guide to find next bigger/smaller range at bottom left of the chart")
in_ext      = input(ext1,                       title = "Line Extension",       group = group, options=[ext1, ext2])
c_style     = input(typ1,                       title = "Cardinal Cross Style", group = group, options=[typ1, typ2, typ3])
o_style     = input(typ3,                       title = "Ordinal Cross Style",  group = group, options=[typ1, typ2, typ3])
g_col       = input(color.new(#606060, 35),     title = "Line Color",           group = group)
alert_typ   = input(alert2,                     title = "Alert Frequency",      group = "Alert", options=[alert1, alert2])

//-----------------------
var cardinal    = c_style == typ1 ? line.style_solid : c_style == typ2 ? line.style_dashed : line.style_dotted
var ordinal     = o_style == typ1 ? line.style_solid : o_style == typ2 ? line.style_dashed : line.style_dotted
var ext_type    = in_ext  == ext1 ? extend.left      : extend.none

//-----------------------
// Generate Gann Square of 9 Sequence
//-----------------------
var table_size  = 1200
var gann_table  = array.new_float()
var val         = 1 
var increment   = 1
var diag_check  = 3

if(barstate.isfirst)
    array.push(gann_table, 1)
    
    for cnt = 0 to table_size-1
        val := val+increment
        array.push(gann_table, val)
        
        if(val==pow(diag_check,2))
            increment   += 1
            diag_check  += 2

//-----------------------
// Funtions
//-----------------------
get_mult(_val, adj)=>
    float initial   = 0.00001
    float res       = initial
    float check     = na
    
    int   cnt_run   = 0
    for i = 0 to 12
        check := adj * (initial * pow(10., i))
        cnt_run := cnt_run+1
        if(_val < check)
            res := 100.0/(initial * pow(10., i))
            break
    
    [res, cnt_run]

get_idx(_src)=>
    int   idx = array.indexof(gann_table, _src)
    float tmp = 0
    int   res = na, int r_idx = na, int s_idx = na
    
    // if src==gann
    if(idx!=-1)
        r_idx := idx+1
    
    // if src != gann
    if(idx==-1)
        for cnt = 0 to (array.size(gann_table)-1)

            tmp := array.get(gann_table, cnt)
            if(tmp>_src and not r_idx)
                r_idx := cnt
            if(r_idx)
                break

    res := r_idx
    
f_line(_yloc, _idx)=>
    line.new(
          x1     = bar_index-50,    x2 = bar_index+10, xloc=xloc.bar_index, 
          y1     = _yloc,           y2 = _yloc, 
          style  = _idx%2==1 ? cardinal : ordinal, 
          extend = ext_type, 
          color  = g_col)

f_label(_txt, _yloc) =>
    label.new(
          x         = bar_index+10, xloc = xloc.bar_index, 
          y         = _yloc, 
          text      = _txt,
          color     = #00000000, 
          textcolor = g_col,
          size      = size.normal, 
          style     = label.style_label_left,
          textalign = text.align_left)
          
//-----------------------
// Main Calculation
//-----------------------
varip int   r_loc       = na                // Keep index of SnR levels array
varip int   s_loc       = na
varip float r1          = na                // Temporary placeholder to check the range size
varip float s1          = na    
varip       r_list      = array.new_float() // Array to keep SnR level intrabar //workaround as varip cant be use with line array
varip       s_list      = array.new_float()
var         g_line      = array.new_line()  // Array for drawing SnR Lines
var         g_lbl       = array.new_label() // Array for drawing SnR Price Labels
varip float curr_size   = na
varip float prev_size   = na
varip int   run_num     = 0                 // Keep track no of execution of main calculation since it may go up to 1000+ iterations per run
varip int   count_run   = 0                 // Keep track no of execution of main calculation since it may go up to 1000+ iterations per run

//-----------------------
float   atr         = sma(tr, 30)
int     lookback    = max(min(bar_index, 20), 1)
float   range       = na
float   range2      = (highest(high, lookback) - lowest(low, lookback)) // Alternative when atr not available
float   mult        = na
float   src         = na
float   min_range   = atr[1] * atr_size

//-----------------------
if(barstate.isnew)          // reset on newbar
    run_num   := 0
    count_run   := 0
    r_loc       := na, r1 := na
    s_loc       := na, s1 := na
    curr_size   := na
    prev_size   := na
    array.clear(r_list)
    array.clear(s_list)

if(barstate.islast and run_num == 0)
    run_num += 1
    
    for i = 0.0 to 4.5 by 0.5   //10 iterations max
        for j = 1 to 10         //10 iterations max
            range       := atr * (j*3)
            [_mul,_run] = get_mult(nz(range, range2), (5-i)) //12 iterations max
            mult        := _mul
            src         := floor(source*mult)

            r_loc       := get_idx(src)
            s_loc       := r_loc-1
            r1          := round_to_mintick(array.get(gann_table, r_loc)/mult)
            s1          := round_to_mintick(array.get(gann_table, s_loc)/mult)
            count_run   := count_run + 1 //+ _run
            curr_size   := r1-s1
            
            if(curr_size >= min_range)  //break loop if found 
                break
        
            prev_size := curr_size
                
        if(curr_size >= min_range)      //break loop if found 
            break
    
    for i=0 to maxline-1
        float r_val = r_loc + i < 0 or r_loc + i > table_size-1 ? na : round_to_mintick(array.get(gann_table, r_loc + i)/mult)
        float s_val = s_loc - i < 0 or s_loc - i > table_size-1 ? na : round_to_mintick(array.get(gann_table, s_loc - i)/mult)

        if(r_val)
            array.push(r_list, r_val)

        if(s_val)
            array.push(s_list, s_val)
    
//-----------------------
// S/R Drawing
//-----------------------
float   _temp  = na
int     r_size  = array.size(r_list)
int     s_size  = array.size(s_list)

if(barstate.islast)
    if(array.size(g_line) > 0)
        for z = 0 to array.size(g_line)-1
            oldline = array.shift(g_line)
            line.delete(oldline)
    
    if(array.size(g_lbl) > 0)
        for z = 0 to array.size(g_lbl)-1
            oldlbl = array.shift(g_lbl)
            label.delete(oldlbl)
            
    if(r_size>0)
        for i=0 to r_size-1
            _temp := array.get(r_list, i)
            array.push(g_line, f_line(_temp, r_loc+i))
            array.push(g_lbl,  f_label("("+tostring(_temp, format.mintick)+")", _temp))

    if(s_size>0)
        for i=0 to s_size-1
            _temp := array.get(s_list, i)
            array.push(g_line, f_line(_temp, s_loc-i))
            array.push(g_lbl,  f_label("("+tostring(_temp, format.mintick)+")", _temp))

//-----------------------
// Alert
//-----------------------

alertup = (crossover(close, r1)  or (close[1] < close  and close==r1))
alertdn = (crossunder(close, s1) or (close[1] > close  and close==s1))

string alerttxt = na

if(alertup)
    alerttxt :="Cross Above "+ tostring(r1, format.mintick)
if(alertdn)
    alerttxt :="Cross Below "   + tostring(s1, format.mintick)

var alert_freq = alert_typ==alert1 ? alert.freq_once_per_bar : alert.freq_once_per_bar_close    

if(alertup or alertdn)
    alert(alerttxt, alert_freq)

//-----------------------
// Table for Range Guideline
//-----------------------
var PTable  = table.new(position = position.bottom_left, columns = 2, rows = 2)
prv_range   = prev_size/atr[1] > 0.1 ? tostring(prev_size/atr[1], "0.00") : "Not Available"
nxt_range   = curr_size/atr[1] < 10 and count_run < 80 ? tostring(curr_size/atr[1], "0.00") : "Not Available"

if(barstate.islast and disp_rguide)
    table.cell(PTable, 0, 0, text="Previous Range: ",   text_color=g_col, text_halign=text.align_left, text_size=size.small)
    table.cell(PTable, 1, 0, text="< " + prv_range,    text_color=g_col, text_halign=text.align_left, text_size=size.small)
    table.cell(PTable, 0, 1, text="Next Range: ",       text_color=g_col, text_halign=text.align_left, text_size=size.small)
    table.cell(PTable, 1, 1, text="> " + nxt_range,     text_color=g_col, text_halign=text.align_left, text_size=size.small)

```
