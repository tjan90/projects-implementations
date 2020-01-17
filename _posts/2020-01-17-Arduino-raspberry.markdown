---
layout: post
title:  "Connecting Arduino with Raspberry"
date:   2020-01-17
categories: C Arduino RaspberryPi Mosquitto
---
Application designed to sending data to user via MQTT. Arduino acts as the sensor connected via WiFi and communicate with broker, which is installed in Raspberry. MQTT Client app is required in ur phone to monitor the data you will receive.

###Hardware Requirements:
Arduino Uno WiFi Rev2
Raspberry Pi 3B
LCD Display (Optional)
MQTTClient (Andriod/iOS)


###Setting Up:
####Raspberry Pi
Firs we need to setup the raspberry to work as broker, so we need to install mosquitto.

Here is the link to setup mosquitto on raspberry.

To use the new repository you should first import the repository package signing key follow the below comman, the wget command is used to download single file and stores in a current directory.

```
wget http://repo.mosquitto.org/debian/mosquitto-repo.gpg.key

sudo apt-key add mosquitto-repo.gpg.key
```


Then make the repository available to apt
```
cd /etc/apt/sources.list.d/
```

Enter the following

for wheezy
```
sudo wget http://repo.mosquitto.org/debian/mosquitto-wheezy.list

```
for jessie
```
sudo wget http://repo.mosquitto.org/debian/mosquitto-wheezy.list

```

to install the mqtt mosquitto for the raspberry pi follow the below steps use sudo before the command if not using root
```
sudo -i
```
The above command is not mandatory ,it is if you wish to use root or you will need to prefix each below command with sudo eg sudo apt-get update

The below command is used to update the source list
```
apt-get update
```
After updating type the following commands to install mosquitto broker as shown in the image 1.
```
apt-get install mosquitto
```
the above command is to install mqtt mosquitto broker.


####Arduino
Below is the code to setup wifi connection and credential for connect to raspberry.

Certain things must be configured in order to make this network
- WiFi credentials
WiFi details are found in arduino_secrets.h file that is int he project directory, process is same as we did in the AWS IoT connection but without certificate. Certificate are only required if you are using SSL connection or some sort on encryption library like WolfSSL or SharkSSL.
- MQTT broker details
MQTT broker details are IP address of the broker, Port of broker and topic to which device is subscribed.

```
#include <ArduinoMqttClient.h>
#include <WiFiNINA.h> // for MKR1000 change to: #include <WiFi101.h>
#include <DHT.h>
#include <BH1750.h>
#include <Wire.h>
#include <hd44780.h>
#include <hd44780ioClass/hd44780_I2Cexp.h>
hd44780_I2Cexp lcd;

#include "arduino_secrets.h"
///////please enter your sensitive data in the Secret tab/arduino_secrets.h
char ssid[] = SECRET_SSID;        // your network SSID (name)
char pass[] = SECRET_PASS;    // your network password (use for WPA, or use as key for WEP)
BH1750 lightMeter(0x6B);
float lux;
WiFiClient wifiClient;
MqttClient mqttClient(wifiClient);
const int LCD_COLS = 20;
const int LCD_ROWS = 4;
int sensorpin = A0;
int ledPin = 7;
int val = 0;

const char broker[] = "172.20.10.6";
int        port     = 1883;
const char topic[]  = "arduino/simple";

const long interval = 1000;
unsigned long previousMillis = 0;

int count = 0;

#define DHTPIN 2     // what pin we're connected to
#define DHTTYPE DHT22   // DHT 22  (AM2302)
DHT dht(DHTPIN, DHTTYPE);

unsigned long lastMillis = 0;

int lightVal;
float temperatureVal;
void setup() {
  //Initialize serial and wait for port to open:
  Serial.begin(9600);
  dht.begin();
  Wire.begin();
  lightMeter.begin();
  while (!Serial) {
    ; // wait for serial port to connect. Needed for native USB port only
  }
    pinMode(ledPin, OUTPUT);
  int status;
  status = lcd.begin(LCD_COLS, LCD_ROWS);

  // attempt to connect to Wifi network:
  Serial.print("Attempting to connect to WPA SSID: ");
  Serial.println(ssid);
  while (WiFi.begin(ssid, pass) != WL_CONNECTED) {
    // failed, retry
    Serial.print(".");
    delay(5000);
  }

  Serial.println("You're connected to the network");
  Serial.println(WiFi.localIP());

  // You can provide a unique client ID, if not set the library uses Arduino-millis()
  // Each client must have a unique client ID
  // mqttClient.setId("clientId");

  // You can provide a username and password for authentication
  // mqttClient.setUsernamePassword("username", "password");

  Serial.print("Attempting to connect to the MQTT broker: ");
  Serial.println(broker);

  if (!mqttClient.connect(broker, port)) {
    Serial.print("MQTT connection failed! Error code = ");
    Serial.println(mqttClient.connectError());

    while (1);
  }

  Serial.println("You're connected to the MQTT broker!");
  Serial.println();
}

void loop() {
  lcd.setCursor(0,0);
  lcd.print("Smart Agriculture");
  lcd.setCursor(0,1);

  val = analogRead(sensorpin);      
  Serial.println(val);
  lcd.print("Current Temp: ");
  lcd.print(val-25);
  lcd.setCursor(0,2);
  lcd.print("Current Humid: ");
  lcd.print(val);

   lux = lightMeter.readLightLevel();
    //lightVal = analogRead(A4);
    temperatureVal = dht.readTemperature();
  // call poll() regularly to allow the library to send MQTT keep alives which
  // avoids being disconnected by the broker
  mqttClient.poll();

  // avoid having delays in loop, we'll use the strategy from BlinkWithoutDelay
  // see: File -> Examples -> 02.Digital -> BlinkWithoutDelay for more info
  unsigned long currentMillis = millis();

  if (currentMillis - previousMillis >= interval) {
    // save the last time a message was sent
    previousMillis = currentMillis;

    Serial.println("\nSending message to topic: ");
    Serial.println(topic);
    Serial.print("light:");
    Serial.print(val);
    Serial.print(", Temp: ");
    Serial.print(val-25);


    mqttClient.beginMessage(topic);
    mqttClient.print("light:");
    mqttClient.print(val);
    mqttClient.print(", Temp: ");
    mqttClient.print(val-25);
    mqttClient.endMessage();


  }
  delay(3000);
}
```
