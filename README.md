#include <ESP8266WiFi.h>       
#include <ESP8266WebServer.h>  
#include <EEPROM.h>            

const int analogPin1 = A0;  
const int analogPin2 = D6;  
const int output1 = D1;
const int output2 = D2;
const int output3 = D3;

int eqSetting, crossoverSetting, limiterSetting;
int lowSetting, midSetting, subSetting;

ESP8266WebServer server(80);

void saveSettings() {
  EEPROM.put(0, eqSetting);
  EEPROM.put(4, crossoverSetting);
  EEPROM.put(8, limiterSetting);
  EEPROM.put(12, lowSetting);
  EEPROM.put(16, midSetting);
  EEPROM.put(20, subSetting);
  EEPROM.commit();
}

void loadSettings() {
  EEPROM.get(0, eqSetting);
  EEPROM.get(4, crossoverSetting);
  EEPROM.get(8, limiterSetting);
  EEPROM.get(12, lowSetting);
  EEPROM.get(16, midSetting);
  EEPROM.get(20, subSetting);
}

void handleRoot() {
  String html = "<h1>DLMS Management</h1>";
  html += "<form action='/update' method='POST'>";
  html += "<p>EQ: <input type='number' name='eq' value='" + String(eqSetting) + "'></p>";
  html += "<p>Crossover: <input type='number' name='crossover' value='" + String(crossoverSetting) + "'></p>";
  html += "<p>Limiter: <input type='number' name='limiter' value='" + String(limiterSetting) + "'></p>";
  html += "<p>Low: <input type='number' name='low' value='" + String(lowSetting) + "'></p>";
  html += "<p>Mid: <input type='number' name='mid' value='" + String(midSetting) + "'></p>";
  html += "<p>Sub: <input type='number' name='sub' value='" + String(subSetting) + "'></p>";
  html += "<input type='submit' value='Update'>";
  html += "</form>";
  html += "<p>Device IP: " + WiFi.localIP().toString() + "</p>";
  
  server.send(200, "text/html", html);
}

void handleUpdate() {
  if (server.hasArg("eq")) eqSetting = server.arg("eq").toInt();
  if (server.hasArg("crossover")) crossoverSetting = server.arg("crossover").toInt();
  if (server.hasArg("limiter")) limiterSetting = server.arg("limiter").toInt();
  if (server.hasArg("low")) lowSetting = server.arg("low").toInt();
  if (server.hasArg("mid")) midSetting = server.arg("mid").toInt();
  if (server.hasArg("sub")) subSetting = server.arg("sub").toInt();

  saveSettings();
  server.sendHeader("Location", "/");
  server.send(303);
}

void setup() {
  Serial.begin(115200);

  // Konfigurasi IP statis (opsional)
  IPAddress local_IP(192, 168, 1, 184);
  IPAddress gateway(192, 168, 1, 1);
  IPAddress subnet(255, 255, 255, 0);
  
  if (!WiFi.config(local_IP, gateway, subnet)) {
    Serial.println("Gagal mengonfigurasi IP statis");
  }

  // Koneksi Wi-Fi
  WiFi.begin("SSID", "PASSWORD");  
  Serial.print("Menghubungkan ke Wi-Fi");
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 30) { 
    delay(1000);
    Serial.print(".");
    attempts++;
  }
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println();
    Serial.println("Terhubung ke Wi-Fi");
    Serial.print("Alamat IP: ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println();
    Serial.println("Gagal terhubung ke Wi-Fi");
    // Coba konfigurasi ulang dengan DHCP jika gagal
    WiFi.config(0U, 0U, 0U);
    WiFi.begin("SSID", "PASSWORD");
    if (WiFi.status() != WL_CONNECTED) {
      Serial.println("Gagal dengan DHCP juga. Periksa pengaturan.");
      while (true);  
    }
  }

  server.on("/", handleRoot);
  server.on("/update", HTTP_POST, handleUpdate);
  server.begin();
  Serial.println("Server dimulai");

  pinMode(analogPin1, INPUT);
  pinMode(analogPin2, INPUT);
  pinMode(output1, OUTPUT);
  pinMode(output2, OUTPUT);
  pinMode(output3, OUTPUT);

  EEPROM.begin(512); 
  loadSettings();
}

void loop() {
  server.handleClient();

  eqSetting = analogRead(analogPin1); 
  delay(50);

  crossoverSetting = eqSetting / 4;
  limiterSetting = eqSetting / 8;

  lowSetting = analogRead(analogPin2); 
  delay(50);

  midSetting = lowSetting / 4;
  subSetting = lowSetting / 8;

  analogWrite(output1, lowSetting);
  analogWrite(output2, midSetting);
  analogWrite(output3, subSetting);

  saveSettings();
}
