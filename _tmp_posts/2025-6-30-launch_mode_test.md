---
layout: post
title: "Activity LaunchMode"
date:   2025-6-30
tags: [Android, Activity]
comments: true
author: braisedp
toc : true
---

<!-- more -->

## Activity LaunchMode

`Activity`一共有五种`LaunchMode`，分别是：
- `Standard`：标准模式，每启动一个`Activity`都会创建一个新的实例
- `SingleTop`： 栈顶复用模式，如果栈顶存在`Activity`实例，则复用该实例并回调`onNewIntent`方法
- `SingleTask`：栈内复用模式，如果栈内存在一个`Activity`实例，则复用该实例并置于栈顶，回调`onNewIntent`方法
- `SingleInstance`：单实例模式，此模式的`Activity`只能单独位于一个任务栈中，再次启动`Activity`时，均不会再创建`Activity`的实例，并回调`onNewIntent`方法
- `SingleInstancePerTask`：全局共用一个`Activity`，不同于`SingleInstance`模式，该模式允许其栈内运行某些其他实例

## LaunchMode 测试

### 实例

**MainActivity**

```java
public class MainActivity extends Activity implements View.OnClickListener {

    Button mainButton;

    Button subButton;
    TextView view;

    private static int count = 0;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        this.setContentView(R.layout.main_activity);
        count += 1;
        mainButton = (Button) this.findViewById(R.id.main_button);
        subButton = (Button) this.findViewById(R.id.sub_button);
        mainButton.setOnClickListener(this);
        subButton.setOnClickListener(this);
        view = this.findViewById(R.id.count_text);
        view.setText("mainActivity "+ count);
    }

    @Override
    public void onClick(View v) {
        if(v.equals(mainButton)){
            Intent intent = new Intent(getBaseContext(),MainActivity.class);
            startActivity(intent);
        }else{
            Intent intent = new Intent(getBaseContext(),SubActivity.class);
            startActivity(intent);
        }
    }
}
```

**SubActivity**

```java
public class MainActivity extends Activity implements View.OnClickListener {

    ... // same with main

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        ... // same with main
        view.setText("subActivity "+ count);
    }

    ... // same with main
}

```

### `standard`模式测试

在`manifest`文件中，配置`MainActivity`和`SubActivity`的`launchMode`为`standard`

```xml
...
<activity android:name=".MainActivity" android:exported="true" android:launchMode="standard" android:process=".mainProcess">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
<activity android:name=".SubActivity" android:exported="true" android:launchMode="standard" android:process=".mainProcess"/>
...
</manifest>
```

启动项目后，`MainActivity`显示：

![first launch](../images/2025-6-30-launch_mode_test/standard_launch.png)

调用一次`startActivity`开启一个新的`MainActivity`显示：

![second launch](../images/2025-6-30-launch_mode_test/standard_launch2.png)

此时，通过`dumpsys activity activities`查看任务栈，显示：

![second stack](../images/2025-6-30-launch_mode_test/standard_stack.png)

，可以看到，创建了两个不一样的`MainActivity`实例

### `singleTop`模式测试

修改`manifest`文件，设置`LaunchMode`为`singleTop`

```xml
...
<activity android:name=".MainActivity" android:exported="true" android:launchMode="singleTop" android:process=".mainProcess">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
<activity android:name=".SubActivity" android:exported="true" android:launchMode="singleTop" android:process=".mainProcess"/>
...
</manifest>
```
启动项目后，`MainActivity`显示：

![first launch](../images/2025-6-30-launch_mode_test/singletop_launch.png)

再启动一个`MainActivity`，显示：

![second launch](../images/2025-6-30-launch_mode_test/singletop_launch2.png)

`dumpsys activity activities`显示：

![first stack](../images/2025-6-30-launch_mode_test/singletop_stack1.png)

可以看到，由于`MainActivity`位于栈顶，此时会复用`MainActivity`

启动一次`SubActivity`后再启动`MainActivity`显示：

![third launch](../images/2025-6-30-launch_mode_test/singletop_launch3.png)

`dumpsys activity activities`显示：

![second stack](../images/2025-6-30-launch_mode_test/singletop_stack2.png)

，可以看到，由于`MainActivity`不在栈顶，会创建一个新的`MainActivity`实例并放入栈顶

### `singleTask`模式测试

修改`AndroidMainfest.xml`文件：
```xml
<activity android:name=".MainActivity" android:exported="true" android:launchMode="singleTask">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
<activity android:name=".SubActivity" android:exported="true" android:launchMode="singleTask"/>
```
启动一次`MainActivity`，显示：

![first launch](../images/2025-6-30-launch_mode_test/singletask_launch.png)

启动一次`SubActivity`，`dumpsys activity activities`显示：

![first stack](../images/2025-6-30-launch_mode_test/singletask_stack.png)

再次启动`MainActivity`，显示：

![second launch](../images/2025-6-30-launch_mode_test/singletask_launch2.png)

，`dumpsys activity activities`显示：

![second stack](../images/2025-6-30-launch_mode_test/singletask_stack2.png)

，可以看到，`MainActivity`被复用了，同时栈中元素被删除并将`MainActivity`放入了栈顶

再次启动`SubActivity`，显示：

![third launch](../images/2025-6-30-launch_mode_test/singletask_launch3.png)

，可以看到，由于`SubActivity`记录被销毁，会重新创建一个`SubActivity`实例

### `singleInstance`模式测试

修改`AndroidMainfest.xml`文件：
```xml
<activity android:name=".MainActivity" android:exported="true" android:launchMode="singleInstance">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
<activity android:name=".SubActivity" android:exported="true" android:launchMode="singleInstance"/>
```

启动`MainActivity`和`SubActivity`，`dumpsys activity activities`显示：

![first stack](../images/2025-6-30-launch_mode_test/singleinstance_stack.png)

，可以看到，为每个实例单独开启了一个进程

再次启动`MainActivity`，显示：

![second launch](../images/2025-6-30-launch_mode_test/singleinstance_launch.png)

再次启动`SubActivity`，显示：

![third launch](../images/2025-6-30-launch_mode_test/singleinstance_launch2.png)

，可以看到，`MainActivity`和`SubActivity`都被复用了

修改`AndroidManifest.xml`：
```xml
<activity android:name=".MainActivity" android:exported="true" android:launchMode="singleInstance" android:process=".mainProcess">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
<activity android:name=".SubActivity" android:exported="true" android:launchMode="singleInstance" android:process=".mainProcess"/>
```
，通过`dumpsys activity activities`，可以看到：

![second stack](../images/2025-6-30-launch_mode_test/singleinstance_stack2.png)

，即使指定了相同的process，仍然会运行在不同的task中

### `singleInstancePerTask`测试

修改`AndroidMainfest.xml`文件：
```xml
<activity android:name=".MainActivity" android:exported="true" android:launchMode="singleInstancePerTask">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
<activity android:name=".SubActivity" android:exported="true" />
```
运行一次`MainActivity`和`SubActivity`，`dumpsys`显示：

![first stack](../images/2025-6-30-launch_mode_test/singleinstancepertask_stack.png)

，可以看到，`SubActivity`可以和`MainActivity`运行在一个task中，如果修改`singleInstance`的manifest如下：
```xml
<activity android:name=".MainActivity" android:exported="true" android:launchMode="singleInstance">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
<activity android:name=".SubActivity" android:exported="true" />
```
查看其`dumpsys`显示：
![second stack](../images/2025-6-30-launch_mode_test/singleinstance_stack3.png)

，可以发现，`SubActivity`不可以与`MainActivity`运行在一个task中

再次启动`SubActivity`，观察`dumpsys`显示：
![third stack](../images/2025-6-30-launch_mode_test/singleinstancepertask_stack2.png)
，可以发现，和`singleTask`相同，同一个栈中的其他记录被删除了，并且将`MainActivity`放入了栈顶

修改`AndroidMainfest.xml`文件：
```xml
<activity android:name=".MainActivity" android:exported="true" android:launchMode="singleInstancePerTask">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
<activity android:name=".SubActivity" android:exported="true" android:launchMode="singleInstancePerTask"/>
```
此时启动`MainActivity`和`SubActivity`，`dumpsys`显示：
![forth stack](../images/2025-6-30-launch_mode_test/singleinstancepertask_stack3.png)

，由于`MainActivity`和`SubActivity`都是`singleInstancePerTask`，于是在同一个task中，只能存在他们其中的一个，于是两者运行在了不同的task中

修改manifest文件为：
```xml
<activity android:name=".MainActivity" android:exported="true" android:launchMode="singleTask" android:process=".mainProcess">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
<activity android:name=".SubActivity" android:exported="true" android:launchMode="singleInstancePerTask" android:process=".mainProcess"/>
```

启动`MainActivity`和`SubActivity`，可以看到：

![fifth stack](../images/2025-6-30-launch_mode_test/singleinstancepertask_stack4.png)

，即使指定了同一个task，`MainActivity`和`SubActivity`还是运行在了不同的task中

但是修改manifest文件为：
```xml
<activity android:name=".MainActivity" android:exported="true" android:launchMode="singleInstancePerTask" android:process=".mainProcess">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
<activity android:name=".SubActivity" android:exported="true" android:launchMode="singleTask" android:process=".mainProcess"/>
```
后，则通过`dumpsys`可以看到：
![sixth stack](../images/2025-6-30-launch_mode_test/singleinstancepertask_stack5.png)
，两个`Activity`运行在了同一个task中

## 结合`Activity`启动流程看`LaunchMode`

### 开始启动`Activity`

在调用`startActivity`方法时，会调用到`ActivityStarter`的`executeRequest`方法：
```java
private int executeRequest(Request request) {
    ...
    ActivityInfo aInfo = request.activityInfo;
    ...
    final int launchMode = aInfo != null ? aInfo.launchMode : 0;
    ...
    final ActivityRecord r = new ActivityRecord.Builder(mService)
        ...
        .setActivityInfo(aInfo)
        ...
        .build();
    ...
    mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
        request.voiceInteractor, startFlags, checkedOptions,
        inTask, inTaskFragment, balVerdict, intentGrants, realCallingUid, transition,
        isIndependent);

    ...
}
```
，可以看到，执行该方法会创建一个`ActivityRecord`并设置其`ActivityInfo`为`ainfo`，`ActivityInfo`中包含了`LaunchMode`，之后方法调用了`startActivityUnchecked`方法：

```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, ActivityOptions options, Task inTask,
        TaskFragment inTaskFragment,
        BalVerdict balVerdict,
        NeededUriGrants intentGrants, int realCallingUid, Transition transition,
        boolean isIndependentLaunch) {
        ...
        result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, options, inTask, inTaskFragment, balVerdict,
                intentGrants, realCallingUid);
        ...
}
```
，调用了`startActivityInner`方法：

### 计算启动标志`launchingTaskFlags`

```java
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, ActivityOptions options, Task inTask,
        TaskFragment inTaskFragment, BalVerdict balVerdict,
        NeededUriGrants intentGrants, int realCallingUid) {
    setInitialState(r, options, inTask, inTaskFragment, startFlags, sourceRecord,
            voiceSession, voiceInteractor, balVerdict.getCode(), realCallingUid);

    computeLaunchingTaskFlags();
    mIntent.setFlags(mLaunchFlags);

    ...
```
，在方法中，首先调用了`setInitialState`方法：
```java
private void setInitialState(ActivityRecord r, ActivityOptions options, Task inTask,
        TaskFragment inTaskFragment, int startFlags,
        ActivityRecord sourceRecord, IVoiceInteractionSession voiceSession,
        IVoiceInteractor voiceInteractor, @BalCode int balCode, int realCallingUid) {
    reset(false);
    ...
    mLaunchMode = r.launchMode;

    mLaunchFlags = adjustLaunchFlagsToDocumentMode(
                r, LAUNCH_SINGLE_INSTANCE == mLaunchMode,
                LAUNCH_SINGLE_TASK == mLaunchMode, mIntent.getFlags());
    
    ...

    if (mLaunchMode == LAUNCH_SINGLE_INSTANCE_PER_TASK) {
        mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
    }
    ...
}
```
，在方法中设置了`mLaunchMode`和`mLaunchFlags`，如果`mLaunchMode`为`LAUNCH_SINGLE_INSTANCE_PER_TASK`，则设置`mLaunchFlags`为`FLAG_ACTIVITY_NEW_TASK`，然后`startActivityInner`调用了`computeLauchingTaskFlags`方法：
```java
private void computeLaunchingTaskFlags() {
    ...
    if (mInTask == null) {
        ...
        } else if (mSourceRecord.launchMode == LAUNCH_SINGLE_INSTANCE) {
            mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
        } else if (isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
            mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
        }
    }
    ...
}
```
分为下面情况：
1. 如果源`Activity`的启动模式为`LAUCH_SINGLE_INSTANCE`，则需要为新`Activity`创建一个新任务，于是设置`mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK`
2. 如果`Activity`的启动模式为`LAUCH_SINGLE_INSTANCE`或`LAUNCH_SINGLE_TASK`，对于第一种情况，由于`inTask`为空，于是设置`mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK`

继续到`startActivtyInner`中：

### 计算目标task

```java

int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, ActivityOptions options, Task inTask,
        TaskFragment inTaskFragment, BalVerdict balVerdict,
        NeededUriGrants intentGrants, int realCallingUid) {
    
    ...

    final Task prevTopRootTask = mPreferredTaskDisplayArea.getFocusedRootTask();
    final Task prevTopTask = prevTopRootTask != null ? prevTopRootTask.getTopLeafTask() : null;

    final Task reusedTask = resolveReusableTask(includeLaunchedFromBubble);

    ...


    final Task targetTask = reusedTask != null ? reusedTask : computeTargetTask();
    final boolean newTask = targetTask == null;
    mTargetTask = targetTask;

    ...

}
```
方法首先调用了`resolveReusableTask`方法：
```java
private Task resolveReusableTask(boolean includeLaunchedFromBubble) {
    boolean putIntoExistingTask = ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0 &&
        (mLaunchFlags & FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
        || isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK);
    
    if (putIntoExistingTask) {
        if (LAUNCH_SINGLE_INSTANCE == mLaunchMode) {
            intentActivity = mRootWindowContainer.findActivity(mIntent, mStartActivity.info,
                    false );
   
            ...
        } ...
        else {
            intentActivity =
                    mRootWindowContainer.findTask(mStartActivity, mPreferredTaskDisplayArea,
                            includeLaunchedFromBubble);
        }
    }
    ...
    return intentActivity != null ? intentActivity.getTask() : null;
        
}
```
，如果`LaunchFlags`为`FLAG_ACTIVITY_NEW_TASK`且`(LaunchFlags & FLAG_ACTIVITY_MULTIPLE_TASK) == 0`，表示要在一个新进程中创建`Activity`同时不允许`Activity`在多个进程中存在实例，此时需要找到一个可以复用的task，同样，如果`LaunchMode`为`LAUNCH_SINGLE_INSTANCE`或者`LAUNCH_SINGLE_TASK`，如果有现有的包含他们的task，则就要在已存在的task中找到他们所在的task并加入，然后方法会找到`Actiivty`可以复用的task

通过`computeTargetTask`方法，得出启动`Activity`的task：
```java
private Task computeTargetTask() {
    if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
            && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
        return null;
    } else if (mSourceRecord != null) {
        return mSourceRecord.getTask();
    } else if (mInTask != null) {
        if (!mInTask.isAttached()) {
            getOrCreateRootTask(mStartActivity, mLaunchFlags, mInTask, mOptions);
        }
        return mInTask;
    } else {
        final Task rootTask = getOrCreateRootTask(mStartActivity, mLaunchFlags, null /* task */,
                mOptions);
        final ActivityRecord top = rootTask.getTopNonFinishingActivity();
        if (top != null) {
            return top.getTask();
        } else {
            rootTask.removeIfPossible("computeTargetTask");
        }
    }
    return null;
}
```
1. 当一个`Activity`不需要添加到任何一个task时，返回`null`，表示需要新建一个task
2. 否则，当源`Activity`存在时，将其添加到源`Activity`的task中
3. 如果指定的task不为空，则将其添加到指定的task中
4. 否则，获取根task，并获取根task的top `Activity`，若不为空，返回该根task


### `Activity`的复用

在`startActivityInner`中调用了如下方法：
```java
private int deliverToCurrentTopIfNeeded(Task topRootTask, NeededUriGrants intentGrants) {
    ...
}
```
，判断是否复用当前的top `Activity`

在`startActivityInner`中，还调用了如下方法：
```java
if (targetTaskTop != null) {
    ...
    startResult = recycleTask(targetTask, targetTaskTop, reusedTask, intentGrants, balVerdict);
}
```
```java
int recycleTask(Task targetTask, ActivityRecord targetTaskTop, Task reusedTask,
        NeededUriGrants intentGrants, BalVerdict balVerdict) {
    ...
    if (reusedTask != null) {
        if (targetTask.intent == null) {
            targetTask.setIntent(mStartActivity);
        } 
        ...
    }
    ...
    //将要复用的task放到最顶端
    //如果需要删除栈中对应记录则执行删除
    setTargetRootTaskIfNeeded(targetTaskTop);
    ...
    complyActivityFlags(targetTask,
            reusedTask != null ? reusedTask.getTopNonFinishingActivity() : null, intentGrants);

    if (mAddingToTask) {
        mSupervisor.getBackgroundActivityLaunchController().clearTopIfNeeded(targetTask,
                mSourceRecord, mStartActivity, mCallingUid, mRealCallingUid, mLaunchFlags,
                mBalCode);
        return START_SUCCESS;
    }

    if (mMovedToTopActivity != null) {
        targetTaskTop = mMovedToTopActivity;
    }
    targetTaskTop = targetTaskTop.finishing
            ? targetTask.getTopNonFinishingActivity()
            : targetTaskTop;


    resumeTargetRootTaskIfNeeded();

    ...

    mLastStartActivityRecord = targetTaskTop;
    return mMovedToFront ? START_TASK_TO_FRONT : START_DELIVERED_TO_TOP;
}
```


观察不同`Activity`复用的情况：
1. 对于`standard`模式：

2. 对于`singleTop`模式：在`deliverToCurrentTopIfNeeded`中，存在：
```java
final ActivityRecord top = topRootTask.topRunningNonDelayedActivityLocked(mNotTop);
final boolean dontStart = top != null
        && top.mActivityComponent.equals(mStartActivity.mActivityComponent)
        && top.mUserId == mStartActivity.mUserId
        && top.attachedToProcess()
        && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
        || LAUNCH_SINGLE_TOP == mLaunchMode)
        && (!top.isActivityTypeHome() || top.getDisplayArea() == mPreferredTaskDisplayArea);
if (!dontStart) {
    return START_SUCCESS;
}
...
top.getTaskFragment().clearLastPausedActivity();
if (mDoResume) {
    mRootWindowContainer.resumeFocusedTasksTopActivities();
}
...
deliverNewIntent(top, intentGrants);
...
```
，当top `Activity`满足复用条件时，会复用该`Activity`且向其发送一个`Intent`

3. 对于`singleTask`模式：

在`complyActivityFlags`中，执行了如下语句：
```java
else if ((mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                || isDocumentLaunchesIntoExisting(mLaunchFlags)
                || isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK,
                        LAUNCH_SINGLE_INSTANCE_PER_TASK)) {
    final ActivityRecord clearTop = targetTask.performClearTop(mStartActivity,
        mLaunchFlags, finishCount);
    if (clearTop != null && !clearTop.finishing) {
        if (finishCount[0] > 0) {
            mMovedToTopActivity = clearTop;
        }
        if (clearTop.isRootOfTask()) {
            clearTop.getTask().setIntent(mStartActivity);
        }
        deliverNewIntent(clearTop, intentGrants);
    }
}
```
，如果选择了复用任务，则对该task进行清空

4. 对于`singleInstance`模式：
在`startActivityInner`中进行判断：
```java
if (LAUNCH_SINGLE_INSTANCE == mLaunchMode && mSourceRecord != null
        && targetTask == mSourceRecord.getTask()) {
    final ActivityRecord activity = mRootWindowContainer.findActivity(mIntent,
            mStartActivity.info, false);
    if (activity != null && activity.getTask() != targetTask) {
        activity.destroyIfPossible("Removes redundant singleInstance");
    }
}
```
，如果目标`Activity`需要运行在源`Activity`的task中时，需要销毁其他task中的目标`Activity`，对于`singleInstance`模式，在`computeTargetTask`中，`mSourceRecord`是优先于`mInTask`的

5. 对于`singleInstancePerTask`模式：

```java
else if ((mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                || isDocumentLaunchesIntoExisting(mLaunchFlags)
                || isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK,
                        LAUNCH_SINGLE_INSTANCE_PER_TASK)) {
    final ActivityRecord clearTop = targetTask.performClearTop(mStartActivity,
        mLaunchFlags, finishCount);
    if (clearTop != null && !clearTop.finishing) {
        if (finishCount[0] > 0) {
            mMovedToTopActivity = clearTop;
        }
        if (clearTop.isRootOfTask()) {
            clearTop.getTask().setIntent(mStartActivity);
        }
        deliverNewIntent(clearTop, intentGrants);
    }
}
```

待更新...

## `LaunchFlags`总结

| LaunchFlag| 含义|
|-          | -   |
| `FLAG_ACTIVITY_CLEAR_TASK`| 会清空任务中的其他旧活动，需要与`FLAG_ACTIVTTY_NEW_TASK`一起使用|
|`FLAG_ACTIVITY_CLEAR_TOP`|在当前任务中时，在新活动上面的活动会被关闭，新活动不会重新启动，只会接收new intent|
| `FLAG_ACTIIVTY_NEW_TASK`| 新活动会成为历史栈中的新任务的开始，如果新活动已存在于一个为它运行的任务中，那么不会启动，只会把该任务移到屏幕最前| 
|`FLAG_ACTIVITY_MULTIPLE_TASK`|与`FLAG_ACTIVITY_NEW_TASK`和`FLAG_ACTIVITY_NEW_DOCUMENT`一起使用，会在创建`Activity`时重新创建一个task|
|`FLAG_ACTIVITY_NEW_DOCUMENT`|给启动的活动开启一个新的任务记录|

