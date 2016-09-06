# SecurityManager

This document provides a porting guide for the class [`SecurityManager`]
available in [BLE_API] version 2.5.1.

The [`SecurityManager`] API is declared in the header file [`SecurityManager.h`].

## Description

[`SecurityManager`] is an abstract class of BLE_API which provide function to
 manage the security of a BLE link.

A valid port of BLE_API should provide an implementation for this class.

## API


### `init`

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
ble_error_t SecurityManager::init(
    bool enableBonding = true,
    bool requireMITM = true,
    SecurityIOCapabilities_t iocaps  = IO_CAPS_NONE,
    const Passkey_t passkey = NULL
);
```


### `getLinkSecurity`

```c++
/**
 * Get the security status of a connection.
 *
 * @param[in]  connectionHandle   Handle to identify the connection.
 * @param[out] securityStatusP    Security status.
 *
 * @return BLE_ERROR_NONE or appropriate error code indicating the failure reason.
 */
ble_error_t SecurityManager::getLinkSecurity(
    Gap::Handle_t connectionHandle,
    LinkSecurityStatus_t* securityStatusP
);
```

### `setLinkSecurity`

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
ble_error_t SecurityManager::setLinkSecurity(Gap::Handle_t connectionHandle, SecurityMode_t securityMode);
```


### `purgeAllBondingState`

```c++
/**
 * Delete all peer device context and all related bonding information from
 * the database within the security manager.
 *
 * @retval BLE_ERROR_NONE             On success, else an error code indicating reason for failure.
 * @retval BLE_ERROR_INVALID_STATE    If the API is called without module initialization or
 *                                    application registration.
 */
ble_error_t SecurityManager::purgeAllBondingState(void);
```


### `getAddressesFromBondTable`

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
ble_error_t SecurityManager::getAddressesFromBondTable(Gap::Whitelist_t& addresses) const;
```

### `reset`

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
ble_error_t SecurityManager::reset(void);
```


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
