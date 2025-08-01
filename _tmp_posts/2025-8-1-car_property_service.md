---
layout: post
title: "Android Automative CarPropertyService"
date:   2025-8-1
tags: [Android, Automative, CarPropertyService]
comments: true
author: braisedp
toc : true
---

<!-- more -->

## 监听Property

```java
public boolean subscribePropertyEvents(int propertyId,
        @NonNull CarPropertyEventCallback carPropertyEventCallback) {
    return subscribePropertyEvents(List.of(
            new Subscription.Builder(propertyId).setUpdateRateHz(DEFAULT_UPDATE_RATE_HZ)
            .build()), /* callbackExecutor= */ null, carPropertyEventCallback);
}

public boolean subscribePropertyEvents(int propertyId,
        @FloatRange(from = 0.0, to = 100.0) float updateRateHz,
        @NonNull CarPropertyEventCallback carPropertyEventCallback) {
    return subscribePropertyEvents(List.of(
            new Subscription.Builder(propertyId).setUpdateRateHz(updateRateHz).build()),
            /* callbackExecutor= */ null, carPropertyEventCallback);
}

public boolean subscribePropertyEvents(int propertyId, int areaId,
        @NonNull CarPropertyEventCallback carPropertyEventCallback) {
    return subscribePropertyEvents(List.of(
            new Subscription.Builder(propertyId).addAreaId(areaId).setUpdateRateHz(1f)
                    .build()),
            /* callbackExecutor= */ null, carPropertyEventCallback);
}

public boolean subscribePropertyEvents(int propertyId, int areaId,
        @FloatRange(from = 0.0, to = 100.0) float updateRateHz,
        @NonNull CarPropertyEventCallback carPropertyEventCallback) {
    Subscription subscription = new Subscription.Builder(propertyId).addAreaId(areaId)
            .setUpdateRateHz(updateRateHz).build();
    return subscribePropertyEvents(List.of(subscription), /* callbackExecutor= */ null,
            carPropertyEventCallback);
}
```

```java
public boolean subscribePropertyEvents(@NonNull List<Subscription> subscriptions,
        @Nullable @CallbackExecutor Executor callbackExecutor,
        @NonNull CarPropertyEventCallback carPropertyEventCallback) {
    requireNonNull(subscriptions);
    List<CarSubscription> subscribeOptions = convertToCarSubscribeOptions(subscriptions);
    return subscribePropertyEventsInternal(subscribeOptions, callbackExecutor,
            carPropertyEventCallback);
}
```

```java
private List<CarSubscription> convertToCarSubscribeOptions(List<Subscription> subscriptions) {
    List<CarSubscription> carSubscribeOptions = new ArrayList<>();
    for (int i = 0; i < subscriptions.size(); i++) {
        Subscription clientOption = subscriptions.get(i);
        CarSubscription internalOption = new CarSubscription();
        internalOption.propertyId = clientOption.getPropertyId();
        internalOption.areaIds = clientOption.getAreaIds();
        internalOption.updateRateHz = clientOption.getUpdateRateHz();
        internalOption.enableVariableUpdateRate = clientOption.isVariableUpdateRateEnabled();
        internalOption.resolution = clientOption.getResolution();
        carSubscribeOptions.add(internalOption);
    }
    return carSubscribeOptions;
}
```

```java
private boolean subscribePropertyEventsInternal(List<CarSubscription> subscribeOptions,
        @Nullable @CallbackExecutor Executor callbackExecutor,
        CarPropertyEventCallback carPropertyEventCallback) {
    ...

    //初始化回调函数的Executor，若为空则设置为Car car的main executor
    if (callbackExecutor == null) {
        callbackExecutor = mExecutor;
    }

    List<CarSubscription> sanitizedSubscribeOptions;

    ...

    List<CarSubscription> updatedSubscribeOptions;

    synchronized (mLock) {

        CarPropertyEventCallbackController cpeCallbackController =
                mCpeCallbackToCpeCallbackController.get(carPropertyEventCallback);
        ...

        mSubscriptionManager.stageNewOptions(carPropertyEventCallback, sanitizedSubscribeOptions);
        /**
         * 这一步向CarPropertyService注册carPropertyEventCallback
         **/
        var maybeUpdatedCarSubscriptions = applySubscriptionChangesLocked();
        ...
        updatedSubscribeOptions = maybeUpdatedCarSubscriptions.get();

        //给callback指定一个controller
        if (cpeCallbackController == null) {
            cpeCallbackController =
                    new CarPropertyEventCallbackController(carPropertyEventCallback,
                            callbackExecutor);
            mCpeCallbackToCpeCallbackController.put(carPropertyEventCallback,
                    cpeCallbackController);
        }

        //向controller中添加要监听的property
        for (int i = 0; i < sanitizedSubscribeOptions.size(); i++) {
            CarSubscription option = sanitizedSubscribeOptions.get(i);
            int propertyId = option.propertyId;
            float sanitizedUpdateRateHz = option.updateRateHz;
            int[] areaIds = option.areaIds;

            //如果监听类型是变化时回调
            if (sanitizedUpdateRateHz == 0) {
                cpeCallbackController.addOnChangeProperty(propertyId, areaIds);
            } else {
            //如果监听类型是定时回调
                cpeCallbackController.addContinuousProperty(propertyId, areaIds,
                        sanitizedUpdateRateHz, option.enableVariableUpdateRate,
                        option.resolution);
            }
            //添加controller到Property的controller set中
            ArraySet<CarPropertyEventCallbackController> cpeCallbackControllerSet =
                    mPropIdToCpeCallbackControllerList.get(propertyId);
            if (cpeCallbackControllerSet == null) {
                cpeCallbackControllerSet = new ArraySet<>();
                mPropIdToCpeCallbackControllerList.put(propertyId, cpeCallbackControllerSet);
            }
            cpeCallbackControllerSet.add(cpeCallbackController);
        }
    }

    ...

    //触发initial value事件
    try {
        mService.getAndDispatchInitialValue(getInitialValuePropIdAreaIds,
                mCarPropertyEventToService);
    }

    ...

    return true;
}
```

```java
@GuardedBy("mLock")
private Optional<List<CarSubscription>> applySubscriptionChangesLocked() {
    List<CarSubscription> updatedCarSubscriptions = new ArrayList<>();
    List<Integer> propertiesToUnsubscribe = new ArrayList<>();

    mSubscriptionManager.diffBetweenCurrentAndStage(updatedCarSubscriptions,
            propertiesToUnsubscribe);

    ...

    try {
        //需要注册的subscription
        if (!updatedCarSubscriptions.isEmpty()) {
            if (!registerLocked(updatedCarSubscriptions)) {
                Slog.e(TAG, "failed to register subscriptions: " + updatedCarSubscriptions);
                mSubscriptionManager.dropCommit();
                return Optional.empty();
            }
        }

        //需要注销的subscription
        if (!propertiesToUnsubscribe.isEmpty()) {
            for (int i = 0; i < propertiesToUnsubscribe.size(); i++) {
                if (!unregisterLocked(propertiesToUnsubscribe.get(i))) {
                    Slog.w(TAG, "Failed to unsubscribe to: " + VehiclePropertyIds.toString(
                            propertiesToUnsubscribe.get(i)));
                    mSubscriptionManager.dropCommit();
                    return Optional.empty();
                }
            }
        }
    } 
    ...

    mSubscriptionManager.commit();
    return Optional.of(updatedCarSubscriptions);
}
```

```java
private boolean registerLocked(List<CarSubscription> options) {
    try {
        mService.registerListener(options, mCarPropertyEventToService);
    } 
    ...
}
```

```java
public void registerListener(List<CarSubscription> carSubscriptions,
        ICarPropertyEventListener carPropertyEventListener)
        throws IllegalArgumentException, ServiceSpecificException {
    ...
    List<CarSubscription> sanitizedOptions =
            validateAndSanitizeSubscriptions(carSubscriptions);

    CarPropertyServiceClient finalClient;
    synchronized (mLock) {
        //创建一个client对象
        CarPropertyServiceClient client = getOrCreateClientForBinderLocked(
                carPropertyEventListener);
        if (client == null) {
            return;
        }

        for (int i = 0; i < sanitizedOptions.size(); i++) {
            CarSubscription option = sanitizedOptions.get(i);
            mSubscriptionUpdateRateHistogram.logSample(option.updateRateHz);
            ...
        }

        //写入stage
        mSubscriptionManager.stageNewOptions(client, sanitizedOptions);

        //stage写入current
        try {
            applyStagedChangesLocked();
        } 

        ...

        mSubscriptionManager.commit();

        //设置client监听的property
        for (int i = 0; i < sanitizedOptions.size(); i++) {
            CarSubscription option = sanitizedOptions.get(i);
            if (option.updateRateHz != 0) {
                client.addContinuousProperty(
                        option.propertyId, option.areaIds, option.updateRateHz,
                        option.enableVariableUpdateRate, option.resolution);
            } else {
                client.addOnChangeProperty(option.propertyId, option.areaIds);
            }
        }
        finalClient = client;
    }

    ...
}

```

```java
void applyStagedChangesLocked() throws ServiceSpecificException {
    List<CarSubscription> filteredSubscriptions = new ArrayList<>();
    List<Integer> propertyIdsToUnsubscribe = new ArrayList<>();
    mSubscriptionManager.diffBetweenCurrentAndStage(/* out */ filteredSubscriptions,
            /* out */ propertyIdsToUnsubscribe);
    ...

    if (!filteredSubscriptions.isEmpty()) {
        try {
            mPropertyHalService.subscribeProperty(filteredSubscriptions);
        } 
        ...
    }

    for (int i = 0; i < propertyIdsToUnsubscribe.size(); i++) {
        ...
        try {
            mPropertyHalService.unsubscribeProperty(propertyIdsToUnsubscribe.get(i));
        } 
        ...
    }
}
```

```java
public void subscribeProperty(List<CarSubscription> carSubscriptions)
        throws ServiceSpecificException {
    synchronized (mLock) {
        for (int i = 0; i < carSubscriptions.size(); i++) {
            ...
            mSubManager.stageNewOptions(new ClientType(CAR_PROP_SVC_REQUEST_ID),
                    List.of(newCarSubscription(halPropId, areaIds, updateRateHz,
                    carSubscription.enableVariableUpdateRate, carSubscription.resolution)));
        }
        try {
            updateSubscriptionRateLocked();
        }
        ...
    }
}
```

```java
private void updateSubscriptionRateLocked() throws ServiceSpecificException {
    ArrayList<CarSubscription> diffSubscribeOptions = new ArrayList<>();
    List<Integer> propIdsToUnsubscribe = new ArrayList<>();
    mSubManager.diffBetweenCurrentAndStage(diffSubscribeOptions, propIdsToUnsubscribe);
    try {
        if (!diffSubscribeOptions.isEmpty()) {
            ...
            mVehicleHal.subscribeProperty(this, toHalSubscribeOptions(diffSubscribeOptions));
        }
        for (int halPropId : propIdsToUnsubscribe) {
            ...
            mVehicleHal.unsubscribeProperty(this, halPropId);
        }
        mSubManager.commit();
    }
    ...
}
```

```java
public void subscribeProperty(HalServiceBase service, List<HalSubscribeOptions>
        halSubscribeOptions) throws IllegalArgumentException, ServiceSpecificException {
    synchronized (mLock) {
        PairSparseArray<RateInfo> previousState = cloneState(mRateInfoByPropIdAreaId);
        SubscribeOptions[] subscribeOptions = createVhalSubscribeOptionsLocked(
                service, halSubscribeOptions);
        ...
        try {
            mSubscriptionClient.subscribe(subscribeOptions);
        }
        ...
    }
}
```
