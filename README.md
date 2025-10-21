# ESP32 DS18B20 Temperature Sensors with Home Assistant Integration

Monitor multiple DS18B20 temperature sensors using an ESP32 and automatically integrate them with Home Assistant via MQTT.

Follow this awesome Tutorial for details: 

https://randomnerdtutorials.com/esp32-multiple-ds18b20-temperature-sensors/

## Features

- 🌡️ Support for multiple DS18B20 temperature sensors
- 📡 WiFi connectivity
- 🔄 Automatic MQTT discovery in Home Assistant
- 📊 Real-time temperature reporting (configurable interval)
- 🔌 Simple wiring - all sensors on one data pin
- 💾 Retained MQTT messages for persistence

## Hardware Requirements

- ESP32 development board
- DS18B20 temperature sensor(s)
- 4.7kΩ resistor (pull-up resistor)
- Jumper wires
- Breadboard (optional)

## Wiring Diagram

```
DS18B20 Sensor → ESP32
─────────────────────────
VCC (Red)      → 3.3V
GND (Black)    → GND
DATA (Yellow)  → GPIO 4

Pull-up resistor (4.7kΩ):
Connect between DATA and VCC
```

**Note:** Multiple DS18B20 sensors can be connected in parallel to the same data pin.

## Software Requirements

### Arduino IDE Libraries

Install these libraries via Arduino IDE Library Manager:

1. **OneWire** by Paul Stoffregen
2. **DallasTemperature** by Miles Burton
3. **PubSubClient** by Nick O'Leary

### Home Assistant Requirements

- Home Assistant with MQTT broker configured (Mosquitto recommended)
- MQTT Discovery enabled (enabled by default)

## Installation

### 1. Find Your Sensor Addresses

Before using this code, you need to find the unique addresses of your DS18B20 sensors. Upload this sketch first:

```cpp
#include <OneWire.h>
#include <DallasTemperature.h>

#define ONE_WIRE_BUS 4
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

void setup() {
  Serial.begin(115200);
  sensors.begin();
  
  Serial.println("Scanning for DS18B20 sensors...");
  Serial.print("Found ");
  Serial.print(sensors.getDeviceCount());
  Serial.println(" devices.");
  
  DeviceAddress tempAddress;
  for(int i = 0; i < sensors.getDeviceCount(); i++) {
    if(sensors.getAddress(tempAddress, i)) {
      Serial.print("Sensor ");
      Serial.print(i);
      Serial.print(": { ");
      for(uint8_t j = 0; j < 8; j++) {
        Serial.print("0x");
        if(tempAddress[j] < 16) Serial.print("0");
        Serial.print(tempAddress[j], HEX);
        if(j < 7) Serial.print(", ");
      }
      Serial.println(" }");
    }
  }
}

void loop() {}
```

Copy the addresses from the Serial Monitor output.

### 2. Configure the Main Code

1. Clone this repository
2. Open `esp32_temp_mqtt.ino` in Arduino IDE
3. Update the configuration section:

```cpp
// WiFi credentials
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// MQTT Broker settings
const char* mqtt_server = "YOUR_HA_IP_ADDRESS";  // e.g., "192.168.1.100"
const char* mqtt_port = 1883;
const char* mqtt_user = "YOUR_MQTT_USERNAME";
const char* mqtt_password = "YOUR_MQTT_PASSWORD";

// Your sensor addresses (from step 1)
DeviceAddress sensor1 = { 0x28, 0xFF, 0x64, 0x1F, 0x49, 0x95, 0xF2, 0xAF };
DeviceAddress sensor2 = { 0x28, 0xFF, 0x64, 0x1F, 0x49, 0x97, 0x60, 0x93 };
```

4. Select your ESP32 board and port
5. Upload the code

### 3. Verify in Home Assistant

1. Go to **Settings → Devices & Services → MQTT**
2. You should see a new device: "ESP32 Temperature Sensors"
3. The device will have 2 entities (or more if you added additional sensors)

## Configuration Options

### Update Interval

Change how often temperatures are reported (default: 60 seconds):

```cpp
const long interval = 60000;  // milliseconds
```

### Add More Sensors

To add additional sensors, simply:

1. Define the new sensor address:
```cpp
DeviceAddress sensor3 = { 0x28, 0xFF, ... };
```

2. Add a topic:
```cpp
const char* temp3_topic = "esp32temp/sensor3";
```

3. Add discovery configuration in `publishDiscovery()`:
```cpp
String discovery3 = "{\"name\":\"Temperature Sensor 3\","
                    "\"state_topic\":\"esp32temp/sensor3\","
                    "\"unique_id\":\"esp32_temp_sensor_3\","
                    "\"unit_of_measurement\":\"°C\","
                    "\"device_class\":\"temperature\","
                    "\"state_class\":\"measurement\","
                    "\"device\":{\"identifiers\":[\"esp32temp\"],\"name\":\"ESP32 Temperature Sensors\",\"manufacturer\":\"ESP32\",\"model\":\"DS18B20\"}}";
client.publish("homeassistant/sensor/esp32temp3/config", discovery3.c_str(), true);
```

4. Publish temperature in `loop()`:
```cpp
float temp3C = sensors.getTempC(sensor3);
dtostrf(temp3C, 6, 2, temp3_str);
client.publish(temp3_topic, temp3_str, true);
```

## MQTT Topics

The code uses the following MQTT topic structure:

- **Discovery:** `homeassistant/sensor/esp32temp[1-2]/config`
- **State:** `esp32temp/sensor[1-2]`

## Troubleshooting

### Sensors not appearing in Home Assistant

1. Check MQTT broker is running: `Settings → Add-ons → Mosquitto broker`
2. Verify MQTT discovery is enabled: `Settings → Devices & Services → MQTT → Configure`
3. Check MQTT logs: Subscribe to `homeassistant/#` in Developer Tools → MQTT
4. Restart Home Assistant: `Settings → System → Restart`

### ESP32 keeps disconnecting

- Check that `client.loop()` is being called regularly
- Verify WiFi signal strength
- Check MQTT broker logs for connection issues

### Temperature reads -127°C

- Sensor not connected properly
- Wrong sensor address
- Bad sensor (replace)
- Insufficient power supply

### Discovery messages fail to publish

- Increase MQTT buffer size (already set to 512 bytes)
- Check Serial Monitor for "FAILED" messages
- Verify MQTT credentials are correct

## Home Assistant Dashboard Example

Add this to your Lovelace dashboard:

```yaml
type: entities
entities:
  - entity: sensor.temperature_sensor_1
    name: Living Room
  - entity: sensor.temperature_sensor_2
    name: Bedroom
title: Temperature Monitoring
```

Or use a gauge card:

```yaml
type: gauge
entity: sensor.temperature_sensor_1
min: 0
max: 40
name: Living Room Temperature
unit: °C
severity:
  green: 18
  yellow: 25
  red: 30
```

## Example Automations

### Temperature Alert

```yaml
alias: High Temperature Alert
trigger:
  - platform: numeric_state
    entity_id: sensor.temperature_sensor_1
    above: 30
action:
  - service: notify.mobile_app
    data:
      message: "Temperature is too high: {{ states('sensor.temperature_sensor_1') }}°C"
```

### Temperature-Based Fan Control

```yaml
alias: Auto Fan Control
trigger:
  - platform: numeric_state
    entity_id: sensor.temperature_sensor_1
    above: 25
action:
  - service: switch.turn_on
    target:
      entity_id: switch.fan
```

## License

MIT License - feel free to use and modify for your projects.

## Credits

Based on the original DS18B20 tutorial by Rui Santos at [Random Nerd Tutorials](https://randomnerdtutorials.com)

## Contributing

Pull requests are welcome! For major changes, please open an issue first to discuss what you would like to change.

## Support

If you encounter issues:
1. Check the Troubleshooting section above
2. Review the Serial Monitor output for error messages
3. Open an issue on GitHub with your configuration and error logs
