This project contains the code and documentation for the wheel testing
project. This project was started by the team participating in
[Build18 2020](http://www.build18.org/garage/project/583/), and was
continued by that group and other members of Apex.

## To-dos

* This repository still needs a lot of documentation, particularly for
  the electrical and mechanical assemblies
* The hardware for the project is still a prototype and needs a final revision
* The code
* The parts list below is very incomplete

## Parts

* [T-Slot Photo Interrupter](https://www.adafruit.com/product/3985)

## Code Dependencies

* Arduino megaAVR Boards (in the Arduino Board Manager), for the Arduino Uno WiFi Rev2

## Research

I didn't have time to write or test anything, but I did a bit of
research, which is documented here. Please remove this note and modify
the text below to match the final project and any further research, as
relevant.

Photo interrupter: The I/O Circuit Diagram tab of the Photo
Interrupter specs page (linked below) has a wiring diagram. My
circuits skills are a bit rusty, but I believe the "Out" pin is where
we should connect the Arduino's input pin (as we had previously), but
also the "Load" in the diagram is a pull-up resistor on the input pin,
which I believe we did not have. The sensor should be connected to a
digital input pin, not an analog pin. In the "Light-ON" mode of
operation (selected by connecting the "L" terminal to +5V), when the
IR beam is interrupted, the built-in LED and the output transistor
will turn off. When the transistor is off, the load is disconnected
from ground, so the pull-up resistor pulls the input pin to +5V. In
the original design, we didn't have this pull-up resistor, so the
input pin would have been electrically disconnected and just reading
noise, which explains the erratic measurements we saw. When the beam
is not interrupted, the transistor is enabled, pulling the OUT pin to
ground, so the input pin will read ground. A pull-up resistor can be
enabled with `pinMode(pin, INPUT_PULLUP)`. Arduino's page on digital
pins, linked below, is a good read.

Interrupts: Before proceeding, read Arduino's page about interrupts,
linked below (taking 18-349 would also be helpful). Especially because
this project will use WiFi, using interrupts to count encoder ticks is
the right way to go. The one caveat of using interrupts with WiFi is
that certain WiFi operations _might_ disable interrupts, which would
lead to missing encoder ticks. I haven't done the research or testing
to know if this is an issue, though. As much as possible, I would
avoid performing explicit WiFi operations during a run, though some
may still run in the background. One possible approach to reading the
encoder is to create a timer interrupt and read the pin in that
interrupt. I'd advise against this because performing a `digitalRead`
may take a long time, and you want interrupts to be _fast_ for two
reasons: so that they don't block whatever code was "interrupted" for
a long time; and because if an interrupt runs for longer than the time
between interrupts, you'll always be running interrupt code _and_
you'll miss some interrupts. Instead, I'd create an interrupt on the
input pin connected to the encoder in mode `CHANGE`. (If it becomes
necessary to do a `digitalRead` in the ISR to check whether the
encoder pin is high or low, I would instead make two interrupts: one
in mode RISING and one in mode FALLING.) The ISR can be very simple:
increment a counter or two and record the current time in a table. For
recording the current time, use `millis()` or `micros()` (you
shouldn't expect these functions to work correctly if your interrupts
take a long time, but for a very short ISR they'll be totally fine).
The WiFi Rev2 board may also have a real-time clock (RTC), which you
should **not** use in an ISR, because the RTC functions are
long-running. The RTC is probably more accurate than the normal
Arduino time functions, so it could be good to use it in some form,
perhaps by using it in `loop()` to calibrate `millis()`. That should
probably be a stretch goal, though. Another stretch goal could be to
detect forward and backward rotation, which requires two encoders,
offset by about half the width of a slot in the encoder wheel. The
waveforms would then look like
[this](https://www.leniwiec.org/wp-content/uploads/2014/04/rysunek_1.png).

Time: `millis()` and `micros()` both return `unsigned long`s. On most
Arduinos, `int`s are 16-bit and `long`s are 32-bit. This means that
after an hour of operation, `micros()` will overflow a `long`. That's
not really a problem if you always subtract your reading from some
start time, because one run will never be over an hour. It also means
you can't easily store the result of `micros()` in an `unsigned int`,
as it overflows after 65 ms (though this could also work, because even
if you send the timestamps to a computer as a 16-bit type, you could
extend them into a 32-bit type by determining each time that overflow
occured and adjusting the subsequent readings, or you coult just
record time deltas instead of timestamps). Using `unsigned int`s for
`millis()` would probably also be fine, with 65 seconds until
overflow. `millis()` won't overflow an `unsigned long` for almost 50
days. Whether you need millisecond or microsecond resolution depends
on the speed of the wheels, how many slots are in the encoder wheel,
and how much noise you're willing to tolerate in the readings. If when
a wheel is going its fastest, there are still 2-3 milliseconds between
encoder interrupt fires, I would say millisecond resolution is
sufficient, but that's ultimately a design decision. The danger of
using `long`s to store timestamps is that you might run out of memory.
I would definitely not use dynamic memory allocation (`malloc` or
`new`); instead, statically allocate the array of timestamps (as a
global), and the Arduino IDE should tell you if you're using too much
memory. You might run out of space even using `int`s, in which case
you either need to lower the resolution further (perhaps you can get
away with only storing 8-bit time deltas, storing fewer timestamps, or
compressing data somehow) or get a better board. Arduinos are kind of
notorious for not having much memory, so this might just be necessary.

Bluetooth: I didn't look into it, but the WiFi Rev2 board does have
Bluetooth. This might be harder to set up, but could work better
because on Bluetooth a device can push data to a host, which is less
common on WiFi.

## Additional Resources

* [Photo Interrupter specs](http://www.ia.omron.com/product/item/2229/)
* [Arduino digital pins (`INPUT_PULLUP`)](https://www.arduino.cc/en/Tutorial/DigitalPins)
* [Arduino interrupts](https://www.arduino.cc/reference/en/language/functions/external-interrupts/attachinterrupt/)
* [More on interrupts](http://gammon.com.au/interrupts)
* Arduino [`int`](https://www.arduino.cc/reference/en/language/variables/data-types/int)
  and [`long`](https://www.arduino.cc/reference/en/language/variables/data-types/long)
