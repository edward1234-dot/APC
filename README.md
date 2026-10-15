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
 *
 * No extra libraries needed: the LCD driver is built into this sketch
 * and only uses the Wire library that ships with the Arduino IDE.
 */

#include <Wire.h>

// ------------------------------------------------- built-in LCD driver ----
// Minimal driver for a 16x2 HD44780 LCD behind a PCF8574 I2C backpack.
// Inherits from Print so screen.print("text") and screen.print(number) work.
class TinyLCD : public Print {
public:
  TinyLCD(uint8_t addr, uint8_t cols, uint8_t rows)
    : _addr(addr), _cols(cols), _rows(rows), _backlight(0x08) {}

  void init() {
    Wire.begin();
    delay(50);
    expanderWrite(_backlight);
    delay(1000);

    // Put the LCD into 4-bit mode (standard HD44780 start-up sequence)
    write4bits(0x03 << 4); delayMicroseconds(4500);
    write4bits(0x03 << 4); delayMicroseconds(4500);
    write4bits(0x03 << 4); delayMicroseconds(150);
    write4bits(0x02 << 4);

    command(0x28);  // 4-bit mode, 2 lines, 5x8 font
    command(0x0C);  // display on, cursor off, blink off
    command(0x06);  // cursor moves right after each character
    clear();
  }

  void clear() {
    command(0x01);
    delay(2);
  }

  void setCursor(uint8_t col, uint8_t row) {
    static const uint8_t rowOffsets[] = {0x00, 0x40, 0x14, 0x54};
    if (row >= _rows) row = _rows - 1;
    command(0x80 | (col + rowOffsets[row]));
  }

  void backlight()   { _backlight = 0x08; expanderWrite(0); }
  void noBacklight() { _backlight = 0x00; expanderWrite(0); }

  // Called by Print for every character
  size_t write(uint8_t value) override {
    send(value, RS_BIT);
    return 1;
  }

private:
  // PCF8574 pin mapping used by the common backpack:
  // P0=RS, P1=RW, P2=EN, P3=backlight, P4..P7 = D4..D7
  static const uint8_t RS_BIT = 0x01;
  static const uint8_t EN_BIT = 0x04;

  uint8_t _addr, _cols, _rows, _backlight;

  void command(uint8_t value) { send(value, 0); }

  void send(uint8_t value, uint8_t mode) {
    write4bits((value & 0xF0) | mode);
    write4bits(((value << 4) & 0xF0) | mode);
  }

  void write4bits(uint8_t value) {
    expanderWrite(value);
    pulseEnable(value);
  }

  void pulseEnable(uint8_t value) {
    expanderWrite(value | EN_BIT);
    delayMicroseconds(1);
    expanderWrite(value & ~EN_BIT);
    delayMicroseconds(50);
  }

  void expanderWrite(uint8_t value) {
    Wire.beginTransmission(_addr);
    Wire.write(value | _backlight);
    Wire.endTransmission();
  }
};

// ----------------------------------------------------------- main sketch --
namespace pins {
  const uint8_t ALARM  = 8;
  const uint8_t STATUS = 9;
}

const uint16_t FLASH_PERIOD_MS = 150;

TinyLCD screen(0x27, 16, 2);   // change 0x27 to 0x3F if your LCD stays blank

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
