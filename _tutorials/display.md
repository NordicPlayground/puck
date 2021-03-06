---
layout: tutorial
title: Display Puck
description: How to create your own Display Puck
image: images/displaypuck.png
order: 4
---

> ![](../images/displaypuck.png)

This tutorial is part of a tutorial series on the [Nordic Pucks](../tutorials.html).
I am assuming that you have already read the [introduction tutorial] and the [location puck tutorial](location.html).

This tutorial will describe how to create a Bluetooth LE-enabled puck with an e-paper display. The display we have used is 264 x 176 pixels, and we're going to design a protocol which lets a connected device transmit an arbitrary black/white image to it via BLE. We will go through the necessary hardware setup, as well as the code needed to support it.

[View the finished project on mbed.org](https://mbed.org/teams/Nordic-Pucks/code/display-puck/)

> ![](../images/display-closeup.jpg)
> If you're cramped for cash, you can try building casing out of cardboard and packaging tape.

# Requirements
- 1x [Nordic nRF51822 mbed mKIT development board](https://mbed.org/platforms/Nordic-nRF51822/) ([Buy one here!](http://www.semiconductorstore.com/cart/pc/viewPrd.asp?idproduct=49226))
- 1x [E-Paper display](http://www.seeedstudio.com/depot/27-ePaper-Panel-p-1596.html?cPath=34_82)
- 1x [E-Paper shield](http://www.seeedstudio.com/depot/Small-epaper-Shield-p-1597.html)
- 20x [Female -> Female jumper wires](http://www.seeedstudio.com/depot/1-pin-dualfemale-jumper-wire-100mm-50pcs-pack-p-260.html?cPath=44_47)
- 5x [Male -> Female jumper wires](http://www.seeedstudio.com/depot/1-Pin-FemaleMale-Jumper-Wire-125mm-50pcs-pack-p-1319.html?cPath=44_47)


# Hardware

We will be using an e-paper display from seeedstudio. It comes with a shield which we are going to wire onto our nRF51822.


> ![](../images/display_wrapped.jpg)
> The display as it comes from seeedstudio.

The e-paper shield is an arduino shield, but since we're using a different form-factor we will have to run all the connections with single wires instead of simply plugging the shield directly. We connected the following pins using single wires:

> |----------+-----------------------|
> | nRF51822 | Shield                |
> |----------+-----------------------|
> | p2       | D2 (M_EPD_PANEL_ON)   |
> | p3       | D3 (M_EPD_BORDER)     |
> | p4       | D4 (M_/SD_CS)         |
> | p5       | D5 (M_EPD_PWM)        |
> | p6       | D6 (M_EPD_/RESET)     |
> | p7       | D7 (M_EPD_BUSY)       |
> | p8       | D8 (M_EPD_DISCHARGE)  |
> | p26      | D9 (M_/WORD_STOCK_CS) |
> | p0       | D10 (M_/EPD_CS)       |
> | p27      | A1 (M_OE123)          |
> | p28      | A2 (M_CKV)            |
> | p29      | A3 (M_STV_IN)         |
> |----------+-----------------------|
> | p20      | MOSI                  |
> | p22      | MISO                  |
> | p25      | SCK                   |
> |----------+-----------------------|
> | VCC      | VCC                   |
> | GND      | GND                   |
> |----------+-----------------------|
> 
> Pin mapping between the mbed and the e-paper shield.

> ![](../images/IMG_2675.JPG)
> Wires connected to the shield.

In the future a pin adaptor might be available for the nRF51822 which will make it easier to connect arduino shields without using single wires.

# Software

Now that the hardware is all assembled, let's get on to writing some code.
We assume you've already read the [Location Puck tutorial](location.html).
We're going to use the same Puck library for the display, so we create a new project and set it up just like the location puck.

The e-paper display shield has an SPI interface, which is the interface we are going to use to flash it with images.
Luckily, we have ported an already existing Arduino e-paper library over to mbed which takes care of the protocol details.
This library should be included in your project. Grab it at [our seeedstudio-epaper repo on mbed](http://mbed.org/teams/Nordic-Pucks/code/seeedstudio-epaper/).
With it included, the EPD _(Electronic Paper Display)_ can be declared with the proper pin settings:

> {% highlight cpp %}
EPD_Class EPD(p0, p2, p3, p8, p5, p6, p7);
{% endhighlight %}

This gives us an `EPD_Class` upon which we can call convienient high-level methods such as `image` and `clear` to flash images, and clear the screen, respectively.

Now that we have the possibility to write images to the screen, we need a way to transmit images over Bluetooth LE.
For the display puck we have defined a custom GATT service ([read more about how GATT works in our BLE tutorial](ble.html)) with it's own UUID `"bftj display    "`.
A bluetooth UUID is 128 bits long, so we use a convention with 16 letters of 8 bit each. `'bftj'` is a general prefix we've decided to use for all our pucks' UUIDs, to avoid collisions with other vendors.

The display service provides two characteristics: one for sending commands such as `start image` and `end image`, and one for transmitting serialized compressed image data.

The nRF51822 doesn't have a lot of available memory (8kB of RAM is available to the programmer). As the display has 46464 pixels, we're not able to store entire images in memory at once.
Because of this, our protocol revolves around sending and rendering images in chunks, so that only parts of the image ever needs to reside in memory at once.
Specifically, the image transmission protocol first transfers the LZ77-compressed upper half of the image, in-place-decompresses it, and then draws it to the e-paper display.
The same is then done for the lower part of the image.

To control this flow, we need to define some commands:

> {% highlight cpp %}
#define COMMAND_NOOP 0
#define COMMAND_CLEAR 1
#define COMMAND_IMAGE_UPPER 2
#define COMMAND_IMAGE_LOWER 3
#define COMMAND_BEGIN_UPPER 4
#define COMMAND_BEGIN_LOWER 5
{% endhighlight %}

Each pixel in the display can only be completely black or completely white (no greys, no colors), which means we only need one bit for each pixel.
Using som bit-twiddling operations, we can pack 8 pixels in each byte of data.
This means that a bit-packed image takes 5808 bytes of storage.
Since we send only half an image at a time, we need to be able to hold 2904 bytes of image data in RAM at a time.

To reduce the amount of data that needs to be sent over the bluetooth link, the bit-packed image is compressed before transmission.
When researching data compression algorithms, the LZ77 algorithm was found to have very little memory overhead during decompression (a plus in our memory constrained system) as well as a decent compression rate.
It has a worst-case compressed file size slightly larger than the decompressed file size, so to be safe the receive buffer is initialized with a little extra room.

> {% highlight cpp %}
#define IMAGE_SIZE 2904
#define BUFFER_SIZE 2917
#define Y_SIZE 88
#define X_SIZE 264
uint8_t buffer[BUFFER_SIZE];
int currentCommand = COMMAND_NOOP;
{% endhighlight %}

Because of little available memory, we can't afford the luxury of having separate receive and image buffers.
Therefore, received data is stored at the end of the image buffer, and will be decompressed to the same space.
As we use the LZ77 compression algorithm, data will be decompressed incrementally. As data is decompressed, space will be freed up in the image buffer for storing the image.

Now, onto the code for receiving the command instructions.
When we receive an `IMAGE_BEGIN` command, we set the `receiveIndex` (the index that we will write the received data to) to the end of the image buffer.
It is then written backwards into the buffer as it is received. When we receive a command that tells us all the data has been transmitted, the received segment is flipped, decompressed, and finally written to the display.

> ![](../images/receive%20image%20data.png)

The code for this is as follows. Most of the magic here is happening inside the LZ-compression, which we will not be looking into. The decompressed image is then flashed to the display.

> {% highlight cpp %}
case COMMAND_IMAGE_UPPER:
    puck->disconnect();
    LOG_INFO("Writing image to top half of display.\n");
    reverseBufferSegment(receiveIndex, BUFFER_SIZE);
    LZ_Uncompress(buffer + receiveIndex, buffer, BUFFER_SIZE - receiveIndex);
    LOG_INFO("Uncompressed %i bytes of data.\n", BUFFER_SIZE - receiveIndex);
    EPD.begin(EPD_2_7);
    EPD.start();
    EPD.image(buffer, 0, EPD.lines_per_display / 2);
    EPD.end();
    break;
{% endhighlight %}

The code for flashing the lower half is more or less the same, with the exception of the coordinates we pass to the EPD.image method.

Next, receiving the data itself. The data characteristic is 20 bytes long (the maximum size), so we need to send the data in chunks of 20 bytes.
All we really need is a callback for the data characteristic write, which stores the received bytes in the receive buffer.

Now that we've got the methods for handling data transfer ready, let's hook it all up in the main function.


> {% highlight cpp %}
int main() {
    DigitalOut SD_CS(p4);
    DigitalOut WORD_STOCK_CS(p26);
    SD_CS = 1;
    WORD_STOCK_CS = 1;
{% endhighlight %}

First, some additional setup for the display shield.
The shield supports some additional peripheral functions such as using an SD card for storage, but we are not going to use these features.
Therefore, we should bring their SPI Chip Select signals high to avoid any confusion on who is using the SPI bus.


> {% highlight cpp %}
puck->addCharacteristic(DISPLAY_SERVICE_UUID, COMMAND_UUID, 1);
puck->addCharacteristic(DISPLAY_SERVICE_UUID, DATA_UUID, 20);
{% endhighlight %}

Next, we need to let the puck library know which characteristics we want to use, and how long they are.



> {% highlight cpp %}
puck->onCharacteristicWrite(&COMMAND_UUID, onCommandWritten);
puck->onCharacteristicWrite(&DATA_UUID, onDataWritten);
{% endhighlight %}

We also hook up some callbacks to react on characteristic writes.

> {% highlight cpp %}
puck->init(0x5EED);
{% endhighlight %}

After we are done configuring our puck object, we can init the puck with a 16 bit identifier.
It is important that `init(..)` gets called after we are done doing configuration such as `addCharacteristic(..)` etc., as `init(..)` initializes the puck based on the configuration it has received.


> {% highlight cpp %}
while (puck->drive());
{% endhighlight %}

Finally, we keep the puck alive in a simple while loop. This will concede control to the Bluetooth LE Soft Device, letting it do its magic.
