FINAL WORKING CONTROLLER CODE:


#include <SoftwareSerial.h>

SoftwareSerial mySerial(3, 2); // Bluetooth TX, RX

const int buttonPin = 4;
const int ledPin = 13;

const unsigned long debounceDelay = 50;
const unsigned long tapThreshold = 300;
const unsigned long comboTimeout = 500;

int rawState = HIGH;
int stableState = HIGH;
int lastStableState = HIGH;
unsigned long lastDebounceTime = 0;

unsigned long pressStartTime = 0;
unsigned long lastReleaseTime = 0;
int pressCount = 0;

bool turningActive = false;
char activeCommand = 0;   // <- streamed every loop

void sendCmd(const char* cmd) {
  mySerial.println(cmd);
  Serial.println(cmd);
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

  // ========== DEBOUNCE ==========
  if (reading != rawState) {
    rawState = reading;
    lastDebounceTime = now;
  }

  if ((now - lastDebounceTime) > debounceDelay) {

    if (rawState != stableState) {
      lastStableState = stableState;
      stableState = rawState;

      // ======== PRESSED ========
      if (stableState == LOW) {
        pressStartTime = now;
        digitalWrite(ledPin, HIGH);
      }

      // ======== RELEASED ========
      else {
        digitalWrite(ledPin, LOW);
        unsigned long pressDuration = now - pressStartTime;

        // If we were turning -> STOP that streaming
        if (turningActive) {
          activeCommand = 'S';   // stream S until next command
          turningActive = false;
          pressCount = 0;
          return;
        }

        // Short tap
        if (pressDuration <= tapThreshold) {
          pressCount++;
          lastReleaseTime = now;
        }
      }
    }
  }

  // ======== HOLD DETECTION ========
  if (stableState == LOW && !turningActive) {
    unsigned long held = now - pressStartTime;

    if (held > tapThreshold) {

      // HOLD = LEFT
      if (pressCount == 0) {
        activeCommand = 'L';
        turningActive = true;
        return;
      }

      // TAP + HOLD = RIGHT
      if (pressCount == 1) {
        activeCommand = 'R';
        turningActive = true;
        return;
      }
    }
  }

  // ======== TAP COMBOS ========
  if (pressCount > 0 && (now - lastReleaseTime) > comboTimeout && !turningActive) {

    if (pressCount == 1) activeCommand = 'F';  // stream forward
    else if (pressCount == 2) activeCommand = 'S';  // stream stop
    else if (pressCount == 3) activeCommand = 'B';  // stream backward

    pressCount = 0;
  }

  // ======== CONTINUOUS STREAM (NO AUTO-STOP) ========
  if (activeCommand) {
    if      (activeCommand == 'L') sendCmd("L");
    else if (activeCommand == 'R') sendCmd("R");
    else if (activeCommand == 'F') sendCmd("F");
    else if (activeCommand == 'S') sendCmd("S");
    else if (activeCommand == 'B') sendCmd("B");
  }
}
