   import smbus, time
   bus = smbus.SMBus(1)
   bus.write_byte_data(0x40, 0x00, 0x10)
   bus.write_byte_data(0x40, 0xFE, 121)
   bus.write_byte_data(0x40, 0x00, 0x20)
   time.sleep(0.01)
   bus.write_byte_data(0x40, 0x00, 0xA0)
   for pulse in (1000, 1500, 2000, 1500):
       ticks = int(pulse / 20000 * 4096)
       bus.write_i2c_block_data(0x40, 0x06, [0, 0, ticks & 0xFF, ticks >> 8])
       time.sleep(1.5)
