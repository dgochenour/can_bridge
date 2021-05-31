(c) Copyright, Real-Time Innovations, 2020.  All rights reserved.
RTI grants Licensee a license to use, modify, compile, and create derivative
works of the software solely for use with RTI Connext DDS. Licensee may
redistribute copies of the software provided that all such copies are subject
to this license. The software is provided "as is", with no warranty of any
type, including any warranty for fitness for any purpose. RTI is under no
obligation to maintain or support the software. RTI shall not be liable for
any incidental or consequential damages arising out of the use or inability
to use the software.

# CAN<-->DDS Bridge Applications

This example contains an implementation of two applications, that together, form a CAN-over-DDS bridge:

Both applications rely on the SocketCAN functionality built into the Linux kernel. More information about SocketCAN can be found in the Linux kernel documentation:
- https://www.kernel.org/doc/Documentation/networking/can.txt

## Assumptions

- Micro 2.4.x libraries are present on the build system
- CMake is available on the build system
- The applications will run on a Linux system
- Physical CAN interfaces and an appropriate driver are present

## Type-Support Overview

A data type based on the Linux kernel's can_frame is defined in can_frame.idl.

For the type to be useable by Connext Micro, type-support files must be generated 
that implement a type-plugin interface.  The type-support files required by this
project are generated automatically when running CMake. If the user wanted to 
change the type in the IDL file and then regenerate files, CMake will notice the
change and the files will be regenerated when compiling the project:

The generated source files are can_frame.c, can_frameSupport.c, and 
can_framePlugin.c. Associated header files are also generated.

## Files Overview

### Applications

can_to_dds.c:
- An application that reads from a CAN socket and publishes data from the CAN 
frame to a DDS topic

dds_to_can.c:
- An application that reads from a DDS Topic and publishes data from the DDS 
sample as a CAN frame over a CAN interface

### Generated files

can_framePlugin.c:
- This file creates the plugin for the can_frame data type.  This file contains 
the code for serializing and deserializing the can_frame type, creating, 
copying, printing and deleting the can_frame type, determining the size of the 
serialized type, and handling hashing a key, and creating the plug-in.

can_frameSupport.c
- This file defines the can_frame type and its typed DataWriter, DataReader, and 
Sequence.

can_frame.c
- This file contains the APIs for managing the can_frame type. 


## DPSE Discovery Configuration

This example makes use of Micro's Dynamic Participant Static Endpoint (DPSE) 
discovery plugin. This choice was made based on the fact that safety-certified 
libraries generated from the RTI Connext Cert product only implement the static 
DPSE discovery plugin, so DPDE is not an option. Because many CAN applications
are part of an automotive system, it is assumed that Cert libraries would be
required.

DPSE requires all DomainParticipants to have names unique to the global dataspace
(i.e. - the Domain) and each endpoint (DataWriter or DataReader) must have a 
similarly unique RTPS Object ID. All of these values are centrally defined in 
the "discovery_constants.h" file.

## Additional configuration

Note that there a few variables at the top of main() in both the can_to_dds and
dds_to_can code that may need to be changed to match your system:

    // user-configurable values
    char *PEER = "239.255.0.1";
    char *LOOPBACK_NAME = "lo";
    char *ETH_NIC_NAME = "wlan0";

By default, the remote peer is set to the well-known multicast group used by DDS
(allowing the examples to discover each other on the same or different machines)
and DDS Domain 0 is used. The names of the network interfaces should match the actual
host where the example is running.

# Compiling w/ cmake

The following build-time assumptions are made:

- The environment variable RTIMEHOME or NDDSHOME is set to the Connext Micro installation 
directory. 
- Micro libraries exist in your installation for the architecture in question. 
If you are unsure if this is the case, please consult the product documentation.
    - https://community.rti.com/static/documentation/connext-micro/2.4.12/doc/html/usersmanual/index.html
- If the example is being built on x64 for other architectures, such as for the 
Raspberry Pi, the appropriate toolchain should be installed. Cross-compilation
is beyond the scope of this readme.

## On the Linux build machine

    $ cd your\project\directory 
    $ mkdir build
    $ cd build

For Micro 2:
    $ cmake .. -DCONNEXTDDSMICRO_ARCH=<your architecture> -DRTI_CONNEXT_SDK=micro2 && cmake --build . --config Release

For Pro:
    $ cmake .. -DCONNEXTDDS_ARCH=<your architecture> -DRTI_CONNEXT_SDK=pro && cmake --build . --config Release

The executables can be found in the *build* directory

# Running can_to_dds and dds_to_can

There are many configurations in which this example code could be run, but what 
is descibed below is a two-machine scenario, each with at least one CAN interface
and both attached to the same IP network.

## Ubuntu 20.04 machine

There are two applications running on the Ubuntu machine
- the cangen utility, which will be used to generate random CAN traffic
- the can_to_dds executable, which receives CAN and publishes DDS

## Raspberry Pi 4

There are two applications running on the RPi4 machine
- the candump utility, which will be used to listen to the local CAN network and
dump packets to stdout
- the dds_to_can executable, which receives DDS samples and publishes to CAN

The Raspberry Pi has 2 physical CAN interfaces, can0 and can1, which we are 
connecting together with wires. The dds_to_can application will write to can0, 
and because can0 and can1 are wired together, the candump utility can read the 
traffic from can1.

## STEP-BY-STEP
1) In a new shell on the Ubuntu machine, navigate to the project directory and
start the can_to_dds executable
    - $ cd path/to/your/project
    - $ ./build/can_to_dds <domain_id> <can_interface_name>

2) In a new shell on the Raspberry Pi, start the dds_to_can executable from 
wherever it resides
    - $ cd path/to/executable
    - $ ./dds_to_can <domain_id> <can_interface_name>

3) On the Raspberry Pi, open a 2nd shell and start the candump utility
    - $ candump can1

4) Back on the Ubuntu machine, open a new shell and start generating CAN traffic
with the cangen utility. The following command generates 20 random CAN frames on 
the vcan0 interface, where the can_to_dds application is listening.
    - $ cangen vcan0 -v -n 20

5) You should now see CAN traffic displayed on the candump shell on the 
Raspberry Pi!

# Other Useful Information

## Installing can-utils

The can-utils package provides several useful utilities, including applications
to generate and consume CAN traffic. On Ubuntu, can-utils can be installed
using apt-get:

    $ sudo apt-get install can-utils

Alternatively, can-utils may be cloned from github and built any platform.

https://github.com/linux-can/can-utils

## Raspberry Pi CAN shield information

The Seeed "2 Channel CAN BUS FD Shield for Raspberry Pi" was used to test the 
functionality of this example. Information about this product can be found at the
links below:
- https://www.seeedstudio.com/2-Channel-CAN-BUS-FD-Shield-for-Raspberry-Pi-p-4072.html
- https://wiki.seeedstudio.com/2-Channel-CAN-BUS-FD-Shield-for-Raspberry-Pi/

## Starting CAN interfaces on the Raspberry Pi

If CAN interfaces are not yet setup on your RPi, the following commands will 
bring them up:

    $ sudo ip link set can0 up type can bitrate 1000000 
    $ sudo ip link set can1 up type can bitrate 1000000 
    
    $ sudo ifconfig can0 txqueuelen 65536
    $ sudo ifconfig can1 txqueuelen 65536

## Starting a vcan interface on Ubuntu 20.04

If the vcan interface is not yet setup on your Ubuntu machine, the following 
commands will bring them up:

    $ sudo ip link add dev vcan0 type vcan 
    $ sudo ifconfig vcan0 up