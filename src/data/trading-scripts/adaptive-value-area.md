---
title: "Adaptive Value Area"
description: "An advanced Support & Resistance indicator using pivot points, volume analysis, and Kalman filters to define adaptive value zones."
fullDescription: "This comprehensive indicator dynamically plots Support and Resistance zones based on pivot high/lows and volume validation. It integrates a sophisticated 'SafeLine' trend system using Kalman filters and SuperSmoother algorithms to reduce noise. Additionally, it features an optimized SuperTrend for trailing stops and includes a built-in Shariah compliance status panel for Malaysian stocks."
image: "/adaptive-value-area.png"
---

```pine
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © fareidzulkifli
// Link for font change: https://lingojam.com/FontChanger

//@version=5
var title_txt='𝐀𝐝𝐚𝐩𝐭𝐢𝐯𝐞 𝐕𝐚𝐥𝐮𝐞 𝐀𝐫𝐞𝐚 [𝐀𝐕𝐀] 𝐕𝟐.𝟎'
indicator('Adaptive Value Area', shorttitle=title_txt, overlay=true)


//-------------------------------------------------
// Library
//-------------------------------------------------
import fareidzulkifli/MasterLibrary/6       as fn
import fareidzulkifli/ShariahLibrary/2      as shariah

//---------
color           red     = #ff1100
color           green   = #00CD16
color           gold    = #FFCD00

//-------------------------------------------------
// Parameters
//-------------------------------------------------
groupsig="Trailing Stop (Optimized SuperTrend)"
outline = input.bool(true,   title="Show Outline", group="𝐀.𝐕.𝐀.")
ts_type = input.string('No', title="Enable Trailing Stop", options=["No","Signal Only", "Signal and Line"], group=groupsig)

//---------
disp_sts = ts_type=="No" ? false : true
disp_stl = ts_type=="Signal and Line"? true : false

groupsnr        =   "Support & Resistance Zone"
disp_snr        =   input.bool(false,   "Display Levels",               group= groupsnr)
nPiv            =   input.int(3,        "Tracker Size",                 group= groupsnr, minval=2,  maxval=7,  step   = 1)
atrLen          =   14
sensitivity     =   input.int(40,       "S/R Sensitivity",              group= groupsnr, minval=10,  maxval=50,  step   = 1)   
mult            =   input.float(0.3,    "S/R Zone Size",                group= groupsnr, minval=0.1, maxval=0.9, step   = 0.1)
vfs             =   input.bool(true,    "Use Volume Filter",            group= groupsnr, inline="Vol")
vlen            =   input.int(20,       "| Volume Length",              group= groupsnr, inline="Vol",  minval=5,  maxval=50,  step   = 1)

left            =   60 - sensitivity
right           =   60 - sensitivity

bullCol         =   input.color(green,  "Swing Low",                    group= "Zone Style")
bearCol         =   input.color(red,    "Swing High",                   group= "Zone Style")
// histCol         =   input.color(color.orange,    "Historical Zone",     group= "Zone Style")
bgTransp        =   input.int(95,       "Zone Background Transparency", group= "Zone Style")
bordTransp      =   input.int(70,       "Zone Border Transparency",     group= "Zone Style")


alert_cross     =   input.bool(true,    "Price Cross Alert",            group= "Alert")
alert_signal    =   input.bool(true,    "Start/End Alert",              group= "Alert")
//-------------------------------------------------
// Variables
//-------------------------------------------------
int             n       = bar_index
float           src     = hl2

string          signal  = na
float           atr     = ta.atr(20)

//------
// S & R
//------
var pivotHigh   =   array.new_box   (nPiv)
var pivotLows   =   array.new_box   (nPiv)  
var highBull    =   array.new_bool  (nPiv)
var lowsBull    =   array.new_bool  (nPiv)
var bullBorder  =   color.new       (bullCol,       bordTransp)
var bullBgCol   =   color.new       (bullCol,       bgTransp)
var bearBorder  =   color.new       (bearCol,       bordTransp)
var bearBgCol   =   color.new       (bearCol,       bgTransp)

//-------------------------------------------------
// Functions
//-------------------------------------------------
f_lim(i_ref,i_pct)=>
    x=(-1*i_ref)/((i_pct/100)-1)

sma(y)=>ta.sma(close, y)

f_supertrend(Factor, Pd, bool pivotpoint=false, int prd=2, int smooth=12)=>
    float Trailingsl = na
    int Trend = 0

    float ph = ta.pivothigh(prd, prd)
    float pl = ta.pivotlow(prd, prd) 
    
    _atr = ta.atr(Pd)
    float TUp = na
    float TDown = na  
    
    if pivotpoint

        // calculate the Center line using pivot points
        var float center = na
        float lastpp = ph ? ph : pl ? pl : na
        if lastpp
            if na(center)
                center := lastpp
                center
            else
                //weighted calculation
                center := (center * 2 + lastpp) / 3
                center
        
        // upper/lower bands calculation
        Up = fn.supersmoother(center - Factor * _atr, smooth)
        Dn = fn.supersmoother(center + Factor * _atr, smooth)
        
        // get the trend
        TUp         := close[1] > TUp[1] ? math.max(Up, TUp[1]) : Up
        TDown       := close[1] < TDown[1] ? math.min(Dn, TDown[1]) : Dn
        Trend       := close > TDown[1] ? 1 : close < TUp[1] ? -1 : nz(Trend[1], 1)
        Trailingsl  := Trend == 1 ? TUp : TDown
    
    color linecolor = na
    linecolor := Trend == 1 and nz(Trend[1]) == 1 ? green : Trend == -1 and nz(Trend[1]) == -1 ? color.new(color.red,100) : na
    
    
    int stsignal = na
    if Trend == 1 and Trend[1] == -1
        stsignal := 1 //buy
    else if Trend == -1 and Trend[1] == 1
        stsignal := 2 //sell

    [Trailingsl, linecolor, stsignal, Trend]


f_klmf(_src)=>
    p = _src
    //value1 = swma(p-p[1])
    float value1= na
    value1 := 0.2*(p-p[1]) + 0.8*nz(value1[1])
    
    float value2 = na
    value2 := 0.1*(ta.tr) + 0.8*nz(value2[1])
    lambda = math.abs(value1/value2)
    alpha = (-lambda*lambda + math.sqrt(lambda*lambda*lambda*lambda + 16*lambda*lambda))/8
    float klmf = na
    klmf := alpha*p + (1-alpha)*nz(klmf[1])


//------
// S & R
//------
// To revisit in next update
diff_hist   = false
_sethistory(_x, _max)=>
    if  array.size(_x) >= _max and diff_hist==true
        _box         = array.get(_x, _max-1)
        // box.set_bgcolor(_box, color.new(histCol, bgTransp))
        // box.set_border_color(_box, color.new(histCol, bordTransp))

del_hist   = 0
_delhistory(_x, _max)=>
    if  array.size(_x) == _max-1 and del_hist==1
        hist_box         = array.pop(_x)
        box.delete(hist_box)


_arrayLoad(_x, _max, _val) =>  
    array.unshift(_x, _val)   
    if  array.size(_x) > _max
        array.pop(_x)

_getBox(_x,_i)   =>
    _box         = array.get(_x, _i)
    _t           = box.get_top(_box)
    _b           = box.get_bottom(_box)
    [_t, _b]
    
_align(_x,_y)    =>
    for i = 0 to array.size(_x)-1
        [_T, _B] = _getBox(_y, 0)
        [_t, _b] = _getBox(_x, i)
        
        if _T > _b and _T < _t or 
           _B < _t and _B > _b or 
           _T > _t and _B < _b or 
           _B > _b and _T < _t
            
            box.set_top (array.get(_y, 0), _t)
            box.set_bottom (array.get(_y, 0), _b)

_extend(_x) =>
    for i = 0 to array.size(_x)-1
        box.set_right(array.get(_x, i), n+20)

//-------------------------------------------------
// S & R Variables
//-------------------------------------------------
v               =   volume
av              =   ta.sma(v, vlen)
vf              =   vfs and v ? (v > (av*1.)) : true
hlc             =   false
shi             =   hlc ? hlcc4 : high
slo             =   hlc ? hlcc4 : low
pivot_high      =   ta.pivothigh (shi, left, right) 
pivot_low       =   ta.pivotlow (slo, left, right) 

patr            =   ta.atr (atrLen)* mult

HH              =   pivot_high + patr
HL              =   pivot_high - patr
LH              =   pivot_low + patr
LL              =   pivot_low - patr

//-------------------------------------------------
// Indicator
//-------------------------------------------------
len1=fn.clamp(n, 1, 30)
len2=fn.clamp(n, 1, 100)
len3=fn.clamp(n, 1, 200)



// SafeLine
//-------------------------------------------------

src1 = f_klmf(close)    // Shortline and Safe line
src2 = f_klmf(low)      //Stopline

if n<=200
    src1 := close
    src2 := low
    

float stopline  = math.min(fn.kalman(src2, int(len3)), fn.supersmoother(fn.kalman(src1, int(len2)), 1))
float safeline  = math.max(fn.supersmoother(fn.kalman(src1, int(len2)), 1), stopline)
float line1     = math.max(fn.supersmoother(fn.kalman(src1, int(len1)), 1), safeline)

//-------------------------------------------------
// Signal
//-------------------------------------------------
[supertrend, stcol, stsignal, sttrend]=f_supertrend(2, 10, true, 2, 15)

valid=safeline>stopline
showsignal=valid and valid[1]

midstart=false
midend  =false

if showsignal and showsignal[1]==false and sttrend==1 and sttrend[1]==1
    midstart:=true

if showsignal==false and showsignal[1]==true //and sttrend==1 and sttrend[1]==1
    midend:=true


var string lastsignal   = na
bool start_signal       = false
bool end_signal         = false

if (stsignal==1 or midstart)
    start_signal    := true
    lastsignal      := "Start"

if (stsignal==2 or midend) and lastsignal=="Start"
    end_signal      := true
    lastsignal      := "End"

//-------------------------------------------------
// Rendering
//-------------------------------------------------
//--------------------------
var blank=#00000000
l0 = plot(hl2,          color=blank, editable=false)
l1 = plot(line1,        color=outline?color.blue    : blank, editable=false)
l2 = plot(safeline,     color=outline?color.green   : blank, editable=false)
l3 = plot(stopline,     color=red, linewidth=2, editable=false)

col(v1, v2, int transp=90, bool neg=false)=>
    clr=#00000000
    if v1>v2
        clr:=color.new(#00bcd4, transp)
    else if v1<=v2 and neg
        clr:=color.new(red, 100)

fill(l0, l3, col(hl2, stopline, 100, true))
fill(l1, l3, col(line1, stopline, 92))
fill(l2, l3, col(safeline, stopline, 70))

//--------------------------
//Supertrend
plot(disp_stl ? supertrend : na, color=(showsignal?stcol:#00000000), linewidth=2)

plotshape(showsignal and disp_sts and           start_signal ? supertrend : na, title='OST Start', text='Start',   location=location.absolute, style=shape.labelup, size=size.tiny, color=color.new(green, 0), textcolor=color.new(color.white, 0))
plotshape((showsignal or midend) and disp_sts and end_signal ? supertrend : na, title='OST End',   text='End',     location=location.absolute, style=shape.labeldown, size=size.tiny, color=color.new(red, 0), textcolor=color.new(color.white, 0))

//--------------------------
// S & R
if pivot_high and vf[right] and disp_snr
    _arrayLoad      (highBull,  nPiv,   false)
    _arrayLoad      (pivotHigh, nPiv,   box.new( n[right]-5,       HH, n+20, HL, 
                                                 xloc           =   xloc.bar_index,
                                                 border_color   =   bearBorder,   
                                                 bgcolor        =   bearBgCol))

if pivot_low and vf[right] and disp_snr
    _arrayLoad      (lowsBull,  nPiv,   true)
    _arrayLoad      (pivotLows, nPiv,   box.new( n[right]-5,       LH, n+20, LL,    
                                                 xloc           =   xloc.bar_index,
                                                 border_color   =   bullBorder, 
                                                 bgcolor        =   bullBgCol))

_align(pivotHigh, pivotHigh)
_align(pivotHigh, pivotLows)
_align(pivotLows, pivotLows)
_align(pivotLows, pivotHigh)

_extend(pivotHigh)
_extend(pivotLows)

//-------------------------------------------------
// Alert
//-------------------------------------------------


alert1  = ta.crossunder(close, line1)
alert2  = ta.crossunder(close, safeline)
alert3  = ta.crossunder(close, stopline)
alert4  = ta.crossover(close, line1)
alert5  = ta.crossover(close, safeline)
alert6  = ta.crossover(close, stopline)

if safeline>stopline and alert_cross
    if alert1
        alert("Cross Below ShortLine", alert.freq_once_per_bar)
    if alert2
        alert("Cross Below SafeLine", alert.freq_once_per_bar)
    if alert3
        alert("Cross Below StopLine", alert.freq_once_per_bar)

    if alert4
        alert("Cross Above ShortLine", alert.freq_once_per_bar)
    if alert5
        alert("Cross Above SafeLine", alert.freq_once_per_bar)
    if alert6
        alert("Cross Above StopLine", alert.freq_once_per_bar)

if alert_signal
    if showsignal and disp_sts and start_signal
        alert("Start Signal", alert.freq_once_per_bar_close)
    if (showsignal or midend) and disp_sts and end_signal
        alert("End Signal", alert.freq_once_per_bar_close)
//----------------------------------
//----------------------------------
// ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
// Info Panel
// ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
group3="Panel"
txt_sz1 = "Large",          txt_sz2  = "Normal",         txt_sz3  = "Small"

disp_ta         = input.bool(true,          group=group3, title = "Display Shariah Panel")
ta_size         = input.string(txt_sz2,     group=group3, title = "  Size:",    options=[txt_sz1, txt_sz2, txt_sz3, "Mobile"])

//----------------------------------
// Info Panel Variables
//----------------------------------
var ta_text_size    = ta_size == txt_sz1 ? size.large : ta_size == txt_sz2 ? size.normal : ta_size == txt_sz3 ? size.small : size.tiny
var t_position      = position.bottom_left
var dark=true
var head_col    = dark ? color.white : color.white
var head_bgc    = dark ? color.black : color.black
var text_col    = dark ? color.white : color.black


//----------------------------------
// Functions
//----------------------------------
fill_ta(_table, _row, _label, _value, _clrtyp) =>
    t_size      = ta_text_size
    header      = _value=="Header" ? true : false
    
    col_text    = header ? head_col : dark? color.gray : color.gray//text_col
    col_bg      = header ? head_bgc : #00000000
    _ValueText  = header ? "" : _value == "gap" ? "" : _value
    
    table.cell(_table, 0, _row, _label, bgcolor = col_bg, text_color = col_text,
      text_size = t_size, text_halign = header ? text.align_left : text.align_left)
    
    table.cell(_table, 1, _row, _ValueText, bgcolor = col_bg, text_color = header?head_col:col_text, 
      text_size = t_size, text_halign = text.align_right)

//----------------------------------
// Generate Table
//----------------------------------
var TA_table = table.new(position = position.bottom_right,
                          columns = 3, 
                          rows = 5, 
                          bgcolor = #00000000, 
                          frame_color = #000000, 
                          frame_width = 0, 
                          border_color = text_col,
                          border_width = 0)


// =====================================================================================================================
// TA Data
// =====================================================================================================================
ticker  = syminfo.type=='stock'?syminfo.tickerid:'NASDAQ:AAPL'
tos     = syminfo.type=='stock'?request.financial(ticker, 'TOTAL_SHARES_OUTSTANDING', "FQ", barmerge.gaps_off, ignore_invalid_symbol =true):-1.0

//Prep panel Data

string  mcap            = str.tostring(tos*close, format.volume) 
var     shariah         = shariah.status() ? "Compliant" : "N/A"

var myx = syminfo.prefix=='MYX' or syminfo.prefix=='MYX_DLY'

irow=-1
if barstate.islast and disp_ta
    if myx
        irow+=1, fill_ta(TA_table, irow, "❄️ Shariah Status",    shariah, 0)
    
    if tos>0 and syminfo.type=='stock'
        irow+=1, fill_ta(TA_table, irow, "🏦 Market Cap",       mcap, 0)

    // irow+=1, fill_ta(TA_table, irow, "📈 MA20>MA50",            trend20, 0)
    // irow+=1, fill_ta(TA_table, irow, "📈 MA50>MA200",           trend50, 0)
    // irow+=1, fill_ta(TA_table, irow, "📊 Relative Volume",  rvol, 0)
    
// bgcol=dailytrend==1?color.new(color.blue, 85):na
// bgcolor(bgcol)


//----------------------------------
// END
//----------------------------------

```
