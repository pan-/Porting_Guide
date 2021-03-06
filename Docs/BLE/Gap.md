# Gap

This document provides a porting guide for the class [`Gap`] available
in the [ble module] version 2.5.1.

The [`Gap`] API is declared in the header file [`Gap.h`].


## Description

[`Gap`] is an abstract class of BLE_API which provide access to all functions
related to the Bluetooth GAP layer. The main functionalities are:
* Advertisement management
* Scan of peers advertising
* Connection/Disconnection to peers
* Whitelist management
* Manage characteristics values of the `Generic Access Service`

A valid port of BLE_API should provide an implementation for this class.


## Note

For the rest of this document, the fictive name of the port used in implementation
example is `Foo`. In this port, the class `GapFoo` is the implementation of [`Gap`].

```c++
class GapFoo : public Gap {
public:
    GapFoo(/*constructors arguments*/);

    virtual ble_error_t setAddress(BLEProtocol::AddressType_t type, const BLEProtocol::AddressBytes_t address);
    virtual ble_error_t getAddress(BLEProtocol::AddressType_t* typeP, BLEProtocol::AddressBytes_t address);

    virtual uint16_t getMinAdvertisingInterval() const;
    virtual uint16_t getMinNonConnectableAdvertisingInterval() const;
    virtual uint16_t getMaxAdvertisingInterval() const;

    virtual ble_error_t stopAdvertising();

    virtual ble_error_t stopScan();

    virtual ble_error_t connect(
        const BLEProtocol::AddressBytes_t peerAddr,
        BLEProtocol::AddressType_t peerAddrType,
        const ConnectionParams_t* connectionParams,
        const GapScanningParams* scanParams
    );
    virtual ble_error_t disconnect(Handle_t connectionHandle, DisconnectionReason_t reason);


    virtual ble_error_t getPreferredConnectionParams(ConnectionParams_t* params);
    virtual ble_error_t setPreferredConnectionParams(const ConnectionParams_t* params);

    virtual ble_error_t updateConnectionParams(Handle_t handle, const ConnectionParams_t* params);

    virtual ble_error_t setDeviceName(const uint8_t* deviceName);
    virtual ble_error_t getDeviceName(uint8_t* deviceName, unsigned* lengthP);

    virtual ble_error_t setAppearance(GapAdvertisingData::Appearance appearance);
    virtual ble_error_t getAppearance(GapAdvertisingData::Appearance* appearanceP);

    virtual ble_error_t setTxPower(int8_t txPower);
    virtual void getPermittedTxPowerValues(const int8_t** valueArrayPP, size_t* countP);

    virtual uint8_t getMaxWhitelistSize() const;
    virtual ble_error_t getWhitelist(Whitelist_t& whitelist) const;
    virtual ble_error_t setWhitelist(const Whitelist_t& whitelist);

    virtual ble_error_t setAdvertisingPolicyMode(AdvertisingPolicyMode_t mode);
    virtual AdvertisingPolicyMode_t getAdvertisingPolicyMode() const;
    virtual ble_error_t setScanningPolicyMode(ScanningPolicyMode_t mode);
    virtual ScanningPolicyMode_t getScanningPolicyMode() const;
    virtual ble_error_t setInitiatorPolicyMode(InitiatorPolicyMode_t mode);
    virtual InitiatorPolicyMode_t getInitiatorPolicyMode() const;

protected:
    virtual ble_error_t startRadioScan(const GapScanningParams& scanningParams);

public:
    virtual ble_error_t initRadioNotification();

private:
    virtual ble_error_t setAdvertisingData(const GapAdvertisingData& advData, const GapAdvertisingData& scanResponse) = 0;
    virtual ble_error_t startAdvertising(const GapAdvertisingParams& ) = 0;    

public:
    virtual ble_error_t reset();

};
```

## API

### `setAddress`

```c++
/**
 * Set the BTLE MAC address and type. Please note that the address format is
 * least significant byte first (LSB). Please refer to BLEProtocol::AddressBytes_t.
 *
 * @param[in] type The type of the BLE address to set.
 * @param[in] address The BLE address to set if the address type is PUBLIC or
 * RANDOM_STATIC.
 *
 * @return BLE_ERROR_NONE on success.
 */
ble_error_t Gap::setAddress(BLEProtocol::AddressType_t type, const BLEProtocol::AddressBytes_t address);
```

This function affect a MAC address to the BLE device. It is important to note that
this function allow a user to set any kind of address including random private
resolvable addresses and random private non resolvable addresses. For those types
of addresses, the address parameter is simply ignored and it is expected that the
implementation generate a new address of the address type requested.

#### Possible implementation

```c++
ble_error_t GapFoo::setAddress(BLEProtocol::AddressType_t type, const BLEProtocol::AddressBytes_t address) {
    int status;
    switch (type) {
        case BLEProtocol::AddressType::PUBLIC:
            status = blefoo_set_public_address(address);
            break;
        case BLEProtocol::AddressType::RANDOM_STATIC:
            status = blefoo_set_random_static_address(address);
            break;
        case BLEProtocol::AddressType::RANDOM_PRIVATE_RESOLVABLE:
            status = blefoo_set_private_resolvable_address();
            break;
        case BLEProtocol::AddressType::RANDOM_PRIVATE_NON_RESOLVABLE:
            status = blefoo_set_private_non_resolvable_address();
            break;
        default:
            return BLE_ERROR_PARAM_OUT_OF_RANGE;
    }
    return translate_blefoo_error(status);
}
```


### `getAddress`

```c++
/**
 * Fetch the current BTLE MAC address and type.
 *
 * @param[out] type The current BLE address type.
 * @param[out] address The current BLE address.
 *
 * @return BLE_ERROR_NONE on success.
 */
ble_error_t Gap::getAddress(BLEProtocol::AddressType_t* type, BLEProtocol::AddressBytes_t address);
```

This function fetch the current address and address type of the device.

#### Possible implementation

```c++
ble_error_t GapFoo::getAddress(BLEProtocol::AddressType_t* type, BLEProtocol::AddressBytes_t address) {
    // call the stack to get the current address
    blefoo_address_type_t blefoo_type;
    blefoo_address_bytes_t blefoo_address;
    int err = blefoo_get_current_address(&blefoo_type, blefoo_address);
    if (err) {
        return translate_blefoo_error(err);
    }

    // convert from values from blefoo format to BLE_API format
    *type = translate_blefoo_address_type(blefoo_type);
    translate_blefoo_address_bytes(/*dest*/address, /*src*/blefoo_address);
    return BLE_ERROR_NONE;
}
```


### `getMinAdvertisingInterval`

```c++
/**
 * Get the minimum advertising interval in milliseconds for connectable
 * undirected and connectable directed event types supported by the
 * underlying BLE stack.
 *
 * @return Minimum Advertising interval in milliseconds for connectable
 * undirected and connectable directed event types.
 */
uint16_t Gap::getMinAdvertisingInterval() const;
```

If the stack follow the recommendations of the bluetooth specification, this
implementation is straightforward, it should just return the value 20. A lot of
stacks defines such constants in 625us units, a porter has to be careful to  
return them as milliseconds.

#### Possible implementation

```c++
uint16_t GapFoo::getMinAdvertisingInterval() const {
    // blefoo is compliant with bluetooth specifications:
    return 20;
}
```

### `getMinNonConnectableAdvertisingInterval`

```c++
/**
 * Get the minimum advertising interval in milliseconds for scannable
 * undirected and non-connectable undirected even types supported by the
 * underlying BLE stack.
 *
 * @return Minimum Advertising interval in milliseconds for scannable
 * undirected and non-connectable undirected event types.
 */
uint16_t Gap::getMinNonConnectableAdvertisingInterval() const;
```

If the stack follow the recommendations of the bluetooth specification, this
implementation is straightforward, it should just return the value 100. A lot of
stacks defines such constants in 625us units, a porter has to be careful to  
return them as milliseconds.

#### Possible implementation

```c++
uint16_t GapFoo::getMinNonConnectableAdvertisingInterval() const {
    // blefoo is compliant with bluetooth specifications:
    return 100;
}
```


### `getMaxAdvertisingInterval`

```c++
/**
 * Get the maximum advertising interval in milliseconds for all event types
 * supported by the underlying BLE stack.
 *
 * @return Maximum Advertising interval in milliseconds.
 */
uint16_t Gap::getMaxAdvertisingInterval() const;
```

If the stack follow the recommendations of the bluetooth specification, this
implementation is straightforward, it should just return the value 10240. A lot
of stacks defines such constants in 625us units, a porter has to be careful to  
return them as milliseconds.

#### Possible implementation

```c++
uint16_t GapFoo::getMaxAdvertisingInterval() const {
    // blefoo is compliant with bluetooth specifications:
    return 10240;
}
```


### `stopAdvertising`

```c++
/**
 * Stop advertising. The current advertising parameters remain in effect.
 *
 * @retval BLE_ERROR_NONE if successfully stopped advertising procedure.
 */
ble_error_t Gap::stopAdvertising();
```

This function will request the underlying ble stack to stop advertisements. If
the advertisements stops, the implementation should set the field `advertising`
of the flag `Gap::state` to the value 0;

#### Possible implementation

```c++
ble_error_t GapFoo::stopAdvertising() {
    int err = blefoo_stop_advertising();
    if (err) {
        return translate_blefoo_error(err);
    }

    // update the GapState
    state.advertising = 0;
    return BLE_ERROR_NONE;
}
```

### `stopScan`

```c++
/**
 * Stop scanning. The current scanning parameters remain in effect.
 *
 * @retval BLE_ERROR_NONE if successfully stopped scanning procedure.
 */
ble_error_t Gap::stopScan();
```

This function will just request the underlying ble stack to stop the scanning process.

#### Possible implementation

```c++
ble_error_t GapFoo::stopScan() {
    return translate_blefoo_error(blefoo_stop_scan());
}
```


### `connect`

```c++
/**
 * Create a connection (GAP Link Establishment).
 *
 * @param[in] peerAddr 48-bit address, LSB format.
 * @param[in] peerAddrType Address type of the peer.
 * @param[in] connectionParams Connection parameters.
 * @param[in] scanParams Paramters to be used while scanning for the peer.
 *
 * @return  BLE_ERROR_NONE if connection establishment procedure is started
 * successfully. The connectionCallChain (if set) will be invoked upon a
 * connection event.
 */
virtual ble_error_t connect(
    const BLEProtocol::AddressBytes_t  peerAddr,
    BLEProtocol::AddressType_t         peerAddrType,
    const ConnectionParams_t* connectionParams,
    const GapScanningParams* scanParams
);
```

This function initiate a connection procedure to a peer. Following the start
of the connection procedure, it is expected that the ble stack generate
several events:
* If the connection succeed, it is expected that the port of BLE_API call the
function [`Gap::processConnectionEvent`] to report to the user the connection event.
```
         GapImplementation                  BLEStack
               +--+--+                      +--+--+
                  |                            |
                  |                            |
     connect()---->                            |
                  |  Initiate connection       |
                  |---------------------------->
                  |                            |
BLE_ERROR_NONE<---|                            |
                  |   Connection success       |
                  <~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
                  |                            |
       processConnectionEvent(...)             |
                  |                            |
 user callback<---|                            |
                  +                            +
```

* If the connection process timeout, it is expected that the implementation
call the function [`Gap::processTimeout`] with `Gap::TIMEOUT_SRC_CONN` as
parameter.
```
         GapImplementation                  BLEStack
               +--+--+                      +--+--+
                  |                            |
                  |                            |
     connect()---->                            |
                  |  Initiate connection       |
                  |---------------------------->
                  |                            |
BLE_ERROR_NONE<---|                            |
                  |   Connection timeout       |
                  <~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
                  |                            |
  processTimeout(Gap::TIMEOUT_SRC_CONN)        |
                  |                            |
 user callback<---|                            |
                  +                            +
```

* When the connections end, it is expected that the implementation
call the function [`Gap::processDisconnectionEvent`] with the handle of the
connection which end and the disconnection reason as a parameter.
```
         GapImplementation                  BLEStack
               +--+--+                      +--+--+
                  |                            |
                  |                            |
     connect()---->                            |
                  |  Initiate connection       |
                  |---------------------------->
                  |                            |
BLE_ERROR_NONE<---|                            |
                  |   Connection success       |
                  <~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
                  /                            /
                  /                            /
                  /                            /
                  |       disconnection        |
                  <~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
                  |                            |
      processDisconnectionEvent(...)           |
                  |                            |                                    
 user callback<---|                            |
                  +                            +
```

If parameters in input are not valid or if the stack cannot process the request,
none of this events should be generated, instead the function should return
an appropriate error code.


### `disconnect`

```c++
/**
 * This call initiates the disconnection procedure, and its completion will
 * be communicated to the application with an invocation of the
 * disconnectionCallback.
 *
 * @param[in] reason The reason for disconnection; to be sent back to the peer.
 * @param[in] connectionHandle The handle of the connection to disconnect from.
 *
 * @return  BLE_ERROR_NONE if disconnection was successful.
 */
virtual ble_error_t disconnect(Handle_t connectionHandle, DisconnectionReason_t reason);
```

This function initiate a disconnection procedure. Once the disconnection is done,
the stack should call the function [`Gap::processDisconnectionEvent`]:

```
         GapImplementation                  BLEStack
               +--+--+                      +--+--+
                  |                            |
                  |                            |
  disconnect()---->                            |
                  |  Initiate disconnection    |
                  |---------------------------->
                  |                            |
BLE_ERROR_NONE<---|                            |
                  |      disconnection         |
                  <~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
                  |                            |
      processDisconnectionEvent(...)           |
                  |                            |                                    
 user callback<---|                            |
                  +                            +
```

If parameters in input are not valid or if the stack cannot process the request,
the function should return an appropriate error code.


### `setPreferredConnectionParams`

```c++
/**
 * Set the GAP peripheral preferred connection parameters. These are the
 * defaults that the peripheral would like to have in a connection. The
 * choice of the connection parameters is eventually up to the central.
 *
 * @param[in] params The structure containing the desired parameters.
 *
 * @return BLE_ERROR_NONE if the preferred connection params were set
 * correctly.
 */
virtual ble_error_t setPreferredConnectionParams(const ConnectionParams_t* params);
```

This function set the preferred connection parameters for the peripheral role.
These parameters should be visible in the characteristic  
`Peripheral Prefered Connection Parameters` of the `Generic Access Service`.


### `updateConnectionParams`

```c++
/**
 * Update connection parameters. In the central role this will initiate a
 * Link Layer connection parameter update procedure. In the peripheral role,
 * this will send the corresponding L2CAP request and wait for the central
 * to perform the procedure.
 *
 * @param[in] handle Connection Handle.
 * @param[in] params Pointer to desired connection parameters. If NULL is provided
 * on a peripheral role, the parameters in the PPCP characteristic of the GAP
 * service will be used instead.
 *
 * @return BLE_ERROR_NONE if the connection parameters were updated correctly.
 */
virtual ble_error_t updateConnectionParams(Handle_t handle, const ConnectionParams_t* params);
```

This function initiate a `Connection Parameters Request Procedure` for a given
connection. There is no user callback to call at the end of the procedure.


### `setDeviceName`

```c++
/**
 * Set the device name characteristic in the GAP service.
 *
 * @param[in] deviceName The new value for the device-name. This is a UTF-8
 * encoded, <b>NULL-terminated</b> string.
 *
 * @return BLE_ERROR_NONE if the device name was set correctly.
 */
virtual ble_error_t setDeviceName(const uint8_t* deviceName);
```

Set the name of the device. This change should be visible in the
`Device Name Characteristic` of the `Generic Access Service`.


### `getDeviceName`

```c++
/**
 * Get the value of the device name characteristic in the GAP service.
 *
 * @param[out] deviceName Pointer to an empty buffer where the UTF-8 *non NULL-
 * terminated* string will be placed. Set this value to NULL in order to obtain
 * the deviceName-length from the 'length' parameter.
 *
 * @param[in,out] lengthP
 *  - (on input) Length of the buffer pointed to by deviceName;
 *  - (on output) the complete device name length (without the null terminator).
 *
 * @return BLE_ERROR_NONE if the device name was fetched correctly from the
 * underlying BLE stack.
 *
 * @note If the device name is longer than the size of the supplied buffer,
 * length will return the complete device name length, and not the number of bytes
 * actually returned in deviceName. The application may use this information to
 * retry with a suitable buffer size.
 */
virtual ble_error_t getDeviceName(uint8_t* deviceName, unsigned* lengthP);
```

This function fetch the value of the `Device Name Characteristic` of the
`Generic Access Service`.


### `setAppearance`

```c++
/**
 * Set the appearance characteristic in the GAP service.
 *
 * @param[in] appearance The new value for the device-appearance.
 *
 * @return BLE_ERROR_NONE if the new appearance was set correctly.
 */
virtual ble_error_t setAppearance(GapAdvertisingData::Appearance appearance);
```

This function update the value of the `Appearance Characteristic` of the `Generic
Access Service`.

### `getAppearance`

```c++
/**
 * Get the appearance characteristic in the GAP service.
 *
 * @param[out] appearance
 *               The current device-appearance value.
 *
 * @return BLE_ERROR_NONE if the device-appearance was fetched correctly
 *         from the underlying BLE stack.
 */
virtual ble_error_t getAppearance(GapAdvertisingData::Appearance* appearanceP);
```

This function fetch the value of the `Appearance Characteristic` of the
`Generic Access Service`.


### `setTxPower`

```c++
/**
 * Set the radio's transmit power.
 *
 * @param[in] txPower Radio's transmit power in dBm.
 *
 * @return BLE_ERROR_NONE if the new radio's transmit power was set correctly.
 */
virtual ble_error_t setTxPower(int8_t txPower);
```


### `getPermittedTxPowerValues`

```c++
/**
 * Query the underlying stack for permitted arguments for setTxPower().
 *
 * @param[out] valueArrayPP Out parameter to receive the immutable array of Tx values.
 * @param[out] countP Out parameter to receive the array's size.
 */
virtual void getPermittedTxPowerValues(const int8_t** valueArrayPP, size_t* countP);
```


### `getMaxWhitelistSize`

```c++
/**
 * Get the maximum size of the whitelist.
 *
 * @return Maximum size of the whitelist.
 *
 * @experimental
 */
virtual uint8_t getMaxWhitelistSize() const;
```

This API query the maximum number of addresses that can be contained by the
ble underlying stack.


### `getWhitelist`

```c++
/**
 * Get the internal whitelist to be used by the Link Layer when scanning,
 * advertising or initiating a connection depending on the filter policies.
 *
 * @param[in,out]   whitelist
 * - (on input) whitelist.capacity contains the maximum number of addresses to
 *   be returned.
 * - (on output) The populated whitelist with copies of the addresses in the
 *   implementation's whitelist.
 *
 * @return BLE_ERROR_NONE if the implementation's whitelist was successfully
 * copied into the supplied reference.
 *
 * @experimental
 */
virtual ble_error_t getWhitelist(Whitelist_t& whitelist) const;
```


### `setWhitelist`

```c++
/**
 * Set the internal whitelist to be used by the Link Layer when scanning,
 * advertising or initiating a connection depending on the filter policies.
 *
 * @param[in] whitelist A reference to a whitelist containing the addresses to
 * be added to the internal whitelist.
 *
 * @return BLE_ERROR_NONE if the implementation's whitelist was successfully
 * populated with the addresses in the given whitelist.
 *
 * @note The whitelist must not contain addresses of type @ref
 *       BLEProtocol::AddressType_t::RANDOM_PRIVATE_NON_RESOLVABLE, this
 *       this will result in a @ref BLE_ERROR_INVALID_PARAM since the
 *       remote peer might change its private address at any time and it
 *       is not possible to resolve it.
 * @note If the input whitelist is larger than @ref getMaxWhitelistSize()
 *       the @ref BLE_ERROR_PARAM_OUT_OF_RANGE is returned.
 *
 * @experimental
 */
virtual ble_error_t setWhitelist(const Whitelist_t& whitelist);
```

### `setAdvertisingPolicyMode`

```c++
/**
 * Set the advertising policy filter mode to be used in the next call
 * to startAdvertising().
 *
 * @param[in] mode The new advertising policy filter mode.
 *
 * @return BLE_ERROR_NONE if the specified policy filter mode was set
 * successfully.
 *
 * @experimental
 */
virtual ble_error_t setAdvertisingPolicyMode(AdvertisingPolicyMode_t mode);
```

The stack should apply this policy when the device start advertising.

### `getAdvertisingPolicyMode`

```c++
/**
 * Get the advertising policy filter mode that will be used in the next
 * call to startAdvertising().
 *
 * @return The set advertising policy filter mode.
 *
 * @experimental
 */
virtual AdvertisingPolicyMode_t getAdvertisingPolicyMode() const;
```


### `setScanningPolicyMode`

```c++
/**
 * Set the scan policy filter mode to be used in the next call
 * to startScan().
 *
 * @param[in] mode The new scan policy filter mode.
 *
 * @return BLE_ERROR_NONE if the specified policy filter mode was set
 *         successfully.
 *
 * @experimental
 */
virtual ble_error_t setScanningPolicyMode(ScanningPolicyMode_t mode);
```

The stack should apply this policy when the device start scanning.


### `getScanningPolicyMode`

```c++
/**
 * Get the scan policy filter mode that will be used in the next
 * call to startScan().
 *
 * @return The set scan policy filter mode.
 *
 * @experimental
 */
virtual ScanningPolicyMode_t getScanningPolicyMode() const;
```

### `setInitiatorPolicyMode`

```c++
/**
 * Set the initiator policy filter mode to be used.
 *
 * @param[in] mode
 *              The new initiator policy filter mode.
 *
 * @return BLE_ERROR_NONE if the specified policy filter mode was set
 *         successfully.
 *
 * @experimental
 */
virtual ble_error_t setInitiatorPolicyMode(InitiatorPolicyMode_t mode);
```

### `getInitiatorPolicyMode`

```c++
/**
* Get the initiator policy filter mode that will be used.
*
* @return The set scan policy filter mode.
*
* @experimental
*/
virtual InitiatorPolicyMode_t getInitiatorPolicyMode() const;
```


### `startRadioScan`

```c++
/**
 * Start scanning procedure in the underlying BLE stack.
 *
 * @param[in] scanningParams The GAP scanning parameters.
 *
 * @return BLE_ERROR_NONE if the scan procedure started successfully.
 */
virtual ble_error_t startRadioScan(const GapScanningParams &scanningParams);
```

This function start a scan procedure if the parameter `scanningParams` is suitable
for the underlying BLE_STACK. The scan procedure should follow the parameters
provided by `scanningParams`.

Every time the device receive an advertisement packet, the function
`Gap::processAdvertisementReport` should be called to report to the upper layer
that a packet has been received.

```
               GapImplementation                        BLEStack
                  +----+----+                         +----+----+
                       |                                   |
Gap::startScan() +----->                                   |
                       |        start_scan()               |
                       +----------------------------------->
                       |        scan started               |
                       <-----------------------------------|
  BLE_ERROR_NONE <-----|                                   |
                       /                                   /
                       |                                   <~~~~~~~~~+ advertisement report
                       /       advertisement report        /
                       <~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
                       |                                   |
       Gap::processAdvertisementReport()                   |
                       |                                   |
   advertisement <-----+                                   |
                       /                                   /
                       |                                   <~~~~~~~~~+ advertisement report
                       /       advertisement report        /
                       <~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
                       |                                   |
       Gap::processAdvertisementReport()                   |
                       |                                   |
   advertisement <-----+                                   |
                       /                                   /
                       /                                   /
                       +                                   +
```

If the scan is active, a call to `Gap::stopScan()` or `Gap::reset()` should stop
the scan.

```
               GapImplementation                        BLEStack
                  +----+----+                         +----+----+
                       |                                   |
Gap::startScan() +----->                                   |
                       |        start_scan()               |
                       +----------------------------------->
                       |                                   |
                       /                                   /
                       /                                   /
 Gap::stopScan() or    /                                   /
 Gap::reset()    +----->                                   |
                       |        stop_scan()                |
                       +----------------------------------->
                       |        scan_stopped               |
                       <-----------------------------------|
  BLE_ERROR_NONE <-----|                                   |
                       |                                   |
                       +                                   +
```

If `scanningParams.getTimeout()` is not equal to 0, the device should scan for a
period equal to the value of `scanningParams.getTimeout()` in seconds. Once the
device has scanned for this period of time, the implementation should call
`Gap:processTimeout()` with `Gap::TIMEOUT_SRC_SCAN` as  parameter.

```
               GapImplementation                        BLEStack
                  +----+----+                         +----+----+
                       |                                   |
Gap::startScan()       |                                   |
where timeout=N  +----->                                   |
                       |        start_scan()               |
                       +----------------------------------->
                       /                                   /
                       /                                   /
                       <~~~~~~+ scan timeout (N s later)   |
                       |                                   |
                       |        stop_scan()                |
                       +----------------------------------->
                       |        scan_stopped               |
                       <-----------------------------------|
                       |                                   |
Gap:processTimeout() <-+                                   |
                       |                                   |
                       +                                   +
Note: timeout event may come from the stack in this case, it is likely that there
is no need to stop the scan "manually".
```

If `scanningParams.getTimeout()` is equal to 0, the scan should continue until
`Gap::stopScan` or `Gap::reset` is called.

If the function is called while a scan is ongoing, new parameters scan parameters
are applied and the timeout is reset to the new value.

If `scanningParams.getActiveScanning()` is equal to true, the stack should
generate scan requests and report scan response receive using the function
`Gap::processAdvertisementReport()` otherwise, the stack should not generate
scan request and should not report scan responses.

If the scanning policy is set to `SCAN_POLICY_FILTER_ALL_ADV`, advertisements from
devices which are not in the whitelist should not be reported.


### `initRadioNotification`

Implementation of this API is optional.

```c++
/**
 * Initialize radio-notification events to be generated from the stack.
 * This API doesn't need to be called directly.
 *
 * Radio Notification is a feature that enables ACTIVE and INACTIVE
 * (nACTIVE) signals from the stack that notify the application when the
 * radio is in use.
 *
 * The ACTIVE signal is sent before the radio event starts. The nACTIVE
 * signal is sent at the end of the radio event. These signals can be used
 * by the application programmer to synchronize application logic with radio
 * activity. For example, the ACTIVE signal can be used to shut off external
 * devices, to manage peak current drawn during periods when the radio is on,
 * or to trigger sensor data collection for transmission in the Radio Event.
 *
 * @return BLE_ERROR_NONE on successful initialization, otherwise an error code.
 */
virtual ble_error_t initRadioNotification();
```

TODO: document this when there is a way to stop it ....


### `setAdvertisingData`

```c++
/**
 * Functionality that is BLE stack-dependent and must be implemented by the
 * ported. This is a helper function to set the advertising data in the
 * BLE stack.

 * @param[in] advData The new advertising data.
 * @param[in] scanResponse The new scan response data.
 *
 * @return BLE_ERROR_NONE if the advertising data was set successfully.
 */
virtual ble_error_t setAdvertisingData(const GapAdvertisingData &advData, const GapAdvertisingData &scanResponse);
```

This function set advertising data's and scan response data's. If the device is
already scanning, the new values should be for the next round of advertisement.

### `startAdvertising`

```c++
/**
 * Functionality that is BLE stack-dependent and must be implemented by the
 * ported. This is a helper function to start the advertising procedure in
 * the underlying BLE stack.
 *
 * @param[in] The advertising parameters.
 *
 * @return BLE_ERROR_NONE if the advertising procedure was successfully
 * started.
 */
virtual ble_error_t startAdvertising(const GapAdvertisingParams& advertisingParams);
```

This function start an advertising procedure if the advertising configuration
contained in `advertisingParams` is valid. Otherwise, it return an appropriate
error code.

```
                   GapImplementation                        BLEStack
                      +----+----+                         +----+----+
                           |                                   |
Gap::startAdvertising() +-->                                   |
                           |        start_advertising()        |
                           +----------------------------------->
                           |        advertising started        |
                           <-----------------------------------|
      BLE_ERROR_NONE <-----|                                   |
                           /                                   /
                           /                                   /~~~> advertisement
                           /                                   /
                           /                                   /~~~> advertisement
                           /                                   /
                           /                                   /~~~> advertisement                                                       
                           /                                   /
                           +                                   +
```

If advertising is active, a call to `Gap::stopAdvertising()` or `Gap::reset()`
should stop advertisings.

```
                   GapImplementation                        BLEStack
                      +----+----+                         +----+----+
                           |                                   |
Gap::startAdvertising() +-->                                   |
                           |        start_advertising()        |
                           +----------------------------------->
                           |                                   |                           
                           /                                   /
                           /                                   /
Gap::stopAdvertising() or  /                                   /
     Gap::reset()    +----->                                   |
                           |        stop_advertising()         |
                           +----------------------------------->
                           |        advertising stopped        |
                           <-----------------------------------|                                
      BLE_ERROR_NONE <-----|                                   |
                           |                                   |
                           +                                   +
```


If `advertisingParams.getTimeout()` is not equal to 0, the device should advertise
for a period equal to the value of `advertisingParams.getTimeout()` in seconds.
Once the device has advertised for this period of time, the implementation should
call `Gap:processTimeout()` with `Gap::TIMEOUT_SRC_ADVERTISING` as  parameter.

```
                   GapImplementation                            BLEStack
                      +----+----+                              +----+----+
                           |                                        |
  Gap::startAdvertising()  |                                        |
    where timeout=N  +----->                                        |
                           |        start_advertising()             |
                           +---------------------------------------->
                           /                                        /
                           /                                        /
                           <~~~~~~+ advertising timeout (N s later) |
                           |                                        |
                           |        stop_advertising()              |
                           +---------------------------------------->
                           |        scan_stopped                    |
                           <----------------------------------------|
                           |                                        |
    Gap:processTimeout() <-+                                        |
                           |                                        |
                           +                                        +
Note: timeout event may come from the stack in this case, it is likely that there
is no need to stop advertisements "manually".
```


If `advertisingParams.getTimeout()` is equal to 0, advertising should continue until
`Gap::stopAdvertising` or `Gap::reset` is called.

If the function is called while advertising is ongoing, new parameters scan parameters
are applied and the timeout is reset to the new value.

If `advertisingParams.getAdvertisingType()` is equal to
`GapAdvertisingParams::ADV_CONNECTABLE_UNDIRECTED`  or
`GapAdvertisingParams::ADV_SCANNABLE_UNDIRECTED`, the stack should
generate a scan response with the scan response payload set by `Gap::setAdvertisingData`
if a scanner make a scan request. Otherwise, the device should not respond scan
request.

If `advertisingParams.getAdvertisingType()` is equal to
`GapAdvertisingParams::ADV_NON_CONNECTABLE_UNDIRECTED`, the stack not should
listen for scan requests.

If `advertisingParams.getAdvertisingType()` is equal to
`GapAdvertisingParams::ADV_CONNECTABLE_UNDIRECTED`, the stack should listen and
accept connections request.

If `advertisingParams.getAdvertisingType()` is equal to
`GapAdvertisingParams::ADV_SCANNABLE_UNDIRECTED` or
`GapAdvertisingParams::ADV_NON_CONNECTABLE_UNDIRECTED`, the stack should not
for connections request.

If the advertising policy is set to `ADV_POLICY_FILTER_SCAN_REQS`, scan requests
from devices which are not in the whitelist should be ignored.

If the advertising policy is set to `ADV_POLICY_FILTER_CONN_REQS`, connection
requests from devices which are not in the whitelist should be ignored.

If the advertising policy is set to `ADV_POLICY_FILTER_ALL_REQS`, scan and
connection requests from devices which are not in the whitelist should be
ignored.


### `reset`

```c++
/**
 * Notify all registered onShutdown callbacks that the Gap instance is
 * about to be shutdown and clear all Gap state of the
 * associated object.
 *
 * This function is meant to be overridden in the platform-specific
 * sub-class. Nevertheless, the sub-class is only expected to reset its
 * state and not the data held in Gap members. This shall be achieved by a
 * call to Gap::reset() from the sub-class' reset() implementation.
 *
 * @return BLE_ERROR_NONE on success.
 *
 * @note  Currently a call to reset() does not reset the advertising and
 * scan parameters to default values.
 */
virtual ble_error_t reset();
```

This function reset the state of the [`Gap`] object.

All ongoing operations launched by [`Gap`] should stop (connections, scan,
advertising, radio notification).

[`Gap`] should rollback to its initial state (advertising, scan parameters,
advertising payload, scan payload ), ...

An implementation should call this function from the base (`Gap::reset()`) as its
first statement. The implementation from [`Gap`] does the following things:
* calling the shutdownCallChain
* reset `Gap::state`
* reset all callbacks.


[ble module]: https://github.com/ARMmbed/ble
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
