# Android数据和文件存储

参考：

[Android 存储基础](https://www.jianshu.com/p/93c9f5e2d2a7)

## **应用专属存储空间**

存储仅供应用使用的文件，可以存储到 **内部存储卷中的专属目录 **或 **外部存储空间中的其他专属目录**。使用内部存储空间中的目录保存其他应用不应访问的敏感信息。

早期Android手机的外部存储对应sd卡，如今都是内置到手机里。但依旧存与内部存储区分

### 内部存储空间

路径：`/data/user/0/`(对应`data/data`)。

- 自己访问无需申请权限
- 其他app无权访问
- 卸载删除app对应包目录

#### 目录及API

| 目录       | 路径                              | API                      |
| ---------- | --------------------------------- | ------------------------ |
| 根路径     | /data/user/0/[package]            | context.dataDir          |
| files      | /data/user/0/[package]/files      | context.filesDir         |
| cache      | /data/user/0/[package]/cache      | context.cacheDir         |
| code_cache | /data/user/0/[package]/code_cache | context.codeCacheDir     |
| no_backup  | /data/user/0/[package]/no_backup  | context.noBackupFilesDir |

所有的文件夹都是在package下。

#### 访问文件

1. 使用IO流
2. 使用`context.openFileInput`/`openFileOutput`（该api只对准files目录）

>访问 `res/raw`目录下的文件使用`resource.openRawResource(R.raw.id)`
>
>访问 `assets/` 目录下的文件使用`resources.assets.open(文件路径)`

### App专属外部存储空间

路径：`storage/emulated/0/Android`(对应`sdcard/`)

- 自己访问不需要申请权限
- 其他app申请读写权限就可以访问
- 卸载删除app包目录

#### 目录及API

| 目录       | 路径                                                  | API                                                     |
| ---------- | ----------------------------------------------------- | ------------------------------------------------------- |
| media      | /storage/emulated/0/Android/media/[package]           | context.externalMediaDirs                               |
| cache      | /storage/emulated/0/Android/data/[package]/cache      | context.externalCacheDir                                |
| files/DCIM | /storage/emulated/0/Android/data/[package]/files/DCIM | context.getExternalFilesDir(Environment.DIRECTORY_DCIM) |
| files      | /storage/emulated/0/Android/data/[package]/files      | context.getExternalFilesDir("")                         |

`/storage/emulated/0/Android/data/[package]`下分为`files`/和`cache/`两个文件夹，其中`files/`下可以创建Android提供的多种类型的文件夹，如`DCIM/`，`Pictures/`。

`externalMediaDirs`路径比较特殊

#### 访问文件

IO流

#### 验证空间可用

由于外部存储空间可能是可移除的sd卡，因此读写前需要验证是否可用

```kotlin
//验证可读写
fun isExternalStorageWritable(): Boolean {
    return Environment.getExternalStorageState() == Environment.MEDIA_MOUNTED
}

//验证只读
fun isExternalStorageReadable(): Boolean {
     return Environment.getExternalStorageState() in
        setOf(Environment.MEDIA_MOUNTED, Environment.MEDIA_MOUNTED_READ_ONLY)
}
```

#### 选择存储位置

由于外部存储空间可能不仅有手机内置的，也可能还有sd卡或多个sd卡，因此需要选择哪个外部存储。

```kotlin
val externalStorageVolumes: Array<out File> =
        ContextCompat.getExternalFilesDirs(applicationContext, null)
val primaryExternalStorage = externalStorageVolumes[0]//0表示默认的主外部存储
```



## 共享存储

- 用户需要提供给其他应用访问
- 卸载后也不会清除

#### 媒体

如照片、视频

- **图片**（包括照片和屏幕截图），存储在 `DCIM/` 和 `Pictures/` 目录中。系统将这些文件添加到 [`MediaStore.Images`](https://developer.android.google.cn/reference/android/provider/MediaStore.Images) 表格中。
- **视频**，存储在 `DCIM/`、`Movies/` 和 `Pictures/` 目录中。系统将这些文件添加到 [`MediaStore.Video`](https://developer.android.google.cn/reference/android/provider/MediaStore.Video) 表格中。
- **音频文件**，存储在 `Alarms/`、`Audiobooks/`、`Music/`、`Notifications/`、`Podcasts/` 和 `Ringtones/` 目录中，以及位于 `Music/` 或 `Movies/` 目录中的音频播放列表中。系统将这些文件添加到 [`MediaStore.Audio`](https://developer.android.google.cn/reference/android/provider/MediaStore.Audio) 表格中。
- **下载的文件**，存储在 `Download/` 目录中。在搭载 Android 10（API 级别 29）及更高版本的设备上，这些文件存储在 [`MediaStore.Downloads`](https://developer.android.google.cn/reference/android/provider/MediaStore.Downloads) 表格中。此表格在 Android 9（API 级别 28）及更低版本中不可用。

#### 文档和其他文件

Documents-->存储如.pdf类型等文件