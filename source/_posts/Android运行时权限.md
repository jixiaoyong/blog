---
title: Android运行时权限
tags: android
abbrlink: a2863875
date: 2018-11-25 13:10:08
---

# 简介

本文介绍了Android运行时权限的一些处理流程。

Android运行时权限是Android6之后出现的处理权限的新方式，此前开发者只需要应用需要的权限在AndroidManifest.xml文件中声明即可，现在则需要在使用到对应权限时检测是否有该权限并作出相应处理。

# 正文

## 一般流程

1. 在`AndroidManifest.xml`中声明所需权限

2. 在使用之前检查是否有该权限`checkSelfPermission()`,如果有则继续相应操作

3. 如果没有权限则检测是否需要向用户解释为什么需要该权限`ActivityCompat.shouldShowRequestPermissionRationale()`，再决定如何申请权限`requestPermissions()`

   > 需要说明的是，shouldShowRequestPermissionRationale()在第一次申请该权限时会返回false，第二次申请时返回true；
   >
   > 但是如果用户选择了*不再提醒* 则会一直返回false。所以如果判断当前并非第一次申请该权限，并且返回结果为false，就说明用户选择了不再提示，一般就需要提示用户到设置中开启对应权限。

4. 申请权限的结果在`onRequestPermissionsResult()`方法中返回，根据用户对权限的处理结果决定接下来的操作

## 代码

`onCreate()`方法中调用对应方法

```kotlin
mSharedPreferences = getSharedPreferences(packageName, Context.MODE_PRIVATE)
checkCameraDeviceAndPremissions()
```

`checkCameraDeviceAndPremissions()`具体内容

```kotlin
private fun checkCameraDeviceAndPremissions() {
     ...
    //[2]每次使用之前检测是否有改权限
        if (checkSelfPermission(Manifest.permission.CAMERA) != PackageManager.PERMISSION_GRANTED) {
            safeRequestCameraPermission()
        } else {
            safeOpenCamera(cameraId)
        }
    }
```

对申请结果进行处理：

```kotlin
override fun onRequestPermissionsResult(requestCode: Int, permissions: Array<out String>, grantResults: IntArray) {
        when (requestCode) {
            //[4]处理请求权限的结果
            REQUEST_CAMERA -> {
                if (grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    safeOpenCamera(cameraId)
                } else {
                    var noCameraPermissionDialog = AlertDialog.Builder(this@MainActivity)
                        .setTitle("警告⚠️")
                        .setMessage("没有相机权限，不可继续！\n请赋予相机权限")
                        .setCancelable(false)
                        .setPositiveButton("Yes") { _, _ ->
                            safeRequestCameraPermission()
                        }
                        .setNegativeButton("No") { _, _ -> finish() }
                        .create()
                    noCameraPermissionDialog.show()
                }
            }
        }
    }
```

`safeRequestCameraPermission()`的内容，这里才是处理申请权限的相关代码

```kotlin
private fun safeRequestCameraPermission() {
    //[3]检测是否需要解释为什么需要改权限
    if (ActivityCompat.shouldShowRequestPermissionRationale(this, Manifest.permission.CAMERA)) {
        /**
         * 第一次请求时为false
         * 第二次请求时为true，需要解释为什么需要这个权限
         * 若用户选择了不再提示则一直为false
         * 综上，如果不是第一次请求该权限，并且返回值为false，那么可以判断用户选择了不再提示
         */
        //向用户解释为什么需要改权限
        var noCameraDialog = AlertDialog.Builder(this@MainActivity)
            .setTitle("提示️")
            .setMessage("本应用正常运行需要相机权限，点击确认开始赋予权限")
            .setCancelable(false)
            .setPositiveButton("Yes") { _, _ ->
                //用户同意后开始申请权限
                doRequestCameraPermission()
            }
            .create()
        noCameraDialog.show()
    } else {
        Log.d("TAG", "count " + mSharedPreferences.getInt(KEY_COUNT_REQUEST_CAMERA_PERMISSION, 0))
        if (mSharedPreferences.getInt(KEY_COUNT_REQUEST_CAMERA_PERMISSION, 0) > 1) {
            //TODO 用户拒绝了赋予权限，并且选择了“不再提醒”，提示用户到设置中开启
        } else {
            doRequestCameraPermission()
        }
    }
}

private fun doRequestCameraPermission() {
    //每次申请权限时更新计数器
    var count = mSharedPreferences.getInt(KEY_COUNT_REQUEST_CAMERA_PERMISSION, 0) + 1
    mSharedPreferences.edit().putInt(KEY_COUNT_REQUEST_CAMERA_PERMISSION, count).apply()
    requestPermissions(arrayOf(Manifest.permission.CAMERA), REQUEST_CAMERA)
}
```



# 附录

样例代码: [Github](https://github.com/jixiaoyong/Notes-Files/commit/f41afa99c24cde1dab619462754435f2a2afc64e#diff-67ccb5e5c6c34760486a1071b23338a2)