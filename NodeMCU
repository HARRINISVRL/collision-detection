#include <Wire.h>
#include <MPU6050.h>
#include <ThingSpeak.h>
#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>

MPU6050 mpu;

// ----- USER CONFIG -----
#define SECRET_CH_ID        1234567UL        // replace with your channel number
#define SECRET_WRITE_APIKEY "YOUR_WRITE_KEY" // replace with your ThingSpeak Write API Key
const char* ssid = "YOUR_SSID";
const char* password = "YOUR_WIFI_PASSWORD";
const char* webhookURL = ""; // optional webhook URL
unsigned long channelID = SECRET_CH_ID;
const char* writeAPIKey = SECRET_WRITE_APIKEY;

// Pins
const int vibrationSensorPin = D1;
const int interruptPin = D2;
const int ignitionDisablePin = D5;
const int alcoholSensorPin = A0;

// ----- PARAMETERS -----
const float accelScale = 16384.0;
const float gyroScale  = 131.0;
const float emaAlpha = 0.02;
float accelBaseline = 0.0;
const float wAccel = 0.6, wGyro = 0.2, wVibe = 0.1, wAlcohol = 0.4;
const float accelSpikeThresholdG = 2.0;
const int vibrationActive = HIGH;
const int alcoholThresholdRaw = 300;
const float confirmScoreThreshold = 0.8;
const unsigned long confirmWindowMs = 1500;
const unsigned long postDelayMs = 5000;
const int ignitionDisableHoldMs = 60000;

bool ignitionDisabled = false;
unsigned long ignitionDisabledUntil = 0;

WiFiClient client;

volatile unsigned long lastMpuInterruptAt = 0;
volatile bool mpuInterruptFlag = false;
void IRAM_ATTR handleInterrupt() {
  mpuInterruptFlag = true;
  lastMpuInterruptAt = millis();
}

void sendAlert(float ax_g, float ay_g, float az_g, float gx_dps, float gy_dps, float gz_dps, int vibration, int alcoholRaw, const char* severity) {
  ThingSpeak.setField(1, ax_g);
  ThingSpeak.setField(2, ay_g);
  ThingSpeak.setField(3, az_g);
  ThingSpeak.setField(4, gx_dps);
  ThingSpeak.setField(5, gy_dps);
  ThingSpeak.setField(6, gz_dps);
  ThingSpeak.setField(7, vibration);
  ThingSpeak.setField(8, alcoholRaw);
  ThingSpeak.setStatus(String("Severity: ") + severity);
  ThingSpeak.writeFields(channelID, writeAPIKey);

  if (webhookURL && strlen(webhookURL) > 5) {
    HTTPClient http;
    http.begin(client, webhookURL);
    http.addHeader("Content-Type", "application/json");
    String payload = "{";
    payload += "\"ax\":" + String(ax_g, 3) + ",";
    payload += "\"ay\":" + String(ay_g, 3) + ",";
    payload += "\"az\":" + String(az_g, 3) + ",";
    payload += "\"gx\":" + String(gx_dps, 3) + ",";
    payload += "\"gy\":" + String(gy_dps, 3) + ",";
    payload += "\"gz\":" + String(gz_dps, 3) + ",";
    payload += "\"vibration\":" + String(vibration) + ",";
    payload += "\"alcohol\":" + String(alcoholRaw) + ",";
    payload += "\"severity\":\"" + String(severity) + "\"";
    payload += "}";
    int httpCode = http.POST(payload);
    Serial.printf("Webhook POST: %d\n", httpCode);
    http.end();
  }
}

void setup() {
  Serial.begin(115200);
  delay(200);

  pinMode(vibrationSensorPin, INPUT);
  pinMode(interruptPin, INPUT_PULLUP);
  pinMode(ignitionDisablePin, OUTPUT);
  digitalWrite(ignitionDisablePin, LOW);

  attachInterrupt(digitalPinToInterrupt(interruptPin), handleInterrupt, RISING);

  Wire.begin();
  mpu.initialize();
  mpu.setDMPEnabled(false);
  accelBaseline = 0.0;

  Serial.println("Connecting to WiFi...");
  WiFi.begin(ssid, password);
  unsigned long startWiFi = millis();
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
    if (millis() - startWiFi > 15000) {
      Serial.println("\nWiFi connect timeout, continuing.");
      break;
    }
  }
  Serial.println("\nWiFi connected.");
  ThingSpeak.begin(client);
}

// ----- TIMING CONTROL -----
const unsigned long sampleInterval = 120; // sampling interval in ms
unsigned long lastSampleTime = 0;

unsigned long lastEventAt = 0;
bool eventArmed = true;

void loop() {
  unsigned long currentMillis = millis();

  if (currentMillis - lastSampleTime >= sampleInterval) {
    lastSampleTime = currentMillis;

    int16_t ax_raw, ay_raw, az_raw, gx_raw, gy_raw, gz_raw;
    mpu.getAcceleration(&ax_raw, &ay_raw, &az_raw);
    mpu.getRotation(&gx_raw, &gy_raw, &gz_raw);

    float ax_g = (float)ax_raw / accelScale;
    float ay_g = (float)ay_raw / accelScale;
    float az_g = (float)az_raw / accelScale;
    float gx_dps = (float)gx_raw / gyroScale;
    float gy_dps = (float)gy_raw / gyroScale;
    float gz_dps = (float)gz_raw / gyroScale;

    float accelMag = sqrt(ax_g * ax_g + ay_g * ay_g + az_g * az_g);
    if (accelBaseline == 0.0) accelBaseline = accelMag;
    accelBaseline = (1.0 - emaAlpha) * accelBaseline + emaAlpha * accelMag;

    float accelDeltaG = accelMag - accelBaseline;
    float accelScore = constrain(accelDeltaG / accelSpikeThresholdG, 0.0, 1.0);
    float gyroMag = sqrt(gx_dps * gx_dps + gy_dps * gy_dps + gz_dps * gz_dps);
    float gyroScore = constrain(gyroMag / 200.0, 0.0, 1.0);

    int vibrationValue = digitalRead(vibrationSensorPin);
    float vibeScore = (vibrationValue == vibrationActive) ? 1.0 : 0.0;

    int alcoholRaw = analogRead(alcoholSensorPin);
    bool alcoholFlag = (alcoholRaw >= alcoholThresholdRaw);
    float alcoholScore = alcoholFlag ? 1.0 : 0.0;

    float fusionScore = (wAccel * accelScore + wGyro * gyroScore + wVibe * vibeScore + wAlcohol * alcoholScore) / (wAccel + wGyro + wVibe + wAlcohol);

    Serial.printf("a=%.2fg base=%.2fg d=%.2fg s=%.2f g=%.1fdps gs=%.2f v=%d vs=%.2f alc=%d fs=%.3f\n",
                  accelMag, accelBaseline, accelDeltaG, accelScore, gyroMag, gyroScore, vibrationValue, vibeScore, alcoholRaw, fusionScore);

    static unsigned long confirmStart = 0;
    if (fusionScore >= confirmScoreThreshold) {
      if (confirmStart == 0) confirmStart = currentMillis;
      if (currentMillis - confirmStart >= confirmWindowMs && eventArmed) {
        const char* severity = "light";
        if (accelDeltaG > 4.0) severity = "severe";
        else if (accelDeltaG > 2.5) severity = "moderate";

        Serial.println(">>> Collision CONFIRMED! Sending alert...");
        sendAlert(ax_g, ay_g, az_g, gx_dps, gy_dps, gz_dps, vibrationValue, alcoholRaw, severity);

        lastEventAt = currentMillis;
        eventArmed = false;
        confirmStart = 0;

        bool vehicleStationary = (accelBaseline < 1.2 && accelDeltaG < 0.2);
        if (alcoholFlag && vehicleStationary) {
          Serial.println("Alcohol + Stationary -> Ignition disabled.");
          digitalWrite(ignitionDisablePin, HIGH);
          ignitionDisabled = true;
          ignitionDisabledUntil = currentMillis + ignitionDisableHoldMs;
        }
      }
    } else {
      confirmStart = 0;
    }
  }

  if (!eventArmed && millis() - lastEventAt > postDelayMs) {
    eventArmed = true;
    Serial.println("Re-armed.");
  }

  if (ignitionDisabled && millis() > ignitionDisabledUntil) {
    Serial.println("Re-enabling ignition.");
    digitalWrite(ignitionDisablePin, LOW);
    ignitionDisabled = false;
  }
}
