# Voice Panel

Voice Panel is an Android Voice Assistant for Home Assistant powered by the [Snips Voice Playtform](https://snips.ai/), a private, powerful and customizable voice assistant technology. Snips processes all language input on the device and nothing is ever sent to the cloud.

Voice Panel uses Snips to acta as a voice interface for your Home Assistant components. At this time, you can control your alarm system, lights, windows, blinds, switches and retrieve the weather.  You wake the application by using the wake-word "Hey, Snips" or by using face detection, which acts at the wake-word.  Then simply ask the to "turn off the living room lights" for example.

Currently the application has several limitations.  The Snips Android SDK does not work as a satelite, all messages must be forworded to Home Assistant using MQTT and speach processed on the device using TTS.  The Android SDK also does not support custom wake-words at this time.  You must always use "Hey, Snips" with Voice Panel unless you want to try the face detection wake-word feature which starts listening when it sees a face in the camera. 


## Support

For issues, feature requests, comments or questions, use the [Github issues tracker](https://github.com/thanksmister/vioce-panel-android/issues).  

## Features
- Face activated wake-word (no need to say "Hey, Snips".
- Basic control over Home Assistant components
- Stream video, detect motion, detect faces, and read QR Codes.
- Capture and emailing images when the alarm is disabled.
- MQTT commands to remotely control the application (speak text, play audio, display notifications, alerts, etc.).
- Device sensor data reporting over MQTT (temperature, light, pressure, battery, etc.).
- MQTT Day/Night mode based on the sun from Home Assistant
- MQTT weather data to display weather from Home Assistant

## Screen Shots:

![ui](https://user-images.githubusercontent.com/142340/47121221-42808f80-d248-11e8-813a-72b3be85eae3.png)
![landscape](https://user-images.githubusercontent.com/142340/47121143-e3227f80-d247-11e8-979a-3bdf19ceb998.png)

## Hardware & Software 

- Android Device running Android OS 5.0.1 (SDK 21) or greater. Though performance on older devices may not be ideal for use. 

## Installation

You can download and install the latest release from the [release section](https://github.com/thanksmister/voice-panel-android/releases). 

## Home Assistant Setup

For the Voice Assistant to work with Home Assistant, you neeed to install the Snips component or the Snips add-on. 

-[Snips Component ](https://www.home-assistant.io/components/snips/)
-[Snips Add-On](https://www.home-assistant.io/addons/snips/)

Additionally, you need to create a file to store your [intent scripts](https://www.home-assistant.io/components/intent_script/) and customize the assistants behavior.  Create an "intents.yaml" file in your confi directory, then link this from the configuration.yaml file by adding this line at the bottom: 

```intent_script: !include intents.yaml```

You will place intents scripts within the new "intents.yaml" file to work with the various components currently supported by application.  Home Assistant already has a [bundled scripts](https://github.com/tschmidty69/hass-snips-bundle-intents) included when you add the Snips platform to Home Assistant.  Voice Panel also has custom scripts to extend upon the basic functionality.  The additional scripts can be added to your intents yaml file to use these features. 

Here is an example of scripts to report the weather (using Darksky) and to use the MQTt Manual Alarm Componet:

```
searchWeatherForecast:
  speech:
    type: plain
    text: >
      The weather is currently 
      {{states('sensor.dark_sky_temperature') | round(0)}} 
      degrees outside and 
      {{states('sensor.dark_sky_daily_summary')}} 
      The high today will be 
      {{ states('sensor.dark_sky_daytime_high_temperature') | round(0)}}
      and 
      {{ states('sensor.dark_sky_hourly_summary')}}
HaAlarmStatus:
  speech:
    type: plain
    text: >
      {% if is_state("alarm_control_panel.ha_alarm", "disarmed") -%}
        The alarm is not active.
      {%- endif %}
      {% if is_state("alarm_control_panel.ha_alarm", "armed_home") -%}
        The alarm is in home mode.
      {%- endif %}
      {% if is_state("alarm_control_panel.ha_alarm", "armed_away") -%}
        The alarm is in away mode.
      {%- endif %}
HaAlarmHome:
  speech:
    type: plain
    text: > 
      OK, alarm set to home.
  action:
    - service: alarm_control_panel.alarm_arm_home
      data:
        entity_id: alarm_control_panel.ha_alarm
HaAlarmAway:
  speech:
    type: plain
    text: >
      OK, the alarm set to away, you have 60 seconds to leave.
  action:
    - service: alarm_control_panel.alarm_arm_away
      data:
        entity_id: alarm_control_panel.ha_alarm
HaAlarmDisarm:
  speech:
    type: plain
    text: >
      The alarm has been deactivated.
  action:
    - service: alarm_control_panel.alarm_disarm
      data:
        entity_id: alarm_control_panel.ha_alarm
```


## MQTT Alarm 

To use the Alarm feature of Home Assistant with Voice Panel, you need to install the MQTT Manual Alarm Component in Home Assistant.

-[Manual Alarm Control Panel with MQTT Support](https://www.home-assistant.io/components/alarm_control_panel.manual_mqtt/)

Then under settings (gear icon) enter the MQTT information that you configured in Home Assistant for your MQTT service.  Then under the Alarm setting, enable the MQTT Alarm control.  

You will see a lock icon at the bottom of the screen which will display the current alarm mode and provide a manual means for arming and disarming the alarm in addition to the voice controls.


### Supported Command and Publish States

- Command topic:  home/alarm/set
- Command payloads: ARM_HOME, ARM_AWAY, DISARM
- Publish topic: home/alarm
- Publish payloads: disarmed, armed_away, armed_home, pending, triggered (armed_night not currently supported).

### Example Home Assistant Setup

```
alarm_control_panel:
  - platform: manual_mqtt
    state_topic: home/alarm
    command_topic: home/alarm/set
    pending_time: 60
    trigger_time: 1800
    disarm_after_trigger: false
    delay_time: 30
    armed_home:
      pending_time: 0
      delay_time: 0
    armed_away:
      pending_time: 60
      delay_time: 30
```

-- If I set the the alarm mode home, the alarm will immediately be on without any pending time.  If the alarm is triggered,      there will be no pending time before the siren sounds.   If the alarm mode is away, I have 60 seconds to leave before the      alarm is active and 30 seconds to disarm the alarm when entering.   

-- Notice that my trigger_time is 1800 and disarm_after_trigger is false, this means the alarm runs for 1800 seconds until it    stops and it doesn't reset after its triggerd. 

## Weather Updates (Darksky)

If you would like to get weather updates, you will need to add thje Darksky component to Home Assistant and enter your 
your [Dark Sky API](https://darksky.net/dev/) key. You would then setup an automation to send the updated weather data to Voice Panel using MQTT. 

If you want to have Voice Panel report the weather as well, you can set up a new intent that will speak the current weather conditions in your "intents.yaml" file.  Here is an example intent used to get today's weather: 

```
searchWeatherForecast:
  speech:
    type: plain
    text: >
      The weather is currently 
      {{states('sensor.dark_sky_temperature') | round(0)}} 
      degrees outside and 
      {{states('sensor.dark_sky_daily_summary')}} 
      The high today will be 
      {{ states('sensor.dark_sky_daytime_high_temperature') | round(0)}}
      and 
      {{ states('sensor.dark_sky_hourly_summary')}}
```

## MQTT Sensor and State Data
If MQTT is properly configured, the application can publish data and states for various device sensors, camera detections, and application states. Each device required a unique base topic which you set in the MQTT settings, the default is "voicepanel".  This distinguishes your device if you are running multiple devices.  

### Device Sensors
The application will post device sensors data per the API description and Sensor Reading Frequency. Currently device sensors for Pressure, Temperature, Light, and Battery Level are published. 

#### Sensor Data
Sensor | Keys | Example | Notes
-|-|-|-
battery | unit, value, charging, acPlugged, usbPlugged | ```{"unit":"%", "value":"39", "acPlugged":false, "usbPlugged":true, "charging":true}``` |
light | unit, value | ```{"unit":"lx", "value":"920"}``` |
magneticField | unit, value | ```{"unit":"uT", "value":"-1780.699951171875"}``` |
pressure | unit, value | ```{"unit":"hPa", "value":"1011.584716796875"}``` |
temperature | unit, value | ```{"unit":"°C", "value":"24"}``` |

*NOTE:* Sensor values are device specific. Not all devices will publish all sensor values.

* Sensor values are constructued as JSON per the above table
* For MQTT
  * WallPanel publishes all sensors to MQTT under ```[voicepanel]/sensor```
  * Each sensor publishes to a subtopic based on the type of sensor
    * Example: ```voicepanel/sensor/battery```
    
#### Home Assistant Examples
```YAML
sensor:
  - platform: mqtt
    state_topic: "voicepanel/sensor/battery"
    name: "Alarm Panel Battery Level"
    unit_of_measurement: "%"
    value_template: '{{ value_json.value }}'
    
 - platform: mqtt
    state_topic: "voicepanel/sensor/temperature"
    name: "WallPanel Temperature"
    unit_of_measurement: "°C"
    value_template: '{{ value_json.value }}'

  - platform: mqtt
    state_topic: "voicepanel/sensor/light"
    name: "Alarm Panel Light Level"
    unit_of_measurement: "lx"
    value_template: '{{ value_json.value }}'
    
  - platform: mqtt
    state_topic: "voicepanel/sensor/magneticField"
    name: "Alarm Panel Magnetic Field"
    unit_of_measurement: "uT"
    value_template: '{{ value_json.value }}'

  - platform: mqtt
    state_topic: "voicepanel/sensor/pressure"
    name: "Alarm Panel Pressure"
    unit_of_measurement: "hPa"
    value_template: '{{ value_json.value }}'
```

### Camera Motion, Face, and QR Codes Detections
In additional to device sensor data publishing. The application can also publish states for Motion detection and Face detection, as well as the data from QR Codes derived from the device camera.  

Detection | Keys | Example | Notes
-|-|-|-
motion | value | ```{"value": false}``` | Published immediately when motion detected
face | value | ```{"value": false}``` | Published immediately when face detected
qrcode | value | ```{"value": data}``` | Published immediately when QR Code scanned

* MQTT
  * WallPanel publishes all sensors to MQTT under ```[voicepanel]/sensor```
  * Each sensor publishes to a subtopic based on the type of sensor
    * Example: ```voicepanel/sensor/motion```

#### Home Assistant Examples

```YAML
binary_sensor:
  - platform: mqtt
    state_topic: "voicepanel/sensor/motion"
    name: "Motion"
    payload_on: '{"value":true}'
    payload_off: '{"value":false}'
    device_class: motion 
    
binary_sensor:
  - platform: mqtt
    state_topic: "voicepanel/sensor/face"
    name: "Face Detected"
    payload_on: '{"value":true}'
    payload_off: '{"value":false}'
    device_class: motion 
  
sensor:
  - platform: mqtt
    state_topic: "voicepanel/sensor/qrcode"
    name: "QR Code"
    value_template: '{{ value_json.value }}'
    
```

### Application State Data
The application canl also publish state data about the application such as the current dashboard url loaded or the screen state.

Key | Value | Example | Description
-|-|-|-
currentUrl | URL String | ```{"currentUrl":"http://hasbian:8123/states"}``` | Current URL the Dashboard is displaying
screenOn | true/false | ```{"screenOn":true}``` | If the screen is currently on

* State values are presented together as a JSON block
  * eg, ```{"currentUrl":"http://hasbian:8123/states","screenOn":true}```
* MQTT
  * WallPanel publishes state to topic ```[voicepanel]/state```
    * Default Topic: ```voicepanel/state```

## MJPEG Video Streaming

Use the device camera as a live MJPEG stream. Just connect to the stream using the device IP address and end point. Be sure to turn on the camera streaming options in the settings and set the number of allowed streams and HTTP port number. Note that performance depends upon your device (older devices will be slow).

#### Browser Example:

```http://192.168.1.1:2971/camera/stream```

#### Home Assistant Example:

```YAML
camera:
  - platform: mjpeg
    mjpeg_url: http://192.168.1.1:2971/camera/stream
    name: Alarm Panel Camera
```
## MQTT Commands
Interact and control the application and device remotely using either MQTT commands, including using your device as an announcer with Google Text-To-Speach. Each device required a unique base topic which you set in the MQTT settings, the default is "voicepanel".  This distinguishes your device if you are running multiple devices.  

### Commands
Key | Value | Example Payload | Description
-|-|-|-
audio | URL | ```{"audio": "http://<url>"}``` | Play the audio specified by the URL immediately
wake | true | ```{"wake": true}``` | Wakes the screen if it is asleep
speak | data | ```{"speak": "Hello!"}``` | Uses the devices TTS to speak the message
alert | data | ```{"alert": "Hello!"}``` | Displays an alert dialog within the application
notification | data | ```{"notification": "Hello!"}``` | Displays a system notification on the devie

* The base topic value (default is "voicepanel") should be unique to each device running the application unless you want all devices to receive the same command. The base topic and can be changed in the application settingssettings.
* Commands are constructed via valid JSON. It is possible to string multiple commands together:
  * eg, ```{"clearCache":true, "relaunch":true}```
* MQTT
  * WallPanel subscribes to topic ```[voicepanel]/command```
    * Default Topic: ```voicepanel/command```
  * Publish a JSON payload to this topic (be mindfula of quotes in JSON should be single quotes not double)

### Google Text-To-Speach Command
You can send a command using either HTTP or MQTT to have the device speak a message using Google's Text-To-Speach. Note that the device must be running Android Lollipop or above. 

Example format for the message topic and payload: 

```{"topic":"voicepanel/command", "payload":"{'speak':'Hello!'}"}```

## Notes

- To use TTS and the Camera you will need Android Lollipop SDK or greater as well as camera permissions. Older versions of Android are currently not supported.  The application is locked into the landscape mode for usability.  It is meant to run on dedicated tablets or large screen devices that will be used mainly for an alarm control panel. 

## Acknowledgements

Special thanks to Colin O'Dell who's work on the Home Assistant Manual Alarm Control Panel component and his [MQTT Alarm Panel](https://github.com/colinodell/mqtt-control-panel) helped make this project possible.  Thanks to [Juan Manuel Vioque](https://github.com/tremebundo) for Spanish translations and [Gerben Bol](https://gerbenbol.com/) for Dutch translations and [Jorge Assunção](https://github.com/jorgeassuncao) for Portuguese. 

## Contributors
[Sergio Viudes](https://github.com/sjvc) for Fingerprint unlock support and his [Home-Assistant-WebView](https://github.com/sjvc/Home-Assistant-WebView) component.

