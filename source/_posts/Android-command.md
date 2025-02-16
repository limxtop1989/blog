---
title: Android command
date: 2025-02-16 11:23:48
categories:
- [Android]
tags: Android, adb, egrep, 正则表达式，
---

# 概述
在 Android 开发过程中，命令行工具的使用能极大提升开发效率。命令行不仅能帮助开发者快速调试、自动化流程，还能在开发过程中快速定位问题、进行批量操作等，本文将详细展开介绍。

# Log 过滤查询

日志分析时，我们时常需要将包含某个关键字的日志过滤出来，如果只是在一个文件里，使用文本编辑器都可以实现目的。但如果需要高级的查询条件，特别是待分析的日志散落在多个目录下，或者需要实时过滤 `adb logcat -b all` 的输出时，命令行时最简洁高效但。比如我们需要在多个日志文件里，将 “18:20:任意多个字符keyword1 & 18:21:任意多个字符keyword1 & keyword2”的日志过滤高亮显示出来，如下命令即可。

```Bash
egrep -irn --color '18:2[01]:.*keyword1|keyword2'
```

配合管道命令，就可以边操作，边实时过滤查看日志输出。代码怎么执行的，运行时逻辑大概一目了然。

`adb logcat -b all | egrep -in --color 'keyword1|keyword2'`

每个系统命令都可以在 terminal 通过 `man` 命令查询其API，比如 `egrep` 输入 `man egrep` 获得其使用说明。

:important: [正则表达式](https://en.wikipedia.org/wiki/Regular_expression)
:important: 做开发总是要学会英语，阅读英文资料的，如何一年半载掌握英语阅读能力，见[英语自学指南]()
# 文本替换
我们可以使用 sed 命令来替换所有 Log.d 为 Log.i：

```bash
find . -name "*.java" -exec sed -i 's/Log\.d/Log\.i/g' {} +
```
- find . -name "*.java"：查找所有 .java 文件。
- -exec：对找到的文件执行后面的命令。
- sed -i 's/Log\.d/Log\.i/g'：在文件中替换 Log.d 为 Log.i，-i 表示直接修改文件。

# 查看当前 resume 的 Activity
```bash
adb shell dumpsys activity activities | egrep -in --color 'dumpsys activity activities' -A 100
```

日志输出如下：
```bash
1:ACTIVITY MANAGER ACTIVITIES (dumpsys activity activities)
2-Display #0 (activities from top to bottom):
3-  * Task{9cb2e7c #7568 type=standard A=10265:com.tencent.mm U=0 visible=true visibleRequested=true mode=fullscreen translucent=false sz=3}
4-    mLastPausedActivity: ActivityRecord{8d3b19b u0 com.tencent.mm/.plugin.offline.ui.WalletOfflineEntranceUI t-1 f}}
5-    mLastNonFullscreenBounds=Rect(276, 696 - 805, 1776)
6-    isSleeping=false
7-    topResumedActivity=ActivityRecord{3d295a u0 com.tencent.mm/.framework.app.UIPageFragmentActivity t7568}
8-    * Hist  #2: ActivityRecord{3d295a u0 com.tencent.mm/.framework.app.UIPageFragmentActivity t7568}
9-      packageName=com.tencent.mm processName=com.tencent.mm
10-      launchedFromUid=10265 launchedFromPackage=com.tencent.mm launchedFromFeature=null userId=0
11-      app=ProcessRecord{94fb2cb 29639:com.tencent.mm/u0a265}
12-      Intent { cmp=com.tencent.mm/.framework.app.UIPageFragmentActivity (has extras) }
13-      rootOfTask=false task=Task{9cb2e7c #7568 type=standard A=10265:com.tencent.mm}
14-      taskAffinity=10265:com.tencent.mm
15-      mActivityComponent=com.tencent.mm/.framework.app.UIPageFragmentActivity
53-    * Hist  #1: ActivityRecord{c7145cd u0 com.tencent.mm/.plugin.mall.ui.MallIndexUIv2 t7568}
54-      packageName=com.tencent.mm processName=com.tencent.mm
55-      launchedFromUid=10265 launchedFromPackage=com.tencent.mm launchedFromFeature=null userId=0
56-      app=ProcessRecord{94fb2cb 29639:com.tencent.mm/u0a265}
57-      Intent { flg=0x20000000 cmp=com.tencent.mm/.plugin.mall.ui.MallIndexUIv2 (has extras) }
58-      rootOfTask=false task=Task{9cb2e7c #7568 type=standard A=10265:com.tencent.mm}
59-      taskAffinity=10265:com.tencent.mm
60-      mActivityComponent=com.tencent.mm/.plugin.mall.ui.MallIndexUIv2
98-    * Hist  #0: ActivityRecord{f0316e7 u0 com.tencent.mm/.ui.LauncherUI t7568}
99-      packageName=com.tencent.mm processName=com.tencent.mm
100-      launchedFromUid=10265 launchedFromPackage=com.tencent.mm launchedFromFeature=null userId=0
101-      app=ProcessRecord{94fb2cb 29639:com.tencent.mm/u0a265}
```
# 查看当前 resume 的 Fragment

```bash
adb shell dumpsys activity top
```

更多 [dumpsys](https://developer.android.com/tools/dumpsys) 见链接和自行搜索。
# 查看 Activity 启动调度流程

```bash
adb logcat -b all | egrep -in --color 'am_|wm_|activitytask|displayed'
```

以桌面启动 wordcard 应用为例，日志输出：
```bash
41911:02-15 22:11:28.678  1622  4086 I wm_task_created: 7584
41915:02-15 22:11:28.686  1622  4086 I wm_task_moved: [7584,7584,0,1,9]
41916:02-15 22:11:28.686  1622  4086 I wm_task_to_front: [0,7584,0]
41917:02-15 22:11:28.687  1622  4086 I wm_create_task: [0,7584,7584,0]
41918:02-15 22:11:28.687  1622  4086 I wm_create_activity: [0,107658451,7584,com.maxim.wordcard.free/com.maxim.wordcard.MainActivity,android.intent.action.MAIN,NULL,NULL,270532608]
41919:02-15 22:11:28.687  1622  4086 I wm_task_moved: [7584,7584,0,1,9]
41921:02-15 22:11:28.689  1622  4086 I wm_pause_activity: [0,62924082,com.google.android.apps.nexuslauncher/.NexusLauncherActivity,userLeaving=true,pauseBackTasks]
41923:02-15 22:11:28.691  1622  4086 I ActivityTaskManager: START u0 {act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10200000 cmp=com.maxim.wordcard.free/com.maxim.wordcard.MainActivity bnds=[787,671][1034,986]} with LAUNCH_MULTIPLE from uid 10232 (BAL_ALLOW_ALLOWLISTED_COMPONENT) result code=0
41928:02-15 22:11:28.693  1622  1782 I am_uid_running: 10320
41929:02-15 22:11:28.693 27385 27385 I wm_on_top_resumed_lost_called: [62924082,com.google.android.apps.nexuslauncher.NexusLauncherActivity,topStateChangedWhenResumed]
41937:02-15 22:11:28.694 27385 27385 I wm_on_paused_called: [62924082,com.google.android.apps.nexuslauncher.NexusLauncherActivity,performPause,0]
41938:02-15 22:11:28.695  1622  4091 I wm_add_to_stopping: [0,62924082,com.google.android.apps.nexuslauncher/.NexusLauncherActivity,makeInvisible]
41945:02-15 22:11:28.711  1622  1809 I am_proc_start: [0,2035,10320,com.maxim.wordcard.free,next-top-activity,{com.maxim.wordcard.free/com.maxim.wordcard.MainActivity}]
41948:02-15 22:11:28.720  1622  1805 I am_pss  : [27027,10155,com.google.android.googlequicksearchbox:search,185773056,120262656,48736256,273444864,0,3,20]
41971:02-15 22:11:28.732  1622  4086 I am_proc_bound: [0,2035,com.maxim.wordcard.free]
41976:02-15 22:11:28.735  1622  4086 I am_uid_active: 10320
42055:02-15 22:11:28.871  1622  1965 I am_unfreeze: [27271,com.google.android.adservices.api,6]
42056:02-15 22:11:28.871  1622  4086 I am_uid_active: 10257
42081:02-15 22:11:28.935  1622  1965 I am_unfreeze: [23781,com.android.vending,6]
42092:02-15 22:11:28.969  1622  4091 I am_uid_active: 10174
42093:02-15 22:11:28.969  1622  1965 I am_unfreeze: [26122,com.google.android.webview:webview_service,6]
42100:02-15 22:11:28.986  1622  4091 I am_uid_running: 99529
42107:02-15 22:11:28.998  1622  1809 I am_proc_start: [0,2196,99529,com.google.android.webview:sandboxed_process0:org.chromium.content.app.SandboxedProcessService0:0,,{com.maxim.wordcard.free/org.chromium.content.app.SandboxedProcessService0:0}]
42118:02-15 22:11:29.013  1622  4086 I am_proc_bound: [0,2196,com.google.android.webview:sandboxed_process0:org.chromium.content.app.SandboxedProcessService0:0]
42123:02-15 22:11:29.028  1622  4086 I wm_restart_activity: [0,107658451,7584,com.maxim.wordcard.free/com.maxim.wordcard.MainActivity]
42124:02-15 22:11:29.029  1622  4086 I wm_set_resumed_activity: [0,com.maxim.wordcard.free/com.maxim.wordcard.MainActivity,minimalResumeActivityLocked - onActivityStateChanged]
42125:02-15 22:11:29.030  1622  4086 I am_uid_active: 99529
42151:02-15 22:11:29.158  2035  2035 I wm_on_create_called: [107658451,com.maxim.wordcard.MainActivity,performCreate,12]
42153:02-15 22:11:29.162  2035  2035 I wm_on_start_called: [107658451,com.maxim.wordcard.MainActivity,handleStartActivity,4]
42176:02-15 22:11:29.229  2035  2035 I wm_on_resume_called: [107658451,com.maxim.wordcard.MainActivity,RESUME_ACTIVITY,0]
42178:02-15 22:11:29.240  2035  2035 I wm_on_top_resumed_gained_called: [107658451,com.maxim.wordcard.MainActivity,topStateChangedWhenResumed]
42206:02-15 22:11:29.268  1622  1783 I wm_wallpaper_surface: [0,0,Window{277b556 u0 com.google.android.apps.nexuslauncher/com.google.android.apps.nexuslauncher.NexusLauncherActivity}]
42249:02-15 22:11:29.328  1622  1779 I wm_activity_launch_time: [0,107658451,com.maxim.wordcard.free/com.maxim.wordcard.MainActivity,661]
42250:02-15 22:11:29.328  1622  1779 I ActivityTaskManager: Displayed com.maxim.wordcard.free/com.maxim.wordcard.MainActivity for user 0: +661ms
```

- wm_task_created ： 创建 WordCard 所在的 Task。
- wm_create_activity： WMS 创建好 WordCard 的 MainActivity。
- wm_on_create_called： Activity 执行 onCreate 完成后，通知WMS 生命周期回调完成。
- Displayed： 启动 com.maxim.wordcard.free 耗时 661ms。
# Activity 启动耗时
```Bash
ActivityTaskManager: Displayed com.google.android.dialer/.extensions.GoogleDialtactsActivity for user 0: +514ms
```

# 安装 apk
```shell
adb install -r fileName.apk
```
- -b :  :important: Back up any existing files before overwriting them by renaming them to file.old.  See -B for specifying a different backup suffix.

```shell
adb push fileName.apk remote
```
# 应用卸载
```shell
adb shell pm uninstall com.example.MyApp
```
# adb启动activity
Call activity manager
```shell
$ adb shell am start -n ｛包(package)名｝/｛包名｝.{活动(activity)名称}
```

如：启动浏览器
```shell
adb shell am start -n com.android.systemui/com.android.browser.BrowserActivity --ei x 1 --ei y 2
```
- ei : Intent 携带整型参数 x = 1 & y = 2.

携带启动参数见：Specification for intent arguments 一节
# adb启动service
```shell
$ adb shell am start service -n ｛包(package)名｝/｛包名｝.{服务(service)名称}
```

如：启动自己应用中一个service
```shell
adb shell am startservice -n com.android.traffic/com.android.traffic.maniservice
```
携带启动参数见：Specification for intent arguments 一节
# adb发送broadcast
$ adb shell $ am broadcast -a <广播动作> 如：发送一个网络变化的广播
```
adb shell am broadcast -a android.net.conn.CONNECTIVITY_CHANGE
```
携带启动参数见：Specification for intent arguments 一节
# 清空应用的文件，包含数据库和 SharePreference
Call package manager
```bash
adb shell pm clear package
```
# Copy files to and from a device
To copy a file or directory and its sub-directories _from_ the device, do the following:

```shell
adb pull remote local
```

To copy a file or directory and its sub-directories _to_ the device, do the following:

```shell
adb push local remote
```

Replace `local` and `remote` with the paths to the target files/directory on your development machine (local) and on the device (remote). For example:

```shell
adb push myfile.txt /sdcard/myfile.txt
```

示例：截屏
```bash
adb shell screencap -p /sdcard/screenshot.png
adb pull /sdcard/screenshot.png
```
- 第一条命令在设备上截屏并保存到 /sdcard/screenshot.png。
- 第二条命令将截屏文件从设备拉取到本地。
# Call package manager (pm)
Within an adb shell, you can issue commands with the package manager (pm) tool to perform actions and queries on app packages installed on the device.

While in a shell, the pm syntax is:
```shell
pm command
```

You can also issue a package manager command directly from adb without entering a remote shell. For example:



| Command                     | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| install [options] path      | Install a package, specified by path, to the system.<br>Options:<br><br>`-r`: Reinstall an existing app, keeping its data.<br>`-t`: Allow test APKs to be installed. Gradle generates a test APK when you have only run or debugged your app or have used the Android Studio Build > Build APK command. If the APK is built using a developer preview SDK, you must include the -t option with the install command if you are installing a test APK.<br>`-d`: Allow version code downgrade. |
| uninstall [options] package | Removes a package from the system.<br>Options:<br><br>-k: Keep the data and cache directories after package removal.                                                                                                                                                                                                                                                                                                                                                                        |
| clear package               | Delete all data associated with a package.                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
# Take a screenshot
The screencap command is a shell utility for taking a screenshot of a device display.

While in a shell, the screencap syntax is:
```shell
screencap filename
```
To use screencap from the command line, enter the following:
```shell
adb shell screencap /sdcard/screen.png
```
Here's an example screenshot session, using the adb shell to capture the screenshot and the pull command to download the file from the device:
```shell
$ adb shell
shell@ $ screencap /sdcard/screen.png
shell@ $ exit
$ adb pull /sdcard/screen.png
```

# Record a video
The screenrecord command is a shell utility for recording the display of devices running Android 4.4 (API level 19) and higher. The utility records screen activity to an MPEG-4 file. You can use this file to create promotional or training videos or for debugging and testing.

In a shell, use the following syntax:

```shell
screenrecord [options] filename
```

To use screenrecord from the command line, enter the following:

```shell
adb shell screenrecord /sdcard/demo.mp4
```

Stop the screen recording by pressing Control+C. Otherwise, the recording stops automatically at three minutes or the time limit set by --time-limit.

To begin recording your device screen, run the screenrecord command to record the video. Then, run the pull command to download the video from the device to the host computer. Here's an example recording session:

```shell
$ adb shell
shell@ $ screenrecord --verbose /sdcard/demo.mp4
(press Control + C to stop)
shell@ $ exit
$ adb pull /sdcard/demo.mp4
```

The screenrecord utility can record at any supported resolution and bit rate you request, while retaining the aspect ratio of the device display. The utility records at the native display resolution and orientation by default, with a maximum length of three minutes.

Limitations of the screenrecord utility:

- Audio is not recorded with the video file.
- Video recording is not available for devices running Wear OS.
- Some devices might not be able to record at their native display resolution. If you encounter problems with screen recording, try using a lower screen resolution.
- Rotation of the screen during recording is not supported. If the screen does rotate during recording, some of the screen is cut off in the recording.


Table 4. screenrecord options

| Options             | Description                                                                                                                                                                                                                                                                                                      |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| --help              | Display command syntax and options                                                                                                                                                                                                                                                                               |
| --size widthxheight | Set the video size: 1280x720. The default value is the device's native display resolution (if supported), 1280x720 if not. For best results, use a size supported by your device's Advanced Video Coding (AVC) encoder.                                                                                          |
| --bit-rate rate     | Set the video bit rate for the video, in megabits per second. The default value is 20Mbps. You can increase the bit rate to improve video quality, but doing so results in larger movie files. The following example sets the recording bit rate to 6Mbps:<br>`screenrecord --bit-rate 6000000 /sdcard/demo.mp4` |
| --time-limit time   | Set the maximum recording time, in seconds. The default and maximum value is 180 (3 minutes).                                                                                                                                                                                                                    |
| --rotate            | Rotate the output 90 degrees. This feature is experimental.                                                                                                                                                                                                                                                      |
| --verbose           | Display log information on the command-line screen. If you do not set this option, the utility does not display any information while running.                                                                                                                                                                   |

# Specification for intent arguments

For activity manager commands that take an `intent` argument, you can specify the intent with the following options:

`-a action`

Specify the intent action, such as `android.intent.action.VIEW`. You can declare this only once.

`-d data_uri`

Specify the intent data URI, such as `content://contacts/people/1`. You can declare this only once.

`-t mime_type`

Specify the intent MIME type, such as `image/png`. You can declare this only once.

`-c category`

Specify an intent category, such as `android.intent.category.APP_CONTACTS`.

`-n component`

Specify the component name with package name prefix to create an explicit intent, such as `com.example.app/.ExampleActivity`.

`-f flags`

Add flags to the intent, as supported by `[setFlags()](https://developer.android.com/reference/android/content/Intent#setFlags(int))`.

`--esn extra_key`

Add a null extra. This option is not supported for URI intents.

`-e | --es extra_key extra_string_value`

Add string data as a key-value pair.

`--ez extra_key extra_boolean_value`

Add boolean data as a key-value pair.

`--ei extra_key extra_int_value`

Add integer data as a key-value pair.

`--el extra_key extra_long_value`

Add long data as a key-value pair.

`--ef extra_key extra_float_value`

Add float data as a key-value pair.

`--eu extra_key extra_uri_value`

Add URI data as a key-value pair.

`--ecn extra_key extra_component_name_value`

Add a component name, which is converted and passed as a `[ComponentName](https://developer.android.com/reference/android/content/ComponentName)` object.

`--eia extra_key extra_int_value[,extra_int_value...]`

Add an array of integers.

`--ela extra_key extra_long_value[,extra_long_value...]`

Add an array of longs.

`--efa extra_key extra_float_value[,extra_float_value...]`

Add an array of floats.

`--grant-read-uri-permission`

Include the flag `[FLAG_GRANT_READ_URI_PERMISSION](https://developer.android.com/reference/android/content/Intent#FLAG_GRANT_READ_URI_PERMISSION)`.

`--grant-write-uri-permission`

Include the flag `[FLAG_GRANT_WRITE_URI_PERMISSION](https://developer.android.com/reference/android/content/Intent#FLAG_GRANT_WRITE_URI_PERMISSION)`.

`--debug-log-resolution`

Include the flag `[FLAG_DEBUG_LOG_RESOLUTION](https://developer.android.com/reference/android/content/Intent#FLAG_DEBUG_LOG_RESOLUTION)`.

`--exclude-stopped-packages`

Include the flag `[FLAG_EXCLUDE_STOPPED_PACKAGES](https://developer.android.com/reference/android/content/Intent#FLAG_EXCLUDE_STOPPED_PACKAGES)`.

`--include-stopped-packages`

Include the flag `[FLAG_INCLUDE_STOPPED_PACKAGES](https://developer.android.com/reference/android/content/Intent#FLAG_INCLUDE_STOPPED_PACKAGES)`.

`--activity-brought-to-front`

Include the flag `[FLAG_ACTIVITY_BROUGHT_TO_FRONT](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_BROUGHT_TO_FRONT)`.

`--activity-clear-top`

Include the flag `[FLAG_ACTIVITY_CLEAR_TOP](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_CLEAR_TOP)`.

`--activity-clear-when-task-reset`

Include the flag `[FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET)`.

`--activity-exclude-from-recents`

Include the flag `[FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS)`.

`--activity-launched-from-history`

Include the flag `[FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY)`.

`--activity-multiple-task`

Include the flag `[FLAG_ACTIVITY_MULTIPLE_TASK](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_MULTIPLE_TASK)`.

`--activity-no-animation`

Include the flag `[FLAG_ACTIVITY_NO_ANIMATION](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_NO_ANIMATION)`.

`--activity-no-history`

Include the flag `[FLAG_ACTIVITY_NO_HISTORY](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_NO_HISTORY)`.

`--activity-no-user-action`

Include the flag `[FLAG_ACTIVITY_NO_USER_ACTION](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_NO_USER_ACTION)`.

`--activity-previous-is-top`

Include the flag `[FLAG_ACTIVITY_PREVIOUS_IS_TOP](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_PREVIOUS_IS_TOP)`.

`--activity-reorder-to-front`

Include the flag `[FLAG_ACTIVITY_REORDER_TO_FRONT](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_REORDER_TO_FRONT)`.

`--activity-reset-task-if-needed`

Include the flag `[FLAG_ACTIVITY_RESET_TASK_IF_NEEDED](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_RESET_TASK_IF_NEEDED)`.

`--activity-single-top`

Include the flag `[FLAG_ACTIVITY_SINGLE_TOP](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_SINGLE_TOP)`.

`--activity-clear-task`

Include the flag `[FLAG_ACTIVITY_CLEAR_TASK](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_CLEAR_TASK)`.

`--activity-task-on-home`

Include the flag `[FLAG_ACTIVITY_TASK_ON_HOME](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_TASK_ON_HOME)`.

`--receiver-registered-only`

Include the flag `[FLAG_RECEIVER_REGISTERED_ONLY](https://developer.android.com/reference/android/content/Intent#FLAG_RECEIVER_REGISTERED_ONLY)`.

`--receiver-replace-pending`

Include the flag `[FLAG_RECEIVER_REPLACE_PENDING](https://developer.android.com/reference/android/content/Intent#FLAG_RECEIVER_REPLACE_PENDING)`.

`--selector`

Requires the use of `-d` and `-t` options to set the intent data and type.
