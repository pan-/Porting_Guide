# BLEInstanceBase

* `createBLEInstance`: why is it call create if it is not a creation that is
expected ???
* What about virtual destructor ???
* `init`: What is the deal with BLE::InstanceID_t, it is pointless since other
members does not takes this parameter....
* `getGattClient const`: where it is ???
* `init`: two returns mechanism for the sake of "backward" compatibility seems
irrelevant
* `reset`: How this function can fail, what is possible to do in such case ?
* `getVersion`: Why on hearth this is not const ???


# Gap
`setAddress`: Optional arguments depending on other arguments values, what a
great example of a kitchensink API. What about the address parameter, is it in/out
in case of PRIVATE_RESOLVABLE_ADDRESS ?
`getAddress`: Why not const ?
`GapState_t`: Why fields are not bool ?
`connection timeout`: It should be possible to know which connection started timeout.
`updateConnectionParams`: Why there is no user callback for this ??
`startScan` is inconstitent with other APIs.
where is `processRadioNotification` ?
where is the function stopRadioNotification ?
`reset`: what a mess, it is difficult to state what the implementation should do +
forcing the porter to call the function from the base as a start is a design defect.


# GattServer
* `handleEvent` why events are multiplexed here, it is not consistent with the rest
of the API.
* Where are reliable write and writable auxiliaries properties ?
* `GattServer::serviceCount` and `GattServer::characteristicCount` has no reasons
to exist at this level, it is an implementation detail.
* `handleDataReadEvent` is not called when data is read
* `areUpdatesEnabled`: Why is this virtual ? It can be build upon `read` if the
CCCD handle is set when a characteristic is registered. Plus, there is no way to
know if indication or notification are enabled.
* `areUpdatesEnabled`: without connection handle, this function does not make sense.
* if an implementation does not allow to read value of GATT Attribute, applications
are not portable at all.
* `GattCharacterstic::setXXXAuthorizationCallback` is buggy, setting an authorization
callback once the characteristic has already been registered in a GattServer will
not does what is expected.
* `read` and `write` have shared security.
