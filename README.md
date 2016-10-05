# Micro Trace Buffer with GDB
This helper reads the micro trace buffer (MTB) of a Cortex M0+ and translates it to
lines of code.

# Use
There are two aspects to using the Cortex M0+ micro trace buffer. First, you
must change your code to turn on the MTB and provide it with a RAM buffer for
the trace packets. Second, you use GDB to read the current state of the RAM
buffer and translate it into lines of code instead of bare program counters.

All examples are for the Microchip SAMD21. Other Cortex M0+s should work but the
addresses of the MTB registers may differ.

## Code Changes
First, create a global array to store the trace packets.

    // Stores 2 ^ TRACE_BUFFER_MAGNITUDE_PACKETS packets.
    // 7 -> 128 packets
    #define TRACE_BUFFER_MAGNITUDE_PACKETS 7
    // Size in uint32_t. Two per packet.
    #define TRACE_BUFFER_SIZE (1 << (TRACE_BUFFER_MAGNITUDE_PACKETS + 1))
    // Size in bytes. 8 bytes per packet.
    #define TRACE_BUFFER_SIZE_BYTES (TRACE_BUFFER_SIZE << 3)
    __attribute__((__aligned__(TRACE_BUFFER_SIZE_BYTES))) uint32_t mtb[TRACE_BUFFER_SIZE];

Next, you may need to define macros for the MTB registers. Atmel's cmsis does this for us:

    #define REG_MTB_POSITION           (*(uint32_t *)0x41006000U)
    #define REG_MTB_MASTER             (*(uint32_t *)0x41006004U)
    #define REG_MTB_FLOW               (*(uint32_t *)0x41006008U)
    #define REG_MTB_BASE               (*(uint32_t *)0x4100600CU)

Now, configure and turn on the trace buffer by setting the appropriate register values. This will continuously log to `mtb` as a circular buffer.
(Usually done in main().)

    REG_MTB_POSITION = ((uint32_t) (mtb - REG_MTB_BASE)) & 0xFFFFFFF8;
    REG_MTB_FLOW = (((uint32_t) mtb - REG_MTB_BASE) + TRACE_BUFFER_SIZE_BYTES) & 0xFFFFFFF8;
    REG_MTB_MASTER = 0x80000000 + (TRACE_BUFFER_MAGNITUDE_PACKETS - 1);

In cases where you end up in an infinite loop after an error such as a hard fault its important to turn the MTB off before entering the loop. If you don't turn it off, then it will fill up with loop jumps.

    REG_MTB_MASTER = 0x00000000;

## Use from GDB
Now, once the trace is being written to RAM you can read it back with GDB. You can do this manually by printing the `mtb` variable but its more helpful to see the trace history by line of code.

First, source the micro-trace-buffer.py into your gdb. Make sure you are using gdb with python enabled. In the arm toolchain use `arm-none-eabi-gdb-py` instead of `arm-none-eabi-gdb`.

    source micro-trace-buffer.py

Now, during a breakpoint you can run `micro-trace-buffer` or `mtb` for short to get the history from newest to oldest. (If it paginates you can type `q` then `<enter>` to quit early.) Here is an example:

    (gdb) mtb
    0x0001e87e asf/sam0/drivers/usb/stack_interface/usb_device_udd.c 1131 1 times
    0x0001e254 asf/sam0/drivers/usb/stack_interface/usb_device_udd.c 176 1 times
    0x0001e262 asf/sam0/drivers/usb/stack_interface/usb_device_udd.c 177 1 times
    0x0001bca8 ../lib/libc/string0.c 32 1 times
    0x0001bcc2 ../lib/libc/string0.c 60 1 times
    0x0001bcb8 ../lib/libc/string0.c 59 1 times
    0x0001bcc2 ../lib/libc/string0.c 60 1 times
    0x0001bcb8 ../lib/libc/string0.c 59 1 times
    0x0001bcc2 ../lib/libc/string0.c 60 1 times
    0x0001bcb8 ../lib/libc/string0.c 59 1 times
    0x0001bcc2 ../lib/libc/string0.c 60 1 times
    0x0001bcba ../lib/libc/string0.c 59 2 times
    0x0001bcf0 ../lib/libc/string0.c 65 2 times
    0x0001e266 asf/sam0/drivers/usb/stack_interface/usb_device_udd.c 192 1 times
    0x0001e27e asf/common/services/sleepmgr/sleepmgr.h 152 1 times
    0x0001e282 asf/common/services/sleepmgr/sleepmgr.h 160 2 times
    0x0001dec4 asf/common/utils/interrupt/interrupt_sam_nvic.h 152 1 times

The first number is the last pc to occur at that line in the log. The second part is the source file name. The third part is the source line number. The last part is the number of times the line occurs in the log. This is useful to condense tight loops.

Also, beware that this log only includes the pcs around jumps. If you see neighboring entries with the same file then the code between lines may have been run rather than jumped over.
