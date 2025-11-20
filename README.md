# DIG3602-Final-Project
Code for the Arduino final project:


#include <SoftwareSerial.h>

SoftwareSerial mySerial(3, 2); // Bluetooth TX, RX

// Pins
const int buttonPin = 4;  // Arcade button (one side to pin4, other to GND; use INPUT_PULLUP)
const int ledPin = 13;    // LED feedback

// Debounce / timing
const unsigned long debounceDelay = 50;   // ms
const unsigned long tapThreshold   = 300; // max duration (ms) to count as a tap
const unsigned long comboTimeout   = 500; // ms to wait after last release before acting on taps

// State tracking
int rawState = HIGH;
int stableState = HIGH;      // debounced stable state
int lastStableState = HIGH;
unsigned long lastDebounceTime = 0;

unsigned long pressStartTime = 0;
unsigned long lastReleaseTime = 0;

int pressCount = 0;

void setup() {
  Serial.begin(9600);
  mySerial.begin(38400);

  pinMode(buttonPin, INPUT_PULLUP);
  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, LOW);

  Serial.println("Two-tap controller (debounced) ready.");
}

void loop() {
  unsigned long now = millis();

  // Read raw pin
  int reading = digitalRead(buttonPin);

  // If the reading changed, reset the debounce timer
  if (reading != rawState) {
    rawState = reading;
    lastDebounceTime = now;
  }

  // If the reading has been stable longer than debounceDelay, accept it as the stable state
  if ((now - lastDebounceTime) > debounceDelay) {
    if (rawState != stableState) {
      lastStableState = stableState;
      stableState = rawState;

// Transition detected: LOW = pressed, HIGH = released (using INPUT_PULLUP)
      if (stableState == LOW) {
        // Button just pressed
        pressStartTime = now;
        digitalWrite(ledPin, HIGH); // LED on during press
        Serial.println("Pressed");
      } else {
// Button just released
        digitalWrite(ledPin, LOW); // LED off on release
        unsigned long pressDuration = now - pressStartTime;
        Serial.print("Released. Duration (ms): ");
        Serial.println(pressDuration);

  if (pressDuration <= tapThreshold) {
          // Count as a tap
          pressCount++;
          lastReleaseTime = now;
          Serial.print("Tap counted. pressCount = ");
          Serial.println(pressCount);
        } else {
          // It was a long press - for this simplified mode we ignore it
          Serial.println("Long press (ignored in this mode).");
          // Optionally reset pressCount if you want long press to cancel sequence:
          // pressCount = 0;
        }
      }
    }
  }

  // If we have taps and enough time has passed since last release, interpret the sequence
  if (pressCount > 0 && (now - lastReleaseTime) > comboTimeout) {
    if (pressCount == 1) {
      Serial.println("Action: FORWARD (1 tap)");
      mySerial.println("F");
    } else if (pressCount == 2) {
      Serial.println("Action: STOP (2 taps)");
      mySerial.println("S");
    } else {
      Serial.print("Ignored sequence of ");
      Serial.print(pressCount);
      Serial.println(" taps.");
    }
    pressCount = 0; // reset
  }

  // loop fast
}
