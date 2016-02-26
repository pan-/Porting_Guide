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
