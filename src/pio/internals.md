

### TX FIFO and RX FIFO

The TX FIFO is for data going into the state machine. Your program pushes data in, and the state machine pulls it out when it needs it. The RX FIFO works the other way. The state machine pushes data in, and your program reads it out.

Each FIFO holds four 32-bit words. That is not a lot. But you can hook them up to DMA, which can keep them full or empty automatically without the CPU doing anything.

If you only need data to flow in one direction, you can join the TX and RX FIFOs into one FIFO that is eight words deep. You pick the direction when you set up the state machine.

### ISR and OSR

These are the shift registers. They are how data actually moves between the FIFOs and the pins.

The OSR is the Output Shift Register. When the state machine pulls a word from the TX FIFO, it goes into the OSR. From there, the state machine shifts bits out of it one at a time, or a few at a time, and sends them to the pins.

The ISR is the Input Shift Register. It works the other way. Bits come in from the pins and get shifted into the ISR. When enough bits have been collected, the ISR pushes the result into the RX FIFO.

Think of the shift registers as the middle layer. The FIFOs hold full 32-bit words. The pins deal with individual bits. The shift registers sit between them and do the conversion.


### X and Y

Two scratch registers. Each one is 32 bits. You can use them to store a number, count down, or hold a value you want to use later. They are not for complex math. They are for simple things like a counter or a loop variable.
