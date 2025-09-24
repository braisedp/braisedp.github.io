---
layout: post
title: "Vhal get与set流程"
date:   2025-9-5
tags: [Android, Automotive, Vhal]
comments: true
author: braisedp
toc : true
---

<!-- more -->

## get

### HIDL实现

#### 时序图

```mermaid
sequenceDiagram
participant CS as CarService
participant VHM as VehicleHalManager
participant VH as XXXVehicleHal
participant VPS as VehiclePropertyStore
participant VOP as VehiclePropValuePool

autonumber
CS ->>+ VHM : get
VHM->>VH: get
VH->>VPS: readValueOrNull
VPS->>VPS: getRecordIdLocked
VPS->>VPS: readValueOrNull
VPS->>VPS: getValueOrNullLocked
VPS-->>VH: std::unique_ptr<VehiclePropValue>
VH->>VOP: obtain
VOP-->>VH: VehiclePropValuePtr
VH-->>VHM: VehiclePropValuePtr
VHM-->>CS: StatusCode, VehiclePropValue
```

#### 具体流程

**step 1** 调用`get`方法，这一步会向下调用`VehicleHal`的`get`方法，并通过`_hidl_cb`返回`StatusCode`和`VehiclePropValue`
```cpp
Return<void> VehicleHalManager::get(const VehiclePropValue& requestedPropValue, get_cb _hidl_cb) {
    //根据配置进行一些检查
    ...
    StatusCode status;

    //调用VehicleHal的get方法
    auto value = mHal->get(requestedPropValue, &status);
    _hidl_cb(status, value.get() ? *value : kEmptyValue);

    return Void();
}
```

**step 2** 这一步会先调用`VehiclePropertyStore`的`readValueOrNull`方法，获取一个指向`VehiclePropertyValue`的唯一指针，然后调用`VehiclePropValuePool::obtain`方法进行对象复用
```cpp
VehicleHal::VehiclePropValuePtr XXXVehicleHal::get(
        const VehiclePropValue& requestedPropValue, StatusCode* outStatus) {
    auto propId = requestedPropValue.prop;
    ...

    auto& pool = *getValuePool();
    VehiclePropValuePtr v = nullptr;

    switch (propId) {
        ...
        default:
            ...
            //从VehiclePropertyStore中读取值
            auto internalPropValue = mPropStore->readValueOrNull(requestedPropValue);
            //通过VehicleObjectPool进行对象复用
            if (internalPropValue != nullptr) {
                v = getValuePool()->obtain(*internalPropValue);
            }

            *outStatus = v != nullptr ? StatusCode::OK : StatusCode::INVALID_ARG;
            break;
    }
    if (v.get()) {
        v->timestamp = elapsedRealtimeNano();
    }
    return v;
}
```

**step 3** 这一步首先会获取record id， record id由property id, area id 和 token三部分组成：
```cpp
std::unique_ptr<VehiclePropValue> VehiclePropertyStore::readValueOrNull(
        const VehiclePropValue& request) const {
    MuxGuard g(mLock);
    //首先找到request对应的record id
    RecordId recId = getRecordIdLocked(request);
    //然后根据record id获取值
    const VehiclePropValue* internalValue = getValueOrNullLocked(recId);
    return internalValue ? std::make_unique<VehiclePropValue>(*internalValue) : nullptr;
}
```

**step 4** 获取record id 时会向本地的configs中查询是否由property id对应的config，如果没有则返回空，否则设置token。
```cpp
VehiclePropertyStore::RecordId VehiclePropertyStore::getRecordIdLocked(
        const VehiclePropValue& valuePrototype) const {
    RecordId recId = {
        .prop = valuePrototype.prop,
        .area = isGlobalProp(valuePrototype.prop) ? 0 : valuePrototype.areaId,
        .token = 0
    };

    auto it = mConfigs.find(recId.prop);
    if (it == mConfigs.end()) return {};

    if (it->second.tokenFunction != nullptr) {
        recId.token = it->second.tokenFunction(valuePrototype);
    }
    return recId;
}
```

**step 5** 获取到record id后，会根据record id调用`getValueOrNullLocked`，获取属性值并返回。

```cpp
std::unique_ptr<VehiclePropValue> VehiclePropertyStore::readValueOrNull(
        int32_t prop, int32_t area, int64_t token) const {
    RecordId recId = {prop, isGlobalProp(prop) ? 0 : area, token };
    MuxGuard g(mLock);
    const VehiclePropValue* internalValue = getValueOrNullLocked(recId);
    return internalValue ? std::make_unique<VehiclePropValue>(*internalValue) : nullptr;
}
```

**step 6** `getValueOrNullLocked`方法会从`mProertyValues`中找到对应的`value`并返回。
```cpp
const VehiclePropValue* VehiclePropertyStore::getValueOrNullLocked(
        const VehiclePropertyStore::RecordId& recId) const  {
    auto it = mPropertyValues.find(recId);
    return it == mPropertyValues.end() ? nullptr : &it->second;
}
```

**step 7** 这一步通过`VehiclePropValuePool::obtain`方法从对象池中返回一个可用的对象。

```cpp
VehiclePropValuePool::RecyclableType VehiclePropValuePool::obtain(
        const VehiclePropValue& src) {
    ...
    VehiclePropertyType type = getPropType(src.prop);
    size_t vecSize = getVehicleRawValueVectorSize(src.value, type);;
    auto dest = obtain(type, vecSize);

    dest->prop = src.prop;
    dest->areaId = src.areaId;
    dest->status = src.status;
    dest->timestamp = src.timestamp;
    copyVehicleRawValue(&dest->value, src.value);

    return dest;
}
```

#### 对象复用
对于`VehiclePropValuePool::obtain`方法，其会通过`type`和`vecSize`从对象池中获取一个可复用的对象：

```cpp
VehiclePropValuePool::RecyclableType VehiclePropValuePool::obtain(
        VehiclePropertyType type, size_t vecSize) {
    return isDisposable(type, vecSize)
            ? obtainDisposable(type, vecSize)
            : obtainRecylable(type, vecSize);
}
```
，会根据`type`和`vecSize`判断是不是可以复用的：

```cpp
bool isDisposable(VehiclePropertyType type, size_t vecSize) const {
    return vecSize > mMaxRecyclableVectorSize || VehiclePropertyType::STRING == type ||
            VehiclePropertyType::MIXED == type;
}
```
，如果vecSize过大，或者其类型为String或者MIXED，则不可复用。

如果对象可以复用，那么这一步就会从`mValueTypePools`中获取一个对象池（没有则创建），然后从对象池中获取一个可以复用的对象：

```cpp
VehiclePropValuePool::RecyclableType VehiclePropValuePool::obtainRecylable(
        VehiclePropertyType type, size_t vecSize) {
    int32_t key = static_cast<int32_t>(type)
                | static_cast<int32_t>(vecSize);

    std::lock_guard<std::mutex> g(mLock);
    auto it = mValueTypePools.find(key);

    if (it == mValueTypePools.end()) {
        auto newPool(std::make_unique<InternalPool>(type, vecSize));
        it = mValueTypePools.emplace(key, std::move(newPool)).first;
    }
    return it->second->obtain();
}
```

如果对象不可以复用，则直接创建一个对象，该对象在被delete时直接销毁：
```cpp
VehiclePropValuePool::RecyclableType VehiclePropValuePool::obtainDisposable(
        VehiclePropertyType valueType, size_t vectorSize) const {
    return RecyclableType {
        createVehiclePropValue(valueType, vectorSize).release(),
        mDisposableDeleter
    };
}
```

查看`ObjectPool`的定义，在进行`obtain`时，其会从`mObjects`中获取一个`unique_ptr`，然后通过`wrap`方法进行包装，而`wrap`方法会给`unique_ptr`新增一个`Deleter`，在对象被销毁时，调用`recycle`方法，这个方法会将对象重新放回到`mObjects`中。
```cpp
template <typename T>
using recyclable_ptr = typename std::unique_ptr<T, Deleter<T>>;

template<typename T>
class ObjectPool {
public:
    ObjectPool() = default;
    virtual ~ObjectPool() = default;

    virtual recyclable_ptr<T> obtain() {
        std::lock_guard<std::mutex> g(mLock);
        ...
        auto o = wrap(mObjects.front().release());
        mObjects.pop_front();

        return o;
    }

    ObjectPool& operator =(const ObjectPool &) = delete;
    ObjectPool(const ObjectPool &) = delete;

protected:
    ...
    
    virtual void recycle(T* o) {
        ...
        mObjects.push_back(std::unique_ptr<T> { o } );
    }

private:
    const Deleter<T>& getDeleter() {
        if (!mDeleter.get()) {
            Deleter<T> *d = new Deleter<T>(std::bind(
                &ObjectPool::recycle, this, std::placeholders::_1));
            mDeleter.reset(d);
        }
        return *mDeleter.get();
    }

    recyclable_ptr<T> wrap(T* raw) {
        return recyclable_ptr<T> { raw, getDeleter() };
    }

private:
    mutable std::mutex mLock;
    std::deque<std::unique_ptr<T>> mObjects;
    std::unique_ptr<Deleter<T>> mDeleter;
};
```
#### VehiclePropertyStore中始终存储最新值

在`XXXVehicleHal`中存在如下方法，用于给下层在属性值改变时进行调用，可以看到，当下层通知属性值改变时，`XXXVehicleHal`就会将新值写入到`VehiclePropertyStore`中，保证`get`调用获取的始终是最新值。

```cpp
void XXXVehicleHal::onPropertyValue(const VehiclePropValue &value, bool updateStatus)
{
    VehiclePropValuePtr updatedPropValue = getValuePool()->obtain(value);

    if (mPropStore->writeValue(*updatedPropValue, updateStatus))
    {
        ...
    }
}
```

### AIDL实现
#### 时序图
```mermaid
sequenceDiagram
participant CS as CarService
participant VH as XXXVehicleHal
participant VHw as XXXVehicleHalware
participant H as PendingRequestHandler
participant VPS as VehiclePropertyStore
CS->>+VH : getValues
par 
activate VH
VH->>VH: getOrCreateClient
deactivate VH
VH->>+VHw: getValues
deactivate VH
VHw->>+H: addRequest
deactivate VHw
deactivate H
and
activate H
H->>H:handleRequestOnce
H->>+VHw: handleGetValueRequest

activate VHw
VHw->>VHw:getValue
VHw->>+VPS: readValue
activate VPS
VPS->>VPS: readValueLocked
VPS-->>VHw: propertyValue
deactivate VPS
VHw-->>H: propertyValue
deactivate VHw
deactivate VHw
end
H-->>CS: callback
deactivate H
```

#### 详细步骤

**step 1** 在AIDL实现中，Vhal允许多个get请求合并并通过异步的方式返回，于是在第一步，会根据callback创建对应的client用于记录get属性的callback对象。然后将get请求传递给`VehicleHardware`：
```cpp
ScopedAStatus XXXVehicleHal::getValues(const CallbackType& callback,
                                            const GetValueRequests& requests) {
                                    
    ...
    std::vector<GetValueRequest> hardwareRequests;
    ...

    std::shared_ptr<GetValuesClient> client;
    {
        ...

        client = getOrCreateClient(&mGetValuesClients, callback, mPendingRequestPool);
    }

    ...

    if (StatusCode status =
                mVehicleHardware->getValues(client->getResultCallback(), hardwareRequests);
        status != StatusCode::OK) {
            ...
    }
    return ScopedAStatus::ok();
}
```

**step 2** 收到get请求后，`VehicleHardware`需要记录每个get请求对应的callback用于当请求收到响应时进行回调：
```cpp
StatusCode XXXVehicleHardware::getValues(std::shared_ptr<const GetValuesCallback> callback,
                                        const std::vector<GetValueRequest>& requests) const {
    for (auto& request : requests) {
        ...
        mPendingGetValueRequests.addRequest(request, callback);
    }

    return StatusCode::OK;
}
```

**step 3** `PendingRequestHandler`的`addRequest`方法则是将request和callback存入一个map中：

```cpp
template <class CallbackType, class RequestType>
void XXXVehicleHardware::PendingRequestHandler<CallbackType, RequestType>::addRequest(
        RequestType request, std::shared_ptr<const CallbackType> callback) {
    mRequests.push({
            request,
            callback,
    });
}
```

**step 4** 在`PendingRequestHandler`创建时，会通过一个thread执行`handleRequestOnce`方法用于异步处理get请求。方法首先会从map不断取出request，执行`VehicleHardware::handleGetValueRequest`方法，然后调用callback用于返回get响应：

```cpp
template <>
void XXXeVehicleHardware::PendingRequestHandler<FakeVehicleHardware::GetValuesCallback,
                                                GetValueRequest>::handleRequestsOnce() {
    std::unordered_map<std::shared_ptr<const GetValuesCallback>, std::vector<GetValueResult>>
            callbackToResults;
    for (const auto& rwc : mRequests.flush()) {
        ...
        auto result = mHardware->handleGetValueRequest(rwc.request);
        ...
    }
    for (const auto& [callback, results] : callbackToResults) {
        ...
        (*callback)(std::move(results));
        ...
    }
}
```

**step 5** 实际执行`getValue`方法，获取最新值并返回：
```cpp
GetValueResult XXXVehicleHardware::handleGetValueRequest(const GetValueRequest& request) {
    GetValueResult getValueResult;
    getValueResult.requestId = request.requestId;

    auto result = getValue(request.prop);
    ...
    return getValueResult;
}
```

**step 6** 方法从`VehiclePropertyStore`中读取属性值并返回：
```cpp
XXXVehicleHardware::ValueResultType XXXVehicleHardware::getValue(
        const VehiclePropValue& value) const {
    ...

    auto readResult = mServerSidePropStore->readValue(value);
    ...

    return std::move(readResult);
}
```

**step 7** `readValue`方法根据属性的找到对应的record 和recordId，并执行`readValueLocked`方法：
```cpp
VehiclePropertyStore::ValueResultType VehiclePropertyStore::readValue(
        const VehiclePropValue& propValue) const {
    std::scoped_lock<std::mutex> g(mLock);

    int32_t propId = propValue.prop;
    const VehiclePropertyStore::Record* record = getRecordLocked(propId);
    ...

    VehiclePropertyStore::RecordId recId = getRecordIdLocked(propValue, *record);
    return readValueLocked(recId, *record);
}
```

在AIDL实现的`VehiclePropertyStore`中，属性值与属性的存储方式如下：

![VehiclePropertyStore AIDL](../images/2025-9-5-vhal_get_set/AidlVehiclePropertyStore.svg)

**step 8** 这一步会从record中根据recordId找到对应的value，并通过`VehiclePropValuePool`进行对象复用：

```cpp
VhalResult<VehiclePropValuePool::RecyclableType> VehiclePropertyStore::readValueLocked(
        const RecordId& recId, const Record& record) const REQUIRES(mLock) {
    if (auto it = record.values.find(recId); it != record.values.end()) {
        return mValuePool->obtain(*(it->second));
    }
    ...
}
```

## set

### HIDL实现
#### 时序图
```mermaid
sequenceDiagram
participant CS as CarService
participant VHM as VehicleHalManager
participant VH as XXXVehicleHal
participant Hw as Connector/others
CS->>VHM: set
VHM->>VHM: handlePropertySetEvent
VHM->>VH: set
VH->>Hw: set value
```

#### 详细步骤

**step 1** 第一步首先会调用自己的`handlePropertySetEvent`方法，然后将set请求传递到下层：
```cpp
Return<StatusCode> VehicleHalManager::set(const VehiclePropValue &value) {
    auto prop = value.prop;
    ...

    handlePropertySetEvent(value);

    auto status = mHal->set(value);

    return Return<StatusCode>(status);
}
```


**step 2** 这一步主要会回调订阅了属性的`EVENTS_FROM_ANDROID`事件的callback：
```cpp
void VehicleHalManager::handlePropertySetEvent(const VehiclePropValue& value) {
    auto clients =
        mSubscriptionManager.getSubscribedClients(value.prop, SubscribeFlags::EVENTS_FROM_ANDROID);
    for (const auto& client : clients) {
        client->getCallback()->onPropertySet(value);
    }
}
```

**step 3** 这一步会首先从`VehiclePropertyStore`中读取属性的当前值，然后进行一些下层调用：
```cpp
StatusCode XXXVehicleHal::set(const VehiclePropValue &propValue)
{
    constexpr bool updateStatus = false;
    ...
    auto currentPropValue = mPropStore->readValueOrNull(propValue);

    ...

    /**
    * 这里需要实现与硬件的交互
    */

    return StatusCode::OK;
}
```


### AIDL实现
#### 时序图

```mermaid
sequenceDiagram
participant CS as CarService
participant VH as XXXVehicleHal
participant VHw as XXXVehicleHalware
participant H as PendingRequestHandler
participant VPVP as VehiclePropValuePool
participant VPS as VehiclePropertyStore
CS->>+VH : setValues
par 
activate VH
VH->>VH: getOrCreateClient
deactivate VH
VH->>+VHw: setValues
deactivate VH
VHw->>+H: addRequest
deactivate VHw
deactivate H
and
activate H
H->>H:handleRequestOnce
H->>+VHw: handleSetValueRequest

activate VHw
VHw->>VHw:setValue
VHw->>+VPVP: obtain
VPVP-->>VHw: recyclable property value
deactivate VPVP
VHw->>+VPS: writeValue
deactivate VPS
VHw-->>H: {}
deactivate VHw
deactivate VHw
end
H-->>CS: callback
deactivate H
```

#### 详细步骤

**step 1** set请求允许多个请求合并，并异步执行，然后通过callback返回set响应。与get相同，这一步也会首先创建一个对应的client用于记录一个callback上的不同get请求。然后，向下层传递set请求：

```cpp
ScopedAStatus XXXVehicleHal::setValues(const CallbackType& callback,
                                            const SetValueRequests& requests) {
    ...
    std::vector<SetValueRequest> hardwareRequests;
    ...
    std::unordered_set<int64_t> hardwareRequestIds;
    for (const auto& request : hardwareRequests) {
        hardwareRequestIds.insert(request.requestId);
    }

    std::shared_ptr<SetValuesClient> client;
    {
        ...
        client = getOrCreateClient(&mSetValuesClients, callback, mPendingRequestPool);
    }

    ...

    if (StatusCode status =
                mVehicleHardware->setValues(client->getResultCallback(), hardwareRequests);
        status != StatusCode::OK) {
            ...
    }

    return ScopedAStatus::ok();
}
```

**step 2** 与get类似，这一步会将set请求放到一个等待队列中：
```cpp
StatusCode XXXVehicleHardware::setValues(std::shared_ptr<const SetValuesCallback> callback,
                                        const std::vector<SetValueRequest>& requests) {
    for (auto& request : requests) {
        mPendingSetValueRequests.addRequest(request, callback);
    }

    return StatusCode::OK;
}
```

**step 3** 与get类似，这一步会从等待队列中取出所有的set请求，并执行`handleSetValueRequest`方法，然后回调callback：

```cpp
void XXXVehicleHardware::PendingRequestHandler<FakeVehicleHardware::SetValuesCallback,
                                                SetValueRequest>::handleRequestsOnce() {
    std::unordered_map<std::shared_ptr<const SetValuesCallback>, std::vector<SetValueResult>>
            callbackToResults;
    for (const auto& rwc : mRequests.flush()) {
        ...
        auto result = mHardware->handleSetValueRequest(rwc.request);
        ...

    }
    for (const auto& [callback, results] : callbackToResults) {
        ...
        (*callback)(std::move(results));
        ...
    }
}
```

**step 4** 这一步实际调用了`setValue`方法：

```cpp
SetValueResult XXXVehicleHardware::handleSetValueRequest(const SetValueRequest& request) {
    SetValueResult setValueResult;
    setValueResult.requestId = request.requestId;

    if (auto result = setValue(request.value); !result.ok()) {
        ...
    } else {
        ...
    }

    return setValueResult;
}
```


**step 5** 可以看到`setValue`方法首先会调用`VehiclePropValuePool`的`obtain`方法返回一个复用的对象，然后写入到`VehiclePropertyStore`中，保证了get可以始终获取最新值。最后执行一些下层的操作。
```cpp
VhalResult<void> XXXVehicleHardware::setValue(const VehiclePropValue& value) {
    ...

    auto updatedValue = mValuePool->obtain(value);
    int64_t timestamp = elapsedRealtimeNano();
    updatedValue->timestamp = timestamp;

    auto writeResult = mServerSidePropStore->writeValue(std::move(updatedValue));
    ...
    //执行一些与硬件交互

    return {};
}
```