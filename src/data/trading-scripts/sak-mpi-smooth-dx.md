---
title: "SAK-MPI Smooth DX"
description: "Applies advanced digital signal processing filters to the Directional Movement Index (DX)."
fullDescription: "Based on John F. Ehlers' 'Swiss Army Knife' indicator, this script allows traders to smooth the Directional Movement (DX) components (DI+ and DI-) using advanced DSP algorithms. Users can choose from filters like SuperSmoother, Gaussian, Butterworth, BandPass, and more. This results in a cleaner, less noisy representation of trend direction and strength compared to the standard ADX/DMI indicator."
image: "/sak-mpi-smooth-dx.png"
---

```pine
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © fareidzulkifli

// Description : This SwissArmyKnife - MultiPurposeIndicator allows user to modify the Directional index based on one of filtering tools proposed by John F.Ehlers . 
// Details of each filtering type can be read in Ehlers Technical Papers: "Swiss Army Knife Indicator" and/or his book "Cybernetics Analysis for Stock and Futures"

// Disclaimer:
// These study scripts was built only to test/visualize an idea to see its viability and if it can be used to optimize existing strategy.
// This is experimental indicator. Any ideas to further improve this indicator are welcome :)


//@version=4
study("SAK-MPI: Smooth DX", overlay=false)

//************************************************************************************************************//
// Parameter
//************************************************************************************************************//

length      = input(title="Length", type=input.integer, defval=50, minval=1)
type        = input(title="Filter Type", defval="SuperSmoother", options=["SuperSmoother", "Ehlers EMA", "Gaussian", "Butterworth", "High Pass", "2 Pole High Pass", "BandPass", "BandStop", "SMA", "EMA", "RMA"])

//************************************************************************************************************//
// Funtion
//************************************************************************************************************//

//********************************************//
// Ehler SwissArmyKnife Funtion
//********************************************//
SAK_smoothing(_type, _src, _length) =>
    c0          = 1.0 
    c1          = 0.0 
    b0          = 1.0 
    b1          = 0.0 
    b2          = 0.0 
    a1          = 0.0 
    a2          = 0.0 
    alpha       = 0.0 
    beta        = 0.0 
    gamma       = 0.0 
    pi          = 2 * asin(1)
    cycle       = 2 * pi / _length
    
    if _type == "Ehlers EMA"
        alpha   := (cos(cycle) + sin(cycle) - 1) / cos(cycle)
        b0      := alpha
        a1      := 1 - alpha
    if _type == "Gaussian"
        beta    := 2.415 * (1 - cos(cycle))
        alpha   := -beta + sqrt((beta * beta) + (2 * beta))
        c0      := alpha * alpha
        a1      := 2 * (1 - alpha)
        a2      := -(1 - alpha) * (1 - alpha)
    if _type == "Butterworth"
        beta    := 2.415 * (1 - cos(cycle))
        alpha   := -beta + sqrt((beta * beta) + (2 * beta))
        c0      := alpha * alpha / 4
        b1      := 2
        b2      := 1
        a1      := 2 * (1 - alpha)
        a2      := -(1 - alpha) * (1 - alpha)
    if type == "High Pass"
        alpha   := (cos(cycle) + sin(cycle) - 1) / cos(cycle)
        c0      := 1 - alpha / 2
        b1      := -1
        a1      := 1 - alpha
    if type == "2 Pole High Pass"
        beta    := 2.415 * (1 - cos(cycle))
        alpha   := -beta + sqrt((beta * beta) + (2 * beta))
        c0      := (1 - alpha / 2) * (1 - alpha / 2)
        b1      := -2
        b2      := 1
        a1      := 2 * (1 - alpha)
        a2      := -(1 - alpha) * (1 - alpha)
    if type == "BandPass"
        beta    := cos(cycle)
        gamma   := 1 / cos(cycle*2*0.1) // delta default to 0.1. Acceptable delta -- 0.05<d<0.5
        alpha   := gamma - sqrt((gamma * gamma) - 1)
        c0      := (1 - alpha) / 2
        b2      := -1
        a1      := beta * (1 + alpha)
        a2      := -alpha
    if _type == "BandStop"
        beta    := cos(cycle)
        gamma   := 1 / cos(cycle*2*0.1) // delta default to 0.1. Acceptable delta -- 0.05<d<0.5
        alpha   := gamma - sqrt((gamma * gamma) - 1)
        c0      := (1 + alpha) / 2
        b1      := -2 * beta
        b2      := 1
        a1      := beta * (1 + alpha)
        a2      := -alpha
    if _type == "SMA"
        c1      := 1 / _length
        b0      := 1 / _length
        a1      := 1
    if _type == "EMA"
        alpha   := 2/(_length+1)
        b0      := alpha
        a1      := 1 - alpha
    if _type == "RMA"
        alpha   := 1 / _length
        b0      := alpha
        a1      := 1 - alpha   
    _Input        = _src
    _Output       = 0.0
    _Output      := (c0 * ((b0 * _Input) + (b1 * nz(_Input[1])) + (b2 * nz(_Input[2])))) + (a1 * nz(_Output[1])) + (a2 * nz(_Output[2])) - (c1 * nz(_Input[_length])) 

//********************************************//
// SuperSmoother Funtion
//********************************************//

supersmoother(_src, _length) =>
    pi      = 2 * asin(1)
    s_a1    = exp(-sqrt(2) * pi / _length)
    s_b1    = 2 * s_a1 * cos(sqrt(2) * pi / _length)
    s_c3    = -pow(s_a1, 2)
    s_c2    = s_b1
    s_c1    = 1 - s_c2 - s_c3
    ss      = 0.0
    ss     := s_c1 * _src + s_c2 * nz(ss[1], _src[1]) + s_c3 * nz(ss[2], _src[2])

    
//***************************************************************************************************************************************//
// DX Function
//***************************************************************************************************************************************//

DirectionUp             = high-high[1] > low[1]-low ? max(high-high[1], 0): 0
DirectionalDn           = low[1]-low > high-high[1] ? max(low[1]-low, 0): 0

DIPlus  = 0.0
DIMinus = 0.0

if(type != "SuperSmoother")
    DIPlus      := SAK_smoothing(type, DirectionUp, length)
    DIMinus     := SAK_smoothing(type, DirectionalDn, length)

if(type == "SuperSmoother")
    DIPlus      := supersmoother(DirectionUp, length)
    DIMinus     := supersmoother(DirectionalDn, length)


//************************************************************************************************************//
// Drawing
//************************************************************************************************************//
OutputColor   = DIPlus > DIMinus ? #3FE800 : DIPlus < DIMinus ? #FD0000 : color.blue

up = plot(DIPlus, color=#3FE800, title="DI+", linewidth=2)
dn = plot(DIMinus, color=#FD0000, title="DI-", linewidth=2)

fill(up, dn, color=OutputColor)
```
