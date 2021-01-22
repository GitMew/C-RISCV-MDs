# CASS Notes --- C & RISC-V Programming --- Pointers to Pointers

There's a special use-case for pointers I want to highlight. We'll start by the way I actually first came into contact with pointers, and subsequently came to know their usefulness.

## Pointers to pointers in C
At one point in time, some other students and I were building an Arduino robot for shooting ping pong balls. To time shots and to blink LEDs, global timer variables stored the time since the last relevant event (e.g. when the last shot was made), so that, using the system clock (the `millis()` builtin), we could check whether enough milliseconds had passed for the next event to take place.

For e.g. three LEDs (red, green and blue), the code could have looked like:
```c
#define TIMER_THRESH_MS_RED   1000
#define TIMER_THRESH_MS_GREEN 2000
#define TIMER_THRESH_MS_BLUE  3000

unsigned long timer_ms_red   = 0;
unsigned long timer_ms_green = 0;
unsigned long timer_ms_blue  = 0;

bool checkAndUpdateTimer_red() {
    if (millis() - timer_ms_red >= TIMER_THRESH_MS_RED) {  // Doesn't account for rollover of integers, but you get the idea.
        timer_ms_red = millis();
        return true;
    }
    return false;
}
bool checkAndUpdateTimer_green() {
    if (millis() - timer_ms_green >= TIMER_THRESH_MS_GREEN) {
        timer_ms_green = millis();
        return true;
    }
    return false;
}
bool checkAndUpdateTimer_blue() {
    if (millis() - timer_ms_blue >= TIMER_THRESH_MS_BLUE) {
        timer_ms_blue = millis();
        return true;
    }
    return false;
}
```
Hopefully, you see where I'm going with this: code duplication is bad. Yet, we have a conundrum here: each method assigns a value *to a variable defined in a higher scope*. That's easy enough. If we want to unify these functions into one for any amount of timers, a naïve first guess is to write:
```c
bool checkAndUpdateTimer(unsigned long timer_ms, int delay_ms) {
    if (millis() - timer_ms >= delay_ms) {
        timer_ms = millis();  // This doesn't do what you're trying to do!
        return true;
    }
    return false;
}
```
The assignment of `timer_ms` is done locally, but whatever value we set that timer parameter to can't escape the function, since *parameters are only accessible in the scope of the function*! Indeed: calling `checkAndUpdateTimer(timer_ms_red, TIMER_THRESH_MS_RED)` will *copy* the value of variable `timer_ms_red` into parameter `timer_ms`, and then the function will later reassign `timer_ms`, never touching `timer_ms_red`.

What we essentially want to do, is to *be able to vary the higher-scope variable that's assigned in the function*. We want to "pass a variable itself", not its value. *This* is where I met pointers. Using pointers, we ended up writing the following beauty (minus the rollover math):
```c
bool delayProtocol(unsigned long* timer_ms_variable, int delay_ms) {
    if (millis() - *timer_ms_variable >= delay_ms) {
        *timer_ms_variable = millis();
        return true;
    }
    return false;
}
```
Now we can "pass a variable" using a call like `checkAndUpdateTimer(&timer_ms_red, TIMER_THRESH_MS_RED)`, `checkAndUpdateTimer(&timer_ms_green, TIMER_THRESH_MS_GREEN)`, and `checkAndUpdateTimer(&timer_ms_blue, TIMER_THRESH_MS_BLUE)` (notice the ampersands). This works; we essentially have a machine that, via a pointer, is able to switch between the external variable it modifies (which itself is akin to a pointer).

Make no mistake, Java programmers: although you're working with references all the time in Java (allowing you to change an object's state remotely), you're **not** passing *the reference to the variable storing the reference to your object*. Java would hence have the same problem we had at first (although in Java, you would deal with timers differently and not have the problem at all).

## Pointers to pointers in RISC-V assembly
In assembly, you're used to throwing around addresses all day long. If an array starts at the address in register `x5`, then walking through it is as easy as updating the value we have by using `addi x5, x5, +4`. Simple address math.

What you aren't used to, however, is accessing *addresses of addresses of things* (where the "things" could be made up of more addresses). Let's say your boss tells you to program a **linked list**, and for it, you were writing a procedure `prepend` (i.e. you add an element to the front of the list, meaning you'll at least need the address of the first [value, pointer] pair in memory). However, **your boss tells you that it has to execute in-place**. That means you don't get to return the new address of the [value, pointer] pair you created in memory. How will you ever communicate to the caller the new starting address of the list, then?

The answer is that there must be a place in memory already where the caller of `prepend` expects the new address of the first element to appear. That means that, not only do you need the address of the first element of the list, but you actually need *the address of that address of the first element in the list*, so that you can *change* where the caller will go look for the first element of the list later on.

Hence, the first argument `x10` to `prepend` would not link like:
```
x10 = &firstElement -> heap@0x0 = firstElement
```
and instead, it'd link in the following fashion:
```
x10 = &(&firstElement) -> heap@0x0 = &firstElement -> heap@0x1 = firstElement
```
Of course, because it's a linked list, to get to `secondElement`, you would look a couple bytes to the right of `firstElement` to find the address to `secondElement`, to then follow it to get to `secondElement` that way. Indeed, we have three kinds of addresses to handle, and it makes your brain explode.
