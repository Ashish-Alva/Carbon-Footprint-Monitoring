# Carbon-Footprint-Monitoring
This project aims in developing a real-time energy monitoring system with carbon footprint calculation from electrical appliance and also smart control of electrical appliances.

# Introduction
Effective management and monitoring of energy usage is becoming more and more important as sustainability and energy efficiency gain importance.
In an era where energy efficiency and environmental sustainability are paramount, the Smart Appliance Control and Carbon Footprint Monitoring System represents a forward-thinking approach to managing energy consumption and reducing carbon emissions.

# Problem Statement 
In the modern world, excessive energy consumption is a big problem, and using electrical appliances inefficiently contributes greatly to this problem. 
Effective energy management is hampered by the frequent lack of real-time monitoring and remote control capabilities in current systems. Furthermore, consumers are unable to easily evaluate how their electricity use affects the environment, including the carbon footprint. 
The necessity of this project is driven by the absence of an integrated system that offers remote control, monitoring, and environmental effect assessment.

# Methodology
* A carbon footprint is the total sum of greenhouse gas (GHG) emissions caused by an organization, event, product, or person.
* The increasing concentration of GHGs in the atmosphere can accelerate climate change and global warming.
* It is very necessary to measure these emissions from our day-to-day activities.

![image](https://github.com/user-attachments/assets/5464b96b-7be1-45c8-bce6-1f1d5452864e)

# Calculation Methodology for carbon footprint:
* Electricity : Input value (in KWh/Yr) X 0.85 (Emission Factor) =  Output value in (Kg of CO2)
* Petrol: Input Value(In Litres/Yr) X 2.296(Emission Factor) =  Output value in (Kg of CO2)
* Diesel: Input Value(In Litres/Yr) X 2.653 (Emission Factor) =  Output value in (Kg of CO2)
* LPG: Input Value(In Kg/Yr) X 2.983 (Emission Factor) =  Output value in (Kg of CO2)
Your Carbon Footprint :  Add (1+2+3+4) = Output value in (Kg of CO2)

# Block Diagram
![image](https://github.com/user-attachments/assets/ca4376cb-e364-4c2a-9a62-c624d6b1cc9f)


# Circuit Diagram
![image](https://github.com/user-attachments/assets/458272a6-30a5-44cd-ade8-8d51c1913d07)

# Code
    #define BLYNK_TEMPLATE_ID "*********"  // Define Blynk template ID
    #define BLYNK_TEMPLATE_NAME "*********"  // Define Blynk template name
    #define BLYNK_PRINT Serial  // Enable Serial printing for Blynk debug information
    
    #include "EmonLib.h"  // Include EmonLib for energy monitoring
    #include <EEPROM.h>  // Include EEPROM library for storing data
    #include <WiFi.h>  // Include WiFi library for network connectivity
    #include <BlynkSimpleEsp32.h>  // Include Blynk library for ESP32
    #include <Wire.h>  // Include Wire library for I2C communication
    #include <LiquidCrystal_I2C.h>  // Include LiquidCrystal_I2C library for LCD display
    #include <HTTPClient.h>  // Include HTTPClient library for HTTP requests
    #include <ArduinoJson.h>  // Include ArduinoJson library for JSON handling
    
    LiquidCrystal_I2C lcd(0x27, 16, 2);  // Initialize LCD with I2C address 0x27 and size 16x2
    
    // Define your Telegram bot token and chat ID
    const char* telegramBotToken = "*********";  // Telegram bot token
    const char* telegramChatID = "*********";  // Telegram chat ID
    
    // Constants for calibration
    const float vCalibration = 42.5;  // Voltage calibration factor
    const float currCalibration = 1.80;  // Current calibration factor
    
    // Blynk and WiFi credentials
    const char auth[] = "*********";  // Blynk authentication token
    const char ssid[] = "*********";  // WiFi SSID
    const char pass[] = "*********";  // WiFi password
    
    // EnergyMonitor instance
    EnergyMonitor emon;  // Create an instance of EnergyMonitor
    
    // Timer for regular updates
    BlynkTimer timer;  // Create a Blynk timer instance
    
    // Variables for energy calculation
    float kWh = 0.0;  // Variable to store energy consumed in kWh
    float cost = 0.0;  // Variable to store cost of energy consumed
    const float ratePerkWh = 6.5;  // Cost rate per kWh
    unsigned long lastMillis = millis();  // Variable to store last time in milliseconds
    
    // EEPROM addresses for each variable
    const int addrKWh = 12;  // EEPROM address for kWh
    const int addrCost = 16;  // EEPROM address for cost
    
    // Display page variable
    int displayPage = 0;  // Variable to track current LCD display page
    
    // Reset button pin
    const int resetButtonPin = 4;  // Pin for reset button (change to your button pin)
    
    // Function prototypes
    void sendEnergyDataToBlynk();  // Prototype for sending energy data to Blynk
    void readEnergyDataFromEEPROM();  // Prototype for reading energy data from EEPROM
    void saveEnergyDataToEEPROM();  // Prototype for saving energy data to EEPROM
    void updateLCD();  // Prototype for updating LCD display
    void changeDisplayPage();  // Prototype for changing LCD display page
    void sendBillToTelegram();  // Prototype for sending bill to Telegram
    void resetEEPROM();  // Prototype for resetting EEPROM data
    
    void setup() {
      WiFi.begin(ssid, pass);  // Start WiFi connection
      while (WiFi.status() != WL_CONNECTED) {
        delay(500);  // Wait for WiFi connection
      }
      Blynk.begin(auth, ssid, pass);  // Start Blynk connection
    
      // Initialize the LCD
      lcd.init();  // Initialize the LCD
      lcd.backlight();  // Turn on LCD backlight
    
      // Initialize EEPROM
      EEPROM.begin(32);  // Initialize EEPROM with 32 bytes of storage
    
      // Initialize the reset button pin
      pinMode(resetButtonPin, INPUT_PULLUP);  // Set reset button pin as input with pull-up resistor
    
      // Read stored data from EEPROM
      readEnergyDataFromEEPROM();  // Read energy data from EEPROM
    
      // Setup voltage and current inputs
      emon.voltage(35, vCalibration, 1.7);  // Configure voltage measurement: input pin, calibration, phase shift
      emon.current(34, currCalibration);  // Configure current measurement: input pin, calibration
    
      // Setup timers
      timer.setInterval(2000L, sendEnergyDataToBlynk);  // Set timer to send energy data to Blynk every 2 seconds
      timer.setInterval(2000L, changeDisplayPage);  // Set timer to change display page every 2 seconds
      timer.setInterval(60000L, sendBillToTelegram);  // Set timer to send bill to Telegram every 60 seconds
    }
    
    void loop() {
      Blynk.run();  // Run Blynk
      timer.run();  // Run timers
    
      // Check if the reset button is pressed
      if (digitalRead(resetButtonPin) == LOW) {  // If reset button is pressed (assuming button press connects to ground)
        delay(200);  // Debounce delay
        resetEEPROM();  // Reset EEPROM data
      }
    }
    
    void sendEnergyDataToBlynk() {
      emon.calcVI(20, 2000);  // Calculate voltage and current: number of half wavelengths (crossings), time-out
      float Vrms = emon.Vrms;  // Get root mean square voltage
      float Irms = emon.Irms;  // Get root mean square current
      float apparentPower = emon.apparentPower;  // Get apparent power
    
      // Calculate energy consumed in kWh
      unsigned long currentMillis = millis();  // Get current time in milliseconds
      kWh += apparentPower * (currentMillis - lastMillis) / 3600000000.0;  // Update kWh
      lastMillis = currentMillis;  // Update last time
    
      // Calculate the cost based on the rate per kWh
      cost = kWh * ratePerkWh;  // Calculate cost
    
      // Save the latest values to EEPROM
      saveEnergyDataToEEPROM();  // Save energy data to EEPROM
    
      // Send data to Blynk
      Blynk.virtualWrite(V0, Vrms);  // Send voltage to Blynk virtual pin V0
      Blynk.virtualWrite(V1, Irms);  // Send current to Blynk virtual pin V1
      Blynk.virtualWrite(V2, apparentPower);  // Send apparent power to Blynk virtual pin V2
      Blynk.virtualWrite(V3, kWh);  // Send energy in kWh to Blynk virtual pin V3
      Blynk.virtualWrite(V4, cost);  // Send cost to Blynk virtual pin V4
    
      // Update the LCD with the new values
      updateLCD();  // Update LCD display
    }
    
    void readEnergyDataFromEEPROM() {
      EEPROM.get(addrKWh, kWh);  // Read kWh from EEPROM
      EEPROM.get(addrCost, cost);  // Read cost from EEPROM
    
      // Initialize to zero if values are invalid
      if (isnan(kWh)) {
        kWh = 0.0;  // Set kWh to 0 if invalid
        saveEnergyDataToEEPROM();  // Save to EEPROM
      }
      if (isnan(cost)) {
        cost = 0.0;  // Set cost to 0 if invalid
        saveEnergyDataToEEPROM();  // Save to EEPROM
      }
    }
    
    void saveEnergyDataToEEPROM() {
      EEPROM.put(addrKWh, kWh);  // Save kWh to EEPROM
      EEPROM.put(addrCost, cost);  // Save cost to EEPROM
      EEPROM.commit();  // Commit EEPROM changes
    }
    
    void updateLCD() {
      lcd.clear();  // Clear LCD display
      if (displayPage == 0) {
        lcd.setCursor(0, 0);  // Set cursor to first row
        lcd.printf("V:%.fV I: %.fA", emon.Vrms, emon.Irms);  // Print voltage and current
        lcd.setCursor(0, 1);  // Set cursor to second row
        lcd.printf("P: %.f Watt", emon.apparentPower);  // Print power and energy
      } else if (displayPage == 1) {
        lcd.setCursor(0, 0);  // Set cursor to first row
        lcd.printf("Energy: %.2fkWh", kWh);  // Print energy
        lcd.setCursor(0, 1);  // Set cursor to second row
        lcd.printf("Cost: %.2f", cost);  // Print cost
      }
    }
    
    void changeDisplayPage() {
      displayPage = (displayPage + 1) % 2;  // Change display page
      updateLCD();  // Update LCD display
    }
    
    void sendBillToTelegram() {
      String message = "Total Energy Consumed: " + String(kWh, 2) + " kWh\nTotal Cost: ₹" + String(cost, 2);  // Create message
      HTTPClient http;  // Create HTTP client
      http.begin("https://api.telegram.org/bot" + String(telegramBotToken) + "/sendMessage");  // Begin HTTP request
      http.addHeader("Content-Type", "application/json");  // Add header
    
      DynamicJsonDocument jsonDoc(256);  // Create JSON document
      jsonDoc["chat_id"] = telegramChatID;  // Set chat ID
      jsonDoc["text"] = message;  // Set message text
    
      String jsonString;  // Create JSON string
      serializeJson(jsonDoc, jsonString);  // Serialize JSON document
      int httpCode = http.POST(jsonString);  // Send HTTP POST request
    
      // Optional: Handle HTTP errors here
      http.end();  // End HTTP request
    }
    
    void resetEEPROM() {
      kWh = 0.0;  // Reset kWh to 0
      cost = 0.0;  // Reset cost to 0
      saveEnergyDataToEEPROM();  // Save to EEPROM
    }


# Budget Estimation
![image](https://github.com/user-attachments/assets/df52cf6f-c002-411d-98ed-1285c43a2361)


# Result
Users of carbon footprint monitoring systems report increased awareness of their energy consumption patterns, often resulting in behavioral changes such as turning off appliances when not in use or adjusting usage times to off-peak hours. Many smart home systems facilitate the integration of renewable energy sources like solar panels, reducing reliance on fossil fuels and allowing real-time monitoring of carbon footprints, enabling adjustments based on energy availability.

# Future Scope
The future scope of the carbon footprint monitoring system goes beyond just tracking the carbon footprint from electricity usage. It includes integrating with smart grids and renewable energy sources to optimize energy consumption. 
Carbon footprint is a combined carbon Emission released from Electricity, Transportation(from petrol, diesel), Diet and also from Product manufacturing and transportation of each and every product. 





