# Qualcomm msm pinctrl
- Max 4 top-level mode multiplexers (TLMM) with their own mmio regions
- Max 300 gpios
- chip-specific amount of pin groups inside tlmm's
- chip-specific pin group locations and bits

## Programming considerations
- To acknowledge an irq for a pin group either write one as the interrupt status bit to the interrupt status register
if the pin group uses high acknowledged interrupts, else write zero.
- If a pin has a gpio irq installed the interrupt detection logic
will still see changes in pin state even when the mux is set to a different one,
to prevent issues caused by this disable the irq before doing any register modifications
necessary for changing the mux and clear any pending irq's inside the irq controller when
all the modifications are done along with acknowledging the irq inside the pin group's
interrupt status register before re-enabling the irq.
- To prevent pin glitching when changing mux setting of a pin group with direction set to output
(as determined by reading from the control register)
to the gpio function the first time, read the IO register and if it's 
input is active + output is not active set the output as active, else if it's
output is active clear it.
- If the chip specifies an egpio function:
When changing mux setting of a pin group to the egpio function
check if the pin indicates support for egpio and if it does clear the egpio enable bit.
Else if the function is not the egpio function set the egio enable bit to claim pin ownership.
