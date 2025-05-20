# Cabin-of-dream
Operate parameters of cabin
#include <ESP8266WiFi.h>
#include <ESPAsyncWebServer.h>
#include <GyverServo.h>
#include <MicroDS18B20.h>
#include <Adafruit_NeoPixel.h>
#include <EEPROM.h>

// --- НАСТРОЙКИ ---
#define PIN_SERVO 5           // Пин подключения серво (D1 на D1 mini)
#define PIN_LED 6             // Пин ленты (D2 на D1 mini)
#define NUM_PIXELS 30         // Кол-во светодиодов в ленте
#define WIFI_SSID "Cabin-Of-Dream"
#define WIFI_PASSWORD "12345678"

// --- ОБЪЕКТЫ ---
GyverServo servo;
MicroDS18B20<9> tempSensor;  // Датчик DS18B20 на пине D9
Adafruit_NeoPixel strip(NUM_PIXELS, PIN_LED, NEO_GRB + NEO_KHZ800);
AsyncWebServer server(80);

// --- ПЕРЕМЕННЫЕ ---
float currentTemp = 0.0;
float setTemp = 25.0;
int mode = 0;  // 0 - Aqua, 1 - Cyclone, 2 - Temp adaptive
int manualAngle = 90;
bool autoMode = true;

void setup() {
  Serial.begin(115200);

  // Точка доступа Wi-Fi
  WiFi.softAP(WIFI_SSID, WIFI_PASSWORD);
  Serial.println("Точка доступа запущена. IP: " + WiFi.softAPIP().toString());

  // Инициализация датчиков
  tempSensor.setResolution(12);
  strip.begin();
  strip.show();
  servo.attach(PIN_SERVO);
  servo.write(manualAngle);

  // --- Веб интерфейс ---
  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    String html = "<html><head><meta charset='UTF-8'></head><body>";
    html += "<h2>Cabin of Dream - Control Panel</h2>";
    html += "<p>Температура: " + String(currentTemp, 1) + "°C</p>";
    html += "<p>Уставка: " + String(setTemp, 1) + "°C</p>";
    html += "<p>Серво: " + String(manualAngle) + "° (" + (autoMode ? "Авто" : "Ручной") + ")</p>";
    html += "<p><button onclick='fetch(\"/mode?val=0\")'>Aqua</button> ";
    html += "<button onclick='fetch(\"/mode?val=1\")'>Cyclone</button> ";
    html += "<button onclick='fetch(\"/mode?val=2\")'>Temp adaptive</button></p>";
    html += "<p><button onclick='fetch(\"/servo?auto=1\")'>Авто серво</button> ";
    html += "<button onclick='fetch(\"/servo?auto=0\")'>Ручной серво</button></p>";
    html += "<p><button onclick='fetch(\"/angle?val=0\")'>0°</button> ";
    html += "<button onclick='fetch(\"/angle?val=90\")'>90°</button> ";
    html += "<button onclick='fetch(\"/angle?val=180\")'>180°</button></p>";
    html += "</body></html>";
    request->send(200, "text/html", html);
  });

  server.on("/mode", HTTP_GET, [](AsyncWebServerRequest *request){
    if (request->hasParam("val")) {
      mode = request->getParam("val")->value().toInt();
      handleLEDMode();
      request->send(200, "text/plain", "Mode changed");
    }
  });

  server.on("/servo", HTTP_GET, [](AsyncWebServerRequest *request){
    if (request->hasParam("auto")) {
      autoMode = request->getParam("auto")->value().toInt();
      request->send(200, "text/plain", "Servo mode updated");
    }
  });

  server.on("/angle", HTTP_GET, [](AsyncWebServerRequest *request){
    if (request->hasParam("val")) {
      manualAngle = constrain(request->getParam("val")->value().toInt(), 0, 180);
      servo.write(manualAngle);
      request->send(200, "text/plain", "Manual angle set");
    }
  });

  server.begin();
}

void loop() {
  // Считываем температуру
  currentTemp = tempSensor.getTemp();
  tempSensor.requestTemp();

  // Автоматическое управление серво
  if (autoMode) {
    if (currentTemp < setTemp - 1) {
      servo.write(0);
    } else if (currentTemp > setTemp + 1) {
      servo.write(180);
    } else {
      servo.write(90);
    }
  }

  // Обновляем ленту
  handleLEDMode();

  delay(1000);
}

void handleLEDMode() {
  switch (mode) {
    case 0:  // Aqua
      for (int i = 0; i < NUM_PIXELS; i++) {
        strip.setPixelColor(i, strip.Color(0, 255, 255));
      }
      break;

    case 1:  // Cyclone
      for (int i = 0; i < NUM_PIXELS; i++) {
        strip.setPixelColor(i, strip.Color(random(100,255), random(100,255), random(100,255)));
      }
      break;

    case 2:  // Температурный
      if (currentTemp < setTemp) {
        strip.fill(strip.Color(0, 0, 255));  // Синий
      } else {
        strip.fill(strip.Color(255, 0, 0));  // Красный
      }
      break;
  }
  strip.show();
}
