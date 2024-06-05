#include <ESP8266WiFi.h>
#include <Adafruit_MQTT.h>
#include <Adafruit_MQTT_Client.h>
#include <WiFiClientSecure.h>
#include <ArduinoJson.h>

// Define pins for the ultrasonic sensor
#define trigPin D4
#define echoPin D5

// Adafruit IO server details
#define AIO_SERVER      "io.adafruit.com"
#define AIO_SERVERPORT  1883  
#define AIO_USERNAME    "your username"
#define AIO_KEY         "Adafruit_Id"
#define MAX_DISTANCE 100

// WiFi credentials
const char* ssid = "wifi_name";
const char* password = "wifi_password";

// Create WiFi client
WiFiClientSecure client;
// Create MQTT client
Adafruit_MQTT_Client mqtt(&client, AIO_SERVER, AIO_SERVERPORT, AIO_USERNAME, AIO_KEY);
// Define the MQTT topic for publishing dustbin level data
Adafruit_MQTT_Publish dustbinlevel = Adafruit_MQTT_Publish(&mqtt, AIO_USERNAME "/feeds/dustbinlevel");

long duration; // Duration of the ultrasonic pulse
int distance;  // Calculated distance from the ultrasonic sensor
float dustbinPercentage; // Calculated percentage of the dustbin fill level

void setup() {
  Serial.begin(9600); // Initialize serial communication
  delay(1000);

  // Connect to WiFi
  WiFi.begin(ssid, password);
  int retries = 0;
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
    retries++;
    if (retries > 10) {
      Serial.println("WiFi connection failed. Restarting...");
      ESP.restart(); // Restart the ESP if WiFi connection fails
    }
  }
  Serial.println();
  Serial.println("WiFi connected");

  // Connect to Adafruit IO
  if (!mqtt.connect()) {
    Serial.println("Connection to Adafruit IO failed");
    return;
  }
  Serial.println("Connected to Adafruit IO");

  // Initialize ultrasonic sensor pins
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
}

void loop() {
  // Trigger the ultrasonic sensor
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  // Measure the duration of the pulse
  duration = pulseIn(echoPin, HIGH, 10000);
  if (duration == 0) {
    Serial.println("Error: pulseIn timeout");
    return;
  }

  // Calculate the distance based on the duration of the pulse
  distance = duration * 0.034 / 2;
  // Calculate the percentage of the dustbin fill level
  dustbinPercentage = ((float)distance / MAX_DISTANCE) * 100;
  dustbinPercentage = constrain(dustbinPercentage, 0, 100);

  // Publish the dustbin level data to Adafruit IO
  if (!publishData()) {
    Serial.println("Error: publishing data to Adafruit IO");
    return;
  }

  // Print the dustbin percentage to the serial monitor
  Serial.print("Dustbin Percentage: ");
  Serial.print(dustbinPercentage);
  Serial.println();

  delay(1000); // Delay for 1 second before the next measurement
}

// Function to publish dustbin level data to Adafruit IO
bool publishData() {
  DynamicJsonDocument jsonDoc(256);
  jsonDoc["dustbin_percentage"] = dustbinPercentage;

  String jsonData;
  serializeJson(jsonDoc, jsonData);
  if (!mqtt.publish(AIO_USERNAME "/Feeds/dustbinlevel", jsonData.c_str())) {
    return false;
  }
  return true;
}
