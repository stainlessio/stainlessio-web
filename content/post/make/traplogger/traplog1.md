---
title: Trapeze Datalogger Project
date: 2013-07-04
aliases:
  - "/make/traplogger/2013-07-11-traplog1"
tags:
  - make
  - traplog
  - data
  - Tableau
summary: A hardware project to build a datalogger of accelerometer data and strap it to my body while I do trapeze.
---

### Project Abstract

An accelerometer equiped datalogger is strapped to a person doing aerial (trapeze, and rope) to measure the force applied to that person through different movements.

### Parts List

A small number of parts are needed, mostly in thanks to [tiny circuits][tc1] and their amazing products.  I could have gone with the lilypad version of these, but opted to not make it part of my clothing since aerial is so rough on fabrics.

* [tinyduino board][tc-td]
* [accelerometer tiny shield][tc-ats]
* [sd card tiny shield][tc-sdts]

They each look something like this:

![TinyDuino][tc-td-img] ![Accelerometer][tc-ats-img] ![Sd Card][tc-sdts-img]

These have been ordered, and I'm waiting for them to come in.  I'm planning to mount them inside a piece of PVC, and then strap that to my chest.

### Output

Once I collect the data, I'm planning to visualize the data in [Tableau][tableau] because it allows me to discover something useful and interesting from the data.

[tc1]: http://tiny-circuits.com/
[tc-td]: http://tiny-circuits.com/shop/tinyduino-processor-board/
[tc-td-img]: /static/images/tinyduino.png
[tc-ats]: http://tiny-circuits.com/shop/tinyshield-accelerometer/
[tc-ats-img]: /static/images/accel.png
[tc-sdts]: http://tiny-circuits.com/shop/tinyshield-microsd/
[tc-sdts-img]: /static/images/sd.png
[tableau]: http://tableausoftware.com/
