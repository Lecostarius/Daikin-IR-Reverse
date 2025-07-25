# Daikin-IR-Reverse

## Lecostarius motivation

I wanted to enable the AC system in a home automation environment of my own design.
There are several info websites I need to evaluate, mostly, price at the EPEX (epexspot.com, https://www.vattenfall.de/strom/tarife/oekostrom-dynamik-boersenpreise);
others, more of interest than of automatic value, are https://www.energy-charts.info, https://www.zeit.de/wirtschaft/energiemonitor-strompreis-gaspreis-erneuerbare-energien-ausbau.
And, I need weather information for a good forecast of my solar harvest. DWD gives this information, ready to download, there is a special server for it: opendata.dwd.de
with some info at https://www.dwd.de/DE/leistungen/opendata/opendata.html.


I found this very cool repository and forked it.
There are other IR libraries, like https://github.com/Arduino-IRremote/Arduino-IRremote and https://github.com/crankyoldgit/IRremoteESP8266
which are both official Arduino libraries and can be installed using the Arduino IDE. Inside the IRremoteESP8266 project there is even a
file https://github.com/crankyoldgit/IRremoteESP8266/blob/master/src/ir_Daikin.cpp which lists a bunch of references to other projects
and websites that also dealt with Daikin remotes. Also, there is a big project that unfortunately is 
all written in Java and hence unsuitable for embedded platforms under https://www.harctoolbox.org, quite interesting as really simplified
hardware is used is https://www.harctoolbox.org/arduino_nano.html. The Arduino Nano can drive 40 mA current with its GPIO pins, whereas the
RP2350 can not source more than 12 mA, and even that must be configured (standard is 4 mA).

Unfortunately, I do not have the same Daikin IR remote control, mine is a **ARC480A78**. 

![daikin1](https://github.com/user-attachments/assets/7583744a-645e-47c6-b980-6723a0c98584)

Since the remotes tend to transmit the entire state into one long sequence, there should be fields in the protocol for

- desired temperature
- fan on/off
- two bits for two swing modes (left/right and up/down)
- the pure fan mode
- cool mode
- heat mode
- the timer: at least two time values
- powerful mode
- comfort mode
- econo mode
  

## pre experiments by Lecostarius

The IRremoteESP8266 project does not compile for ESP32 modules, only for ESP8266. I have exactly one Node MCU still available; using this
with D5 as input for my IR receiver works - I can get some numbers, from a Sony remote it appears kinda repeatable, from the Daikin remote less so:
Example with Sony, executable IRrecvDemo:
```
10
type:4
810
type:4
810
type:4
A90
type:4
EA2CA9E8
type:-1
```
The code 10 refers to numeric key "1" on the remote, the 810 to key 2, the A90 to the ON/OFF key. Type 4 means Sony protocol decoded. Often, the decoding
seems to fail and the result is like in the last case, EA2CA9E8, and a type of -1 to indicate failure. When I try this demo with my Daikin remote,
and using the OFF key, the result is like this:

```
B3E2CB06
type:-1
AF1C2E0C
type:-1
B3E2CB06
type:-1
33EF3D7C
type:-1
4068375F
type:-1
```
The recommended program to read out a remote is IRrecvDumpVx, with a V2 and a V3 available. Though the V3 seems to have mostly over-the-air update
capability as difference to V2, I tried both, and since V3 works, I will stick to that. This program yields more stable and useful output. Here is
one sample output using a Sony remote using the ON/OFF button:

```
Library   : v2.8.6

Protocol  : SONY
Code      : 0xA90 (12 Bits)
Raw Timing[103]:
   +  2354, -   630,    +  1174, -   630,    +   572, -   630,    +  1174, -   630, 
   +   572, -   632,    +  1172, -   630,    +   572, -   630,    +   572, -   630, 
   +  1174, -   630,    +   572, -   630,    +   572, -   630,    +   572, -   630, 
   +   572, - 25872,    +  2374, -   632,    +  1172, -   632,    +   570, -   632, 
   +  1174, -   604,    +   572, -   630,    +  1174, -   632,    +   572, -   630, 
   +   572, -   630,    +  1174, -   630,    +   572, -   630,    +   572, -   630, 
   +   572, -   630,    +   572, - 25796,    +  2378, -   630,    +  1174, -   630, 
   +   572, -   630,    +  1174, -   630,    +   572, -   630,    +  1174, -   630, 
   +   572, -   630,    +   572, -   630,    +  1174, -   630,    +   572, -   630, 
   +   572, -   630,    +   572, -   630,    +   572, - 25798,    +  2376, -   630, 
   +  1172, -   632,    +   572, -   630,    +  1172, -   630,    +   572, -   630, 
   +  1172, -   630,    +   572, -   630,    +   572, -   630,    +  1174, -   630, 
   +   572, -   630,    +   574, -   628,    +   572, -   630,    +   574

uint16_t rawData[103] = {2354, 630,  1174, 630,  572, 630,  1174, 630,  572, 632,  1172, 630,  572, 630,  572, 630,  1174, 630,  572, 630,  572, 630,  572, 630,  572, 25872,  2374, 632,  1172, 632,  570, 632,  1174, 604,  572, 630,  1174, 632,  572, 630,  572, 630,  1174, 630,  572, 630,  572, 630,  572, 630,  572, 25796,  2378, 630,  1174, 630,  572, 630,  1174, 630,  572, 630,  1174, 630,  572, 630,  572, 630,  1174, 630,  572, 630,  572, 630,  572, 630,  572, 25798,  2376, 630,  1172, 632,  572, 630,  1172, 630,  572, 630,  1172, 630,  572, 630,  572, 630,  1174, 630,  572, 630,  574, 628,  572, 630,  574};  // SONY A90
uint32_t address = 0x1;
uint32_t command = 0x15;
uint64_t data = 0xA90;

```
The address is 0x1 for the TV and 0x97 for a VCR (the remote has some buttons that are not meant for the TV but for a VCR). I analyzed the Sony remote
with the oscilloscope and found that it starts with a 2400 us start impulse, and after that, always has 600 us OFF and either 600 us ON (for what I think
a 0 bit is), or 1400 ms ON (for a 1 bit). There are 12 bit of data for most keys, some have 15 bits.

I can find most of this in the result above. We start with a 2354 us ON impulse, then I see the 600 (actually 630) us OFF, followed by a 1 (1174 us ON) -
my oscilloscope measurement of 1400 us was apparently a bit off. The entire sequence here is 1-0-1-0-1-0-0-1-0-0-0-0. Then, we see a 26 ms break (with 
nothing transmitted), and after that, the entire sequence is repeated (since it appears I pushed the button for a long time). There is a total of 4 repeats
in the measurement above.

Now, the Daikin. 

I tried again with the oscilloscope. The Daikin uses a preamble of six pulses of 420 us each (with a OFF time in-between of also 420 us). After the preamble,
there is a long (roughly 30000 us) pause, and after that, there is a start code of 1700 us ON and 1700 us OFF. After this, the information is transmitted
in pulses where the information is coded into the length of the OFF time: each ON pulse has 400 us, and the OFF time is either 400 or 1260 us. I interpret
the 400 us OFF time as a logical 0 and the 1260 us OFF time as a logical one. The total length is really difficult to count on the oscilloscope screen. Just
by total length, my estimate is that roughly 150 bit are transmitted.

When running IRrecvDumpV3, the result is as follows. Unfortunately, the protocol is UNKNOWN. Here, I hit the yellow OFF button on the remote:

```
Library   : v2.8.6

Protocol  : UNKNOWN
Code      : 0x2444E4FE (160 Bits)
Raw Timing[319]:
   +   382, -   470,    +   430, -   436,    +   430, -   438,    +   430, -   440, 
   +   430, -   440,    +   430, - 25444,    +  3452, -  1764,    +   408, -  1330, 
   +   406, -   442,    +   428, -   440,    +   430, -   440,    +   430, -  1308, 
   +   430, -   440,    +   430, -   438,    +   430, -   440,    +   430, -   440, 
   +   430, -  1310,    +   430, -   438,    +   430, -  1312,    +   430, -  1306, 
   +   430, -   440,    +   428, -  1330,    +   408, -  1310,    +   430, -  1312, 
   +   428, -  1330,    +   408, -  1310,    +   430, -   438,    +   430, -   438, 
   +   430, -  1332,    +   406, -   440,    +   430, -   440,    +   430, -   438, 
   +   430, -   442,    +   428, -   440,    +   428, -   438,    +   430, -   438, 
   +   430, -   442,    +   430, -   436,    +   430, -   440,    +   430, -   440, 
   +   430, -   438,    +   428, -   438,    +   430, -   438,    +   430, -   440, 
   +   430, -   440,    +   430, -   436,    +   430, -   440,    +   430, -   438, 
   +   430, -   440,    +   430, -   438,    +   430, -   440,    +   430, -   440, 
   +   430, -   444,    +   430, -   432,    +   430, -   440,    +   430, -   438, 
   +   430, -  1332,    +   384, -   464,    +   430, -   438,    +   430, -  1330, 
   +   408, -  1330,    +   404, -   444,    +   430, -   438,    +   430, -   440, 
   +   430, -   438,    +   430, -   440,    +   428, -   440,    +   430, -   440, 
   +   430, -   436,    +   430, -   438,    +   430, -   440,    +   430, -   438, 
   +   430, -   440,    +   430, -   438,    +   430, -   438,    +   430, -   440, 
   +   430, -  1328,    +   382, -   464,    +   432, -  1328,    +   408, -  1332, 
   +   408, -  1330,    +   410, -  1328,    +   382, -  1358,    +   382, -   462, 
   +   430, -   442,    +   430, -   438,    +   430, -   436,    +   430, -   440, 
   +   430, -   442,    +   430, -   438,    +   430, -   440,    +   430, -   436, 
   +   430, -   440,    +   430, -   434,    +   432, -   438,    +   432, -   440, 
   +   430, -   386,    +   482, -   436,    +   432, -   440,    +   426, -   440, 
   +   406, -   462,    +   432, -   440,    +   428, -   450,    +   428, -   458, 
   +   378, -   490,    +   380, -   490,    +   402, -   462,    +   382, -   490, 
   +   380, -   488,    +   402, -   468,    +   378, -   418,    +   450, -   490, 
   +   380, -   490,    +   404, -   464,    +   378, -   490,    +   378, -   492, 
   +   378, -   490,    +   378, -   488,    +   378, -   492,    +   378, -   490, 
   +   378, -   488,    +   376, -   492,    +   380, -   490,    +   378, -   490, 
   +   378, -   492,    +   378, -   490,    +   356, -   522,    +   378, -  1352, 
   +   380, -   490,    +   378, -  1358,    +   382, -   488,    +   380, -   490, 
   +   382, -   486,    +   382, -  1336,    +   402, -  1332,    +   428, -   466, 
   +   406, -  1330,    +   408, -   462,    +   408, -   462,    +   408, -   462, 
   +   408, -   458,    +   408, -  1310,    +   430, -   462,    +   406, -   462, 
   +   380, -   490,    +   404, -   464,    +   380, -  1334,    +   428, -   468, 
   +   380, -   486,    +   380, -   490,    +   380, -   490,    +   380, -   488, 
   +   380, -  1338,    +   402, -   466,    +   424, -   446,    +   400, -   468, 
   +   400, -   468,    +   426, -   444,    +   400, -   468,    +   400

uint16_t rawData[319] = {382, 470,  430, 436,  430, 438,  430, 440,  430, 440,  430, 25444,  3452, 1764,  408, 1330,  406, 442,  428, 440,  430, 440,  430, 1308,  430, 440,  430, 438,  430, 440,  430, 440,  430, 1310,  430, 438,  430, 1312,  430, 1306,  430, 440,  428, 1330,  408, 1310,  430, 1312,  428, 1330,  408, 1310,  430, 438,  430, 438,  430, 1332,  406, 440,  430, 440,  430, 438,  430, 442,  428, 440,  428, 438,  430, 438,  430, 442,  430, 436,  430, 440,  430, 440,  430, 438,  428, 438,  430, 438,  430, 440,  430, 440,  430, 436,  430, 440,  430, 438,  430, 440,  430, 438,  430, 440,  430, 440,  430, 444,  430, 432,  430, 440,  430, 438,  430, 1332,  384, 464,  430, 438,  430, 1330,  408, 1330,  404, 444,  430, 438,  430, 440,  430, 438,  430, 440,  428, 440,  430, 440,  430, 436,  430, 438,  430, 440,  430, 438,  430, 440,  430, 438,  430, 438,  430, 440,  430, 1328,  382, 464,  432, 1328,  408, 1332,  408, 1330,  410, 1328,  382, 1358,  382, 462,  430, 442,  430, 438,  430, 436,  430, 440,  430, 442,  430, 438,  430, 440,  430, 436,  430, 440,  430, 434,  432, 438,  432, 440,  430, 386,  482, 436,  432, 440,  426, 440,  406, 462,  432, 440,  428, 450,  428, 458,  378, 490,  380, 490,  402, 462,  382, 490,  380, 488,  402, 468,  378, 418,  450, 490,  380, 490,  404, 464,  378, 490,  378, 492,  378, 490,  378, 488,  378, 492,  378, 490,  378, 488,  376, 492,  380, 490,  378, 490,  378, 492,  378, 490,  356, 522,  378, 1352,  380, 490,  378, 1358,  382, 488,  380, 490,  382, 486,  382, 1336,  402, 1332,  428, 466,  406, 1330,  408, 462,  408, 462,  408, 462,  408, 458,  408, 1310,  430, 462,  406, 462,  380, 490,  404, 464,  380, 1334,  428, 468,  380, 486,  380, 490,  380, 490,  380, 488,  380, 1338,  402, 466,  424, 446,  400, 468,  400, 468,  426, 444,  400, 468,  400};  // UNKNOWN 2444E4FE

```


When comparing this with my oscilloscope measurement, I can clearly see the six preamble pulses and the long pause. The start pulse is different though, it is not 1700 us ON and 1700 us OFF but rather 3400 us ON and 1700 us OFF. After that, I can see 
that the actual ON time is 490 us (not 420), and OFF is either 380 us or 1320 us. This is reasonably close to what I see with the oscilloscope, so I tend to believe that the analysis is OK. The length of the decoded sequence
is 160 bit. However, obviously this also counts the 6 preamble bits and what I call the start sequence (for a total of 7 bit), so that the payload might be just 153 bits - fairly close to my estimate of 150 bit, 
so even this might be both complete and accurate.

I checked, and the first 32 bit (4 bytes) of the payload are identical for the OFF key and the AUTO key on the remote. So, it seems there is some sort of a header sequence.
I see a

```
1000 1000 0101 1011 1110 0100 0000 0000 (88 5B E4 00 in MSB reading, 11 DA 27 00 in LSB reading)
```
This assumes that the start sequence (the very long ON and the very long OFF after the long pause) are not coding anything.

Apparently, my remote is using exactly the same header string as the one analyzed by Mr Blafois. But, well, my sequence appears shorter. Further analysis is required.

Here is what I get if I hit the green AUTO button at 24 set temperature:
11,DA,27,00,00,01,30,00,A0,0F,00,00,00,00,00,C5,02,08,C1,remain:1 remaining bits: 0

And here for 23 set temperature:
11,DA,27,00,00,01,2E,00,A0,0F,00,00,00,00,00,C5,02,08,BF,remain:1 remaining bits: 0

And for 23.5:
11,DA,27,00,00,01,2F,00,A0,0F,00,00,00,00,00,C5,02,08,C0,remain:1 remaining bits: 0

And for the lowest possible temperature, 18:
11,DA,27,00,00,01,24,00,A0,0F,00,00,00,00,00,C5,02,08,B5,remain:1 remaining bits: 0

Temperature - in half degree steps - is coded into the seventh byte of the transmitted sequence (the last byte(s) appear to be a checksum).

I observe the following. If I change only temperature, the remote will send another command, and this one - as opposed to the green AUTO button - is
recognized by the library as DAIKIN152 protocol and properly decoded! This command has only 152 bits, not 160. 

Sometimes, even the green AUTO button is recognized as DAIKIN152 protocol. 

I tried the demo sketch TurnOnDaikinAC. It is not meant for my remote, I had to change a few things, but now I am able to switch on my AC using
this sketch.

I tried to run it on ESP32, but that does not work, despite some comments in the source code. The issue is mainly with some hardware timer classes.

There is another Arduino library, called IRRemote, which is more modern and supports also ESP32. However, this one has less protocols covered, in particular,
it does not cover my Daikin (nor any other AC, as far as I can see).

The code is in my Arduino folder under 8266 stuff. With VSCode I can load the corresponding library, in the Arduino/libraries/IRRemoteESP8266/ folder.



## Approach


To reverse this protocol, I used an Arduino Uno, 38Khz Vishay infrared receiver (Chinese ebay copy), a simple Sketch and a Java program to plot the data and decode RAW data issued from Arduino.
Of course, I used a Daikin infrared remote control: **ARC470A1**
The Arduino sketch is very basic, and stores the duration in microseconds of states (1 or 0). It waits for a button being started and IR transmission to start, and stops after a timeout.

![Arduino Wiring](https://github.com/blafois/Daikin-IR-Reverse/raw/master/doc/Analysis-Sketch.png)

## Analysis (decoding)

Each remote-control transmission produces 3 frames. The 2 first frames are 8 bytes long while the 3rd one is 19 bytes.

The IR protocol is using Pulse Distance Encoding. This was guessed by observing the HIGH states duration is the same all the time.
Measures made with Arduino are as follow (not necessarily accurate, Arduino is not a precise logic analyzer!).

| State         | Minimum (us) | Maximum (us) | Average (us) |
| ------------- | ------------ | ------------ | ------------ |
| HIGH          | 424          | 500          | 452          |
| LOW LONG (1)  | 1264         | 1300         | 1286         |
| LOW SHORT (0) | 396          | 436          | 419          |

Bytes are coded using Least Significant Bit (LSB).

## Protocol documentation

Based on multiple tests, here is what I was able to determine on the protocol used by Daikin.

### Frame 1 (Code 0xc5)

The first frame is always the same, except when "comfort" mode is enabled.
```
11 da 27 00 c5 00 00 d7 
```

```
Offset  Description            Length     Example        Decoding
========================================================================================================
0-3     Header                 4          11 da 27 00
6       Comfort mode           1          10
7       Checksum               1          d7	
```

Comfort mode:
```
11 da 27 00 c5 00 10 e7       Comfort mode ENABLED
11 da 27 00 c5 00 00 d7       Comfort mode DISABLED
```

### Frame 2 (Code 0x42)

```
11 da 27 00 42 00 00 54 
```

### Frame 3 (Code 0x00)

Sample frame transmitted:
```
11 da 27 00 00 49 3c 00 50 00 00 06 60 00 00 c1 80 00 8e
```

```
00:     11 da 27 00 00 49 3c 00 
08:     50 00 00 06 60 00 00 c1 
10:     80 00 8e 
```

Frame specification:
```
Offset  Description            Length     Example        Decoding
========================================================================================================
00-03   Header                 4          11 da 27 00
04      Message Identifier     1          00	
05      Mode, On/Off, Timer    1          49             49 = Heat, On, No Timer
06      Temperature            1          30             It is temperature x2. 0x30 = 48 / 2 = 24°C
08      Fan / Swing            1          30             30 = Fan 1/5 No Swing. 3F = Fan 1/5 + Swing. 
0a-0c   Timer delay            3          3c 00 60           
0d      Powerful               1          01             Powerful enabled
10      Econo                  1          84             4 last bits
12      Checksum               1          8e             Add all previous bytes and do a OR with mask 0xff
```

#### Mode
The various mode supported by my remote control are:
* Heater
* Cooler
* Fan
* Automatic
* Dry

Only the first 4 bits of this byte is changing:

```
0-   AUTO
2-   DRY
3-   COLD
4-   HEAT
6-   FAN
```

**Nota bene:** in *DRY* and *FAN* mode, the temperature transmitted is 25°C (it is not relevant).

#### Timers and Power

To manage the On/Off state as well as timer modes, the last 4 bits are used.

```
Bit     Value    Description
=======================================
0       8        Always "1"        
1       4        Timer OFF enabled
2       2        Timer ON enabled
3       1        Current State. 1 = On, 0 = Off
```

The timer mode can be "delayed power on" or "delayed power off". The timer delay is explained later.

Few examples:
```
-8   Off, No timer
-9   On, No timer
-a   Off, Timer "On"
-c   Off, Timer "Off"
-d   On, Timer "Off"
```

#### Temperature
My remote control supports temperature between 10 and 30 degrees. Coding of temperature is quite easy to reverse: take the temperature in Celsius, multiply by 2, and code it in heax.
For example:

```
Desired temperature in °C:     20
Multiply it by 2:              40
Convert it in hex:           0x28
```

#### Fan Speed, Swing
The remote control supports 3 modes:
* Manual from 1 to 5
* Silent
* Auto

The remote also has a "Swing" mode.

The top 4 bits are used to code the fan-mode, and the 4 last bits are used to code the swing function.

Modes:
```
3    Fan 1/5
4    Fan 2/5
5    Fan 3/5
6    Fan 4/5
7    Fan 5/5
A    Automatic	
B    Silent	
```

Swing:
```
0    Swing disabled
F    Swing enabled
```

Few examples
```
3F   Fan 1/5 and Swing enabled
70   Fan 5/5 and Swing disabled
```

#### Timer

There are 2 times: timer for Power On and timer for Power Off.
The type of timer is coded at offset 5

The timer delay position and coding defers depending of timer type.

For Timer *ON*, the delay is on offset 0a and 0b, with the following coding:

Few examples:
```
4H = 4 * 60 = 240 minutes = 0x00f0. This will be coded as f0 00
5H = 5 * 60 = 300 minutes = 0x012c. This will be coded as 2c 01
```

In Java, this can be decoded using the following code snippet:
```Java
int timerDuration = (0x100 * message[0xb] + message[0xa]) / 60;
```

For Timer *OFF*, the delay is on offset 0b and 0c, with the following coding:

Few examples:
```
1H = 1 * 60 = 60 minutes = 0x003C. This will be coded as c6 03
5H = 5 * 60 = 300 minutes = 0x012c. This will be coded as 2c 01
```

In Java, this can be decoded using the following code snippet:
```Java
int timerDuration = ((message[0xc] << 4) | (message[0xb] >> 4)) / 60;
```


#### Powerful

The Powerful mode sets the AC at his maximum capacity for 20 minutes, and then reverts back to the previous state.
The previous state is transmitted in the frame. So basically, the Powerful state is just 1 bit in addition to a normal frame.

For example, the following frame will set the powerful mode during 20 minutes in heat state, and then will heat normally at 20°C with fan speed 3:
```
11 da 27 00 00 49 28 00 50 00 00 06 60 01 00 c1 80 00 7b 
```

#### Econo

The 4 last bits are used to determine if econo mode is enabled or not

```
80   Econo is DISABLED
84   Econo is ENABLED
```

#### Checksum
The last byte of frame is a checksum. All previous bytes are added and only the 2 bytes are kept. If you compute it with modern languages, simply apply 0xFF mask.

Example with this frame:
```
11 da 27 00 00 49 3c 00 50 00 00 06 60 00 00 c1 80 00
```

```
11 + +da + 27 + 00 + 00 + 49 + 3c + 00 + 50 + 00 + 00 + 06 + 60 + 00 + 00 + c1 + 80 + 00 
   = 38e

  03 8e
& 00 ff 
   = 8e 
```

### Build a message from scratch

```
|   header    | Msg Id | Mode | Temp | Fixed | Fan | Fixed |  Timers  | Pwrful | Fixed |                         | Fixed | Checksum |
| 11 da 27 00 |   00   |  xx  |  xx  |   00  |  xx |   00  | xx xx xx |   0x   |   00  | 8x |   00  |    xx    |
```

