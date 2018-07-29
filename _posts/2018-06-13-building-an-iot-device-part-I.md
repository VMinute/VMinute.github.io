---
layout: post
title: Building an IoT Device (part I)
date: 2018-06-13 12:00:00
tags: arduino esp32
author: vminute
---
<amp-img src="{{ site.baseurl }}assets/images/blog/2018-06-13-picture.jpg" width="644" height="431" layout="responsive" alt="" class="mb3"></amp-img>

Since I am working on embedded software and devices around two hundred days per year, I decided that my hobby project should also involve devices and code.

I decided to build a bedside clock for my kids. 

I didn’t want to build an alarm clock (I think that you should not be tortured by such a device during your childhood), but a device that could tell them if it was sleeping time (sometimes they wake up at 4:00 AM and ask with their loudest possible voice if it’s already time to wake-up) and what is the plan for the day (going to school/kindergarten, play in a rugby/volleyball match, celebrating a friend birthday etc.). 

I wanted to use a graphical display to show the current time and additional information but just wanted to turn it on when required to not disturb their sleep.

I decided to build a connected device taking information from a calendar where I and my wife could set expected wake-up times and “events” for the coming day. Since my device is effectively collecting information from the network, I think it can be legitimately described as an “IoT” device. 

Those were my basic “specifications”, in addition to that, I wanted something not too expensive (the cheapest the better) and also wanted to play a bit with ESP-32 that looks like a very interesting platform for this kind of devices. It provides wi-fi and Bluetooth connectivity and is in the same price range of most “traditional” microcontrollers. It also provides enough RAM and flash to implement a basic graphic UI and secure connectivity.

Arduino support on ESP-32 seems to be solid enough to base my hobby project on it and was probably the simplest tool to set up. I was not willing to spend too much time in setting up a toolchain etc. so I behaved like a real maker and using <a href="https://github.com/espressif/arduino-esp32">ESP-32 Arduino core</a> I was able to build and deploy a test sketch in less than 10 minutes.

I also decided to document the design and build process to let other people build the same device or, even better, improve it. 
Detailed build instructions are on <a href="https://www.instructables.com/id/Connected-Bedside-Clock-for-Kids/">Instructables</a>.

In this series of articles, I’ll document with more technical details the different aspects of the design. 

## Components and hardware design

After having described my rough specifications for the project (anyway better than those I got from some customers in the past…) I started designing my device.

I needed to connect a display and probably using a small graphical display on I2C is the simplest solution from an HW point of view, just need power and two signals (clock and data). Many of those displays are based on SSD1306 and usually support monochrome displays with low resolution (in my case 128x64 pixels). The OLED ones are bright enough to not require a backlight and can be powered using 5V power supply.
Those displays are also quite cheap (2-5$) and well supported by <a href="https://learn.adafruit.com/adafruit-all-about-arduino-libraries-install-use/arduino-libraries">Adafruit’s libraries for Arduino</a>.

<amp-img src="{{ site.baseurl }}assets/images/blog/2018-06-13-display.jpg" width="644" height="484" layout="responsive" alt="" class="mb3"></amp-img>

I needed an easy way to turn on the display and using a Passive InfraRed sensor (PIR) sounded like the best solutions. Those sensors are super easy to interface (you need just a digital input pin) and can be configured by reducing their sensitivity (usually with a potentiometer) or their field of view (with some duct tape) to trigger them with the right amount of movement. I wanted it to trigger when someone was waving a hand close to it and with some patience and duct tape, it wasn’t hard to reach this goal. PIR sensors are also quite cheap (2-3$) fitting the other mandatory requirement of my project. Most PIR sensors require a 5V power supply and can generate 3.3V signals, making them compatible with the ESP-32.

<amp-img src="{{ site.baseurl }}assets/images/blog/2018-06-13-pir.jpg" width="644" height="484" layout="responsive" alt="" class="mb3"></amp-img>

There are lots of ESP-32 based modules around. I was planning to use Adafruit’s Huzzah, but I couldn’t find a solution for a quick and cheap (again) delivery to Italy, so I just searched for a clone on Amazon. Most of the features are directly provided by the ESP-32 chip itself, so you should not expect many differences between them. Some ESP-32 modules (like WROOM-32) are very small but don’t have an easy to access pinout. Since I didn’t have too many constraints about the size I went for a module with standard 2.54’’ (0.1 inches) connectors. I used a <a href="https://github.com/playelek/pinout-doit-32devkitv1">module from doit</a> that costs less than 10€.

<amp-img src="{{ site.baseurl }}assets/images/blog/2018-06-13-module.jpg" width="644" height="484" layout="responsive" alt="" class="mb3"></amp-img>

It also requires a 5V power supply, making a simple 5V capable power brick the only additional component needed for this design.

## Schematic

I wanted to lay out a nice schematic for my project so I downloaded <a href="https://www.altium.com/circuitmaker/overview">Altium’s Circuit Maker</a>. It’s a free tool for schematic and layout design that comes with a very rich library of components. Of course, it was missing the DOIT ESP32 module, but I was able to design it myself and upload it for other people willing to use it.

The tool is not always user-friendly (at least not friendly with me…) but I managed to design a nice looking schematic and <a href="https://github.com/VMinute/KidsClock/blob/master/Documentation/Connected%20Clock%20for%20Kids.PDF">export it as PDF</a>. 
You can also share it with other people on Circuit Maker’s website, as <a href="https://workspace.circuitmaker.com/Projects/Details/Valter-Minute/Connected-Alarm-Clock-for-Kids">I did</a>.

<amp-img src="{{ site.baseurl }}assets/images/blog/2018-06-13-schematic.png" width="998" height="772" layout="responsive" alt="" class="mb3"></amp-img>

As you can see from the schematic is pretty simple. All components are powered using 5V. The output pin of the PIR sensor is connected to GPIO12. The signals will be high (3.3V) as long as movement is detected. Polling the pin will allow us to turn on the display when something is moving near the sensor.

The display is connected to I2C using GPIO21 and GPIO22, we don’t need pull-up (usually required for I2C communication) because those are already on the module. In the schematic, you see also a reset signal and a 3.3V power signal for the display. Those are part of the standard SSD1306 interface, but the display I used was not requiring them, so they are not connected in this schematic.

Actually, the number of connection was so limited that I just used jumper cables to connect pins from the different components without any PCB or breadboard.

## Case

<amp-img src="{{ site.baseurl }}assets/images/blog/2018-06-13-3dcase.png" width="644" height="430" layout="responsive" alt="" class="mb3"></amp-img>

I designed a case for the device and 3D printed it. I used <a href="https://www.tinkercad.com/">tinkercad</a> to draw it (I really like the user interface and the fact that you can access your designs from a simple web browser) and also <a href="https://www.tinkercad.com/things/jcxc8QNp8hI-connected-alarm-clock-for-kids#/">this design is available</a>. 
My kids share a bunk bed, so I needed something that could be attached to the bed frame, feel free to change the design to fit your own needs, of course.

Since I needed an easy way to remove the device from its location (to fix/debug it) and power it on and off (to quickly fix software issues by rebooting…) I decided to use magnets to connect the main case to a mounting plate that was going to be attached to the bed using wood screws.
Components inside the case simply lock in place with the help of a couple of plastic pieces.
