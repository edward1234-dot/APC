/*
 * Sentinel - indicator board (Arduino side)
 * ------------------------------------------
 * The Raspberry Pi does the sensing; this board handles what people see:
 * a flashing alarm LED, a steady "armed" LED, and a 16x2 I2C LCD that
 * shows the detection count.
 *
 * Serial protocol (9600 baud, one command per line):
 *   ALARM        start flashing the alarm LED
 *   COUNT <n>    show n detections on the LCD
 *   RESET        stop the alarm, zero the count, show ARMED
 *   STATUS_OFF   everything off, LCD shows OFF
 *
 * Pins: alarm LED D8, status LED D9, LCD SDA->A4, SCL->A5 (address 0x27)
 */

#include <Wire.h>
#include <LiquidCrystal_I2C.h>

namespace pins {
  const uint8_t ALARM  = 8;
  const uint8_t STATUS = 9;
}

const uint16_t FLASH_PERIOD_MS = 150;

LiquidCrystal_I2C screen(0x27, 16, 2);

enum class Mode { ARMED, ALARM, OFF };

Mode mode = Mode::ARMED;
long detectionCount = 0;
unsigned long lastFlash = 0;
String inbox;

// ---------------------------------------------------------------- display --
void refreshScreen() {
  screen.clear();
  screen.setCursor(0, 0);
  screen.print("Detections: ");
  screen.print(detectionCount);
  screen.setCursor(0, 1);
  switch (mode) {
    case Mode::ARMED: screen.print("Status: ARMED");  break;
    case Mode::ALARM: screen.print("Status: ALARM!"); break;
    case Mode::OFF:   screen.print("Status: OFF");    break;
  }
}

void setMode(Mode next) {
  mode = next;
  switch (mode) {
    case Mode::ARMED:
      digitalWrite(pins::ALARM, LOW);
      digitalWrite(pins::STATUS, HIGH);
      break;
    case Mode::ALARM:
      digitalWrite(pins::STATUS, HIGH);
      break;
    case Mode::OFF:
      digitalWrite(pins::ALARM, LOW);
      digitalWrite(pins::STATUS, LOW);
      break;
  }
  refreshScreen();
}

// --------------------------------------------------------------- commands --
void dispatch(const String &cmd) {
  if (cmd == "ALARM") {
    setMode(Mode::ALARM);
  } else if (cmd.startsWith("COUNT ")) {
    detectionCount = cmd.substring(6).toInt();
    refreshScreen();
  } else if (cmd == "RESET") {
    detectionCount = 0;
    setMode(Mode::ARMED);
  } else if (cmd == "STATUS_OFF") {
    setMode(Mode::OFF);
  }
}

// Collect characters until a newline, then act on the line.
void pollSerial() {
  while (Serial.available()) {
    char c = Serial.read();
    if (c == '\n') {
      inbox.trim();
      dispatch(inbox);
      inbox = "";
    } else if (c != '\r') {
      inbox += c;
    }
  }
}

// ------------------------------------------------------------------ setup --
void setup() {
  pinMode(pins::ALARM, OUTPUT);
  pinMode(pins::STATUS, OUTPUT);
  Serial.begin(9600);
  inbox.reserve(16);

  screen.init();
  screen.backlight();
  setMode(Mode::ARMED);
}

void loop() {
  pollSerial();

  if (mode == Mode::ALARM && millis() - lastFlash >= FLASH_PERIOD_MS) {
    lastFlash = millis();
    digitalWrite(pins::ALARM, !digitalRead(pins::ALARM));
  }
}
