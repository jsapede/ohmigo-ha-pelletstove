# ohmigo-ha-pelletstove
automate pellet stove using ohmigo and **homeassistant**

This repository aims to describe method to implement automation and Homeassistant inclusion of **a pellet stove that has neither native wifi connection, nor dry contact command entrypoint**, but a wired temperature sensor.

the key idea is to send orders to the pellet using the wired temperature sensor (witch is a kty80 in my case), by mimicking its behaviour using a specific device that sends variable resistor values to the stove.

# Problem description
My pellet stove has only a simple temperature regulation using a target temperature and a cold hysteresis (witch in my case is set a -1°C). 

i.e. when target temperature is set at 21°C, heating will start when 20°C is sensed on the wired sensor and will stop as soon as the target temperature is reached (21°C). And so on ... alternating starts and stops.

My pellet stove has only three settings : 
1. forced heat : the previous described cycle will infinitely run
2. Programmated heat : the previous cycle is infinitely run over a time defined programmation (i.e. hour of start / hour fo end / target temperature)
3. Stop : no heating at all

the main problem is that the pellet stove digital input **allows only 3 time/temperature-based schedules** within a day.

Moreover, my stove has no way to put a wired/wireless command that could be included in homeassistant, neither an external command through dry contact that also coud be driven by homeassistant.

And finally the digital input/output of the stove has only 1°C precision wutch is sometimes limiting.

# Approach
As there's no other way to communicate with the pellet stove except reverse engineering the electronics, the key idea is to send orders to the stove using the wired temperature sensor (witch is a kty80 in my case), by mimicking its behaviour using a speecific device that sends variable ressistor values.

So said, sending a low resistor value will simulate low temperature (< target + hysteresis) and force the stove to burn up, sending a high resistor value will simulate high temperature (> target) and the stove will stop burning.

As a gift, sending corrections to the temperature sensor ( +/- x°C) will allow to simulate variations in the target temperature and so allow much more than 3 time-based schedules.


# Material

The main equipement needed is a Ohmigo "[Ohm on Wifi](https://www.ohmigo.io/en/product-page/ohmigo-ohm-on-wifi)" witch has a specific firmware version including Homeassistant communication by mqtt.

To create a fallaback on the wired sthermal sensor, we also use a simple [zigbee dry-contact with a NC/NO wiring](https://fr.aliexpress.com/item/1005005800957363.html).

A deported wireless zigbee / bluettooth temperature sensor (i use [xiaomi bluetooth](https://fr.aliexpress.com/item/1005006750142144.html) [converted to zigbee](https://smarthomescene.com/guides/convert-xiaomi-lywsd03mmc-from-bluetooth-to-zigbee/)).

Some additionnal WAGO connectors, 5V USB power supply, wires, and electrician pliers are needed to build and the whole system.

# Wiring scheme
On my stove the sensor is wired on 39 and 40 connectors :

![image](https://github.com/user-attachments/assets/67dea1a8-5cb7-47e1-bbee-338b24855a10)

from the motherboard we assume : 
- Grey wire for the "ground"
- red wire for the R<sub>ohm</sub>

so, to get a fallback with the wired sensor in case of problem with the ohmigo : 

- put the red wire of the sensor on the NC connector of the dry contact
- Put the red wire on "Ohm" of the ohmigo on the NO connector of the dry contact
- Put the red wire from the COM on the dry contact to the motherboard
- ling all the grounds in the same WAGO connector

![image](https://github.com/user-attachments/assets/b05c7db0-5ced-4cc0-9a97-6215ef23845e)

dont forget : 
-  to wire the 230V Neutral/Line to the dry contact
-  power supply 5v to the microUSB connector of the ohmigo

# Calibration

once the ohmigo is conneced and included in HA, we need to calibrate it to match with the original wired sensor. 

1. Put ohmigo in "resistance mode"
2. inject a resistance value and see how the stove convert it to temperature
3. repeat the process

![image](https://github.com/user-attachments/assets/6cd6eb96-2fbc-4df2-883e-dad5ba858cf4)

*NOTE : on my stove, only integer values of temperature are reported by the digital input/output. the correct way is to slowly increment/decrement resistor values until a stable temperature level is reached*

once temperature / resistance couples are set on a sufficient temperature range (0-50°C), build an interpolation using linear regression (using excel trend curves for example) :

R<sub>ohm</sub> = f (T<sub>°C</sub>) = a * T<sup>2</sup> + b*T + c

with a,b,c constants.

![image](https://github.com/user-attachments/assets/4cf182bc-dc41-41c5-a43f-b2fb4b4336b1)

we then have a simple mathematical model to inject the correct resistance value in the wired sensor according to the temperature we want.


# Pellet stove settings

The pellet stove is maintained in **"Programmed Mode"** over an extended range of hours in a day, and with a fixed target temperature (21°C) witch will allow : 
- to have a hardware fallback for turning off the heating during the night
- to have a temperature fallbac in case of ohmigo defaillance, reverting to the wired sensor
- to set up a target temperature offset witch will allow to have a control on the real target temperature
- to allow start / stop control using extreme resistor values injected in the sensor


For example : set a time based schedule from 5:00 to 23:00 with a target a 21°C :

-  at 5:00 the stove will start to listen to the wired sensor
-  at 23:00 the stove will stop burning and stop to listen to the wired sensor
-  between 5:00 and 23:00 when temperature will reach Target + hysteresis : 21°C + (-1°C) = 20°C, stove will autostart burning until reaching target (21°C)


# Homeassistant

The basics will be to construct a climate template to control the stove, and add some light automations and script to gain full control over the stove without making "inception like" control loops.

## Helpers

### fake startup switch / stratup boolean (for future use)

### climate template / customize

## Scripts 

### Start and stop scripts

### Target temperature correction script

### ohmigo resistance update script

## Automations

## Lovelace Card




