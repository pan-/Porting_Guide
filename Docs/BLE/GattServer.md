# BLEInstanceBase

This document provides a porting guide for the class [`GattServer`] available
in [BLE_API] version 2.5.1.

The [`GattServer`] API is declared in the header file [`GattServer.h`].


## Description

[`GattServer`] is an abstract class of BLE_API which provide function to create
and manage a GATT server.

A valid port of BLE_API should provide an implementation for this class.

## Requirements

Some more info about the module comes here.

## Note

Besides API override, it is important that an implementation of this class
call the right callbacks when an event is fired from the underlying stack.

### CCCD Written
When a CCCD is written, the implementation should notify upper layer that this
event happen:
* If notification and/or indication is enabled, the implementation should call
the function `GattServer::handleEvent` with the `type` parameter equal to
`GattServerEvents::GATT_EVENT_UPDATES_ENABLED` and the `attributeHandle`
parameter equal to the handle of the characteristic value which own this CCCD.
* If notification and/or indication is disabled, the implementation should call
the function `GattServer::handleEvent` with the `type` parameter equal to
`GattServerEvents::GATT_EVENT_UPDATES_DISABLED` and the `attributeHandle`
parameter equal to the handle of the characteristic value which own this CCCD.


### Data sent event
Once a GattServer has send data packet, the implementation of the [`GattServer`]
should call the function `GattServer::handleDataSentEvent()` with the number of
packets transmitted as parameter.

### Indication received event
Once a [`GattServer`] as received the confirmation that an *indication* has been
received by a client, the implementation of [`GattServer`] should call the
function `GattServer::handle` with
`GattServerEvents::GATT_EVENT_CONFIRMATION_RECEIVED` as the event type and the
handle of the characteristic value which has done the indication as second
parameter.

### Data written event
After a client has written data on the server, the [`GattServer`] implementation
should call the function `GattServer::handleDataWrittenEvent`.

### Data read event
If the BLE stack can *emit an event* when data is read from the GATT server,
then the [`GattServer`] implementation should call the function
`GattServer::handleDataReadEvent` when a client read data on the server.


### Authorization request
GATT characteristics can be registered into the server with a read authorization
or write authorization callback.

#### Read authorization
If the function `GattCharacteristic::isReadAuthorizationEnabled` return `true`
for a registered characteristic then when a read request is received for this
characteristic, the server should call the function
`GattCharacteristic::authorizeRead` of this characteristic to know if read
is authorized or not.

If `GattCharacteristic::authorizeRead` return
`GattCharacteristic::AUTH_CALLBACK_REPLY_SUCCESS` then the characteristic can
be read and the server should send back the value of the characteristic.

Otherwise, the server should reply to the request with the status returned by
`GattCharacteristic::authorizeRead`.

It is safe to call the function `GattCharacteristic::authorizeRead` before any
read, if there is no user callback registered, the function will always return
`true`.

#### Write authorization
If the function `GattCharacteristic::isWriteAuthorizationEnabled` return `true`
for a registered characteristic then when a write request is received for this
characteristic, the server should call the function
`GattCharacteristic::authorizeWrite` of this characteristic to know if writing
is authorized or not.

If `GattCharacteristic::authorizeWrite` return
`GattCharacteristic::AUTH_CALLBACK_REPLY_SUCCESS` then the characteristic can
be written and the server should return success if the characteristic has been
correctly written.

Otherwise, the server should reply to the request with the status returned by
`GattCharacteristic::authorizeWrite`.

It is safe to call the function `GattCharacteristic::authorizeWrite` before any
write, if there is no user callback registered, the function will always return
`true`.

### Security request
If a GATT characteristic registered in the server require a certain level of
security to be accessed (`GattCharacteristic::getrequiredSecurity`) and if the
connection is not at the level of security required, then any access to this
characteristic by a GATT client trigger a security request from the GATT server.
The parameter `authRequest` (see Bluetooth specification 4.2 [Vol 3, Part H] -
3.6.7  ) of the security request should match with the required security of the
characteristic.


## API

### `addService`

```c++
/**
 * Add a service declaration to the local server ATT table. Also add the
 * characteristics contained within.
 *
 * @param[in] service The service to be added.
 *
 * @return BLE_ERROR_NONE if the service was successfully added.
 */
ble_error_t GattServer::addService(GattService& service);
```

This function add a new service into the GATT server.

In case of success, the following postconditions should be true:

* The GATT handle of the service instance should be set (using the function
`GattService::setHandle`).
* The GATT handle of the attribute value of all characteristics should be set.
The attribute can be accessed using the function
`GattCharacterstic::getValueAttribute` and the value of the handle can be set
using the function `GattAttribute::setHandle`.
* The GATT handle of each characteristic descriptors should be set. Descriptors
attributes can be accessed using the function `GattCharacterstic::getDescriptor`
and the value of the handle can be set using the function
`GattAttribute::setHandle`.

It is expected that the implementation generate some characteristics descriptors
in the following case:
* If the property of the characteristics contains the value
`GattCharacteristic::BLE_GATT_CHAR_PROPERTIES_EXTENDED_PROPERTIES` and no
descriptor `Characteristic Extended Properties` provided, the implementation
should generate one and add it while the Characteristic is added to the server .
* If the characteristic can notify or indicate values change and there is no
`Client Characteristic Configuration Descriptor` provided, the implementation
should generate one and add it while the characteristic is added to the server.

Values of characteristics and descriptors should be kept and managed by the GATT
server implementation until the next call to `GattServer::reset` or
`BLE::shutdown`. The value provided by `GattAttribute::getValuePtr` is only here
for initialization purposes.

An implementation should kept a reference of any characteristics registered.
This reference will be used later by the implementation when a characteristic is
read or written.


Once a service has been registered, it is expected that the service is available
to GATT clients which are connected to the device.


### `read`

This api cames in two flavor:

```c++
/**
 * Read the value of a characteristic or a descriptor from the local GATT server.
 *
 * @param[in] attributeHandle Attribute handle for the value attribute of the
 * characteristic or an attribute handle of a descriptor.
 * @param[out]    buffer A buffer to hold the value being read.
 * @param[in,out] lengthP Length of the buffer being supplied. If the value is
 * longer than the size of the supplied buffer, this variable will hold upon
 * return the total attribute value length (excluding offset). The application
 * may use this information to allocate a suitable buffer size.
 *
 * @return BLE_ERROR_NONE if a value was read successfully into the buffer.
 */
ble_error_t GattServer::read(GattAttribute::Handle_t attributeHandle, uint8_t buffer[], uint16_t* lengthP);
```

```c++
/**
 * Read the value of a characteristic or a descriptor from the local GATT server.
 *
 * @param[in] connectionHandle Connection handle.
 * @param[in] attributeHandle Attribute handle for the value attribute of the
 * characteristic or an attribute handle of a descriptor.
 * @param[out] buffer A buffer to hold the value being read.
 * @param[in,out] lengthP Length of the buffer being supplied. If the
 * value is longer than the size of the supplied buffer, this variable will hold
 * upon return the total attribute value length (excluding offset). The
 * application may use this information to allocate a suitable buffer size.
 *
 * @return BLE_ERROR_NONE if a value was read successfully into the buffer.
 *
 * @note This API is a version of the above, with an additional connection handle
 * parameter to allow fetches for connection-specific multivalued attributes
 * (such as the CCCDs).
 */
ble_error_t GattServer::read(Gap::Handle_t connectionHandle, GattAttribute::Handle_t attributeHandle, uint8_t* buffer, uint16_t* lengthP);
```

These functions read the value of a GATT attribute of the local GATT server. The
parameter `connectionHandle` is only useful to read a CCCD attribute.


### `write`

Like `read`, this function is defined in two flavors:

```c++
/**
 * Update the value of a characteristic or a descriptor on the local GATT server.
 *
 * @param[in] attributeHandle Handle for the value attribute of the
 * characteristic or an attribute handle of a descriptor.
 * @param[in] value A pointer to a buffer holding the new value.
 * @param[in] size Size of the new value (in bytes).
 * @param[in] localOnly Should this update be kept on the local GATT server
 * regardless of the state of the notify/indicate flag in the CCCD for this
 * Characteristic? If set to true, no notification or indication is generated.
 *
 * @return BLE_ERROR_NONE if we have successfully set the value of the attribute.
 */
ble_error_t GattServer::write(GattAttribute::Handle_t attributeHandle, const uint8_t* value, uint16_t size, bool localOnly = false);
```

```c++
/**
 * Update the value of a characteristic or a descriptor on the local GATT server.
 * A version of the same as the above, with a connection handle parameter to
 * allow updates for connection-specific multivalued attributes (such as the
 * CCCDs).
 *
 * @param[in] connectionHandle Connection handle.
 * @param[in] attributeHandle Handle for the value attribute of the
 * characteristic or an attribute handle of a descriptor.
 * @param[in] value A pointer to a buffer holding the new value.
 * @param[in] size Size of the new value (in bytes).
 * @param[in] localOnly Should this update be kept on the local GattServer
 * regardless of the state of the notify/indicate flag in the CCCD for this
 * Characteristic? If set to true, no notification or indication is generated.
 *
 * @return BLE_ERROR_NONE if we have successfully set the value of the attribute.
 */
ble_error_t GattServer::write(Gap::Handle_t connectionHandle, GattAttribute::Handle_t attributeHandle, const uint8_t* value, uint16_t size, bool localOnly = false);
```

### `areUpdatesEnabled`

```c++
/**
 * Determine the connection-specific updates-enabled status (notification or
 * indication) from a characteristic's CCCD.
 *
 * @param[in] connectionHandle The connection handle.
 * @param[in] characteristic The characteristic.
 * @param[out] enabledP Upon return, *enabledP is true if updates are enabled,
 -* else false.
 *
 * @return BLE_ERROR_NONE if the connection and handle are found. False otherwise.
 */
ble_error_t GattServer::areUpdatesEnabled(Gap::Handle_t connectionHandle, const GattCharacteristic &characteristic, bool* enabledP);
```

### `isOnDataReadAvailable`

```c++
/**
 * A virtual function to allow underlying stacks to indicate if they support
 * onDataRead(). It should be overridden to return true as applicable.
 *
 * @return true if onDataRead is supported, false otherwise.
 */
bool GattServer::isOnDataReadAvailable() const;
```

If the BLE stack can *emit an event* when data is read from the GATT server,
this function should return true and  when data is read from the GATT server,
the implementation should call the function: `GattServer::handleDataReadEvent`.


### `reset`

```c++
/**
 * Notify all registered onShutdown callbacks that the GattServer is
 * about to be shutdown and clear all GattServer state of the
 * associated object.
 *
 * This function is meant to be overridden in the platform-specific
 * sub-class. Nevertheless, the sub-class is only expected to reset its
 * state and not the data held in GattServer members. This shall be achieved
 * by a call to GattServer::reset() from the sub-class' reset()
 * implementation.
 *
 * @return BLE_ERROR_NONE on success.
 */
virtual ble_error_t reset(void);
```

This function should reset the state of the GattServer instance. The
implementation should call the function `GattServer::reset` from the base class
before any operation.


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
