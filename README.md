# ohmigo-ha-pelletstove

This repository aims to describe method to implement automation and Homeassistant inclusion of **a pellet stove that neither has native wifi connection, nor dry contact command entrypoint**, but a only Wired Temperature Input (WTI).

# Glossary :

- ETC : **E**xternal **T**emperature **C**orrection (°C)
- ETS : **E**xternal **T**emperature **S**ensor (°C)
- OTI : **O**hmigo **T**emperature **Input** (°C)
- ORI : **O**hmigo **R**esistance **Input** (milliOhms)
- RTT : **R**eal **T**arget **T**emperature (°C)
- SHY : **S**tove (cold) **HY**steresis (°C)
- STT : **S**tove **T**arget **T**emperature (°C)
- WTI : **W**ired **T**emperature **I**nput (°C)

# Problem description
My pellet stove has only a simple temperature regulation based on a Stove Target Temperature (STT) and a Stove cold HYsteresis (SHY witch in my case is set a -1°C). 

When STT is set at 21°C, ignition will start when the WTI sends 20°C and will extinct as soon as WTI sends temperature over STT (21°C) ... and so on ... alternating starts and stops.

- when WTI =< (STT + SHY) : ignition
- when (STT + SHY) < WTI < STT : heating
- when WTI >= STT : extinction

My pellet stove has only three settings : 
1. forced heat : the previous described cycle will infinitely run
2. Programmated heat : the previous cycle is infinitely run over a time defined programmation (i.e. hour of start / hour fo end / target temperature)
3. Stop : no heating at all

# Material

The main equipement needed is a Ohmigo "[Ohm on Wifi](https://www.ohmigo.io/en/product-page/ohmigo-ohm-on-wifi)" witch has a specific firmware version including Homeassistant communication by MQTT. The ohmigo generates a resistive value according to a resistance setting in its UI. Ohmigo will replace the WTI.

A deported wireless zigbee / bluettooth temperature sensor (i use [xiaomi bluetooth](https://fr.aliexpress.com/item/1005006750142144.html) [converted to zigbee](https://smarthomescene.com/guides/convert-xiaomi-lywsd03mmc-from-bluetooth-to-zigbee/)) will act as the External Temperature Sensor (ETS)

To create a fallaback on the wired sthermal sensor, we also use a simple [zigbee dry-contact with a NC/NO wiring](https://fr.aliexpress.com/item/1005005800957363.html). In some cases we will want to go back to the original wired sensor.

Some additionnal WAGO connectors, 5V USB power supply, wires, and electrician pliers are needed to build and the whole system.

#  Method
As there's no other way to communicate with the pellet stove except reverse engineering the electronics, the key idea is to delude the stove with fake temperarature inputs (OTI) sent by variating the resistance of the Ohmigo and so, to send controled resistance inputs (ORI) to the stove

So said, sending a low resistance value will simulate low temperature (<= target + hysteresis) and force the stove to ignite, and sending a high resistance value will simulate high temperature (>= target) and the stove will extinct.

The main objective is to replace WTS by a controlled input value issuing from the ohmigo (ORI).

*NOTE : as we will replace the WTI by the ohuigo, from here we will use OTI as the main temperature input for the stove*

- if OTI = 5°C << (STT + SHY) : ignition
- if OTI = 45°C >> STT : extinction

*NOTE : you'll have to adapt thoses limits to your case*

As a gift, the Ohmigo control will allow us to relay external temperature sensor (ETS) to the stove and add some corrections (ETC +/- x°C) to simulate variations in the target temperature and so allow great flexibility on the stove scheduling.

For example :
-  on the stove we set STT = 21°C as a basis
-  if ETS >= 20°C ohmigo will generate a fake temperature including the desired correction OTI = (ETS + ETC) with, for example, ETC = 1°C. So we will have OTI = 21°C >= STT. Shis will lead to control over extinction at 20°C instead of 21°C of the STT
-  ETC qill be the difference between the Real Target Temperature (RTT) and the internal Stove Target Temperature (STT) 

![image](https://github.com/user-attachments/assets/d0bda2c6-aef3-433a-8696-2a9d4fa516ec)


# Wiring scheme
On my stove the sensor is wired on 39 and 40 connectors :

![image](https://github.com/user-attachments/assets/67dea1a8-5cb7-47e1-bbee-338b24855a10)

from the motherboard we assume : 
- Grey wire for the "ground"
- red wire for the R<sub>ohm</sub>

so, to wire the ohmigo and keep a fallback with the wired sensor in case of problem : 

- put the red wire of the sensor on the NC connector of the dry contact
- Put the red wire on "Ohm" of the ohmigo on the NO connector of the dry contact
- Put the red wire from the COM on the dry contact to the motherboard of the stove
- Unify all the grounds in the same WAGO connector to the motherboard of the stove

![image](https://github.com/user-attachments/assets/b05c7db0-5ced-4cc0-9a97-6215ef23845e)

dont forget : 
-  to wire the 230V Neutral/Line to the dry contact
-  power supply 5v to the microUSB connector of the ohmigo

![image](https://github.com/user-attachments/assets/f398bff6-7683-4c5c-beab-5917c57262c1)

*NOTE: Fallaback on the wired sensor in't necessary*

# Ohmigo settings

## Setup
Ohmigo runs modified Tasmota. You'll have to connect it to WIFI and setup MQTT communication.

![image](https://github.com/user-attachments/assets/0755d749-7435-493b-88fa-2c3bef413303)

Ohmiigo will be set on "Operating mode : Resistance"

![image](https://github.com/user-attachments/assets/d90d98fa-1a35-4eb6-8aa6-62f118187cc0)

*NOTE : Ohmigo can also be set in "temperature" mode and use hardwired models of temperature sensors. this case is not covered by this method. Adapt to your case*

Once setup, you'il be able to see it in HA : 

![image](https://github.com/user-attachments/assets/49bb2e49-f27e-4d16-ac03-8fdafc0239ba)

*NOTE : As we use "Resistance Mode" temperature change on the UI has no effect on the ohmigo*

and with MQTTExplorer : 

![image](https://github.com/user-attachments/assets/39b4a976-554f-4f2c-baaa-b59fc6ddb2bf)

## Control
Ohmigo can be manually controlled via the input number on the MQTT integration and throuh the input entity created by the MQTT connection

**However, for the rest of this tutorial, we will directly use MQTT commands**

refering to the MQTTExplorer screenshot above, sending new resistance values to the ohmigo (ORI) to delude the stove will be done using the MQTT topic : **aha/18fe34ed492b/oowifi_resistance/cmd_t**

**This command MUST be fed with milliohms values, with integer precision. Ohmigo will convert them into ohm values with 3 digits precition as seen on the state topic (aha/18fe34ed492b/oowifi_resistance/stat_t)**

## Calibration

Once the ohmigo is conneced and included in HA, we need to calibrate it to match with the original wired sensor : 

1. Ohmigo is set in "resistance mode"
2. input a resistance value in the MQTT UI and see how the stove convert it to temperature value
3. repeat the process over a large temperature range

![image](https://github.com/user-attachments/assets/6cd6eb96-2fbc-4df2-883e-dad5ba858cf4)

*NOTE : on my stove, only integer values of temperature are reported by the digital input/output. I slowly increment/decrement resistance value until a stable temperature level is reached and then take it as reference*.

Once temperature / resistance couples are set on a sufficient temperature range (0-50°C), build a function ORI = f(OTI) using linear regression (using excel trend curves for example) :

R<sub>ohm</sub> = f (T<sub>°C</sub>) = a * T<sup>2</sup> + b * T + c

with a,b,c constants.

**IN MY CASE** i have :
- a = 0.0173
- b = 6.9597
- c * 813.1

*NOTE : i used quadratic equation as it gives perfect consistency (R<sup>2</sup>=1). You have to adapt to your case*

![image](https://github.com/user-attachments/assets/4cf182bc-dc41-41c5-a43f-b2fb4b4336b1)

Now, we then have a simple mathematical model to inject the correct resistance (ORI) value in the wired sensor according to the temperature we want to send (OTI).

**IN MY CASE converting the model to give milliohms** to Ohmigo will lead to : ORI = ((0.0173 * OTI<sup>2</sup>) + (6.9597 * OTI) + 813.1) * 1000

**YOU WILL HAVE TO ADAPT YOUR FORMULA AND PARAMETERS ACCORDING TO THE RESULTS OF YOUR CALIBRATION**


# Pellet stove settings

We have to setup the stove in a state where he will listen to his sensor to ignite / heat / extinct.

I put the pellet stove in **"Programmed Mode"** over an extended range of hours in a day, and with a fixed target temperature (STT = 21°C) witch will allow : 
- to have a hardware fallback for extinct during the night
- to have a target temperature fallback in case of ohmigo defaillance, reverting to the wired sensor
- to set up a target temperature correction witch will allow to have a control on the real target temperature
- to allow start / stop control using extreme resistor values injected in the sensor

For example : set a time based schedule from 5:00 to 23:00 with a target (STT) at 21°C + cold hysteresis (SHY) at (-1°C)

-  at 5:00 the stove will start to listen to the wired sensor
-  at 23:00 the stove will stop ignition and stop to listen to the wired sensor
-  between 5:00 and 23:00 when OTS <= (STT + SHY) = 20°C, stove will autoignite until reaching OTS >= STT (21°C) and then will autoextinct

**With this mode, the stove will keep the control on the ignition sequences according to its own hysteresis (SHY), so yu will have to adapt STT and SHY according to your case**

**It's possible to get full control of the stove igntion sequence by injecting start / stop orders to the stove. Climate thermostat created below will have to be adjusted**


# Homeassistant

The basics will be to construct a climate template to control the stove, and add some light automations and script to gain full control over the stove without making "inception like" control loops.

## Sensors

the external temperature sensor (ETS) issues from the xiaomi MQTT connection.

Entity is set at : `sensor.ets`


## Helpers

### Target temperature of the stove

to facilitate modifications we introduce in HA an input number that is equalt to the STT target temperature of the stove (i.e. 21°C in our example).

Entity is set at : `intput_number.stt`

in my case value is set at 21°C according to the stove STT. if we decide to change the basic setting of the stove we only have this value to change manually for everything continue to works normally.

### temperature correction offset

for "on the fly" changing the target temperature target we need to implement a correction offset on the external temperature sensor (we earlier named ETC) in the form of a "input number" helper :

Entity is set at : `input_number.etc`

For example : 

as said, the stove is working with its own target temperature we set a 21°C : 

if we want a lower real target temperature (RTT) we have to tell sthe stove that it reaches its own target earlier, so if we want 20°C instead of 21°C we have to increase the value of the temperature sensor `sensor.ETS` with an offset `input_number.ETC` witch is set at the difference between the stove target temperature (21°C) and our real own target temperature (20°C) :

`input_number.ETC` = (STT - RTT) = (21°C - 20°C) = 1°C

ohmigo Temperature value will be OTI = `sensor.ETS` + `input_number.etc`

so each time we will change the target temperature on the climate, we will have to change this offset. this is done with an automation and a script described later in ths document.


### fake startup switch / stratup boolean (for future use)

to build correctly a climate template from the UI and be able to correct the Real Target Temperautre (RTT) from the UI we need to have a switch that mimmick ignite/extinct of the stove.

In reality we only want that the stove regulates itself ignition/extinction so we create a "virtual ignition binary" 

Entity of this virtual input is set at : `input_boolean.virtual_stove_ignition`

```
switch:
  - platform: template
    switches:
      stove_ignition:
        friendly_name: "Pellet Stove Ignition"
        value_template: "{{ is_state('input_boolean.virtual_stove_ignition', 'on') }}"
        turn_on:
          action: switch.turn_on
          target:
            entity_id: input_boolean.virtual_stove_ignition
        turn_off:
          action: switch.turn_off
          target:
            entity_id: input_boolean.virtual_stove_ignition
```

This fake ignition switch, only change the state of a virtual input boolean : `input_boolean.virtual_stove_ignition` witch has no impact a this time.

*NB : for future use it wille be possible to add extra automation to force ignit/extinction from this virtual input boolean*


### climate template / customize

through the UI we generate a helper with **climate** entity : `climate.stove` 

![image](https://github.com/user-attachments/assets/3a6cd3c1-5ce0-44b9-9a0e-69c0265d88ee)

and set :
-  Temperature sensor : `sensor.ets`
-  Switch : `switchs.tove_ignition`

thent we set the temperature preset modes in the UI : 

![image](https://github.com/user-attachments/assets/12a58bf7-3bc9-4027-8e1d-e2c1d7e6740b)

**Precisions and other climate settings aren't available on the UI, so i set them in the configuration.yaml**

```

homeassistant:
  customize:
    climate.stove:
      #initial_hvac_mode: "off"
      precision: 0.1
      target_temp_step: 0.5
```


## Scripts 

### Start and stop scripts
here we use two extremal values of resistance to force the stove to ignite or to force it to extinct.

Forced ignition : we send OTI = 5°C to the stove (i.e. ORI = 849000 milliohm according to my calibration)

this one is **useless at this time**. Will be useful if we want to completely manage the stove from the climate template

```
alias: Stove Ignition
sequence:
  - action: mqtt.publish
    metadata: {}
    data:
      topic: aha/18fe34ed492b/oowifi_resistance/cmd_t
      payload: "849000"
description: ""
```
Forced extinction : we send OTI = 45°C to the stove (i.e. ORI =  1161000 milliohms according to my calubration)

```
alias: Stove Extinction
sequence:
  - action: mqtt.publish
    metadata: {}
    data:
      topic: aha/18fe34ed492b/oowifi_resistance/cmd_t
      payload: "1161000"
description: ""
```

### Target temperature correction script

Everytime the climate temperature settings (RTT) change we will have to adapt the temperature correction (ETC), so we create a light script thaw will be called by automation

```
action: input_number.set_value
metadata: {}
data:
  value: >-
    {{ (states('input_number.stt') | float -
    state_attr('climate.stove', 'temperature') | float) | round(1)  }}
target:
  entity_id: input_number.etc
```

### ohmigo resistance update script
We use the mathematical model we built during calibration and introduce an offset correction to manipulate the real target temperature of the stove and convert OTI to ORI (resistance in milliohms)

so in my case it will be : **ORI = ((0.0173 * OTI<sup>2</sup>) + (6.9597 * OTI) + 813.1) * 1000**

with OTI = `sensor.ets + input_number.etc`

*NB : as the ohmigo only accepts integer values in millihoms on the mqtt command, we need to multiply by 1000 and round(0) before sending the value*

```
alias: update resistance ohmigo
sequence:
  - action: mqtt.publish
    metadata: {}
    data:
      retain: false
      topic: aha/18fe34ed492b/oowifi_resistance/cmd_t
      payload: >-
        {{ ((0.0173 * (((states('sensor.ets') |
        float) + (states('input_number.etc') | float) )**2) +
        6.9597 * ((states('sensor.ets') | float)
        + (states('input_number.etc') | float) )+ 813.1)
        *1000) | round(0) }}
      evaluate_payload: false
description: ""
```

## Automations

### update Ohigo resistance (ORI) 

Here we update the Ohmigo resistance value (ORI) each time the external temperature sensor (ETS) value changes :

```
alias: update ohmigo resistance
description: ""
triggers:
  - trigger: state
    entity_id:
      - sensor.ets
conditions: []
actions:
  - action: script.turn_on
    metadata: {}
    data: {}
    target:
      entity_id:
        - script.update_resistance_ohmigo
mode: single
```

### Coupling stove extinction with thermostat 

Here we establish coupling between thermostate settings and the stove control :

- When Thermostat is set to OFF : Set OTI to 45°C to force stove stop + deactivate ORI update to avoid restarts + stop the scheduler
- when thermostat is send to "HEAT" :  activate ORI update + force ORI update according to RTT set on the thermostat

```
alias: Gestion Thermostat Poele
description: ""
triggers:
  - alias: Démarrage thermostat
    trigger: state
    entity_id:
      - climate.stove
    from: "off"
    to: heat
    id: demarrage thermostat
  - alias: arret thermostat
    trigger: state
    entity_id:
      - climate.stove
    from: heat
    to: "off"
    id: arret thermostat
conditions: []
actions:
  - choose:
      - conditions:
          - condition: trigger
            id:
              - arret thermostat
        sequence:
          - action: automation.turn_off
            data:
              stop_actions: true
            target:
              entity_id: automation.update_ohmigo_resistance
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.stove_extinction
          - action: switch.turn_off
            metadata: {}
            data: {}
            target:
              entity_id: switch.schedule_temperature_calendar
      - conditions:
          - condition: trigger
            id:
              - demarrage thermostat
        sequence:
          - action: automation.turn_on
            data: {}
            target:
              entity_id: automation.update_ohmigo_resistance
          - action: automation.trigger
            target:
              entity_id: automation.update_ohmigo_resistance
            data:
              skip_condition: true
mode: single
```

## Update Temperature correction (ETC)

Here we adjust the temperature correction each time the real temperature target (RTT) is set on the thermostat **in heating mode**

when the thermostat is in "OFF" Mode, RTT/ETC update is bypassed to avoid unwanted ignitions


```
alias: ETC update
description: ""
triggers:
  - trigger: state
    entity_id:
      - climate.stove
    attribute: temperature
conditions: []
actions:
  - action: script.turn_on
    metadata: {}
    data: {}
    target:
      entity_id: script.etc_update
  - if:
      - condition: state
        entity_id: climate.stove
        state: heat
    then:
      - action: script.turn_on
        metadata: {}
        data: {}
        target:
          entity_id:
            - script.update_resistance_ohmigo
mode: single

```
## Automatic WTS fallback

This automation is not necessary but given as example.

The dry contact switch entity is : `input_boolean.etat_ohmigo`

```
alias: fallback sonde poele
description: ""
triggers:
  - trigger: mqtt
    topic: aha/18fe34ed492b/avty_t
    payload: offline
    alias: déconnexion ohmigo
    id: déconnexion ohmigo
  - alias: reconnexion ohmigo
    trigger: mqtt
    topic: aha/18fe34ed492b/avty_t
    payload: online
    id: reconnexion ohmigo
conditions: []
actions:
  - choose:
      - conditions:
          - condition: trigger
            id:
              - déconnexion ohmigo
        sequence:
          - action: input_boolean.turn_off
            metadata: {}
            data: {}
            target:
              entity_id: input_boolean.etat_ohmigo
        alias: ohmigo déconnecté
      - conditions:
          - condition: trigger
            id:
              - reconnexion ohmigo
        sequence:
          - if:
              - condition: state
                entity_id: input_boolean.etat_ohmigo
                state: "off"
            then:
              - action: input_boolean.turn_on
                metadata: {}
                data: {}
                target:
                  entity_id: input_boolean.etat_ohmigo
        alias: ohmigo reconnecté
mode: single

```

## Lovelace Card

I use [Simple Thermostat](https://github.com/nervetattoo/simple-thermostat)

![image](https://github.com/user-attachments/assets/ac55c503-f239-46b1-a161-76e624816e5c)

```
type: vertical-stack
cards:
  - type: custom:simple-thermostat
    entity: climate.stove
    layout:
      mode:
        headings: false
        icons: true
    header:
      icon: mdi:fireplace
      toggle:
        entity: switch.schedule_temperature_calendar
        name: Mode AUTO
    hide:
      state: true
    control:
      hvac:
        name: true
        icon: true
      preset:
        icon: true
        name: true
```

and couple it with a scheduler : 

![image](https://github.com/user-attachments/assets/a37a32af-e2fa-41aa-98d6-a4d8c4f80261)


# Conclusion 

with only the ohmigo and some lightweight automations we got simple but full control over the stove.

- Putting the thermostat to OFF will extinct the stove
- Putting the thermostat so "HEAT" will send corrected temperatures through the ohmigo to delude the stove adjusting its ingition / extinction to RTT instead of STT
- Stove will ignite using RTT and its own hysteresis

# Whats Next ?

-  Add fallback to the external temperature Sensor (ETS) using combined multiple sensors
-  include generic climate thermostat inside a PID-style thermostat like [versatile Thermostat](https://github.com/jmcollin78/versatile_thermostat) for better regulation
-  make notificatons with ULANZI TC001 to have external output of the thermostat settings

