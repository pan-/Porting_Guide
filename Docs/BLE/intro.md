# BLE Porting guide

Description of ble in mbed.
Explain that it work on mbed-os and mbed-classic.

## Table of Content

* [Overview](#Overview)
    * Port as a yotta module/ mbed classic library. Explain where BLE fit in
    mbed-os and mbed-classic.
    * Description of BLE repository
    * Description of the interfaces and main modules
    * Explanation of our completion/callback model
    * Difference between mbed-os VS mbed-classic. Explain things about minar,
    interrupt context, main context, ...
    * Coding rules
    * link to BLE_API doc
* Step by Step
    * Prerequisites: mbed os target, yotta, git, compiler, ble module board, ...
    * Create the yotta module of the port of BLE_API
    * Generate the module skeleton
    * Implementing modules step by step
        * Overview
        * BLEInstanceBase
        * Gap
        * GattServer
        * GattClient
        * SecurityManager
* Debugging
* Testing
* Validation
* FAQ


## Overview

### BLE_API status
BLE_API is a Bluetooth Low Energy abstraction. It work on mbed-os and mbed-classic.
BLE_API repository is mainly hosted on [Github](https://github.com/ARMmbed/ble)
and [mirrored](https://developer.mbed.org/teams/Bluetooth-Low-Energy/code/BLE_API/)
on mbed-classic website.

Since BLE_API is just a BLE abstraction it requires an implementation to be useful.
Abstractions of BLE_API are provided by vendor, and this guide is all about how to
write such abstraction.

It is expected that a vendor implementation is distributed as a [yotta](http://yottadocs.mbed.com/)
[module](http://yottadocs.mbed.com/tutorial/tutorial.html) for mbed-os and as a
library for mbed-classic. To do so, a vendor just has to push it's implementation
as a yotta module into the registry and publish a library for mbed-classic.

### BLE_API repository
A port of BLE_API is, normally, not used directly by the end user. It is expected
that mbed-os and mbed-classic users wrote their ble application using BLE_API and
nothing else. To make this works, it is required that vendors provide implementation
for certain parts of BLE_API.

The BLE_API repository contains two folders:
* [`ble`](https://github.com/ARMmbed/ble/tree/master/ble): This folder contains
all headers file accessible to the user which want to build a bluetooth low energy
application with mbed. It contains all abstractions necessary to such task.
* [`source`](https://github.com/ARMmbed/ble/tree/master/source): This folder
contains implementation of platform independent parts of BLE_API.

The folder ble contains all declarations of types, functions and classes provided
by BLE_API, a porter does not have to provide an implementation for all of this
files but it is important to understand what is available here before starting the
porting effort.
* [`BLE.h`]: This file is the entry point to the whole BLE_API. It contains the
class [`BLE`] which provide access to all functionalities of BLE_API.
* [`BLEInstanceBase.h`]:Contains the definition of the class [`BLEInstanceBase`],
an interface which manage the vendor BLE stack and provide access to other interfaces
of BLE_API. [`BLEInstanceBase`] can be seen as a private implementation of the class [`BLE`].
* [`Gap.h`]: Contains the declaration of the class [`Gap`] which manage all operations
related to the GAP layer.
* [`GattServer.h`]: Contains the declaration of the class [`GattServer`] which provide
primitives to create a GATT server and interacts with its characteristics.
* [`GattClient.h`]: Contains the class [`GattClient`] which provide primitives to discover
and interract with a distant GATT server.
* [`SecurityManager.h`]: Contains the class [`SecurityManager`] which provide
functionalities of the Security Manager layer of Bluetooth low energy.
* [`UUID.h`]: Provide the class definition of the class [`UUID`] which abstract 16 bits
and 128 bits UUID.
* [`FunctionPointerWithContext.h`]: Define the class[`FunctionPointerWithContext`]
which is used as the callback abstraction of BLE_API.
* [`CallChainOfFunctionPointersWithContext.h`]: Define the class
[`CallChainOfFunctionPointersWithContext`] which is used allow a user to store
multiple callbacks for an event.   
* [`GapAdvertisingData.h`]: Define the class [`GapAdvertisingData`] which ease the
construction of GAP advertising and scan response data payload.
* [`GapAdvertisingParams.h`]: Define the class [`GapAdvertisingParams`] which
facilitate the construction of GAP advertising parameters.
* [`GapAdvertisingParams.h`]: Define the class [`GapAdvertisingParams`] which
facilitate the construction of GAP advertising parameters for a Broadcaster device.
* [`GapEvents.h`]: Define the class [`GapEvents`] which contains the definition
of some events related to the GAP layer.
* [`GapScanningParams.h`]: Define the class [`GapScanningParams`] which
facilitate the construction of GAP scanning parameters for an Observer device.
* [`GattCharacteristic.h`]: Define the class [`GattCharacteristic`] which model a
GATT characteristic used by a [`GattServer`].
* [`GattAttribute.h`]: Define the class [`GattAttribute`] which model a GATT
attribute used by a [`GattServer`] or a [`GattClient`].
* [`GattService.h`]: Define the class [`GattService`] which model a GATT service
used by a [`GattServer`].
* [`GattCharacteristic.h`]: Define the class [`GattCharacteristic`] which model a
GATT characteristic used by a [`GattServer`].
* [`DiscoveredService.h`]: Define the class [`DiscoveredService`] which represent
service discovered by a [`GattClient`].
* [`DiscoveredCharacteristic.h`]: Define the class [`DiscoveredCharacteristic`]
which represent a characteristic discovered by a [`GattClient`].
* [`DiscoveredCharacteristicDescriptor.h`]: Define the class
[`DiscoveredCharacteristicDescriptor`] which represent a descriptor of a characteristic
discovered by a [`GattClient`] discovery procedure.
* [`ServiceDiscovery.h`]: **Should not be here.** The class [`ServiceDiscovery`]
can be used as a starting point for implementation of service discovery.
* [`CharacteristicDescriptorDiscovery.h`]: The class [`CharacteristicDescriptorDiscovery`]
contains the types of the values passed to the callbacks of the characteristic
descriptor discovery procedure.
* [`GattCallbackParamTypes.h`]: Contains various types used by [`GattClient`] and
[`GattServer`] callbacks.
* [`GattServer.h`]: Contains definition of some events used by [`GattServer`].
* [`BLEProtocol.h`]: Some definitions related to the BLE protocol.
* [`ble-common.h`]: Contains definitions which are shared in multiple bits of BLE_API.
* [`SafeBool.h`]: Contain the utility class [`SafeBool`] which implement the safe bool
idiom in c++03.
* [`deprecate.h`]: Provide a way to deprecate bits of API.


### Architecture

The main entry point of BLE_API is the class [`BLE`], it allow the user to initialize/
shutdown the [`BLE`] stack and also provide accessors to the classes which models
a layer of the Bluetooth protocol.

[`BLE`] class use an interface to a private implementation named [`BLEInstanceBase`]
to provide all those functions.

It is expected that a port of BLE_API provide implementations for the following
classes.

* [`BLEInstanceBase`]: Manage the ble stack, register to access instances of
implementations of other other abstraction.
* [`Gap`] Model the Gap layer
* [`GattServer`] contains primitives to build and manage a GattServer
* [`GattClient`] contains primitives to discover and interact with a distant GATT
server.
* [`SecurityManager`] security manager layer.

This can be summarized by the following diagram:

```
                               +---------------------------------------+
                               |         BLEInstanceBase               |
                               +---------------------------------------+
                               |  Gap gap()= 0                         |
              +----------------+  GattServer gattServer()= 0           <--------------+
              |                |  GattClient gattClient()= 0           |              |
              |                |  SecurityManager securityManager()= 0 |              |
              |                +---------------------------------------+              |
              |                                                                       |
              |                                                                       |
              |                                                                       |
+-------------+----------------------+                         +----------------------+-------------------+
|                BLE                 |                         |        PortXXXBleInstance                |
+------------------------------------+                         +------------------------------------------+
|  Gap gap()                         |                         | PortXXXGap gap()                         +----------+
|  GattServer gattServer()           |                         | PortXXXGattServer gattServer()           +--------+ |
|  GattClient gattClient()           |                         | PortXXXGattClient gattClient()           +------+ | |
|  SecurityManager securityManager() |                         | PortXXXSecurityManager securityManager() +----+ | | |
+------------------------------------+                         +------------------------------------------+    | | | |
                                                                                                               | | | |
+------------------------------------+                         +------------------------------------------+    | | | |
|              Gap                   <-------------------------+         PortXXXGap                       +----+ | | |
+------------------------------------+                         +------------------------------------------+      | | |
                                                                                                                 | | |
+------------------------------------+                         +------------------------------------------+      | | |
|          GattServer                <-------------------------+         PortXXXGattServer                +------+ | |
+------------------------------------+                         +------------------------------------------+        | |
                                                                                                                   | |
+------------------------------------+                         +------------------------------------------+        | |
|           GattClient               <-------------------------+         PortXXXGattClient                +--------+ |
+------------------------------------+                         +------------------------------------------+          |
                                                                                                                     |
+------------------------------------+                         +------------------------------------------+          |
|          SecurityManager           <-------------------------+         PortXXXSecurityManager           +----------+
+------------------------------------+                         +------------------------------------------+
```


### Callback model
All asynchronous operations in BLE_API will call at some point a completion handler
also named callback. BLE_API does not use raw function pointer as callbacks. Instead,
BLE_API, use the class template [`FunctionPointerWithContext`] to model its completion
handler.
It plays nicely with c++ and classes, it allow for instance to create a callback
from an instance of an object and a member function. Raw function pointers are
also supported by this abstraction.

It is important to note that all instances of [`FunctionPointerWithContext`] accept
a single argument as a result of asynchronous operation. This means that if an
asynchronous operation produce several values as a result, this results will be
packed in a structure.

The observer pattern is massively used in BLE_API It to observe recurring event
sources to be observed. The class [`CallChainOfFunctionPointersWithContext`] is
used across the API to store a list of [`FunctionPointerWithContext`] as observers
and fire notification to them.

Execution of BLE_API callbacks does not happen in the same context if the user
run is application in mbed-os or in mbed-classic.
* When mbed-os is used, it is expected that all callbacks run in thread context.
The porter has to take care of this.
* When mbed-classic is used, it is expected that all callbacks runs in irq context.


### Minar
mbed-os use a scheduler named [minar](https://github.com/ARMmbed/minar) to defer
work from IRQ context to thread context. Most ble stacks came with some kind of
event handler/process function. Porter of BLE_API needs to take care that if their  
port run on mbed-os, event handling should happen in thread context.

To do so, porters just have to wrap their event handling function in a function
which defer process handling via minar if the port is built for mbed-os.

```c++
// Detect if the file is built for mbed-os
#ifdef YOTTA_CFG_MBED_OS
#include <minar/minar.h>
#endif

// function of the BLE stack which handle/process all events
void XXX_process_ble_stack_events();


// function which wrap the event handler, this function will be used by the port
void process_ble_stack_events() {
#ifdef YOTTA_CFG_MBED_OS
    // Defer execution of `XXX_process_ble_stack_events` to thread context.
    minar::Scheduler::postCallback(XXX_process_ble_stack_events);
#else
    // mbed-classic, just execute the event handling function as is.
    XXX_process_ble_stack_events();
#endif
}
```

### Error handling

### TODO: coding rules

## Step by step

### Prerequisites
mbed os target, yotta, git, compiler, ble module board, ...

### yotta module
How to create the yotta module of the port of BLE API

### module skeleton
TODO: provide skeleton of the module

### Step by step implementation

Overview....

#### [`BleInstanceBase`]

Porting guide for this class is accessible [here](./BleInstanceBase.md)

#### [`Gap`]

Porting guide for this class is accessible [here](./Gap.md)

#### [`GattServer`]

Porting guide for this class is accessible [here](./GattServer.md)

#### [`GattClient`]

Porting guide for this class is accessible [here](./GattClient.md)

#### [`SecurityManager`]

Porting guide for this class is accessible [here](./SecurityManager.md)


[`BLE.h`]: https://github.com/ARMmbed/ble/blob/master/ble/BLE.h
[`BLE`]: https://docs.mbed.com/docs/ble-api/en/master/api/classBLE.html
[`BLEInstanceBase.h`]: https://github.com/ARMmbed/ble/blob/master/ble/BLEInstanceBase.h
[`BLEInstanceBase`]: https://docs.mbed.com/docs/ble-api/en/master/api/classBLEInstanceBase.html
[`Gap.h`]: https://github.com/ARMmbed/ble/blob/master/ble/Gap.h
[`Gap`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGap.html
[`GattServer.h`]: https://github.com/ARMmbed/ble/blob/master/ble/GattServer.h
[`GattServer`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGattServer.html
[`GattClient.h`]: https://github.com/ARMmbed/ble/blob/master/ble/GattClient.h
[`GattClient`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGattClient.html
[`SecurityManager.h`]: https://github.com/ARMmbed/ble/blob/master/ble/SecurityManager.h
[`SecurityManager`]: https://docs.mbed.com/docs/ble-api/en/master/api/classSecurityManager.html
[`UUID.h`]: https://github.com/ARMmbed/ble/blob/master/ble/UUID.h
[`UUID`]: https://docs.mbed.com/docs/ble-api/en/master/api/classUUID.html
[`FunctionPointerWithContext.h`]: https://github.com/ARMmbed/ble/blob/master/ble/FunctionPointerWithContext.h
[`FunctionPointerWithContext`]: https://docs.mbed.com/docs/ble-api/en/master/api/classFunctionPointerWithContext.html
[`CallChainOfFunctionPointersWithContext.h`]: https://github.com/ARMmbed/ble/blob/master/ble/CallChainOfFunctionPointersWithContext.h
[`CallChainOfFunctionPointersWithContext`]: https://docs.mbed.com/docs/ble-api/en/master/api/classCallChainOfFunctionPointersWithContext.html
[`GapAdvertisingData.h`]: https://github.com/ARMmbed/ble/blob/master/ble/GapAdvertisingData.h
[`GapAdvertisingData`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGapAdvertisingData.html
[`GapAdvertisingParams.h`]: https://github.com/ARMmbed/ble/blob/master/ble/GapAdvertisingParams.h
[`GapAdvertisingParams`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGapAdvertisingParams.html
[`GapEvents.h`]: https://github.com/ARMmbed/ble/blob/master/ble/GapEvents.h
[`GapEvents`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGapEvents.html
[`GapScanningParams.h`]: https://github.com/ARMmbed/ble/blob/master/ble/GapScanningParams.h
[`GapScanningParams`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGapScanningParams.html
[`GattService.h`]: https://github.com/ARMmbed/ble/blob/master/ble/GattService.h
[`GattService`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGattService.html
[`GattCharacteristic.h`]: https://github.com/ARMmbed/ble/blob/master/ble/GattCharacteristic.h
[`GattCharacteristic`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGattCharacteristic.html
[`GattAttribute.h`]: https://github.com/ARMmbed/ble/blob/master/ble/GattAttribute.h
[`GattAttribute`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGattAttribute.html
[`DiscoveredService.h`]: https://github.com/ARMmbed/ble/blob/master/ble/DiscoveredService.h
[`DiscoveredService`]: https://docs.mbed.com/docs/ble-api/en/master/api/classDiscoveredService.html
[`DiscoveredCharacteristic.h`]: https://github.com/ARMmbed/ble/blob/master/ble/DiscoveredCharacteristic.h
[`DiscoveredCharacteristic`]: https://docs.mbed.com/docs/ble-api/en/master/api/classDiscoveredCharacteristic.html
[`DiscoveredCharacteristicDescriptor.h`]: https://github.com/ARMmbed/ble/blob/master/ble/DiscoveredCharacteristicDescriptor.h
[`DiscoveredCharacteristicDescriptor`]: https://docs.mbed.com/docs/ble-api/en/master/api/classDiscoveredCharacteristicDescriptor.html
[`ServiceDiscovery.h`]: https://github.com/ARMmbed/ble/blob/master/ble/ServiceDiscovery.h
[`ServiceDiscovery`]: https://docs.mbed.com/docs/ble-api/en/master/api/classServiceDiscovery.html
[`CharacteristicDescriptorDiscovery.h`]: https://github.com/ARMmbed/ble/blob/master/ble/CharacteristicDescriptorDiscovery.h
[`CharacteristicDescriptorDiscovery`]: https://docs.mbed.com/docs/ble-api/en/master/api/classCharacteristicDescriptorDiscovery.html
[`GattCallbackParamTypes.h`]: https://github.com/ARMmbed/ble/blob/master/ble/GattCallbackParamTypes.h
[`GattServerEvents.h`]: https://github.com/ARMmbed/ble/blob/master/ble/GattServerEvents.h
[`BleProtocol.h`]: https://github.com/ARMmbed/ble/blob/master/ble/BleProtocol.h
[`ble-common.h`]: https://github.com/ARMmbed/ble/blob/master/ble/ble-common.h
[`SafeBool.h`]: https://github.com/ARMmbed/ble/blob/master/ble/SafeBool.h
[`SafeBool`]: https://docs.mbed.com/docs/ble-api/en/master/api/classSafeBool.html
[`deprecate.h`]: https://github.com/ARMmbed/ble/blob/master/ble/deprecate.h
