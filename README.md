# avalon_pwm
Avalon PWM Controller for Qsys

About avalon_pwm
----------------

avalon_pwm is a PWM controller with Avalon bus interface which is intended to
be used with Qsys integration tool and NIOS II CPU. It offers up to 32 synchronous
channels with configurable polarity, optional preload registers and guaranteed
dead time, which makes it suitable for motor control.

Installation
------------

Copy the complete file structure of the repository in a folder inside your Quartus
project. When you start Qsys, a new "Avalon PWM Controller" component should appear
under "Project / Basic fucntions / I/O".

Configuration
-------------

The following parameters can be configured for a PWM controller instance:

- Clock prescaler width (default: 16) - the width of an internal counter which
will be used as a clock divider. The divider value will be programmable at
runtime in the range of 1..(2^N + 1), or 1..65536 with the default width.
The PWM counter will be fed with a clock with this prescaler applied.

- PWM cycle counter width (default: 8) - the width of the PWM cycle conter
which defines the duty cycle resolution. Duty cycle on all channels will be
programmable in steps of 1/2^N, or 1/256 with default PWM counter width.
PWM period will have a fixed value of 2^(N+1) clock cycles of PWM clock
(after the prescaler).

- Number of PWM output pins (default: 4) - the number of PWM channels.
All channels will have the same PWM frequency and phase, but each channel
will have its own programmable duty cycle and polarity.

- Preload registers (default: no) - if enabled, new duty cycle values are
applied twice per PWM period, in the middle of a pulse. This is mostly
useful if you plan to enable interrupts that fire just after, to update
duty cycles in every PWM period.

Operation
---------

During a PWM cycle, the counter goes up and down, hence a period of 
2^(N+1) clock cycles. The current value of the PWM counter is constantly
compared with duty cycles of each channel, and the channel output activates
and desactivates when the counter value goes below or above the duty cycle,
respectively.

For outputs with POLARITY set to 0 (positive), the average output is
proportional to the duty cycle. For POLARITY = 1 (negative), it's the
inverse:

    POLARITY
      ___________           _____________________           _________
    0            |_________|                     |_________|
	             |_________|                     |_________|
	1 ___________|         |_____________________|         |_________
	             |         |                     |         | 
                 |         |                     |         | 
	PWM_CNT      |         |                     |         | 
    7 _ _ _ _ _ _|_ ___    |                     |  ___    |
                 |_|   |_  |                     |_|   |_  |
    5 _ _ _ _ _ _|_ _ _ _|_|_ _ _ _ _ _ _ _ _ _ _|_ _ _ _|_|
              _|           |_                 _|           |_
            _|               |_             _|               |_
          _|                   |_         _|                   |_
        _|                       |_     _|                       |_
    0 _|                           |___|                           |_
	  0 1 2 3 4 5 6 7 8 . . . . . . 15                              31 TICKS

The fact that the PWM cycle counter counts both up and down guarantees that
there is a fixed dead time between PWM channes with different duty cycles.
For instance, below is a sketch of a positive polarity channel with a duty
cycle of 3 together with a negative polarity channel with a duty cycle of 5.
As you can see, there are exactly two PWM ticks between the moment when one of
the signal goes low and the other signal goes high:

    POLARITY
      _______                   _____________                   _____
    0        |_________________|             |_________________|
	         |    _________    |             |    _________    |
	1 _______|___|         |___|_____________|___|         |___|_____
	         |   |         |   |             |   |         |   |
             |   |         |   |             |   |         |   |
	PWM_CNT  |   |         |   |             |   |         |   |
    7 _ _ _ _|_ _|_ ___    |   |             |   |  ___    |   |
             |   |_|   |_  |   |             |   |_|   |_  |   |
    5 _ _ _ _|_ _|_ _ _ _|_|_ _|_ _ _ _ _ _ _|_ _|_ _ _ _|_|   |
             |_|           |_  |             |_|           |_  |
    3 _ _ _ _|_ _ _ _ _ _ _ _|_|_ _ _ _ _ _ _|_ _ _ _ _ _ _ _|_|
          _|                   |_         _|                   |_
        _|                       |_     _|                       |_
    0 _|                           |___|                           |_
	  0 1 2 3 4 5 6 7 8 . . . . . . 15                              31 TICKS

In order to make sure that dead time is not violated while the duty cycle
values are being updated, two strategies are possible:

- for precise timing control, use preload registers and interrupts

- otherwise, don't use preload registers and update the duty cycles on related
channels in a way that preserves dead time: if you need to reduce the duty cycle,
update it on positive polarity channel first, and do the other way around if you
need to increase it.

The control register contails 4 bits which affect all PWM channels simultaneously:

- OUT_ENA (bit 0) - "1" (default) enables the output signal on all PWM pins.
"0" sets all PWM pins to its polarity level. This is the equivalent of setting
all duty cycles to 0.

- CNT_ENA (bit 1) - "1" (default) enables the PWM cycle counter. "0" stops the
PWM cycle counter and holds the last state of PWM pins.

- IRQL_ENA (bit 2) - "1" enables an IRQ on PWM cycle counter reaching 0.
"0" (default) disables the interrupt.

- IRQH_ENA (bit 3) -"1" enables an IRQ on PWM cycle counter reaching the max value.
"0" (default) disables the interrupt.
