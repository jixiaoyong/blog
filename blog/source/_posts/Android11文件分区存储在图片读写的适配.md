---
title: Android11文件分区存储在图片读写的适配
date: 2021-06-01 14:34:25

---

# 说明

当APP目标版本是Android 10（API 29）及以后时，由于Android引入了分区存储，APP不能直接通过路径访问文件，访问外部存储空间中的媒体文件除了需要`READ_EXTERNAL_STORAGE` 或 `WRITE_EXTERNAL_STORAGE` 权限之外，需要通过其他APP分享的`Uri`读写文件，同理要给其余APP分享文件也许要通过`FileProvider`生成`Uri`并赋予对应的权限。

本文以从相册中获取图片、请求系统裁剪并返回图片为例展示对应的适配方法。

# 实际操作

1.**从相册中获取图片**

从相册中获取到的图片`Uri`一般如：`content://raw//storage/emulated/0/DCIM/Camera/IMG_20210531_183008.HEIC`

app内部要读取其内容的话，可以通过`context.getContentResolver().openInputStream(imageUri)`

或者

```java
Cursor cursor = context.getContentResolver().query(uri, filePathColumn, null, null, null);//从系统表中查询指定Uri对应的照片
            cursor.moveToFirst();
            int columnIndex = cursor.getColumnIndex(filePathColumn[0]);
            if (columnIndex >= 0) {
                picturePath = cursor.getString(columnIndex); 
                }
```

等方式读取，操作该图片。

2.**将外部文件保存到本地并获取Uri**

由于上述方式获取到的`Uri`只对本APP赋予了权限，要是希望将此图片分享给第三方APP进一步加工处理，则可能出现第三方APP没有读写权限而导致操作失败的情况，为了避免这种情况，可以将获取到的图片缓存到APP私有目录，并且重新生成`Uri`并赋予将要处理该图的第三方APP对应权限。

将外部文件缓存本地的步骤参考第一步操作即可自行完成，主要讲解一下如何将对外分享的`Uri`赋予读写权限。

下面这个方法在不同系统分别采用不同方式获取文件对应的`Uri`。

```java
    public static Uri getUriForFile(Context context, File file) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
            return FileProvider.getUriForFile(context, "com.your.app.packagename.fileprovider", file);
        } else {
            return Uri.fromFile(file);
        }
    }
```

其中`com.your.app.packagename.fileprovider`是[`FileProvider`](https://developer.android.google.cn/reference/androidx/core/content/FileProvider)的`authorities`。

要使用`FileProvider`可以参考[定义FileProvider](https://developer.android.google.cn/reference/androidx/core/content/FileProvider#ProviderDefinition)操作，一般只需要修改`authorities`即可，同时如果是开发Android库，为了避免与主工程已有的`FileProvider`冲突，可以继承`FileProvider`类并修改下文中`name`字段。

```xml
<manifest>
    ...
    <application>
        ...
        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="com.mydomain.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>
        ...
    </application>
</manifest>
```

同时，为了定义此`FileProvider`可以使用的文件目录范围，可以在`res/xml`文件夹中新建`file_paths.xml`并做如下配置，也可参考官方文档或者[Android N 7.0 FileProvider 兼容适配 原理解析](https://www.jianshu.com/p/6192f04eca11)：

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <root-path
        name="camera_photos"
        path="" />
    <external-files-path // 对应Context#getExternalFilesDir(String)获取的路径，一般为存储卡中Android/data/com.your.app.packagename/file下面的目录
        name="external_files" 
        path="." />
</paths>
```

3.**对外分享有权限的Uri**

对于上述步骤获取到的图片Uri赋予权限有两种方式：

第一种，通过`Intent`传递出去的`imgUri`，可以使用以下方式：

```java
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
intent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
intent.setDataAndType(imgUri, "image/*");
```

但是这种只适用于主动分享出去的文件，在调用第三方APP裁剪的场景中，一般还需要一个`outPutUri`用于保存裁剪之后的图片，对于这种场景，可以查询可能会调用的APP并赋予其访问权限：

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
    List<ResolveInfo> resInfoList = context.getPackageManager()
            .queryIntentActivities(intent, PackageManager.MATCH_ALL);
    for (ResolveInfo resolveInfo : resInfoList) {
        String packageName = resolveInfo.activityInfo.packageName;
        contextWrap.getActivity().grantUriPermission(packageName, outPutUri,
                Intent.FLAG_GRANT_WRITE_URI_PERMISSION | Intent.FLAG_GRANT_READ_URI_PERMISSION);
    }
}
```

这样不管是分享出去的原图，还是裁剪之后保存的图片都给第三方APP赋予了权限，保证其可以正常访问。



# 参考文献

https://developer.android.google.cn/reference/androidx/core/content/FileProvider

[Android N 7.0 FileProvider 兼容适配 原理解析](https://www.jianshu.com/p/6192f04eca11)

