---
layout: post
title: "记一个 Android 14 适配引发的Android 存储权限问题"
date:  2023-11-05 18:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android", "存储"]
---

## 一、bug 背景
项目中有下面这样一段代码，在 Android T 版本运行正常，现在适配到 Android U 上之后，运行时 crash 了。。。。

```
...
values.put(MediaStore.Images.Media.DATA, file.absolutePath)
values.put(MediaStore.Images.Media.DISPLAY_NAME, file.name)
...
resolver.update(uri, values, null, null)
```

大概的错误信息如下：

![](/assets/images/技术/编程/Android/Android%2014%20存储权限/pic1.png)

因为涉及到 Android 的媒体权限，这篇文章主要是针对 Android 媒体权限做的一些总结。

## 二、Android 数据存储
随着 Android 版本迭代，官方也在不断优化 Android 数据存储方式，其中涉及到数据存储的性能、安全性、用户隐私等诸多因素。

如今，最新的官方文档的介绍如下：
Android 使用的文件系统提供了如下几种保存应用数据的选项：

- **应用专属存储空间：** 存储仅供应用使用的文件，可以存储到内部存储卷中的专属目录或外部存储空间中的其他专属目录。使用内部存储空间中的目录保存其他应用不应访问的敏感信息。
- **共享存储：** 存储您的应用打算与其他应用共享的文件，包括媒体、文档和其他文件。
- **偏好设置：** 以键值对形式存储私有原始数据。DataStore 提供了一种更现代的方式来存储本地数据。您应该使用 DataStore 而非 SharedPreferences
- **数据库：** 使用 Room 持久性库将结构化数据存储在专用数据库中。

|文件类型|内容类型|访问方法|所需权限|其他应用是否可以访问|卸载应用时是否移除文件|
|---|---|---|---|---|---|
|应用专属文件|仅供您的应用使用的文件|从内部存储空间访问，可以使用 getFilesDir() 或 getCacheDir() 方法从外部存储空间访问，可以使用 getExternalFilesDir() 或 getExternalCacheDir() 方法|	从内部存储空间访问，可以使用 getFilesDir() 或 getCacheDir() 方法从外部存储空间访问，可以使用 getExternalFilesDir() 或 getExternalCacheDir() 方法|	否|是|
|媒体文件|可共享的媒体文件（图片、音频文件、视频）|可共享的媒体文件（图片、音频文件、视频）|在 Android 11（API 级别 30）或更高版本中，访问其他应用的文件需要 READ_EXTERNAL_STORAGE。在 Android 10（API 级别 29）中，访问其他应用的文件需要 READ_EXTERNAL_STORAGE 或 WRITE_EXTERNAL_STORAGE。在 Android 9（API 级别 28）或更低版本中，访问所有文件均需要相关权限|	是，但其他应用需要 READ_EXTERNAL_STORAGE 权限	|否|
|文档和其它文件|其他类型的可共享内容，包括已下载的文件|存储访问框架|无|是，可以通过系统文件选择器访问|否|
|应用偏好设置|键值对|Jetpack Preferences 库|无|否|是|
|数据库|结构化数据|Room 持久性库|无|否|是|

### 2.1 Android 6.0 动态申请权限
Android 6.0 为了防止应用申请不必要的权限，对权限进行了分组，对于危险权限，需要动态申请权限，这里就不展开了，现在市场是 Android 6.0 以下的机器可以忽略了。

### 2.2 Android 10 作用域存储
Android 10 开始引入了作用域存储的概念。

什么是作用域存储呢？在 Android 10 以前，外部存储属于公共空间，不计入在应用程序占用的空间，所用应用都有权限随意访问，并且用户卸载了应用，对于该应用创建的文件也会被保留下来。

从 Android 10 开始，对 SD 卡的使用做了很大的限制，每个应用只有权限读取自己的外置存储空间关联的目录。获取该关联目录的代码是：

```
/storage/emulated/0/Android/data/<包名>/files
```

该目录下的文件会被记入应用程序所占用的空间。同时也会随应用卸载而被删除。

那如何访问其它的目录呢？比如读取手机相册中的图片，或者想手机相册中添加一张图片。为此， Android 系统针对文件类型进行了分类，图片、音频、视频这三类文件可以通过 MediaStore API 来进行访问，其它类型的文件需要使用系统的文件选择器来进行访问。

另外，当我们的应用程序向媒体库贡献的图片、音频或者视频会自动拥有其读写权限，不需要额外申请 `READ_EXTERNAL_STORAGE` 和 `WRITE_EXTERNAL_STORAGE` 权限。而如果你要读取其它应用程序向媒体库贡献的图片、音频或者视频，则必须要申请 READ_EXTERNAL_STORAGE 权限才行。而 `WRITE_EXTERNAL_STORAGE` 权限似乎也没什么用了，官方表示将会在未来的 Android 版本中被废弃。

在 Android 10 中对于作用域存储适配的要求不是那么严格，没有强制要求。此前的使用方式，也可以在 Android 10 手机上成功运行。而即便 targetSdkVersion 已经指定成了 29， 如果你还不想进行作用域存储的适配，只需要在 AndroidManifest.xml 文件中加入如下配置即可：

```
<manifest ... >
    <application android:requestLegacyExternalStorage="true" ...>
        ...
    </application>
</manifest>
```

然鹅， Android 11 中已经开始强制启用作用域存储。所以上面的仅做了解即可。

#### 2.2.1 读取媒体库的文件
过去直接获取相册中图片的绝对路径，现在在作用域存储当中，我们只能借助 MediaStore API 获取到图片的 Uri 。以图片为例：

```
val cursor = ContentResolverCompat.query(
            context.contentResolver,
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            null,
            null,
            null,
            "${MediaStore.MediaColumns.DATE_ADDED} desc",
            null
        )
        cursor?.use { 
            while (it.moveToNext()) {
                val id = it.getLong(it.getColumnIndexOrThrow(MediaStore.MediaColumns._ID))
                val uri = ContentUris.withAppendedId(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, id)
                println("image uri is $uri")
            }
        }
```

上面的代码是通过 ContentResolver 获取到相册中所有图片的 id，再借助 ContentUris 将 id 拼装成一个完整的 Uri 对象，一张图片的格式大致如下：

```
content://media/external/images/media/321
```

#### 2.2.2 写入文件到媒体库
向媒体库中写入文件要复杂一些，因为不同系统版本之间处理方式不太一样。
还是以图片为例。

```
fun saveBitmapToAlbum(context: Context, bitmap: Bitmap, displayName:String, mimeType: String, compressFormat: CompressFormat) {
        val values = ContentValues().apply {
            put(MediaStore.MediaColumns.DISPLAY_NAME, displayName)
            put(MediaStore.MediaColumns.MIME_TYPE, mimeType)
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                put(MediaStore.MediaColumns.RELATIVE_PATH, Environment.DIRECTORY_DCIM)
            } else {
                put(
                    MediaStore.MediaColumns.DATA,
                    "${Environment.getExternalStorageDirectory().path}/${Environment.DIRECTORY_DCIM}/$displayName"
                )
            }
        }
        val uri = context.contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values)
        uri?.let {
            val outputStream = context.contentResolver.openOutputStream(uri)
            outputStream?.use {
                bitmap.compress(compressFormat, 100, it)
            }
        }
    }
```

首先需要构建一个 ContentValues 对象，然后向这个对象添加三个重要数据：

- DISPLAY_NAME： 图片显示的名称
- MIME_TYPE： 图片的 mime 类型
- 图片的存储路径
图片的存储路径在 Android 10 和之前的系统版本处理方式不太一样。在 Android 10 中，新增了一个 `RELATIVE_PATH` 常量，标识文件存储的相对路径，可选值有
- DIRECTORY_DCIM 表示相册
- DIRECTORY_PICTURES 表示图片
- DIRECTORY_MOVIES 表示电源
- DIRECTORY_MUSIC 表示音乐

而在 Android 10 之前的系统版本中没有 RELATIVE_PATH, 需要我们使用 DATA 常量（在 Android 10 中废弃），并拼装出一个文件存储的绝对路径才行。

有了 ContentValues 对象之后，接下来调用 ContentResolver 的 insert() 方法，插入图片的 Uri。有了 Uri 之后，再向该 Uri 所对应的图片写入数据。调用 ContentResolver 的 openOutputStream() 方法获得文件的输出流，然后将 Bitmap 对象写入到该输出流中即可。

#### 2.2.3 下载文件到 Download 目录
在 Android 10 之前我们下载文件，通常会下载到 Download 目录，这是一个专门用于存放下载文件的目录。而从 Android 10 开始，我们已经不能以绝对路径的方式访问外置存储空间了。主要有以下两种方式：

1. 将文件下载到应用程序的关联目录下。这样也无需申请额外权限。前面说了应用关联目录，这样的有以下几个特点：
- 下载的文件会被计入到应用程序的占用控件当中
- 如果应用程序被卸载了，改文件也会一同被删除
- 只能被当前应用访问，其它程序没有读取权限

2. 对 Android 10 系统进行适配。仍然将文件下载到 Download 目录下。
具体操作，和向相册中添加一种图片的过程差不多，Android 10 中新增了一种 Downloads 集合，专门用于执行文件下载操作。

```
suspend fun downloadFile(context: Context, fileUrl: String, fileName: String) = withContext(Dispatchers.IO) {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.Q) {
            // Android Q 以前的使用方式，指定决对路径进行下载
            // ...

        } else {
            val values = ContentValues().apply {
                put(MediaStore.MediaColumns.DISPLAY_NAME, fileName)
                put(MediaStore.MediaColumns.RELATIVE_PATH, Environment.DIRECTORY_DOWNLOADS)
            }
            val uri =
                context.contentResolver.insert(MediaStore.Downloads.EXTERNAL_CONTENT_URI, values)

            uri?.let {
                runCatching {
                    val url = URL(fileUrl)
                    val connection = (url.openConnection() as HttpURLConnection).apply {
                        requestMethod = "GET"
                        connectTimeout = 8000
                        readTimeout = 8000
                    }
                    val inputStream = connection.inputStream
                    val bis = BufferedInputStream(inputStream)
                    val outputStream = context.contentResolver.openOutputStream(it)
                    outputStream?.let { os ->
                        val bos = BufferedOutputStream(os)
                        val buffer = ByteArray(1024)
                        var bytes = bis.read(buffer)
                        while (bytes >= 0) {
                            bos.write(buffer, 0, bytes)
                            bos.flush()
                            bytes = bis.read(buffer)
                        }
                        bos.close()
                        os.close()
                    }
                    bis.close()
                }.onSuccess {

                }.onFailure {

                }
            }
        }
    }
```

主要的注意点在于， MediaStore.Downloads 是 Android 10 中新增的 API, 如果要兼容 Android 10 以下，还需要使用之前的绝对路径方式进行文件下载。

#### 2.2.4 使用文件选择器
我们要读取 SD 卡上非图片、音频、视频类的文件，比如打开一个 PDF 文件，则不能再使用 MediaStore API 了，需要使用文件选择器。且必须是手机系统内置的文件选择器。

```
val pickFileLauncher: ActivityResultLauncher<Intent> =
        registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
            if (result.resultCode == AppCompatActivity.RESULT_OK) {
                val uri = result.data?.data  // 选择的文件的 uri
                // ... 处理结果
            }
        }

fun pickFile(context: Context) {
    val intent = Intent(Intent.ACTION_OPEN_DOCUMENT).apply { 
        addCategory(Intent.CATEGORY_OPENABLE)
        type = "*/*"
    }
    pickFileLauncher.launch(intent)
}
```

启动系统的文件选择器，这里 Intent 的 action 和 category 都是固定不变的。Type 属性可以用于对文件类型进行过滤。比如 `image/*` 标识只显示图片类型的文件，注意 type 必须要指定，否则会产生崩溃。

#### 2.2.5 特定程序选择器
通常，我们选择照片时可以使用特定程序选择器：

```
val selectPhotoIntent = Intent(Intent.ACTION_PICK).apply{
    setDataAndType(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, "image/*")
}
mRequestPhotoLauncher.launch(selectPhotoIntent)
```

这里说明：
`Intent.ACTION_OPEN_DOCUMENT` 和 `Intent.ACTION_PICK` 都是用于获取数据的 Intent Action，但是它们的使用场景和功能略有不同。

>Intent.ACTION_PICK 主要用于从已安装的应用程序中选择数据并返回结果，通常用于选择特定类型的数据，如图像、视频、音频等。比如在使用系统相册应用时，就可以使用 Intent.ACTION_PICK 来选择需要展示的照片。

>而 Intent.ACTION_OPEN_DOCUMENT 则是用于从系统文档提供程序中选择文档并返回结果，通常用于选择任何类型的文档，如 PDF、Word 文档等。通过使用 Intent.ACTION_OPEN_DOCUMENT，用户可以访问系统的文件系统，并选择任何类型的文件。除了选择文件外，Intent.ACTION_OPEN_DOCUMENT 还可以为选定的文件提供读写权限，这对于应用程序需要读写文件时非常有用。

因此，`Intent.ACTION_PICK` 更适合选择特定类型的数据，而 `Intent.ACTION_OPEN_DOCUMENT` 更适合访问系统文档和选择任何类型的文件。

### 2.3 Android 13 细化的媒体权限
Google 在 Android 13 上对本地数据访问做了更进一步的细化。

`WRITE_EXTERNAL_STORAGE` 权限还没有被废弃，但是我们几乎不可能使用它了。
但是，Google 对 `READ_EXTRERNAL_STORGE 权限下手了。从 Android 13 开始，如果你的应用程序 targetSdk 指定到了 33 或以上，那么 `READ_EXTRERNAL_STORGE` 权限就完全失去了作用，申请它将不会产生任何效果。

与此相对应的，Google 新增了 `READ_MEDIA_IMAGES`、`READ_MEDIA_VIDEO` 和 `READ_MEDIA_AUDIO` 这三个运行时权限，分别用于管理手机的照片、视频和音频文件。

以前只要申请 `READ_EXTRERNAL_STORGE` 权限就可以了，现在不行了，得按需申请。用户从而能够更加精细地了解你的应用到底申请了哪些媒体权限。

为了考虑向下的兼容性，在 `AndroidManifest.xml` 文件中应该这样写：

```
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" android:maxSdkVersion="32" />
```

也就是说，在 Android 12 及一下的系统，我们仍然要声明 READ_EXTERNAL_STORAGE 权限，在代码中动态申请权限时也要做同样的逻辑处理才行。

### 2.4 特殊高级权限 MANAGE_EXTERNAL_STORAGE
前面提到，从 Android 10 开始，申请 `READ_EXTERNAL_STORAGE` 权限，也只能读取到其它应用的媒体类型文件，如果想要获取共享存储空间中的所有文件，怎么办？比如文件管理器类的应用或者病毒扫描类应用。莫慌， Android 11 开始， Google 引出了一个特殊的权限，`MANAGE_EXTERNAL_STORAGE`, 该权限将授权读写所有共享存储内容，同时包含非媒体类型的文件。注意：获得这个权限的应用还是无法访问其它应用的专属目录，无论是外部存储还是内部存储，及私有文件以及关联目录文件，都无法访问。因为这些目录在存储卷上显示为 `Android/data/` 的子目录。

Google Play 通知, 这是 Android 11 引入的一项新的隐私政策限制。如果你在你的应用中申请了该权限，你会看到这样一条警告信息：

```
The Google Play store has a policy that limits usage of MANAGE_EXTERNAL_STORAGE
```

为了限制对共享存储的广泛访问，Google Play 商店已更新其政策，用来评估以 Android 11（API 级别 30）或更高版本为目标平台且通过 `MANAGE_EXTERNAL_STORAGE` 权限请求“所有文件访问权”的应用

大多数情况下访问其它应用程序的私有文件，更应该考虑使用 `FileProvider` 或者 `ContentProvider`。

要使用"所有文件访问权"，步骤如下：

1. 在 AndroidManifest.xml 文件中声明 `MANAGE_EXTERNAL_STORAGE` 权限。
2. 使用 `Intent.ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION` 操作将用户引导至一个系统设置页面，在该页面上，用户可以为您的应用启用以下选项：**授予所有文件的管理权限**。

如需确定你的应用是否已获得 `MANAGE_EXTERNAL_STORAGE` 权限，请调用 `Environment.isExternalStorageManager()`。

具体的执行权限范围包括：

- 对共享存储空间中的所有文件的读写访问权限。
***注意：/sdcard/Android/media⁠ 目录是共享存储空间的一部分。***
- 对 MediaStore.Files 表的内容的访问权限。
- 对 USB On-The-Go (OTG) 驱动器和 SD 卡的根目录的访问权限。
- 除 `/Android/data/`、`/sdcard/Android` 以及 `/sdcard/Android` 的大多数子目录外，对所有内部存储目录的写入权限。该写入权限包括文件路径访问权限。

一般来说，如下类型的应用才必须使用 `MANAGE_EXTERNAL_STORAGE` 权限。

- 文件管理器
- 备份和恢复应用
- 防病毒应用
- 文档管理应用
- 设备上的文件搜索
- 磁盘和文件加密
- 设备到设备数据迁移

## 三、解决 bug
前面说了这么多，跟开头提到的 bug 有什么关系？

从日志信息来看： `Mutation of _data is not allowed.` ，这个问题还得从源码来分析。找到这个异常抛出的地方：

```
/packages/providers/MediaProvider/src/com/android/providers/media/MediaStore.java
```

insertInternal() 方法中：

![](/assets/images/技术/编程/Android/Android%2014%20存储权限/pic2.png)

好家伙！还真是判断了 targetSdk >= 34 才会抛出这个异常，这也解释了为什么 T 版本上运行正常，升级到 Android U 之后会 crash。

看这段代码逻辑，首先如果我们更新的 values 的 column 信息不包含 `sDataColumns` 中的 column。就不会触发这个异常，那要看看这个 `sDataColumns` 是什么。

```
private static final ArrayMap<String, Object> sDataColumns = new ArrayMap<>();

static {
    sDataColumns.put(MediaStore.MediaColumns.DATA, null);
    sDataColumns.put(MediaStore.Images.Thumbnails.DATA, null);
    sDataColumns.put(MediaStore.Video.Thumbnails.DATA, null);
    sDataColumns.put(MediaStore.Audio.PlaylistsColumns.DATA, null);
    sDataColumns.put(MediaStore.Audio.AlbumColumns.ALBUM_ART, null);
}
```

刚好，日志中的 `_data` 刚好就是这个 MerdiaStore.MediaColumns.DATA.

其次， 如果 `isCallingPackageManager` 也不会触发这个 bug

```
private boolean isCallingPackageManager() {
  return mCallingIdentity.get().hasPermission(PERMISSION_IS_MANAGER);
}
```

```
/packages/providers/MediaProvider/src/com/android/providers/media/LocalCallingIdentity.java

 private boolean hasPermissionInternal(int permission) {
	boolean targetSdkIsAtLeastT = getTargetSdkVersion() > Build.VERSION_CODES.S_V2;
	// While we're here, enforce any broad user-level restrictions
	if ((uid == Process.SHELL_UID) && context.getSystemService(UserManager.class)
			.hasUserRestriction(UserManager.DISALLOW_USB_FILE_TRANSFER)) {
		throw new SecurityException(
				"Shell user cannot access files for user " + UserHandle.myUserId());
	}
	
	switch (permission) {
		case PERMISSION_IS_SELF:
			return checkPermissionSelf(context, pid, uid);
		case PERMISSION_IS_SHELL:
			return checkPermissionShell(uid);
		case PERMISSION_IS_MANAGER:
			return checkPermissionManager(context, pid, uid, getPackageName(), attributionTag);
		case PERMISSION_IS_DELEGATOR:
			return checkPermissionDelegator(context, pid, uid);
			
			... 
```

```
/packages/providers/MediaProvider/src/com/android/providers/media/util/PermissionUtils.java

/**
 * Check if the given package has been granted the "file manager" role on
 * the device, which should grant them certain broader access.
 */
 public static boolean checkPermissionManager(@NonNull Context context, int pid,
         int uid, @NonNull String packageName, @Nullable String attributionTag) {
     return checkPermissionForDataDelivery(context, MANAGE_EXTERNAL_STORAGE, pid, uid,
             packageName, attributionTag,
             generateAppOpMessage(packageName,sOpDescription.get()));
 }
```

可以看到，如果用户授予了应用 `MANAGE_EXTERNAL_STORAGE` 权限，则也不会触发这个异常。

自此，真相大白，针对该问题，有两种解决方案： 第一，申请 `MANAGE_EXTERNAL_STORAGE` 权限，第二，代码中去掉 `values.put(MediaStore.Images.Media.DATA, file.absolutePath)` 这个。
综合前面权限讲解，显然我们应该使用第二种解决方案。 `MediaStore.Images.Media.DATA` 这一列，我们没有必要去更新。