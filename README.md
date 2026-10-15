#!/usr/bin/env python3
"""
Sentinel - motion-activated alarm for Raspberry Pi
==================================================

Same behaviour as before, but the servo is driven by a small PCA9685
driver built into this file, so the adafruit ServoKit library is NOT needed.
It only uses smbus (preinstalled on Raspberry Pi OS, or: sudo apt install python3-smbus).

What it does
  * Watches a PIR sensor and counts every detection with a timestamp.
  * On detection: beeps the buzzer, tells the Arduino to flash the alarm
    LED and show the running count on the LCD, then sweeps a servo
    across the room through a PCA9685 driver board.
  * A push button resets the count and re-arms the system.

Hardware map
  PIR sensor .......... GPIO 27
  Buzzer .............. GPIO 22
  Reset button ........ GPIO 4   (3-pin module, idle HIGH, LOW when pressed)
  PCA9685 ............. I2C (address 0x40), servo on channel 0, V+ from external 5 V
  Arduino ............. USB serial (/dev/ttyUSB0) - LEDs and 16x2 LCD
"""

import sys
import time
from datetime import datetime

import serial
from gpiozero import Button, Buzzer, MotionSensor

try:
    from smbus2 import SMBus
except ImportError:
    try:
        from smbus import SMBus
    except ImportError:
        SMBus = None


# --------------------------------------------------------------------------
# Settings - tweak these without touching the logic below
# --------------------------------------------------------------------------
class Config:
    PIR_PIN = 27
    BUZZER_PIN = 22
    BUTTON_PIN = 4

    ARDUINO_PORT = "/dev/ttyUSB0"
    ARDUINO_BAUD = 9600

    I2C_BUS = 1              # Raspberry Pi header I2C bus
    PCA9685_ADDRESS = 0x40
    SERVO_CHANNEL = 0
    SERVO_MIN_US = 500       # pulse width at 0 degrees (microseconds)
    SERVO_MAX_US = 2500      # pulse width at 180 degrees (microseconds)
    SERVO_REST = 90          # degrees, where the servo parks
    SERVO_SWEEP = (180, 0)   # positions visited during a scan
    SERVO_PAUSE = 1.0        # seconds to hold each sweep position

    BEEPS = 5                # buzzer pulses per detection
    BEEP_LENGTH = 0.2        # seconds on / off


def now() -> str:
    """Timestamp used for every log line."""
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")


def log(message: str) -> None:
    print(f"[{now()}] {message}")
    sys.stdout.flush()


# --------------------------------------------------------------------------
# Minimal PCA9685 driver (replaces adafruit ServoKit)
# --------------------------------------------------------------------------
class PCA9685:
    MODE1 = 0x00
    PRESCALE = 0xFE
    LED0_ON_L = 0x06

    def __init__(self, bus_number: int, address: int, frequency: int = 50):
        self.bus = SMBus(bus_number)
        self.address = address
        self.period_us = 1_000_000 / frequency

        prescale = int(round(25_000_000 / (4096 * frequency)) - 1)
        self.bus.write_byte_data(self.address, self.MODE1, 0x10)       # sleep
        self.bus.write_byte_data(self.address, self.PRESCALE, prescale)
        self.bus.write_byte_data(self.address, self.MODE1, 0x20)       # wake, auto-increment
        time.sleep(0.005)
        self.bus.write_byte_data(self.address, self.MODE1, 0xA0)       # restart

    def set_pulse_us(self, channel: int, pulse_us: float) -> None:
        ticks = int(pulse_us / self.period_us * 4096)
        ticks = max(0, min(4095, ticks))
        reg = self.LED0_ON_L + 4 * channel
        self.bus.write_i2c_block_data(
            self.address, reg, [0, 0, ticks & 0xFF, ticks >> 8]
        )

    def off(self, channel: int) -> None:
        """Stop sending pulses on a channel (servo goes limp)."""
        reg = self.LED0_ON_L + 4 * channel
        self.bus.write_i2c_block_data(self.address, reg, [0, 0, 0, 0x10])


# --------------------------------------------------------------------------
# Arduino link - LEDs and LCD live on the other side of this serial port
# --------------------------------------------------------------------------
class ArduinoLink:
    def __init__(self, port: str, baud: int):
        self.port = serial.Serial(port, baud, timeout=1)
        time.sleep(2)  # the board reboots when the port opens

    def _send(self, line: str) -> None:
        self.port.write(f"{line}\n".encode())

    def alarm(self, count: int) -> None:
        self._send("ALARM")
        self._send(f"COUNT {count}")

    def rearm(self) -> None:
        self._send("RESET")

    def shutdown(self) -> None:
        self._send("STATUS_OFF")
        self.port.close()


# --------------------------------------------------------------------------
# Servo scanner - optional, the system still runs if the board is missing
# --------------------------------------------------------------------------
class Scanner:
    def __init__(self, channel: int):
        self.channel = channel
        self.driver = None
        if SMBus is None:
            log("smbus not installed - scanner disabled (sudo apt install python3-smbus)")
            return
        try:
            self.driver = PCA9685(Config.I2C_BUS, Config.PCA9685_ADDRESS)
            self.park()
        except Exception as exc:  # board missing or I2C fault
            log(f"PCA9685 not found ({exc}) - scanner disabled")
            self.driver = None

    def _move(self, angle: float) -> None:
        span = Config.SERVO_MAX_US - Config.SERVO_MIN_US
        pulse = Config.SERVO_MIN_US + (angle / 180.0) * span
        self.driver.set_pulse_us(self.channel, pulse)

    def park(self) -> None:
        if self.driver is not None:
            self._move(Config.SERVO_REST)

    def sweep(self) -> None:
        if self.driver is None:
            return
        log("Scanning area...")
        for angle in Config.SERVO_SWEEP:
            self._move(angle)
            time.sleep(Config.SERVO_PAUSE)
        self.park()
        time.sleep(0.5)


# --------------------------------------------------------------------------
# The alarm itself
# --------------------------------------------------------------------------
class Sentinel:
    def __init__(self):
        self.detections = 0

        self.pir = MotionSensor(Config.PIR_PIN)
        self.buzzer = Buzzer(Config.BUZZER_PIN)
        self.button = Button(
            Config.BUTTON_PIN, pull_up=None, active_state=False, bounce_time=0.05
        )

        self.arduino = ArduinoLink(Config.ARDUINO_PORT, Config.ARDUINO_BAUD)
        self.scanner = Scanner(Config.SERVO_CHANNEL)

        self.pir.when_motion = self.on_motion
        self.button.when_pressed = self.on_reset

    # -- event handlers ----------------------------------------------------
    def on_motion(self) -> None:
        self.detections += 1
        log(f"MOTION DETECTED  -> total so far: {self.detections}")
        self.arduino.alarm(self.detections)
        self.sound_alarm()
        self.scanner.sweep()

    def on_reset(self) -> None:
        log(f"RESET pressed - clearing {self.detections} detection(s)")
        self.detections = 0
        self.buzzer.off()
        self.scanner.park()
        self.arduino.rearm()
        log("System re-armed")

    # -- helpers -----------------------------------------------------------
    def sound_alarm(self) -> None:
        for _ in range(Config.BEEPS):
            self.buzzer.on()
            time.sleep(Config.BEEP_LENGTH)
            self.buzzer.off()
            time.sleep(Config.BEEP_LENGTH)

    def run(self) -> None:
        self.arduino.rearm()
        log("Sentinel armed - waiting for motion (Ctrl+C to stop)")
        try:
            while True:
                time.sleep(1)
        except KeyboardInterrupt:
            pass
        finally:
            self.buzzer.off()
            self.scanner.park()
            self.arduino.shutdown()
            log(f"Sentinel stopped after {self.detections} detection(s)")


if __name__ == "__main__":
    Sentinel().run()
