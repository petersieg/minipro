### Build directory for Mac OSX 10.6.8 Snow Leopard

Contains build minipro 0.6. Require Mac Ports libusb (GCC 4.2 + pkgconfig to rebuild)

To make it compile, some files (prom.c, tl866a.c and tl866IIplus.c) had to be changed:

Declare for loop variables outside the for loop (C99).

Put libusb into lib search path.
