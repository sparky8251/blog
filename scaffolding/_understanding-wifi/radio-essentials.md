+++
title = "WiFi and Radio Essentials"
description = "The radio science fundamentals you need to understand wireless networking"
weight = 1
+++

Before we can begin, I want to explain what this guide aims to be and what it does not aim to be.

This guide aims to aid the technical home user in understanding WiFi and aid in the troubleshooting and setup of WiFi networks. You will have to learn radio science to some degree as it's core to how all this works, but we will not do deep dives.

This will NOT be a radio science/engineering degree, a wireless network engineering test aid, and it will aim to simplify in areas where it's not technically correct to aid understanding for "the common technical home user".

Suggestions for improvements in this vein are welcome.

## The Essentials of Radio Science

### Decibels: What They Are, How to Relate to Them, And Why You Must Care

**Decibels** are how we measure the energy in waves, be they pressure waves (sound) or EM waves (radio signals). It is a logarithmic scale, meaning every 3dB change doubles/halves the power in the waves. Since it's for waves, we can compare hearing with your ears to how your WiFi devices hear. We won't do that just yet, but here's a simple graph showing you some common dB ratings for hearing to give you some grasp of the concept.

```
130 dB ░░░ Threshold of Pain
120 dB ░░░ Jet Engine/Thunder Clap
110 dB ░░░ Car Horn/Chainsaw
100 dB ░░░ Loud Motorcycle
90  dB ░░░ Hair Dryer/Lawnmower
80  dB ░░░ Alarm Clock/Garbage Disposal
70  dB ░░░ Vacuum Cleaner
60  dB ░░░ Normal Conversation
50  dB ░░░ Quiet Conversation
40  dB ░░░ Refrigerator Hum/Quiet Office
30  dB ░░░ Quiet Library
20  dB ░░░ Rustling Leaves/Whispering
10  dB ░░░ Breathing
0   dB ░░░ Threshold of Human Hearing
```

Now for the fun bit, there are dB, dBm, and dBi units out there. They are all related but mean slightly different things.

**dB (Decibel)** is a relative unit used to express a ratio between two power levels and it's a relative measurement. It's not an absolute measurement like F/C is for temperature.

**dBm (Decibels relative to a milliwatt)** is an absolute unit of power, it references a fixed unit of power in 1mW with 0 dBm being 1mW of power.

**dBi (Decibels relative to an isotropic radiator)** is a unit for measuring an antenna's gain. Gain is how well an antenna can focus power in a specific direction in relation to the reference point, an isotropic radiator that is theoretical and radiates power equally in all directions.

Putting this all together, it's not uncommon to see things like -70dBm signal strength reports for WiFi as they are weak signals. Additionally, you can use simple rules to estimate how much power is added/removed given a specific dB change. 3dB is doubling/halving the power, 10dB is 10 times or 1/10th the power. So putting it all together, you have a 1,000 times increase in power from normal conversation (60 dB) to a hair dryer (90 dB) which is a 10-times power increase, three times (10x10x10=1000). This same principle applies even from -70 dBm to -67 dBm, that's a doubling of power and can make real differences in your signal strength and thus WiFi speeds.

### Frequency, Wavelength, and Free Space Path Loss

<!-- TODO: the math to show how frequencies and distance impact it -->

### Attenuation and Gain

<!-- TODO: table of common materials and things that trigger it. touch on antennas and how gain works and reference dBi once more. Include a blurb on polarization and how it's VITAL to have everything similarly polarized and how WiFi is linearly polarized due to how easy it is to change the angles of portable devices -->

### RSSI, Noise Floor, and SNR

<!-- TODO: table of common frequency noise floors in a few conditions + an SNR graph showing common wifi "health" states -->

### Simplex, Full-Duplex, and Half-Duplex

### Regulations

<!-- just a blurb about their importance and how it differs by nation/region and can open/close doors for different people as a result -->

## WiFi Essentials

### Channels and Bandwidth

<!-- how they relate to frequencies and things like bleed -->

### Hidden Nodes and CSMA/CA

<!-- mentions of things like RTS/CTS, and transmission delays. even how transmission slots widen/shrink as legacy wifi nodes are heard slowing down faster ones -->

### DTIM Period and Beacon Interval

## Common Problems and Troubleshooting

### Range Issues

<!-- estimating resultant SNR with antennas, FSPL, attenuation estimates, and so on to help you realize when you are just unable to reach further without changes and suggest changes to make like moving plants/pots, hanging the wifi AP on the ceiling, etc -->

### Channel Selection

<!-- channel overlap, finding uncrowded channels with wifi scanners/analyzers, adhering to rules and regs and how they can impact selection -->

### Unexpected Drops and Excessive Battery Drain

<!-- DTIM and beacon interval stuff here and the balancing act it creates -->
