# 四大组件之ContentProvider

## [URI与URL](https://blog.csdn.net/qq_32595453/article/details/80563142)

> URI：统一资源标志符。
>
> URL：统一资源定位符。URI的子集，主要用在各种WWW客户程序和服务器程序上。
>
> **只要能唯一标识资源的就是URI，在URI的基础上给出其资源的访问方式的就是URL**

## 概述

> 使用应用A的 `Context` 中的 `ContentResolver` 对象与B应用的`ContentProvider`进行通信。`ContentResolver`充当调用者；`ContentProvider`充当数据的提供者，两者之间实现Binder通信

<img src="pic\image-20210311153309348.png" alt="image-20210311153309348" style="zoom: 67%;" />



##  内容URI

> 要查询内容提供者，你需要以如下格式的URI的形式来指定查询字符串
>
> <img src="pic\image-20210310224125099.png" alt="image-20210310224125099" style="zoom:67%;" />
>
> ```java
> ContentUris.withAppendedId(UserDictionary.Words.CONTENT_URI, 4)//content://user_dictionary/words/4
> ```
>
> 

## 实现ContentProvider步骤

1. `Manifest`注册组件并设置`Authority`

   ```xml
   <provider
         android:name=".xxx.MyContentProvider"
         android:authorities="${applicationId}.mycontentporvider"
         android:exported="true"/>
   ```
   
2. 继承`ContentProvider`并注册合法路径；初始化DBHelper

   ```kotlin
   class MyContentProvider : ContentProvider() {
       private val mUriMatcher = UriMatcher(UriMatcher.NO_MATCH)
       private lateinit var mDBHelper: MyDatabaseHelper
       companion object {
           const val PATH_STUDENT = "Student"  
           const val CODE_STUDENT = 1
           const val PATH_STUDENT_ID = "Student/#" //*：匹配由任意长度的任何有效字符组成的字符串；#：匹配由任意长度的数字字符组成的字符串
           const val CODE_STUDENT_ID = 2
       }
   
       override fun onCreate(): Boolean {
           mDBHelper = MyDatabaseHelper(context)
           context?.apply {
               val authority = "$packageName.mycontentporvider"
               mUriMatcher.addURI(authority, PATH_STUDENT, CODE_STUDENT)//注册合法URI - CODE的映射关系
               mUriMatcher.addURI(authority, PATH_STUDENT_ID, CODE_STUDENT_ID)
           }
           return true
       }
   }
   ```

3. 实现增删改查

   ```kotlin
    override fun query(uri: Uri, projection: Array<out String>?, selection: String?, selectionArgs: Array<out String>?, sortOrder: String?): Cursor? {
           val db = mDBHelper.writableDatabase
           when (mUriMatcher.match(uri)) {//匹配注册的合法URI对应的CODE
               CODE_STUDENT -> {
                   return db.query("Student", projection, selection, selectionArgs, null, null, sortOrder)//2.查询
               }
               CODE_STUDENT_ID -> {
                   val id = ContentUris.parseId(uri)//解析出path中的id
                   return db.query("Student", projection, "id=?", arrayOf("$id"), null, null, sortOrder)
               }
           }
           return null
       }
   ```

   

   
## 使用ContentResolver步骤

1. 定义合法URI指向ContentProvider数据库表

   ```kotlin
   val uri = Uri.parse("content://[applicationId].mycontentporvider/Student")
   uri1 = ContentUris.withAppendedId(uri, 1)//可追加id
   ```

2. 调用`ContentResolver`对`ContentProvider`进行操作

   ```kotlin
    val cursor = contentResolver.query(uri, null, null, null, null)
               val listRes = mutableListOf<String>()
               when (cursor?.count) {
                   0 -> {
                       Toast.makeText(this, "没有数据", Toast.LENGTH_SHORT).show()
                   }
                   null -> {
                       Toast.makeText(this, "错误❌", Toast.LENGTH_SHORT).show()
                   }
                   else -> {
                       while (cursor.moveToNext()) {
                           val name = cursor.getString(cursor.getColumnIndex("name"))
                           val age = cursor.getInt(cursor.getColumnIndex("age"))
                       }
                   }
     }
    cursor?.close()
   ```

   

   

   

   

   

   

   









