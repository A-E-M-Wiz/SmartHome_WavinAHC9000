# Wavin-AHC-9000-mqtt
This is a MQTT interface for Wavin AHC-9000 developed for the ESP8266, with the goal of being able to control this heating controller from a home automation system.

## Hardware
The AHC-9000 uses modbus to communicate over a half duplex RS422 connection. It has two RJ45 connectors for this purpose, which can both be used. 
<BR>
The following schematic shows how to connect an ESP8266 to the AHC-9000:
<br>

![Schematic](/electronics/schematic.png)

Components with links to devices on eBay
* ESP8266.
  <br> [NodeMCU Lua Amica V2 ESP8266 ESP12F Module CP2102 WiFi IoT Unsoldered Arduino DIY](https://www.ebay.de/itm/143790404692 )
* 24V to 3v3 switchmode converter
  <br> [DC-DC 3V 5V 12V 24V 36V 48V Buck Adjustable Step Down Voltage Converter Module](https://www.ebay.de/itm/272806096268).
* RS422 to TTL converter
  <br> [MAX3485 TTL to RS485 UART Converter Serial Adapter Module 3.3V-5V for Node](https://www.ebay.de/itm/185483701909)

<br>

## Software

### Setup
Install Visual Studio Code, and install Platform.IO plugin.
<br>
Install [USB driver CP210x USB to UART](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers) from Silicon Labs. Once downloaded, right click on ````silabser.inf```` and select Install. Plug in the ESP8266 using USB, and dobbelcheck in Device Manager that the devices shows up under Ports (COM & LPT) and is listed as ````Silicon Labs CP210x USB to UART Bridge (COMX)````.

<br>

### Bootloader mode
The ESP8266 will enter the serial bootloader when GPIO0 is held low on reset. Otherwise it will run the program in flash. <br>
The Flash BTN on the NodeMCU board pulls GPIO0 low when pressed. However, the automatic bootloader will automatically set the device into Firmware Download Mode when flashing the board through the USB interface. As such, there should be no need to press the Flash BTN. For more info, refer to the [official docs](https://docs.espressif.com/projects/esptool/en/latest/esp8266/advanced-topics/boot-mode-selection.html). <br>
In fact, the Flash BTN can be used as an Input BTN. Set ````pinMode(0, INPUT_PULLUP)```` and you will read LOW if the button is pressed.

<br>

### Configuration
src/PrivateConfig.h contains 5 constants, that should be changed to fit your own setup.

`WIFI_SSID`, `WIFI_PASS`, `MQTT_SERVER`, `MQTT_USER`, and `MQTT_PASS`.

<br>

### Compiling
Using Visual Studio Code, open the directory containing `platformio.ini` from this project, and click build/upload. If you use a different board than nodemcu, remember to change the `board` variable in `platformio.ini`.


### Testing
Assuming you have a working mqtt server setup, you should now be able to control your AHC-9000 using mqtt. If you have the [Mosquitto](https://mosquitto.org/) mqtt tools installed on your mqtt server, you can execude:
```
mosquitto_sub -u username -P password -t heat/# -v
```
to see all live updated parameters from the controller.

To change the target temperature for a output, use:
```
mosquitto_pub -u username -P password -t heat/floorXXXXXXXXXXXX/1/target_set -m 20.5
```
where the number 1 in the above command is the output you want to control and 20.5 is the target temperature in degree celcius. XXXXXXXXXXXX is the MAC address of the Esp8266, so it will be unique for your setup.

### Integration with HomeAssistant
If you have a working mqtt setup in [HomeAssistant](https://home-assistant.io/), all you need to do in order to control your heating from HomeAssistant is to enable auto discovery for mqtt in your `configuration.yaml`.
```
mqtt:
  discovery: true
  discovery_prefix: homeassistant
```
You will then get a climate and a battery sensor device for each configured output on the controller.

If you don't like auto discovery, you can add the entries manually. Create an entry for each output you want to control. Replace the number 0 in the topics with the id of the output and XXXXXXXXXXXX with the MAC of the Esp8266 (can be determined with the mosquitto_sub command shown above)
```
climate wavinAhc9000:
  - platform: mqtt
    name: floor_kitchen
    current_temperature_topic: "heat/floorXXXXXXXXXXXX/0/current"
    temperature_command_topic: "heat/floorXXXXXXXXXXXX/0/target_set"
    temperature_state_topic: "heat/floorXXXXXXXXXXXX/0/target"
    mode_command_topic: "heat/floorXXXXXXXXXXXX/0/mode_set"
    mode_state_topic: "heat/floorXXXXXXXXXXXX/0/mode"
    modes:
      - "heat"
      - "off"
    availability_topic: "heat/floorXXXXXXXXXXXX/online"
    payload_available: "True"
    payload_not_available: "False"
    qos: 0

sensor wavinBattery:
  - platform: mqtt
    state_topic: "heat/floorXXXXXXXXXXXX/0/battery"
    availability_topic: "heat/floorXXXXXXXXXXXX/online"
    payload_available: "True"
    payload_not_available: "False"
    name: floor_kitchen_battery
    unit_of_measurement: "%"
    device_class: battery
    qos: 0
```
