# Reference: All about Resets
Various reset sources are available. Thus far the options have been the same on all modern AVRs. All six reset causes are frequently encountered important to know about. It's also important to be aware of the possibility that your code could reset - or appear to - without the chip actually resetting. I refer to this as a "dirty reset", and it is a Bad Thing. Frequently that is what is happening when you see an AVR get into a bad state after some adverse event, and not work until powrcycled or reset. That is discussed in it'as own section further down.


| Reset Source         | Flag name         | Flag bit | Notes     |
|----------------------|-------------------|----------|------------
| UPDI Reset           | RSTCTRL_UPDIRF_bm |  1 << 5  | Done during programming several times. |
| Software Reset (SWR) | RSTCTRL_SWRF_bm   |  1 << 4  | When requested by the application |
| Watchdog Reset (WDR) | RSTCTRL_WDRF_bm   |  1 << 3  | When WDT expires w/out wdr instruction<br/>wdr executed before window in windowed mode|
| External Reset       | RSTCTRL_EXTRF_bm  |  1 << 2  | When reset pin is brought LOW |
| Brownout Reset (BOR) | RSTCTRL_BORF_bm   |  1 << 1  | When Vdd lower than BOD threshold and BOD enabled. |
| Power on reset (POR) | RSTCTRL_PORF_bn   |       1  | On power on (Vdd goes above V<sub>POR</sub> from below V<sub>PORR</sub>)

* All of these resets restore the peripheral registers to their power on state - with the exception of the reset flag register.
* The WDT does not turn itself back on when it causes a reset on modern AVRs.
* A BOR will clear all other reset flags except PORF.
* A POR will clear all other reset flags.


## Determining reset cause
The cause of reset is recorded whenever any reset takes place. These flags are only cleared manually and by BOR and POR. On non-optiboot boards - that's it - you're responsible for reading and clearing them. Resets other than power on reset will not clear flags already set. If you are using Optiboot, when it jumps to the application (after bootloader runs normally if that reset cause is an entry condition), it will save the reset cause to the `GPIOR0` or `GPR0` register (both the new name, `GPR.GPR0` and the old style `GPIOR0` will work), and clear all of the flags. The 6 lower bits are used to record UPDI reset, Software Reset, Watchdog Reset, External Reset, Brownout Reset and Power-on Reset, in that order. Multiple can be set at once when multiple reset conditions occur. A BOD reset often happens as the power supply voltage is ramping up.

```c++
  /* Non-optiboot minimaum reset flag checking */
  /* bare minimum */
  uint8_t resetflags = RSTCTRL.RSTFR; // Read the flags.
  RSTCTRL.RSTFR = resetflags;         // Clear them by writing a 1 to them.

  /* Recommended for non-optiboot */
void init_reset_flags() {  // This example is only valid for NON OPTIBOOT configurations.
  uint8_t reset_flags = RSTCTRL.RSTFR; // Read into a local variabke, it will go out of scope and be gone before the app starts.
  RSTCTRL.RSTFR = reset_flags;         // Clear the flags by writing the value back to it.
  if (reset_flags == 0){               // if no flags, reset wasn't clean - reset cleanly.
    _PROTECTED_WRITE(RSTCTRL.SWRR, 1);
  }
}


/* Optiboot - no particular requirement */
  uint8_t resetflags = GPR.GPR0;      // No special action required when Optiboot is used because it does this itself.
```

As a matter of best practices, and for any deployed or production system you should check the reset flags as soon as possible unless using Ootiboot (it does this for you), you should always check the reset flags. Ideally, override init_reset_flags() as shown in the examples. T - as I was considering the poteential hazard of dirty resets, ideally by overriding init_reset_flags(), a new function added in 1.3.7 which gets called very early in the initialization process - essentiallhy as soon as we're no longer running the blob of initial assembly and AVR-GCC output can be run (we call it in .init3, right after r1 is written 0 abd the stack pointer initialized. There are three things you need to do. 1. Check whether the flags all clear (ie, RSTCTRL.RSTFR == 0) and if so issue a software or watchdog reset immediately. 2. Clear the flags. 3. Store any information your code will need from this register to a global variable.

Pretty straightforwar to do (why it's important is discussed in the dirty reset section below. I realuzed that these were more of a potential problem on these parts than classic AVRs for a few reasons (though they absolutely are a problem there too)

## The Reset Pin
When the reset pin is acting as reset, the pullup is always enabled. If the pin is ever brought to a logic LOW, it will reset the chip, Just like on the classic AVRs, you should never jumper an output pin to reset aqd drive it low to reset. (We have two great ways to reset from software now!).


### AutoReset
Normal Arduino boards autoreset wbn the serial port is opened. It does this throughg the "DTR autoreset circuit". AVRdude, like any program that doesn't override the default behavior will set DTR ad RTSlines low while the serial port is open. This circuit converts the transition into a pulse, allowing the chip to leave reset too. (your finer serial terminals let you control these two pins. Generally speaking DTR and RTS are interchangible, abd you use whichever one the serial adapter makes available more easily.

It will reset the chip when serial connection is opened like on typical Arduino boards. When using a bootloader, this is very convenient and allows for "normal" uploads without manually resetting it. When no bootloader is in use, it is still often useful to ensure that you see all the output from the start of the sketch, though it is sometimes preferable to have one adapter connected to the serial port continually with a tool outside the Arduino. Occasionally (for example, if you have it connected to a computer controlling a long running task, but the computer has turned itself off and gone into power save mode) autoreset is your enemy. Restablishing a connection would trigger autoreset in that case, while if it were not present, that isn't an issue. The internal pullup on Reset is always enabled as long as the pin hasn't been turned into a normal input. This internal pullup, like all the others, is fairly weak. Adding a 10k resistor to Vcc, even if not using the full autoreset circuit, can provide improved noise rejection, though it is not required. If you aren't using autoreset at all, but noise on the reset pin is expected to or has been found to be a problem, combine the resistor with a 0.1uF cap between Reset and Ground, which will not interfere with a reset button, but will prevent plausible sources of electrical noise from causing and unwanted reset. A capacitor between reset and ground will defeat autoreset (though in order to be guaranteed to be effective, it must be at least 4 times the value of the autoreset capacitor to be certain of preventing a reset, though one as small as 1/9th the value of the autoreset capacitor could potentially interefere (the classical method of using a capacitor of 10 uF to disable autoreset on an Uno with a 0.1uF autoreset cap greatly exceeds what is necessary, though it ensures that multiple mistakes would have to be made for it to not work). Note that if DTR is tied to ground when the serial adapter is not connected, the autoreset capacitor will provide the same noise immunity.

A representative schematic is shown on the left below.

* 1 Small signal diode (specifics aren't important, as long as it's approximately a standard diode)
* 1 0.1uF Ceramic Capacitor
* 1 10k Resistor

If you aren't using autoreset, but the pin is still reset, it is suggested to connect that capacitor and resistor as shown on the right.


![Autoreset circuit and non-autoreset circuit](ResetCircuits.png "Reset Circuit Examples")


If the reset pin is still causing spurious external resets with that sort of countermeasure, after verifying that the resets are actually external resets, rather than some other kind, you should question whether the reset line is somehow unintentionally connected to a data line or similarly miswired. Overcoming a 10k resistor and 0.1uF capacitor takes an unusually large amounts of EMI, and is not plausible except in the most extreme circumstances. Whatever the reason, it likely points to a design flaw somewhere. However, unlike classic AVR parts you can disable reset without causing any problems reprogramming these parts

### Reset pin as Input
The Reset pin can be configured as an input. It will be `PIN_PF6` if this option is selected. It can never be made into an output, unlike on tinyAVR and most classic AVR parts. Unlike classic AVRs, *setting the reset pin as an input poses no particular difficulty for reprogramming*. This can be selected from the Tools -> Reset Pin menu; you must "burn bootloader" to apply the selection to a given chip - it is not set on normal uploads because it's on the same fuse as some non-safe settings (we don't have a facility to modify a certain bit within a fuse, only to set it). The only cost to doing this is that you dob't have a reset pin anymore.

## Triggering a reset from software
These parts support a native software reset operation; on classic AVRs, the only way was to enable the watchdog timer, and then wait for it to time out. With the modern AVR devices, there are two ways to trigger a reset from software: A watchdog timer reset (as before), and the native software reset. Unlike classic AVRs, after a WDT reset, the watchdog timer is not forced on with the minimum timeout (on classic devices, this is why doing a WDT restart with very old bootloaders instead hung the board - the bootloader wasn't smart enough to turn it off before it was reset by it).

These two methods of resetting the chip allow you to signal to the application or bootloader what sort of condition triggered the reset.

Additionally, while the bootloader, if used (see below) will run after a software reset, it will NOT run after a watchdog reset (well - it will run, but only long enough to read the reset flag register and see that it was restarted by the WDT: That means that either the bootloader just ran, finished, and reset the device (If we didn't jump to the app in this case, we'd just sit in the bootloader doing WDT resets forever), that the application suffered from a WDT timeout due to a bug or adverse conditions (that's not the bootloader's business to get involved in) or that the application intentionally triggered a WDT reset. None of those are "entry conditions" for the bootloader, so it just stashes the reset flags, clears them, and jumps to the app).This allows the application a convenient mechanism to restart itself without having to potentially wait through the bootloader attempting to communicate with whatever is connected to the serial port.

Note: While the windowed mode would at first seem to suggest that you could reset much faster by setting it and then executing `WDR` until it resets from missing the window, you don't gain nearly as much as you'd think. First, the initial WDR needs to be synchronized - 2-3 WDT clocks, ie, 2-3 ms. Additional WDRs executed while this is happening are ignored. Only when the second WDR makes it to the watchdog timer domain will it reset the system. So the overall time to restart is 6-9ms. Instead 10-11 ms (sync delay + minimum timeout).

```c++
void resetViaWDT() {
  _PROTECTED_WRITE(WDT.CTRLA,WDT_PERIOD_8CLK_gc); //enable the WDT, minimum timeout
  while (1); // spin until reset
}

```c++
void resetViaWDTFaster() {
  _PROTECTED_WRITE(WDT.CTRLA,WDT_WINDOW_8CLK_gc | WDT_PERIOD_8CLK_gc); //enable the WDT, minimum timeout, minimum window.
  while (1) __asm__ __volatile__ ("wdr"::);
  // execute WDR's until reset. The loop should in total take 3 clocks (the compiler will implement it as wdr, rjmp .-4)
  // but because of the sync delay described above, it will run thousands of times before the first premature (from the
  // WDT's perspective) wdr finally makes it to the WDT domain and slams into the closed window.
}

void resetViaSWR() {
  _PROTECTED_WRITE(RSTCTRL.SWRR,1);
}
```

## Using watchdog to reset when hung
If you only worked with the watchdog timer as an Arduino user - you might not even know why it's called that, or what the original concept was, and just know it as that trick to do a software reset on classic AVRs, and as a way to generate periodic interrupts. The "purpose" of a watchdog timer is to detect when the part has become hung - either because it's wound up in an infinite loop due to a bug, or because it wound up in a bad state due to a glitch on the power supply or other adverse hardware event, has been left without a clock by an external clock source failing, went to sleep waiting for some event which doesn't end up happening (or without correctly enabling whatever is supposed to wake it up) - and issue a reset. It is often anthropomorphized as a dog, who needs to be "fed" or "pet" periodically, or else he will "bite" (commonly seen in comments - the latter generally only when intentionally triggering it, as in `while (1); //wait for the watchdog to bite`).

You would first initialize the WDT like:
```c++
_PROTECTED_WRITE(WDT.CTRLA,settings); //enable the WDT
```

To configure the WDT to reset the device after a period of time, replace `settings` above with the desired WDT timeout period from this table. If is getting stuck somewhere that causes it to repeatedly reset the WDT or you are concerned that it might, you can configure it in window mode to reset if an attempt is made to reset the watchdog timer within the specified period. To do this, bitwise OR the two, eg: `_PROTECTED_WRITE(WDT.CTRLA, WDT_PERIOD_8KCLK_gc | WDT_WINDOW_16CLK_gc );` would set the WDT to reset the device if two attempts to reset the watchdog were ever made within 16 milliseconds (before the "window" opens), or if no reset was performed in the 8 seconds after that (when the window closes). If no window period is included, there will be no delay before the window opens after the last wdr instruction. Note that in all cases, multiple wdr instructions fired in very rapid succession (2-3 ms) will be ignores as it will still be synchronizing the previous one to the WDT domain, and will be ignored. The window mode only starts opening and closing the window after the first wdr after being configured.
A configuration with a window setting but no period is malformed and nonsensical, and such a confiuguration should not be used.


| Timeout | WDT period name      | WDT Window name      |
|---------|----------------------|----------------------|
|  0.008s | WDT_PERIOD_8CLK_gc   | WDT_WINDOW_8CLK_gc   |
|  0.016s | WDT_PERIOD_16CLK_gc  | WDT_WINDOW_16CLK_gc  |
|  0.032s | WDT_PERIOD_32CLK_gc  | WDT_WINDOW_32CLK_gc  |
|  0.064s | WDT_PERIOD_64CLK_gc  | WDT_WINDOW_64CLK_gc  |
|  0.128s | WDT_PERIOD_128CLK_gc | WDT_WINDOW_128CLK_gc |
|  0.256s | WDT_PERIOD_256CLK_gc | WDT_WINDOW_256CLK_gc |
|  0.512s | WDT_PERIOD_512CLK_gc | WDT_WINDOW_512CLK_gc |
|  1.024s | WDT_PERIOD_1KCLK_gc  | WDT_WINDOW_1KCLK_gc  |
|  2.048s | WDT_PERIOD_2KCLK_gc  | WDT_WINDOW_2KCLK_gc  |
|  4.096s | WDT_PERIOD_4KCLK_gc  | WDT_WINDOW_4KCLK_gc  |
|  8.192s | WDT_PERIOD_8KCLK_gc  | WDT_WINDOW_8KCLK_gc  |

### Resetting the WDT
A typical use of this is to have the main loop (generally loop() in an Arduino sketch) reset the watchdog at the start or end of each loop, so when a function it calls ends up hung, we can use:

```c
// As a function
void wdt_reset() {
  __asm__ __volatile__ ("wdr"::);
}
```

Or

```c++
// as a macro (which is all that wdt.h does)
#define wdt_reset() __asm__ __volatile__ ("wdr"::)
```

### Disabling WDT
In some cases you may only want the WDT enabled when certain routines prone to hanging due to external conditions, and then turn it off again.
```c++
_PROTECTED_WRITE(WDT.CTRLA,0); //Yeah, that's it.
```

At the other extreme you may want it to be impossible for code, even the cleverest bugs, to turn it off. You can lock the WDT in it's current configuration by writing the WDT_LOCK bit in WDT.STATUS to 1 - only a system reset will unset the bit.

```c++
__PROTECTED_WRITE(WDT.STATUS,WDT_LOCK_bm); // call after setting WDT to desired configuration.
```
For even more protection (and more nuisance in keeping the WDT from biting at all times). you can set the WDTCFG fuse via UPDI programming. At startup, that value is copied to WDT.CTRLA, and the lock bit in WDT.STATUS is set. UPDI programming is the only thing that can undo the WDT if congfiguree through the fuses. This feature is not exposed by the core - you must manually run avrdude or SerialUPDI and use it to write that fuse. UPDI uploads do not set it (the current value is left in place), and "burn bootloader" will reset that fuse to 0 (the point of burn bootloader is initializing the chip to a fully known state; it only involves a bootloader if you've chosen an optiboot board configuration.

### Summary and mini-example
So overall, if you wanted your sketch to reset if you ever spent longer than 8 seconds between loop() iterations and also detect when a WDT reset just occurred and take special actions in setup you might do this
```c++
void wdt_enable() {
  _PROTECTED_WRITE(WDT.CTRLA,WDT_PERIOD_8KCLK_gc); // no window, 8 seconds
}

void wdt_reset() {
  __asm__ __volatile__ ("wdr"::);
}

void wdt_disable() {
  _PROTECTED_WRITE(WDT.CTRLA,0);
}

/* If you at some point plan to put the chip to sleep you need to turn off the WDT or it will reset you out of sleep... */
void goToSleep() {
  wdt_disable();
  sleep_cpu();
  wdt_enable();  // turn the WDT back on promptly when we awaken.
}

void loop() {
  wdt_reset(); // reset watchdog.
  <snip - rest of loop goes here>
}

/* for NON-OPTIBOOT */
void setup() {
  wdt_enable(); //we're super paranoid, so we turn on WDT immediately!
  uint8_t resetflags = RSTCTRL.RSTFR; // Read the flags
  RSTCTRL.RSTFR = resetflags; // clear them for next time (only power on reset will clear then otherwise)
  if (resetflags & RSTCTRL_WDRF) { //means it was a WDT reset
    NotifyUserOfHangAndWDTReset();
    if (resetflags != RSTCTRL_WDRF) {
      Serial.println("Weird, we clear the flags upon reset, how did we end up with more than just one reset cause flag?"); // far from unheardof.
    }
  }
  <snip - rest of your setup goes here>
}
/* For OPTIBOOT */
void setup() {
  wdt_enable(); //we're super paranoid, so we turn on WDT immediately.
  uint8_t resetflags = GPR.GPR0; // Optiboot stashes the reset flags here before clearing them to honor entry conditions
  // GPR.GPR0 = 0; // no need to clear because this is reset at startup to 0 either way - unless you need it later..
  if (resetflags == RSTCTRL_WDRF) { //means it was a WDT reset. In optiboot configurations, this should always be accompanied by another flag unless it was a wdt reset from the application.
    NotifyUserOfHangAndWDTReset();
  }
  <snip - rest of your setup goes here>
}

```


## The WRONG way to reset from software
I have seen people throw around `asm volatile("jmp 0");` as a solution to a need to reset from software. **Don't do that** - all compiled C code makes assumptions about the state from which it is run. Jumping to 0 from a running application violates several of them unless you take care to avoid those pitfalls (if I were to add a comment after that line, it would read something like `// Reset chip uncleanly to produce unpredictable results`. Resetting with a jump to 0 was always risky business and should never be done on any part, ever (certainly not without taking a bunch of precautions and knowing exactly what you're getting into (noboedy who did this seemed to be). Now that we finally have a right way to do software reset, there is absolutely no excuse for intentionally triggering a dirty reset.

## The danger of "dirty" resets
Now that intended resets have been covered, we need to confront the other kind of resets: unintended ones that happen unexpectedly, those aforementioned dirty resets. Both Optiboot and the init() function (which runs before setup to make the chip run at the speed you asked, start millis, and so on) assume that everything starts in it's reset configuration - a great deal more code would otherwise be need. So does the vast majority of user code. Making the assumption that the sketch starts with everything in the reset configuration a very helpful simplifying assumption. It is almost always true, too. It is not true under a variety of error conditions, though, and after bad things happen, it's possible for the the program counter to start from 0x0000 again, as if it were reset, but for no hardware rewet to occur. The registers could take any value (but will generally be whatever they were set to immediately before the bad thing happened.. The consequences can range from unnoticable to catastrophic. Depending on the nature of the incident and the code that runs after a "reset", very different things could end up happening. You may notice only an unexpected reset, or it may hang until manually rewset, or get into some strange tight loop that it normally doewn't. Fortunatwely, it is easy to prevent a dirty reset from causing trouble with very simple countermeasures.

Most of us have seen an an AVR get into a "bad state" somehow, where it's hung despite having a clock signal. Many times people reset and forget about it. If you are trying to make a more robust product, suitable for use in production, or one which would be a serious invconvenience were if it failed suddenly (temporarily or permanently), be sure you've taken all the precautions against this. Most of the time, if they had taken these steps below, they weould have reset out of the bugged state without that undesired behavior.

Very simply, always check and clear the reset flags. (read, and then write the value back to it. These are the kind of bits you write 1 to in order to clear), while writing a 0 to a bit does nothing). If the value you read was 0 (no reset cause), immediately do a software reset. When the sketch comes back after that rewet, you will see the software reset flag, and know that the chip was cleanly initialized. *That is all you need to do.*


**If Optiboot is in use, you are all set** - no further action is needed, Optiboot already clears the reset flags and stashes them in a `GPR.GPR0` in order to ensure it can honor reset conditions and does not get broken by invalid assumptions about reset state. Since the GPRs do not persist across restarts, there's no need to clear it. By the time your app runs, they actual reset flags have been cleared. Checking reset causes is a good idea still, but you don't need do anything special if you see that the reset cause stashed in GPR0 is 0 or - much more liekly - indicates a watchdog reset and nothing else meaning that Optiboot was run with either no reset cause, or from a WDT reset. (Optiboot uses that to get a hardware resset and a timeout and ensure user code starts with a clean state, all in one). This isn't to say you shouldn't be concerned to see that unexpectedly. If you do, that's a clue that there may be some soet of instability in your code or hardware.

**Otherwise, check early for best resukts**  You want to check for them as soon as possible, so as to mininmize time spent in a bugged state potentially generating wrong output and finding ways to break these countermeasures. The most obvious place is at the start of setup. Checking and clearing there is a good idea - you don't even need to trigger the reset the reset or save the value if you do it in setu and the specific cause doesn't matter to you. The core already checked them before doing anything else and would have reset if it saw that no flags were set. This suffuicient for non-production decvices:
```c++
void setup() {
  RSTCTRL.RSTFR = RSTCTRL.RSTFR;
  // REST OF SETUP HERE!
}
```

However, it's ideal to do this even earlier before execution has even reached `main()`, much less `setup()`. We check and do system reset if it's 0 internally. That is done in the `init_reset_flags()` function. We call that before doing anything else on startup, including any other hardware initialization. Just reading and clearing the flags during setup will get the job done if you don't need to know the reset cause and it is not performing any particularly important function, but for higher reliability, you want the "gap" between start of execution and the clearing to be as small as possible. This is the recommended solution:

Example 1: Override, check and clear flags and discard the value - for any app that doesn't need to make decisions based on reset flags
```c++
void init_reset_flags() {  // This example is only valid for NON OPTIBOOT configurations.
  uint8_t reset_flags = RSTCTRL.RSTFR; // Read into a local variabke, it will go out of scope and be gone before the app starts.
  RSTCTRL.RSTFR = reset_flags;         // Clear the flags by writing the value back to it.
  if (reset_flags == 0){               // if no flags, reset wasn't clean - reset cleanly.
    _PROTECTED_WRITE(RSTCTRL.SWRR, 1);

  }
}
//nothing spoecial needed anywhere else
```

Example 2: Override, check and clear flags, and retain the value, and use it to print a message about the reset cause.
```c++
uint8_t reset_flags=0x80; // this bit is not a reset flag, thus if this is still the result, it means the velow function wasn't called.
                          // That can only happen on versions before 1.3.7 or when using Optiboot, in which case it is already done for you.
void init_reset_flags() {
  reset_flags = RSTCTRL.RSTFR; // Read into a global variable. It will be available to the rest of the application
  if (reset_flags == 0){               // if no flags, reset wasn't clean - reset cleanly.
    _PROTECTED_WRITE(RSTCTRL.SWRR, 1);
  }
  RSTCTRL.RSTFR = reset_flags;         // Clear the flags by writing the value back to it.
}
void setup() {
  Serial.begin(116200);
  if (reset_flags == 0x80 ) {
    Serial.println("Using old version or compiling for Optiboot. ")
  }
  if (reset_flags & RSTCTRL_UPDIRF_bm) {
    Serial.println("Reset by UPDI (code just upoloaded now)");
  }
  if (reset_flags & RSTCTRL_WDRF_bm) {
    Serial.println("reset by WDT timeout");
  }
  if (reset_flags & RSTCTRL_SWRF_bm) {
    Serial.println("reset at request of user code.");
  }
  if (reset_flags & RSTCTRL_EXTRF_bm) {
    Serial.println("Reset because reset pin brought low");
  }
  if (reset_flags & RSTCTRL_BORF_bm) {
    Serial.println("Reset by voltage brownout");
  }
  if (reset_flags & RSTCTRL_PORF_bm) {
    Serial.println("Reset by power on");
  }
}
```
Example 3: Override, check and clear flags. Signal that it's a recovery from dirty reset by turning on an LED during the WDT timeout - this can be used to aid in debugging when you don't understand why it is resetting. You can check additional things in this case, like CPUINT.STATUS (low two bits in particular - these indicate if it thinks an interrupt is running at this point. If they are not 0, you're firing an interrupt that doesn't exist (and you're also not reading the warnings!!!)
```c++
  uint8_t reset_flags;
  void init_reset_flags() {
    reset_flags = RSTCTRL.RSTFR;   // Read flags. Declare variable as local or global as needed.
    if (reset_flags == 0){                    // if no flags, reset wasn't clean - reset cleanly.
      _PROTECTED_WRITE(WDT.CTRLA,WDT_PERIOD_2KCLK_gc); // Set the watchdog to bite in 2 seconds.
      openDrainFast(LED_BUILTIN, LOW);        // This turns LED on if it's active low - but it's unlikely to be. This is a trick to fast port direction setting without a pinModeFast()
      digitalWriteFast(LED_BUITIN, HIGH);     // The above line is key because it sets the direction to output. Now we can write it high, and get the output we want. Comment out if LED is active LOW
      while(1);                               // wait for timeout
    }
    RSTCTRL.RSTFR = reset_flags;              // Clear the flags by writing the value back to it.
  }
```


**In Optiboot configurations, the bootloader handles the recovery from dirty resets** and this information is only useful for helping to figure out why. `GPR.GPR0` will show an ordinary WDT and software reset as the cause, because that's what Optiboot did to fix it.

So how can this happen?

### Power supply or clock issues
Are the decoupling caps in place? Does it coincide with a change in load that might cause the power rail to depart from it's nominal voltage? Is the crystal working correctty? Is everything well soldered? SMD crystals tend to be particularly hard to solder well, sometimes forming intermitent connections that only work sometimes L

### Interrupt enabled while ISR does not exist
Amazingly, it's not an error to do `ISR(Something not an interrupt vector) {...}` - it gets a warning for misspelled vector, but not an error (one of the reasons we force warnings on things like THAT are only warnings, not errors). Enabling a interrupt without an ISR and letting it trigger will achieve the same thing: BAD_ISR handler is called. The default implementation is to jump to 0. No flags are cleared nor interrupts disabled. the default `BAD_ISR` implementation just jumps to 0... And since these parts track whether the code is supposed to be in an interrupt (see CPUINT.STATUS LVLnEX bit), you end up outside of an interrupt, with the hardware still thinking your're in one and only elevated priority interrupts can fire, and everything else will be assuming you'e got a badwill be awful and broken. Don't enable an interrupt without the ISR defined, and the misspelled vector warnings virtually always indicate that this is going to happen.

### Scribbling over memory
Local variables that exceed what will fit in the registers get created on the stack. The stack is also where the return address of a function call is stored. If we were to be accessing an array that's on the stack, but wrote off the end of it, we could overwrite the return address, so that we returned to an address based on that instead of where we were. Frequently this will point to empty space, and execution will skid along the 0xFFFF's of empty flash (this is an skip-if-bit-in-register-set instruction, so either every one, or every other one will be processed, depending on the state of that working register - specifically with all 1's, it looks at the high bit of the last register, r31 - the high byte of the Z pointer), which continues at 1 word per cycle because skips take an extra cycle if skipping, until reaching the end and wrapping around. At that point, if the skip-if's aren't skipping, you'll land on the reset vector. Otherwise you'll land on the second half of the reset vector and execute that mess as an instruction, then hit the NMI interrupt vector, or some other vector which isn't set by the core; the BAD_ISR implementation jumps to 0 and there's your dirty reset.

Note that you don't always have empty flash after your sketch - specifically, if using the included bootloader, you won't if the previous program was bigger. That code should remain unreachable, except in this kind of situation. The results are likely to be bizarre, confusing, and obviously incorrect.

#### Writing to the SP (stack pointer)
When you don't *really* know what you're doing, let the compiler manage the stack pointer! Any mistake is likely to will generally result in a dirty resert.

#### A stack-heap collision
Where the local variables on the stack crash into the ones in the heap or ones allocated with malloc (ie, running of ram at runtime. This will lead to both getting corrupted, likely quickly resulting in a a bogus value at the top of the stack (located now somewhere in the heap) being returned to. This can also corrupt a function pointer leading to situation below.

#### Bad inline assembly
If you're writing assembly you should know this.... but be certain that you use the same number of push and pop instructions, and be careful to give the right constraints so the compiler can't assume that a value your changing is constant. It's very easy to forget that, say, that Z-pointer you're writing to with SPM Z+ - if you lie to the compiler and say it's just an input, the compiler will proceed as if the the value won't change. The result is bad and unpredictable behavior.  **These specific bugs only show up in aseembly, not C**

One way or another, you've wound up with 0 or a number pointing to empty flash after your program at the top of the stack, and a return is executed, and you wind up at the beginning in an inconsistent state.

### Bad function pointers
If a function pointer doesn't point to a function, when it gets called, badness ensues. It can act as a jump to any other part of the program. A null pointer, or a function pointer to a location after the end of program will take the path described above before reaching the reset vector. This is really just another way of getting program execution into a part of the the flash where it shouldn't be, combined with the fact that landing in the in empty flash after a program will end up at the reset vector.

### Don't write to `SP`
Those are the program counter (current position that code is executing from) and the stack pointer (location in ram where the most recently pushed value lies). You never need to read or write these - you have instructions to do that for you (PC autoincrements during execution, as you might guess, and is changed by the jump, branch, and call instructions). You shouldn't be writing it, nor reading it for that matter, since it's useless. Because you will alwayws get the address of the instruction that reads it. The stack pointer gets written by compiler generated code at startup to make it point to where it should (even though the hardware guarantees that it will be set there to begin with! On classic AVRs it wasn't, I guess - and avrlibc has been spotty with updates), and potentially at other specific points under the compiler's control. C code should never write to it. Reading it, however does have utility in rare cases (the first practical one that comes to mind is finding out how big the stack is at a given point in execution if you suspect you're suffering a stack-heap collision not long after that.).

### Clock Abuse
Sufficient noise (like if you had a large 5-foot-something broad band antenna, like a human body, touching the crystal, particularly under adverse conditions) it can trigger a reset or hang due to clock failure (in theory that should trigger the clock failure detection, it often seems to just reset instead). Sometimes trying to switch clock sources to a crystal with inappropriate loading capacitors will result in a "bootloop" too. Overclocking far too hard can also do it (though you'll notice that it has lost its ability to get the right answers to math at lower overclocks, generally); I suspect that the actual direct cause of the resets is through the same mechanism by which 1 bits turn into 0's in an over-overclocked part, combined with the roles of the stack pointer and program counter.


### Example of how some common dirty resetrs break stuff, and how this solution fixes it

We have in the DxCore "library" examples one where you can comment out the reset section of init_reset_flags(), and one of three causes of a dirty reset (jump to 0, array overrun, and bad ISR), and see how that dirty reset has measurable impacts - registers don't get reset! But if you don't comment out the reset, like magic, those are all quickly resolved.  It will blink 5 times per second after a dirty reset. Otherwise, it will blink 1 second on, 1 second off.

Try it out if you want to see for yourself how things can go wrong, and how this fixes it!