Beaglebone Black Hardware Counter Capture Driver
================================================

Changes to ddrown version
-------------------------

Adapted version of `pps-gmtimer` from Dan Drown. Some functions were no longer exported by the linux kernel driver (dm-timer). See https://patchwork.kernel.org/patch/10106287/. 

The adapted module accesses the necessary functions through the `omap_dm_timer_ops` struct (`dmtimer-omap.h`), so the driver can compiled without rebuilding the kernel (`4.19.94-ti-r42` on current BBB r3).

The TCLKIN feature is not tested.

Building the module on BBB
--------------------------

```bash
sudo apt update
sudo apt install linux-headers-$(uname -r)
git clone https://github.com/kugelbit/pps-gmtimer
cd pps-gmtimer
make
```

The output after make should be something like this:

```
make -C /lib/modules/4.19.94-ti-r42/build M=$PWD ARCH=arm
make[1]: Entering directory '/usr/src/linux-headers-4.19.94-ti-r42'
  CC [M]  /home/debian/pps-gmtimer/pps-gmtimer.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /home/debian/pps-gmtimer/pps-gmtimer.mod.o
  LD [M]  /home/debian/pps-gmtimer/pps-gmtimer.ko
make[1]: Leaving directory '/usr/src/linux-headers-4.19.94-ti-r42'
```

Installing the Device Tree Overlay on BBB
-----------------------------------------

The device-tree-overlay file (`DD-GPS-00A0.dtbo`) defines to use UART2 (Pins: P9.21, P9.22) as UART for your GPS-Board and to use Timer4 (P8.7) as PPS-Device Pin

 * `make DD-GPS-00A0.dtbo`.
 * `cp DD-GPS-00A0.dtbo /lib/firmware/`

### Add the following lines to `/boot/uEnv.txt`

 * Under the line ```###Custom Cape```
   * dtb_overlay=/lib/firmware/DD-GPS-00A0.dtbo
 * Under the line ```###Cape Universal Enable```
   * enable_uboot_cape_universal=1
   * cape_enable=bone_capemgr.enable_partno=DD-GPS
 * Under the line ```###Disable auto loading of virtual capes (emmc/video/wireless/adc)```
   * disable_uboot_overlay_video=1
   * disable_uboot_overlay_audio=1
 * `sudo reboot`
 
Load `pps-gmtimer` LKM on BBB
-------------------------------------

  * Go to build directory
  * ```sudo insmod pps-gmtimer.ko```
  * Check the log messages with ```sudo dmesg -w```

In the case of success the output for dmesg should be something like this:

```
[  175.901727] pps_gmtimer: loading out-of-tree module taints kernel.
[  175.908995] pps-gmtimer: timer name=timer4 rate=24000000Hz
[  175.909064] pps-gmtimer: using system clock
[  175.913184] pps pps0: new PPS source timer4
[  175.913217] clocksource: timer4: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 79635851949 ns
```

Using an external oscillator (TCLKIN) - NOT TESTED
--------------------------------------------------

To use an external clock source on pin P9.41 (TCLKIN).  It accepts up to a 24MHz clock.

 * Apply the patch kernel-tclkin.patch to your kernel
 * If you're not using a 24MHz clock, update the DEFINE\_CLK\_FIXED\_RATE tclkin\_ck definition in arch/arm/mach-omap2/cclock33xx\_data.c 
 * Rebuild your kernel
 * Use the device tree overlay file DD-GPS-TCLKIN-00A0.dtbo, which has the pinctl changes needed

To use this clock as your system time source:

> `echo timer4 > /sys/devices/system/clocksource/clocksource0/current_clocksource`

If you're not using the timer4 hardware, use the other timer's name in place.

To switch back to the default time source:

> `echo gp_timer > /sys/devices/system/clocksource/clocksource0/current_clocksource`


Monitoring operation
--------------------

 * `cat /sys/class/pps/pps0/assert`

The sysfs files in `/sys/devices/platform/ocp/ocp:pps_gmtimer/` contain the counter's current state:

 * `capture` - the counter's raw value at PPS capture
 * `count_at_interrupt` - the counter's value at interrupt time
 * `interrupt_delta` - the value used for the interrupt latency offset
 * `pps_ts` - the final timestamp value sent to the pps system
 * `timer_counter` - the raw counter value
 * `stats` - the number of captures and timer overflows
 * `timer_name` - the name of time timer hardware
 * `ctrlstatus` - the state of the TCLR register (see the AM335x Technical Reference Manual for bit meanings)

Perl script `watch-pps` will watch these files and produce an output that looks like:

 > 1423775690.000 24000010 169 0.000007041 0 3988681035 -0.000001434

The columns are: pps timestamp, capture difference, cycles between capture and interrupt, interrupt\_delta, raw capture value, offset
