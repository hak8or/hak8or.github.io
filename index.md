---
layout: default
---

## Intro
A chromebook showed up in the mail recently just after I noticed it may not be such a good idea to sit around all day at work and then at home. After all, imagine how much better it would be to spend all that time outdoors with a laptop instead of indoors with a desktop! Granted, it's still spent sitting around but better than nothing.

Anyways, after a few days it seemed my chromebook is running out of power a bit sooner than I thought, so I checked out [powertop](https://01.org/powertop/overview) and viola! 

```bash
PowerTOP 2.8      Overview   Idle stats   Frequency stats   Device stats   Tunables
The battery reports a discharge rate of 7.77 W
The estimated remaining time is 4 hours, 21 minutes

Summary: 770.9 wakeups/second,  25.0 GPU ops/seconds, 0.0 VFS ops/sec and 13.5% CPU use

                Usage       Events/s    Category       Description
            100.0%                      Device         Radio device: btusb
              3.3 ms/s     452.9        Interrupt      [7] INT3432:00
             38.8%                      Device         Display backlight
              0.9 ms/s      78.0        Process        [irq/37-ELAN0000]
              4.1 ms/s      69.9        Process        /usr/bin/firefox
             39.4 ms/s      17.9        Process        /usr/bin/konsole
             44.7 ms/s       3.6        kWork          intel_fbc_work_fn
              5.4 ms/s      35.4        Process        /usr/bin/plasmashell --shut-up
.....
=====================================================
          Package   |             Core    |            CPU 0
                    |                     | C0 active  15.1%
                    |                     | POLL        0.0%    0.4 ms
                    |                     | C1E-BDW     2.4%    0.4 ms
C2 (pc2)   66.1%    |                     |
C3 (pc3)    0.0%    | C3 (cc3)    4.8%    | C3-BDW      5.0%    0.5 ms
C6 (pc6)    0.0%    | C6 (cc6)    1.5%    | C6-BDW      1.5%    0.6 ms
C7 (pc7)    0.0%    | C7 (cc7)   67.7%    | C7s-BDW     3.6%    0.6 ms
C8 (pc8)    0.0%    |                     | C8-BDW      5.8%    1.3 ms
C9 (pc9)    0.0%    |                     | C9-BDW     18.0%    3.7 ms
C10 (pc10)  0.0%    |                     | C10-BDW    40.8%   20.1 ms

.....
```

Turns out I am pulling over seven watts and the SOC/Package is never going below the C2 power state. That's wierd, though there was an [article](https://mjg59.dreamwidth.org/41713.html) recently about Skylake CPU's from Intel recently about how power managment for them under Linux was atrocious with the package being unable to hit below the power state C3. For Haswell/Broadwell, it was due to the SATA peripheral on the package not being properly put into a lower power state, preventing the package from also going into a lower power state.

My Toshiba Chromebook 2 has an Intel Celeron 3215U with, hey, that microcode version seems pretty low!
 
```bash
hak8or@hak8or ~> cat /proc/cpuinfo 
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 6
model           : 61
model name      : Intel(R) Celeron(R) CPU 3215U @ 1.70GHz
stepping        : 4
microcode       : 0x1f
cpu MHz         : 799.996
```

## Microcode Update
So first thing to do is find out how to check what the latest microcode avalible is for my processor, maybe that has something to do with it. Good ole stack overflow and its huge sub communities shows [how](http://askubuntu.com/questions/545925/how-to-update-intel-microcode-properly) to do this. So ```iucode-tool``` is first, then using that I can tell if an update is avalible, should be easy. Just have to replace these commands with the arch equivilent and ... oh man.

```bash
hak8or@hak8or ~> yaourt -R iucode-tool
checking dependencies...

Packages (1) iucode-tool-1.6.1-1
......
==> Validating source files with sha256sums...
    iucode-tool_1.6.1.tar.xz ... Passed
    iucode-tool_1.6.1.tar.xz.asc ... Skipped
==> Verifying source file signatures with gpg...
    iucode-tool_1.6.1.tar.xz ... FAILED
==> ERROR: One or more PGP signatures could not be verified!
==> ERROR: Makepkg was unable to build iucode-tool.
==> Restart building iucode-tool ? [y/N]
==> ------------------------------------
==> n

hak8or@hak8or ~> 
```


## Skip GPG check for Yaourt
Well that's a bummer, so it seems that the GPG check is failing for one of our files. Thankfully yaourt [seems](https://github.com/archlinuxfr/yaourt/issues/108) to have a way to skip that using the --m-arg flag which in this case we want to pass to makepkg. So let's try ```yaourt --m-arg "--skipchecksums --skippgpcheck" -Sb iucode-tool``` and, hm, I get ```makepkg: invalid option '--skipchecksums --skippgpcheck'```. Both checksum skipping and gpg skipping flags seem correct based on what makepkg accepts.

```bash
hak8or@hak8or ~> makepkg --help
makepkg (pacman) 5.0.1

Make packages compatible for use with pacman

Usage: /usr/bin/makepkg [options]

Options:
.......
  --sign           Sign the resulting package with gpg
  --skipchecksums  Do not verify checksums of the source files
  --skipinteg      Do not perform any verification checks on source files
  --skippgpcheck   Do not verify source files with PGP signatures
  --verifysource   Download source files (if needed) and perform integrity checks
```

Passing just one flag seems to work though, so lets just pass the flag to skip the GPG check:

```bash
hak8or@hak8or ~> yaourt --m-arg "--skippgpcheck" -Sb iucode-tool
.....
==> WARNING: Skipping verification of source file PGP signatures.
==> Validating source files with sha256sums...
    iucode-tool_1.6.1.tar.xz ... Passed
    iucode-tool_1.6.1.tar.xz.asc ... Skipped
.....
:: Proceed with installation? [Y/n] 
(1/1) checking keys in keyring                              [################################] 100%
(1/1) checking package integrity                            [################################] 100%
(1/1) loading package files                                 [################################] 100%
(1/1) checking for file conflicts                           [################################] 100%
(1/1) checking available disk space                         [################################] 100%
:: Processing package changes...
(1/1) installing iucode-tool                                [################################] 100%
:: Running post-transaction hooks...
(1/1) Updating manpage index...
hak8or@hak8or ~> 
```

## Back to microcode update
Now to get the package [from](https://downloadcenter.intel.com/download/26156/Linux-Processor-Microcode-Data-File?product=84810) Intel for this CPU (seems to be Broadwell which is Intel's 5th generation family). Next up, remembering the flags for extracting a .tgz file! Eventually I read somewhere you can use ```tar -xzf foo.tgz``` and remember the flags as "Extract Ze File", though I am pretty sure there was suppoused to be a j in there somewhere.

Going back to the directions [here](http://askubuntu.com/questions/545925/how-to-update-intel-microcode-properly) I see the output as follows. Note the && symbol which in bash chains commands together but in fish it does something else. 

```bash
hak8or@hak8or /t/micro> iucode_tool -K/tmp/micro microcode.dat
iucode_tool: Writing microcode firmware file(s) into /tmp/micro
hak8or@hak8or /t/micro> modprobe cpuid && iucode_tool -tb -lS /tmp/micro⏎
hak8or@hak8or /t/micro> sudo modprobe cpuid
hak8or@hak8or /t/micro> iucode_tool -tb -lS /tmp/micro
iucode_tool: system has processor(s) with signature 0x000306d4
iucode_tool: microcode bundle /tmp/micro/microcode.dat: unknown microcode format
hak8or@hak8or /t/micro> ls
06-03-02  06-07-02  06-0b-04  06-16-01  06-1e-05  06-3d-04  06-4f-01  0f-02-09  0f-04-0a
06-05-00  06-07-03  06-0d-06  06-17-06  06-25-02  06-3e-04  06-56-02  0f-03-02  0f-06-02
06-05-01  06-08-01  06-0e-08  06-17-07  06-25-05  06-3e-06  06-5e-03  0f-03-03  0f-06-04
06-05-02  06-08-03  06-0e-0c  06-17-0a  06-26-01  06-3e-07  0f-00-07  0f-03-04  0f-06-05
06-05-03  06-08-06  06-0f-02  06-1a-04  06-2a-07  06-3f-02  0f-00-0a  0f-04-01  0f-06-08
06-06-00  06-08-0a  06-0f-06  06-1a-05  06-2d-06  06-3f-04  0f-01-02  0f-04-03  microcode.dat
06-06-05  06-09-05  06-0f-07  06-1c-02  06-2d-07  06-45-01  0f-02-04  0f-04-04
06-06-0a  06-0a-00  06-0f-0a  06-1c-0a  06-2f-02  06-46-01  0f-02-05  0f-04-07
06-06-0d  06-0a-01  06-0f-0b  06-1d-01  06-3a-09  06-47-01  0f-02-06  0f-04-08
06-07-01  06-0b-01  06-0f-0d  06-1e-04  06-3c-03  06-4e-03  0f-02-07  0f-04-09
hak8or@hak8or /t/micro> rm microcode.dat 
hak8or@hak8or /t/micro> iucode_tool -tb -lS /tmp/micro
iucode_tool: system has processor(s) with signature 0x000306d4
selected microcodes:
001: sig 0x000306d4, pf mask 0xc0, 2016-04-29, rev 0x0024, size 17408
hak8or@hak8or /t/micro> 
```

So it seems there is updated microcode avalible for this processor! But it seems that it isn't getting updated during boot as shown by dmesg.

```bash
hak8or@hak8or /t/micro> dmesg | grep "microcode"
[    0.387063] microcode: CPU0 sig=0x306d4, pf=0x40, revision=0x1f
[    0.387085] microcode: CPU1 sig=0x306d4, pf=0x40, revision=0x1f
[    0.387143] microcode: Microcode Update Driver: v2.01 <tigran@aivazian.fsnet.co.uk>, Peter Oruba
```

Thankfully, as always, the Arch Wiki has a fantastic [article](https://wiki.archlinux.org/index.php/microcode) on how to check if you have the latest version and how to setup grub to update it if you don't. The new microcode update should be in ```/boot/intel-ucode.img``` but for me it wasn't. Turns out for some reason I kept skipping over installing the ```intel-ucode``` package which is why I was missing the new version of the microcode. For AMD processors there is ```linux-firmware``` which is installed by default for arch which includes microcode updates, but it seems ```intel-ucode``` for Intel doesn't come out of the box. 

Lastly, when you install the ```intel-ucode``` package, the new microcode update image is stored in ```/boot/intel-ucode.img``` as expected. Doing a ```grub-mkconfig -o /boot/grub/grub.cfg``` will update grub to include the new microcode and now we've got an updated microcode!

```bash
hak8or@hak8or /t/micro> sudo grub-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found theme: /boot/grub/themes/Antergos-Default/theme.txt
Found Intel Microcode image
Found linux image: /boot/vmlinuz-linux
Found initrd image: /boot/initramfs-linux.img
Found fallback initramfs image: /boot/initramfs-linux-fallback.img
done
```

And after restarting, you should see the following:

```bash
hak8or@hak8or ~> dmesg | grep "microcode"
[    0.000000] microcode: microcode updated early to revision 0x24, date = 2016-04-29
[    0.387159] microcode: CPU0 sig=0x306d4, pf=0x40, revision=0x24
[    0.387170] microcode: CPU1 sig=0x306d4, pf=0x40, revision=0x24
[    0.387210] microcode: Microcode Update Driver: v2.01 <tigran@aivazian.fsnet.co.uk>, Peter Oruba
hak8or@hak8or ~> cat /proc/cpuinfo 
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 6
model           : 61
model name      : Intel(R) Celeron(R) CPU 3215U @ 1.70GHz
stepping        : 4
microcode       : 0x24
cpu MHz         : 799.996
```

## Back to power consumption
After this microcode, the power consumption should improve, and powertop says:

```bash
The battery reports a discharge rate of 7.47 W
The estimated remaining time is 1 hours, 41 minutes

Summary: 642.0 wakeups/second,  2.8 GPU ops/seconds, 0.0 VFS ops/sec and 5.0% CPU use

                Usage       Events/s    Category       Description
              1.1 ms/s     140.0        Process        [rcu_preempt]
              1.1 ms/s     137.2        Process        [rcu_sched]
              2.6 ms/s     118.8        Timer          tick_sched_timer
              6.8 ms/s      56.6        Process        powertop
             72.6 µs/s      39.6        Interrupt      [45] iwlwifi
              1.2 ms/s      38.2        Process        [irq/45-iwlwifi]
==========================================================================
          Package   |             Core    |            CPU 0
                    |                     | C0 active   0.8%
                    |                     | POLL        0.0%    0.3 ms
                    |                     | C1E-BDW     0.1%    0.5 ms
C2 (pc2)   93.8%    |                     |
C3 (pc3)    0.0%    | C3 (cc3)    0.7%    | C3-BDW      0.7%    0.6 ms
C6 (pc6)    0.0%    | C6 (cc6)    0.2%    | C6-BDW      0.2%    0.6 ms
C7 (pc7)    0.0%    | C7 (cc7)   93.6%    | C7s-BDW     0.6%    0.3 ms
C8 (pc8)    0.0%    |                     | C8-BDW      0.6%    0.4 ms
C9 (pc9)    0.0%    |                     | C9-BDW     11.6%   10.6 ms
C10 (pc10)  0.0%    |                     | C10-BDW    80.8%  123.2 ms

                    |             Core    |            CPU 1
                    |                     | C0 active   0.4%
                    |                     | POLL        0.0%    0.0 ms
                    |                     | C1E-BDW     0.0%    0.1 ms
                    |                     |
                    | C3 (cc3)    0.1%    | C3-BDW      0.1%    0.4 ms
                    | C6 (cc6)    0.1%    | C6-BDW      0.1%    0.8 ms
                    | C7 (cc7)   99.1%    | C7s-BDW     0.3%    1.2 ms
                    |                     | C8-BDW      0.8%    1.8 ms
```

Well that's unfortunate, though at least the microcode on this is updated to the newest version. Going back to the article about bad power management under linux for Skylake processors, there was a mention about the SATA peripheral preventing the package from dropping into a lower power state. In that case, checking out the Tunables tab of powertop reveals the following:

```bash
>> Bad           NMI watchdog should be turned off
   Bad           VM writeback timeout
   Bad           Enable SATA link power management for host0
   Bad           Enable SATA link power management for host1
   Bad           Enable Audio codec power management
   Bad           Runtime PM for I2C Adapter i2c-4 (i915 gmbus dpb)
   Bad           Runtime PM for I2C Adapter i2c-5 (i915 gmbus dpd)
   Bad           Runtime PM for I2C Adapter i2c-2 (i915 gmbus vga)
   Bad           Runtime PM for I2C Adapter i2c-3 (i915 gmbus dpc)
   Bad           Autosuspend for unknown USB device 1-4 (8087:07dc)
   Bad           Runtime PM for PCI Device Intel Corporation HD Graphics
   Bad           Runtime PM for PCI Device Intel Corporation Broadwell-U Audio Controller
   Bad           Runtime PM for PCI Device Intel Corporation Wildcat Point-LP USB xHCI Controller
   Bad           Runtime PM for PCI Device Intel Corporation Wildcat Point-LP High Definition Audio Controller
   Bad           Runtime PM for PCI Device Intel Corporation Wildcat Point-LP PCI Express Root Port #1
   Bad           Runtime PM for PCI Device Intel Corporation Wildcat Point-LP LPC Controller
   Bad           Runtime PM for PCI Device Intel Corporation Wildcat Point-LP SATA Controller [AHCI Mode]
   Bad           Runtime PM for PCI Device Intel Corporation Wildcat Point-LP Thermal Management Controller
   Bad           Runtime PM for PCI Device Intel Corporation Broadwell-U Host Bridge -OPI
   Good          Bluetooth device interface status
   Good          Autosuspend for USB device Web Camera-HD [1-3]
   Good          Runtime PM for I2C Adapter i2c-0 (Synopsys DesignWare I2C adapter)
   Good          Runtime PM for I2C Adapter i2c-1 (Synopsys DesignWare I2C adapter)
   Good          Autosuspend for USB device xHCI Host Controller [usb1]
   Good          Autosuspend for USB device xHCI Host Controller [usb2]
   Good          Runtime PM for PCI Device Intel Corporation Wireless 7260
   Good          Wake-on-lan status for device wlp1s0
```

Lets first correct ```Enable SATA link power management for host0``` and the same for host1 and see what that does.

```bash
The battery reports a discharge rate of 4.76 W
The estimated remaining time is 2 hours, 30 minutes

Summary: 67.1 wakeups/second,  0.6 GPU ops/seconds, 0.0 VFS ops/sec and 2.1% CPU use

                Usage       Events/s    Category       Description
             38.8%                      Device         Display backlight
            100.0%                      Device         Radio device: btusb
              0.5 pkts/s                Device         Network interface: wlp1s0 (iwlwifi)
             12.7 ms/s       0.9        Process        /usr/lib/udisks2/udisksd --no-debug
            129.9 µs/s      14.5        Process        [rcu_preempt]
            284.0 µs/s      13.2        Timer          tick_sched_timer
             87.3 µs/s      10.6        Process        [rcu_sched]
            545.1 µs/s       5.0        Process        /usr/bin/pamac-tray
==========================================================================
          Package   |             Core    |            CPU 0
                    |                     | C0 active   0.6%
                    |                     | POLL        0.0%    0.0 ms
                    |                     | C1E-BDW     0.1%    0.6 ms
C2 (pc2)    6.5%    |                     |
C3 (pc3)    0.4%    | C3 (cc3)    0.4%    | C3-BDW      0.5%    0.6 ms
C6 (pc6)    0.1%    | C6 (cc6)    0.1%    | C6-BDW      0.2%    0.5 ms
C7 (pc7)   88.7%    | C7 (cc7)   96.4%    | C7s-BDW     0.5%    0.6 ms
C8 (pc8)    0.0%    |                     | C8-BDW      2.7%    2.8 ms
C9 (pc9)    0.0%    |                     | C9-BDW      5.0%    2.9 ms
C10 (pc10)  0.0%    |                     | C10-BDW    88.2%   72.2 ms

                    |             Core    |            CPU 1
                    |                     | C0 active   0.4%
                    |                     | POLL        0.0%    0.0 ms
                    |                     | C1E-BDW     0.1%    0.8 ms
                    |                     |
                    | C3 (cc3)    0.2%    | C3-BDW      0.2%    0.4 ms
                    | C6 (cc6)    0.1%    | C6-BDW      0.1%    0.6 ms
                    | C7 (cc7)   98.9%    | C7s-BDW     0.2%    0.8 ms
                    |                     | C8-BDW      1.3%    2.1 ms
                    |                     | C9-BDW     13.4%    7.5 ms
                    |                     | C10-BDW    84.2%  105.1 ms
```

And there it is, we are now hitting the C7 power state for the package, with power consumption dropped from ~7.5 watts down to ~4.75 watts, saving us a decent chunk of power! The battery in the chromebook is roughly 45 watthours, so ~6 hours earlier or ~9.5 hours with this power dip. And my screen backlight is set to be pretty bright, so if I drop it to as low as KDE will allow me (seems to be fully off) that's a drain of 3.1 watts in powertop giving me 14.5 hours. Putting the backlight down to a comfortable but dark point results in 3.5 watts drain giving me 12.8 hours.

## Conclusion.
In short, I did the following:

1. Make sure the microcode gets updated upon boot (may not be needed, but good to have regardless).
    - Requires installing the ```intel-ucode``` package and updating grub to use the new microcode image.
2. Change tunables in powertop for SATA link power management.

With the above changes, I was able to get my power draw down from roughly 7.5 watts to 4.75 watts with the backlight to moderatly high brightness. Changing the backlight to a bit dimmer but still totally usable level drops that to 3.5 watts, giving a theoretical screen on time while idle of roughly 12.8 hours.

After spending some time with this though, I get corrupted data on my SSD. Looking further, turns out there is an in depth [article](https://lwn.net/Articles/642065/) on the state of different power consumption states for SATA in Broadwell and Haswell chips. Long story short, there seems to be a disconnect between Intel and Linux maintainers for how to handle these power states, resulting in some combinations of hardware (this chromebook being an example) causing data corruption. I have had one instance of my system hanging for a few seconds only to see corrupted blocks/nodes that ext4 tried to clean up in dmesg.

If you don't mind living on the edge (you do keep backups, right?) then feel free to change the SATA link states in powertop to "Good". If not, then you will lose out on some battery life as the package never enters into a power savings mode of C3 or up. Hopefully this issue gets resolved sometime soon, since all linux based laptops will have this issue unless the user manually fiddles with it in powertop.

