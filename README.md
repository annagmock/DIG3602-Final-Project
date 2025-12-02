FINAL WORKING CONTROLLER CODE:


#include <SoftwareSerial.h>

SoftwareSerial mySerial(3, 2); // Bluetooth TX, RX

// Pins
const int buttonPin = 4;  // Arcade button
const int ledPin = 13;    // LED feedback

// Debounce / timing
const unsigned long debounceDelay = 50;   // ms
const unsigned long tapThreshold   = 300; // ms (max for tap)
const unsigned long comboTimeout   = 500; // ms after last release

// State tracking
int rawState = HIGH;
int stableState = HIGH;
int lastStableState = HIGH;
unsigned long lastDebounceTime = 0;

unsigned long pressStartTime = 0;
unsigned long lastReleaseTime = 0;
int pressCount = 0;

bool turningActive = false;
char turnDirection = 0;  // 'L' or 'R'

// Continuous command output
char currentCommand = 'X';      // X = no movement
unsigned long lastRepeat = 0;
const unsigned long repeatInterval = 100; // send command every 100ms

void setup() {
  Serial.begin(9600);
  mySerial.begin(38400);

  pinMode(buttonPin, INPUT_PULLUP);
  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, LOW);

  Serial.println("Controller Ready: left hold, tap-hold right.");
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
        Serial.println("Pressed");
      }

      // ========== BUTTON RELEASED ==========
      else {
        digitalWrite(ledPin, LOW);
        unsigned long pressDuration = now - pressStartTime;
        Serial.print("Released. Duration = ");
        Serial.println(pressDuration);

        // If released while turning -> STOP (continuous)
        if (turningActive) {
          Serial.println("Action: STOP TURNING");
          mySerial.println("S");
          currentCommand = 'S';   // <-- continuous STOP
          turningActive = false;
          turnDirection = 0;
          pressCount = 0;
          return;
        }

        // Short press = tap
        if (pressDuration <= tapThreshold) {
          pressCount++;
          lastReleaseTime = now;
          Serial.print("Tap counted. pressCount = ");
          Serial.println(pressCount);
        }
        else {
          Serial.println("Long press ignored on release (we detect hold while pressed).");
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
        Serial.println("Action: TURN LEFT (hold)");
        mySerial.println("L");
        currentCommand = 'L';
        turningActive = true;
        turnDirection = 'L';
        return;
      }

      // HOLD after 1 tap = RIGHT
      if (pressCount == 1) {
        Serial.println("Action: TURN RIGHT (hold)");
        mySerial.println("R");
        currentCommand = 'R';
        turningActive = true;
        turnDirection = 'R';
        return;
      }
    }
  }

  // ====== TAP COMBO AFTER TIMEOUT ======
  if (pressCount > 0 && (now - lastReleaseTime) > comboTimeout && !turningActive) {
    switch (pressCount) {

      case 1:
        Serial.println("Action: FORWARD (1 tap)");
        mySerial.println("F");
        currentCommand = 'F';
        break;

      case 2:
        Serial.println("Action: STOP (2 taps)");
        mySerial.println("S");
        currentCommand = 'S';   // <-- continuous STOP
        break;

      case 3:
        Serial.println("Action: BACK (3 taps)");
        mySerial.println("B");
        currentCommand = 'B';
        break;

      default:
        Serial.print("Ignored sequence of ");
        Serial.print(pressCount);
        Serial.println(" taps.");
    }
    pressCount = 0;
  }

  // =====================================================
  // CONTINUOUSLY SEND ACTIVE COMMAND (every 100ms)
  // =====================================================
  if (currentCommand != 'X' && millis() - lastRepeat > repeatInterval) {
    mySerial.println(currentCommand);
    Serial.print("Repeat: ");
    Serial.println(currentCommand);
    lastRepeat = millis();
  }
}
