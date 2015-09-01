### **Chapter 7. Interrupts and Interrupt Handlers**

Processors can be orders of magnitudes faster than the hardware they talk to; it is not ideal for the kernel to issue a request and wait for a response from slower hardware. Instead, the kernel must be free to go and handle other work, dealing with the hardware only after that hardware has actually completed its work. [p113]

How can the processor work with hardware without impacting the machine’s overall performance? As one solution, **polling** incurs overhead, because it must occur repeatedly regardless of whether the hardware is active or ready. A better solution is to provide a mechanism for the hardware to signal to the kernel when attention is needed. This mechanism is called an **interrupt**. This chapter dicusses interrupts and how the kernel responds to them, with special functions called **interrupt handlers**.

### Interrupts

* **Interrupts enable hardware to signal to the processor.**
    * For example, as you type, the keyboard controller (the hardware device that manages the keyboard) issues an electrical signal to the processor to alert the operating system to newly available key presses. These electrical signals are interrupts. The processor receives the interrupt and signals the operating system to enable the operating system to respond to the new data.
* **Hardware devices generate interrupts asynchronously** (with respect to the processor clock). Consequently, the kernel can be interrupted at any time to process interrupts.

An interrupt is produced by electronic signals from hardware devices and directed into input pins on an interrupt controller (a simple chip that multi-plexes multiple interrupt lines into a single line to the processor):

1. Upon receiving an interrupt, the interrupt controller sends a signal to the processor.
2. The processor detects this signal and interrupts its current execution to handle the interrupt.
3. The processor can then notify the operating system that an interrupt has occurred, and the operating system can handle the interrupt appropriately.

Different devices are associated with different interrupts using a <u>unique value</u> associated with each interrupt. This enables the operating system to differentiate between interrupts and to know which hardware device caused which interrupt. In turn, the operating system can service each interrupt with its corresponding handler.

These interrupt values are often called [**interrupt request**](https://en.wikipedia.org/wiki/Interrupt_request_(PC_architecture)) (**IRQ**) lines:

* Each IRQ line is assigned a numeric value. For example, on the classic PC, IRQ zero is the timer interrupt and IRQ one is the keyboard interrupt.
* Some interrupts are dynamically assigned, such as interrupts associated with devices on the PCI bus. Other non-PC architectures have similar dynamic assignments for interrupt values.
* The kernel knows that a specific interrupt is associated with a specific device, and the kernel knows this. The hardware then issues interrupts to get the kernel’s attention.

#### Exceptions and Interrupts*

**Exceptions** are often discussed at the same time as interrupts. Unlike interrupts, exceptions occur synchronously with respect to the processor clock; they are often called **synchronous interrupts**. Exceptions are produced by the processor while executing instructions either in response to a programming error (e.g. divide by zero) or abnormal conditions that must be handled by the kernel (e.g. a page fault). Because many processor architectures handle exceptions in a similar manner to interrupts, the kernel infrastructure for handling the two is similar.

Simple definitions of the two:

* **Interrupts**: asynchronous interrupts generated by hardware.
* **Exceptions**: synchronous interrupts generated by the processor.

System calls (one type of exception) on the x86 architecture are implemented by the issuance of a software interrupt, which traps into the kernel and causes execution of a special system call handler. Interrupts work in a similar way, except hardware (not software) issues interrupts.

### Interrupt Handlers

An [**interrupt handler**](https://en.wikipedia.org/wiki/Interrupt_handler) or **interrupt service routine** (ISR) is the function that the kernel runs in response to a specific interrupt:

* **Each device that generates interrupts has an associated interrupt handler.**
* **The interrupt handler for a device is part of the device’s [**driver**](https://en.wikipedia.org/wiki/Device_driver)** (the kernel code that manages the device).

In Linux, interrupt handlers are normal C functions, which match a specific prototype and thus enables the kernel to pass the handler information in a standard way. What differentiates interrupt handlers from other kernel functions is that the kernel invokes them in response to interrupts and that they run in a special context called **interrupt context**. This special context is occasionally called **atomic context** because code executing in this context is unable to block.

Because an interrupt can occur at any time, an interrupt handler can be executed at any time. It is imperative that the handler runs quickly, to resume execution of the interrupted code as soon as possible. It is important that

* To the hardware: the operating system services the interrupt without delay.
* To the rest of the system: the interrupt handler executes in as short a period as possible.

At the very least, an interrupt handler’s job is to acknowledge the interrupt’s receipt to the hardware. However, interrupt handlers can oftern have a large amount of work to perform.

### Top Halves Versus Bottom Halves

These two goals of an interrupt handler conflict with one another:

* Execute quickly
* Perform a large amount of work

Because of these competing goals, the processing of interrupts is split into two parts, or **halves**:

* **Top half.** The interrupt handler is the top half.  The top half is run immediately upon receipt of the interrupt and performs only the work that is time-critical, such as acknowledging receipt of the interrupt or resetting the hardware.
* **Bottom half.** Work that can be performed later is deferred until the bottom half. The bottom half runs in the future, at a more convenient time, with all interrupts enabled.

Linux provides various mechanisms for implementing bottom halves (discussed in [Chapter 8](ch8.md)).

For example using the network card:

1. When network cards receive packets from the network, the network cards immediately issue an interrupt. This optimizes network throughput and latency and avoids timeouts. [p115]
2. The kernel responds by executing the network card’s registered interrupt.
3. The interrupt runs, acknowledges the hardware, copies the new networking packets into main memory, and readies the network card for more packets. These jobs are the important, time-critical, and hardware-specific work.
    * The kernel generally needs to quickly copy the networking packet into main memory because the network data buffer on the networking card is fixed and miniscule in size, particularly compared to main memory. Delays in copying the packets can result in a buffer overrun, with incoming packets overwhelming the networking card’s buffer and thus packets being dropped.
    * After the networking data is safely in the main memory, the interrupt’s job is done, and it can return control of the system to whatever code was interrupted when the interrupt was generated.
4. The rest of the processing and handling of the packets occurs later, in the bottom half.

This chapter discusses the top half. The next chapter covers the bottom.

### Registering an Interrupt Handler

Each device has one associated driver. If that device uses interrupts (and most do),that driver must register one interrupt handler.

Drivers can register an interrupt handler and enable a given interrupt line for handling with the function `request_irq()`, which is declared in `<linux/interrupt.h>`:

<small>[include/linux/interrupt.h#L117](https://github.com/shichao-an/linux/blob/v2.6.34/include/linux/interrupt.h#L117)</small>

```c
/* request_irq: allocate a given interrupt line */
int request_irq(unsigned int irq,
                irq_handler_t handler,
                unsigned long flags,
                const char *name,
                void *dev)
```

* The first parameter, `irq`, specifies the interrupt number to allocate
    * For some devices (e.g. legacy PC devices such as the system timer or keyboard), this value is typically hard-coded.
    * For most other devices, it is probed or otherwise determined programmatically and dynamically.
* The second parameter, `handler`, is a function pointer to the actual interrupt handler that services this interrupt. This function is invoked whenever the operating system receives the interrupt.

<small>[include/linux/interrupt.h#L80](https://github.com/shichao-an/linux/blob/v2.6.34/include/linux/interrupt.h#L80)</small>
```c
typedef irqreturn_t (*irq_handler_t)(int, void *);
```

Note the specific prototype of the handler function: It takes two parameters and has a return value of `irqreturn_t`.