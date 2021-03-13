### [Android应用数据与文件](https://www.jianshu.com/p/a34c644e3431)

参考文章：[Android | 文件存储](https://www.jianshu.com/p/a34c644e3431)

> 早期的Android设备存储空间较小，有一个内置（build-in）的存储空间，即**内部存储**，另外还有一个可以移除的存储介质，即**外部存储**（如SD卡）。但是随着设备内置存储空间增大，很多设备已经足以将**内置存储空间**一分为二，一块为内部存储，一块为外部存储

##### 内部存储

> 1. 内部存储为本应用创建了私有的专属文件存储空间（路径：/data/user/0/[packageName] 目录：/data/data），
> 2. 目录下操作文件不需要添加读写权限
> 3. 目录的空间通常比较小，在将应用专属文件写入内部存储空间之前，应用应[查询设备上的可用空间](https://developer.android.google.cn/training/data-storage/app-specific#query-free-space)。
> 4. 应用卸载时删除

<img src="pic\10107787-df87a04b0ce65e24.png" style="zoom:67%;" />

API

<img src="pic\10107787-e095df1fc4ad5cc0.webp" style="zoom:67%;" />

- ```java
  //Context下
  getFilesDir()://路径：/data/data/[packageName]/files —— files文件
  getCacheDir()://路径：/data/data/[packageName]/cache —— 缓存文件
  getCodeCacheDir()://路径：/data/data/[packageName]/code_cache —— 存放优化过的代码(如JIT优化)
  getNoBackupFilesDir()://路径：/data/data/[packageName]/no_backup —— 在Backup过程中被忽略的文件
  getDir(“myFile”, MODE_PRIVATE)://路径：/data/[packageName] —— packageName下目录，可以自定义文件夹
  
  openFileOutput(String name,int mode)： //对准 /data/data/[packageName]/files 下name文件的文件输出流;mode=MODE_PRIVATE
  openFileInput(String name)：//对准 /data/data/[packageName]/files 下name文件的输入流
  ```

- ```java
  //创建临时文件 
  val file1 = File.createTempFile("myTemp", ".txt", requireContext().externalCacheDir)
  ```


##### 外部存储

> 1. 外部存储根路径及目录——（路径：/storage/emulated/0 目录：/sdcard）
>
> 2. 外部存储也为应用创建了**私有的专属存储空间**
>
>      数据目录：路径：/storage/emulated/0/Android/data/[packageName] 目录：sdcard/Android/data/[packageName]
>
>      媒体路径：路径：/storage/emulated/0/Android/media/[packageName] 目录：sdcard/Android/media/[packageName]。
>
>      > *在搭载 Android 9（API 级别 28）或更低版本的设备上，只要其他应用具有相应的存储权限，任何应用都可以访问外部存储空间中的应用专属文件。为了让用户更好地管理自己的文件并减少混乱，以 Android 10（API 级别 29）及更高版本为目标平台的应用在默认情况下被授予了对外部存储空间的分区访问权限（即[分区存储](https://developer.android.google.cn/training/data-storage#scoped-storage)）。**启用分区存储后，应用将无法访问属于其他应用的应用专属目录即使申请了动态读写权限**（关闭分区存储功能：AndroidManifest文件——android:requestLegacyExternalStorage="true"）。*
>
> 2. **应用读写外部存储私有区不需要读写权限，但是访问其他区域需要申请动态权限**
>
> 3. 应用卸载后外部存储私有区会被删除



API

<img src="pic\10107787-0da9bd54800f5f4e.webp" style="zoom:67%;" />

- ```kotlin
  //外部存储公共区：Environment下————————————————————————————————————————————————————————————————————————————
  getExternalStorageDirectory()://路径：/storage/emulated/0 目录：/sdcard
  getExternalStoragePublicDirectory(String type)// 对应/sdcard下预置分类文件夹,如/Music、/Download
      
  //外部存储私有区：Context下——————————————————————————————————————————————————————————————————————————————————
  File getExternalCacheDir():// /sdcard/Android/data/cache
  File getExternalFilesDir(String type)// /sdcard/Android/data/files，若type文件夹不存在则会创建该文件夹
  
  File[] getExternalFilesDirs(String type);//和getExternalFilesDir在于可能设备由多个sdcard装载，路径的array
  File[] getExternalMediaDirs();// /sdcard/Android/media
  ```
  
- 验证外部存储挂载
  
    （由于外部存储空间位于用户可能能够移除的物理卷上，因此在尝试从外部存储空间读取应用专属数据或将应用专属数据写入外部存储空间之前，请验证该卷是否可访问。）
    
    ```kotlin
    fun isExternalStorageWritable(): Boolean {
          return Environment.getExternalStorageState() == Environment.MEDIA_MOUNTED
      }
    ```
- 选择物理存储位置

  ```kotlin
  val externalStorageVolumes: Array<out File> =
          ContextCompat.getExternalFilesDirs(applicationContext, null)
  val primaryExternalStorage = externalStorageVolumes[0]
  // 有时，设备也会提供 SD 卡插槽。这意味着设备具有多个可能包含外部存储空间的物理卷，因此您需要选择用于应用专属存储空间的物理卷。返回数组中的第一个元素被视为主外部存储卷。除非该卷已满或不可用，否则请使用该卷。
  ```
  
- **外部专属媒体文件**
  
  ```kotlin
  fun getAppSpecificAlbumStorageDir(context: Context, albumName: String): File? {
      // Get the pictures directory that's inside the app-specific directory on
      // external storage.
      val file = File(context.getExternalFilesDir(
              Environment.DIRECTORY_PICTURES), albumName)
      if (!file?.mkdirs()) {
          Log.e(LOG_TAG, "Directory not created")
      }
      return file
  }
  ```
  
  请务必使用 [`DIRECTORY_PICTURES`](https://developer.android.google.cn/reference/android/os/Environment#DIRECTORY_PICTURES) 等 API 常量提供的目录名称。**这些目录名称可确保系统正确处理文件**。如果没有适合您文件的[预定义子目录名称](https://developer.android.google.cn/reference/android/os/Environment#fields)，您可以改为将 `null` 传递到 `getExternalFilesDir()`。这将返回外部存储空间中的应用专属根目录。
  
  
  
  
  
  