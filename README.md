# ESPmeter

ESPmeter helps monitoring domestic gas consumption; a small device captures 
data from the gas meter and a web application allows visualizing daily gas 
consumption with a granularity of 5 minutes.

A small probe equipped with a Hall effect sensor is placed right on top of the
last drum of the gas counter; the drum rotates when the gas flows and so does
the magnet which generates a pulse in the sensor, signaling a revolution; 
keeping track of revolutions over time allows to calculate the amount of gas 
consumed during a certain period. This device obviously works only with meters 
equipped with a small permanent magnet embedded in one (usually the last) drum 
of the counter. While this project has been designed with gas meters in mind, 
it could theoretically work with other meters, such as water meters.

![Battery compartment on top and electronics on the bottom](docs/pics/boxed.jpg)

## Design

The device has been designed to be extremely easy to build, with a small number
of very common components. Particular attention has been put into energy
efficiency, trying to reduce consumption to the minimum, allowing batteries
to operate for theoretically more than one year. 

The ESP8266 power absorption is too high to be used to directly to sample pulses
generated by the Hall effect sensor. Even using a Reed switch to wake-up the 
ESP at every pulse, consumption would be in the order of 15mA for the whole
ESP boot time + accounting software routine to run, assuming Wi-Fi being 
turned off from the start. This is too much to achieve battery life in the order
of years.

Instead, the ESP is coupled with an Tiny13, which takes care of the pulse 
recording part. The Tiny13 has a very low power consumption and can stay
asleep for most of the time. The ESP would then communicate with the Tiny13 
at regular intervals and then once per day would transmit data via Wi-Fi.

![The guts](docs/pics/top.jpg)

## Sampling frequency

Because we are using an Hall effect sensor and not a Reed switch, to be sure
the Tiny13 doesn't miss a pulse, the sensor has to be read often enough.

In my particular configuration, the drum completes a revolution not more often
than 15 seconds (when I'm using both heater and kitchen stove at maximum power).

During this interval, the magnet is detectable by the sensor for about 1.5
seconds while passing under the sensor. 

Therefore setting the Tiny13 watchdog at 1Hz would ensure that the device is
woken up often enough so that it has time to power up the Hall sensor and detect
the presence of the magnet.

However, the larger the maximum gas flow that can be consumed, the faster the 
drum will spin, reducing the time the magnet is detectable by the sensor;
therefore depending on your meter model and your house maximum gas consumption
you may want to adjust this parameter in the code.

## Hardware

![Schematics](docs/pics/schematics.png)

### Parts

* ATTiny13-20PU
* ESP-12F module
* MCP1702-3302ET low quiescent LDO regulator
* AH276 Hall effect sensor
* 10kΩ resistors (5x)
* 3.9MΩ resistors (2x) (see below)
* 10μF 10V capacitor
* perfboard
* 3 pin header (optional)
* 3 AA battery holder
* plastic enclosure
* 3 lead wire

### The probe

In my build, the Hall effect sensor has been salvaged from a 12V PC fan. 
Perhaps similar sensors can be used; I've found this one to be quite reliable
in sensing the small magnet inside the meter. It is also drawing a very small 
amount of power and working well at 3.3V.

![Finished probe](docs/pics/hall.jpg)

I've mounted the sensor on a small strip of perfboard, then soldered a decently 
looking wire so that the whole probe can be easily placed in the slot on top of 
the meter drums. I have embedded the probe into hot glue, then properly trimmed 
it to give it some sort of semi-professional look.

Below you can see the finished probe installed on my meter, temporarily hold in 
place with some paper. Note the small shiny magnet glued on top of the number 6 
in the last drum.

![Probe in the meter](docs/pics/meter.jpg)

## Software on Tiny13 side:

Code is written in C and compiled via [avr-gcc](https://gcc.gnu.org/wiki/avr-gcc).

Most of time, the ATTiny is in sleep state. Internal timer is used to wake up
regularly, power the Hall effect sensor through one of the ATTiny pins and read
its output. The Tiny stores 3 hours of pulse counting in its own memory and 
it also regularly samples battery voltage every 3 hours. 

Communication with ESP is achieved via a simplified implementation of the I2C
protocol, derived from "AVR311: Using the TWI Module as I2C Slave" Atmel 
application note available [here](http://ww1.microchip.com/downloads/en/AppNotes/atmel-2565-using-the-twi-module-as-i2c-slave_applicationnote_avr311.pdf)

Battery voltage is fed through a voltage divider and compared with the 3.3V 
provided by the voltage regulator via the ADC. This is not very accurate but
it should be good enough to roughly estimate battery discharge rate. 

What it is sent to the ESP is contained in the following struct:

	typedef struct pulse_log_t {
		uint8_t checksum;
		uint8_t vcc;
		uint16_t ticks;
		uint8_t frames[LOG_FRAMES];
	} pulse_log_t;

* checksum is a simple modulo 256 of sum of the other bytes of the struct
* vcc is updated every 3 hours and it's the output of the ADC used to measure
battery voltage (see below)
* ticks is the number of seconds since last communication with the ESP
* frames is an array of 36 bytes, every byte is the number of pulses recorded in
the corresponding 5 minutes interval. 36 * 5m = 180m = 3h

## Software on ESP8266 side:

[NodeMCU](http://www.nodemcu.com/) firmware powers the ESP8266 side. 
The ESPmeter code is written in [Lua](http://www.lua.org/).

ESP8266 could theoretically stay alseep for 3 consecutive hours, however due to 
the ESP wake-up counter implementation, it's impossible to sleep longer than 
about 71 minutes (4294967295us). The ESP therefore sleeps 1 hour, and when 
it wakes up, depending on the hour, it either:

* 3, 6, 9, 12, 15, 18, 21: retrieves data from Tiny13 and stores it in its RTC
* 0: retrieves data from Tiny13, sends the whole RTC memory dump to server 
and performs clock syncronization using the response from server
* rest: goes back to sleep immediately for another hour

After sending content to ESP, the Tiny13 clears its own memory and a new 3
hours log is initialized.

ESP8266 RTC memory is configured as a 32-bit integer array. The 40 bytes struct
dumped from the Tiny13 is converted into a 10 elements array of 32-bits 
wide integers and stored into the RTC memory. RTC memory survives deep sleep 
cycles while the normal RAM does not so this makes it ideal for the job.

Here the RTC memory map (unit: 32bit array)

    [00 - 09]:  Used by rtctime library to store RTC calibration data
    [10 - 10]:  Clock calibration counter (+ checksum)
    [11 - 91]:  8 slots, 10 elements each, every element is 4 bytes
                every slot corresponds to a Tiny13 memory dump (40 bytes). 
                Checksum is performed on the Tiny13 and checked on the ESP 
                before sending the slot content to the server


## Compensate for ADC (in-)accuracy

The Tiny13 ADC is used to monitor batteries voltage so the user can be alerted
when they need to be replaced. The Tiny13 ADC is a 10-bits ADC configured to
compare the Tiny13 VCC with pin PB2, which is fed the unregulated battery 
voltage through a voltage divider made with two resistors of nominally equal 
value. In the schematics, these resistors are 3.9MΩ but they can be replaced
with other values, as long as they are the same. A high value will limit the 
current drain and this can be usful to prolong battery life. 

The ADC is a 10-bits but only 8 are used here and because the two resistors 
forming the voltage divider are unlikely to be identical, it's advisable to
run a series of simple measurements to tune the Tiny13 reading.

With a variable voltage supply, we measured different input voltages from 6.65V
down to 2.82V, filling the table below; we are interested in the 
V<sub>BATT</sub> / V<sub>ADC</sub>
column which relates the unregulated battery voltage with the voltage sampled
by the ADC. Three fresh AA battery will provide a maximum of 4.5V and it is 
unlikely that they will be able to provide enough current to power the circuit 
once their output will be below 3V; in practice the useful interval is 
narrower and you can use a set of fresh batteries and a set of almost depleted
ones to take two measurements and average them, it should give you a good enough
accuracy. 

|V<sub>REF</sub>|V<sub>BATT</sub>|V<sub>PB2</sub>|ADLH|V<sub>ADC</sub>|V<sub>BATT</sub> / V<sub>ADC</sub>|V<sub>BATT</sub> / V<sub>PB2</sub>|
|----|-----|-----|----|-----|----------|----------|
|3.3 |6.65 |2.75 |255 |3.300|2.015     |2.418     |
|3.3 |5.47 |2.26 |209 |2.705|2.022     |2.420     |
|3.3 |5.03 |2.08 |192 |2.485|2.024     |2.418     |
|3.3 |4.41 |1.831|169 |2.187|2.016     |2.409     |
|3.3 |3.99 |1.655|153 |1.980|2.015     |2.411     |
|3.3 |3.41 |1.417|130 |1.682|2.027     |2.406     |
|2.87|2.92 |1.213|128 |1.441|2.027     |2.407     |
|2.76|2.82 |1.169|127 |1.375|2.052     |2.412     |

Fields:
* V<sub>REF</sub>: regulated voltage read by multimeter
* V<sub>BATT</sub>: battery voltage read by multimeter
* V<sub>PB2</sub>: divider voltage read by multimeter
* ADLH: value of the ADC register
* V<sub>ADC</sub>: calculated voltage from ADC (=Vref / 255 * ADLH)
* V<sub>BATT</sub> / V<sub>ADC</sub>: ratio between battery voltage read by multimeter and by ADC
* V<sub>BATT</sub> / V<sub>PB2</sub>: ratio between battery voltage and voltage divider (read by multimeter)

As expected, the voltage divider formed by the two equal resistors is not 
producing half of the input voltage probably due to tolerance in resistor
value. On the other hand, the ADC luckily enough has also some offset which 
compensates almost perfectly, producing a readout of almost exactly 1/2 of the 
battery voltage. 

