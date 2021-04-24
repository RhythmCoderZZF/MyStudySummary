# 探索ThreadLocal与Handler

## ThreadLocal

> `ThreadLocal`的存在是为了解决——**如何将数据与线程绑定起来，从而该数据只能在绑定的线程里访问，而其它线程无法访问**
>
> 第一种方案：利用一个`Map`保存`Thread`和`Value`的键值对。
>
> 第二种方案：将`Value`直接由每个`Thread`对象保存。显然第二种方案更易于管理。
>
> <img src="pic\image-20210402144749543.png" alt="image-20210402144749543" style="zoom:67%;" />

## Handler

<img src="pic\image-20210406101914254.png" alt="image-20210406101914254" style="zoom:67%;" />



