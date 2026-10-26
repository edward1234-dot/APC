// motor.ino — listen.ino plus a motor.
// "1" / "0"      LED on / off (as before)
// "M,<speed>"    motor: -255 (full reverse) .. 0 (stop) .. 255 (full forward)
// Anything else: ignored.
// Wiring is the Lesson 2 table: IN1 D7, IN2 D8, ENA D9 (jumper off),
// battery on the L298N, driver GND to Uno GND.

const int LED = 13;
const int IN1 = 7;
const int IN2 = 8;
const int ENA = 9;

void setMotor(int speed) {
  speed = constrain(speed, -255, 255);
  digitalWrite(IN1, speed > 0);      // forward when positive
  digitalWrite(IN2, speed < 0);      // reverse when negative
  analogWrite(ENA, abs(speed));      // size is the speed
}

void setup() {
  pinMode(LED, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(ENA, OUTPUT);
  setMotor(0);                       // a known state before anything arrives
  Serial.begin(9600);
  Serial.println("ready");
}

void loop() {
  if (Serial.available() > 0) {
    String line = Serial.readStringUntil('\n');
    line.trim();

    if (line == "1") {
      digitalWrite(LED, HIGH);
    } else if (line == "0") {
      digitalWrite(LED, LOW);
    } else if (line.startsWith("M,")) {
      int speed = line.substring(2).toInt();   // "M,-80" -> -80 
      setMotor(speed);
      Serial.print("ok ");                     // tell the Pi what we did
      Serial.println(speed);
    }
    // Anything else: not in the agreement. Do nothing.
  }
}
