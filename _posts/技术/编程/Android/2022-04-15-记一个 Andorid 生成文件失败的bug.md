---
layout: post
title: "记一个 Andorid 生成文件失败的bug"
date:  2022-04-15 18:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android", "MediaStore"]
---

Android生成文件失败：java.lang.IllegalStateException:Failed to build unique file: /storage/emulated/0/...

## 1.问题来源
App 调用相机拍照，中间有一些处理过程，然后将这张照片插入系统图片数据库中。

```
MediaStore.Images.Media.insertImage(getContentResolver(), photoPath, null, null);
```

很长一段时间，这段代码都运行的好好的，可是突然有一天，测试妹妹报告一个bug，拍照之后，保存图片失败了，并且在 Gallery 中也看不到这张图片。
幸运地是，专业的测试妹妹，抓到了Log，很快找到了错误地方：

```
 E DatabaseUtils: Writing exception to parcel
 E DatabaseUtils: java.lang.IllegalStateException: Failed to build unique file: /storage/emulated/0/Pictures/Image.jpg bucket_display_name=Pictures volume_name=external_primary date_modified=null date_expires=null _display_name=Image.jpg mime_type=image/jpeg _data=/storage/emulated/0/Pictures/Image.jpg _size=null is_trashed=0 is_pending=0 bucket_id=-1617409521 relative_path=Pictures/
 E DatabaseUtils: 	at com.android.providers.media.MediaProvider.ensureFileColumns(MediaProvider.java:2624)
 E DatabaseUtils: 	at com.android.providers.media.MediaProvider.ensureUniqueFileColumns(MediaProvider.java:2368)
 E DatabaseUtils: 	at com.android.providers.media.MediaProvider.updateInternal(MediaProvider.java:5265)
 E DatabaseUtils: 	at com.android.providers.media.MediaProvider.update(MediaProvider.java:4953)
 E DatabaseUtils: 	at android.content.ContentProvider$Transport.update(ContentProvider.java:457)
 E DatabaseUtils: 	at android.content.ContentProviderNative.onTransact(ContentProviderNative.java:230)
 E DatabaseUtils: 	at android.os.Binder.execTransactInternal(Binder.java:1159)
 E DatabaseUtils: 	at android.os.Binder.execTransact(Binder.java:1123)
 W MediaStore: Failed to insert image
 W MediaStore: java.lang.IllegalStateException: Failed to build unique file: /storage/emulated/0/Pictures/Image.jpg bucket_display_name=Pictures volume_name=external_primary date_modified=null date_expires=null _display_name=Image.jpg mime_type=image/jpeg _data=/storage/emulated/0/Pictures/Image.jpg _size=null is_trashed=0 is_pending=0 bucket_id=-1617409521 relative_path=Pictures/
 W MediaStore: 	at android.os.Parcel.createExceptionOrNull(Parcel.java:2381)
 W MediaStore: 	at android.os.Parcel.createException(Parcel.java:2357)
 W MediaStore: 	at android.os.Parcel.readException(Parcel.java:2340)
 W MediaStore: 	at android.database.DatabaseUtils.readExceptionFromParcel(DatabaseUtils.java:190)
 W MediaStore: 	at android.database.DatabaseUtils.readExceptionFromParcel(DatabaseUtils.java:142)
 W MediaStore: 	at android.content.ContentProviderProxy.update(ContentProviderNative.java:649)
 W MediaStore: 	at android.content.ContentResolver.update(ContentResolver.java:2356)
 W MediaStore: 	at android.content.ContentResolver.update(ContentResolver.java:2318)
 W MediaStore: 	at android.provider.MediaStore$Images$Media.insertImage(MediaStore.java:2097)
 W MediaStore: 	at android.provider.MediaStore$Images$Media.insertImage(MediaStore.java:2059)
```

## 2.问题分析
显然这个异常并不是我们 App 中定义的，从日志可以看到程序执行到 `MediaStore.Images.Media.insertImage()` 方法出现异常了，异常是 `MediaProvider.ensureFileColumns` 抛出的，关键字是 `Failed to build unique file` 。
那只能查找源码了。在源码中搜 `MediaProvider` 的 `ensureFileColumns` 方法中,的确看到抛出了上述异常。

![](/assets/images/技术/编程/Android/MediaProvider/pic1.png)

可以看到，这个一样应该是 `FileUtils.buildUniqueFile()` 或者 `FileUtils.buidlNonUniqueFile()` 方法抛出的。查看源码，我们发现是在 `FileUtils.buildUniqueFile()` 调用 `FileUtils.buildUniqueFileWithExtension()` 方法，抛出了异常。

![](/assets/images/技术/编程/Android/MediaProvider/pic2.png)

看下这个方法大致就可以明白，同一名称的文件会被系统在默认添加（1...）等数字用以标识，例如我有一个aa.txt文件，当我要再次生成aa.txt时，系统会帮我生成aa (1).txt文件，再生成则是aa (2).txt。当括号中的名称数量大于32（含32，也就是说同一文件名的数量超过33个时）后就抛异常。

至此，破案了。
我们在调用

```
MediaStore.Images.Media.insertImage(getContentResolver(), photoPath, null, null);
```

方法时，第三个参数，插入数据库的文件的名称，传的 null, 系统会默认生成一个文件名，Image.jpg, 再次插入的则为 Image(1).jpg...

## 3.修改方案
修改方案明了了，只需要再插入系统数据库时，指定文件名。并且要尽量保持唯一。看来，系统接口设计很严谨，接口中每一个参数都是有意义的，比如该第四个参数，description，文件的描述，最好也要赋值。