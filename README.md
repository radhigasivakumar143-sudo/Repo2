#include <DHT.h>
#include <ESP8266WiFi.h>

#define DHTPIN 2
#define DHTTYPE DHT11
#define MQ2 A0
#define FLAME 3
#define BUZZER 4

String apiKey = "YOUR_THINGSPEAK_API_KEY"; 
const char* ssid = "YOUR_WIFI_NAME"; 
const char* password = "YOUR_WIFI_PASSWORD";

const char* server = "api.thingspeak.com";

DHT dht(DHTPIN, DHTTYPE);
WiFiClient client;

void setup() {
  Serial.begin(115200);
  dht.begin();
  
  pinMode(MQ2, INPUT);
  pinMode(FLAME, INPUT);
  pinMode(BUZZER, OUTPUT);

  Serial.println("Connecting to WiFi...");
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  Serial.println("\nWiFi Connected!");
}

void loop() {
  float temp = dht.readTemperature();
  float humidity = dht.readHumidity();
  int smoke = analogRead(MQ2);
  int flame = digitalRead(FLAME);

  Serial.print("Temp: "); Serial.print(temp);
  Serial.print(" Humidity: "); Serial.print(humidity);
  Serial.print(" Smoke: "); Serial.print(smoke);
  Serial.print(" Flame: "); Serial.println(flame);

  // 🔥 Fire Condition
  if (temp > 45 || smoke > 300 || flame == LOW) {
    digitalWrite(BUZZER, HIGH);
    Serial.println("🔥 Fire Detected!");
  } else {
    digitalWrite(BUZZER, LOW);
  }

  // 📡 Send Data to ThingSpeak
  if (client.connect(server, 80)) {
    String postStr = apiKey;
    postStr += "&field1=";
    postStr += String(temp);
    postStr += "&field2=";
    postStr += String(humidity);
    postStr += "&field3=";
    postStr += String(smoke);
    postStr += "&field4=";
    postStr += String(flame);
    postStr += "\r\n\r\n";

    client.print("POST /update HTTP/1.1\n");
    client.print("Host: api.thingspeak.com\n");
    client.print("Connection: close\n");
    client.print("X-THINGSPEAKAPIKEY: " + apiKey + "\n");
    client.print("Content-Type: application/x-www-form-urlencoded\n");
    client.print("Content-Length: ");
    client.print(postStr.length());
    client.print("\n\n");
    client.print(postStr);
  }

  client.stop();
  delay(15000);  // ThingSpeak delay
}