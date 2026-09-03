# Hacking BrainCo Revo2 Dexterous Hands onto a Unitree R1 from PC2

> A hands-on retrofit/build log for bringing BrainCo Revo2 dexterous
> hands to life on a Unitree R1 using the robot's PC2 computer.

## The Retrofit

This build log shows how we retrofitted and brought BrainCo Revo2
dexterous hands to life on a Unitree R1 using the robot's PC2 computer.

The interesting part of this retrofit is that much of the software was
already hiding on PC2. Instead of building a new stack from scratch, we
identified the USB-to-serial hardware, verified Linux serial access,
found the existing BrainCo/Unitree source code and prebuilt binaries,
adapted and rebuilt the service for the serial devices on this
particular R1, and then used Unitree DDS to control the hands.

Once everything was working, the hacked-together communication path
looked roughly like this:

``` text
BrainCo Revo2 hands
        ↓
USB / serial interface
        ↓
PC2
        ↓
BrainCo hand service
        ↓
Unitree DDS
        ↓
Control application
```

The BrainCo service creates separate DDS command and state topics for
the left and right hands:

``` text
rt/brainco/left/cmd
rt/brainco/left/state
rt/brainco/right/cmd
rt/brainco/right/state
```

Each hand exposes six normalized controls:

    Index Control
  ------- -----------------
        0 Thumb
        1 Thumb auxiliary
        2 Index
        3 Middle
        4 Ring
        5 Pinky

Finger position and speed are represented on a normalized `0.0–1.0`
scale.

------------------------------------------------------------------------

## Table of Contents

1.  [Get Inside the R1: SSH into PC2](#1-get-inside-the-r1-ssh-into-pc2)
2.  [First Check: Can PC2 See the
    Hands?](#2-first-check-can-pc2-see-the-hands)
3.  [Trace the Hardware at the USB
    Level](#3-trace-the-hardware-at-the-usb-level)
4.  [Unlock Serial-Port Access for the unitree
    User](#4-unlock-serial-port-access-for-the-unitree-user)
5.  [Dig Through PC2 for Existing BrainCo
    Code](#5-dig-through-pc2-for-existing-brainco-code)
6.  [Find Unitree's BrainCo Hand
    Bridge](#6-find-unitrees-brainco-hand-bridge)
7.  [Decode the Hand IDs and Serial
    Settings](#7-decode-the-hand-ids-and-serial-settings)
8.  [The Key Hack: Fix the Serial-Port
    Mismatch](#8-the-key-hack-fix-the-serial-port-mismatch)
9.  [Patch the Source Directly on
    PC2](#9-patch-the-source-directly-on-pc2)
10. [Rebuild the Modified BrainCo
    Service](#10-rebuild-the-modified-brainco-service)
11. [Find the R1's Actual DDS Network
    Interface](#11-find-the-r1s-actual-dds-network-interface)
12. [Bring the Serial-to-DDS Bridge
    Online](#12-bring-the-serial-to-dds-bridge-online)
13. [Make the Hands Move with Unitree's Built-In
    Test](#13-make-the-hands-move-with-unitrees-built-in-test)
14. [Why the First Two-Hand Command Doesn't
    Work](#14-why-the-first-two-hand-command-doesnt-work)
15. [Get Both BrainCo Hands Moving at the Same
    Time](#15-get-both-brainco-hands-moving-at-the-same-time)
16. [Hack the Test Client for Individual Finger
    Control](#16-hack-the-test-client-for-individual-finger-control)
17. [Build a Safer Single-Finger
    Test](#17-build-a-safer-single-finger-test)
18. [Rebuild and Run the Modified
    Test](#18-rebuild-and-run-the-modified-test)
19. [Retrofit File and Directory
    Reference](#19-retrofit-file-and-directory-reference)
20. [Quick Hack/Retrofit Command
    Reference](#20-quick-hackretrofit-command-reference)
21. [What We Learned from the
    Retrofit](#what-we-learned-from-the-retrofit)
22. [Safety and Modification Note](#safety-and-modification-note)

------------------------------------------------------------------------

## 1. Get Inside the R1: SSH into PC2

The work was performed directly on the R1's PC2 computer over SSH.

Once connected, the shell looked like:

``` bash
unitree@ubuntu:~$
```

Before hacking on the hand-control software, we first made sure Linux
could actually see the hand interface.

------------------------------------------------------------------------

## 2. First Check: Can PC2 See the Hands?

The first check was:

``` bash
ls -l /dev/ttyUSB*
```

Initially this returned:

``` text
ls: cannot access '/dev/ttyUSB*': No such file or directory
```

We also checked both common Linux USB serial device types:

``` bash
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

Nothing appeared.

At that point, the problem was clearly below the SDK/application layer.
There was no point hacking on BrainCo software until Linux could
actually detect the hardware.

------------------------------------------------------------------------

## 3. Trace the Hardware at the USB Level

Run:

``` bash
lsusb
```

Initially PC2 only reported its root hubs:

``` text
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

After reconnecting the hand interface, `lsusb` showed:

``` text
Bus 001 Device 004: ID 0403:6011 Future Technology Devices International, Ltd FT4232H Quad HS USB-UART/FIFO IC
```

This was the first real breakthrough in the retrofit.

PC2 was seeing an **FTDI FT4232H Quad HS USB-UART/FIFO**.

The kernel log confirmed that the FTDI driver recognized it and created
four serial interfaces:

``` text
/dev/ttyUSB0
/dev/ttyUSB1
/dev/ttyUSB2
/dev/ttyUSB3
```

The FT4232H is a four-channel device, which explains why one physical
USB device produced four `/dev/ttyUSB` entries.

After reconnecting it, we confirmed:

``` bash
ls -l /dev/ttyUSB*
```

Output:

``` text
crw-rw---- 1 root dialout 188, 0 /dev/ttyUSB0
crw-rw---- 1 root dialout 188, 1 /dev/ttyUSB1
crw-rw---- 1 root dialout 188, 2 /dev/ttyUSB2
crw-rw---- 1 root dialout 188, 3 /dev/ttyUSB3
```

If you're troubleshooting a similar installation, another useful command
is:

``` bash
sudo dmesg -w
```

Leave that running while unplugging and reconnecting the USB interface.
You should see Linux enumerate the FTDI device and attach the individual
serial interfaces.

------------------------------------------------------------------------

## 4. Unlock Serial-Port Access for the unitree User

Notice the permissions:

``` text
root dialout
```

That means members of the Linux `dialout` group can access the serial
interfaces.

Check your groups:

``` bash
groups
```

Originally, our `unitree` account was not a member of `dialout`.

Add it:

``` bash
sudo usermod -aG dialout unitree
```

Then disconnect the SSH session and reconnect.

Check again:

``` bash
groups
```

We then had:

``` text
unitree adm dialout cdrom sudo audio dip video plugdev render i2c lpadmin gdm sambashare weston-launch gpio
```

At this point the `unitree` account could access:

``` text
/dev/ttyUSB0
/dev/ttyUSB1
/dev/ttyUSB2
/dev/ttyUSB3
```

without changing the serial-device permissions manually.

------------------------------------------------------------------------

## 5. Dig Through PC2 for Existing BrainCo Code

Before installing anything new, we dug through the existing PC2
filesystem to see what Unitree had already left us.

Run:

``` bash
find ~ -maxdepth 4 -type d \( -iname "*brainco*" -o -iname "*revo*" -o -iname "*stark*" \) 2>/dev/null
```

That search revealed something useful: this R1 already had BrainCo
software sitting on PC2.

Two particularly important directories were:

``` text
/home/unitree/stark-serialport-example
/home/unitree/brainco_hand_service
```

The `stark-serialport-example` repository contained multiple BrainCo
examples, including:

``` text
~/stark-serialport-example/python/revo2
~/stark-serialport-example/python/revo2_tactile_grasp
~/stark-serialport-example/python/revo2_can
~/stark-serialport-example/python/revo2_canfd
~/stark-serialport-example/linux/revo2
```

There were also Revo1 and EtherCAT examples.

So on an R1 configured like this one, it is worth digging through PC2
before cloning another SDK or building a new stack. The pieces you need
may already be on the robot.

------------------------------------------------------------------------

## 6. Find Unitree's BrainCo Hand Bridge

The other important directory was:

``` bash
cd ~/brainco_hand_service
```

This contained the source code for the Unitree/BrainCo DDS bridge.

The prebuilt executables were located in:

``` text
~/brainco_hand_service/bin/
```

Check them with:

``` bash
ls -l ~/brainco_hand_service/bin
```

Our installation contained:

``` text
brainco_hand_server
test_brainco_hand_server
```

The first executable is the actual serial-to-DDS hand service.

The second is a prebuilt example client that publishes commands to the
service.

The corresponding source files are:

``` text
~/brainco_hand_service/main.cpp
~/brainco_hand_service/test/test_brainco_hand_server.cpp
```

The build directory is:

``` text
~/brainco_hand_service/build/
```

### Source, Build, and Executables

  ------------------------------------------------------------------------------------------------
  Purpose                             Path
  ----------------------------------- ------------------------------------------------------------
  Main service source                 `~/brainco_hand_service/main.cpp`

  Test client source                  `~/brainco_hand_service/test/test_brainco_hand_server.cpp`

  Build directory                     `~/brainco_hand_service/build/`

  Service executable                  `~/brainco_hand_service/bin/brainco_hand_server`

  Test executable                     `~/brainco_hand_service/bin/test_brainco_hand_server`
  ------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 7. Decode the Hand IDs and Serial Settings

Looking directly at `main.cpp` showed the hand configuration:

``` cpp
constexpr uint8_t L_id = 0x7e;
constexpr uint8_t R_id = 0x7f;
constexpr uint32_t baudrate = 460800;
```

Therefore:

  Setting      Value
  ------------ ----------------
  Left hand    `0x7E` / `126`
  Right hand   `0x7F` / `127`
  Baud rate    `460800`

The service initializes the BrainCo Revo2 hardware using Modbus:

``` cpp
init_cfg(
    StarkHardwareType::STARK_HARDWARE_TYPE_REVO2_BASIC,
    StarkProtocolType::STARK_PROTOCOL_TYPE_MODBUS,
    LogLevel::LOG_LEVEL_ERROR,
    1024
);
```

It then opens a serial connection and requests device information:

``` cpp
DeviceHandler* handle = modbus_open(port.c_str(), baudrate);
DeviceInfo* info = modbus_get_device_info(handle, slave_id);
```

This is useful because the service isn't merely assuming that a serial
port corresponds to a particular hand. It can query the device and check
its SKU.

------------------------------------------------------------------------

## 8. The Key Hack: Fix the Serial-Port Mismatch

This is where the retrofit hit its main software mismatch: the supplied
`main.cpp` was looking for different serial-device names.

Its serial-port discovery code was looking for:

``` text
/dev/ttyCH343USB0
/dev/ttyCH343USB1
```

However, our FTDI interface was creating:

``` text
/dev/ttyUSB0
/dev/ttyUSB1
/dev/ttyUSB2
/dev/ttyUSB3
```

That mismatch was enough to keep the existing service from finding our
hardware.

The original section was:

``` cpp
if (path.rfind("/dev/ttyCH343USB0", 0) == 0)
    ports.push_back(path);

if (path.rfind("/dev/ttyCH343USB1", 0) == 0)
    ports.push_back(path);
```

For this R1, the fix was to change the detection to:

``` cpp
if (path.rfind("/dev/ttyUSB", 0) == 0)
    ports.push_back(path);
```

That small patch lets the service discover and probe all of the FT4232H
serial interfaces on this R1.

> **Important:** This is a hardware/configuration-specific hack, not a
> universal R1 modification.

Do not automatically make the same change on every robot. First run:

``` bash
ls -l /dev/ttyUSB*
```

and:

``` bash
ls -l /dev/ttyCH343USB*
```

to see what your particular PC2 exposes.

------------------------------------------------------------------------

## 9. Patch the Source Directly on PC2

This PC2 image did not have `nano` installed:

``` text
-bash: nano: command not found
```

However, `vi` was available:

``` bash
cd ~/brainco_hand_service
vi main.cpp
```

After making the serial-port change, rebuild the project.

------------------------------------------------------------------------

## 10. Rebuild the Modified BrainCo Service

Go into the existing build directory:

``` bash
cd ~/brainco_hand_service/build
```

Then:

``` bash
make -j6
```

A successful build looked like:

``` text
Scanning dependencies of target brainco_hand_server
[ 25%] Building CXX object CMakeFiles/brainco_hand_server.dir/main.cpp.o
[ 75%] Built target test_brainco_hand_server
[100%] Linking CXX executable ../bin/brainco_hand_server
[100%] Built target brainco_hand_server
```

Notice where CMake places the executable:

``` text
../bin/brainco_hand_server
```

So after compiling, the executable remains:

``` text
~/brainco_hand_service/bin/brainco_hand_server
```

------------------------------------------------------------------------

## 11. Find the R1's Actual DDS Network Interface

With serial detection fixed, the next part of the retrofit was getting
the bridge onto the correct Unitree DDS network interface.

Run:

``` bash
ip -br link
```

Our R1 PC2 showed:

``` text
lo       UNKNOWN
dummy0   DOWN
eth10    UP
docker0  DOWN
```

The important part was:

``` text
eth10 UP
```

The repository README uses `eth0` as an example, but this R1's PC2 was
actually running the relevant link on `eth10`.

That is another place where the stock example and the real robot can
differ, so check the actual R1 instead of assuming the interface name.

------------------------------------------------------------------------

## 12. Bring the Serial-to-DDS Bridge Online

The executable is not in the repository root.

This will therefore fail:

``` bash
cd ~/brainco_hand_service
sudo ./brainco_hand_server
```

Instead:

``` bash
cd ~/brainco_hand_service/bin
```

Then start the service using the correct network interface:

``` bash
sudo ./brainco_hand_server --network eth10
```

At this point, the modified service becomes the bridge between the
retrofitted BrainCo hands and Unitree DDS.

Conceptually:

``` text
BrainCo serial
      ↓
brainco_hand_server
      ↓
Unitree DDS
      ↓
rt/brainco/left/...
rt/brainco/right/...
```

Keep this process running while using a separate terminal for control
commands.

------------------------------------------------------------------------

## 13. Make the Hands Move with Unitree's Built-In Test

Unitree already supplies a test executable:

``` text
~/brainco_hand_service/bin/test_brainco_hand_server
```

Its source is:

``` text
~/brainco_hand_service/test/test_brainco_hand_server.cpp
```

The built-in test publishes commands to:

``` text
rt/brainco/left/cmd
```

or:

``` text
rt/brainco/right/cmd
```

depending on the argument.

To test the left hand:

``` bash
cd ~/brainco_hand_service/bin
sudo ./test_brainco_hand_server left
```

To test the right:

``` bash
sudo ./test_brainco_hand_server right
```

The stock test repeatedly cycles through these positions:

``` cpp
positions = {0, 0, 0, 0, 0, 0};
```

then:

``` cpp
positions = {0, 1, 1, 1, 1, 1};
```

then:

``` cpp
positions = {1, 1, 1, 1, 1, 1};
```

with delays between each command.

That confirmed the retrofit was alive: all six hand DOFs could be
commanded through DDS.

------------------------------------------------------------------------

## 14. Why the First Two-Hand Command Doesn't Work

This command does **not** run both hands:

``` bash
sudo ./test_brainco_hand_server left && right
```

In Bash, `&&` means:

> Run the next shell command after the previous command exits
> successfully.

So Bash interprets `right` as the name of another executable.

Likewise:

``` bash
sudo ./test_brainco_hand_server left &&
sudo ./test_brainco_hand_server right
```

runs them sequentially rather than simultaneously.

------------------------------------------------------------------------

## 15. Get Both BrainCo Hands Moving at the Same Time

To launch both test programs concurrently:

``` bash
cd ~/brainco_hand_service/bin

sudo ./test_brainco_hand_server left &
sudo ./test_brainco_hand_server right &
wait
```

The `&` puts each test process in the background, allowing both to run
at approximately the same time.

The resulting arrangement is:

``` text
                    brainco_hand_server
                            │
              ┌─────────────┴─────────────┐
              │                           │
   rt/brainco/left/cmd          rt/brainco/right/cmd
              ↑                           ↑
      left test process             right test process
```

To stop test processes:

``` bash
sudo pkill -f test_brainco_hand_server
```

------------------------------------------------------------------------

## 16. Hack the Test Client for Individual Finger Control

The test application's six-element array corresponds to:

``` text
[Thumb, Thumb_aux, Index, Middle, Ring, Pinky]
```

Therefore:

  Finger              Array Element
  ----------------- ---------------
  Thumb                           0
  Thumb auxiliary                 1
  Index                           2
  Middle                          3
  Ring                            4
  Pinky                           5

For example:

``` cpp
positions[2] = 1.0f;
```

changes the requested position of the index finger.

However, this:

``` cpp
cmds()[2].q() = 1.0;
```

cannot be entered directly into Bash.

That is C++ source code, not a Linux shell command.

To create custom hand behaviors, modify:

``` text
~/brainco_hand_service/test/test_brainco_hand_server.cpp
```

and rebuild it.

------------------------------------------------------------------------

## 17. Build a Safer Single-Finger Test

For testing individual fingers, it is better to read the current hand
state first and preserve the other five positions.

The service publishes current state through:

``` text
rt/brainco/left/state
rt/brainco/right/state
```

A custom client can read all six current positions:

``` cpp
std::array<float, 6> positions;

for (int i = 0; i < 6; ++i)
{
    positions[i] = lowstate->msg_.states()[i].q();
}
```

Then change only the index:

``` cpp
positions[2] = 1.0f;
```

Publish:

``` cpp
for (int i = 0; i < 6; ++i)
{
    lowcmd->msg_.cmds()[i].q() = positions[i];
}

lowcmd->unlockAndPublish();
```

Then send the opposite endpoint:

``` cpp
positions[2] = 0.0f;
```

This is much better than sending six arbitrary values when the goal is
simply to test one finger.

For initial testing, we also reduced finger speed from:

``` cpp
finger.dq() = 1.0f;
```

to:

``` cpp
finger.dq() = 0.3f;
```

so the first experimental movement could be slower.

------------------------------------------------------------------------

## 18. Rebuild and Run the Modified Test

After changing:

``` text
~/brainco_hand_service/test/test_brainco_hand_server.cpp
```

recompile:

``` bash
cd ~/brainco_hand_service/build
make -j6
```

The updated executable is generated at:

``` text
~/brainco_hand_service/bin/test_brainco_hand_server
```

Then:

``` bash
cd ~/brainco_hand_service/bin
sudo ./test_brainco_hand_server left
```

or:

``` bash
sudo ./test_brainco_hand_server right
```

------------------------------------------------------------------------

## 19. Retrofit File and Directory Reference

For this R1 PC2 installation, these were the most useful locations.

  ------------------------------------------------------------------------------------------------
  Item                                Location
  ----------------------------------- ------------------------------------------------------------
  BrainCo/Unitree service             `~/brainco_hand_service/`

  Main serial-to-DDS service source   `~/brainco_hand_service/main.cpp`

  Prebuilt service executable         `~/brainco_hand_service/bin/brainco_hand_server`

  Built-in control/test source        `~/brainco_hand_service/test/test_brainco_hand_server.cpp`

  Prebuilt control/test executable    `~/brainco_hand_service/bin/test_brainco_hand_server`

  Build directory                     `~/brainco_hand_service/build/`

  Additional BrainCo/Stark examples   `~/stark-serialport-example/`

  Python Revo2 examples               `~/stark-serialport-example/python/revo2/`

  Linux Revo2 examples                `~/stark-serialport-example/linux/revo2/`
  ------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 20. Quick Hack/Retrofit Command Reference

### Check USB hardware

``` bash
lsusb
```

### Check serial interfaces

``` bash
ls -l /dev/ttyUSB*
```

### Watch USB connections live

``` bash
sudo dmesg -w
```

### Check serial permissions

``` bash
groups
```

### Add `unitree` to the serial-access group

``` bash
sudo usermod -aG dialout unitree
```

Reconnect SSH afterward.

### Find existing BrainCo software

``` bash
find ~ -maxdepth 4 -type d \( -iname "*brainco*" -o -iname "*revo*" -o -iname "*stark*" \) 2>/dev/null
```

### Check network interfaces

``` bash
ip -br link
```

### Build the service

``` bash
cd ~/brainco_hand_service/build
make -j6
```

### Start the server on this R1

``` bash
cd ~/brainco_hand_service/bin
sudo ./brainco_hand_server --network eth10
```

### Test left hand

``` bash
sudo ./test_brainco_hand_server left
```

### Test right hand

``` bash
sudo ./test_brainco_hand_server right
```

### Test both concurrently

``` bash
sudo ./test_brainco_hand_server left &
sudo ./test_brainco_hand_server right &
wait
```

### Stop the test applications

``` bash
sudo pkill -f test_brainco_hand_server
```

------------------------------------------------------------------------

## What We Learned from the Retrofit

The biggest lesson from this retrofit was that the BrainCo software
itself was not really the initial problem.

The debugging path was:

``` text
No /dev/ttyUSB devices
        ↓
Check lsusb
        ↓
Reconnect USB interface
        ↓
FTDI FT4232H detected
        ↓
ttyUSB0–ttyUSB3 created
        ↓
Fix dialout permissions
        ↓
Find existing BrainCo software
        ↓
Discover serial-device naming mismatch
        ↓
Rebuild brainco_hand_server
        ↓
Use correct PC2 network interface
        ↓
Start serial-to-DDS bridge
        ↓
Use existing Unitree test client
        ↓
Control left/right BrainCo hands
        ↓
Extend example for individual fingers
```

This hack also reinforced a useful rule: **before building a completely
new control stack, dig through what is already installed on the robot.**

In this case, PC2 already contained both a collection of BrainCo/Stark
serial examples and Unitree's `brainco_hand_service`, including prebuilt
binaries.

Once the hardware interface, Linux permissions, serial-device names, and
DDS network interface were understood, the existing software provided
most of what was needed.

------------------------------------------------------------------------

## Safety and Modification Note

> **Warning:** The commands and source changes above document one
> specific Unitree R1 + BrainCo Revo2 installation. Serial-device names,
> network interfaces, hand models, firmware, and wiring may differ on
> another robot.

Before issuing motion commands:

-   Keep hands and objects clear of the fingers.
-   Verify which physical hand is being addressed.
-   Begin at reduced speed.
-   Do not assume normalized endpoints correspond to the same physical
    direction on every configuration.
-   For custom behaviors, read the current hand state and modify only
    the intended DOF instead of blindly commanding all six finger
    positions.

------------------------------------------------------------------------

## Project Summary

**Robot:** Unitree R1\
**Hands:** BrainCo Revo2\
**Computer:** Unitree R1 PC2\
**USB interface:** FTDI FT4232H Quad HS USB-UART/FIFO\
**Protocol:** Modbus\
**Baud rate:** `460800`\
**Left hand ID:** `0x7E` / `126`\
**Right hand ID:** `0x7F` / `127`\
**DDS interface on this R1:** `eth10`

### DDS Topics

``` text
rt/brainco/left/cmd
rt/brainco/left/state
rt/brainco/right/cmd
rt/brainco/right/state
```

------------------------------------------------------------------------

*This README documents a specific experimental retrofit and is not an
official Unitree or BrainCo integration.*
