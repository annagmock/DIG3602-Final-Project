FINAL WORKING CONTROLLER CODE:


#include <SoftwareSerial.h>

SoftwareSerial mySerial(3, 2); // Bluetooth TX, RX

// Pins
const int buttonPin = 4;  // Arcade button
const int ledPin = 13;    // LED feedback

// Debounce / timing
const unsigned long debounceDelay = 50;
const unsigned long tapThreshold   = 300;
const unsigned long comboTimeout   = 500;

int rawState = HIGH;
int stableState = HIGH;
int lastStableState = HIGH;
unsigned long lastDebounceTime = 0;

unsigned long pressStartTime = 0;
unsigned long lastReleaseTime = 0;
int pressCount = 0;

bool turningActive = false;
char turnDirection = 0;

// ===== BURST SETTINGS =====
const int burstCount = 80;   // spam even more if you want

// Sends command AS FAST AS POSSIBLE (no delay)
void sendBurst(const char* cmd) {
  for (int i = 0; i < burstCount; i++) {
    mySerial.println(cmd);
    Serial.println(cmd);
    // no delay, full speed spam
  }
}

void setup() {
  Serial.begin(9600);
  mySerial.begin(38400);

  pinMode(buttonPin, INPUT_PULLUP);
  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, LOW);
}

void loop() {
  unsigned long now = millis();
  int reading = digitalRead(buttonPin);

  // ---------------- Debounce ----------------
  if (reading != rawState) {
    rawState = reading;
    lastDebounceTime = now;
  }

  if ((now - lastDebounceTime) > debounceDelay) {
    if (rawState != stableState) {
      lastStableState = stableState;
      stableState = rawState;

      // ========== BUTTON PRESSED ==========
      if (stableState == LOW) {
        pressStartTime = now;
        digitalWrite(ledPin, HIGH);
      }

      // ========== BUTTON RELEASED ==========
      else {
        digitalWrite(ledPin, LOW);
        unsigned long pressDuration = now - pressStartTime;

        // If released while turning -> STOP
        if (turningActive) {
          sendBurst("S");
          turningActive = false;
          turnDirection = 0;
          pressCount = 0;
          return;
        }

        // Short press = tap
        if (pressDuration <= tapThreshold) {
          pressCount++;
          lastReleaseTime = now;
        }
      }
    }
  }

  // ====== CHECK WHILE HELD FOR HOLD ACTION ======
  if (stableState == LOW && !turningActive) {
    unsigned long heldTime = millis() - pressStartTime;

    if (heldTime > tapThreshold) {

      // HOLD with ZERO taps = LEFT
      if (pressCount == 0) {
        sendBurst("L");
        turningActive = true;
        turnDirection = 'L';
        return;
      }

      // HOLD after 1 tap = RIGHT
      if (pressCount == 1) {
        sendBurst("R");
        turningActive = true;
        turnDirection = 'R';
        return;
      }
    }
  }

  // ====== TAP COMBO AFTER TIMEOUT ======
  if (pressCount > 0 && (now - lastReleaseTime) > comboTimeout && !turningActive) {

    if (pressCount == 1) sendBurst("F");
    else if (pressCount == 2) sendBurst("S");
    else if (pressCount == 3) sendBurst("B");

    pressCount = 0;
  }
}
