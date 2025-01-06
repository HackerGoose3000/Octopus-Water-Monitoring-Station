# Octopus-Water-Monitoring-Station
Online resources for the setup and maintenance of the Octopus Water Monitoring station.

#include <WiFi.h>
#include <ESPAsyncWebServer.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// WiFi credentials
const char* ssid = "ATTTUtxUNe 2.4";
const char* password = "ygtv#4f2qruv";

// Sensor pins
const int oneWireBus = 33;      // DS18B20 Temperature Sensor
const int soilSensorPin = 35;   // Soil moisture sensor pin
const int orpPin = 39;          // ORP sensor pin
const int waterSensorPin = 32;  // Water level sensor pin
const int turbidityPin = 26;    // Turbidity sensor pin (digital)
const int tdsSensorPin = 36;    // TDS sensor pin
const int flowSensorPin = 27;   // Flow rate sensor pin
const int pHSensorPin = 34;     // pH sensor pin

// Variables
OneWire oneWire(oneWireBus);
DallasTemperature sensors(&oneWire);

volatile byte pulseCount = 0;
float flowRate = 0.0;
float temperatureC = 0.0;
int soilMoisture = 0;
String turbidity;
float tdsValue = 0.0;
float orpValue = 0.0;
int waterLevel = 0;
float pHValue = 0.0;

// LCD setup
LiquidCrystal_I2C lcd(0x3F, 16, 2);  // I2C address 0x3F, 16x2 display
int lcdDelay=2000;
// Web server instance
AsyncWebServer server(80);

// Time tracking for 5-second updates
unsigned long lastUpdateTime = 0;
const unsigned long updateInterval = 5000;  // 5 seconds (5000 milliseconds)

// Function to count pulses for flow sensor
void IRAM_ATTR pulseCounter() {
  pulseCount++;
}

// Function to read all sensors
void readSensors() {
  // Temperature sensor
  sensors.requestTemperatures();
  temperatureC = sensors.getTempCByIndex(0);

  // Soil moisture sensor
  soilMoisture = analogRead(soilSensorPin);

  // Turbidity sensor (digital)
  if (digitalRead(turbidityPin) == HIGH) {
    turbidity = "HIGH";
  } else {
    turbidity = "LOW";
  }

  // TDS sensor
  int tdsRaw = analogRead(tdsSensorPin);
  float voltageTDS = tdsRaw * (3.3 / 4095.0);
  float compensationVoltage = voltageTDS / (1.0 + 0.02 * (temperatureC - 25.0));
  tdsValue = (133.42 * compensationVoltage * compensationVoltage * compensationVoltage -
              255.86 * compensationVoltage * compensationVoltage +
              857.39 * compensationVoltage) * 0.5;

  // Water level sensor
  waterLevel = analogRead(waterSensorPin);

  // ORP sensor
  static int orpArray[40];
  static int orpArrayIndex = 0;
  orpArray[orpArrayIndex++] = analogRead(orpPin);
  if (orpArrayIndex == 40) orpArrayIndex = 0;
  float averageOrp = 0.0;
  for (int i = 0; i < 40; i++) averageOrp += orpArray[i];
  averageOrp /= 40.0;
  orpValue = ((5.0 * averageOrp / 4095.0) * 1000) - 2380; // Adjust offset as needed

  // Flow rate
  flowRate = (pulseCount / 7.5); // Example calculation for flow rate
  pulseCount = 0;

  // pH sensor
  int rawPH = analogRead(pHSensorPin);
  float voltagePH = rawPH * (3.3 / 4095.0);
  pHValue = 3.3 * voltagePH; // Example conversion (adjust based on your calibration data)
}

// Function to calculate percentage for Soil Moisture and Water Level
float soilMoisturePercent() {
  // Reverse the mapping logic: More moisture -> higher percentage
  return map(soilMoisture, 0, 4095, 100, 0);  // Higher moisture -> higher percentage
}

float waterLevelPercent() {
  // Reverse the mapping logic: More water -> higher percentage
  return map(waterLevel, 0, 4095, 0, 100);  // Higher water level -> higher percentage
}

void setup() {
  // Initialize serial monitor
  Serial.begin(115200);

  // WiFi connection
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("Connected to WiFi");
Serial.println(WiFi.localIP());
  
  // Initialize sensors
  sensors.begin();
  pinMode(turbidityPin, INPUT);
  pinMode(flowSensorPin, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(flowSensorPin), pulseCounter, FALLING);

  // Initialize LCD
  Wire.begin();                // Initialize I2C communication
  lcd.begin(16, 2);            // Initialize the 16x2 LCD
  lcd.setBacklight(1);         // Turn on the backlight

server.on("/", HTTP_GET, [](AsyncWebServerRequest *request) {
    request->send_P(200, "text/html; charset=utf-8", R"rawliteral(
      <!DOCTYPE html>
      <html>
      <head>
        <title>ESP32 Sensor Dashboard</title>
        <script>
          function fetchData() {
            fetch('/data').then(response => response.json()).then(data => {
              document.getElementById('temperature').innerText = data.temperatureC + " °C";
              document.getElementById('soilMoisture').innerText = data.soilMoisture + "%";
              document.getElementById('turbidity').innerText = data.turbidity;
              document.getElementById('tds').innerText = data.tds + " ppm";
              document.getElementById('orp').innerText = data.orp + " mV";
              document.getElementById('waterLevel').innerText = data.waterLevel + "%";
              document.getElementById('flowRate').innerText = data.flowRate + " L/min";
              document.getElementById('pH').innerText = data.pHValue.toFixed(2);
            });
          }
          setInterval(fetchData, 1000);
        </script>
      </head>
      <body>
        <h1>ESP32 Sensor Dashboard</h1>
        <p>Temperature: <span id="temperature"></span></p>
        <p>Soil Moisture: <span id="soilMoisture"></span></p>
        <p>Turbidity: <span id="turbidity"></span></p>
        <p>TDS: <span id="tds"></span></p>
        <p>ORP: <span id="orp"></span></p>
        <p>Water Level: <span id="waterLevel"></span></p>
        <p>Flow Rate: <span id="flowRate"></span></p>
        <p>pH: <span id="pH"></span></p>
      </body>
      </html>
    )rawliteral");
});


  server.on("/data", HTTP_GET, [](AsyncWebServerRequest *request) {
    Serial.println("Received request for /data");
    
    // Read sensor data
    readSensors();
    
    // Prepare JSON response
    String json = "{";
    json += "\"temperatureC\":" + String(temperatureC, 2) + ","; te
    json += "\"soilMoisture\":" + String(soilMoisturePercent()) + ","; 
    json += "\"turbidity\":\"" + turbidity + "\","; 
    json += "\"tds\":" + String(tdsValue, 2) + ","; 
    json += "\"orp\":" + String(orpValue, 2) + ","; 
    json += "\"waterLevel\":" + String(waterLevelPercent()) + ","; 
    json += "\"flowRate\":" + String(flowRate, 2) + ","; 
    json += "\"pHValue\":" + String(pHValue, 2);
    json += "}";

    Serial.println("Sending data: " + json);
    request->send(200, "application/json", json);
  });

  // Start server
  server.begin();
  Serial.println("Server started");
}

void loop() {
  unsigned long currentTime = millis();
  if (currentTime - lastUpdateTime >= updateInterval) {
    lastUpdateTime = currentTime;

    // Read sensor data
    readSensors();

    // Display values on LCD
    lcd.clear();  // Clear the display

    lcd.setCursor(0, 0);
    lcd.print("Temp: " + String(temperatureC, 1) + "C");

    lcd.setCursor(0, 1);
    lcd.print("Soil: " + String(soilMoisturePercent()) + "%");

    delay(lcdDelay);  // Delay to keep values for 1 second
    lcd.clear();

    lcd.setCursor(0, 0);
    lcd.print("pH: " + String(pHValue, 1));

    lcd.setCursor(0, 1);
    lcd.print("Flow: " + String(flowRate, 1) + "L/min");

    delay(lcdDelay);  // Delay to keep values for 1 second
    lcd.clear();

    lcd.setCursor(0, 0);
    lcd.print("Turb: " + turbidity);

    lcd.setCursor(0, 1);
    lcd.print("TDS: " + String(tdsValue, 1) + "ppm");

    delay(lcdDelay);  // Delay to keep values for 1 second
    lcd.clear();

    lcd.setCursor(0, 0);
    lcd.print("ORP: " + String(orpValue, 1) + "mV");

    lcd.setCursor(0, 1);
    lcd.print("Water: " + String(waterLevelPercent()) + "%");

    delay(lcdDelay);  // Delay to keep values for 1 second
    lcd.clear();
    
    lcd.setCursor(6, 0);
    lcd.print("Link:");
    lcd.setCursor(1, 1);
    
    lcd.print(String("192.168.1.236"));
    delay(lcdDelay);
    lcd.clear();
  }
}
