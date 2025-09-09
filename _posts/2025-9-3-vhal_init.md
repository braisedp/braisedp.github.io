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

    ...
}
```
，在该步中，首先创建了`SubscriptionManager`，然后将`onPropertyChangeEvent`和`onPropertySetErrorEvent`方法注册到`VehicleHardware`上。

## 属性配置的加载

在AIDL实现中，将属性的初始配置加载从读取`DefaultConfig.h`头文件转移到了读取`XXXpackages.json`文件。 

### JsonConfigLoader加载json文件

查看`JsonConfigParser`的`parseJsonConfig`方法：
```cpp
Result<std::unordered_map<int32_t, ConfigDeclaration>> JsonConfigParser::parseJsonConfig(
        std::istream& is) {
    ...
    std::unordered_map<int32_t, ConfigDeclaration> configsByPropId;
    ...
    Json::Value properties = root["properties"];
    ...
    for (unsigned int i = 0; i < properties.size(); i++) {
        if (auto maybeConfig = parseEachProperty(properties[i], &errors); maybeConfig.has_value()) {
            configsByPropId[maybeConfig.value().config.prop] = std::move(maybeConfig.value());
        }
    }
    ...
    return configsByPropId;
}
```
，查看`parseEachProperty`方法：
```cpp
std::optional<ConfigDeclaration> JsonConfigParser::parseEachProperty(
        const Json::Value& propJsonValue, std::vector<std::string>* errors) {
    ...
    int32_t propId;

    if (!tryParseJsonValueToVariable(propJsonValue, "property", /*optional=*/false, &propId,
                                    errors)) {
        return std::nullopt;
    }

    configDecl.config.prop = propId;
    std::string propStr = propJsonValue["property"].toStyledString();

    //解析access
    parseAccessChangeMode(propJsonValue, "access", propId, propStr, AccessForVehicleProperty,
                        &configDecl.config.access, errors);

    //解析changeMode
    parseAccessChangeMode(propJsonValue, "changeMode", propId, propStr,
                        ChangeModeForVehicleProperty, &configDecl.config.changeMode, errors);
    ...


    tryParseJsonValueToVariable(propJsonValue, "configString", /*optional=*/true,
                                &configDecl.config.configString, errors);

    tryParseJsonArrayToVariable(propJsonValue, "configArray", /*optional=*/true,
                                &configDecl.config.configArray, errors);

    //解析默认值
    parsePropValues(propJsonValue, "defaultValue", &configDecl.initialValue, errors);

    tryParseJsonValueToVariable(propJsonValue, "minSampleRate", /*optional=*/true,
                                &configDecl.config.minSampleRate, errors);

    tryParseJsonValueToVariable(propJsonValue, "maxSampleRate", /*optional=*/true,
                                &configDecl.config.maxSampleRate, errors);

    parseAreas(propJsonValue, "areas", &configDecl, errors);
    ...
    return configDecl;
}
```
，这一步会按照json文件中的每个property的配置按照顺序解析json键值对。重点关注`parseAccessChangeMode`方法和`parsePropValues`函数。

#### 解析 access / change mode

首先查看`parseAccessChangeMode`方法：
```cpp
void JsonConfigParser::parseAccessChangeMode(
        const Json::Value& parentJsonNode, const std::string& fieldName, int propId,
        const std::string& propStr, const std::unordered_map<VehicleProperty, T>& defaultMap,
        T* outPtr, std::vector<std::string>* errors) {
    ...
    if (parentJsonNode.isMember(fieldName)) {
        //从Json文件中加载access/change mode
        auto result = mValueParser.parseValue<int32_t>(fieldName, parentJsonNode[fieldName]);
        ...
        *outPtr = static_cast<T>(result.value());
        return;
    }
    //从defaultMap中加载access/change mode
    auto it = defaultMap.find(static_cast<VehicleProperty>(propId));
    ...
    *outPtr = it->second;
    return;
}
```
，这一步会从Json文件中加载access/change mode，如果Json文件中没有定义access/change mode的话，则会从传入的defaultMap中加载access/ change mode。 观察`AccessForVehicleProperty`或`ChangeModeForVehicleProperty`所在的文件，在文件中可以看到下面一段注释：

```cpp
// AccessForVehicleProperty.h

/**
* DO NOT EDIT MANUALLY!!!
*
* Generated by tools/generate_annotation_enums.py.
*/
```
，查看`/hardware/interfaces/automotive/vehicle/tools/generate_annotation_enums.py`：
```py
...
RE_COMMENT_BEGIN = re.compile("\s*\/\*\*?")
RE_COMMENT_END = re.compile("\s*\*\/")
RE_CHANGE_MODE = re.compile("\s*\* @change_mode (\S+)\s*")
RE_ACCESS = re.compile("\s*\* @access (\S+)\s*")
RE_VALUE = re.compile("\s*(\w+)\s*=(.*)")
...
class Converter:

    def __init__(self, name, annotation_re):
        self.name = name
        self.annotation_re = annotation_re

    def convert(self, input, output, header, footer, cpp):
        processing = False
        in_comment = False
        content = LICENSE + header
        annotation = None
        id = 0
        with open(input, 'r') as f:
            for line in f.readlines():
                ...
                # 注释开始 /**
                if RE_COMMENT_BEGIN.match(line):
                    in_comment = True
                # 注释结束 */
                if RE_COMMENT_END.match(line):
                    in_comment = False
                if in_comment:
                    # 在注释中匹配 @change_mode XXX 或 @access XXX
                    match = self.annotation_re.match(line)
                    # 解析其第二个字段 XXX
                    if match:
                        annotation = match.group(1)
                else:
                    # 注释结束，匹配属性名
                    match = RE_VALUE.match(line)
                    if match:
                        prop_name = match.group(1)
                        ...
                        # 生成代码
                        if cpp:
                            annotation = annotation.replace(".", "::")
                            content += (TAB + TAB + "{VehicleProperty::" + prop_name + ", " +
                                        annotation + "},")
                        else:
                            content += (TAB + TAB + "Map.entry(VehicleProperty." + prop_name + ", " +
                                        annotation + "),")
                        id += 1

        ...
```
，这段脚本的作用是从`VehicleProperty.aidl`文件中解析注释并生成C++或Java代码。给定一个属性如：
```java
/**
 * VIN of vehicle
 *
 * @change_mode VehiclePropertyChangeMode.STATIC
 * @access VehiclePropertyAccess.READ
 */
INFO_VIN = 0x0100 + 0x10000000 + 0x01000000
        + 0x00100000, // VehiclePropertyGroup:SYSTEM,VehicleArea:GLOBAL,VehiclePropertyType:STRING
```
，则会在`ChangeModeForVehicleProperty`和`AccessForVehicleProperty`中生成条目：
```cpp
//change mode
std::unordered_map<VehicleProperty, VehiclePropertyChangeMode> ChangeModeForVehicleProperty = {
    {VehicleProperty::INFO_VIN, VehiclePropertyChangeMode::STATIC},
    ...
}

//access
std::unordered_map<VehicleProperty, VehiclePropertyAccess> AccessForVehicleProperty = {
    {VehicleProperty::INFO_VIN, VehiclePropertyAccess::READ},
    ...
}
```

```java
//change mode
public final class AccessForVehicleProperty { 
    public static final Map<Integer, Integer> values = Map.ofEntries(
        Map.entry(VehicleProperty.INFO_VIN, VehiclePropertyAccess.READ),
        ...
    )
}

//access
public final class ChangeModeForVehicleProperty {
    public static final Map<Integer, Integer> values = Map.ofEntries(
        Map.entry(VehicleProperty.INFO_VIN, VehiclePropertyChangeMode.STATIC),
        ...
    )
}
```

***
注意到，在生成代码时：
```py
content += (TAB + TAB + "{VehicleProperty::" + prop_name + ", " +
            annotation + "},")
```
，此部分`VehicleProperty`是硬编码在脚本中的，同样的，对于方法`parseAccessChangeMode`：
```cpp
void JsonConfigParser::parseAccessChangeMode(
        const Json::Value& parentJsonNode, const std::string& fieldName, int propId,
        const std::string& propStr, const std::unordered_map<VehicleProperty, T>& defaultMap,
        T* outPtr, std::vector<std::string>* errors) 
```
，`VehicleProperty`也是硬编码在代码中的，如果需要实现自定义的`XXXVehicleProperty`，则需要重写相应的内容。
***

### 解析default values

查看`parsePropValues`方法：
```cpp
bool JsonConfigParser::parsePropValues(const Json::Value& parentJsonNode,
                                        const std::string& fieldName, RawPropValues* outPtr,
                                        std::vector<std::string>* errors) {
    ...
    success &= tryParseJsonArrayToVariable(jsonValue, "int32Values",
                                            /*optional=*/true, &(outPtr->int32Values), errors);
    success &= tryParseJsonArrayToVariable(jsonValue, "floatValues",
                                            /*optional=*/true, &(outPtr->floatValues), errors);
    success &= tryParseJsonArrayToVariable(jsonValue, "int64Values",
                                            /*optional=*/true, &(outPtr->int64Values), errors);
    success &= tryParseJsonValueToVariable(jsonValue, "stringValue",
                                            /*optional=*/true, &(outPtr->stringValue), errors);
    ...
}
```
，调用了`tryParseJsonArrayToVariable`方法：
```cpp
bool JsonConfigParser::tryParseJsonArrayToVariable(const Json::Value& parentJsonNode,
                                                    const std::string& fieldName,
                                                    bool fieldIsOptional, std::vector<T>* outPtr,
                                                    std::vector<std::string>* errors) {
    ...
    auto result = mValueParser.parseArray<T>(fieldName, parentJsonNode[fieldName]);
    ...
    *outPtr = std::move(result.value());
    return true;
}
```
，调用了`JsonValueParser::parseArray`方法：
```cpp
template <class T>
Result<std::vector<T>> JsonValueParser::parseArray(const std::string& fieldName,
                                                    const Json::Value& value) const {
    ...
    std::vector<T> parsedValues;
    for (unsigned int i = 0; i < value.size(); i++) {
        auto result = parseValue<T>(fieldName, value[i]);
        ...
        parsedValues.push_back(result.value());
    }
    return std::move(parsedValues);
}
```
，调用了`parseValue`方法：
```cpp
template <class T>
Result<T> JsonValueParser::parseValue(const std::string& fieldName,
                                    const Json::Value& value) const {
    //如果value不是字符串格式的
    if (!value.isString()) {
        return convertValueToType<T>(fieldName, value);
    }
    //如果value是字符串格式的
    auto maybeTypeAndValue = maybeGetTypeAndValueName(value.asString());
    ...
    auto constantParseResult = parseConstantValue(maybeTypeAndValue.value());
    ...
    int constantValue = constantParseResult.value();
    return static_cast<T>(constantValue);
}
```
，上述方法在value不是字符串格式时直接解析为值，如果value不是字符串格式时，首先会调用`maybeGetTypeAndValueName`方法：
```cpp
//JsonConfigLoader.h
inline static const std::string DELIMITER = "::";

//JsonConfigLoader.cpp
std::optional<std::pair<std::string, std::string>> JsonValueParser::maybeGetTypeAndValueName(
        const std::string& jsonFieldValue) const {
    size_t pos = jsonFieldValue.find(DELIMITER);
    ...
    std::string type = jsonFieldValue.substr(0, pos);
    std::string valueName = jsonFieldValue.substr(pos + DELIMITER.length(), std::string::npos);
    ...
    return std::make_pair(type, valueName);
}
```
，这一步回去解析Json文件中类似`"FuelType::FUEL_TYPE_UNLEADED"`的声明，将其拆分为两部分：`type`（`"FuelType"`）和`valueName`（`"FUEL_TYPE_UNLEADED"`）。在执行完`maybeGetTypeAndValueName`后，会调用`parseConstantValue`：
```cpp
Result<int> JsonValueParser::parseConstantValue(
        const std::pair<std::string, std::string>& typeValueName) const {
    const std::string& type = typeValueName.first;
    const std::string& valueName = typeValueName.second;
    auto it = mConstantParsersByType.find(type);
    ...
    auto result = it->second->parseValue(valueName);
    ...
    return result;
}
```
，这一步会在`mConstantParsersByType`中找到与`type`对应的`ConstantParser`对象，`mConstantParsersByType`在`JsonValueParser`的构造函数中被填充：
```cpp
JsonValueParser::JsonValueParser() {
    ...
    mConstantParsersByType["VehicleProperty"] = std::make_unique<ConstantParser<VehicleProperty>>();
    ...
}
```
，`ConstantParser`的定义如下：
```cpp
template <class T>
class ConstantParser final : public ConstantParserInterface {
public:
    ConstantParser() {
        for (const T& v : ndk::enum_range<T>()) {
            std::string name = aidl::android::hardware::automotive::vehicle::toString(v);
            ...
            mValueByName[name] = toInt(v);
        }
    }

    ~ConstantParser() = default;

    Result<int> parseValue(const std::string& name) const override {
        auto it = mValueByName.find(name);
        ...
        return it->second;
    }

private:
    std::unordered_map<std::string, int> mValueByName;
};
```
，其`parseValue`方法会从`mValueByName`中找到`name`对应的值，而构造`ConstantParser`时添加。

***
注意到，在ConstantParser中，使用了`aidl::android::hardware::automotive::vehicle::toString`，对于自定义的`XXXVehicleProperty`，是无法解析的，需要重写该类。
***