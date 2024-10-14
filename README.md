# ohmigo-ha-pelletstove
automate pellet stove using ohmigo and **homeassistant**

This repository aims to describe method to implement automation and Homeassistant inclusion of **a pellet stove that has neither native wifi connection, nor dry contact command entrypoint**, but a wired temperature sensor.

# Glossary :

- ETC : **E**xternal **T**emperature **C**orrection (°C)
- ETS : **E**xternal **T**emperature **S**ensor (°C)
- OTS : **O**hmigo **T**emperature **S**ensor (°C)
- OTR : **O**hmigo **T**emperature **R**esistance (milliOhms)
- RTT : **R**eal **T**arget **T**emperature (°C)
- SHY : **S**tove (cold) **HY**steresis (°C)
- STT : **S**tove **T**arget **T**emperature (°C)
- WTS : **W**ired **T**emperature **S**ensor (°C)

# Problem description
My pellet stove has only a simple temperature regulation using a target temperature (STT) and a cold hysteresis (SHY witch in my case is set a -1°C). 

i.e. when target temperature (STT) is set at 21°C, ignition will start when 20°C is sensed on the wired sensor and will extinct as soon as the target temperature STT is reached (21°C). And so on ... alternating starts and stops :

- when WTS =< (STT + SHY) => ignition
- when (STT + SHY) < WTS < STT => heating
- when WTS >= STT => extinction

My pellet stove has only three settings : 
1. forced heat : the previous described cycle will infinitely run
2. Programmated heat : the previous cycle is infinitely run over a time defined programmation (i.e. hour of start / hour fo end / target temperature)
3. Stop : no heating at all

the main problem is that the pellet stove digital input **allows only 3 time/temperature-based schedules** within a day.

Moreover, my stove has no way to put a wired/wireless command that could be included in homeassistant, neither an external command through dry contact that also coud be driven by homeassistant.

And finally the digital input/output of the stove has only 1°C precision wutch is sometimes limiting.

# Material

The main equipement needed is a Ohmigo "[Ohm on Wifi](https://www.ohmigo.io/en/product-page/ohmigo-ohm-on-wifi)" witch has a specific firmware version including Homeassistant communication by mqtt. The ohmigo generates a resistive value according to a resistance setting in its UI.

To create a fallaback on the wired sthermal sensor, we also use a simple [zigbee dry-contact with a NC/NO wiring](https://fr.aliexpress.com/item/1005005800957363.html). In some cas we will want to go back to the original wired sensor.

A deported wireless zigbee / bluettooth temperature sensor (i use [xiaomi bluetooth](https://fr.aliexpress.com/item/1005006750142144.html) [converted to zigbee](https://smarthomescene.com/guides/convert-xiaomi-lywsd03mmc-from-bluetooth-to-zigbee/)).

Some additionnal WAGO connectors, 5V USB power supply, wires, and electrician pliers are needed to build and the whole system.

#  Method
As there's no other way to communicate with the pellet stove except reverse engineering the electronics, the key idea is to send controled resistance values to the wired sensor throuh the ohmigo.

So said, sending a low resistance value will simulate low temperature (< target + hysteresis) and force the stove to ignite, and sending a high resistance value will simulate high temperature (> target) and the stove will extinct :

*NOTE : from here, as WTS will be replaced by the ohmigo, we will use OTS as the main temperature sensor*

- if OTS = 5°C << (STT + SHY) => ignition
- if OTS = 45°C >> STT => extinction

As a gift, sending corrections (ETC +/- x°C) in addition to en external temperature sensor (ETS) will allow to simulate variations in the target temperature and so allow much more than 3 time-based schedules :

-  set STT = 21°C
-  if ETS >= 20°C ohmigo temperature sensor will generate a value OTS = (ETS + ETC) and ETC = 1°C => OTS = 21°C >= STT => extinction at 20°C instead of 21°C 



# Wiring scheme
On my stove the sensor is wired on 39 and 40 connectors :

![image](https://github.com/user-attachments/assets/67dea1a8-5cb7-47e1-bbee-338b24855a10)

from the motherboard we assume : 
- Grey wire for the "ground"
- red wire for the R<sub>ohm</sub>

so, to get a fallback with the wired sensor in case of problem with the ohmigo : 

- put the red wire of the sensor on the NC connector of the dry contact
- Put the red wire on "Ohm" of the ohmigo on the NO connector of the dry contact
- Put the red wire from the COM on the dry contact to the motherboard of the stove
- Unify all the grounds in the same WAGO connector to the motherboard of the stove

![image](https://github.com/user-attachments/assets/b05c7db0-5ced-4cc0-9a97-6215ef23845e)

dont forget : 
-  to wire the 230V Neutral/Line to the dry contact
-  power supply 5v to the microUSB connector of the ohmigo

![image](https://github.com/user-attachments/assets/f398bff6-7683-4c5c-beab-5917c57262c1)

# Ohmigo settings

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


For example : set a time based schedule from 5:00 to 23:00 with a target (STT) at 21°C + cold hysteresis (SHY) at (-1°C)

-  at 5:00 the stove will start to listen to the wired sensor
-  at 23:00 the stove will stop ignition and stop to listen to the wired sensor
-  between 5:00 and 23:00 when OTS < (STT + SHY) = 20°C, stove will autoignite until reaching OTS >= STT (21°C) and then will autoextinct


# Homeassistant

The basics will be to construct a climate template to control the stove, and add some light automations and script to gain full control over the stove without making "inception like" control loops.

## Sensors

the external temperature sensor (ETS) entity is : `sensor.capteur_salle_a_manger_temperature`


## Helpers

## Target temperature of the stove

to facilitate modifications we introduce in HA an input number that is equalt to the STT target temperature of the stove (i.e. 21°C in our example) : 

so if we decide to change the basic setting of the stove we only have this value to change manually for everything continue to works normally.

## temperature correction offset

for "on the fly" changing the target temperature we need to implement a correction offset on the room temperature sensor (we earlier named WTC) in the form of a "input number" helper : `input_number.correction_sonde_poele`

For example : 

as said, the stove is working with its own target temperature we set a 21°C : 

if we want a lower real target temperzature (RTT) we have to tell sthe stove that it reaches its own target earlier, so if we want 20°C instead of 21°C we have to increase the value of the temperature sensor `sensor.capteur_salle_a_manger_temperature` with an offset `input_number.correction_sonde_poele`witch is set at the difference between the stove target temperature (21°C) and our real own target temperature (20°C) :

`input_number.correction_sonde_poele` = (STT - RTT) = 21°C - 20°C = 1°C

ohmigo Temperature value will be OTS = `sensor.capteur_salle_a_manger_temperature` + `input_number.correction_sonde_poele`

so each time we will change the target temperature on the climate, we will have to change this offset. this is done with an automation and a script described later in ths document.


### fake startup switch / stratup boolean (for future use)

to build correctly a climate template from the UI and be able to correct its target temperazutres from the UI we need to have a switch that ignite/extinct the stove.

In reality we only want that the stove regulates itself ignition/extinction so we create a "fake ignition switch" 

```
####################
# Poele a granulés #
####################

switch:
## Demarrage chauffe thermostat poele a granules / activation switch virtuel
  - platform: template
    switches:
      poele_a_granules_chauffe:
        unique_id: "switch.poele_a_granules_chauffe"
        friendly_name: "Poele mode marche CHAUFFE"
        value_template: "{{ is_state('input_boolean.poele_a_granules_chauffe_virtuel', 'on') }}"
        turn_on:
          action: switch.turn_on
          target:
            entity_id: input_boolean.poele_a_granules_chauffe_virtuel
        turn_off:
          action: switch.turn_off
          target:
            entity_id: input_boolean.poele_a_granules_chauffe_virtuel
```

this fake ignition switch, only change the state of a virtual input boolean : `input_boolean.poele_a_granules_chauffe_virtuel` witch has no impact a this time.

*NB : for future use it wille be possible to add extra automation to force ignit/extinction from this virtual input boolean*


### climate template / customize



## Scripts 

### Start and stop scripts
here we use two extremal values of resistance to force the stove to ignite or to force it to extinct.

Forced ignition : we send 5°C to the stove with OTS (i.e. OTR = 849000 milliohm according to my calibration)

this one is **useless at this time**. Will be useful if we want to completely manage the stove from the climate template

```
alias: Demarrage chauffe poele FORCEE
sequence:
  - action: mqtt.publish
    metadata: {}
    data:
      topic: aha/18fe34ed492b/oowifi_resistance/cmd_t
      payload: "849000"
description: ""
```
Forced extinction : we send 45°C to the stove with OTS (i.e. OTR =  1161000 milliohms according to my calubration)

```
alias: Arret chauffe poele FORCE
sequence:
  - action: mqtt.publish
    metadata: {}
    data:
      topic: aha/18fe34ed492b/oowifi_resistance/cmd_t
      payload: "1161000"
description: ""
```

### Target temperature correction script

### ohmigo resistance update script
we use the mathematical model we built during calibration and introduce an offset correction to manipulate the real target temperature of the stove and convert OTS to OTR (resistance in milliohms)

so in my case it will be : **OTR = ((0.0179 * OTS^2) + (6.9597 * OTS) + 813.1) * 1000**

*NB : as the ohmigo only accepts integer values on the mqtt command, we need to multiply by 1000 and round(0) before sending the value*

```
alias: update resistance ohmigo
sequence:
  - action: mqtt.publish
    metadata: {}
    data:
      retain: false
      topic: aha/18fe34ed492b/oowifi_resistance/cmd_t
      payload: >-
        {{ ((0.0173 * (((states('sensor.capteur_salle_a_manger_temperature') |
        float) + (states('input_number.correction_sonde_poele') | float) )**2) +
        6.9597 * ((states('sensor.capteur_salle_a_manger_temperature') | float)
        + (states('input_number.correction_sonde_poele') | float) )+ 813.1)
        *1000) | round(0) }}
      evaluate_payload: false
description: ""
```

## Automations

## Lovelace Card



