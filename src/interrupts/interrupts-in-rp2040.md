# Interrupts in the RP2040

In the previous chapter, we looked at what interrupts are and the role of the NVIC. Now, lets look at which interrupts are actually available on the RP2040.

Interrupts fall into two groups: system exceptions and external interrupts.

System exceptions are defined by the CPU architecture itself. These include reset and fault handlers. They behave the same way across most Cortex-M chips. 

External interrupts come from peripherals on the RP2040. Each peripheral that can generate an interrupt has an IRQ number and a vector name. These are the names you will see in code.

The table below shows the external interrupts on the RP2040, numbered from 0 to 25. They cover common peripherals such as timers, GPIO, DMA, and communication interfaces like I2C, SPI, and UART.

You do not need to memorize this table. Its purpose is to help you recognize where names like `I2C0_IRQ` or `UART0_IRQ` come from when you see them in examples or documentation.

The full details are in the [RP2040 datasheet, section 2.3.2](https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf) on page 60.

**Important note**: The RP2040 has two Cortex-M0+ cores, each with its own NVIC. On each core, only the lower 26 IRQ signals (0-25) are connected to hardware peripherals. IRQs 26-31 are not connected and will never fire.

In the next chapter, we will see how Embassy uses these interrupts without requiring you to write interrupt handlers manually.

## RP2040 External Interrupts 

The RP2040 has 26 external hardware interrupt lines (IRQ 0-25) connected to each core's NVIC. IRQs 26-31 exist in the NVIC but are not connected to hardware peripherals and will never fire from hardware.

## Timers

| IRQ | Vector        | Description    |
|-----|---------------|----------------|
| 0   | TIMER_IRQ_0   | Timer alarm 0 |
| 1   | TIMER_IRQ_1   | Timer alarm 1 |
| 2   | TIMER_IRQ_2   | Timer alarm 2 |
| 3   | TIMER_IRQ_3   | Timer alarm 3 |

## PWM

| IRQ | Vector        | Description                          |
|-----|---------------|--------------------------------------|
| 4   | PWM_IRQ_WRAP  | Shared PWM interrupt (all slices)    |

## USB

| IRQ | Vector        | Description                |
|-----|---------------|----------------------------|
| 5   | USBCTRL_IRQ   | USB controller interrupt   |

## External Flash (XIP)

| IRQ | Vector   | Description              |
|-----|----------|--------------------------|
| 6   | XIP_IRQ  | XIP interface interrupt |

## PIO

| IRQ | Vector        | Description              |
|-----|---------------|--------------------------|
| 7   | PIO0_IRQ_0    | PIO 0 interrupt 0        |
| 8   | PIO0_IRQ_1    | PIO 0 interrupt 1        |
| 9   | PIO1_IRQ_0    | PIO 1 interrupt 0        |
| 10  | PIO1_IRQ_1    | PIO 1 interrupt 1        |

## DMA

| IRQ | Vector        | Description            |
|-----|---------------|------------------------|
| 11  | DMA_IRQ_0     | DMA interrupt 0        |
| 12  | DMA_IRQ_1     | DMA interrupt 1        |

## GPIO and Core I/O

| IRQ | Vector           | Description                                   |
|-----|------------------|-----------------------------------------------|
| 13  | IO_IRQ_BANK0    | GPIO bank 0 interrupt                          |
| 14  | IO_IRQ_QSPI     | QSPI IO bank interrupt                         |
| 15  | SIO_IRQ_PROC0   | Core 0 SIO interrupt                           |
| 16  | SIO_IRQ_PROC1   | Core 1 SIO interrupt                           |

## Clock

| IRQ | Vector        | Description              |
|-----|---------------|--------------------------|
| 17  | CLOCKS_IRQ    | Clock system interrupt   |

## SPI

| IRQ | Vector     | Description        |
|-----|------------|--------------------|
| 18  | SPI0_IRQ   | SPI0 interrupt     |
| 19  | SPI1_IRQ   | SPI1 interrupt     |

## UART

| IRQ | Vector     | Description        |
|-----|------------|--------------------|
| 20  | UART0_IRQ  | UART0 interrupt    |
| 21  | UART1_IRQ  | UART1 interrupt    |

## ADC

| IRQ | Vector          | Description                              |
|-----|-----------------|------------------------------------------|
| 22  | ADC_IRQ_FIFO   | ADC FIFO interrupt                       |

## I2C

| IRQ | Vector     | Description        |
|-----|------------|--------------------|
| 23  | I2C0_IRQ   | I2C0 interrupt     |
| 24  | I2C1_IRQ   | I2C1 interrupt     |

## Real-Time Clock

| IRQ | Vector     | Description        |
|-----|------------|--------------------|
| 25  | RTC_IRQ    | RTC interrupt      |
