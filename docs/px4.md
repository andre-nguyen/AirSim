# PX4 development

The [PX4 software stack](http://github.com/px4/firmware) is an open source flight controller that runs on various 
hardware (see below for a list).  There is a terrific website at [px4.dev.io](http://px4.dev.io) that gives you tons of information about the PX4 stack 
and how to build it.

The following pixhawk hardware has been tested with AirSim:

1. [3DR Pixhawk v2](https://3dr.com/support/pixhawk/)
2. [3DR Pixhawk mini](https://store.3dr.com/products/3dr-pixhawk)
2. [Pixhawk PX4 2.4.8](http://www.banggood.com/Pixhawk-PX4-2_4_8-Flight-Controller-32-Bit-ARM-PX4FMU-PX4IO-Combo-for-Multicopters-p-1040416.html)
3. [PixFalcon](https://hobbyking.com/en_us/pixfalcon-micro-px4-autopilot.html?___store=en_us)
4. [PixRacer](https://www.banggood.com/Pixracer-Autopilot-Xracer-V1_0-Flight-Controller-Mini-PX4-Built-in-Wifi-For-FPV-Racing-RC-Multirotor-p-1056428.html?utm_source=google&utm_medium=cpc_ods&utm_content=starr&utm_campaign=Smlrfpv-ds-FPVracer&gclid=CjwKEAjw9MrIBRCr2LPek5-h8U0SJAD3jfhtbEfqhX4Lu94kPe88Zrr62A5qVgx-wRDBuUulGzHELRoCRVTw_wcB)

The 3DR Pixhawk Mini works out of the box, the others you may need to re-flash with the latest firmware.

## Source code

Getting the PX4 source code is easy:
````
git clone https://github.com/PX4/Firmware.git
cd Firmware
````

Oh, and if you don't have git yet just run this:

````
sudo apt-get install git
````

We are currently testing using the 1.6.0rc1 version, but the latest master branch should be ok too.
Now to build it you will need the right tools.

## PX4 Build tools

The full instructions are available on the [dev.px4.io](http://dev.px4.io/starting-installing-linux.html) website,
but we've copied the relevant subset of those instructions here for your convenience.

(Note that [BashOnWindows](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide)) can be used to build
the SITL version, but not the ARM firmware for pixhawk harddware).

First run this command to cache your admin credentials:
````
sudo ls
````

Now you can block copy/paste the following to a bash terminal and it should run them all to completion, but be sure
to check each command for success:

````
sudo add-apt-repository ppa:george-edison55/cmake-3.x -y
sudo apt-get update
sudo apt-get install python-argparse git-core wget zip \
    python-empy cmake build-essential genromfs -y
sudo apt-get install python-serial openocd \
    flex bison libncurses5-dev autoconf texinfo libftdi-dev libtool zlib1g-dev -y
sudo apt-get install python-pip python-jinja2 -y
````

## Build SITL version

Now you can make the SITL version that runs in posix, from the Firmware folder you created above:
````
make posix_sitl_default
````

Note: this build system is quite special, it knows how to update git submodules (and there's a lot of them),
then it runs cmake (if necessary), then it runs the build itself.  So in a way the root Makefile is a meta-meta makefile :-) 

It shouldn't take long, about 2 minutes.  If all succeeds, the last line will link the `px4` app, which you can then run using the following:

````
./build_posix_sitl_default/src/firmware/posix/px4 ./posix-configs/SITL/init/ekf2/iris
````

And you should see output that looks like this:

````
creating new parameters file
creating new dataman file

______  __   __    ___ 
| ___ \ \ \ / /   /   |
| |_/ /  \ V /   / /| |
|  __/   /   \  / /_| |
| |     / /^\ \ \___  |
\_|     \/   \/     |_/

px4 starting.

18446744073709551615 WARNING: setRealtimeSched failed (not run as root?)
ERROR [param] importing from 'rootfs/eeprom/parameters' failed (-1)
Command 'param' failed, returned 1
  SYS_AUTOSTART: curr: 0 -> new: 4010
  SYS_MC_EST_GROUP: curr: 2 -> new: 1
INFO  [dataman] Unkown restart, data manager file 'rootfs/fs/microsd/dataman' size is 11797680 bytes
  BAT_N_CELLS: curr: 0 -> new: 3
  CAL_GYRO0_ID: curr: 0 -> new: 2293768
  CAL_ACC0_ID: curr: 0 -> new: 1376264
  CAL_ACC1_ID: curr: 0 -> new: 1310728
  CAL_MAG0_ID: curr: 0 -> new: 196616

````

so this is good, first run sets up the px4 parameters for SITL mode.  Second run has less output.
This app is also an interactive console where you can type commands.  Type 'help' to see what they are
and just type ctrl-C to kill it.  You can do that and restart it any time, that's a great way to reset
any wonky state if you need to (it's equivalent to a Pixhawk hardware reboot).

## ARM embedded tools

If you plan to build the PX4 firmware for real Pixhawk hardware then you will need the gcc cross-compiler
for ARM Cortex-M4 chipset.  You can find out what version, if any, you may already have by typing this
command `arm-none-eabi-gcc --version`.  Note: you do not need this to build the SITL version of PX4.

Note: This does `not` work in BashOnWindows because the arm-none-eabi-gcc tool is a 32-bit app which
BashOnWindows cannot run.  There is a more involved set of steps to get a 64 bit version of the 
arm-none-eabi-gcc comp[iler which you will have to find online.

Anyway, first we make sure to remove any old version of arm-none-eabi-gcc: 

````
sudo apt-get remove gcc-arm-none-eabi gdb-arm-none-eabi binutils-arm-none-eabi gcc-arm-embedded -y
sudo add-apt-repository --remove ppa:team-gcc-arm-embedded/ppa -y
````

That previous command prompts you to hit ENTER, so be sure to run that separately before the following:

````
pushd .
cd ~
wget https://launchpad.net/gcc-arm-embedded/5.0/5-2016-q2-update/+download/gcc-arm-none-eabi-5_4-2016q2-20160622-linux.tar.bz2
tar -jxf gcc-arm-none-eabi-5_4-2016q2-20160622-linux.tar.bz2
exportline="export PATH=$HOME/gcc-arm-none-eabi-5_4-2016q2/bin:\$PATH"
if grep -Fxq "$exportline" ~/.profile; then echo nothing to do ; else echo $exportline >> ~/.profile; fi
. ~/.profile
popd
sudo dpkg --add-architecture i386
sudo apt-get update
sudo apt-get install libc6:i386 libgcc1:i386 libstdc++5:i386 libstdc++6:i386 -y
````

So now when you type this command `arm-none-eabi-gcc --version` and you should see:
````
arm-none-eabi-gcc (GNU Tools for ARM Embedded Processors) 5.4.1 20160609 (release) [ARM/embedded-5-branch revision 237715]
Copyright (C) 2015 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
````

## Build PX4 for ARM hardware

Now  you  can build the PX4 firmware for running on real pixhawk hardware:

````
make px4fmu-v2_default
````

This build will take a little longer because it is building a lot more including the NuttX real time OS,
all the drivers for the sensors in the Pixhawk flight controller, and more.  It is also running the compiler
in super size-squeezing mode so it can fit all that in a 1 megabyte ROM !!

One nice tid bit is you can plug in your pixhawk USB, and type `make px4fmu-v2_default upload` to flash the
hardware with these brand new bits, so you don't need to use QGroundControl for that.


## Some Useful Parameters

PX4 has many customizable parameters (over 700 of them, in fact) and to get best results with AirSim we have
found the following parameters are handy:
````
// be sure to enable the new position estimator module:
param set SYS_MC_EST_GROUP 2

// increase default limits on cruise speed so you can move around a large map more quickly.
param MPC_XY_CRUISE 10             
param MPC_XY_VEL_MAX 10
param MPC_Z_VEL_MAX_DN 2

// increase timeout for auto-disarm on landing so that any long running app doesn't have to worry about it
param COM_DISARM_LAND 60

// make it possible to fly without radio control attached (do NOT do this one on a real drone)
param NAV_RCL_ACT 0

// enable new syslogger to get more information from PX4 logs
param set SYS_LOGGER 1
````