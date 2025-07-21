#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <Adafruit_Sensor.h>
#include <DHT.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

#define OLED_MOSI  D7
#define OLED_CLK   D5
#define OLED_DC    D3
#define OLED_CS    D8
#define OLED_RESET D4

#define DHTPIN     D2
#define DHTTYPE    DHT11

#define RELAY_PUMP D1
#define RELAY_LED  D6

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &SPI, OLED_DC, OLED_RESET, OLED_CS);
DHT dht(DHTPIN, DHTTYPE);
ESP8266WebServer server(80);

const char* ssid = "Patil";  // Change to your WiFi
const char* password = "yourpassword";

bool pumpOn = false;
bool ledOn = false;

void setup() {
  Serial.begin(115200);
  dht.begin();

  pinMode(RELAY_PUMP, OUTPUT);
  pinMode(RELAY_LED, OUTPUT);
  digitalWrite(RELAY_PUMP, LOW);
  digitalWrite(RELAY_LED, LOW);

  if (!display.begin(SSD1306_SWITCHCAPVCC)) {
    Serial.println("OLED Failed");
    while (true);
  }

  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);

  WiFi.begin(ssid, password);
  display.setCursor(0, 0);
  display.print("Connecting WiFi...");
  display.display();

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  display.clearDisplay();
  display.setCursor(0, 0);
  display.print("Connected to:");
  display.setCursor(0, 10);
  display.print(WiFi.localIP());
  display.display();

  server.on("/", handleRoot);
  server.on("/togglePump", togglePump);
  server.on("/toggleLed", toggleLed);
  server.begin();
  Serial.println("Web server started");
}

void loop() {
  server.handleClient();
  updateDisplay();
}

void handleRoot() {
  float t = dht.readTemperature();
  float h = dht.readHumidity();
  int phRaw = analogRead(A0);
  float phValue = (phRaw / 1023.0) * 14.0;

  String html = "<!DOCTYPE html>";
  html += "<html lang='en'>";
  html += "<head>";
  html += "<meta charset='UTF-8'>";  // Ensures proper character encoding
  html += "<meta name='viewport' content='width=device-width, initial-scale=1.0'>";
  html += "<title>🌿 Hydroponic Monitor</title>";
  html += "</head>";
  html += "<body>";
  html += "<h2>🌿 Hydroponic Monitor</h2>";
  html += "🌡️ Temp: " + String(t) + " °C<br>";
  html += "💧 Humidity: " + String(h) + "%<br>";
  html += "🧪 pH: " + String(phValue, 2) + "<br>";
  html += "⚡ Pump: " + String(pumpOn ? "ON" : "OFF") + "<br>";
  html += "<a href='/togglePump'><button>Toggle Pump</button></a><br><br>";
  html += "💡 LED: " + String(ledOn ? "ON" : "OFF") + "<br>";
  html += "<a href='/toggleLed'><button>Toggle LED</button></a>";
  html += "</body>";
  html += "</html>";

  server.send(200, "text/html", html);
}

void togglePump() {
  pumpOn = !pumpOn;
  digitalWrite(RELAY_PUMP, pumpOn ? HIGH : LOW);
  server.sendHeader("Location", "/");
  server.send(303);
}

void toggleLed() {
  ledOn = !ledOn;
  digitalWrite(RELAY_LED, ledOn ? HIGH : LOW);
  server.sendHeader("Location", "/");
  server.send(303);
}

void updateDisplay() {
  float t = dht.readTemperature();
  float h = dht.readHumidity();
  int phRaw = analogRead(A0);
  float phValue = (phRaw / 1023.0) * 14.0;

  display.clearDisplay();
  display.setCursor(0, 0);
  display.print("T: "); display.print(t); display.println(" C");
  display.print("H: "); display.print(h); display.println(" %");
  display.print("pH: "); display.println(phValue, 2);
  display.print("Pump: "); display.println(pumpOn ? "ON" : "OFF");
  display.print("LED: "); display.println(ledOn ? "ON" : "OFF");
  display.display();
}
