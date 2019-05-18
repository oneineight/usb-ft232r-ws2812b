# usb-ft232r-ws2812b
Proof of concept code to drive ws2812b RGB LEDs (NeoPixels) over USB with FTDI FT232R serial adaptor.

![FT232R](https://github.com/oneineight/usb-ft232r-ws2812b/raw/master/FT232R.jpg "FT232R")
![WS2812B](https://github.com/oneineight/usb-ft232r-ws2812b/raw/master/WS2812B.jpg "WS2812B")

This allows a desktop/laptop/tablet/phone with a USB port to drive a strip/ring/etc of WS2812B devices rather than a microcontroller with UART/SPI/GPIO etc. Be careful not to overload your USB  port - some WS2812B can sink about 60 mA when set to full brightness white so don't power a long strip from USB.

The WS2812B is controlled by an unusual PWM (pulse width modulated) protocol on its single data pin (DIN). Each bit is encoded as a high pulse and then a low pulse. The length of the two pulses together is always the same but the relative difference in lengths between the pulses encodes a '1' or a '0'. The specification requires that the data line is clocked at a nominal 800kHz (bit length 1.25 μs ± 0.6 μs) with the pulses of lengths 400 ns, 450 ns, 800 ns & 850 ns ± 150 ns.

The FT232R is used to drive the WS2812B at 1MHz (bit length 1 μs) with pulse lengths of 333 ns & 667 ns. This is slightly outside of the specification but seems to work.

The FT232R is driven using libUSB from user space rather than the kernel serial port driver. The data is streamed as it would be via a serial port driver but the FT232R needs special configuration that isn't possible via the serial port driver.

Each WS2812B bit can be viewed as three bits from the FT232R, effectively a high start bit, a data bit and a low stop bit. The FT232R uses its maximum bit rate of 3Mbaud to provide three bits in 1 μs. The TX line from the FT232R floats high (5V). An asynchronous serial character from the FT232R as usually configured (8 bits, no parity, one stop bit) consists of a low (0V) start bit followed by the data bits and a final high (5V) stop bit.

We configure the FT232R to output 7 data bits so each serial character is encoded as 9 bits including the start and stop bit and can represent 3 WS2812B bits.

The WS2812B requires a high start bit, a low stop bit and a low of 50 μs to finish (reset). This is acheived by an FT232R specific feature to invert the TX line, so it floats low, has a high start bit, inverted data bits and a low stop bit.

The 24 bits (3 bytes) of colour data for each WS2812B device (8 bits red, 8 bits green, 8 bits blue) are encoded as 24 / 3 * 8 = 64 bits = 8 bytes.

FT232R characters are transmitted LSB first (ie right to left in diagram below)

  (0) c 1 0 b 1 0 a (1) 

Digits in brackets are the fixed start and stop bits. a, b & c are the WS2812B data bits.

For example, to transmit three WS2812B '0' bits we use

  (0) 0 1 0 0 1 0 0 (1)

but the TX line has been inverted so the data bits need to be inverted (we've already accounted for the start and stop bits being inverted) so the character sent is

  1011011 = 0x5B = 91 (decimal)

Sending 8 of these characters will be received by the WS2812B as 8 * 3 = 24 bits of colour data. As they are all 0 the LED will be off.

Sending more characters contiguously to a strip of WS2812B devices will cause the bits to be clocked down the strip. When the string of characters stops the TX line will revert to low (0V) which the WS2812B will see as a reset and display the new data.



