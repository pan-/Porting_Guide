# Sleep

This document provides a porting guide for mbed-hal version 1.0.0.

The sleep API is declared in the [sleep_api header file](https://github.com/ARMmbed/mbed-hal/blob/master/mbed-hal/sleep_api.h).

## Requirements
- Declare a `` sleep_s `` target-specific structure. It must contain data for restoring the MCU back to its pre-sleep state.

The sleep functions should enable the lowest MCU low power mode. There are currently limitations that might have implications on this; please see the [limitations section](#limitations).

The minimum set of includes for the implementation is:

```c
#include "sleep_api.h" // sleep API declarations
#include "objects.h"   // sleep_s declaration
```

## API

### void mbed_enter_sleep(sleep_t *obj)

```c
/** Enter the sleep mode
 *
 * @param obj The sleep object that stores required data to restore from sleep
 */
void mbed_enter_sleep(sleep_t *obj);
```

This is the entry function for sleep. It should prepare the MCU for entering the low power mode, store any target-specific data to the ``sleep_t`` object and wait for interrupt (note: in some cases it might be required to wait for an event).

#### Implementation example

```c
void mbed_enter_sleep(sleep_t *obj)
{
    // store some data which should be restored once MCU is awake
    obj->pll_enabled = REG_PLL_RUNNING_FLAG;
    // enable MCU low power mode
    enable_mcu_low_power_mode();
    // wait for interrupt
    __WFI();
    // return once awake
}
```


### void mbed_exit_sleep(sleep_t *obj)

```c
/** Restore the MCU from the specified sleep mode
 *
 * @param obj The sleep object that stores data to restore from sleep
 */
void mbed_exit_sleep(sleep_t *obj);

```

The exit sleep function recovers the MCU from the low power mode it entered using `` mbed_enter_sleep ``.

#### Implementation example

```c
mbed_exit_sleep(sleep_t *obj)
{
    // restore the pre-sleep state
    if (obj->pll_enabled) {
        restore_pll();
    }
    // MCU should be in the state it was in before entering sleep
}
```


## Limitations

There is no common API to track what is active in mbed at the moment. As a result, some MCUs currently enter the least restrictive sleep mode so as not to affect the functionality of peripherals.

## Example

An example of using sleep, implementing a timer with sleep functionality:

```C
void timer_sleep_until(timestamp_t time)
{
    sleep_t obj;
    // set a timer to wake up the device
    timer_set_interrupt(time);
    mbed_sleep_enter_sleep(&obj);
    // we are awake, we can do extra things prior to recovering from sleep
    recover_timer();
    mbed_sleep_exit_sleep(&obj);
}
```
