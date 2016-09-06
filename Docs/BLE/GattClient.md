# BLEInstanceBase

This document provides a porting guide for the class [`GattClient`] available
in [BLE_API] version 2.5.1.

The [`GattClient`] API is declared in the header file [`GattClient.h`].


## Description

[`GattClient`] is an abstract class of BLE_API which provide function to manage
a GATT client.

A valid port of BLE_API should provide an implementation for this class.

## API


### `launchServiceDiscovery`

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
ble_error_t GattClient::launchServiceDiscovery(
    Gap::Handle_t connectionHandle,
    ServiceDiscovery::ServiceCallback_t sc = NULL,
    ServiceDiscovery::CharacteristicCallback_t  cc = NULL,
    const UUID& matchingServiceUUID = UUID::ShortUUIDBytes_t(BLE_UUID_UNKNOWN),
    const UUID& matchingCharacteristicUUIDIn = UUID::ShortUUIDBytes_t(BLE_UUID_UNKNOWN)
);
```

This function launch a global discovery of the remote GATT server, it is a
kitchen sink of discoveries procedure.

Depending on its arguments, this function will adopt different behavior.

#### ServiceCallback == NULL and CharacteristicCallback == NULL
The implementation can do nothing in this case because the user will not be
notified of any discovery anyway. The termination callback set by the user should
still be called (see `GattClient::onServiceDiscoveryTermination`).


#### ServiceCallback == NULL and CharacteristicCallback != NULL

##### ServiceFilter == BLE_UUID_UNKNOWN and CharacteristicFilter == BLE_UUID_UNKNOWN
In this case, the implementation, do the following algorithm:
* launch a `Discover All Primary Service` procedure.
* For each service discovered:
    * launch a `Discover All Characteristics of a Service` procedure.
    * For each characteristic discovered call the `CharacteristicCallBack`
    supplied.

##### ServiceFilter == BLE_UUID_UNKNOWN and CharacteristicFilter != BLE_UUID_UNKNOWN
In this case, the implementation, do the following algorithm:
* launch a `Discover All Primary Service` procedure.
* For each service discovered:
    * launch a `Discover Characteristics by UUID` procedure with the
      `CharacteristicsFilter` supplied.
    * For each characteristic discovered call the `CharacteristicCallBack`
    supplied.


##### ServiceFilter != BLE_UUID_UNKNOWN and CharacteristicFilter == BLE_UUID_UNKNOWN
In this case, the implementation, do the following algorithm:
* launch a `Discover Primary Service by UUID` procedure with the `ServiceFilter`
supplied.
* For each service discovered:
    * launch a `Discover All Characteristics of a Service` procedure.
    * For each characteristic discovered call the `CharacteristicCallBack`
    supplied.


##### ServiceFilter != BLE_UUID_UNKNOWN and CharacteristicFilter != BLE_UUID_UNKNOWN
In this case, the implementation, do the following algorithm:
* launch a `Discover Primary Service by UUID` procedure with the `ServiceFilter`
supplied.
* For each service discovered:
    * launch a `Discover Characteristics by UUID` procedure with the
      `CharacteristicsFilter` supplied.
    * For each characteristic discovered call the `CharacteristicCallBack`
    supplied.



#### ServiceCallback != NULL and CharacteristicCallback == NULL

##### ServiceFilter == BLE_UUID_UNKNOWN and CharacteristicFilter == BLE_UUID_UNKNOWN
In this case, the implementation, do the following algorithm:
* launch a `Discover All Primary Service` procedure.
* for each service discovered call the `ServiceCallback` provided.

##### ServiceFilter == BLE_UUID_UNKNOWN and CharacteristicFilter != BLE_UUID_UNKNOWN
In this case, the implementation, do the following algorithm:
* launch a `Discover All Primary Service` procedure.
* for each service discovered:
    * launch a `Discover Characteristics by UUID` procedure with the
    `CharacteristicsFilter` supplied.
    * If a characteristic is found, call the `ServiceCallback` supplied.

##### ServiceFilter != BLE_UUID_UNKNOWN and CharacteristicFilter == BLE_UUID_UNKNOWN
In this case, the implementation, do the following algorithm:
* launch a `Discover Primary Service by UUID` procedure with the `ServiceFilter`
supplied.
* for each service discovered call the `ServiceCallback` provided.

##### ServiceFilter != BLE_UUID_UNKNOWN and CharacteristicFilter != BLE_UUID_UNKNOWN
In this case, the implementation, do the following algorithm:
* launch a `Discover Primary Service by UUID` procedure with the `ServiceFilter`
supplied.
* for each service discovered:
    * launch a `Discover Characteristics by UUID` procedure with the
    `CharacteristicsFilter` supplied.
    * If a characteristic is found, call the `ServiceCallback` supplied.


#### ServiceCallback != NULL and CharacteristicCallback != NULL

##### ServiceFilter == BLE_UUID_UNKNOWN and CharacteristicFilter == BLE_UUID_UNKNOWN
In this case, the implementation, do the following algorithm:
* launch a `Discover All Primary Service` procedure.
* For each service discovered:
    * call the `ServiceCallback` provided.
    * launch a `Discover All Characteristics of a Service` procedure.
    * For each characteristic discovered call the `CharacteristicCallBack`
    supplied.

##### ServiceFilter == BLE_UUID_UNKNOWN and CharacteristicFilter != BLE_UUID_UNKNOWN
In this case, the implementation, do the following algorithm:
* launch a `Discover All Primary Service` procedure.
* for each service discovered:
    * launch a `Discover Characteristics by UUID` procedure with the
    `CharacteristicsFilter` supplied.
    * If a characteristic is found, call the `ServiceCallback` supplied
    * For each characteristic found, call the `CharacteristicCallBack` provided.

##### ServiceFilter != BLE_UUID_UNKNOWN and CharacteristicFilter == BLE_UUID_UNKNOWN
In this case, the implementation, do the following algorithm:
* launch a `Discover Primary Service by UUID` procedure with the `ServiceFilter`
supplied.
* for each service discovered:
    * call the `ServiceCallback` provided.
    * launch a `Discover All Characteristics of a Service` procedure.
    * For each characteristic discovered call the `CharacteristicCallBack`
    supplied.   


##### ServiceFilter != BLE_UUID_UNKNOWN and CharacteristicFilter != BLE_UUID_UNKNOWN
In this case, the implementation, do the following algorithm:
* launch a `Discover Primary Service by UUID` procedure with the `ServiceFilter`
supplied.
* for each service discovered:
    * launch a `Discover Characteristics by UUID` procedure with the
    `CharacteristicsFilter` supplied.
    * If a characteristic is found, call the `ServiceCallback` supplied.
    * For each characteristic found, call the `CharacteristicCallBack` provided.


At the end of the discovery, the implementation of `GattClient` should call the
termination callback supplied in `GattClient::terminateServiceDiscovery`.

The termination of the discovery procedure can be requested at any time by the
function `GattClient::terminateServiceDiscovery`. If the termination of the
discovery is requested, the discovery should stop and `ServiceCallback` and
`CharacteristicCallBack` should not be called anymore.

`DiscoveredServices` generated by the discovery procedure should have their
field `uuid`, `startHandle` and `endHandle` correctly set.

`DiscoveredCharacteristics` generated by the discovery should have their field
`uuid`, `props`, `declHandle`, `valueHandle` and `connHandle` correctly set.

The field `lastHandle` is useful for characteristic descriptor discovery the
implementation can decide when it has to be set.

If the procedure `Discover All Characteristics of a Service` is used to discover
characteristic, it is recommended to set the field `lastHandle` since it is
*free* to do so.

If the underlying Bluetooth stack does not allow `ATT` procedure, it is also
recommended to replace all `Discover Characteristics by UUID` procedure issued
by this function by the `Discover All Characteristics of a Service` procedure
and filter UUIDs at application level. Such change will help discovery of
descriptors.

If the field `DiscoveredCharacteristic::lastHandle` is not set at this point,
it should remained set to the value `GattAttribute::INVALID_HANDLE`.


### `discoverServices`

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
ble_error_t GattClient::discoverServices(
    Gap::Handle_t connectionHandle,
    ServiceDiscovery::ServiceCallback_t callback,
    GattAttribute::Handle_t startHandle,
    GattAttribute::Handle_t endHandle
);
```

This function will initiate a `Discover All Primary Service` procedure in the
range [`startHandle`:`endHandle`].

If the ServiceCallback supplied is *equal* to NULL, the implementation can skip
the discovery and directly call the termination callback.

This function obey to the same rules as `GattClient::launchServiceDiscovery`.


### `isServiceDiscoveryActive`

```c++
/**
 * Check if service-discovery is currently active.
 * @return true if service-discovery is active, false otherwise.
 */
bool GattClient::isServiceDiscoveryActive(void) const;
```

this function indicate if any service discovery started by a call to
`GattClient::launchServiceDiscovery` or `GattClient::discoverServices` is
running.


### `terminateServiceDiscovery`

```c++
/**
 * Terminate an ongoing service discovery. This should result in an
 * invocation of TerminationCallback if service-discovery is active.
 */
void GattClient::terminateServiceDiscovery(void);
```

This function terminate any service discovery started by
`GattClient::launchServiceDiscovery` or `GattClient::discoverServices`. If one
of those procedure where active, the implementation should call the termination
callback registered by `GattClient::onServiceDiscoveryTermination`.


### `read`

```c++
/**
 * Initiate a GATT Client read procedure by attribute-handle.
 *
 * @param[in] connHandle Handle for the connection with the peer.
 * @param[in] attributeHandle Handle of the attribute to read data from.
 * @param[in] offset The offset from the start of the attribute value to be read.
 *
 * @return BLE_ERROR_NONE if read procedure was successfully started.
 */
ble_error_t GattClient::read(
    Gap::Handle_t connHandle,
    GattAttribute::Handle_t attributeHandle,
    uint16_t offset
);
```

Initiate a `Read Request` of an attribute.

If the attribute handle is the handle of a characteristic value this function map
to the  `Read Characteristic Value` procedure. If the attribute handle is the
handle of a descriptor this function map to the `Read Characteristic Descriptor`
procedure. Both of these procedure are mapped against the `Read Request` of an
attribute.

If the request succeed, the implementation should call
`GattClient::processReadResponse` with the result.

If the request fail, the behavior is unspecified at this point.


### `write`

```c++
/**
 * Initiate a GATT Client write procedure.
 *
 * @param[in] cmd Command can be either a write-request (which generates a
 * matching response from the peripheral), or a write-command (which doesn't
 * require the connected peer to respond).
 * @param[in] connHandle Connection handle.
 * @param[in] attributeHandle Handle for the target attribtue on the remote GATT
 * server.
 * @param[in] length Length of the new value.
 * @param[in] value New value being written.
 *
 * @return BLE_ERROR_NONE if write procedure was successfully started.
 */
ble_error_t GattClient::write(
    GattClient::WriteOp_t cmd,
    Gap::Handle_t connHandle,
    GattAttribute::Handle_t attributeHandle,
    size_t length,
    const uint8_t* value
) const;
```

Initiate a `Write Request` of an attribute.

If the attribute handle is the handle of a characteristic value this function map
to the  `Write Characteristic Value` procedure. If the attribute handle is the
handle of a descriptor this function map to the `Write Characteristic Descriptor`
procedure. Both of these procedure are mapped against the `Write Request` of an
attribute.

If the request succeed, the implementation should call
`GattClient::processWriteResponse` with the result.

If the request fail, the behavior is unspecified at this point.



### `onServiceDiscoveryTermination`

```c++
/**
 * Set up a callback for when serviceDiscovery terminates.
 *
 * @param[in] callback Event handler being registered.
 */
void GattClient::onServiceDiscoveryTermination(
    ServiceDiscovery::TerminationCallback_t callback
);
```

By calling this function, a client set up the callback to call when a service
discovery procedure terminate. The implementation of `GattClient` should keep
track of this callback.


### `discoverCharacteristicDescriptors`

```c++
/**
 * @brief Launch discovery of descriptors for a given characteristic.
 *
 * @details This function will discover all descriptors available for a
 * specific characteristic.
 *
 * @param[in] characteristic The characteristic targeted by this discovery
 * procedure.
 * @param[in] discoveryCallback User function called each time a descriptor is
 * found during the procedure.
 * @param[in] terminationCallback User provided function which will be called
 * once the discovery procedure is terminating. This will get called
 * when all the descriptors have been discovered or if an error occur during the
 * discovery procedure.
 *
 * @return BLE_ERROR_NONE if characteristic descriptor discovery is launched
 * successfully; else an appropriate error.
 */
ble_error_t GattClient::discoverCharacteristicDescriptors(
    const DiscoveredCharacteristic& characteristic,
    const CharacteristicDescriptorDiscovery::DiscoveryCallback_t& discoveryCallback,
    const CharacteristicDescriptorDiscovery::TerminationCallback_t& terminationCallback
);
```

This function start the discovery of characteristic descriptors.

Every time a descriptor is discovered, the implementation should call
`discoveredCallback` with the `DiscoveredCharacteristicDescriptor` found.
When all descriptors have been discovered, the implementation should call
`terminationCallback`.

During a discovery of a characteristic `X`, if
`GattClient::terminateCharacteristicDescriptorDiscovery` is called with `X` as
parameter, the implementation should stop the procedure and call
`terminationCallback`.

Implementation can use the GATT procedure `Discover All Characteristic
Descriptors` to discover the descriptors of a characteristic. To use this
procedure the last handle of the characteristic has to be known, ideally it is
set during characteristics discovery.

If the last handle is not known, it is possible to hijack the GATT procedure
`Discover All Characteristic Descriptors`. To do so, the procedure can be called
with the range (characteristic value handle + 1, 0xFFFF), once a characteristic
declaration UUID is reached, the procedure stop.


### `isCharacteristicDescriptorDiscoveryActive`

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
bool GattClient::isCharacteristicDescriptorDiscoveryActive(
    const DiscoveredCharacteristic& characteristic
);
```

This function test if is a discovery of all descriptors of a characteristic is
ongoing. The implementation should return true if it is the case and false
otherwise.


### `terminateCharacteristicDescriptorDiscovery`

```c++
/**
 * @brief Terminate an ongoing characteristic descriptor discovery.
 *
 * @details This should result in an invocation of the TerminationCallback if
 * the characteristic descriptor discovery is active.
 *
 * @param[in] characteristic
 *              The characteristic on which the running descriptors
 *              discovery should be stopped.
 */
void GattClient::terminateCharacteristicDescriptorDiscovery(
    const DiscoveredCharacteristic& characteristic
);
```

This function terminate an ongoing characteristic descriptor discovery for a
given characteristic. If a characteristic descriptor discovery is ongoing for
the characteristic in input, the implementation should stop the discovery
procedure and call the termination callback associated with the discovery.


### `reset`

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
ble_error_t GattClient::reset(void);
```

This function reset the GattClient to its initial state, all ongoing procedure
should stop and termination callbacks should be called.

The implementation should call the parent function (`GattClient::reset`) before
anything else.

[BLE_API]: https://github.com/ARMmbed/ble
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

[`InitializationCompleteCallbackContext`]: https://docs.mbed.com/docs/ble-api/en/master/api/structBLE_1_1InitializationCompleteCallbackContext.html
