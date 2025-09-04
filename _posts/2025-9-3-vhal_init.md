---
layout: post
title: "Vhal"
date:   2025-9-3
tags: [Android, Automative, vhal]
comments: true
author: braisedp
toc : true
---

# HIDL
## Vhal启动流程

![hidl init](../images/2025-9-3-vhal_init/init.svg)

首先查看`VehicleService`的`main`方法：
```cpp
int main(int /* argc */, char* /* argv */ []) {
    //加载VehiclePropertyStore
    auto store = std::make_unique<VehiclePropertyStore>();
    //加载VehicleConnector
    auto connector = std::make_unique<XXXVehicleConnector>();
    //加载VehicleHal
    auto hal = std::make_unique<XXXVehicleHal>(store.get(), connector.get());
    //加载VehicalHalManager
    auto service = std::make_unique<VehicleHalManager>(hal.get());
    connector->setValuePool(hal->getValuePool());
    android::hardware::configureRpcThreadpool(4, true /* callerWillJoin */);
    ...
    android::status_t status = service->registerAsService();
    ...
}
```
，主要创建了四个类型的对象，分别是`VehiclePropertyStore`、`VehicleConnector`、`VehicleHal`和`vehicleHalManager`，首先
`VehicleService`会创建未初始化的`VehiclePropertyStore`和`VehicleConnector`，然后通过这两个对象创建`VehicleHal`。


### 创建VehicleHal

查看`VehicleHal`的构造函数：
```cpp
XXXVehicleHal::XXXVehicleHal(VehiclePropertyStore *propStore, XXXVehicleConnector *connector)
    : mPropStore(propStore),
    mConnector(connector)
{
    initStaticConfig();
    for (size_t i = 0; i < arraysize(kVehicleProperties); i++)
    {
        mPropStore->registerProperty(kVehicleProperties[i].config);
    }
    mVehicleClient->registerPropertyValueCallback(std::bind(&EmulatedVehicleHal::onPropertyValue,
                                                            this, std::placeholders::_1,
                                                            std::placeholders::_2));
}

```
，主要执行了`initStaicConfig`方法，并向`VehiclePropertyStore`中写入`config`

#### 默认VehiclePropConfig的加载

查看`initStaticConfig`的代码：
```cpp
void EmulatedVehicleHal::initStaticConfig() {
    for (auto&& it = std::begin(kVehicleProperties); it != std::end(kVehicleProperties); ++it) {
        const auto& cfg = it->config;
        VehiclePropertyStore::TokenFunction tokenFunction = nullptr;

        switch (cfg.prop) {
            case OBD_FREEZE_FRAME: {
                tokenFunction = [](const VehiclePropValue& propValue) {
                    return propValue.timestamp;
                };
                break;
            }
            default:
                break;
        }

        mPropStore->registerProperty(cfg, tokenFunction);
    }
}
```
，该方法从`kVehicleProperties`中读取property的配置信息，向`VehiclePropertyStore`中注册配置信息。同样，在`VehicleHal`的构造函数中，也执行了操作：
```cpp
for (size_t i = 0; i < arraysize(kVehicleProperties); i++)
{
    mPropStore->registerProperty(kVehicleProperties[i].config);
}
```
在`DefaultConfig.h`中查看`kVehicleProperties`的定义：
```cpp
const ConfigDeclaration kVehicleProperties[]{
    {.config =
            {
                    .prop = toInt(VehicleProperty::INFO_FUEL_CAPACITY),
                    .access = VehiclePropertyAccess::READ,
                    .changeMode = VehiclePropertyChangeMode::STATIC,
            },
    .initialValue = {.floatValues = {15000.0f}}},
    ...
};
```
，是`ConfigDeclaration`类型的数组，`ConfigDeclaration`的定义在`DefaultConfig.h`中：
```cpp
struct ConfigDeclaration {
    VehiclePropConfig config;
    VehiclePropValue::RawValue initialValue;
    std::map<int32_t, VehiclePropValue::RawValue> initialAreaValues;
};
```
，包含了`VehiclePropConfig`、property的初始物理值和在每个area的对应初始值。

`VehiclePropConfig`的定义在`hardware/interfaces/automotive/vehicle/2.0/types.hal`中：
```cpp
struct VehiclePropConfig {
    int32_t prop;
    VehiclePropertyAccess access;
    VehiclePropertyChangeMode changeMode;
    vec<VehicleAreaConfig> areaConfigs;
    vec<int32_t> configArray;
    string configString;
    float minSampleRate;
    float maxSampleRate;
};
```
，其中一些字段解释如下：
<table>
<tr>
<td rowspan="4">access</td><td>NONE</td><td>无</td>
</tr>
<tr>
<td>READ</td><td>只读</td>
</tr>
<tr>
<td>WRITE</td><td>只写</td>
</tr>
<tr>
<td>READ_WRITE</td><td>可读写</td>
</tr>
<tr>
<td rowspan="3">changeMode</td><td>STATIC</td><td>不变化，不支持subscribe</td>
</tr>
<tr>
<td>ON_CHANGE</td><td>get返回当前值，值改变时该类型属性必须触发值改变事件</td>
</tr>
<tr>
<td>CONTINUOUS</td><td>subscribe 必须采用一定的sample rate</td>
</tr>

</table>

查看`VehiclePropertyStore`的`registerProperty`方法：
```cpp
void VehiclePropertyStore::registerProperty(const VehiclePropConfig& config,
                                            VehiclePropertyStore::TokenFunction tokenFunc) {
    MuxGuard g(mLock);
    mConfigs.insert({ config.prop, RecordConfig { config, tokenFunc } });
}
  
```

，向`mConfigs`中插入了一个`prop`和`RecordConfig`的键值对，其中`mConfigs`是一个property id到`RecordConfig`的map：

```cpp
struct RecordConfig {
    VehiclePropConfig propConfig;
    TokenFunction tokenFunction;
};

std::unordered_map<int32_t /* VehicleProperty */, RecordConfig> mConfigs;
```

### 启动VehicleHalManager

在完成`VehicleHal`的创建后，会进行`VehicleHalManager`的启动，这一步会创建`VehicleHalManager`，并对之前创建的一些对象进行初始化的操作。


首先查看`VehicleHalManager`的构造函数：
```cpp
VehicleHalManager(VehicleHal* vehicleHal)
    : mHal(vehicleHal),
    //首先创建`SubscriptionManager`对象
    mSubscriptionManager(std::bind(&VehicleHalManager::onAllClientsUnsubscribed,
                                    this, std::placeholders::_1)) {
    //然后执行初始化方法
    init();
}
```
，这一步会先创建`SubscriptionManager`的对象用于管理`subscribe/unsubscribe`，然后执行初始化的操作。

#### 创建SubscriptionManager

这一步主要是将`SubscriptionManager`的`mOnPropertyUnsubscribed`方法设置为`VehicleHalManager`的`onAllClientsUnsubscribed`方法，并设置`mCallbackDeathRecipient`的` `为`onCallbackDead`方法：
```cpp
SubscriptionManager(const OnPropertyUnsubscribed& onPropertyUnsubscribed)
        : mOnPropertyUnsubscribed(onPropertyUnsubscribed),
            mCallbackDeathRecipient(new DeathRecipient(
                std::bind(&SubscriptionManager::onCallbackDead, this, std::placeholders::_1)))
{}
```
`onAllClientUnsubscribed`方法主要执行了`VehicleHal`的`unsubscribe`方法：
```cpp
void VehicleHalManager::onAllClientsUnsubscribed(int32_t propertyId) {
    mHal->unsubscribe(propertyId);
}
```

方法`onCallBackDead`在订阅属性的远程对象的binder死亡时，取消该远程对象对属性的订阅：
```cpp
void SubscriptionManager::onCallbackDead(uint64_t cookie) {
    ...
    ClientId clientId = cookie;

    std::vector<int32_t> props;
    {
        MuxGuard g(mLock);
        //获取id对应的client对象
        const auto& it = mClients.find(clientId);
        if (it == mClients.end()) {
            return; 
        }
        const auto& halClient = it->second;
        //获取订阅的属性
        props = halClient->getSubscribedProperties();
    }
    //取消订阅
    for (int32_t propId : props) {
        unsubscribe(clientId, propId);
    }
}
```

#### 初始化VehicleHalManager

在前面的步骤中，完成了所有对象的创建，在这一步，需要完成对创建的对象的初始化操作。

查看`VehicleHalManager::init`方法：

```cpp
void VehicleHalManager::init() {
    ...

    //初始化VehicleHal
    mHal->init(&mValueObjectPool,
                std::bind(&VehicleHalManager::onHalEvent, this, _1),
                std::bind(&VehicleHalManager::onHalPropertySetError, this,
                        _1, _2, _3));

    ...
}
```
执行`VehicleHal`的`init`方法。

#### 初始化VehicleHal

`VehicleHal`的`init`方法在`VehicleHal.h`中：
```cpp
void init(
    VehiclePropValuePool* valueObjectPool,
    const HalEventFunction& onHalEvent,
    const HalErrorFunction& onHalError) {
    mValuePool = valueObjectPool;
    mOnHalEvent = onHalEvent;
    mOnHalPropertySetError = onHalError;

    onCreate();
}
```
，在这一步里，设置`VehicleHal`的`mValuePool`、`mOnHalEvent`和`mOnHalPropertySetError`为`VehicleHalManager`的`mValueObjectPool`、`onHalEvent`和`onHalPorpertySetError`，第一个对象用来进行property对象的复用，第二个和第三个则是设置了两个回调函数。 进行了上述设置后，执行`onCreate`方法：

```cpp
void EmulatedVehicleHal::onCreate() {
    static constexpr bool shouldUpdateStatus = true;

    for (auto& it : kVehicleProperties) {
        VehiclePropConfig cfg = it.config;
        int32_t numAreas = cfg.areaConfigs.size();
        ...
        for (int i = 0; i < numAreas; i++) {
            ...
            VehiclePropValue prop = {
                    .areaId = curArea,
                    .prop = cfg.prop, 
            };
            ...
            mPropStore->writeValue(prop, shouldUpdateStatus);
        }
    }
    ...
}
```
，在`onCreate`方法中，会从`kVehicleProperties`中读取所有的config，并根据其中的area执行`VehiclePropertyStore::writeValue`方法，这一步会将在`kVehicleProperties`中设定的属性值写入到`VehiclePropertyStore`中。

# AIDL

![aidl init](../images/2025-9-3-vhal_init/init2.svg)

## Vhal启动流程

首先查看`VehicleService`中定义的`main`方法：
```cpp
int main(int /* argc */, char* /* argv */[]) {
    ...
    if (!ABinderProcess_setThreadPoolMaxThreadCount(4)) {
        ...
    }
    ABinderProcess_startThreadPool();

    //创建VehicleHardWare对象
    std::unique_ptr<XXXVehicleHardware> hardware = std::make_unique<XXXVehicleHardware>();
    //创建VehicleHal对象
    std::shared_ptr<XXXVehicleHal> vhal =
            ::ndk::SharedRefBase::make<XXXVehicleHal>(std::move(hardware));

    ...
    binder_exception_t err = AServiceManager_addService(
            vhal->asBinder().get(), "android.hardware.automotive.vehicle.IVehicle/default");
    ...

    ABinderProcess_joinThreadPool();
    ...

    return 0;
}
```

### 创建并初始化VehicleHardware

```cpp
XXXVehicleHardware::XXXVehicleHardware()
    : XXXVehicleHardware(DEFAULT_CONFIG_DIR, OVERRIDE_CONFIG_DIR, false) {}

XXXVehicleHardware::XXXVehicleHardware(std::string defaultConfigDir,
                                        std::string overrideConfigDir, bool forceOverride)
    : mValuePool(std::make_unique<VehiclePropValuePool>()),
    mServerSidePropStore(new VehiclePropertyStore(mValuePool)),
    ...
    mRecurrentTimer(new RecurrentTimer()),
    ...
    mDefaultConfigDir(defaultConfigDir),
    mOverrideConfigDir(overrideConfigDir),
    mForceOverride(forceOverride) {
    init();
}
```
，这一步首先创建了一个`VehiclePropValuePool`，然后创建一个`VehiclePropertyStore`，然后，设置了`mDefaultConfigDir`和`mOverrideConfigDir`，后续执行了`init`方法：

```cpp
void XXXVehicleHardware::init() {
    //首先加载配置信息
    for (auto& [_, configDeclaration] : loadConfigDeclarations()) {
        VehiclePropConfig cfg = configDeclaration.config;
        VehiclePropertyStore::TokenFunction tokenFunction = nullptr;

        ...
        //将配置信息注册到`VehiclePropertyStore`中
        mServerSidePropStore->registerProperty(cfg, tokenFunction);

        ...

        //将config中的初始值写入
        storePropInitialValue(configDeclaration);
    }

    ...

    //设置VehiclePropertyStore在property值改变时的回调函数为XXXVehicleHardware::onValueChangeCallback
    mServerSidePropStore->setOnValueChangeCallback(
            [this](const VehiclePropValue& value) { return onValueChangeCallback(value); });
}
```

#### 加载默认配置并设置VehiclePropertyStore
查看`loadConfigDeclarations`方法：
```cpp
std::unordered_map<int32_t, ConfigDeclaration> XXXVehicleHardware::loadConfigDeclarations() {
    std::unordered_map<int32_t, ConfigDeclaration> configsByPropId;
    loadPropConfigsFromDir(mDefaultConfigDir, &configsByPropId);
    if (mForceOverride ||
        android::base::GetBoolProperty(OVERRIDE_PROPERTY, /*default_value=*/false)) {
        loadPropConfigsFromDir(mOverrideConfigDir, &configsByPropId);
    }
    return configsByPropId;
}
```
，主要调用了`loadPropConfigsFromDir`方法：
```cpp
void XXXVehicleHardware::loadPropConfigsFromDir(
        const std::string& dirPath,
        std::unordered_map<int32_t, ConfigDeclaration>* configsByPropId) {
    ...
    if (auto dir = opendir(dirPath.c_str()); dir != NULL) {
        std::regex regJson(".*[.]json", std::regex::icase);
        while (auto f = readdir(dir)) {
            ...
            auto result = mLoader.loadPropConfig(filePath);
            ...
            for (auto& [propId, configDeclaration] : result.value()) {
                (*configsByPropId)[propId] = std::move(configDeclaration);
            }
        }
        closedir(dir);
    }
}
```
，这一步主要通过`mLoader`从json文件中加载配置信息，`mLoader`是`JsonConfigLoader`类的实例，查看其`loadPropConfig`方法：
```cpp
android::base::Result<std::unordered_map<int32_t, ConfigDeclaration>>
JsonConfigLoader::loadPropConfig(std::istream& is) {
    return mParser->parseJsonConfig(is);
}

android::base::Result<std::unordered_map<int32_t, ConfigDeclaration>>
JsonConfigLoader::loadPropConfig(const std::string& configPath) {
    std::ifstream ifs(configPath.c_str());
    ...
    return loadPropConfig(ifs);
}
```
，调用了`parseJsonConfig`方法从json文件中解析配置信息，json文件的格式如下：
```json
{
    "apiVersion": 1,
    "properties": [
        {
            // property id
            "property": "VehicleProperty::INFO_FUEL_CAPACITY",
            // (optional, number/string) property 访问模式
            "access": "VehiclePropertyAccess::READ",
            // (optional, number/string) propert change mode
            "changeMode": "VehiclePropertyChangeMode::STATIC",
            // (optional, string) config string.
            "configString": "blahblah",
            // (optional, array of number/string) config array.
            "configArray": [1, 2, "Constants::HVAC_ALL"],
            // (optional, object) property 的默认值
            "defaultValue": {
                // (optional, array of int number/string) Int values.
                "int32Values": [1, 2, "Constants::HVAC_ALL"],
                // (optional, array of int number/string) Long values.
                "int64Values": [1, 2],
                // (optional, array of float number/string) Float values.
                "floatValues": [1.1, 2.2],
                // (optional, string) String value.
                "stringValue": "test"
            },
            "minSampleRate": 1,
            "maxSampleRate": 10,
            // (optional, array of objects) 不同area的配置
            "areas:" [
                {
                    "areaId": "Constants::DOOR_1_LEFT",
                    // (optional number/string)
                    "minInt32Value": 1,
                    // (optional number/string)
                    "maxInt32Value": 10,
                    // (optional number/string)
                    "minInt64Value": 1,
                    // (optional number/string)
                    "maxInt64Value": 10,
                    // (optional number/string)
                    "minFloatValue": 1,
                    // (optional number/string)
                    "maxFloatValue": 10,
                    // (optional object)
                    "defaultValue": {
                        "int32Values": [1, 2, "Constants::HVAC_ALL"],
                        "int64Values": [1, 2],
                        "floatValues": [1.1, 2.2],
                        "stringValue": "test"
                    }
                }
            ]
        }
     ]
}

```
在执行完`loadConfigDeclarations`后，调用`VehiclePropertyStore::registerProperty`方法，该步与HIDL相同。然后，执行`storePropInitialValue`方法：
```cpp
void FakeVehicleHardware::storePropInitialValue(const ConfigDeclaration& config) {
    const VehiclePropConfig& vehiclePropConfig = config.config;
    int propId = vehiclePropConfig.prop;
    bool globalProp = isGlobalProp(propId);
    size_t numAreas = globalProp ? 1 : vehiclePropConfig.areaConfigs.size();

    for (size_t i = 0; i < numAreas; i++) {
        int32_t curArea = globalProp ? 0 : vehiclePropConfig.areaConfigs[i].areaId;
        VehiclePropValue prop = {
                .areaId = curArea,
                .prop = propId,
                .timestamp = elapsedRealtimeNano(),
        };

        ...

        auto result =
                mServerSidePropStore->writeValue(mValuePool->obtain(prop), /*updateStatus=*/true);
        ...
    }
}
```
，这一步也与HIDL实现类似，向`VehicleProeprtyStore`中写入了config中配置的初始值。

#### 创建VehicleHal

```cpp
XXXVehicleHal::XXXVehicleHal(std::unique_ptr<IVehicleHardware> vehicleHardware)
    : mVehicleHardware(std::move(vehicleHardware)),
    mPendingRequestPool(std::make_shared<PendingRequestPool>(TIMEOUT_IN_NANO)) {
    ...
    IVehicleHardware* vehicleHardwarePtr = mVehicleHardware.get();

    //创建SubscriptionManager
    mSubscriptionManager = std::make_shared<SubscriptionManager>(vehicleHardwarePtr);

    //绑定回调函数
    std::weak_ptr<SubscriptionManager> subscriptionManagerCopy = mSubscriptionManager;
    mVehicleHardware->registerOnPropertyChangeEvent(
            std::make_unique<IVehicleHardware::PropertyChangeCallback>(
                    [subscriptionManagerCopy](std::vector<VehiclePropValue> updatedValues) {
                        onPropertyChangeEvent(subscriptionManagerCopy, updatedValues);
                    }));
    mVehicleHardware->registerOnPropertySetErrorEvent(
            std::make_unique<IVehicleHardware::PropertySetErrorCallback>(
                    [subscriptionManagerCopy](std::vector<SetValueErrorEvent> errorEvents) {
                        onPropertySetErrorEvent(subscriptionManagerCopy, errorEvents);
                    }));

    // 注册心跳监测事件
    mRecurrentAction = std::make_shared<std::function<void()>>(
            [vehicleHardwarePtr, subscriptionManagerCopy]() {
                checkHealth(vehicleHardwarePtr, subscriptionManagerCopy);
            });
    mRecurrentTimer.registerTimerCallback(HEART_BEAT_INTERVAL_IN_NANO, mRecurrentAction);

    ...
}
```
，在该步中，首先创建了`SubscriptionManager`，然后将`onPropertyChangeEvent`和`onPropertySetErrorEvent`方法注册到`VehicleHardware`上。 与HIDL实现不同，会设置一个`VehicleHardware`的健康监测。

