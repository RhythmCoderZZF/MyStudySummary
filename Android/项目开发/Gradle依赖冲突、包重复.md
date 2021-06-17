# Gradle依赖冲突、包重复

## Support库与AndroidX冲突

命令：.\gradlew :app:dependencies

<img src="_pic\image-20210615101631697.png" alt="image-20210615101631697" style="zoom:80%;" />

发现第三方库使用的是support库，而项目使用的是AndroidX出现冲突。

解决方式：

```bash
android.useAndroidX=true
android.enableJetifier=false//表示将依赖包也迁移到androidx
```

<img src="_pic\image-20210615102338491.png" alt="image-20210615102338491" style="zoom:80%;" />

此时support库变成了AndroidX。

## 重复依赖

<img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Android\项目开发\_pic\image-20210615104352978.png" alt="image-20210615104352978" style="zoom:80%;" />

有的时候重复依赖并不会出现编译错误，如果有必要，则需要删除重复依赖

同样使用命令查看是什么库依赖了不同版本，之后在某个库中剔除一个版本：

```groovy
implementation ("androidx.navigation:navigation-ui-ktx:$nav_version"){
    exclude group: "androidx.transition"
}
//h
implementation ("androidx.navigation:navigation-ui-ktx:$nav_version"){
    exclude module:"transition"
}
```

