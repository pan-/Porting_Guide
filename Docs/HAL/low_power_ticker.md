# Low Power Ticker

This document provides a porting guide for mbed-hal version 1.0.0.

The low power ticker API is declared in the [lp_ticker_api header file](https://github.com/ARMmbed/mbed-hal/blob/master/mbed-hal/lp_ticker_api.h).

## Requirements

- A timer, which should be available in most MCU low power modes.
- A 32-bit timer counter (for timers with a lower bit counter, software expansion can be used), it should be counting up.
- An overflow counter (to be able to form a 64-bit timestamp).
- The timer’s minimum resolution should be 1 millisecond.
- sleep HAL implemented (required by ``lp_ticker_sleep_until()``)

The low power timer should be able to provide a wake-up source for MCU sleep that's as low as possible, to reduce the power consumption. It should never stop running.

The most common timer that satisfies all requirements is RTC.

The minimum set of includes for the implementation is:

```c
#include "lp_ticker_api.h"         // lp ticker API
#include "sleep_api.h"             // sleep API
#include "objects.h"               // sleep_t object declaration
#include "uvisor-lib/uvisor-lib.h" // vIRQ_xxx functions if interrupts are used
```

## API

### lp_ticker_init

```c
/** Initialize the low power ticker
 *
 */
void lp_ticker_init(void);
```

This should initialize the resources required for the timer: set up the timer, start the counter, register interrupts. This function should be executed only once.

#### Implementation example

```c
void lp_ticker_init(void)
{
    if (rtc_is_running()) {
        return;
    }
    // initialize rtc
    rtc_init();
    rtc_enable();
    enable_vectors();
}
```

### lp_ticker_read

```c
/** Read the current counter
 *
 * @return The current timer's counter value
 */
uint32_t lp_ticker_read(void);
```

The function returns the current counter value, which should be a 32-bit value. If the timer does not provide a 32-bit counter, it can be extended by software. For instance, if the timer's counter is 24-bit, we can use the lower 8 bits from the overflow counter to form a 32-bit counter.


### lp_ticker_set_interrupt

```c
/** Set interrupt for specified time
 *
 * @param now  The current time
 * @param time The time to be matched
 */
void lp_ticker_set_interrupt(uint32_t now, uint32_t time);
```

The function should set an interrupt for matching the time value.

The *now* argument is intended for calculating the delta between now and time values, which might be useful for some timer implementations. If a timer has a time match interrupt, the *now* argument can be ignored.

#### Implementation example

```c
void lp_ticker_set_interrupt(uint32_t now, uint32_t time)
{
    (void)now;
    timer_set_match_interrupt(time);
}
```


### lp_ticker_get_overflows_counter

```c
/** Get the overflows counter
 *
 * @return The timer's overflow counter
 */
uint32_t lp_ticker_get_overflows_counter(void);
```

The function returns the 32-bit overflow counter. It can be read directly from the timer overflow counter if the timer peripheral provides one, or using a software counter that is incremented when a timer counter overflows.


### lp_ticker_get_compare_match

```c
/** Get compare match register
 *
 * @return The next lp ticker interrupt scheduled time
 */
uint32_t lp_ticker_get_compare_match(void);
```

The function returns the compare match value, which is the next scheduled interrupt for this ticker.


### lp_ticker_sleep_until

```c
/** Set lp ticker interrupt and enter mbed sleep.
 *
 * @param now  The current time
 * @param time The time to be matched
 */
void lp_ticker_sleep_until(uint32_t now, uint32_t time);
```

This function is similar to ``lp_ticker_set_interrupt``. It provides additional sleep functionality. The timer should be set up, then go to sleep for the specified time.

#### Implementation example

```c
void lp_ticker_sleep_until(uint32_t now, uint32_t time)
{
    // set the ticker interrupt
    lp_ticker_set_interrupt(now, time);
    // define sleep_t object required by sleep API
    sleep_t sleep_obj;
    mbed_enter_sleep(&sleep_obj);
    // this might require additional steps before we recover fully from the sleep
    // we don't do any in this example
    mbed_exit_sleep(&sleep_obj);
}
```


## Example

The lp ticker is used by [the MINAR platform implementation minar-platform-mbed](https://github.com/ARMmbed/minar-platform-mbed).

