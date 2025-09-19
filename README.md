# ESP32 Temperature/Humidity Logger with Node-Red

# About
This is a build log on a modular data collection system that uses ESP32 microntrollers as edge nodes an

# Parts Used

# ESP32 Modbus 
```c++
#include <Arduino.h>
#include <WiFi.h>
#include <ModbusIP_ESP8266.h>
#include "DHT.h"

// WiFi credentials
const char* ssid = "Hot Singles In This Area";
const char* password = "TheBabeCave69420";

#define DHT11_PIN 15
DHT dht11(DHT11_PIN, DHT11);

// Modbus instance
ModbusIP mb;

// Example register address
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
  
  // Add a holding register with default value 0
  mb.addHreg(REG_Humidity, 0);
  mb.addHreg(REG_Temperature_C, 0);
  mb.addHreg(REG_Temperature_F, 0);

  dht11.begin();
}

void loop() {
  // Handle incoming Modbus TCP requests
  mb.task();

  float humidity = dht11.readHumidity();
  float temperature_C = dht11.readTemperature();
  float temperature_F = dht11.readTemperature(true);

  delay(2000);

  Serial.printf("Temp: %.1f C, %.1f F  Humidity: %.1f %%\n", temperature_C, temperature_F, humidity);

  // Update holding register 
  mb.Hreg(REG_Humidity, (int)(humidity * 10));
  mb.Hreg(REG_Temperature_C, (int)(temperature_C * 10));
  mb.Hreg(REG_Temperature_F, (int)(temperature_F * 10));
}
```

# Node-Red Dashboard


<p align="left">
  <img src="https://github.com/user-attachments/assets/38f856f1-0acc-4f9c-884f-ccc60b1b1b46" width="48%" height="48%" alt="Left Image">
  <img src="https://github.com/user-attachments/assets/499861c5-1ba8-40b0-86c7-1095d82e36dc" width="48%" height="48%" alt="Right Image">
</p> 

<p align="left">
  <img src="https://github.com/user-attachments/assets/4485846a-588f-4d40-98dd-fdcd6f0a2adf" width="48%" height="48%" alt="Left Image">
  <img src="https://github.com/user-attachments/assets/21b57cbe-8981-4cd2-ba70-b535a1478d03" width="48%" height="48%" alt="Right Image">
</p>


# Closing
