#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <ArduinoJson.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Wire.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <TinyGPS++.h>
#include <time.h>

// ===== OLED =====
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// ===== PINS =====
#define TEMP_PIN 4
#define SOS_BUTTON_PIN 27
#define RXD2 16
#define TXD2 17
#define PULSE_PIN 33
#define BAND_TOUCH_PIN 13  
#define BUZZER_PIN 14   // <--- NEW BUZZER PIN

OneWire oneWire(TEMP_PIN);
DallasTemperature sensors(&oneWire);
TinyGPSPlus gps;
HardwareSerial GPSSerial(1);

// ===== WiFi & Telegram =====
const char* ssid = "STARK";
const char* password = "8951343725";
#define BOT_TOKEN "8292732806:AAEH2LTd76f3llKIf2Ja5oy9_RN61lDKmkY"
#define CHAT_ID   "6227943753"

WiFiClientSecure secured_client;
UniversalTelegramBot bot(BOT_TOKEN, secured_client);

// ===== Time (India) =====
const char* ntpServer = "pool.ntp.org";
const long gmtOffset_sec = 19800;
const int daylightOffset_sec = 0;

// ===== Variables =====
bool sosActive = false;
bool tracking = false;
unsigned long lastSend = 0;
const unsigned long interval = 30000;

// ===== Heartbeat Variables =====
int bpm = 0;
int bpmSmoothed = 0;
int bpmHistory[10] = {0};
int historyIndex = 0;
unsigned long lastBeat = 0;
int threshold = 2000;

// ===== Band Removal =====
bool bandRemoved = false;

// ===== Time structure =====
struct TimeInfo {
  String day;
  String time;
  String date;
};

// ===== Functions =====
void setupTime() {
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
}

TimeInfo getTimeNow() {
  struct tm timeinfo;
  TimeInfo tinfo;
  if (!getLocalTime(&timeinfo)) {
    tinfo.day = "Day Err";
    tinfo.time = "Time Err";
    tinfo.date = "Date Err";
    return tinfo;
  }
  char dayBuffer[10], timeBuffer[10], dateBuffer[15];
  strftime(dayBuffer, 10, "%a", &timeinfo);
  strftime(timeBuffer, 10, "%H:%M:%S", &timeinfo);
  strftime(dateBuffer, 15, "%d-%m-%Y", &timeinfo);

  tinfo.day = String(dayBuffer);
  tinfo.time = String(timeBuffer);
  tinfo.date = String(dateBuffer);
  return tinfo;
}

void sendMessage(String msg) {
  bot.sendMessage(CHAT_ID, msg, "");
  Serial.println("Telegram -> " + msg);
}

void sendLocationTelegram() {
  if (!gps.location.isValid()) return;
  char url[256];
  snprintf(url, sizeof(url), "/bot%s/sendLocation?chat_id=%s&latitude=%.6f&longitude=%.6f",
           BOT_TOKEN, CHAT_ID, gps.location.lat(), gps.location.lng());
  WiFiClientSecure client;
  client.setInsecure();
  if (client.connect("api.telegram.org", 443)) {
    client.print(String("GET ") + url + " HTTP/1.1\r\nHost: api.telegram.org\r\nConnection: close\r\n\r\n");
    while (client.connected() || client.available()) { if (client.available()) client.read(); }
    client.stop();
  }
}

void checkTelegramCommands() {
  int numNew = bot.getUpdates(bot.last_message_received + 1);
  for (int i = 0; i < numNew; i++) {
    String msg = bot.messages[i].text;
    msg.trim();
    Serial.println("Command: " + msg);
    if (msg == "/track") {
      tracking = true;
      sendMessage("Tracking Started");
    } else if (msg == "/stop") {
      tracking = false;
      sendMessage("Tracking Stopped");
    } else if (msg == "/location") {
      sendLocationTelegram();
      sendMessage("Location Sent");
    }
  }
}

// ===== Heartbeat Processing =====
void readHeartbeat() {
  int val = analogRead(PULSE_PIN);
  unsigned long currentTime = millis();

  if (val > threshold) {
    int beatDuration = currentTime - lastBeat;
    if (beatDuration > 300) {
      bpm = 60000 / beatDuration;
      lastBeat = currentTime;
      if (bpm < 60) bpm = 60;
      if (bpm > 140) bpm = 140;

      bpmHistory[historyIndex] = bpm;
      historyIndex = (historyIndex + 1) % 10;
    }
  }

  long sum = 0;
  for (int i = 0; i < 10; i++) sum += bpmHistory[i];
  bpmSmoothed = sum / 10;
  if (bpmSmoothed == 0) bpmSmoothed = 80;
  if (bpmSmoothed < 60) bpmSmoothed = 60;
  if (bpmSmoothed > 140) bpmSmoothed = 140;

  Serial.print("Heartbeat BPM: ");
  Serial.println(bpmSmoothed);
}

void setup() {
  Serial.begin(115200);
  GPSSerial.begin(9600, SERIAL_8N1, RXD2, TXD2);
  sensors.begin();

  pinMode(SOS_BUTTON_PIN, INPUT_PULLUP);
  pinMode(BAND_TOUCH_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);   // <--- BUZZER SETUP

  // OLED
  Wire.begin(21, 22);
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("SSD1306 not found");
    for (;;);
  }
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.display();

  // WiFi
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); }
  Serial.println(" connected!");

  secured_client.setInsecure();
  setupTime();

  Serial.println("Health Monitoring System Ready!");
  sendMessage("Hazard Detection System Online!");
}

void loop() {
  // GPS
  while (GPSSerial.available() > 0) gps.encode(GPSSerial.read());

  // Telegram Commands
  checkTelegramCommands();

  // Temperature
  sensors.requestTemperatures();
  float tempC = sensors.getTempCByIndex(0);

  // ==== HIGH TEMPERATURE BUZZER ====
  if (tempC > 37.5) {
    digitalWrite(BUZZER_PIN, HIGH);  // continuous
    sendMessage("High Temp Alert: " + String(tempC) + "C\n" + getTimeNow().date);
  } 
  else {
    digitalWrite(BUZZER_PIN, LOW);   // stop beep
  }

  // Heartbeat
  readHeartbeat();

  // SOS Button
  if (digitalRead(SOS_BUTTON_PIN) == LOW) {
    unsigned long pressStart = millis();
    while (digitalRead(SOS_BUTTON_PIN) == LOW) {
      if (millis() - pressStart > 2000 && !sosActive) {
        sosActive = true;
        String msg = "⚠️ SOS Triggered!\nTime: " + getTimeNow().time + "\nDate: " + getTimeNow().date;
        if (gps.location.isValid()) msg += "\nLocation: https://maps.google.com/?q=" + String(gps.location.lat(),6) + "," + String(gps.location.lng(),6);
        sendMessage(msg);

        display.clearDisplay();
        display.setCursor(0, 0);
        display.println("SOS Triggered!");
        display.println("Time: " + getTimeNow().time);
        display.println("Date: " + getTimeNow().date);
        display.display();
      }
    }
  } else { sosActive = false; }

  // ==== BAND REMOVED DETECTION + BUZZER 1 BEEP ====
  int touchValue = touchRead(BAND_TOUCH_PIN);
  if (touchValue > 550) {
    if (!bandRemoved) {
      bandRemoved = true;

      // Buzzer single beep
      digitalWrite(BUZZER_PIN, HIGH);
      delay(300);
      digitalWrite(BUZZER_PIN, LOW);

      String msg = "⚠️ Band Removed!\nTime: " + getTimeNow().time + "\nDate: " + getTimeNow().date;
      if (gps.location.isValid()) msg += "\nLocation: https://maps.google.com/?q=" + String(gps.location.lat(),6) + "," + String(gps.location.lng(),6);
      sendMessage(msg);

      display.clearDisplay();
      display.setCursor(0, 0);
      display.println("Band Removed!");
      display.println("Time: " + getTimeNow().time);
      display.println("Date: " + getTimeNow().date);
      display.display();
    }
  } else {
    bandRemoved = false;
  }

  // Tracking interval
  if (tracking && gps.location.isValid() && millis() - lastSend > interval) {
    sendLocationTelegram();
    lastSend = millis();
  }

  // OLED display
  TimeInfo t = getTimeNow();
  display.clearDisplay();
  display.setCursor(0, 0);
  display.println("Hazard Detect Device");
  display.println("Day: " + t.day);
  display.println("Time: " + t.time);
  display.println("Date: " + t.date);
  display.print("Temp: "); display.print(tempC); display.println(" C");
  display.print("BPM: "); display.println(bpmSmoothed);

  if (gps.location.isValid()) {
    display.print("Lat:"); display.print(gps.location.lat(), 2);
    display.print(",Lng:"); display.println(gps.location.lng(), 2);
  } else {
    display.println("No GPS Fix");
  }

  if (sosActive) display.println("Status: SOS Triggered");
  else if (tracking) display.println("Status: Tracking");
  else if (bandRemoved) display.println("Status: Band Removed");
  else display.println("Status: Normal");

  display.display();

  // Serial output
  Serial.println("----- STATUS -----");
  Serial.println("Day: " + t.day);
  Serial.println("Time: " + t.time);
  Serial.println("Date: " + t.date);
  Serial.print("Temp: "); Serial.println(tempC);
  Serial.print("BPM: "); Serial.println(bpmSmoothed);
  if (gps.location.isValid()) {
    Serial.print("Lat: "); Serial.println(gps.location.lat(),2);
    Serial.print("Lng: "); Serial.println(gps.location.lng(),2);
  } else Serial.println("GPS not fixed");

  if (sosActive) Serial.println("Status: SOS Triggered");
  else if (tracking) Serial.println("Status: Tracking");
  else if (bandRemoved) Serial.println("Status: Band Removed");
  else Serial.println("Status: Normal");
  Serial.println("-----------------");

  delay(1000);
}


//updated code//

#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <ArduinoJson.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Wire.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <TinyGPS++.h>
#include <time.h>

// ===== OLED =====
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// ===== PINS =====
#define TEMP_PIN 4
#define SOS_BUTTON_PIN 27
#define RXD2 16
#define TXD2 17
#define PULSE_PIN 33
#define BAND_TOUCH_PIN 13  
#define BUZZER_PIN 14   // <--- NEW BUZZER PIN

OneWire oneWire(TEMP_PIN);
DallasTemperature sensors(&oneWire);
TinyGPSPlus gps;
HardwareSerial GPSSerial(1);

// ===== WiFi & Telegram =====
const char* ssid = "STARK";
const char* password = "8951343725";
#define BOT_TOKEN "8292732806:AAEH2LTd76f3llKIf2Ja5oy9_RN61lDKmkY"
#define CHAT_ID   "6227943753"

WiFiClientSecure secured_client;
UniversalTelegramBot bot(BOT_TOKEN, secured_client);

// ===== Time (India) =====
const char* ntpServer = "pool.ntp.org";
const long gmtOffset_sec = 19800;
const int daylightOffset_sec = 0;

// ===== Variables =====
bool sosActive = false;
bool tracking = false;
unsigned long lastSend = 0;
const unsigned long interval = 30000;

// ===== Heartbeat Variables =====
int bpm = 0;
int bpmSmoothed = 0;
int bpmHistory[10] = {0};
int historyIndex = 0;
unsigned long lastBeat = 0;
int threshold = 2000;

// ===== Band Removal =====
bool bandRemoved = false;

// ===== Time structure =====
struct TimeInfo {
  String day;
  String time;
  String date;
};

// ===== Functions =====
void setupTime() {
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
}

TimeInfo getTimeNow() {
  struct tm timeinfo;
  TimeInfo tinfo;
  if (!getLocalTime(&timeinfo)) {
    tinfo.day = "Day Err";
    tinfo.time = "Time Err";
    tinfo.date = "Date Err";
    return tinfo;
  }
  char dayBuffer[10], timeBuffer[10], dateBuffer[15];
  strftime(dayBuffer, 10, "%a", &timeinfo);
  strftime(timeBuffer, 10, "%H:%M:%S", &timeinfo);
  strftime(dateBuffer, 15, "%d-%m-%Y", &timeinfo);

  tinfo.day = String(dayBuffer);
  tinfo.time = String(timeBuffer);
  tinfo.date = String(dateBuffer);
  return tinfo;
}

void sendMessage(String msg) {
  bot.sendMessage(CHAT_ID, msg, "");
  Serial.println("Telegram -> " + msg);
}

void sendLocationTelegram() {
  if (!gps.location.isValid()) return;
  char url[256];
  snprintf(url, sizeof(url), "/bot%s/sendLocation?chat_id=%s&latitude=%.6f&longitude=%.6f",
           BOT_TOKEN, CHAT_ID, gps.location.lat(), gps.location.lng());
  WiFiClientSecure client;
  client.setInsecure();
  if (client.connect("api.telegram.org", 443)) {
    client.print(String("GET ") + url + " HTTP/1.1\r\nHost: api.telegram.org\r\nConnection: close\r\n\r\n");
    while (client.connected() || client.available()) { if (client.available()) client.read(); }
    client.stop();
  }
}

void checkTelegramCommands() {
  int numNew = bot.getUpdates(bot.last_message_received + 1);
  for (int i = 0; i < numNew; i++) {
    String msg = bot.messages[i].text;
    msg.trim();
    Serial.println("Command: " + msg);
    if (msg == "/track") {
      tracking = true;
      sendMessage("Tracking Started");
    } else if (msg == "/stop") {
      tracking = false;
      sendMessage("Tracking Stopped");
    } else if (msg == "/location") {
      sendLocationTelegram();
      sendMessage("Location Sent");
    }
  }
}

// ===== Heartbeat Processing =====
void readHeartbeat() {
  int val = analogRead(PULSE_PIN);
  unsigned long currentTime = millis();

  if (val > threshold) {
    int beatDuration = currentTime - lastBeat;
    if (beatDuration > 300) {
      bpm = 60000 / beatDuration;
      lastBeat = currentTime;
      if (bpm < 60) bpm = 60;
      if (bpm > 140) bpm = 140;

      bpmHistory[historyIndex] = bpm;
      historyIndex = (historyIndex + 1) % 10;
    }
  }

  long sum = 0;
  for (int i = 0; i < 10; i++) sum += bpmHistory[i];
  bpmSmoothed = sum / 10;
  if (bpmSmoothed == 0) bpmSmoothed = 80;
  if (bpmSmoothed < 60) bpmSmoothed = 60;
  if (bpmSmoothed > 140) bpmSmoothed = 140;

  Serial.print("Heartbeat BPM: ");
  Serial.println(bpmSmoothed);
}

void setup() {
  Serial.begin(115200);
  GPSSerial.begin(9600, SERIAL_8N1, RXD2, TXD2);
  sensors.begin();

  pinMode(SOS_BUTTON_PIN, INPUT_PULLUP);
  pinMode(BAND_TOUCH_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);   // <--- BUZZER SETUP

  // OLED
  Wire.begin(21, 22);
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("SSD1306 not found");
    for (;;);
  }
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.display();

  // WiFi
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); }
  Serial.println(" connected!");

  secured_client.setInsecure();
  setupTime();

  Serial.println("Health Monitoring System Ready!");
  sendMessage("Health Monitoring System Online!");
}

void loop() {
  // GPS
  while (GPSSerial.available() > 0) gps.encode(GPSSerial.read());

  // Telegram Commands
  checkTelegramCommands();

  // Temperature
  sensors.requestTemperatures();
  float tempC = sensors.getTempCByIndex(0);

  // ==== HIGH TEMPERATURE BUZZER ====
  if (tempC > 37.5) {
    digitalWrite(BUZZER_PIN, HIGH);  // continuous
    sendMessage("High Temp Alert: " + String(tempC) + "C\n" + getTimeNow().date);
  } 
  else {
    digitalWrite(BUZZER_PIN, LOW);   // stop beep
  }

  // Heartbeat
  readHeartbeat();

  // SOS Button
  if (digitalRead(SOS_BUTTON_PIN) == LOW) {
    unsigned long pressStart = millis();
    while (digitalRead(SOS_BUTTON_PIN) == LOW) {
      if (millis() - pressStart > 2000 && !sosActive) {
        sosActive = true;
        String msg = "⚠️ SOS Triggered!\nTime: " + getTimeNow().time + "\nDate: " + getTimeNow().date;
        if (gps.location.isValid()) msg += "\nLocation: https://maps.google.com/?q=" + String(gps.location.lat(),6) + "," + String(gps.location.lng(),6);
        sendMessage(msg);

        display.clearDisplay();
        display.setCursor(0, 0);
        display.println("SOS Triggered!");
        display.println("Time: " + getTimeNow().time);
        display.println("Date: " + getTimeNow().date);
        display.display();
      }
    }
  } else { sosActive = false; }

  // ==== BAND REMOVED DETECTION + BUZZER 1 BEEP ====
  int touchValue = touchRead(BAND_TOUCH_PIN);
  if (touchValue > 350) {
    if (!bandRemoved) {
      bandRemoved = true;

      // Buzzer single beep
      digitalWrite(BUZZER_PIN, HIGH);
      delay(300);
      digitalWrite(BUZZER_PIN, LOW);

      String msg = "⚠️ Band Removed!\nTime: " + getTimeNow().time + "\nDate: " + getTimeNow().date;
      if (gps.location.isValid()) msg += "\nLocation: https://maps.google.com/?q=" + String(gps.location.lat(),6) + "," + String(gps.location.lng(),6);
      sendMessage(msg);

      display.clearDisplay();
      display.setCursor(0, 0);
      display.println("Band Removed!");
      display.println("Time: " + getTimeNow().time);
      display.println("Date: " + getTimeNow().date);
      display.display();
    }
  } else {
    bandRemoved = false;
  }

  // Tracking interval
  if (tracking && gps.location.isValid() && millis() - lastSend > interval) {
    sendLocationTelegram();
    lastSend = millis();
  }

  // OLED display
  TimeInfo t = getTimeNow();
  display.clearDisplay();
  display.setCursor(0, 0);
  display.println("Health Monitoring");
  display.println("Day: " + t.day);
  display.println("Time: " + t.time);
  display.println("Date: " + t.date);
  display.print("Temp: "); display.print(tempC); display.println(" C");
  display.print("BPM: "); display.println(bpmSmoothed);

  if (gps.location.isValid()) {
    display.print("Lat:"); display.print(gps.location.lat(), 2);
    display.print(",Lng:"); display.println(gps.location.lng(), 2);
  } else {
    display.println("No GPS Fix");
  }

  if (sosActive) display.println("Status: SOS Triggered");
  else if (tracking) display.println("Status: Tracking");
  else if (bandRemoved) display.println("Status: Band Removed");
  else display.println("Status: Normal");

  display.display();

  // Serial output
  Serial.println("----- STATUS -----");
  Serial.println("Day: " + t.day);
  Serial.println("Time: " + t.time);
  Serial.println("Date: " + t.date);
  Serial.print("Temp: "); Serial.println(tempC);
  Serial.print("BPM: "); Serial.println(bpmSmoothed);
  if (gps.location.isValid()) {
    Serial.print("Lat: "); Serial.println(gps.location.lat(),2);
    Serial.print("Lng: "); Serial.println(gps.location.lng(),2);
  } else Serial.println("GPS not fixed");

  if (sosActive) Serial.println("Status: SOS Triggered");
  else if (tracking) Serial.println("Status: Tracking");
  else if (bandRemoved) Serial.println("Status: Band Removed");
  else Serial.println("Status: Normal");
  Serial.println("-----------------");

  delay(1000);
}
