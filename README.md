### Notes

## Debugging kernels with kgdb

# Introduction

[kgdb](https://www.kernel.org/doc/html/v4.15/dev-tools/kgdb.html) is a kernel
feature that allows an external debugger to connect to a remote system (usually
via a serial port) for debugging of the kernel. It is not so useful for
debugging purely userspace programs for which gdb is the better tool.

IBM's latest POWER9 based OpenPOWER machines all
run [OpenBMC](https://www.openbmc.org/) which makes kernel debugging with kgdb
staight forward as they contain an inbuilt serial console server.

# Preparing the host/target system

Before you can connect to a system with gdb you first need to setup kgdb and
enter the debugger. Unlike debugging a userspace process a kernel debugger
allows the user to gain full access to the system, so all the commands described
below must be run as root.

First we need to tell the kernel which serial port to use for debugging. For
OpenPOWER servers this can be set with:

`echo hvc0 > /sys/module/kgdboc/parameters/kgdboc`

You should see a message similar to this on the console:

`[ 3066.411288] KGDB: Registered I/O driver kgdboc`

It is also possible to set this with the kernel boot argument `kgdboc`. If the
above file does not exist it's likely your kernel has not been built with the
required options. Refer to
the
[kgdb documentation](https://www.kernel.org/doc/html/latest/dev-tools/kgdb.html)
for details on which CONFIG_* options are required. Some distros (or at least
Ubuntu) have these enabled by default.

# Entering the debugger

Entering the debugger requires sending a SysRq key sequence. Whilst it is
possible to send this using a console break sequence it can also be sent via
/proc.

Some distros (notably Ubuntu) may have this method disabled, so first enable it
with:

```echo 1  > /proc/sys/kernel/sysrq```

And enter the kdb monitor with:

```echo g > /proc/sysrq-trigger```

You should see something similar to the following on the host console:

```[78263.190437] sysrq: SysRq : DEBUG
[78266.868334] KGDB: Timed out waiting for secondary CPUs.

Entering kdb (current=0x00000000f8591879, pid 3421) on processor 44 due to Keyboard Entry
[44]kdb> ```

The kdb debugger/monitor provides many commands which can be useful for
debugging. The `help` command will list all of them however to use gdb with the
kernel you will need to enter kgdb. This can be done by running the `kgdb`
command:

```[44]kdb> kgdb
Entering please attach debugger or use $D#44+ or $3#33
```

At this point we are ready to attach our remote gdb instance to the kernel.
Alternatively type `$3#33` to return to the kdb monitor.

# Attaching gdb to the running kernel

A second machine with a gdb installed that supports PowerPC. The easiest way of
doing this is by installing gdb on another PowerPC, however most distributions
also provide versions of gdb for x86 that include PowerPC support. Ubuntu for
example provides the gdb-multiarch package and other distributions may have
similar packages.

You will also need a copy of the kernel binary running on the host, ideally
including debug symbols. If you have built the kernel yourself make sure you
have selected CONFIG_DEBUG_INFO. The kgdb documentation contains more
information on other useful configuration options for kernel debug.

Distribution kernels are not usually built with debug symbols, however versions
with debug symbols are often available as seperate packages. For example Ubuntu
ships them in linux-image-<version>-dbgsym.ddeb files. These don't usually need
to be installed with the package manager and can instead be extracted directly
if desired. For example:

```$ mkdir linux-image-dbgsym
$ cd linux-image-dbgsym
$ ar x ../linux-image-unsigned-4.15.0-33-generic-dbgsym_4.15.0-33.36_ppc64el.ddeb
$ tar -Jxvf data.tar.xz```

Next run gdb against the appropriate kernel binary:

```$ gdb-multiarch usr/lib/debug/boot/vmlinux-4.15.0-33-generic
GNU gdb (Ubuntu 8.0.1-0ubuntu1) 8.0.1
Copyright (C) 2017 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from usr/lib/debug/boot/vmlinux-4.15.0-33-generic...done.
(gdb) ```

To connect to the host we can make use of the OpenBMC serial console server
available on port 2200 like so:

```(gdb) target remote | sshpass -p <OpenBMC Root Password> ssh -p 2200 root@<bmc address>
Remote debugging using | sshpass -p <OpenBMC Root Password> ssh -p 2200 root@<bmc address>
Pseudo-terminal will not be allocated because stdin is not a terminal.
warning: remote target does not support file transfer, attempting to access files from local filesystem.
warning: Unable to find dynamic linker breakpoint function.
GDB will be unable to debug shared library initializers
and track explicitly loaded dynamic code.
0xc0000000001ed850 in arch_kgdb_breakpoint () at /home/alistair/Source/linux/kernel/debug/debug_core.c:1071
1071            wmb(); /* Sync point before breakpoint */
(gdb)```

From here all the standard gdb commands should work:

```(gdb) bt
#0  0xc0000000001ed850 in arch_kgdb_breakpoint () at /home/alistair/Source/linux/kernel/debug/debug_core.c:1071
#1  kgdb_breakpoint () at /home/alistair/Source/linux/kernel/debug/debug_core.c:1072
#2  0xc00000000065d044 in __handle_sysrq (key=103, check_mask=false) at /home/alistair/Source/linux/drivers/tty/sysrq.c:560
#3  0xc00000000065d818 in write_sysrq_trigger (file=<optimized out>, buf=<optimized out>, count=<optimized out>, ppos=<optimized out>)
    at /home/alistair/Source/linux/drivers/tty/sysrq.c:1105
#4  0xc0000000003f0744 in proc_reg_write (file=<optimized out>, buf=<optimized out>, count=<optimized out>, ppos=<optimized out>)
    at /home/alistair/Source/linux/fs/proc/inode.c:243
#5  0xc00000000035a79c in __vfs_write (file=<optimized out>, p=<optimized out>, count=<optimized out>, pos=<optimized out>)
    at /home/alistair/Source/linux/fs/read_write.c:485
#6  0xc00000000035ab98 in vfs_write (file=0xc000001fe97a1900, buf=0x200935e0 "g\n\t ", count=2, pos=0xc00000001a117e00)
    at /home/alistair/Source/linux/fs/read_write.c:549
#7  0xc00000000035af04 in ksys_write (fd=<optimized out>, buf=0x200935e0 "g\n\t ", count=2)
    at /home/alistair/Source/linux/fs/read_write.c:598
#8  0xc00000000000b9e4 in system_call () at /home/alistair/Source/linux/arch/powerpc/kernel/entry_64.S:193
Backtrace stopped: previous frame inner to this frame (corrupt stack?)
(gdb) info reg
r0             0xc00000000065d044       13835058055288836164
r1             0xc00000001a117c00       13835058055719517184
r2             0xc000000001242200       13835058055301308928
r3             0x13     19
r4             0xc000001ff819c9e0       13835058192588589536
r5             0xc000001ff81b3d68       13835058192588684648
r6             0x9000000000009033       10376293541461659699
r7             0x8      8
r8             0xc000000001322200       13835058055302226432
r9             0xc00000000131ea94       13835058055302212244
r10            0x1      1
r11            0xff000000       4278190080
r12            0x2000   8192
r13            0xc0000000013e0000       13835058055303004160
r14            0x0      0
r15            0x0      0
r16            0x0      0
r17            0x0      0
r18            0x0      0
r19            0x100a4068       269107304
r20            0x200934a5       537474213
r21            0x100a4028       269107240
r22            0x20090440       537461824
r23            0x0      0
r24            0x100e0fd8       269357016
r25            0x20090408       537461768
r26            0xc0000000011ae948       13835058055300704584
r27            0xa      10
r28            0xc00000000116af30       13835058055300427568
r29            0x67     103
r30            0xc0000000011611d0       13835058055300387280
r31            0x0      0
pc             0xc0000000001ed850       0xc0000000001ed850 <kgdb_breakpoint+48>
msr            0x9000000000029033       10376293541461790771
cr             0x28002222       671097378
lr             0xc00000000065d044       0xc00000000065d044 <__handle_sysrq+228>
ctr            0x627020 6451232
xer            0x0      0
(gdb)
```

The next post will introduce
[pdbg][https://github.com/open-power/pdbg], a low level PowerPC
debugger that runs independently on the BMC to allow debugging of host
system software regardless of system state.
