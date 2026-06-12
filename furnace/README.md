# Automated Advanced Furnace Control

## Acknowledgement

This setup originates from and heavily borrows from Barsiel's furnace control scripts that are community go-to scripts for years.

## Combustion IC

Controls atmosphere in the combustion chamber of the advanced furnace. This script is specialized for a specific fuel
and should ideally be swapped for a different version tailored to a different fuel.

TODO: investigate if methane and hydrogen differ significantly to warrant separate scripts with different pre-calculated
shortcuts.

### Inputs

Script runs in compact housing named `FA Combustion IC` and takes following stack configuration:

- Offset 1 is mode.
  - Value 0 is idle.
  - Value 1 is maintain set conditions.
  - Value 2 is purge.
- Offset 2 is the minimum target temperature in K
- Offset 3 is the maximum target temperature in K
- Offset 4 is the minimum target pressure in kPa
- Offset 5 is the maximum target pressure in kPa, capped at 50MPa

In any operating mode script actively prevents overpressure. As long as it runs it will force enable output at >50MPa

When furnace is in maintain mode and internal atmopshere is within the set conditions `Setting` on the housing is set to 1.

### Used devices

This script uses devices:
- Volume pump `FA FuelMix Pump`
- Pipe analizer `FA FuelMix PA`
- Volume pump `FA Coolant Pump`
- Pipe analyzer `FA Coolant PA`
- Volume pump `FA Hot pump`
- Pipe analyzer `FA Hot PA`

Pumps must feed into short pipe leading into furnace input slot. Coolant is expected to be at most 500K while Hot is
expected to be at least 1000K.

As of now hot input is not implemented.

Potentially this script will also control fuel mixer, in which case it would need following devices:
- Gas mixer `FA Fuel Mixer` with side input opposite power connection used for oxidizer.
- Pipe analizer `FA Fuel PA` for burnable part of the fuel like H2, CH4 or alcohol.
- Pipe analizer `FA Oxidizer PA` for oxidizer part of the fuel N2O, O2, O3.

## Control IC

Handles user facing controls such as recipe selection dials.
It is responsible for setting appropriate `FA Combustion IC` stack parameters, input of reagents and ejection of the results. 

Having combustion controls moved out responsiveness demands and used lines are reduced. Further split might be warranted
to introduce additional functionality like queues and reagent requests.

TODO: Investigate using (small) vending machine as a material buffer. Require materials to be at exactly size 50 and be
present in the vending machine before furnace is set into mode 1 and reagents subsequently vended into the furnace.

## Furnace hotbox

Furnace loses heat to the environment. More so to the surrounding atmosphere but also to vacuum via radiative heat exhcange.
Previously trick was to put furnace inside a welded frame but it no longer prevents radiative cooling.
Current heat preservation method is instead a hotbox: a 1x1x1 room enclosed in walls or frames with sufficient atmosphere
to prevent radiative interactions and stop loss of heat once it reached equilibrium with the furnace.

Gas in hotbox will heat and expand. To prevent overpressure from popping walls couple active vents named 
`FA Hotbox Vent 1` and `FA Hotbox Vent 2` inside the hotbox are automatically configured by Control IC to maintain pressure
between150kPa and 180kPa. They should be connected to a pipe with inline buffer, 1x2 should be sufficient, to store
excess gaswhile furnace is at its hottest.

Some reddit post suggests 70.45 mols of gas is required. If you feel particularly daring H2 can be used as the
atmosphere but generally inert gas that is safe within operating conditions is much preferred. Safest bet is
nitrogen followed by carbon dioxide.

Initial atmopshere setup is up to you. Use IC to temporarily configure one of the vents to vacuum the room. Make sure to
vacuum the buffer pipe. Once that is done fill the buffer with 71 mol of chosen gas then re-seat and turn on the
Control IC so it can re-configure hotbox vents.

For reference, 1x2 inline buffer will have 71mol at ~650kPa for the gas at 0C coming from an ice crusher.


