# Automated Advanced Furnace Control

## Acknowledgement

This setup originates from and heavily borrows from Barsiel's furnace control scripts that are community go-to scripts for years.

## Burner IC

Controls atmosphere in the combustion chamber of the advanced furnace. This script is specialized for a specific fuel
and should ideally be swapped for a different version tailored to a different fuel.

In any operating mode script actively prevents overpressure. As long as it runs it will force enable output at 40MPa or 55MPa depending on the last loaded minimum pressure. This is to avoid wasting fuel between batches of waspaloy.

TODO: investigate if methane and hydrogen differ significantly to warrant separate scripts with different pre-calculated
shortcuts.

Pumps must feed into a short pipe leading into the furnace gas input. Coolant is expected to be at most 500K while Hot
is expected to be at least 1000K.

As of now hot input is not implemented.

### Inputs

Script runs in compact housing named `FA Burner IC` and takes following stack configuration:

- Index 1: mode.
  - Value 0 is idle.
  - Value 1 is maintain set conditions.
  - Value 2 is purge.
- Index 2 packed target temperature range:
  - 0-23 minimum temperature in K
  - 24-47 maximum temperature in K
- Index 3 packed target pressure range:
  - 0-23 minimum pressure in K
  - 24-47 maximum pressure in K. Maximum pressure is capped at 45 MPA normally but it is increased to 55MPa when minimum
    pressure is at or above 40MPa to accomodate the super alloys.

Lever `FA Purge` overrides mode set by the stack as long as it is pulled. Once it is closed furnace returns to the
defined operating mode.

### Used devices

This script uses devices:
- Optional lever `FA Purge`
- Volume pump `FA FuelMix Pump`
- Pipe analizer `FA FuelMix PA`
- Volume pump `FA Coolant Pump`
- Pipe analyzer `FA Coolant PA`
- Volume pump `FA Hot Pump`
- Pipe analyzer `FA Hot PA`

Fuel mixer:
- Gas mixer `FA Fuel Mixer` with side input opposite power connection used for oxidizer.
- Pipe analizer `FA Fuel PA` for burnable part of the fuel like H2, CH4 or alcohol.
- Pipe analizer `FA Oxidizer PA` for oxidizer part of the fuel N2O, O2, O3.

## Control IC

Script runs in a compact housing named `FA Control IC`. It handles user facing controls such as recipe selection dials
and managing current job and job queue.

It is responsible for setting appropriate `FA Burner IC` stack parameters and ejection of the results. Supply of
reagents is handled by a separate script. 

Split control allows for a better UIX having combustion control moved out and its respective responsiveness demands
reduced. In turn freed capacity is used for a job queue for up to 10 smelt jobs. Queue is on housing stack and can be set
externally as well as via regular furnace controls.

### Used devices

This script uses devices:
- Volume pump `FA FuelMix Pump`
- Pipe analizer `FA FuelMix PA`
- Volume pump `FA Coolant Pump`
- Pipe analyzer `FA Coolant PA`
- Volume pump `FA Hot Pump`
- Pipe analyzer `FA Hot PA`

### Recipe stack

Each ingot recipe gets 10 stacks entries aligned to make dial selection easier by using multiples of 10.

- `+0`: target ingot hash
- `+1`: target ingot batch size
- `+2`: packed min and max temp. 0-23 bits min temperature and 24-47 max temperature,  measured in K
- `+3`: packed min and max pressure. 0-23 bits min pressure and 24-47 max pressure, measured in kPa
- `+4`-`+9`: packed reagent per batch. 0-15 bits for count, 16-47 reagent hash 

Recipe stack is configured by the three fa-recipes scripts, they do not have to run in the same housing and their
execution order affects ingot order on the selection dial. Recipe scripts specifically look for a compact housing named
`FA Control IC` and as such can not be reused elsewhere.

Recipes are batched to align with the ore max stack size 50. Inputs are defined as reagents with separate means required
to map them to the ore or ingots. Supply script will need `d0` and `d1` pins on the housing set to a furnace and to any
printer in order to look up ores and ingots respectively using `rmap`.

Supply/requester script is separated from the control both for the script length and for the pluggability reasons.

### Queue stack

Queue uses stack pointer at index `499` and queue itself at indexes `500` through `511` although `510` and `511` are
reserved for internal operations and must not be used. Add button will be locked when queue is full.

Stack entry is a packed value with 16 bits for count at 0-15 followed by a 32 bits hash at 16-47. Hash must be converted
to a signed value after extraction similarly to the printer stack instructions.

Furnace processes queue in batches as defined by a recipe. This significantly reduces the problem of a large order
getting stuck in its entirety if necessary materials are missing.
Batch size will be substracted from the requested amount and the queue entry updated in place. Furnace will always
produce a full batch even if request is less than that.

## Furnace hotbox

Furnace loses heat to the environment. More so to the surrounding atmosphere but also to the vacuum via radiative heat
exhcange. Current heat preservation method is to put furnace inside a 1x1x1 hotbox room enclosed in walls or frames with
sufficient atmosphere to prevent radiative interactions and stop loss of heat once it reached equilibrium with the
furnace.

Gas in hotbox will heat and expand. To prevent overpressure from popping walls couple active vents named `FA Hotbox Vent
1` and `FA Hotbox Vent 2` inside the hotbox are automatically configured by check-setup script to maintain pressure
between 150kPa and 180kPa. They should be connected to a buffer pipe with inline buffer to store the excess gas while
furnace is at its hottest. With 71-100 mol of gas 1x2 should be plenty under regular conditions.

Initial atmopshere setup is up to you.  
Some reddit post suggests 70.45 mols of gas per 1x1x1 cube is the threshold to fully stop radiative heat loss. If you
feel particularly daring H2 can be used as the atmosphere but generally inert gas that is safe within operating
conditions is much preferred. Mistakes happen. It is best not to cause secondary explosions if hotbox is accidentally
breached. Safest bet is nitrogen followed by carbon dioxide.

My go-to setup method is to add temporary active vent to vacuum the buffer and set one of the hotbox vents to vacuum the
room before sealing it. Once vacuum was achieved the extra vent can be removed and 71-100 mol of gas introduced into the
buffer. Don't forget to use check-setup script to configure hotbox vents afterwards.

For reference, 100mol in a single 10L pipe segment at 273K will be at 22.7 MPa. 20 water ice has exactly 100 mol of
nitrogen.

## Control and UI

Ingot selection dial `FA Ingot Select`, optional memory chip `FA Ingot Select` and hash display console set to read
memory chip. `Mode` setting is configured during recipes load into housing with the help of `fa-recipes-*.ic10`
scripts.
Amount selection dial `FA Ingot Amount`, optional led display `FA Ingot Amount`.
Button (or possibly lever if script is not responsive enough for a button) `FA Ingot Enqueue`.

~~Optional memory chip `FA Target` and a hash display console set to read from it. Value is the currently processed
recipe hash. Not the furnace recipe hash.~~ Not enough lines.
Optional led console `FA Queue Size` shows a number of jobs currently in the queue.
`FA Control IC` housing `Setting` stores hash of the currently loaded order and can be used to connect a hash display.
TBD: Optional led console `FA Furnace Mode` displays statuses `Purge`, `Idle`, `Adjust`, `Holdng`.

Default automatic mode uses queue and requests materials while ejecting results.

Optional lever `FA Manual` enables manual operation mode when pulled. Manual mode does not handle reagents or ejection
and only maintains pressure and temperature using current `FA Ingot Select` setting while ignoring amount dial or
button. After lever was pulled control script waits for the current recipe to finish before engaging manual mode.

Optional led console `FA Mode` reporting statuses `Stdby`, `Next`, `Active`, `Manual`.

In automated mode eject lever cancels current job and prevents next one until lever was closed again. Currently this is
the only way to clear the queue due to lack of spare lines.

Queue follows LIFO order in order to facilitate scheduling smelting of missing steel for Inconel and Astroloy. Queue
shall use pointer and packed entries count + hash. Count shall be corrected upwards towards nearest batch size
breakpoint capping at 500. Match printer instruction?

TODO: handle furnace getting turned off. Currently furnace must stay on. For overpressure purge furnace must be force turned on.

