# targetSdkVersion

参考：

[targetSdkVersion、compileSdkVersion、minSdkVersion作用与区别](https://www.jianshu.com/p/12e42558378a)

## 案例

```java
//Android 27 O
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();

    // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
    // generally not allowed, except if the caller specifies the task id the activity should
    // be launched in.
    if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0
            && options != null && ActivityOptions.fromBundle(options).getLaunchTaskId() == -1) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity "
                + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                + " Is this really what you want?");
    }
   ...
}
```

注释

> 从外部启动一个Activity，没有设置Flag，又没有指定task id会crash。

```java
//Android 28 P
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();

    // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
    // generally not allowed, except if the caller specifies the task id the activity should
    // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
    // maintain this for backwards compatibility.
    final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

    if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
            && (targetSdkVersion < Build.VERSION_CODES.N
                    || targetSdkVersion >= Build.VERSION_CODES.P)
            && (options == null
                    || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity "
                        + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                        + " Is this really what you want?");
    }
   ...
}
```

注释：

> 从外部启动一个Activity，没有设置Flag，又没有指定task id，并且targetSdk版本不在N~O之间才会crash。

意思就是从Android P版本开始，又重新为targetSdkVersoin在N~O之间开放了的这种错误的打开Activity方式。

## 总结

不同版本的Android手机会根据你App指定targetetSdkVersion来进行适配。比如Android出了新版本，新版本对老版本的API做了修改，最后产生的结果和老版本不一样，这个时候新版本会根据targetSdkVersion来做兼容，即：targetSdkVersion为老版本的还是用老的api，新版本的会用新的Api。

如果我们盲目升级了targetetSdkVersion，轻则新版本和老版本显示效果就会不同，重则发生crash。升级时必须做好兼容性测试。