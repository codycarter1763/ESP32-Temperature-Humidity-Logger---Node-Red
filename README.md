# ESP32 Temperature/Humidity Logger with Node-Red
![IMG_5706](https://github.com/user-attachments/assets/f38f252c-6378-4502-a13e-e8c99bf6e744)
<img width="1897" height="918" alt="Living Room Dash" src="https://github.com/user-attachments/assets/d5e9883d-21e6-4923-bfa5-79d0c9c27ab4" />

# About
This is a build log on a modular data collection system that uses ESP32 microntrollers configured as a Modbus RTU server. This intergrates DHT11 temperature and humidity sensors with ESP32 edge devices for use on Node-RED for real-time data collection and visualization. The ESP32s function as low-cost PLC equivalents, transmitting telemetry data to SCADA systems for comprehensive monitoring, data logging, and automation.

Meant for IoT, automation, and industrial applications, this configuration delivers PLC-like control and data acquisition at an accessible price point where affordable hardware meets enterprise-grade protocols.

# Parts Used
The hardware setup is very simple but allows for nearly unlimited data collection combinations depending on your need. 
- ESP32
- DHT11 Temperature/Humidity Sensor

# ESP32 Modbus 
Below is a custom firmware that sets up the ESP32 as a Modbus RTU server that collects temperature and humidity measurements from DHT11 sensors. Sensor data is stored into the ESP32's registers for use in Node-RED. 

This setup can be done with openPLC, though creating your own Modbus firmware allows for more flexibility and sensor compatibility.

## Required Libraries
When running this firmware, install these libraries:
- Adafruit's DHT Sensor Library
- Adafruit Unified Sensor Library
- Emelianov's modbus-esp8266 Library

```c++
/******************************************
 * DHT11 Modbus Data Collection Arduino Code
 * Author: Cody Carter
 * Date: September 2025
 * Version: 1.0.0
 * 
 * This firmware creates a Modbus server on port 502 and 
 * records temperature and humidity data from a DHT11 sensor.
 * 
 * Tested on Node-Red, but easily expandable to other platforms.
 ******************************************/

#include <Arduino.h>
#include <WiFi.h>
#include <ModbusIP_ESP8266.h>
#include "DHT.h"

// WiFi credentials
const char* ssid = "SSID";
const char* password = "PASSWORD";

#define DHT11_PIN 15

// DHT11 instance
DHT dht11(DHT11_PIN, DHT11);

// Modbus instance
ModbusIP mb;

// Register addresses
const int REG_Humidity = 100;
const int REG_Temperature_C = 101;
const int REG_Temperature_F = 102;

void setup() {
  Serial.begin(115200);

  // Connect to WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(200);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected, IP: " + WiFi.localIP().toString());

  mb.server();  // This starts the server on port 502
  Serial.println("Modbus TCP Server started on port 502");
  
  // Add holding registers with default value 0
  mb.addHreg(REG_Humidity, 0);
  mb.addHreg(REG_Temperature_C, 0);
  mb.addHreg(REG_Temperature_F, 0);

  // Begin DHT11 data collection
  dht11.begin();
}

void loop() {
  // Handle incoming Modbus TCP requests
  mb.task();

  // Float values for temperature and humidity
  float humidity = dht11.readHumidity();
  float temperature_C = dht11.readTemperature();
  float temperature_F = dht11.readTemperature(true);

  delay(2000); // Required poll every 2 seconds for DHT11

  Serial.printf("Temp: %.1f C, %.1f F  Humidity: %.1f %%\n", temperature_C, temperature_F, humidity);

  // Update holding registers 
  mb.Hreg(REG_Humidity, (int)(humidity * 10));
  mb.Hreg(REG_Temperature_C, (int)(temperature_C * 10));
  mb.Hreg(REG_Temperature_F, (int)(temperature_F * 10));
}
```

# Node-RED Setup
For my Node_RED setup, I downloaded and hosted the service on my Linux home server to provide an always-on automation environment. This was also useful since hosting PiVPN with Wireguard aloows me to access my dashboard remotely from anywhere in the world. 

To get my block diagram, download and upload the included .json file in my repo above.

## Block Diagram
These blocks below were what handled the ESP32 data transfer and translation. 

<img width="620" height="718" alt="Screenshot 2025-09-19 111513" src="https://github.com/user-attachments/assets/65ec4f7b-e86d-48cc-a963-68443c10f621" />

- First is a Modbus-Read block that will intercept the ESP32 edge devices via a TCP connection. 
- Second, the Modbus is fed into a function that converts the casted temperature and humidity values into float values.
    - This was done since the Modbus only takes int values, so multiplying the measured values by 10 and diving by 10 later to get the original values.
Below is the javascript code in each Scaled to Real function block.

```javascript
var registers = msg.payload;

var Humid_Raw = registers[0];
var Temp_C_Raw = registers[1];
var Temp_F_Raw = registers[2];

// Divide by 10 to get original measured values before int transform
var humid = Humid_Raw / 10;
var temp_C = Temp_C_Raw / 10;
var temp_F = Temp_F_Raw / 10;

msg.payload = {
    humidity: parseFloat(humid.toFixed(1)),
    temperature_C: parseFloat(temp_C.toFixed(1)),
    temperature_F: parseFloat(temp_F.toFixed(1)),
    timestamp: new Date().toISOString(),
    time: new Date().toLocaleTimeString()
}

msg.humidity = humid;
msg.tempC = temp_C;
msg.tempF = temp_F;

var now = new Date().getTime();  // Milliseconds for chart X-axis

msg.payload.humidity_chart = [[now, humid]];           // [timestamp, 52.3]

msg.payload.combined_chart = [
    { series: ["Temperature (°F)"], data: [[now, temp_F]], labels: [""] },
    { series: ["Temperature (°C)"], data: [[now, temp_C]], labels: [""] }
];

// Message 1: Celsius series
var msg1 = {};
msg1.topic = "Celsius";              // ← Series name (groups this line)
msg1.payload = temp_C;               // ← Y-value
msg1.timestamp = now;                // ← X-value (time)

// Message 2: Fahrenheit series  
var msg2 = {};
msg2.topic = "Fahrenheit";           // ← Different series name
msg2.payload = temp_F;               // ← Y-value
msg2.timestamp = now;                // ← X-value (time)

// Message 3: Humidity series  
var msg3 = {};
msg3.topic = "Humidity";           // ← Different series name
msg3.payload = humid;               // ← Y-value
msg3.timestamp = now;                // ← X-value (time)

node.status({
    fill: "green",
    shape: "dot",
    text: "Connected: " + temp_C.toFixed(1) + "°C, " + temp_F.toFixed(1) + "°F, " + humid.toFixed(1) + "%"
});

return [msg1, msg2, msg3];
```

- Lastly, the various charts and graphs that represent the measured data in real time and over a time period.

## FlowFuse Dashboard
Using Node-RED FlowFuse, real-time evaluation and data collection can be developed in a matter of minutes for faster development and less downtime. This setup is easily expandable, where in my case I used three different ESP32 edge devices as a proof of concept.

<img width="1897" height="918" alt="Living Room Dash" src="https://github.com/user-attachments/assets/5bdef169-e1bf-44dc-8e97-3bd8b7fd2a86" />
<img width="1897" height="913" alt="Outside Dash" src="https://github.com/user-attachments/assets/3b9d1598-dc66-43e2-af77-2abe8e29fad6" />
<img width="1898" height="916" alt="3D Printer Room Dash" src="https://github.com/user-attachments/assets/9b8c7c02-79c1-4711-830c-832e47744a59" />

# Closing
Hope you can use this repository for inspiration on how you can use ESP32's with Modbus and have data visualized very quickly. Feel free to take what I did and add your own adjustments!
