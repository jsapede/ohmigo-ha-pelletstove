# ohmigo-ha-pelletstove
automate pellet stove using ohmigo and **homeassistant**

This repository aims to describe method to implement automation and Homeassistant inclusion of **a pellet stove that has neither native wifi connection, nor dry contact command entrypoint**, but a wired temperature sensor.

the key idea is to send orders to the pellet using the wired temperature sensor (witch is a kty80 in my case), by mimicking its behaviour using a speecific device that sends variable ressistor values.

# Problem description
My pellet stove has only a simple temperature regulation using a target temperature and a cold hysteresis (witch in my case is set a -1°C). 

i.e. when target temperature is set at 21°C, heating will start when 20°C is sensed on the wired sensor and will stop as soon as the target temperature is reached (21°C). And so on ...

My pellet stove has only two options for heating : 
1. forced heat : the previous described cycle will infinitely run
2. Programmated heat : the previous cycle is infinitely run over a time defined programmation (i.e. hour of start / hour fo end / target temperature)
3. Stop : no heating at all

the main problem is that the pellet stove digital input **allows only 3 time-based schedules** within a day

Moreover, my stove has no way to put a wired/wireles command that could be included in homeassistant, neither an external command through dry contact that also coud be driven by homeassistant

# Approach
as there's no other way to communicate with the pellet stove except reverse engineering the electronics, the key idea is to send orders to the pellet using the wired temperature sensor (witch is a kty80 in my case), by mimicking its behaviour using a speecific device that sends variable ressistor values.

So said, sending a low resistor value will simulate low temperature (< target + hysteresis) and force the stove to burn up, sending a high resistor value will simulate high temperature (> target) and the stove will stop burning.

As a gift, sending corrections to the temperature sensor ( +/- x°C) will allow to simulate variations in the target temperature and so allow much more than 3 time-based schedules.


# Material

The main equipement needed is a Ohmigo "[Ohm on Wifi](https://www.ohmigo.io/en/product-page/ohmigo-ohm-on-wifi)" witch has a specific firmware version including Homeassistant communication by mqtt.

To create a fallaback on the wired sthermal sensor, we also use a simple [zigbee dry-contact with a NC/NO wiring](https://fr.aliexpress.com/item/1005005800957363.html).

A deported wireless zigbee / bluettooth temperature sensor (i use [xiaomi bluetooth](https://fr.aliexpress.com/item/1005006750142144.html) [converted to zigbee](https://smarthomescene.com/guides/convert-xiaomi-lywsd03mmc-from-bluetooth-to-zigbee/)).

Some additionnal WAGO connectors, 5V USB power supply, wires, and electrician pliers are needed to build and the whole system.

# Wiring scheme

# Pellet stove settings

# Code
