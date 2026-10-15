# Protothreads

Protothreads are small, stackless threads for C, designed by Adam Dunkels for
memory-constrained embedded systems. They let you write code that has to wait
(for a timer, a button, a byte on a serial line) as an ordinary sequence of
statements instead of a hand-written state machine. A protothread has no stack
of its own. All it keeps between runs is one two-byte number, and the library
is a handful of header files full of preprocessor macros, so it works with any
C compiler on anything from an 8-bit microcontroller upwards.

This repository holds version 1.4 of the library with three small fixes. The
first part of this README is a guide to using it. The second part,
[How it works](#how-it-works), shows what the macros turn into. You don't need
that part to get started, but the [rules](#rules-to-remember) you have to follow
only really make sense once you've seen it.

## Contents

- [Adding it to your project](#adding-it-to-your-project)
- [A first protothread](#a-first-protothread)
- [Running protothreads](#running-protothreads)
- [Rules to remember](#rules-to-remember)
- [API reference](#api-reference)
- [Examples](#examples)
- [How it works](#how-it-works)
- [Files in this repository](#files-in-this-repository)
- [Credits and further reading](#credits-and-further-reading)
- [License](#license)

## Adding it to your project

There is nothing to build. Put `src/` on the compiler's include path and include
the header:

```c
#include "pt.h"       /* protothreads */
#include "pt-sem.h"   /* counting semaphores, only if you need them */
```

If you keep the library as a git submodule:

```sh
git submodule add https://github.com/AhmedKishki/Protothreads.git Protothreads
```

```make
CFLAGS += -IProtothreads/src
```

By default the library uses an implementation built on the C `switch`
statement, which works with every C compiler. GCC and Clang users can pick a
second one; see [The GCC implementation](#the-gcc-implementation).

## A first protothread

Say you want to blink an LED every half second and, at the same time, react to
a button. Without threads you end up with a main loop full of timestamps and
flags that remember what each job was in the middle of. With protothreads each
job is written as if it had the CPU to itself:

```c
#include <stdint.h>
#include "pt.h"

/* Your platform provides this: a millisecond counter driven by a timer
   interrupt. */
uint32_t clock_ms(void);

/* A small timer helper. The other examples in this README use it too.
   The unsigned subtraction keeps working when the counter wraps around. */
struct timer {
    uint32_t start;
    uint32_t interval;
};

static void timer_set(struct timer *t, uint32_t interval)
{
    t->start = clock_ms();
    t->interval = interval;
}

static int timer_expired(const struct timer *t)
{
    return clock_ms() - t->start >= t->interval;
}

static struct pt blink_pt, button_pt;

static PT_THREAD(blink(struct pt *pt))
{
    static struct timer timer;

    PT_BEGIN(pt);

    while (1) {
        led_toggle();
        timer_set(&timer, 500);
        PT_WAIT_UNTIL(pt, timer_expired(&timer));
    }

    PT_END(pt);
}

static PT_THREAD(button(struct pt *pt))
{
    static struct timer timer;

    PT_BEGIN(pt);

    while (1) {
        PT_WAIT_UNTIL(pt, button_down());

        /* Give the contacts 20 ms to settle, then look again. */
        timer_set(&timer, 20);
        PT_WAIT_UNTIL(pt, timer_expired(&timer));

        if (button_down()) {
            on_button_press();
            PT_WAIT_WHILE(pt, button_down());
        }
    }

    PT_END(pt);
}

int main(void)
{
    hardware_init();

    PT_INIT(&blink_pt);
    PT_INIT(&button_pt);

    while (1) {
        blink(&blink_pt);
        button(&button_pt);
    }
}
```

`clock_ms()`, `led_toggle()`, `button_down()`, `on_button_press()` and
`hardware_init()` stand in for your own code.

Read `blink` from top to bottom and it does what it says: toggle the LED, start
a timer, wait for it to run out, repeat. Here are the pieces:

- `struct pt` holds a protothread's state. Each protothread needs one, and it
  has to live as long as the protothread does, so make it global, `static`, or
  a field in a struct that lives long enough.
- `PT_INIT()` resets that state, so that the next call starts at the top. Call
  it once before the first call.
- `PT_THREAD(blink(struct pt *pt))` declares the function. It expands to
  `char blink(struct pt *pt)`; the `char` it returns tells the caller whether
  the thread is still running.
- `PT_BEGIN()` and `PT_END()` mark the start and end of the thread's body.
  Everything that waits goes between them.
- `PT_WAIT_UNTIL(pt, condition)` is where the thread waits. If the condition
  is false, the function returns right there. On the next call it continues at
  that same line and checks the condition again. Once it's true, the thread
  carries on with the next statement. If it's already true the first time, the
  thread doesn't stop at all.
- `PT_WAIT_WHILE(pt, condition)` is the same thing the other way round: it
  waits for as long as the condition is true.

The `while (1)` loop in `main` is the scheduler. It calls each protothread in
turn, and each call runs until the thread reaches a wait whose condition is
false. That call returns, and the loop moves on to the next thread. The threads
take turns because each one gives up the CPU at its waits; nothing forces them
to. This is called cooperative multitasking.

Both timers are declared `static`. That isn't a matter of style: a normal local
variable is lost every time the thread waits.
[Rules to remember](#rules-to-remember) covers this and the other rules that
come from how protothreads work.

## Running protothreads

### Who calls a protothread

The library has no scheduler of its own. A protothread runs when something
calls its function, and that can be:

- a main loop that calls every thread in turn, as above;
- an event handler that calls the thread whenever something it might care about
  happens. The thread re-checks its condition, so it does no harm if the event
  turns out to be the wrong one;
- an interrupt handler. A call costs only a few CPU cycles more than a
  hand-written state machine would, so this is practical even in a receive
  interrupt that runs for every byte.

To the caller, a protothread is just a function. It doesn't need to know
anything about protothreads to call one.

### What a call returns

Each call returns one of four values:

| Value | Meaning |
|---|---|
| `PT_WAITING` (0) | Stopped at a wait whose condition was false. |
| `PT_YIELDED` (1) | Stopped at `PT_YIELD()` or `PT_YIELD_UNTIL()`. |
| `PT_EXITED` (2) | Left through `PT_EXIT()`. |
| `PT_ENDED` (3) | Reached `PT_END()`. |

You'll rarely compare against these directly. `PT_SCHEDULE()` calls the thread
and turns the result into yes or no: true while the thread is still running
(waiting or yielded), false once it has exited or ended.

```c
if (!PT_SCHEDULE(transfer(&transfer_pt))) {
    /* The transfer protothread has finished. */
}
```

### When a protothread finishes

When a protothread reaches `PT_END()` or `PT_EXIT()`, its `struct pt` is reset.
Call it again and it starts over from the top. If a job should run only once,
stop calling it once it has finished.

### Sharing data with interrupts

Protothreads never interrupt each other: a thread only gives up the CPU at a
wait. Data shared between protothreads therefore needs no locking, as long as
the thread doesn't wait in the middle of updating it.

Interrupt handlers are another matter. They can run between any two machine
instructions. A variable that an interrupt writes and a protothread reads must
be `volatile`, or the compiler is free to assume it never changes. And if the
CPU can't read it in one go, like a 32-bit millisecond counter on an 8-bit AVR,
read it with interrupts briefly disabled so you don't get half an old value and
half a new one.

## Rules to remember

The macros make a protothread look like a thread, but underneath, the function
returns at every wait and gets called again later. These rules follow from
that. [Where the rules come from](#where-the-rules-come-from) explains each
one.

1. Local variables don't survive a wait. Make them `static`, or keep them in a
   struct that you pass to the thread.
2. Only the protothread function itself can wait. A function it calls can't
   wait on its behalf. Make that function a protothread too and run it with
   `PT_SPAWN()`.
3. Don't put a wait inside a `switch` statement (with the default
   implementation). Use `if`/`else` there. The compiler won't warn you.
4. Put at most one wait on each source line.
5. A protothread that doesn't wait never lets anything else run. Put a
   `PT_YIELD()` inside long loops.
6. Code above `PT_BEGIN()` runs on every call.

"Wait" here means anything that can stop the thread: `PT_WAIT_UNTIL`,
`PT_WAIT_WHILE`, `PT_YIELD`, `PT_YIELD_UNTIL`, `PT_WAIT_THREAD`, `PT_SPAWN`
and `PT_SEM_WAIT`.

## API reference

### Declaring a protothread

| Macro | What it does |
|---|---|
| `struct pt` | A protothread's state. One per protothread; it must outlive the thread. |
| `PT_INIT(pt)` | Resets the state so the next call starts at the top. Call it before the first call. |
| `PT_THREAD(name(args))` | Declares the protothread function. Expands to `char name(args)`. |
| `PT_BEGIN(pt)` | Starts the body. Must come before any wait. |
| `PT_END(pt)` | Ends the body. Resets the state and returns `PT_ENDED`. |

### Waiting

| Macro | What it does |
|---|---|
| `PT_WAIT_UNTIL(pt, cond)` | Waits until `cond` is true. Doesn't stop at all if it already is. |
| `PT_WAIT_WHILE(pt, cond)` | Waits as long as `cond` is true. Same as `PT_WAIT_UNTIL(pt, !(cond))`. |
| `PT_YIELD(pt)` | Stops once, unconditionally. The next call carries on after it. |
| `PT_YIELD_UNTIL(pt, cond)` | Stops at least once, then carries on at the first later call where `cond` is true. |

### Child protothreads

| Macro | What it does |
|---|---|
| `PT_SPAWN(pt, child, call)` | Resets the child's state, then waits until the child has finished. `child` is the child's `struct pt *`, `call` is the call to the child function. |
| `PT_WAIT_THREAD(pt, call)` | Waits until the child has finished, without resetting it first. Call `PT_INIT()` on the child yourself. |

The child is called once every time the parent is called, so it runs at the
parent's pace.

### Leaving and calling

| Macro | What it does |
|---|---|
| `PT_EXIT(pt)` | Leaves the thread now. Resets the state and returns `PT_EXITED`. |
| `PT_RESTART(pt)` | Resets the state and returns `PT_WAITING`. The next call starts at the top. |
| `PT_SCHEDULE(call)` | Makes the call and is true while the thread is still running. |

### Semaphores (`pt-sem.h`)

| Macro | What it does |
|---|---|
| `struct pt_sem` | A counting semaphore: just an `unsigned int` counter. |
| `PT_SEM_INIT(s, count)` | Sets the counter. |
| `PT_SEM_WAIT(pt, s)` | Waits until the counter is above zero, then decrements it. |
| `PT_SEM_SIGNAL(pt, s)` | Increments the counter. Never waits. |

## Examples

These come from the examples on the
[protothreads website](https://dunkels.com/adam/pt/examples.html) and the
paper, adapted to use the timer helper from
[A first protothread](#a-first-protothread). Functions like `send_packet()`
are placeholders for your own code.

### Retransmitting until acknowledged

A sender that keeps sending a packet until the receiver acknowledges it, and
the matching receiver:

```c
#include "pt.h"

static struct timer timer;

PT_THREAD(sender(struct pt *pt))
{
    PT_BEGIN(pt);

    do {
        send_packet();
        /* Wait for an acknowledgement or for the timer to run out,
           whichever comes first. If the timer ran out, send again. */
        timer_set(&timer, TIMEOUT);
        PT_WAIT_UNTIL(pt, acknowledgment_received() ||
                          timer_expired(&timer));
    } while (timer_expired(&timer));

    PT_END(pt);
}

PT_THREAD(receiver(struct pt *pt))
{
    PT_BEGIN(pt);

    /* Wait for a packet, then acknowledge it. */
    PT_WAIT_UNTIL(pt, packet_received());
    send_acknowledgement();

    PT_END(pt);
}
```

The condition can be any C expression, so "wait for this or a timeout" is one
line. The wait also sits inside a `do`/`while` loop;
[How it works](#how-it-works) shows why that's allowed.

### Scrolling text on an LCD

Text that doesn't fit on a one-line display scrolls in from the right, one step
per second:

```c
#include <string.h>
#include "pt.h"

struct state {
    char *text;
    char *scrollptr;
    struct pt pt;
    struct timer timer;
};

PT_THREAD(display_text(struct state *s))
{
    PT_BEGIN(&s->pt);

    if (strlen(s->text) <= LCD_SIZE) {
        /* Short enough: show it right away. */
        lcd_display_text(s->text);
    } else {
        /* Too long: move the start of the visible text one character
           to the right each second. */
        for (s->scrollptr = s->text;
             strlen(s->scrollptr) > LCD_SIZE;
             ++s->scrollptr) {
            lcd_display_text(s->scrollptr);

            timer_set(&s->timer, ONE_SECOND);
            PT_WAIT_UNTIL(&s->pt, timer_expired(&s->timer));
        }
    }

    PT_END(&s->pt);
}
```

Nothing here is `static`. The loop variable, the timer and the `struct pt`
itself all live in `struct state`, which the caller owns. Give each display its
own `struct state` and the same function drives several of them at once. The
function doesn't have to take a `struct pt *` either; it just has to be able to
reach one.

### A code lock

The kind of keypad next to a door: type 1-4-2-3 to open it. Each key has to
come within a second of the previous one, and once the code is complete, no key
may be pressed for half a second before the lock opens.

```c
#include <stdio.h>
#include "pt.h"

static const char code[4] = {'1', '4', '2', '3'};
static struct timer codelock_timer;

/* key_pressed() returns true when a new key has arrived and stores
   it in key. */

static PT_THREAD(codelock_thread(struct pt *pt))
{
    static unsigned int keys;

    PT_BEGIN(pt);

    while (1) {
        /* Read keys until the whole code has been entered, or until
           a wrong key or a timeout sends us back to the start. */
        for (keys = 0; keys < sizeof(code); ++keys) {
            if (keys == 0) {
                /* Wait as long as it takes for the first key. */
                PT_WAIT_UNTIL(pt, key_pressed());
            } else {
                /* After that, each key has one second. */
                timer_set(&codelock_timer, 1000);
                PT_WAIT_UNTIL(pt, key_pressed() || timer_expired(&codelock_timer));

                if (timer_expired(&codelock_timer)) {
                    printf("Code lock timer expired.\n");
                    break;
                }
            }

            if (key != code[keys]) {
                printf("Incorrect key '%c' found\n", key);
                break;
            }
            printf("Correct key '%c' found\n", key);
        }

        if (keys == sizeof(code)) {
            /* Right code. Make sure it wasn't a lucky guess in the middle
               of someone hammering the keys: no key for 500 ms. */
            timer_set(&codelock_timer, 500);
            PT_WAIT_UNTIL(pt, key_pressed() || timer_expired(&codelock_timer));

            if (!timer_expired(&codelock_timer)) {
                printf("Key pressed during final wait, code lock locked again.\n");
            } else {
                printf("Code lock unlocked.\n");
                PT_EXIT(pt);
            }
        }
    }

    PT_END(pt);
}
```

The waits sit inside a `for` loop inside a `while` loop, and `break` jumps out
of the `for` loop across them. Try writing the same thing as a state machine and
count the states: waiting for the first key, waiting for each further key,
waiting out the final half second, and the transitions between them.

### Flashing an error code

A status LED that flashes once per error, pauses for two seconds and starts
again. Flashing *n* times is a job with waits in it, so it gets its own
protothread, and the parent runs it with `PT_SPAWN()`:

```c
#include <stdint.h>
#include "pt.h"

static PT_THREAD(flash(struct pt *pt, uint8_t times))
{
    static uint8_t i;
    static struct timer timer;

    PT_BEGIN(pt);

    for (i = 0; i < times; i++) {
        led_on();
        timer_set(&timer, 200);
        PT_WAIT_UNTIL(pt, timer_expired(&timer));

        led_off();
        timer_set(&timer, 200);
        PT_WAIT_UNTIL(pt, timer_expired(&timer));
    }

    PT_END(pt);
}

static PT_THREAD(status(struct pt *pt))
{
    static struct pt child;
    static struct timer pause;

    PT_BEGIN(pt);

    while (1) {
        PT_SPAWN(pt, &child, flash(&child, error_count()));

        timer_set(&pause, 2000);
        PT_WAIT_UNTIL(pt, timer_expired(&pause));
    }

    PT_END(pt);
}
```

The child's `struct pt` is `static` like any other state that has to survive a
wait. One thing that may surprise you: `error_count()` is called again every
time the parent runs, not just once when the spawn starts.
[How child protothreads work](#how-child-protothreads-work) explains why.

### Producer and consumer with semaphores

The classic bounded buffer: a producer puts items into a buffer with room for
eight, a consumer takes them out, and each has to wait when the buffer is full
or empty. A third protothread starts both and waits until both are done.

```c
#include "pt-sem.h"

#define NUM_ITEMS 32
#define BUFSIZE 8

static struct pt_sem full, empty;

static PT_THREAD(producer(struct pt *pt))
{
    static int produced;

    PT_BEGIN(pt);

    for (produced = 0; produced < NUM_ITEMS; ++produced) {
        PT_SEM_WAIT(pt, &full);
        add_to_buffer(produce_item());
        PT_SEM_SIGNAL(pt, &empty);
    }

    PT_END(pt);
}

static PT_THREAD(consumer(struct pt *pt))
{
    static int consumed;

    PT_BEGIN(pt);

    for (consumed = 0; consumed < NUM_ITEMS; ++consumed) {
        PT_SEM_WAIT(pt, &empty);
        consume_item(get_from_buffer());
        PT_SEM_SIGNAL(pt, &full);
    }

    PT_END(pt);
}

static PT_THREAD(driver_thread(struct pt *pt))
{
    static struct pt pt_producer, pt_consumer;

    PT_BEGIN(pt);

    PT_SEM_INIT(&empty, 0);
    PT_SEM_INIT(&full, BUFSIZE);

    PT_INIT(&pt_producer);
    PT_INIT(&pt_consumer);

    PT_WAIT_THREAD(pt, producer(&pt_producer) & consumer(&pt_consumer));

    PT_END(pt);
}
```

The semaphore names describe when you'd wait on them. `full` counts the free
slots, so the producer blocks on it when the buffer is full; it starts at
`BUFSIZE`. `empty` counts the items, so the consumer blocks on it when the
buffer is empty; it starts at 0.

`add_to_buffer()` and `get_from_buffer()` need no lock. A protothread is never
interrupted by another protothread, only at its own waits, and there are none
inside those calls.

The driver waits on both children at once by joining the calls with `&`, not
`&&`. That trick has a catch, covered in
[How child protothreads work](#how-child-protothreads-work).

## How it works

### Threads, events and the stack

A normal thread, the kind an RTOS gives you, has its own stack. When the
scheduler switches away from a thread, it saves the CPU registers and the stack
pointer, and the thread's stack stays exactly as it was: return addresses, local
variables, all of it. That's what lets a thread stop anywhere, even deep inside
a function it called. The price is memory. Every thread needs a stack big
enough for its worst moment, reserved whether the thread is running or not, and
because that worst case is hard to predict, stacks tend to be oversized. The
paper's example: a 200-byte stack on an MSP430F149 takes almost 10% of the
chip's RAM.

Event-driven code works the other way round. Each event handler runs from start
to finish and returns, so all the handlers share one stack, and it only has to
be as deep as the deepest handler. But a handler can't stop halfway and wait.
Anything that takes several steps with waits in between has to be split across
several calls and tied together with a state variable: a state machine.

Protothreads sit in between. A protothread is an event handler that can wait.
It runs on the shared stack like a handler, and when it waits it simply returns.
Its stack frame is gone after that, and nothing is kept except a note of where
to continue. On the next call it jumps back to that point.

```text
Threads: one stack each, each sized for its own worst case

  [ thread 1 stack ][ thread 2 stack ][ thread 3 stack ]
  stack RAM = the sum of all three

Protothreads and event handlers: one shared stack

  [ shared stack ]
  stack RAM = the largest of the three
```

### A state machine you don't have to write

The paper motivates protothreads with the radio sleep cycle of a made-up
power-saving radio protocol. The specification is six steps:

1. Turn the radio on.
2. Wait until *t*₀ + *t*_awake.
3. Turn the radio off, but only if all communication has finished.
4. If it hasn't, wait until it has, or until *t*₀ + *t*_awake + *t*_wait_max.
5. Turn the radio off. Wait until *t*₀ + *t*_awake + *t*_sleep.
6. Repeat from step 1.

Written event-driven, as a function the system calls whenever a timer expires
or communication finishes:

```c
static enum { ON, WAITING, OFF } state;
static struct timer timer, wait_timer;

void radio_wake_eventhandler(void)
{
    if (state == ON) {
        if (timer_expired(&timer)) {
            timer_set(&timer, T_SLEEP);
            if (!communication_complete()) {
                state = WAITING;
                timer_set(&wait_timer, T_WAIT_MAX);
            } else {
                radio_off();
                state = OFF;
            }
        }
    } else if (state == WAITING) {
        if (communication_complete() || timer_expired(&wait_timer)) {
            state = OFF;
            radio_off();
        }
    } else if (state == OFF) {
        if (timer_expired(&timer)) {
            radio_on();
            state = ON;
            timer_set(&timer, T_AWAKE);
        }
    }
}
```

And as a protothread, called at the same moments:

```c
static PT_THREAD(radio_wake_thread(struct pt *pt))
{
    PT_BEGIN(pt);

    while (1) {
        radio_on();
        timer_set(&timer, T_AWAKE);
        PT_WAIT_UNTIL(pt, timer_expired(&timer));

        timer_set(&timer, T_SLEEP);
        if (!communication_complete()) {
            timer_set(&wait_timer, T_WAIT_MAX);
            PT_WAIT_UNTIL(pt, communication_complete() || timer_expired(&wait_timer));
        }

        radio_off();
        PT_WAIT_UNTIL(pt, timer_expired(&timer));
    }

    PT_END(pt);
}
```

The protothread reads like the specification. The state machine spreads the
same six steps over three branches, and the order in which they happen is
something you have to work out by reading it. In the paper's measurements,
rewriting real programs this way removed most of their state machines outright
and cut the code by about a third.

Behind the scenes the protothread is still a state machine, as the next sections
show. The difference is that the preprocessor writes it for you.

### Local continuations

The mechanism underneath is what the paper calls a *local continuation*. It has
two operations: *set* records the current position inside a function, and
*resume* jumps back to the recorded position. It is "local" because it only
covers a single function. Unlike a thread switch it doesn't save the stack, and
the implementations in this library don't save local variables either. Only the
position.

A protothread is one function plus one local continuation, which is what
`struct pt` holds. Every wait sets the continuation and then returns if its
condition is false. `PT_BEGIN()` resumes it.

`lc.h` defines the interface as four macros, `LC_INIT`, `LC_SET`, `LC_RESUME`
and `LC_END`, and includes one of two implementations: `lc-switch.h` by
default, or whichever header `LC_INCLUDE` names.

### The switch implementation

Here is all of `lc-switch.h`:

```c
typedef unsigned short lc_t;

#define LC_INIT(s)   s = 0
#define LC_RESUME(s) switch(s) { case 0:
#define LC_SET(s)    s = __LINE__; case __LINE__:
#define LC_END(s)    }
```

The position is a source line number. `__LINE__` is a standard preprocessor
macro that turns into the number of the line it appears on. `LC_SET` stores
that number and puts a `case` label with the same number right after it.
`LC_RESUME` opens a `switch` on the stored number and leaves it open, and
`LC_END` closes it.

`pt.h` builds the protothread macros on top. These three are the ones that
matter here:

```c
#define PT_BEGIN(pt) { char PT_YIELD_FLAG = 1; (void)PT_YIELD_FLAG; LC_RESUME((pt)->lc)

#define PT_WAIT_UNTIL(pt, condition)  \
  do {                                \
    LC_SET((pt)->lc);                 \
    if(!(condition)) {                \
      return PT_WAITING;              \
    }                                 \
  } while(0)

#define PT_END(pt) LC_END((pt)->lc); PT_YIELD_FLAG = 0; \
                   PT_INIT(pt); return PT_ENDED; }
```

Take this protothread, where the wait is on line 10 of its file:

```c
static int counter;

PT_THREAD(example(struct pt *pt))
{
    PT_BEGIN(pt);

    while (1) {
        PT_WAIT_UNTIL(pt, counter == 1000);   /* line 10 */
        report();
        counter = 0;
    }

    PT_END(pt);
}
```

Run it through the preprocessor (`gcc -E`), tidy up the layout, and this is
what the compiler actually gets:

```c
char example(struct pt *pt)
{
    {
        char PT_YIELD_FLAG = 1;
        (void)PT_YIELD_FLAG;
        switch (pt->lc) {
        case 0:
            while (1) {
                do {
                    pt->lc = 10;
        case 10:
                    if (!(counter == 1000)) {
                        return 0;               /* PT_WAITING */
                    }
                } while (0);
                report();
                counter = 0;
            }
        }
        PT_YIELD_FLAG = 0;
        pt->lc = 0;
        return 3;                               /* PT_ENDED */
    }
}
```

Follow a few calls through it.

**First call.** `PT_INIT()` has set `pt->lc` to 0, so the switch jumps to
`case 0`, the top of the body. The loop starts, `pt->lc = 10` records the
position, and the condition is checked. Say `counter` is 3: the function
returns 0, `PT_WAITING`.

**Every call after that.** `pt->lc` is 10, so the switch jumps straight to
`case 10`, in the middle of the `while` loop and the `do`/`while (0)`. Nothing
above it runs again. The condition is checked, it's still false, and the
function returns.

**The call where `counter` has reached 1000.** Same jump, but now the condition
is true, so execution drops out of the `do`/`while (0)`, calls `report()`,
resets the counter, goes round the loop, stores 10 again and checks the
condition. `counter` is now 0, so it returns.

Jumping into the middle of a loop looks like it shouldn't compile, but it's
legal C. A `case` label is just a label, and C allows it anywhere inside the
body of its switch, including inside loops and blocks nested there. Tom Duff
used this to unroll a copy loop (the famous "Duff's device"), and Simon Tatham
used it to build coroutines in C, which is where protothreads borrowed it from.

Look at the expanded code once more and you'll find the state machine: a
`switch` over a state variable, with one `case` for every place the thread can
stop. The states are line numbers, and they're stored in an `unsigned short`,
which is where the two bytes per protothread come from.

### How PT_YIELD works

`PT_WAIT_UNTIL()` doesn't stop if its condition is already true. Sometimes you
want to stop anyway, just to let the other threads have a turn, and that's what
`PT_YIELD()` is for. It needs one more trick: the thread has to tell "arriving
here for the first time" apart from "coming back here on the next call".

```c
#define PT_YIELD(pt)          \
  do {                        \
    PT_YIELD_FLAG = 0;        \
    LC_SET((pt)->lc);         \
    if(PT_YIELD_FLAG == 0) {  \
      return PT_YIELDED;      \
    }                         \
  } while(0)
```

With the yield on line 7, it expands to:

```c
    step_one();
    do {
        PT_YIELD_FLAG = 0;
        pt->lc = 7;
    case 7:
        if (PT_YIELD_FLAG == 0) {
            return 1;               /* PT_YIELDED */
        }
    } while (0);
    step_two();
```

`PT_YIELD_FLAG` is the variable `PT_BEGIN()` declares: an ordinary local, set
to 1 at the start of every call. A thread that reaches the yield in the normal
way clears the flag, records the position, finds the flag at 0 and returns. On
the next call `PT_BEGIN()` sets the flag to 1 again, and the switch jumps to
`case 7`, past the line that cleared it. The flag is still 1, so the thread
goes on to `step_two()`.

`PT_YIELD_UNTIL(pt, cond)` is the same with a condition added to the test, so
it always returns at least once and then keeps returning until `cond` is true.

### How child protothreads work

```c
#define PT_SCHEDULE(f)              ((f) < PT_EXITED)

#define PT_WAIT_THREAD(pt, thread)  PT_WAIT_WHILE((pt), PT_SCHEDULE(thread))

#define PT_SPAWN(pt, child, thread) \
  do {                              \
    PT_INIT((child));               \
    PT_WAIT_THREAD((pt), (thread)); \
  } while(0)
```

`PT_WAIT_THREAD()` is an ordinary wait whose condition happens to be a call to
the child. Every time the parent is resumed at that line, the condition is
evaluated, which calls the child, which runs until its own next wait. As long
as the child returns `PT_WAITING` or `PT_YIELDED` (both less than
`PT_EXITED`), the parent returns too. Once the child has exited or ended, the
parent moves on.

`PT_SPAWN()` puts a `PT_INIT()` of the child in front. The wait's `case` label
comes after that `PT_INIT()`, so the child is reset only when the parent first
arrives at the spawn. On later calls the jump lands past it.

Because the child's call sits inside the condition, everything in it is
evaluated again on each call, arguments included. That's why `error_count()` in
the [error code example](#flashing-an-error-code) is read every time the
parent runs.

#### Waiting for two children at once

The bounded buffer example waits like this:

```c
PT_WAIT_THREAD(pt, producer(&pt_producer) & consumer(&pt_consumer));
```

This depends on the bit patterns of the return values:

| Value | Binary | Thread is |
|---|---|---|
| `PT_WAITING` | `00` | running |
| `PT_YIELDED` | `01` | running |
| `PT_EXITED` | `10` | finished |
| `PT_ENDED` | `11` | finished |

Both "finished" values have the high bit set, and `a & b` keeps that bit only
if both `a` and `b` have it. `PT_SCHEDULE()` checks for a value below 2, that
is, the high bit clear. So `PT_SCHEDULE(a & b)` stays true for as long as at
least one of the two is still running. `&&` wouldn't work: it stops evaluating
as soon as the left side is 0, which is `PT_WAITING`, so the right-hand child
would never be called while the left one waits.

The catch is that `&` also keeps calling a child that has already finished, and
a finished protothread starts over when it's called again. So both children have
to finish on the same pass, or the one that finished first starts its work a
second time. In the bounded buffer example they do finish together, but only
if the producer is called first, and C doesn't say which side of `&` gets
evaluated first. If you can't guarantee that the children finish together, keep
a "done" flag for each one and only call those that haven't finished.

### Ending, exiting and restarting

```c
#define PT_EXIT(pt)    do { PT_INIT(pt); return PT_EXITED;  } while(0)
#define PT_RESTART(pt) do { PT_INIT(pt); return PT_WAITING; } while(0)
```

`PT_END()` closes the switch and then does the same: `PT_INIT(pt); return
PT_ENDED;`. All three reset the continuation to 0 before returning, so the next
call lands on `case 0` and starts the thread from the top. They differ only in
what they return. `PT_RESTART()` returns `PT_WAITING`, so a parent waiting on
the thread keeps waiting. `PT_EXIT()` and `PT_END()` return values that count
as finished.

The hidden switch has one more side effect. A `break` that isn't inside one of
your own loops or switches leaves the hidden switch and lands on the code
`PT_END()` placed after it, so the thread ends. With the GCC implementation the
same `break` is a compile error.

### Where the rules come from

**Local variables don't survive a wait.** When a protothread waits, its
function returns and its stack frame is given up. When it's resumed, the jump
skips every declaration and assignment above the wait, so a local variable
holds whatever is left in that patch of stack:

```c
PT_THREAD(broken(struct pt *pt))
{
    int i;                      /* lives on the stack */

    PT_BEGIN(pt);

    for (i = 0; i < 10; i++) {
        PT_WAIT_UNTIL(pt, ready());
        use(i);                 /* garbage once the thread has been resumed */
    }

    PT_END(pt);
}
```

GCC catches this with `-Wall` ("'i' may be used uninitialized"), but only when
optimising (`-O1` and above). At `-O0` it says nothing.

There are two ways out. Declare the variable `static`, which moves it off the
stack into ordinary RAM. That's fine when only one instance of the thread runs,
which is the usual case. If the same function runs as several threads at once,
keep the variables in a struct next to the `struct pt` and pass that in, as the
[LCD example](#scrolling-text-on-an-lcd) does, so each instance has its own
copy.

**Only the protothread function can wait.** The `case` label of a wait has to
be inside the switch that `PT_BEGIN()` opened, and that switch is in the
protothread's function. Put a wait in a helper function and the switch
implementation refuses it: "case label not within a switch statement". The GCC
implementation compiles it if the helper's return type fits, and later jumps
from the protothread straight into the helper's code, which is undefined
behaviour. Turn the helper into a protothread and run it with `PT_SPAWN()`.

**No waits inside a `switch`.** A `case` label belongs to the innermost switch
around it. Put a wait inside your own switch and its `case __LINE__` becomes a
case of your switch, not of the hidden one. It compiles without a word. On the
next call the hidden switch looks for that line number, finds no `case` for it,
skips its entire body and lands at `PT_END()`: the thread quietly ends. A
switch with no waits inside it is fine, and the GCC implementation doesn't have
this problem at all.

**One wait per line.** Two waits on the same line store the same number, so
the compiler sees two identical `case` labels and stops with "duplicate case
value" ("duplicate label" with the GCC implementation). A wait whose condition
spans several lines is fine: it's one macro, so it gets one number.

**A thread that doesn't wait never lets go.** Nothing can take the CPU away
from a protothread. It runs until it returns, and it only returns at a wait or
at the end.

**Code above `PT_BEGIN()` runs on every call.** It comes before the hidden
switch, so it runs before the jump, every time. That's handy for something that
has to happen on every call and wrong for anything that should happen once.

One last thing you may run into: with `-Wextra`, GCC warns "this statement may
fall through" and points at `lc-switch.h`. That's `pt->lc = 10;` running on into
`case 10:`, which is exactly what's supposed to happen. Turn the warning off
with `-Wno-implicit-fallthrough`, or use the GCC implementation.

### The GCC implementation

GCC and Clang support an extension called *labels as values*: `&&label` gives
the address of a label, and `goto *pointer` jumps to it. With that, a local
continuation is simply a code address. Here is `lc-addrlabels.h`:

```c
typedef void * lc_t;

#define LC_INIT(s) s = NULL

#define LC_RESUME(s)    \
  do {                  \
    if(s != NULL) {     \
      goto *s;          \
    }                   \
  } while(0)

#define LC_SET(s)                           \
  do {                                      \
    LC_CONCAT(LC_LABEL, __LINE__):          \
    (s) = &&LC_CONCAT(LC_LABEL, __LINE__);  \
  } while(0)

#define LC_END(s)
```

`LC_SET` puts a label named after the line (`LC_LABEL10` for line 10) and
stores that label's address. `LC_RESUME` jumps to the stored address if there
is one. The wait from the switch example becomes:

```c
do {
    do {
LC_LABEL10:
        pt->lc = &&LC_LABEL10;
    } while (0);
    if (!(counter == 1000)) {
        return 0;
    }
} while (0);
```

To use it, define `LC_INCLUDE` for the whole build:

```make
CFLAGS += -DLC_INCLUDE='"lc-addrlabels.h"'
```

Set it once for every file rather than file by file. The two implementations
give `struct pt` different sizes, and files that share a `struct pt` have to
agree on what it is.

Compared with the switch implementation:

- Waits inside `switch` statements work.
- The continuation is a pointer: two bytes on AVR and MSP430, four on 32-bit
  ARM.
- A call is a few cycles cheaper (see below).
- It only works with GCC and Clang, since it isn't standard C. Build with
  `-std=gnu99` or `-std=gnu11` rather than `-std=c99 -pedantic`.
- One wait per line still applies, local variables still aren't saved, and a
  wait in a helper function is no longer caught by the compiler.

### What it costs

These figures are from the paper, measured on MSP430 and AVR
microcontrollers:

- **RAM.** One continuation per protothread, two bytes. A state machine needs
  one byte for its state variable. In the paper's radio driver the state
  machine used 1 byte, the protothread 2 and a real thread 18. For a
  code-distribution program it was 1, 2 and 34.
- **Time.** On the MSP430, one call to the radio driver took about 92 cycles as
  a state machine, 98 as a protothread with the GCC implementation and 107 with
  the switch implementation (`-Os`). A version on cooperative threads was
  roughly three times slower than the protothread, because of the stack
  switching.
- **Code size.** Usually somewhat larger than a hand-written state machine:
  between 2% and 18% for most of the programs the authors rewrote, 72% for one,
  and 23% smaller for another.
- **Source code.** 31% fewer lines on average, with most of the explicit states
  and transitions gone.

## Files in this repository

| File | Contents |
|---|---|
| `src/pt.h` | The protothread macros. The only header most code needs. |
| `src/pt-sem.h` | Counting semaphores. Includes `pt.h`. |
| `src/lc.h` | The local continuation interface. Selects an implementation. |
| `src/lc-switch.h` | The default implementation, in standard C. |
| `src/lc-addrlabels.h` | The GCC and Clang implementation. |

Changes from the original 1.4 release:

- `lc-switch.h`: `LC_INIT` no longer ends in its own semicolon. Before,
  `PT_INIT(&pt);` expanded to two statements, so
  `if (x) PT_INIT(&pt); else ...` failed with "else without a previous if".
- `pt.h`: `PT_BEGIN` casts `PT_YIELD_FLAG` to `void`. A protothread that never
  yields doesn't use the flag, and `-Wall` used to warn about it.
- `lc-addrlabels.h`: includes `<stddef.h>`, which defines `NULL`. Before,
  selecting this implementation failed with "'NULL' undeclared" unless some
  other header happened to define it.

## Credits and further reading

Protothreads were created by Adam Dunkels at the Swedish Institute of Computer
Science. This repository is a fork of
[markose/Protothreads](https://github.com/markose/Protothreads).

- The [protothreads website](https://dunkels.com/adam/pt/), where the examples
  above come from.
- Adam Dunkels, Oliver Schmidt, Thiemo Voigt and Muneeb Ali,
  ["Protothreads: Simplifying Event-Driven Programming of Memory-Constrained Embedded Systems"](https://dunkels.com/adam/dunkels06protothreads.pdf),
  SenSys 2006. Most of [How it works](#how-it-works) follows this paper.
- Simon Tatham,
  ["Coroutines in C"](https://www.chiark.greenend.org.uk/~sgtatham/coroutines.html),
  the `switch` technique the default implementation is based on.

## License

Protothreads are released under a BSD-style open source license that allows
both commercial and non-commercial use. The only requirement is that you give
credit in the source code and in the documentation of your product. Keep the
copyright notice at the top of each header. The full text is in
[LICENSE](LICENSE).
