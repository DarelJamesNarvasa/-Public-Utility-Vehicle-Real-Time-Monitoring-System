Public-Utility-Vehicle-Real-Time-Monitoring-System

This guide provides a procedure for the IoT PUV Real-Time Monitoring System project. It is designed to help users understand the hardware and software components used in developing a tracking and monitoring system. The system enables real-time tracking and monitoring of vehicle data, such as location, and passenger count, using IoT technologies. 


In this guide, users will learn how to set up the necessary hardware, configure the website, and integrate these elements to create a fully functional real-time monitoring solution.
![Screenshot_61](https://github.com/user-attachments/assets/1a767af9-51a5-4d13-8d2e-3cedd840f445)
![Screenshot_62](https://github.com/user-attachments/assets/aec43681-4bb9-4ff8-9a3c-9f778b85c983)

## Physical Device (Hardware)
Requirements
- Arduino IDE
- Install ESP32 to Arduino IDE
- Donwload and include library

Circuit Diagram
![Screenshot_63](https://github.com/user-attachments/assets/12042f52-42e2-461e-a2b0-839ed42c20c8)

Arduino Code:

'#include <Wire.h>
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <SoftwareSerial.h>
#include <TinyGPS++.h>
#include <LiquidCrystal_I2C.h>


// GPS
TinyGPSPlus gps;
SoftwareSerial SerialGPS(0, 2); // RX, TX
double Latitude, Longitude;


// Button
int buttonState = 0;
const int buttonPin = 13; // D7


// From Arduino Json Data
SoftwareSerial nodemcu(12, 14); // RX, TX  


// I2C LCD
LiquidCrystal_I2C lcd(0x27, 16, 2);


// WiFi
const char *ssid = "Free Wifi"; // Enter your WiFi name
const char *password = "walaypassword"; // Enter your WiFi password


// MQTT Broker
const char *mqtt_broker = "broker.emqx.io";
const char *topic = "rakalmonitoring/p1";
const char *mqtt_username = "msu_marawi_eee";
const char *mqtt_password = "msu_marawi_eee";
const int mqtt_port = 1883;


WiFiClient espClient;
PubSubClient client(espClient);


// Rakal ID
const int rakalID = 1;


String date = "";
String currentTime = "";


void setup() {
  // Serial config.
  Serial.begin(115200); // Serial Monitor
  nodemcu.begin(4800);
  SerialGPS.begin(9600); // GPS Serial
  while(!Serial) continue;
 
  // button config.
  pinMode(buttonPin, INPUT);


  // LCD config.
  lcd.init();
  lcd.clear();
  lcd.backlight();


  delay(1000); // Delay for stability


  // Connecting to a WiFi network
  WiFi.begin(ssid, password);
  while(WiFi.status() != WL_CONNECTED){
    delay(500);
    Serial.println("Connecting to WiFi...");
    lcd.setCursor(0, 0);
    lcd.print("WiFi --------");
    lcd.setCursor(0, 1);
    lcd.print("MQTT --------");
  }
  Serial.println("Connected to the WiFi network");
  lcd.setCursor(0,0);
  lcd.print("WiFi Connected");


  //Connecting to an MQTT broker
  client.setServer(mqtt_broker, mqtt_port);
  /*
  client.setCallback(callback); // ESP8266 RECIEVER FROM MQTT
  */


  while (!client.connected()) {
        String client_id = "esp8266-client-";
        client_id += String(WiFi.macAddress());
        Serial.printf("The client %s connects to the public MQTT broker\n", client_id.c_str());
        if (client.connect(client_id.c_str(), mqtt_username, mqtt_password)) {
            Serial.println("Public EMQX MQTT broker connected");
            lcd.setCursor(0, 1);
            lcd.print("MQTT Connected");
            delay(2000);
        } else {
            Serial.print("Failed with state ");
            Serial.print(client.state());
            lcd.setCursor(0, 1);
            lcd.print("MQTT Failed   ");
            delay(2000);
        }
    }


    // Publish and subscribe
    client.publish(topic, "hello mqtt, I from nodemcu");
    client.subscribe(topic);


    lcd.clear(); // lcd clear
}


void loop() {
  client.loop();
  delay(100);


  // JSON FORMAT
  StaticJsonDocument<1000> doc;
  StaticJsonDocument<1000> dev;


  buttonState = digitalRead(buttonPin);
  Serial.println(buttonState);


  // Received JsonData From Arduino and Transmitte JsonData to MQTT
  if(buttonState == 0){
    DeserializationError error = deserializeJson(doc, nodemcu);
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Scan your card");
    if(error){
      Serial.println("NO DATA TRANSMITT");
      return;
    }
    else{
      lcd.setCursor(0, 1);
      lcd.print("Card Scanned");
           
      doc["latitude"] = Latitude;
      doc["longitude"] = Longitude;
      String rfid_serial = doc["rfid"];
      doc["date"] = date;
      doc["time"] = currentTime;
      String type = doc["type"];
      doc["rakalID"] = rakalID;
      delay(1500);


      // Lcd Info Display
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("RFID:");;
      lcd.setCursor(5, 0);
      lcd.print(rfid_serial);
      lcd.setCursor(0, 1);
      lcd.print("Type:");;
      lcd.setCursor(5, 0);
      lcd.print(type);
      lcd.setCursor(0, 2);
      lcd.print("Lat:");;
      lcd.setCursor(4, 0);
      lcd.print(Latitude);
      lcd.setCursor(0, 0);
      lcd.print("Long:");;
      lcd.setCursor(5, 0);
      lcd.print(Longitude);
      delay(2000);
      lcd.clear();
    }
    // Convert the doc JSON object to a string
    char jsonBuffer[200];
    serializeJson(doc, jsonBuffer);
    serializeJson(doc, Serial);
    Serial.println("");


    client.publish(topic, jsonBuffer);
    delay(100);
    lcd.clear();
  }
  else{
    lcd.setCursor(0, 0);
    lcd.print("Wait for GPS");


    while (SerialGPS.available() > 0){
      if (gps.encode(SerialGPS.read()))
      {
        if (gps.location.isValid())
        {
          Latitude = gps.location.lat();
          Longitude = gps.location.lng();


          Serial.print("Latitude: ");
          Serial.println(Latitude);
          Serial.print("Longitude: ");
          Serial.println(Longitude);


          date = String(gps.date.year()) + "-" + String(gps.date.month()) + "-" + String(gps.date.day());
          int hourUTC = gps.time.hour();
          int hourLocal = (hourUTC + 8) % 24; // Adjust for UTC+8
          currentTime = String(hourLocal) + ":" + String(gps.time.minute()) + ":" + String(gps.time.second());


          dev["latitude"] = Latitude;
          dev["longitude"] = Longitude;
          dev["rakalID"] = rakalID;


          // Convert the JSON object to a string
          char devJsonBuffer[200];
          serializeJson(dev, devJsonBuffer);
          serializeJson(dev, Serial);
          Serial.println("");


          // Publish the JSON data to the MQTT topic
          client.publish(topic, devJsonBuffer);  
          delay(100);
        }
      }
    }
  }
}
'

Output:

