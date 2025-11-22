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

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

#define TEMP_PIN 4
#define BUTTON_PIN 14
#define RXD2 16
#define TXD2 17
#define PULSE_PIN 33 

OneWire oneWire(TEMP_PIN);
DallasTemperature sensors(&oneWire);
TinyGPSPlus gps;
HardwareSerial GPSSerial(1);

const char* ssid = "Manu";
const char* password = "123456789";
#define BOT_TOKEN "8336100722:AAHrYkU5WoDG2kCbX1K02AVv9yUUP-v3SoY"
#define CHAT_ID   "8279358984"

WiFiClientSecure secured_client;
UniversalTelegramBot bot(BOT_TOKEN, secured_client);

const char* ntpServer = "pool.ntp.org";
const long gmtOffset_sec = 19800; 
const int daylightOffset_sec = 0;

unsigned long lastCheck = 0;
bool sosActive = false;
bool tracking = false;
unsigned long lastSend = 0;
const unsigned long interval = 30000; 

int bpm = 0;
int bpmSmoothed = 0;
int bpmHistory[10] = {0};
int historyIndex = 0;
unsigned long lastBeat = 0;
int threshold = 2000; 

struct TimeInfo {
  String day;
  String time;
  String date;
};

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
  strftime(timeBuffer, 10, "%H:%M:%S", &timeinfo);  e
  strftime(dateBuffer, 15, "%d-%m-%Y", &timeinfo);  e

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

void readHeartbeat() {
  int val = analogRead(PULSE_PIN);
  unsigned long currentTime = millis();

  if (val > threshold) {
    int beatDuration = currentTime - lastBeat;
    if (beatDuration > 300) { 
      bpm = 60000 / beatDuration;
      lastBeat = currentTime;

      if (bpm < 55) bpm = 55;
      if (bpm > 120) bpm = 120;

      bpmHistory[historyIndex] = bpm;
      historyIndex = (historyIndex + 1) % 10;
    }
  }

  long sum = 0;
  for (int i = 0; i < 10; i++) sum += bpmHistory[i];
  bpmSmoothed = sum / 10;

  if (bpmSmoothed == 0) bpmSmoothed = 75;
  if (bpmSmoothed < 55) bpmSmoothed = 55;
  if (bpmSmoothed > 120) bpmSmoothed = 120;

  Serial.print("Heartbeat BPM: ");
  Serial.println(bpmSmoothed);
}

void setup() {
  Serial.begin(115200);
  GPSSerial.begin(9600, SERIAL_8N1, RXD2, TXD2);
  sensors.begin();

  pinMode(BUTTON_PIN, INPUT_PULLUP);

  Wire.begin(21, 22);
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("SSD1306 not found");
    for (;;);
  }
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.display();

  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); }
  Serial.println(" connected!");

  secured_client.setInsecure(); 

  setupTime();

  Serial.println("Women Safety System Ready!");
  sendMessage("Women Safety System Online!");
}

void loop() {
  // GPS
  while (GPSSerial.available() > 0) gps.encode(GPSSerial.read());

  checkTelegramCommands();

  sensors.requestTemperatures();
  float tempC = sensors.getTempCByIndex(0);
  if (tempC > 38.0) sendMessage("High Temp Alert: " + String(tempC) + "C\n" + getTimeNow().date);

  readHeartbeat();

  if (digitalRead(BUTTON_PIN) == LOW) {
    unsigned long pressStart = millis();
    while (digitalRead(BUTTON_PIN) == LOW) {
      if (millis() - pressStart > 2000 && !sosActive) {
        sosActive = true;
        String msg = "SOS Triggered\nTime: " + getTimeNow().time + "\nDate: " + getTimeNow().date + "\nDay: " + getTimeNow().day;
        if (gps.location.isValid()) msg += "\nLocation: https://maps.google.com/?q=" + String(gps.location.lat(),6) + "," + String(gps.location.lng(),6);
        sendMessage(msg);

        display.clearDisplay();
        display.setCursor(0, 0);
        display.println("SOS Triggered");
        display.println("Time: " + getTimeNow().time);
        display.println("Date: " + getTimeNow().date);
        display.println("Day: " + getTimeNow().day);
        display.display();
      }
    }
  } else { sosActive = false; }

  if (tracking && gps.location.isValid() && millis() - lastSend > interval) {
    sendLocationTelegram();
    lastSend = millis();
  }

  TimeInfo t = getTimeNow();
  display.clearDisplay();
  display.setCursor(0, 0);
  display.println("Women Safety");
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
  else display.println("Status: Normal");

  display.display();

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
  else Serial.println("Status: Normal");
  Serial.println("-----------------");

  delay(1000);
}
