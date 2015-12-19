node-rpio
=========

This is a high performance nodejs addon which provides access to the Raspberry
Pi GPIO interface, supporting regular GPIO as well as i²c, PWM, and SPI.

[![Build Status](https://travis-ci.org/jperkin/node-rpio.svg?branch=master)](https://travis-ci.org/jperkin/node-rpio)
[![NPM version](https://badge.fury.io/js/rpio.svg)](http://badge.fury.io/js/rpio)

Most other GPIO modules use the `/sys` file system interface, whereas this
addon links directly to Mike McCauley's
[bcm2835](http://www.open.com.au/mikem/bcm2835/) library which `mmap()`s the
underlying hardware.  This makes this module significantly faster than the
alternatives, provides synchronous access making code a lot simpler, as well as
supporting the additional i²c, PWM, and SPI functions which are not all
available via the `/sys` interface.

How much faster?  Here is a [simple
test](https://gist.github.com/jperkin/e1f0ce996c83ccf2bca9):

|              Module | Turn a pin on/off 1 million times (seconds) |
|--------------------:|--------------------------------------------:|
|   rpi-gpio (`/sys`) |                                     701.023 |
|     rpio (`mmap()`) |                                       2.907 |

Writing to the hardware directly also means you don't need any asynchronous
callback handling to wait for `/sys` operations to complete, greatly
simplifying code.  The one drawback is that you need to run as root for access
to `/dev/mem`.  If this is unsuitable for your application you'll need to use
one of the `/sys` modules where you can configure permissions to regular users
via the `gpio` group.

## Install

Easily install the latest via npm:

```bash
$ npm install rpio
```

## API

Requiring the addon initializes the bcm2835 library ready to accept commands.

```js
var rpio = require('rpio');
```

### GPIO

There are two naming schemes when referring to GPIO pins:

* By their physical header location: Pins 1 to 26 (A/B) or Pins 1 to 40 (A+/B+)
* Using the Broadcom hardware map: GPIO 0-25 (B rev1), GPIO 2-27 (A/B rev2, A+/B+)

However, confusingly the Broadcom GPIO map changes between revisions, so for
example pin 3 maps to GPIO 0 on Model B Revision 1 models, but maps to GPIO 2
on all later models.

This means the only sane default mapping is the physical layout, so that the
same code will work on all models regardless of the underlying GPIO mapping.

If you prefer to use the Broadcom GPIO scheme for whatever reason, you can use
`.setLayout()` to switch:

```js
rpio.setLayout('physical');     /* Use the default, physical P1 - P26/P40 */
rpio.setLayout('gpio');         /* Use the Broadcom GPIO numbering */
```

Before reading from or writing to a pin you need to configure it as read-only
or read-write.

```js
rpio.setInput(11);         /* Configure pin 11 as read-only */
rpio.setOutput(12);        /* Configure pin 12 as read-write */

/* Alternatively use the general purpose setFunction() */
rpio.setFunction(11, rpio.INPUT);
rpio.setFunction(12, rpio.OUTPUT);
```

Now you can read or write values with `.read()` and `.write()`.  The two
`rpio.LOW` and `rpio.HIGH` constants are provided if you prefer to avoid magic
numbers.

```js
/* Read value of pin 11 */
console.log('Pin 11 is set to ' + rpio.read(11));

/* Set pin 12 high (i.e. write '1' to it) */
rpio.write(12, rpio.HIGH);

/* Set pin 12 low (i.e. write '0' to it) */
rpio.write(12, rpio.LOW);
```

#### GPIO Demo

The code below continuously flashes an LED connected to pin 11 at 100Hz.

```js
var rpio = require('rpio');

rpio.setOutput(11);

/* Set the pin high every 10ms, and low 5ms after each transition to high */
setInterval(function() {
        rpio.write(11, rpio.HIGH);
        setTimeout(function() {
                rpio.write(11, rpio.LOW);
        }, 5);
}, 10);
```

### i²c

i²c is primarily of use for driving LCD displays, and makes use of pins 3 and 5
(GPIO0/GPIO1 on Rev 1, GPIO2/GPIO3 on Rev 2 and newer).  The bcm2835 library
automatically detects which Raspberry Pi revision you are running, so you do
not need to worry about which i²c bus to configure.

To get started call `.i2cBegin()` which assigns pins 3 and 5 to i²c use.  Until
`.i2cEnd()` is called they won't be available for GPIO use.

```js
rpio.i2cBegin();
```

Configure the slave address.  This is between `0 - 0x7f`, and it can be helpful
to run the `i2cdetect` program to figure out where your devices are if you are
unsure.

```js
rpio.i2cSetSlaveAddress(0x20);
```

Set the baud rate.  You can do this two different ways, depending on your
preference.  Either use `.i2cSetBaudRate()` to directly set the speed in hertz,
or `.i2cSetClockDivider()` to set it based on a divisor of the base 250MHz
rate.

```js
rpio.i2cSetBaudRate(100000);    /* 100kHz */
rpio.i2cSetClockDivider(2500);  /* 250MHz / 2500 = 100kHz */
```

Read from and write to the i²c slave.  Both functions take a buffer and
optional length argument, defaulting to the length of the buffer if not
specified.

```js
var txbuf = new Buffer([0x0b, 0x0e, 0x0e, 0x0f]);
var rxbuf = new Buffer(32);

rpio.i2cWrite(txbuf);		/* Sends 4 bytes */
rpio.i2cRead(rxbuf, 16);	/* Reads 16 bytes */
```

Finally, turn off the i²c interface and return the pins to GPIO.

```js
rpio.i2cEnd();
```

#### i²c Demo

The code below writes two strings to a 16x2 LCD.

```js
var rpio = require('rpio');

/*
 * Magic numbers to initialise the i2c display device and write output,
 * cribbed from various python drivers.
 */
var init = new Buffer([0x03, 0x03, 0x03, 0x02, 0x28, 0x0c, 0x01, 0x06]);
var LCD_LINE1 = 0x80, LCD_LINE2 = 0xc0;
var LCD_ENABLE = 0x04, LCD_BACKLIGHT = 0x08;

/*
 * Data is written 4 bits at a time with the lower 4 bits containing the mode.
 */
function lcdwrite4(data)
{
        rpio.i2cWrite(Buffer([(data | LCD_BACKLIGHT)]));
        rpio.i2cWrite(Buffer([(data | LCD_ENABLE | LCD_BACKLIGHT)]));
        rpio.i2cWrite(Buffer([((data & ~LCD_ENABLE) | LCD_BACKLIGHT)]));
}
function lcdwrite(data, mode)
{
        lcdwrite4(mode | (data & 0xF0));
        lcdwrite4(mode | ((data << 4) & 0xF0));
}

/*
 * Write a string to the specified LCD line.
 */
function lineout(str, addr)
{
        lcdwrite(addr, 0);

        str.split('').forEach(function (c) {
                lcdwrite(c.charCodeAt(0), 1);
        });
}

/*
 * We can now start the program, talking to the i2c LCD at address 0x27.
 */
rpio.i2cBegin();
rpio.i2cSetSlaveAddress(0x27);
rpio.i2cSetBaudRate(10000);

for (var i = 0; i < init.length; i++)
        lcdwrite(init[i], 0);

lineout("node.js i2c LCD!", LCD_LINE1);
lineout("npm install rpio", LCD_LINE2);

rpio.i2cEnd();
```

### PWM

Pulse Width Modulation (PWM) allows you to create analog output from the
digital pins.  This can be used, for example, to make an LED appear to pulse
rather than be fully off or on.

Only certain pins support PWM:

* 26-pin models: pin 12
* 40-pin models: pins 12, 19, 33, 35

To enable a PIN for PWM, use the `rpio.PWM` argument to `setFunction()`:

```js
rpio.setFunction(12, rpio.PWM); /* Use pin 12 */
```

Set the PWM refresh rate with `pwmSetClockDivider()`.  This is a power-of-two
divisor of the base 19.2MHz rate, with a maximum value of 4096 (4.6875kHz).

```js
rpio.pwmSetClockDivider(64);    /* Set PWM refresh rate to 300kHz */
```

Set the PWM range for a pin with `pwmSetRange()`.  This determines the maximum
pulse width.

```js
rpio.pwmSetRange(12, 1024);
```

Finally, set the PWM width for a pin with `pwmSetData()`.

```js
rpio.pwmSetData(12, 512);
```

#### PWM Demo

The code below pulses an LED 5 times before exiting.

```js
var rpio = require('rpio');

var pin = 12;           /* P12/GPIO18 */
var range = 1024;       /* LEDs can quickly hit max brightness, so only use */
var max = 128;          /*   the bottom 8th of a larger scale */
var clockdiv = 8;       /* Clock divider (PWM refresh rate), 8 == 2.4MHz */
var interval = 5;       /* setInterval timer, speed of pulses */
var times = 5;          /* How many times to pulse before exiting */

/*
 * Enable PWM on the chosen pin and set the clock and range.
 */
rpio.setFunction(pin, rpio.PWM);
rpio.pwmSetClockDivider(clockdiv);
rpio.pwmSetRange(pin, range);

/*
 * Repeatedly pulse from low to high and back again until times runs out.
 */
var direction = 1;
var data = 0;
var pulse = setInterval(function() {
        rpio.pwmSetData(pin, data);
        if (data === 0) {
                direction = 1;
                if (times-- === 0) {
                        clearInterval(pulse);
                        rpio.setFunction(pin, rpio.INPUT);
                        return;
                }
        } else if (data === max) {
                direction = -1;
        }
        data += direction;
}, interval, data, direction, times);
```

### SPI

SPI switches pins 19, 21, 23, 24 and 25 (GPIO7-GPIO11) to a special mode where
you can bulk transfer data at high speeds to and from SPI devices, with the
controller handling the chip enable, clock and data in/out functions.

| Pin | Function |
|----:|---------:|
|  19 |     MOSI |
|  21 |     MISO |
|  23 |     SCLK |
|  24 |      CE0 |
|  25 |      CE1 |

Once SPI is enabled, the SPI pins are unavailable for GPIO use until `spiEnd()`
is called.

```js
rpio.spiBegin();           /* Switch GPIO7-GPIO11 to SPI mode */
```

Choose which of the chip select / chip enable pins to control:

```js
/*
 *  Value | Pin
 *  ------|---------------------
 *    0   | SPI_CE0 (24 / GPIO8)
 *    1   | SPI_CE1 (25 / GPIO7)
 *    2   | Both
 */
rpio.spiChipSelect(0);
```

Commonly chip enable (CE) pins are active low, and this is the default.  If
your device's CE pin is active high, use `spiSetCSPolarity()` to change the
polarity.

```js
rpio.spiSetCSPolarity(0, rpio.HIGH);    /* Set CE0 high to activate */
```

Set the SPI clock speed with `spiSetClockDivider()`.  This is a power-of-two
divisor of the base 250MHz rate, defaulting to `0 == 65536 == 3.81kHz`.

```js
rpio.spiSetClockDivider(128);   /* Set SPI speed to 1.95MHz */
```

Set the SPI Data Mode:

```js
/*
 *  Mode | CPOL | CPHA
 *  -----|------|-----
 *    0  |  0   |  0
 *    1  |  0   |  1
 *    2  |  1   |  0
 *    3  |  1   |  1
 */
rpio.spiSetDataMode(0);         /* 0 is the default */
```

Once everything is set up we can transfer data.  Data is sent and received in
8-bit chunks via buffers which should be the same size.

```js
var txbuf = new Buffer([0x3, 0x0, 0xff, 0xff]);
var rxbuf = new Buffer(txbuf.length);

rpio.spiTransfer(txbuf, rxbuf, txbuf.length);
```

If you only need to send data and do not care about the data coming back, you
can use the slightly faster `spiWrite()` call:

```js
rpio.spiWrite(txbuf, txbuf.length);
```

When you're finished call `.spiEnd()` to release the pins back to general
purpose use.

```js
rpio.spiEnd();
```

#### SPI Demo

The code below reads the 128x8 contents of an AT93C46 serial EEPROM.

```js
var rpio = require('rpio');

rpio.spiBegin();
rpio.spiChipSelect(0);                  /* Use CE0 */
rpio.spiSetCSPolarity(0, rpio.HIGH);    /* AT93C46 chip select is active-high */
rpio.spiSetClockDivider(128);           /* AT93C46 max is 2MHz, 128 == 1.95MHz */
rpio.spiSetDataMode(0);

/*
 * There are various magic numbers below.  A quick overview:
 *
 *   tx[0] is always 0x3, the EEPROM "READ" instruction.
 *   tx[1] is set to var i which is the EEPROM address to read from.
 *   tx[2] and tx[3] can be anything, at this point we are only interested in
 *     reading the data back from the EEPROM into our rx buffer.
 *
 * Once we have the data returned in rx, we have to do some bit shifting to
 * get the result in the format we want, as the data is not 8-bit aligned.
 */
var tx = new Buffer([0x3, 0x0, 0x0, 0x0]);
var rx = new Buffer(4);
var out;
var i, j = 0;

for (i = 0; i < 128; i++, ++j) {
        tx[1] = i;
        rpio.spiTransfer(tx, rx, 4);
        out = ((rx[2] << 1) | (rx[3] >> 7));
        process.stdout.write(out.toString(16) + ((j % 16 == 0) ? "\n" : " "));
}
rpio.spiEnd();
```

## Authors And Licenses

Mike McCauley wrote `src/bcm2835.{c,h}` which are under the GPL.

I wrote the rest, which is under the ISC license unless otherwise specified.
