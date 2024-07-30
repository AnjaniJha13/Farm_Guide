# Farm_Guide
/* Connections
Relay. D3
Btn.   D7
Soil.  A0
PIR.   D5
Temp.  D4
*/

//Include the library files
#define BLYNK_TEMPLATE_ID "TMPL3F4jbMfuU"
#define BLYNK_TEMPLATE_NAME "FarmGuide"
#define BLYNK_AUTH_TOKEN "ug46nkKnrnnPQGwAn6XJlPz_1TaA4S3l"
#define BLYNK_PRINT Serial
#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>
#include <DHT.h>

char auth[] = "ug46nkKnrnnPQGwAn6XJlPz_1TaA4S3l";  //Enter your Blynk Auth token
char ssid[] = "realme";  //Enter your WIFI SSID
char pass[] = "nfr9393z";  //Enter your WIFI Password

DHT dht(D4, DHT11);  //(DHT sensor pin,sensor type)  D4 DHT11 Temperature Sensor
BlynkTimer timer;

//Define component pins
#define SOIL_PIN A0     // A0 Soil Moisture Sensor
#define PIR_PIN D5      // D5 PIR Motion Sensor
#define RELAY_PIN D3    // D3 Relay
#define BUTTON_PIN D7   // D7 Button
#define VPIN_BUTTON V12 // Virtual pin for Blynk button

int PIR_ToggleValue;
int relayState = LOW;
int buttonState = HIGH;

void setup() {
  Serial.begin(9600);
  pinMode(PIR_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  Blynk.begin(auth, ssid, pass, "blynk.cloud", 80);
  dht.begin();

  timer.setInterval(1000L, soilMoistureSensor);
  timer.setInterval(1000L, DHT11Sensor);
  timer.setInterval(500L, checkPhysicalButton);
}

//Get the DHT11 sensor values
void DHT11Sensor() {
  float h = dht.readHumidity();
  float t = dht.readTemperature();

  if (isnan(h) || isnan(t)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }
  Blynk.virtualWrite(V0, t);
  Blynk.virtualWrite(V1, h);

  Serial.print("Temperature: ");
  Serial.print(t);
  Serial.print(" °C, Humidity: ");
  Serial.print(h);
  Serial.println(" %");
}

//Get the soil moisture values
void soilMoistureSensor() {
  int value = analogRead(SOIL_PIN);
  value = map(value, 0, 1024, 0, 100);
  value = (value - 100) * -1;

  Blynk.virtualWrite(V3, value);
  Serial.print("Soil Moisture: ");
  Serial.print(value);
  Serial.println(" %");
}

//Get the PIR sensor values
void PIRSensor() {
  bool value = digitalRead(PIR_PIN);
  if (value) {
    Blynk.logEvent("motion detector", "WARNING! Motion Detected!"); //Enter your Event Name
    Blynk.virtualWrite(V5, 255); // Turn on virtual LED
  } else {
    Blynk.virtualWrite(V5, 0); // Turn off virtual LED
  }
}

BLYNK_WRITE(V6) {
  PIR_ToggleValue = param.asInt();  
}

BLYNK_CONNECTED() {
  // Request the latest state from the server
  Blynk.syncVirtual(VPIN_BUTTON);
}

BLYNK_WRITE(VPIN_BUTTON) {
  relayState = param.asInt();
  digitalWrite(RELAY_PIN, relayState);
}

void checkPhysicalButton() {
  if (digitalRead(BUTTON_PIN) == LOW) {
    // buttonState is used to avoid sequential toggles
    if (buttonState != LOW) {
      // Toggle Relay state
      relayState = !relayState;
      digitalWrite(RELAY_PIN, relayState);
      // Update Button Widget
      Blynk.virtualWrite(VPIN_BUTTON, relayState);
    }
    buttonState = LOW;
  } else {
    buttonState = HIGH;
  }
}

void loop() {
  if (PIR_ToggleValue == 1) {
    PIRSensor();
  }

  Blynk.run(); // Run the Blynk library
  timer.run(); // Run the Blynk timer
}
