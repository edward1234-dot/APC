from adafruit_servokit import ServoKit
import time
kit = ServoKit(channels=16)
s = kit.servo[0]
s.set_pulse_width_range(500, 2500)
for a in (0, 90, 180, 90):
    s.angle = a
    time.sleep(1.5)
