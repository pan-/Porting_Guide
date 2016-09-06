# BLE Porting guide

mbed BLE is the Bluetooth Low Energy abstraction for the mbed platform.
It work on mbed-os 2, 3 and 5.
Sources are mainly hosted on [Github](https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE)
and mirrored [here](https://developer.mbed.org/teams/Bluetooth-Low-Energy/code/BLE_API/)
for mbed-os 2 and [here](https://github.com/ARMmbed/ble) for mbed os 3.

Since mbed BLE is just an abstraction over BLE capable chips and modules, it
requires an implementation to be useful.
Abstractions of mbed BLE are provided by vendor, and this guide is all about how to
write such implementation.

It is expected that a vendor implementation is distributed as a part of the OS
([here])[https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/targets]
if the radio is on the SOC or as an external git or mercurial repository.

## mbed BLE organization
A port of mbed BLE is, normally, not used directly by the end user. It is expected
that mbed users wrote their ble application using mbed BLE and nothing else.
To make this works, it is required that vendors provide implementation for certain
parts of mbed BLE.

The mbed BLE directory contains three folders:
* [`ble`](https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble):
Contain all the public headers of mbed BLE.
* [`source`](https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/source):
Contains the implementation of the platform independent parts of mbed BLE.
* [`targets`](https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/source):
Contains the implementation of mbed BLE for radios living on the SOC.

The folder ble contains all declarations of types, functions and classes provided
by BLE_API, a porter does not have to provide an implementation for all of this
files but it is important to understand what is available here before starting the
porting effort.
* [`BLE.h`]: This file is the entry point to the whole mbed BLE. It contains the
class [`BLE`] which provide access to all functionalities of mbed BLE.
* [`BLEInstanceBase.h`]:Contains the definition of the class [`BLEInstanceBase`],
an interface which manage the vendor BLE stack and provide access to other interfaces
of mbed BLE. [`BLEInstanceBase`] can be seen as a private implementation of the class [`BLE`].
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
which is used as the callback abstraction of mbed BLE.
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
* [`ble-common.h`]: Contains definitions which are shared in multiple bits of mbed BLE.
* [`SafeBool.h`]: Contain the utility class [`SafeBool`] which implement the safe bool
idiom in c++03.
* [`deprecate.h`]: Provide a way to deprecate bits of API.


## Architecture

The main entry point of mbed BLE is the class [`BLE`], it allow the user to initialize/
shutdown the [`BLE`] stack and also provide accessors to the classes which models
the different layers of the Bluetooth protocol.

[`BLE`] class use an interface to a private implementation named [`BLEInstanceBase`]
to provide all those functions.

It is expected that a port of mbed BLE provide implementations for the following
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
All asynchronous operations (commands or listening to spurious event) in mbed
BLE will call at some point a callback registered at a previous point in time.

An implementation of mbed BLE should **not** execute any pending callbacks right
away but instead, it should do it when the user ask mbed BLE to process the events
by using the function `BLE::processEvents`.

To signal that there is works to do, an implementation should call the function
`BLEInstanceBase::signalEventsToProcess`. This will signal the user code that
the processing of the events should be done.

This model plays nicely with event driven or polling based applications and allow
the user code to be executed in a single thread. It also naturally fits with
most stacks (HCI or internal ones) mbed BLE tries to abstract. Those mechanisms
(signal/process) are usually already there.

## [`BleInstanceBase`]

Porting guide for this class is accessible [here](./BLEInstanceBase.md)

## [`Gap`]

Porting guide for this class is accessible [here](./Gap.md)

## [`GattServer`]

Porting guide for this class is accessible [here](./GattServer.md)

## [`GattClient`]

Porting guide for this class is accessible [here](./GattClient.md)

## [`SecurityManager`]

Porting guide for this class is accessible [here](./SecurityManager.md)


[`BLE.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/BLE.h
[`BLE`]: https://docs.mbed.com/docs/ble-api/en/master/api/classBLE.html
[`BLEInstanceBase.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/BLEInstanceBase.h
[`BLEInstanceBase`]: https://docs.mbed.com/docs/ble-api/en/master/api/classBLEInstanceBase.html
[`Gap.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/Gap.h
[`Gap`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGap.html
[`GattServer.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/GattServer.h
[`GattServer`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGattServer.html
[`GattClient.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/GattClient.h
[`GattClient`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGattClient.html
[`SecurityManager.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/SecurityManager.h
[`SecurityManager`]: https://docs.mbed.com/docs/ble-api/en/master/api/classSecurityManager.html
[`UUID.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/UUID.h
[`UUID`]: https://docs.mbed.com/docs/ble-api/en/master/api/classUUID.html
[`FunctionPointerWithContext.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/FunctionPointerWithContext.h
[`FunctionPointerWithContext`]: https://docs.mbed.com/docs/ble-api/en/master/api/classFunctionPointerWithContext.html
[`CallChainOfFunctionPointersWithContext.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/CallChainOfFunctionPointersWithContext.h
[`CallChainOfFunctionPointersWithContext`]: https://docs.mbed.com/docs/ble-api/en/master/api/classCallChainOfFunctionPointersWithContext.html
[`GapAdvertisingData.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/GapAdvertisingData.h
[`GapAdvertisingData`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGapAdvertisingData.html
[`GapAdvertisingParams.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/GapAdvertisingParams.h
[`GapAdvertisingParams`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGapAdvertisingParams.html
[`GapEvents.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/GapEvents.h
[`GapEvents`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGapEvents.html
[`GapScanningParams.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/GapScanningParams.h
[`GapScanningParams`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGapScanningParams.html
[`GattService.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/GattService.h
[`GattService`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGattService.html
[`GattCharacteristic.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/GattCharacteristic.h
[`GattCharacteristic`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGattCharacteristic.html
[`GattAttribute.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/GattAttribute.h
[`GattAttribute`]: https://docs.mbed.com/docs/ble-api/en/master/api/classGattAttribute.html
[`DiscoveredService.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/DiscoveredService.h
[`DiscoveredService`]: https://docs.mbed.com/docs/ble-api/en/master/api/classDiscoveredService.html
[`DiscoveredCharacteristic.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/DiscoveredCharacteristic.h
[`DiscoveredCharacteristic`]: https://docs.mbed.com/docs/ble-api/en/master/api/classDiscoveredCharacteristic.html
[`DiscoveredCharacteristicDescriptor.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/DiscoveredCharacteristicDescriptor.h
[`DiscoveredCharacteristicDescriptor`]: https://docs.mbed.com/docs/ble-api/en/master/api/classDiscoveredCharacteristicDescriptor.html
[`ServiceDiscovery.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/ServiceDiscovery.h
[`ServiceDiscovery`]: https://docs.mbed.com/docs/ble-api/en/master/api/classServiceDiscovery.html
[`CharacteristicDescriptorDiscovery.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/CharacteristicDescriptorDiscovery.h
[`CharacteristicDescriptorDiscovery`]: https://docs.mbed.com/docs/ble-api/en/master/api/classCharacteristicDescriptorDiscovery.html
[`GattCallbackParamTypes.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/GattCallbackParamTypes.h
[`GattServerEvents.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/GattServerEvents.h
[`BleProtocol.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/BleProtocol.h
[`ble-common.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/ble-common.h
[`SafeBool.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/SafeBool.h
[`SafeBool`]: https://docs.mbed.com/docs/ble-api/en/master/api/classSafeBool.html
[`deprecate.h`]: https://github.com/ARMmbed/mbed-os/tree/master/features/FEATURE_BLE/ble/deprecate.h
