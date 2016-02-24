# BLEInstanceBase

This document provides a porting guide for the class [`BLEInstanceBase`] available
in the [ble module] version 2.5.1.

The [`BLEInstanceBase`] API is declared in the header file [`BLEInstanceBase.h`].


## Description

[`BLEInstanceBase`] is the private implementation of the class [`BLE`]. It is a
pure abstract base class which provide the following functionalities:
* Manage the state of the underlying BLE stack.
* Access to instances of abstractions of BLE_API: ([`Gap`], [`GattServer`], [`GattClient`],
[`SecurityManager`]).  

Since abstractions of BLE_API are accessed through this class, it is common that
implementations of this class contains the instances of those abstractions.  

## Requirements

Some more info about the module comes here.

## Note

For the rest of this document, the fictive name of the port used in implementation
example is `Foo`. In this port, the following classes exists:
* `BLEInstanceFoo`: implement [`BLEInstanceBase`]
* `GapFoo`: implement [`Gap`]
* `GattServerFoo`: implement [`GattServer`]
* `GattClientFoo`: implement [`GattClient`]
* `SecurityManagerFoo`: implement [`SecurityManager`]


```c++
class BLEInstanceFoo : public BLEInstance {

    typedef FunctionPointerWithContext<BLE::InitializationCompleteCallbackContext*> InitCallback_t;

public:
    friend BLEInstanceBase* createBLEInstance(void);

    BLEInstanceFoo(/*constructor arguments*/);

    virtual ble_error_t init(BLE::InstanceID_t instanceID, InitCallback_t initCallback);
    virtual ble_error_t shutdown(void);
    virtual bool hasInitialized(void) const;

    virtual const char* getVersion(void);

    virtual GapFoo& getGap();
    virtual const GapFoo& getGap() const;

    virtual GattServerFoo& getGattServer();
    virtual const GattServerFoo& getGattServer() const;

    virtual GattClientFoo& getGattClient();

    virtual SecurityManagerFoo& getSecurityManager();
    virtual const SecurityManagerFoo& getSecurityManager() const;

    virtual void waitForEvent(void);

private:

    private void whenStackInitialized(/*parameters*/ int error);

    // store needed data for the init procedure
    BLE::InstanceID_t initInstanceID; /*= 0*/
    InitCallback_t initCallback; /*= NULL*/

    // singleton to instances of the other abstractions of BLE API
    // Such instances will be allocated and instantiated upon request.  
    mutable GapFoo* gapInstance; /*= NULL*/
    mutable GattServerFoo* gattServerInstance; /*= NULL*/
    mutable GattClientFoo* gattClientInstance; /*= NULL*/
    mutable SecurityManagerFoo* securityManagerInstance; /*= NULL*/

    // singleton of this class
    static BLEInstanceFoo instance;
};

```





## API

### `createBLEInstance`

```c++
/**
 * Get the instance of the implementation of BLEInstanceBase.
 */
extern BLEInstanceBase* createBLEInstance(void);
```

This function is used by BLE_API to get access to the instance of the
implementation of [`BLEInstanceBase`]. Its name can be misleading, because it is
not expected that a new instance of [`BLEInstanceBase`] will be created at each
call. Conceptually, it is equivalent to a function `getInstance` of a singleton.
It is also important to note that BLE_API will **never** delete objects resulting
from this call.

#### Possible implementation

It is recommended to implement this function as a static singleton.

```c++
// use static storage for this variable, it will lives for the whole duration
// of the program.
// Usage of dynamic memory is strongly discouraged here since BLE_API will never
// delete variable acquired with createBLEInstance.
BLEInstanceFoo BLEInstanceFoo::instance = BLEInstanceFoo(/*constructor arguments*/);

BLEInstanceBase* createBLEInstance(void) {
    return &BLEInstanceFoo::instance;
}
```


### `init`

```c++
/**
 * Initialize the underlying BLE stack.
 *
 * @param[in] instanceID The ID of the instance to initialize.
 * @param[in] initCallback A callback for when initialization completes for a BLE
 * instance.
 *
 * @return BLE_ERROR_NONE if the initialization procedure was started successfully.
 */
ble_error_t BLEInstanceBase::init(
    BLE::InstanceID_t instanceID,
    FunctionPointerWithContext<BLE::InitializationCompleteCallbackContext*> initCallback
);
```

This function is called when a user call the function `BLE::init`, this function
is supposed to initialize the underlying Bluetooth stack and what needs to be
initialized in the implementation of `BLEInstanceBase`.

The first parameter, `instanceID` is used to retrieve the BLE instance which make
this request. The second parameter, `initCallback` is the completion handler of
this initialization process. It takes an [`InitializationCompleteCallbackContext`]
as a parameter. This callback should be called when the initialization procedure ends.

#### Possible implementation

```c++
typedef FunctionPointerWithContext<BLE::InitializationCompleteCallbackContext*> InitCallback_t;

ble_error_t BLEInstanceFoo::init(BLE::InstanceID_t instanceID, InitCallback_t callback) {

    // check if it is already initialized
    if (hasInitialized()) {
        BLE::InitializationCompleteCallbackContext result = {
            BLE::Instance(instanceID),
            BLE_ERROR_ALREADY_INITIALIZED
        };
        callback(result);
        return BLE_ERROR_ALREADY_INITIALIZED;
    }

    // launch the initialization of the stack, it returns some kind of error
    // and will call the function BLEInstanceFoo::whenStackInitialized when
    // the initialization process is done
    int err = blefoo_init(BLEInstanceFoo::whenStackInitialized);
    if(err) {
        // translate error from the stack to ble_error_t
        ble_error_t error = translate_blefoo_error(err);

        BLE::InitializationCompleteCallbackContext result = {
            BLE::Instance(instanceID),
            error
        };
        callback(result);
        return error;
    }

    // process correctly launched, it is safe to store arguments now
    // they will be used later by whenStackInitialized
    initInstanceID = instanceID;
    initCallback = callback;

    return BLE_ERROR_NONE;
}

void BLEInstanceFoo::whenStackInitialized(int error) {
    // access to the singleton of BLEINstanceFoo
    BLEInstanceFoo& self = BLEInstanceFoo::instance;

    BLE::InitializationCompleteCallbackContext result = {
        BLE::Instance(self.initInstanceID),
        translate_blefoo_error(error)
    };

    self.initCallback(result);
}
```


### `hasInitialized`

```c++
/**
 * Check whether the underlying stack has already been initialized,
 * possible with a call to init().
 *
 * @return true if the initialization has completed for the underlying BLE stack.
 */
bool BLEInstanceBase::hasInitialized(void) const;
```

#### Possible implementation

```c++
bool BLEInstanceFoo::hasInitialized(void) const {
    return blefoo_has_initialized();
}
```


### `shutdown`

```c++
/**
 * Shutdown the underlying BLE stack. This includes purging the stack of
 * GATT and GAP state and clearing all state from other BLE components
 * such as the SecurityManager. init() must be called afterwards to
 * re-instantiate services and GAP state.
 *
 * @return BLE_ERROR_NONE if the underlying stack and all other services of
 * the BLE API were shutdown correctly.
 */
virtual ble_error_t shutdown(void) = 0;
```

Clearing all states here means calling the member function `reset` of the instances
of [`Gap`], [`GattServer`], [`GattClient`] and [`SecurityManager`]. If those
instances are instantiated on demand, it is good to delete them in this function.

#### Possible implementation

```c++
ble_error_t BLEInstanceFoo::shutdown() {
    // it is not possible to shutdown an instance which is not initialized
    if(!hasInitialized()) {
        return BLE_ERROR_INITIALIZATION_INCOMPLETE;        
    }

    // In this case, we take advantage of RAII,
    // the reset function is called in the destructor of
    // GapFoo, GattServerFoo, GattClientFoo and SecurityManagerFoo
    delete gapInstance;
    gapInstance = NULL;
    delete gattServerInstance;
    gattServerInstance = NULL;
    delete gattClientInstance;
    gattClientInstance = NULL;
    delete securityManagerInstance;
    securityManagerInstance = NULL;

    // pseudo function which shutdown the stack
    // it will also update its internal state to not initialized
    int error = blefoo_shutdown_stack();
    if(error) {
        return translate_blefoo_error(err);
    }

    return BLE_ERROR_NONE;
}
```


### `getVersion`

```c++
/**
 * Fetches a string representation of the underlying BLE stack's version.
 *
 * @return A pointer to the string representation of the underlying BLE stack's
 * version.
 */
const char* BLEInstanceBase::getVersion(void);
```

There is no particular result expected for this function. It is here to provide
insight of what is the version of the stack used.

#### Possible implementation

```c++
const char* BLEInstanceFoo::getVersion() {
    // pseudo function which return the version of the BLE stack as a string
    return blefoo_get_stack_version();
}
```


### `getGap`

```c++
/**
 * Accessor to Gap. This function is used by BLE::gap().
 *
 * @return A reference to a Gap object associated to this BLE instance.
 */
Gap& BLEInstanceBase::getGap();
const Gap& BLEInstanceBase::getGap() const;
```

This function return the unique instance of the implementation of [`Gap`].

#### Possible implementation

```c++
GapFoo& BLEInstanceFoo::getGap() {
    if(gapInstance == NULL) {
        gapInstance = new GapFoo(/*constructor arguments*/);
    }
    return *gapInstance;
}

const GapFoo& BLEInstanceFoo::getGap() const {
    if(gapInstance == NULL) {
        gapInstance = new GapFoo(/*constructor arguments*/);
    }
    return *gapInstance;
}
```



### `getGattServer`

```c++
/**
 * Accessor to GattServer. This function is used by BLE::gattServer().
 *
 * @return A reference to a GattServer object associated to this BLE instance.
 */
GattServer& BLEInstanceBase::getGattServer();
const GattServer& BLEInstanceBase::getGattServer() const;
```

This function return the unique instance of the implementation of [`GattServer`].

#### Possible implementation

```c++
GattServerFoo& BLEInstanceFoo::getGattServer() {
    if(gattServerInstance == NULL) {
        gattServerInstance = new GattServerFoo(/*constructor arguments*/);
    }
    return *gattServerInstance;
}

const GattServerFoo& BLEInstanceFoo::getGattServer() const {
    if(gattServerInstance == NULL) {
        gattServerInstance = new GattServerFoo(/*constructor arguments*/);
    }
    return *gattServerInstance;
}
```


### `getGattClient`

```c++
/**
 * Accessors to GattClient. This function is used by BLE::gattClient().
 *
 * @return A reference to a GattClient object associated to this BLE instance.
 */
GattClient& BLEInstanceBase::getGattClient();
const GattClient& BLEInstanceBase::getGattClient() const;
```

This function return the unique instance of the implementation of [`GattClient`].

#### Possible implementation

```c++
GattClientFoo& BLEInstanceFoo::getGattClient() {
    if(gattClientInstance == NULL) {
        gattClientInstance = new GattClientFoo(/*constructor arguments*/);
    }
    return *gattClientInstance;
}

const GattClientFoo& BLEInstanceFoo::getGattClient() const {
    if(gattClientInstance == NULL) {
        gattClientInstance = new GattClientFoo(/*constructor arguments*/);
    }
    return *gattClientInstance;
}
```


### `getSecurityManager`

```c++
/**
 * Accessors to SecurityManager. This function is used by BLE::securityManager().
 *
 * @return A reference to a SecurityManager object associated to this BLE instance.
 */
SecurityManager& BLEInstanceBase::getSecurityManager();
const SecurityManager& BLEInstanceBase::getSecurityManager() const;
```

This function return the unique instance of the implementation of [`SecurityManager`].

#### Possible implementation

```c++
SecurityManagerFoo& BLEInstanceFoo::getSecurityManager() {
    if(securityManagerInstance == NULL) {
        securityManagerInstance = new SecurityManagerFoo(/*constructor arguments*/);
    }
    return *securityManagerInstance;
}

SecurityManagerFoo& BLEInstanceFoo::getSecurityManager() const {
    if(securityManagerInstance == NULL) {
        securityManagerInstance = new SecurityManagerFoo(/*constructor arguments*/);
    }
    return *securityManagerInstance;
}
```


### `waitForEvent`

```c++
/**
 * Yield control to the BLE stack or to other tasks waiting for events.
 * refer to BLE::waitForEvent().
 */
void BLEInstanceBase::waitForEvent(void);
```

This is a sleep function that will return when there is an application-specific
interrupt, but the MCU might wake up several times before returning (to service
the stack). This is not always interchangeable with WFE().

This function is used by mbed-classic users to go to sleep when they use BLE_API.

#### Possible implementation

```c++
void BLEInstanceFoo::waitForEvent(void) {
    // call the stack function which put the device to sleep
    blefoo_WFE();
}
```






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

[`InitializationCompleteCallbackContext`]: https://docs.mbed.com/docs/ble-api/en/master/api/structBLE_1_1InitializationCompleteCallbackContext.html
