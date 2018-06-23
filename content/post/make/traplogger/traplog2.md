---
title: Datalogger build day one
aliases:
  - "/make/traplogger/2013-07-11-traplog2"
tags:
  - make
  - traplog
  - accelerometer
  - arduino
date: 2013-07-11
summary: Building a datalogger is stupidly easy, once you figure out how to get the SD card working
---

I received my [Tiny Circuits](http://tiny-circuits.com) package on Tuesday, and started playing around with it that night and the next day. They are amazingly easy to work with, except one thing: my big fingers have a hard time screwing the super-tiny nuts onto the bolts to hold them together, but that's just an annoyance more than anything.

Stacking the tinyduino, usb adapter, accelerometer, and sd card shields together, in less than 10 minutes, I had what would eventually be my datalogger. One small change is that I'm going to take one of the protoboards that came with the starter set, and solder a battery connector, and a small switch of some sort and an led onto it so I can have some type of control while not attached to the usb, and I'll probably take the usb off of the stack to reduce the size.

The code for the arduino is really simple:

    #!java
    // Trapeze Data Logger v0.1
    // Russell Hay
    // BSD Licensed
    //
    // Some Code Taken from http://tiny-circuits.com/learn/using-the-accelerometer-tinyshield/

    #include <SD.h>
    #include <Wire.h>
    #define CHIP_SELECT 10
    #define I2CADDR 0x18
    #define RANGE 0x03 // 2g
    #define BANDWIDTH 0x08 // 7.81Hz

    boolean error = false;
    int x,y,z;
    float t;

    void setup() {
      pinMode(10, OUTPUT);
      if (!SD.begin(CHIP_SELECT)) {
        error = true;
      }
      accel_init();

      if (error) {
        pinMode(13, OUTPUT);
      }
    }

    void accel_init() {
      Wire.beginTransmission(I2CADDR);
      Wire.write(0x0F);
      Wire.write(RANGE);
      Wire.endTransmission();

      Wire.beginTransmission(I2CADDR);
      Wire.write(0x10);
      Wire.write(BANDWIDTH);
      Wire.endTransmission();
    }

    void accel_read() {
      uint8_t buffer[8];

      Wire.beginTransmission(I2CADDR);
      Wire.write(0x02);
      Wire.endTransmission();
      Wire.requestFrom(I2CADDR, 7);  // Read 7 bytes

      for(int i=0; i<7; i++)
      {
        buffer[i] = Wire.read();
      }

      x = (buffer[1] << 8 | buffer[0]) >> 6;
      y = (buffer[3] << 8 | buffer[2]) >> 6;
      z = (buffer[5] << 8 | buffer[4]) >> 6;
      t = (buffer[6] * 0.5) + 24.0;
    }

    void loop() {
      if (error) {
        error_loop();
      } else {
        log_loop();
      }
      delay(70);
    }

    void error_loop() {
      digitalWrite(13, HIGH);
      delay(50);
      digitalWrite(13, LOW);
    }

    void log_loop() {
      accel_read();
      File file = SD.open("trapeze.csv", FILE_WRITE);
      if (file) {
        file.print(x); file.print('|');
        file.print(y); file.print('|');
        file.print(z); file.print('|');
        file.println(t);
        file.close();
      } else {
        error = true;
      }
    }

But the only problem is that my sd card shield can't seem to interact with the card that I have. I tried the cardinfo sd example as outlined by the tiny circuits documentation, and that worked fine, but as soon as I tried any other functions, `SD.begin(10)` fails, and I'm not sure why. I'm working on debugging that to figure out what's going on.

Until I figure that out, I've programmed a small sketch that just streams the data via serial to the usb port, so I can start working on sound making tonight.
