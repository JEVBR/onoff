## onoff

GPIO based I/O and interrupt detection with Node.js on Linux boards such as the
BeagleBone or Raspberry Pi.

onoff provides a constructor function called Gpio which can be used to make
Gpio objects corresponding to Linux GPIOs. The Gpio constructor function has
three arguments; a GPIO #, a direction, and an optional interrupt generating
edge. Examples of its usage can be seen in the code below. The Gpio methods
available are as follows:

  * read(callback) - Read GPIO value asynchronously
  * readSync() - Read GPIO value synchronously
  * write(value, callback) - Write GPIO value asynchronously
  * writeSync(value) - Write GPIO value synchronously
  * watch(callback) - Watch and wait for GPIO to interrupt
  * direction() - Read GPIO direction
  * edge() - Read GPIO interrupt generating edge
  * unexport() - Reverse the effect of exporting the GPIO to userspace

GPIOs on Linux are identified by unsigned integers. These are the numbers that
should be passed to the onoff Gpio constructor function when exporting GPIOs
to userspace. For example, pin P1_11 on the Raspberry Pi P1 expansion header
corresponds to GPIO #17 in Raspbian Linux. 17 is therefore the number to pass
to the onoff Gpio constructor when using pin P1_11 on the P1 expansion header.

## Installation

    $ npm install onoff

## Blink the LED on GPIO #17 for 5 seconds

Thr and the following two examples need to be run by the superuser to function
correctly. The final example demonstrates a technique for handling superuser
issues.

```js
var Gpio = require('onoff').Gpio, // Constructor function for Gpio objects.
    ledGpio = new Gpio(17, 'out'),   // Export GPIO #17 as an output.
    iv;

// Toggle the state of the LED on GPIO #17 every 200ms.
// Here synchronous methods are used. Asynchronous methods are also available.
iv = setInterval(function() {
    ledGpio.writeSync(ledGpio.readSync() === 0 ? 1 : 0); // 1 = on, 0 = off.
}, 200);

// Stop blinking the LED and turn it off after 5 seconds.
setTimeout(function() {
    clearInterval(iv);    // Stop blinking
    ledGpio.writeSync(0); // Turn LED off.
    ledGpio.unexport();   // Unexport GPIO and free resources
}, 5000);
```

## Blink the LED on GPIO #17 20 times

```js
var Gpio = require('onoff').Gpio, // Constructor function for Gpio objects.
    ledGpio = new Gpio(17, 'out');   // Export GPIO #17 as an output.

// Toggle the state of the LED on GPIO #17 every 200ms 'count' times.
// Here asynchronous methods are used. Synchronous methods are also available.
function blink(count) {
    if (count <= 0) return ledGpio.unexport();

    // Asynchronous read.
    ledGpio.read(function(err, value) {
        if (err) throw err;
        // Asynchronous write.
        ledGpio.write(value === 0 ? 1 : 0, function(err) {
            if (err) throw err;
        });
    });

    setTimeout(function() {
        blink(count -1);
    }, 200);
}

blink(20);
```

## Wait for the button on GPIO #18 to interrupt

```js
var Gpio = require('onoff').Gpio,  // Constructor function for Gpio objects.
    buttonGpio = new Gpio(18, 'in', 'both'); // Export GPIO #18 as an interrupt
                                             // generating input.

console.log('Please press the button on GPIO #18...');

// The callback passed to watch will be called when the button on GPIO #18 is
// pressed. 
buttonGpio.watch(function (err, value) {
    if (err) throw err;

    console.log('Button pressed!, its value was ' + value);

    buttonGpio.unexport(); // Unexport GPIO and free resources
});
```

## How handle superuser issues

In gereral, superuser privileges are required for exporting and using GPIOs.
However, running all processes that access GPIOs as the superuser will be
unacceptable for most. To resolve this issue onoff can be used as follows:

Step 1 - Export GPIOs as superuser

Create a simple program for exporting GPIOs and execute this program with
superuser privileges. In addition to exporting the GPIOs, this program
will automatically change the access permissions for the GPIOs value file
allowing all users read and write access.
 
```js
var Gpio = require('onoff').Gpio,
    ledGpio = new Gpio(17, 'out');
```

Step 2 - The application can be run by a non-superuser

Highspeed blinking:

```js
var Gpio = require('../../onoff').Gpio,
    ledGpio = new Gpio(/* 38 */ 17, 'out'),
    time = process.hrtime(),
    herz,
    i;

for (i = 0; i != 50000; i += 1) {
    ledGpio.writeSync(1);
    ledGpio.writeSync(0);
}

time = process.hrtime(time);
herz = Math.floor(i / (time[0] + time[1] / 1E9));

console.log('Frequency = ' + herz / 1000 + 'KHz');
```

Step 3 - Unexport GPIOs as superuser

Create a simple program for unexporting GPIOs and execute this program with
superuser privileges.
 
```js
var Gpio = require('onoff').Gpio,
    ledGpio = new Gpio(17, 'out');

ledGpio.unexport();
```

## Additional Information

onoff has been tested on the BeagleBone (Ångström) and Raspberry Pi (Raspbian).
The suitability of onoff for a particular Linux board is highly dependent on
how GPIO interfaces are made available on that board. The
[GPIO interfaces](http://www.kernel.org/doc/Documentation/gpio.txt)
documentation describes GPIO access conventions rather than standards that must
be followed so GPIO can vary from platform to platform. For example, onoff
relies on sysfs files located at /sys/classes/gpio being available. However,
these sysfs files for userspace GPIO are optional and may not be available on a
particular platform.

As its name hopefully indicates, onoff can be used for turning things on and
off and detecting interrupts at a "reasonable" frequency. It's not intended for
use in [bit banging](http://en.wikipedia.org/wiki/Bit_banging) applications.

