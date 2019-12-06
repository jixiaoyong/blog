---
title: Android AlarmManager设置重复任务
date: 2019-12-6 18:18:18
---

近期有一个实现定时启动APP提醒用户的需求，一番比较之后觉得用闹钟`AlarmManager`实现比较合适，本文是对此过程的梳理，属于比较基础性的内容。

# 需求

需求需要实现

> "每天在指定时间范围内,循环提示用户直到满足指定的条件"

拆分需求：

1. 每天都要提醒
2. 在时间范围内一直循环
3. 满足条件后结束当天循环

# 方案选择

Android中可以用到的循环任务实现有`Handler`、`Timer`、` ScheduledExecutorService `（这三个可以看[这里](https://blog.csdn.net/qq_27489007/article/details/79220609)），还有最近的[`WorkManager`]( https://developer.android.google.cn/topic/libraries/architecture/workmanager )和我们要用到的[`AlarmManager`]( https://developer.android.google.cn/training/scheduling/alarms )。

>  WorkManager offers a backwards compatible (API level 14+) API leveraging [`JobScheduler`](https://developer.android.google.cn/reference/android/app/job/JobScheduler) API (API level 23+) and above to help optimize battery life and batch jobs and a combination of [`AlarmManager`](https://developer.android.google.cn/reference/android/app/AlarmManager) & [`BroadcastReceiver`](https://developer.android.google.cn/reference/android/app/BroadcastReceiver) on lower devices. 

这几个方案中，前三者都需要APP在前台运行，`WorkManager`和`AlarmManager`则在APP退出之后也可以使用，甚至在低版本上`WorkManager`底层也是通过`AlarmManager`实现的。

`WorkManager`主要倾向于保证任务在APP退出，甚至设备关机重启等情况下也会被执行，虽然也提供循环任务的，但是无法确保在精确的时间得到执行，且最小间隔15min。

相比之下，`AlarmManager`可以确保任务在指定时间（精确的时间）得到执行，并且对于循环的间隔也更加灵活。

![Android推荐选择方案](https://developer.android.google.cn/images/guide/background/bg-job-choose.svg)

# 分析

据Android官网介绍，闹钟主要用于在应用程序生命周期之外进行定时操作。

闹钟具有以下特征：

> - 它们可让您按设定的时间和/或间隔触发 intent。
> - 您可以将它们与广播接收器结合使用，以启动服务以及执行其他操作。
> - 它们在应用外部运行，因此即使应用未运行，或设备本身处于休眠状态，您也可以使用它们来触发事件或操作。
> - 它们可以帮助您最大限度地降低应用的资源要求。您可以安排定期执行操作，而无需依赖定时器或持续运行后台服务。

需要注意的是，Android为了避免重复闹钟可能带来的性能消耗，推荐使用不是很精确的`setInexactRepeating()`， 而不是精确的`setRepeating()`，并且在`API19+`之后的所有的重复闹钟都不是精确的，如果需要精确闹钟需要使用 `setWindow(int, long, long, android.app.PendingIntent)` 或`setExact(int, long, android.app.PendingIntent)`。 重复闹钟具有以下特征：

>- 闹钟类型。要了解详情，请参阅[选择闹钟类型](https://developer.android.google.cn/training/scheduling/alarms#type)。
>- 触发时间。如果您指定的触发时间为过去的时间，则闹钟会立即触发。
>- 闹钟的间隔。例如，每天一次、每小时一次、每 5 分钟一次，等等。
>- 闹钟触发的待定 intent。当您设置了使用同一待定 intent 的第二个闹钟时，它会替换原始闹钟。

## 闹钟类型

闹钟有两个类型：

1. 距离系统启动后的时间，主要用于“间隔多久重复一次”这样的需求

   `ELAPSED_REALTIME` 距离开机时间多久后启用闹钟，如果系统在休眠中则不会唤醒

   `ELAPSED_REALTIME_WAKEUP` 在系统休眠时也会唤醒系统

2. 精确的时间UTC，主要用于“在当天下午8点整开始”等这样的需求

   `RTC` 在指定的时间触发闹钟，不会唤醒机器

   `RTC_WAKEUP` 在指定时间触发闹钟，并且唤醒设备

## 触发时间

闹钟触发的时间，分为从设备上次启动时间和精准时间两种。

如果触发的时间早于当前系统时间的话，系统会根据过去的时间和重复间隔选择一个合适的时间来触发（有几分钟内的误差）。

从实际运行来看，使用`ELAPSED_*`的基本上会立即（几秒钟）触发该闹钟，并且每次循环间隔有几毫秒的误差。

使用`RTC_*`则会在刚开始的两三次出现间隔时间小于指定时间的情况，后期稳定：

设置的闹钟间隔为10分钟，闹钟开始时间早于当前时间，唤醒结果如下
```kotlin
alarmMgr.setRepeating(AlarmManager.RTC_WAKEUP,
            calendar.timeInMillis,
            10 * 60 * 1000,
            alarmIntent
        )
2019-12-06 14:13:46.696 
2019-12-06 14:16:26.634 
2019-12-06 14:24:26.765 
2019-12-06 14:34:26.579 
2019-12-06 14:43:46.785 
```

## 间隔时间

间隔时间有两种：

1.  `AlarmManager interval` 如果设置的是`setInexactRepeating()`，则需要设置`AlarmManager `指定的几种间隔时间。
2. 任意时间 `setRepeating()`方法可以使用任意时间

## 待定的intent

> 当您设置了使用同一待定 intent 的第二个闹钟时，它会替换原始闹钟 

待定的`Intent`是一个`PendingIntent`，可以用来打开`Service`，`Activity`，`Broadcast`等等。

```kotlin
private fun getPendingIntent(
        context: Context,
        action: String,
        requestCode: Int
    ): PendingIntent {
        return PendingIntent.getBroadcast(context, requestCode, Intent(action), 0)
    }
```

注意这里的`requestCode`，当不需要该闹钟时可以根据这个来取消。

## 取消闹钟

```kotlin
alarmManager.cancel(getPendingIntent(context,ACTION,RequestCode))
```

## 在重启时恢复闹钟

由于闹钟会在设备关机的时候被取消，所以需要监听设备开机广播（`android.intent.action.BOOT_COMPLETED`），并且恢复闹钟。

# 具体实现

## 设置一个每天指定时间循环的闹钟

```kotlin
private fun setupDailyAlarmClock( context: Context,startTime: Pair<Int, Int>) {
        
        val alarmMgr = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager
        val alarmIntent = getPendingIntent(
            context,
            BROADCAST_ACTION_REPEAT,
            RequestCode.START_REPEAT_INVENTORY
        )
        // Set the alarm to start at xx:xx
        val calendar: Calendar = Calendar.getInstance().apply {
            timeInMillis = System.currentTimeMillis()
            set(Calendar.HOUR_OF_DAY, startTime.first)
            set(Calendar.MINUTE, startTime.second)
            set(Calendar.SECOND, 0)
        }

        // 1 day
        alarmMgr.setRepeating(
            AlarmManager.RTC_WAKEUP,
            calendar.timeInMillis,
            AlarmManager.INTERVAL_DAY,
            alarmIntent
        )
    }
```

在每天指定时间到了之后，开始设置一个间隔10分钟唤醒一次的闹钟，直到超时或者满足指定的条件后取消该闹钟。

## 监听每日循环的闹钟

监听其发送的广播`BROADCAST_ACTION_REPEAT`。

启用当日循环闹钟：

```kotlin
    fun setupRepeatAlarmClock(context: Context) {
        val startTime = SharePreferencesUtils.sharedPreferences
            .getString(KEY_STARTT_TIME,DEF_INVENTORY_TIME)?.toFormatTime() ?: return

        val alarmMgr = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager
        val alarmIntent = getPendingIntent(
            context,
            BROADCAST_ACTION_START,
            RequestCode.START_INVENTORY
        )

        val tenMinutes = DEF_INVENTORY_DURATION * 60 * 1000
        alarmMgr.setRepeating(
            AlarmManager.ELAPSED_REALTIME_WAKEUP,//从开机后多久
            SystemClock.elapsedRealtime(),//当前自开机完后的时间
            tenMinutes,//每十分钟循环一次
            alarmIntent
        )
    }
```

在广播接收器中收听到`BROADCAST_ACTION_START`后去开启任务

##  条件满足后关闭当日循环闹钟

在收到`BROADCAST_ACTION_START`后检测到已经超时或其他满足取消条件的情况，则取消任务。

或者可以再订一个结束时间的闹钟，到时间后取消当日循环闹钟。

```kotlin
val alarmManager = getSystemService(Context.ALARM_SERVICE) as AlarmManager
alarmManager.cancel(pIntent)
```

注意这里的`pIntent`需要与设置闹钟时的`PendingIntent`一致(满足`Intent.filterEquals()`的条件)。

# 参考资料

[Android定时任务及循环任务基础大集合](https://blog.csdn.net/qq_27489007/article/details/79220609 )

[安排重复闹钟 Android官网](https://developer.android.google.cn/training/scheduling/alarms )