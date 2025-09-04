---
title: "nRF51 RBPCONF bypass for firmware dumping"
date: 2025-09-04T21:41:00+02:00
toc: false
images:
  - /images/nrf51/uicr.png
  - /images/nrf51/rbpconf.png
  - /images/nrf51/auto_read.png         
  - /images/nrf51/firmware.png      
  - /images/nrf51/protection.png     
  - /images/nrf51/setup.png
  - /images/nrf51/board.png             
  - /images/nrf51/manual_read.png   
  - /images/nrf51/prot_mem_read.png
  - /images/nrf51/time.png
  - /images/nrf51/extracted_secret.png
  - /images/nrf51/noprotection.png  
  - /images/nrf51/rbpconf.png        
  - /images/nrf51/uicr.png
tags:
  - Hardware hacking
  - Physical security
image:
  - /images/nrf51/setup.png
---

A while ago I read about the [firmware dumping technique proposed by Include Security](https://blog.includesecurity.com/2015/11/firmware-dumping-technique-for-an-arm-cortex-m0-soc/) to bypass RBPCONF (Readback Protection) on nRF51 family MCUs. Recently I could spend some time attempting to replicate the effects of their research.

The nRF51 series is Nordic Semiconductor’s family of low-power SoCs built around an ARM Cortex-M0. If you’ve played with Bluetooth Low Energy (BLE) gadgets from a few years back such as beacons, smart locks or fitness trackers there’s a good chance they had an nRF51 inside. 

This technique is significant because retrieving the firmware is almost always the first step before any meaningful reverse engineering. Once the binary is available, an attacker can perform static or dynamic analysis to uncover hardcoded secrets, or look for exploitable bugs that could compromise the security of the system. For connected products such as smart locks, wearables, and IoT sensors, the impact of such access can be substantial.

What makes this bypass interesting is its non-invasive nature. Other approaches often involve some type of glitching, manipulating reset lines, or even destructive decapsulation, all of which risk damaging the device and require specialized equipment. In this case, the attack relies only on software manipulation through standard debugging interfaces. The target remains fully functional while its memory is exfiltrated, making the method practical and appealing.

So I decided to visit Aliexpress, bought two of the cheapest board I could find featuring an nRF51822 and went to work.

## nRF51 security in a nutshell

As numerous other MCUs, the nRF51 family offers some way to prevent final users of the device from reading the flash memory of the MCU and recover the firmware. Much of the device’s security posture is controlled through a set of non-volatile registers called **User Information Configuration Registers (UICR)** located at address `0x10001000`. This registers are what allows for enabling protections on this devices family.

|![UICR overview](/images/nrf51/uicr.png#center)|
| :----------------------------------: |
|              UICR overview             |

Registers inside UICR we are going to focus onto are: `CLENR0` and `RBPCONF`.

Register `CLENR0` is what allows dividing code flash in two areas: Code Region 0 (CR0) and Code Region 1 (CR1). CR0 always starts at `0x00000000`, and its size is defined by the `CLENR0` register. Everything above that boundary falls into CR1. If `CLENR0` is left untouched (`0xFFFFFFFF`), the whole flash is simply treated as CR1. 

sThere are a few differences between CR0 and CR1: 

- Code running in CR0 cannot be modified by code running in CR1.
- Pages of CR0 cannot be erased, CR0 is erasable only via chip-wide wipe (`ERASEALL`)

Generally speaking, this renders CR0 a more secure memory area suitable for critical software components or sensitive data storage.

### Understanding RBPCONF 

An nRF51 device can be configured with three different security configurations based on `RBPCONF`: no protection, `PR0` protection and `PALL` protection.

|![RBCONF Register](/images/nrf51/rbpconf.png#center)|
| :----------------------------------: |
|              RBCONF overview             |

`PR0` register allows for locking CR0 memory area. This protection, when enabled, prevents firmware running from CR1 from reading CR0 memory as well as locking the read-back on CR0 area from SWD debugger.  

The nRF51 also exposes the `PALL` protection, which locks down all code memory (both CR0 and CR1). Once `PALL` is enabled, no external debugger reads are possible. The only way to disable this protection is to perform a full chip erase (`ERASEALL`), which wipes both application and configuration data, protecting the firmware from exfiltration. The `PALL` mechanism is intended as "production lock" and is the strongest protection mechanism available in these MCUs. 

One security measure other SoCs offer, is to completely disable debugger accesses. However this is not a possibility with the nRF51 and even though the debugger won't be able to access flash memory directly, this poses a risk to the device and it is exactly the weakness we are abusing to access the whole memory.

### Exploiting RBPCONF

The protection mechanisms in place allow only code already present in the CR0 to read flash memory.

External debuggers are blocked from issuing flash memory reads directly. So, in order to extract data from the protected memory we will abuse the execution environment against itself. The register state, unlike the flash, remains observable and editable through the debugger interface.

The `RBPCONF` bypass is based on the fact that, being able to control CPU registers, it would be possible to halt and single step the CPU executing the firmware and change the behavior of instructions by altering generic purpose CPU registers values. This way it is possible to hijack instructions already residing in CR0 memory area. 
With this knowledge available it's just a matter of finding a load instruction that will take an address from one of the registers and puts the value from that address back to one of the registers.

Nordic’s model assumes that isolating CR0 and requiring ERASEALL for modification is sufficient to protect sensitive firmware. However, the lack of debugger lockout means these guarantees can be bypassed.

## The Setup

**Hardware:**  

- nRF51822 dev board 
- SEGGER J-Link
- Linux host machine  

**Software:**  

- [SEGGER Embedded Studio](www.segger.com/products/development-tools/embedded-studio/) 
- [OpenOCD](https://openocd.org/) 
- [nRF Command Line Tools](https://www.nordicsemi.com/Products/Development-tools/nrf-command-line-tools/download) (not strictly required, but handy for enabling protections & restoring a locked device) 
- arm-none-eabi-gdb + Python for automation 

The board I'm using is this one:

|![nRF51 board](/images/nrf51/board.png#center)|
| :----------------------------------: |
|              nRF51822 dev board             |

Fortunately SWD pins (SWDIO, SWDCLK) are exposed and it was possible to attempt opening a debug interface straight away. However pin distancing was not compatible with any breakout or breadboard I had at home, so I resorted to using some integrated circuit hooks and these ended up working just as fine.

|![nRF51 board connected](/images/nrf51/setup.png#center)|
| :----------------------------------: |
|              Dev board connected to debugger             |

Once board's pins VCC, GND, SWDIO and SWDCLK are connected to J-Link's corresponding pins it was possible to obtain SWD debug access both with J-Link suite tools and with OpenOCD using the commands:

`openocd -f /usr/share/openocd/scripts/interface/jlink.cfg -c "transport select swd" -f /usr/share/openocd/scripts/target/nrf51.cfg`

`JLinkGDBServer -device nrf51822_XXAA -if SWD`

Personally I'm more keen on using OpenOCD for this exploitation specifically because J-Link suite wouldn't allow opening  a GDB connection to nRF51 if the `PALL` protection is enabled. 

Once the SWD connection is achieved, we need to flash the device with a custom firmware, to do so I used SEGGER Embedded Studio as it was the fastest way to setup the SDK and toolchain for our target device. 

For this demo a firmware containing a single `printf("SECRET")` is used and the objective will be to confirm whether the string can still be retrieved with `PALL` protection enabled.

|![firmware](/images/nrf51/firmware.png#center)|
| :----------------------------------: |
|              Dummy firmware            |

Once the firmware is flashed, we can perform a few tests without protection enabled in order to better understand the differences in the behavior of the debugger when the protection is OFF and when it is ON.

With the GDB server running, we can attempt connecting and reading some memory:

|![no_protection](/images/nrf51/noprotection.png#center)|
| :----------------------------------: |
|              Memory read demo - no protection            |

As it is possible to see, bytes are correctly read back from flash memory.
In addition notice how we also read  `0xE000ED00 (CPUID)` this register will always be populated with `0x410CC200` on nRF51 and the value is readable when `PALL` protection is in place as well, rendering it a good reference value for future experiments.

At this point we can enable the `PALL` protection with command `nrfjprog --rbp ALL`, reboot the target device and attempt performing the same operations again.

|![protection_on](/images/nrf51/protection.png#center)|
| :----------------------------------: |
|              Memory read demo - with protection            |

We note how the same addresses come back as `0x00000000` when read back. 

## Dumping the firmware

The exploitation of the vulnerability goes through 2 steps: locating a load instruction and abusing it to read protected memory.

To achieve the first, it is possible to use GDB to set each register to address `0xE000ED00` since it contains a known value. If the instruction PC is pointing is a load, we should find the expected value (`0x410CC200`) in one of the registers.

We will simply need to issue commands:

```shell
set $r0=$r1=$r2=$r3=$r4=$r5=$r6=$r7=$r8=$r9=$r10=$r11=$r12=0xE000ED00
stepi
info registers
```

And we will monitor the output of `info registers` to see if any of the register gets populated with the expected value.

That's exactly what happens after a few iterations of this:

|![manual load](/images/nrf51/manual_read.png#center)|
| :----------------------------------: |
|              Load instruction found          |

We can see that after the execution of instruction at `PC=0x82` we have our expected value in register `r0`.
Once the load instruction is located, it is necessary to attempt reading bytes from the protected memory.

So now we populate registers with value `0x00000000` and attempt reading that word.

|![protected memory read](/images/nrf51/prot_mem_read.png#center)|
| :----------------------------------: |
|              Value read from CR0         |

and as expected we are able to read the first byte of protected memory.

Now it's time to render the exploit practical and to automate the firmware extraction and we can achieve this in several ways, I've decided to simply script the interaction with GDB commands. 
Depending on the exact MCU you're working with, you may encounter different flash sizes. In this case we are working with an `nRF51822_XXAA` which is the most commonly available variant of nRF51822 around and features 256kB of flash memory.

The extraction script will look something like this:

```shell
set $i = 0
set $end = 0x3FFFF
while $i <= $end
  set $pc=0x82

  set $r0=$r1=$r2=$r3=$r4=$r5=$r6=$r7=$r8=$r9=$r10=$r11=$r12=$i
  stepi

  printf "0x%08x\n", $r0
  set $i=$i+4
end
```

and the output on the terminal when the script is executed looks like this:

|![scripted read](/images/nrf51/auto_read.png#center)|
| :----------------------------------: |
|             Scripted memory read         |

After letting it run the script will have gone through the entire code space. A simple parser will then be used to cleanup the retrieved data and rebuild the binary file.

> For ease of use I've decided to write a more usable script capable of automatically detecting the load function, dump the firmware and reassemble it to generate a binary output. 
>
> You can find it at: [https://github.com/0xless/nRF51-RBPCONF-Bypass](https://github.com/0xless/nRF51-RBPCONF-Bypass) 

Once the script has finished running, in under 54 minutes, we get `firmware.bin` file:

|![dump complete](/images/nrf51/time.png#center)|
| :----------------------------------: |
|             Dump completed in 54 minutes         |

And performing `xxd` on the extracted firmware allow us to recover out `SECRET` value.

|![secret extracted](/images/nrf51/extracted_secret.png#center)|
| :----------------------------------: |
|             Secret value recovered from firmware         |

This proves the functioning of the exploit and the practicality of this attack.

The PoC script developed for this attack is far from perfect, it takes a lot of time to execute and the GDB + Python interaction is a bit "hacky".  A huge improvement it is possible to make is to aim to find instructions that load more bytes of flash memory to registers; an example would be `LDMIA Rn!, {Rlist}` that allow up to 32 bytes loads in a single step. When reading the memory, it is easy to evaluate the OPCODE of the instruction just read, if it's a `LDMIA` instruction, then we can edit the PC register value to jump there and attempt exfiltrating memory at higher speed. 

I've attempted sketching this using GDB scripting, but as the complexity of the project increased I realized this scripting language was not the tool for the job. For future developments I want to rewrite the whole script in python using library `pyswd` which looks promising in handling SWD interactions directly via python, however it is compatible with ST-Link debuggers only (which I don't own).

## Conclusions

The nRF51’s RBPCONF was designed to keep firmware safe, but not allowing final users to disable debugger access weakens the model. Even under PALL protection, careful register manipulation and instruction reuse make full extraction possible with modest tools.  

In this post we demonstrated how it is possible to do so and walked all the steps needed to discover the issue and replicate the attack.