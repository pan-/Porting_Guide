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



#### [`Gap`]

#### [`GattServer`]


#### [`GattClient`]


##### `launchServiceDiscovery`

```c++
/**
 * Launch service discovery. Once launched, application callbacks will be
 * invoked for matching services or characteristics. isServiceDiscoveryActive()
 * can be used to determine status, and a termination callback (if one was set up)
 * will be invoked at the end. Service discovery can be terminated prematurely,
 * if needed, using terminateServiceDiscovery().
 *
 * @param[in]  connectionHandle
 *              Handle for the connection with the peer.
 * @param[in]  sc
 *              This is the application callback for a matching service. Taken as
 *              NULL by default. Note: service discovery may still be active
 *              when this callback is issued; calling asynchronous BLE-stack
 *              APIs from within this application callback might cause the
 *              stack to abort service discovery. If this becomes an issue, it
 *              may be better to make a local copy of the discoveredService and
 *              wait for service discovery to terminate before operating on the
 *              service.
 * @param[in]  cc
 *              This is the application callback for a matching characteristic.
 *              Taken as NULL by default. Note: service discovery may still be
 *              active when this callback is issued; calling asynchronous
 *              BLE-stack APIs from within this application callback might cause
 *              the stack to abort service discovery. If this becomes an issue,
 *              it may be better to make a local copy of the discoveredCharacteristic
 *              and wait for service discovery to terminate before operating on the
 *              characteristic.
 * @param[in]  matchingServiceUUID
 *              UUID-based filter for specifying a service in which the application is
 *              interested. By default it is set as the wildcard UUID_UNKNOWN,
 *              in which case it matches all services. If characteristic-UUID
 *              filter (below) is set to the wildcard value, then a service
 *              callback will be invoked for the matching service (or for every
 *              service if the service filter is a wildcard).
 * @param[in]  matchingCharacteristicUUIDIn
 *              UUID-based filter for specifying characteristic in which the application
 *              is interested. By default it is set as the wildcard UUID_UKNOWN
 *              to match against any characteristic. If both service-UUID
 *              filter and characteristic-UUID filter are used with non-wildcard
 *              values, then only a single characteristic callback is
 *              invoked for the matching characteristic.
 *
 * @note     Using wildcard values for both service-UUID and characteristic-
 *           UUID will result in complete service discovery: callbacks being
 *           called for every service and characteristic.
 *
 * @note     Providing NULL for the characteristic callback will result in
 *           characteristic discovery being skipped for each matching
 *           service. This allows for an inexpensive method to discover only
 *           services.
 *
 * @return
 *           BLE_ERROR_NONE if service discovery is launched successfully; else an appropriate error.
 */
virtual ble_error_t launchServiceDiscovery(Gap::Handle_t                               connectionHandle,
                                           ServiceDiscovery::ServiceCallback_t         sc                           = NULL,
                                           ServiceDiscovery::CharacteristicCallback_t  cc                           = NULL,
                                           const UUID                                 &matchingServiceUUID          = UUID::ShortUUIDBytes_t(BLE_UUID_UNKNOWN),
                                           const UUID                                 &matchingCharacteristicUUIDIn = UUID::ShortUUIDBytes_t(BLE_UUID_UNKNOWN));
```


##### `discoverServices`

OPTIONAL !!!!!

```c++
/**
 * Launch service discovery for services. Once launched, service discovery will remain
 * active with service-callbacks being issued back into the application for matching
 * services. isServiceDiscoveryActive() can be used to
 * determine status, and a termination callback (if set up) will be invoked
 * at the end. Service discovery can be terminated prematurely, if needed,
 * using terminateServiceDiscovery().
 *
 * @param[in]  connectionHandle
 *              Handle for the connection with the peer.
 * @param[in]  callback
 *              This is the application callback for a matching service.
 *              Note: service discovery may still be active
 *              when this callback is issued; calling asynchronous BLE-stack
 *              APIs from within this application callback might cause the
 *              stack to abort service discovery. If this becomes an issue, it
 *              may be better to make a local copy of the discoveredService and
 *              wait for service discovery to terminate before operating on the
 *              service.
 * @param[in]  matchingServiceUUID
 *              UUID-based filter for specifying a service in which the application is
 *              interested. By default it is set as the wildcard UUID_UNKNOWN,
 *              in which case it matches all services.
 *
 * @return
 *           BLE_ERROR_NONE if service discovery is launched successfully; else an appropriate error.
 */
virtual ble_error_t discoverServices(Gap::Handle_t                        connectionHandle,
                                     ServiceDiscovery::ServiceCallback_t  callback,
                                     const UUID                          &matchingServiceUUID = UUID::ShortUUIDBytes_t(BLE_UUID_UNKNOWN));
```


##### `discoverServices`

```c++
/**
 * Launch service discovery for services. Once launched, service discovery will remain
 * active with service-callbacks being issued back into the application for matching
 * services. isServiceDiscoveryActive() can be used to
 * determine status, and a termination callback (if set up) will be invoked
 * at the end. Service discovery can be terminated prematurely, if needed,
 * using terminateServiceDiscovery().
 *
 * @param[in]  connectionHandle
 *              Handle for the connection with the peer.
 * @param[in]  callback
 *              This is the application callback for a matching service.
 *              Note: service discovery may still be active
 *              when this callback is issued; calling asynchronous BLE-stack
 *              APIs from within this application callback might cause the
 *              stack to abort service discovery. If this becomes an issue, it
 *              may be better to make a local copy of the discoveredService and
 *              wait for service discovery to terminate before operating on the
 *              service.
 * @param[in]  startHandle, endHandle
 *              Handle range within which to limit the search.
 *
 * @return
 *           BLE_ERROR_NONE if service discovery is launched successfully; else an appropriate error.
 */
virtual ble_error_t discoverServices(Gap::Handle_t                        connectionHandle,
                                     ServiceDiscovery::ServiceCallback_t  callback,
                                     GattAttribute::Handle_t              startHandle,
                                     GattAttribute::Handle_t              endHandle);
```


##### `isServiceDiscoveryActive`

```c++
/**
 * Check if service-discovery is currently active.
 *
 * @return true if service-discovery is active, false otherwise.
 */
virtual bool isServiceDiscoveryActive(void) const;
```


##### `terminateServiceDiscovery`

```c++
/**
 * Terminate an ongoing service discovery. This should result in an
 * invocation of TerminationCallback if service-discovery is active.
 */
virtual void terminateServiceDiscovery(void);
```



##### `read`

```c++
/**
 * Initiate a GATT Client read procedure by attribute-handle.
 *
 * @param[in] connHandle
 *              Handle for the connection with the peer.
 * @param[in] attributeHandle
 *              Handle of the attribute to read data from.
 * @param[in] offset
 *              The offset from the start of the attribute value to be read.
 *
 * @return
 *          BLE_ERROR_NONE if read procedure was successfully started.
 */
virtual ble_error_t read(Gap::Handle_t connHandle, GattAttribute::Handle_t attributeHandle, uint16_t offset);
```



##### `write`

```c++
/**
 * Initiate a GATT Client write procedure.
 *
 * @param[in] cmd
 *              Command can be either a write-request (which generates a
 *              matching response from the peripheral), or a write-command
 *              (which doesn't require the connected peer to respond).
 * @param[in] connHandle
 *              Connection handle.
 * @param[in] attributeHandle
 *              Handle for the target attribtue on the remote GATT server.
 * @param[in] length
 *              Length of the new value.
 * @param[in] value
 *              New value being written.
 *
 * @return
 *          BLE_ERROR_NONE if write procedure was successfully started.
 */
virtual ble_error_t write(GattClient::WriteOp_t    cmd,
                          Gap::Handle_t            connHandle,
                          GattAttribute::Handle_t  attributeHandle,
                          size_t                   length,
                          const uint8_t* value) const;
```


##### `onServiceDiscoveryTermination`

```c++
/**
 * Set up a callback for when serviceDiscovery terminates.
 *
 * @param[in] callback
 *              Event handler being registered.
 */
virtual void onServiceDiscoveryTermination(ServiceDiscovery::TerminationCallback_t callback);
```


##### `discoverCharacteristicDescriptors`

```c++
/**
 * @brief Launch discovery of descriptors for a given characteristic.
 *
 * @details This function will discover all descriptors available for a
 *          specific characteristic.
 *
 * @param[in] characteristic
 *              The characteristic targeted by this discovery procedure.
 * @param[in] discoveryCallback
 *              User function called each time a descriptor is found during
 *              the procedure.
 * @param[in] terminationCallback
 *              User provided function which will be called once the
 *              discovery procedure is terminating. This will get called
 *              when all the descriptors have been discovered or if an
 *              error occur during the discovery procedure.
 *
 * @return
 *   BLE_ERROR_NONE if characteristic descriptor discovery is launched
 *   successfully; else an appropriate error.
 */
virtual ble_error_t discoverCharacteristicDescriptors(
    const DiscoveredCharacteristic& characteristic,
    const CharacteristicDescriptorDiscovery::DiscoveryCallback_t& discoveryCallback,
    const CharacteristicDescriptorDiscovery::TerminationCallback_t& terminationCallback);
```



##### `isCharacteristicDescriptorDiscoveryActive`

```c++
/**
 * @brief Indicate if the discovery of characteristic descriptors is active
 *        for a given characteristic or not.
 *
 * @param[in] characteristic
 *              The characteristic concerned by the descriptors discovery.
 *
 * @return true if a descriptors discovery is active for the characteristic
 *         in input; otherwise false.
 */
virtual bool isCharacteristicDescriptorDiscoveryActive(const DiscoveredCharacteristic& characteristic);
```


##### `terminateCharacteristicDescriptorDiscovery`

```c++
/**
 * @brief Terminate an ongoing characteristic descriptor discovery.
 *
 * @details This should result in an invocation of the TerminationCallback if
 *          the characteristic descriptor discovery is active.
 *
 * @param[in] characteristic
 *              The characteristic on which the running descriptors
 *              discovery should be stopped.
 */
virtual void terminateCharacteristicDescriptorDiscovery(const DiscoveredCharacteristic& characteristic);
```


##### `reset`

```c++
/**
 * Notify all registered onShutdown callbacks that the GattClient is
 * about to be shutdown and clear all GattClient state of the
 * associated object.
 *
 * This function is meant to be overridden in the platform-specific
 * sub-class. Nevertheless, the sub-class is only expected to reset its
 * state and not the data held in GattClient members. This shall be achieved
 * by a call to GattClient::reset() from the sub-class' reset()
 * implementation.
 *
 * @return BLE_ERROR_NONE on success.
 */
virtual ble_error_t reset(void);
```




#### [`SecurityManager`]



##### `init`

```c++
/**
 * Enable the BLE stack's Security Manager. The Security Manager implements
 * the actual cryptographic algorithms and protocol exchanges that allow two
 * devices to securely exchange data and privately detect each other.
 * Calling this API is a prerequisite for encryption and pairing (bonding).
 *
 * @param[in]  enableBonding Allow for bonding.
 * @param[in]  requireMITM   Require protection for man-in-the-middle attacks.
 * @param[in]  iocaps        To specify the I/O capabilities of this peripheral,
 *                           such as availability of a display or keyboard, to
 *                           support out-of-band exchanges of security data.
 * @param[in]  passkey       To specify a static passkey.
 *
 * @return BLE_ERROR_NONE on success.
 */
virtual ble_error_t init(bool                     enableBonding = true,
                         bool                     requireMITM   = true,
                         SecurityIOCapabilities_t iocaps        = IO_CAPS_NONE,
                         const Passkey_t          passkey       = NULL);
```


##### `getLinkSecurity`

```c++
/**
 * Get the security status of a connection.
 *
 * @param[in]  connectionHandle   Handle to identify the connection.
 * @param[out] securityStatusP    Security status.
 *
 * @return BLE_ERROR_NONE or appropriate error code indicating the failure reason.
 */
virtual ble_error_t getLinkSecurity(Gap::Handle_t connectionHandle, LinkSecurityStatus_t* securityStatusP);
```



##### `setLinkSecurity`

```c++
/**
 * Set the security mode on a connection. Useful for elevating the security mode
 * once certain conditions are met, e.g., a particular service is found.
 *
 * @param[in]  connectionHandle   Handle to identify the connection.
 * @param[in]  securityMode       Requested security mode.
 *
 * @return BLE_ERROR_NONE or appropriate error code indicating the failure reason.
 */
virtual ble_error_t setLinkSecurity(Gap::Handle_t connectionHandle, SecurityMode_t securityMode);
```



##### `purgeAllBondingState`

```c++
/**
 * Delete all peer device context and all related bonding information from
 * the database within the security manager.
 *
 * @retval BLE_ERROR_NONE             On success, else an error code indicating reason for failure.
 * @retval BLE_ERROR_INVALID_STATE    If the API is called without module initialization or
 *                                    application registration.
 */
virtual ble_error_t purgeAllBondingState(void);
```


##### `getAddressesFromBondTable`

```c++
/**
 * Get a list of addresses from all peers in the bond table.
 *
 * @param[in,out]   addresses
 *                  (on input) addresses.capacity contains the maximum
 *                  number of addresses to be returned.
 *                  (on output) The populated table with copies of the
 *                  addresses in the implementation's whitelist.
 *
 * @retval BLE_ERROR_NONE             On success, else an error code indicating reason for failure.
 * @retval BLE_ERROR_INVALID_STATE    If the API is called without module initialization or
 *                                    application registration.
 *
 * @experimental
 */
virtual ble_error_t getAddressesFromBondTable(Gap::Whitelist_t& addresses) const;
```

##### WHAT TO DO WITH THIS CRAP ???

```c++
/**
 * To indicate that a security procedure for the link has started.
 */
virtual void onSecuritySetupInitiated(SecuritySetupInitiatedCallback_t callback) {securitySetupInitiatedCallback = callback;}

/**
 * To indicate that the security procedure for the link has completed.
 */
virtual void onSecuritySetupCompleted(SecuritySetupCompletedCallback_t callback) {securitySetupCompletedCallback = callback;}

/**
 * To indicate that the link with the peer is secured. For bonded devices,
 * subsequent reconnections with a bonded peer will result only in this callback
 * when the link is secured; setup procedures will not occur (unless the
 * bonding information is either lost or deleted on either or both sides).
 */
virtual void onLinkSecured(LinkSecuredCallback_t callback) {linkSecuredCallback = callback;}

/**
 * To indicate that device context is stored persistently.
 */
virtual void onSecurityContextStored(HandleSpecificEvent_t callback) {securityContextStoredCallback = callback;}

/**
 * To set the callback for when the passkey needs to be displayed on a peripheral with DISPLAY capability.
 */
virtual void onPasskeyDisplay(PasskeyDisplayCallback_t callback) {passkeyDisplayCallback = callback;}

```



##### `reset`

```c++
/**
 * Notify all registered onShutdown callbacks that the SecurityManager is
 * about to be shutdown and clear all SecurityManager state of the
 * associated object.
 *
 * This function is meant to be overridden in the platform-specific
 * sub-class. Nevertheless, the sub-class is only expected to reset its
 * state and not the data held in SecurityManager members. This shall be
 * achieved by a call to SecurityManager::reset() from the sub-class'
 * reset() implementation.
 *
 * @return BLE_ERROR_NONE on success.
 */
virtual ble_error_t reset(void);
```







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
