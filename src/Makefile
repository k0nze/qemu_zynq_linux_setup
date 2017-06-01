CC=arm-linux-gnueabihf-gcc
CFLAGS=-mcpu=cortex-a9

all: helloworld

helloworld: helloworld.c
	$(CC) $(CFLAGS) helloworld.c -o helloworld

.PHONY clean:
	-rm helloworld
