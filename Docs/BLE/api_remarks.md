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