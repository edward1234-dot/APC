# send.py — the Pi end of the link.
# Type a line, press Enter, it goes down the cable. Ctrl-C to stop.

import serial
import time

ser = serial.Serial('/dev/ttyUSB0', 9600, timeout=0.2)   # the same 9600 as the sketch
time.sleep(2)                        # the Uno resets when the port opens. Wait it out.
print(ser.readline().decode().strip())   # the "ready" line the sketch prints

try:
    while True:
        msg = input('> ')
        ser.write((msg + '\n').encode())   # a line = the message, then '\n'
        time.sleep(0.1)
        reply = ser.readline().decode().strip()
        if reply:
            print(reply)
finally:
    ser.close()
