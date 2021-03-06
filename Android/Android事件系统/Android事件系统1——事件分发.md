参考：[反思｜Android 事件拦截机制的设计与实现](https://juejin.cn/post/6844904128397705230)

# 自定义View事件篇

Android的事件分发是典型的责任链设计模式。

责任链模式中需要两个角色——`请求`与`节点`，在事件分发机制中 View充当节点，MotionEvent充当请求；

每个节点必须具备两个行为——请求的`处理`与`分发`，而View中的`dispatchTouchEvent`方法扮演着请求的`处理`与`分发`。

## 事件传递核心原理8图

### 不拦截事件

消费：onTouchEvent返回true

1. 不消费任何事件

   <img src="pic\image-20210329085653526.png" alt="image-20210329085653526" style="zoom:67%;" />

2. 消费所有事件

   <img src="pic\image-20210329085719062.png" alt="image-20210329085719062" style="zoom:67%;" />

3. 只消费Down，不消费Move事件

   <img src="pic\image-20210329085832685.png" alt="image-20210329085832685" style="zoom:67%;" />

**小结**：

1. 消费Down后，事件不会回溯

2. 消费Down后，消费不消费Move（Up事件也一样）都不会回溯事件，只影响`Acrivity.onTouchEvent`的接收。

> 这里的回溯特指的是：事件再传递给Parent的onToucnEvent，让Parent去处理事件。实际上程序还会回到Parent去执行，这是递归的特性。

### 中途拦截Down

拦截：onInterceptTouchEvent返回true

1. 拦截Down但不消费

   <img src="pic\image-20210329090147326.png" alt="image-20210329090147326" style="zoom:67%;" />

2. 拦截Down并消费所有事件

   <img src="pic\image-20210329090642614.png" alt="image-20210329090642614" style="zoom:67%;" />

3. 拦截Down且只消费Down

   <img src="pic\image-20210329090804101.png" alt="image-20210329090804101" style="zoom:67%;" />

**小结**：

1. 拦截消费Down后，事件不会回溯

2. 拦截消费Down后，消费不消费Move（Up事件也一样）都不会回溯事件，只影响`Acrivity.onTouchEvent`的接收。

### 中途拦截Move

拦截Move的前提是事件已经被Child给消费了，否则Move都不会经过Parent。

1. 拦截并消费Move

   <img src="pic\image-20210329091026143.png" alt="image-20210329091026143" style="zoom:67%;" />

2. 拦截但不消费Move

   <img src="pic\image-20210329091054565.png" alt="image-20210329091054565" style="zoom:67%;" />

**小结**：

拦截Move是最复杂也是开发中最常见的事件冲突解决方案。

1. 由之前得出的结论，Move是否消费只会影响`Acrivity.onTouchEvent`的接收
2. 拦截Move后，此刻的Move事件会转换成Cancel事件传递给Child通知其——你的事件被拦截了，做好重置工作吧
3. 之后的Move事件只会传递给拦截事件View的`onTouchEvent`。

### 总结

1. 想要去操控一个View，必须在Down事件传递给他的onTouchEvent返回true，表示消费事件。之后的Move事件才能定位到该View，并传递给他。Move事件的消费与否不影响事件传递。
2. Down事件传递过程是树的遍历，它属于“排头兵”，一个个的遍历要消费事件的View，因此十分消耗性能；Move事件传递过程是基于Down事件已经探查出的一条消费事件的View链（TouchTargt），它属于“富二代”，伸手就能要到消费事件的View。
3. Parent拦截掉Down事件会造成Child完全接收不到事件的后果，因此开发中很少去中断Down事件，而是去中断Move事件。

## 结合源码分析事件传递机制

### 中途不拦截

- B不消费`down`事件，及后续`Move`、`Up`事件传递过程

  ```java
  // 🚩Down事件 -> A.dispatchTouchEvent() -> 绕过A.onInterceptTouchEvent() -> 遍历出接收事件的Child->👇 
  if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {//-> B.dispatchTouchEvent()，返回false代表未消费
   }
  
  
    if (mFirstTouchTarget == null) {//mFirstTouchTarget = null,轮到A处理DOWN事件
                  handled = dispatchTransformedTouchEvent(ev, canceled, null,
                          TouchTarget.ALL_POINTER_IDS);//交给A.onTouchEvent()去处理事件，返回值决定handled的值
    } 
  return handled
      
  /* —————————————————————————————————————————————————————————————————————————————————————————————————— */    
      
  // 🚩Move UP事件 -> A.dispatchTouchEvent() -> 绕过A.onInterceptTouchEvent() -> 跳过遍历（只有Down才遍历）->👇
  if (mFirstTouchTarget == null) {//mFirstTouchTarget = null,轮到A处理Move、Up事件
                  handled = dispatchTransformedTouchEvent(ev, canceled, null,
                          TouchTarget.ALL_POINTER_IDS);//交给A.onTouchEvent()去处理事件，返回值决定handled的值
    }
  return handled
  ```

  小结：B未消费`Down`事件，`mFirstTouchTarget`为空，就轮到A.`onTouchEvent()执行`，之后的`Move`和`Up`事件是否传给A取决于A.`onTouchEvent`是否消费`Down`事件。

- 若B消费`down`事件，及后续`move、up`事件传递过程

  ```java
  // 🚩Down事件 -> A.dispatchTouchEvent() -> 绕过A.onInterceptTouchEvent() -> 遍历出接收事件的Child ->👇 
  if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {//调用 B.dispatchTouchEvent()，返回true代表消费
              
              newTouchTarget = addTouchTarget(child, idBitsToAssign);//newTouchTarget=mFirstTouchTarget 被初始化(TouchTarget中有一个属性用于保存接收事件的Child(这里是B)，并且TouchTarget使用next链式连接)
              alreadyDispatchedToNewTouchTarget = true;
              break;
   }
  
  
  
    if (mFirstTouchTarget == null) {//❌mFirstTouchTarget ≠ null,所以轮不到A处理DOWN事件
                  handled = dispatchTransformedTouchEvent(ev, canceled, null,
                          TouchTarget.ALL_POINTER_IDS);
    } else {
         TouchTarget target = mFirstTouchTarget;
         if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
             handled = true;//🚩
   }
  return handled
  /* —————————————————————————————————————————————————————————————————————————————————————————————————— */    
      
  // 🚩Move UP事件 -> A.dispatchTouchEvent() -> 绕过A.onInterceptTouchEvent() -> 跳过遍历（只有Down才遍历）->👇
  if (mFirstTouchTarget == null) {//❌mFirstTouchTarget ≠ null,所以轮不到A处理Move、Up事件
                  handled = dispatchTransformedTouchEvent(ev, canceled, null,
                          TouchTarget.ALL_POINTER_IDS);
    } else {
       TouchTarget predecessor = null;
       TouchTarget target = mFirstTouchTarget;//mFirstTouchTarget保存了接收事件的child B
       while (target != null) {
            final TouchTarget next = target.next;//取出child B，将事件传给B
                if (dispatchTransformedTouchEvent(ev, cancelChild, target.child, target.pointerIdBits)) {
                    handled = true;
                }
              predecessor = target;
              target = next;
      }
   }
  return handled
  ```

  小结：

  > B消费`Down`事件后，`mFirstTouchTarget`被初始化并保存B，轮不到A.`onTouchEvent()执行`，
  >
  > 之后的`Move`和`Up`事件直接通过`mFirstTouchTarget`传递给B。（即使B未消费`Move`、`Up`事件，事件也会分发给B）


### 中途拦截Down

**A要拦截事件**

- A一开始就拦截`Down`事件，及后续`Move、Up`事件传递过程

  ```java
  // 🚩Down事件 -> A.dispatchTouchEvent() ->👇
  final boolean intercepted;
  if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
     final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;//disallowIntercept = false
     if (!disallowIntercept) {
         intercepted = onInterceptTouchEvent(ev);//A.onInterceptTouchEvent =>true，导致之后的down无法分发。
      } else {
          intercepted = false;
       }
  } else {
    intercepted = true;
  }
  if (!canceled && !intercepted) {//intercepted=true导致无法遍历child}
  
  /* —————————————————————————————————————————————————————————————————————————————————————————————————— */ 
      
  // 🚩Move、Up事件 -> A.dispatchTouchEvent() ->👇
  final boolean intercepted;
  if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {//Move、Up不会再走A.onInterceptTouchEvent方法
  } else {
    intercepted = true;
  }
  ```

  小结：A拦截`Down`事件导致不会遍历child，`mFirstTouchTarget`为空，就轮到A.`onTouchEvent()执行`。之后的`Move`和`Up`事件也不会再进入A.`onInterceptTouchEvent`。

### 中途拦截Move

- A开始不拦截`Down`，B消费`Down`事件，之后A再拦截`Move`、`Up`事件

  ```java
  // 🚩Move、Up事件 -> A.dispatchTouchEvent() ->👇
  final boolean intercepted;
  if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {//mFirstTouchTarget已经保存了B
     final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;//disallowIntercept = false
     if (!disallowIntercept) {
         intercepted = onInterceptTouchEvent(ev);//A.onInterceptTouchEvent =>true。
      } else {
          intercepted = false;
       }
  } else {
    intercepted = true;
  }
  
  alreadyDispatchedToNewTouchTarget=false
  if (mFirstTouchTarget == null) {//❌
                  handled = dispatchTransformedTouchEvent(ev, canceled, null,
                          TouchTarget.ALL_POINTER_IDS);
              } else {
                  TouchTarget predecessor = null;
                  TouchTarget target = mFirstTouchTarget;
                  while (target != null) {
                      final TouchTarget next = target.next;
                      if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {//❌
                          handled = true;
                      } else {
                          final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                  || intercepted;//intercepted=true
                          //1. cancelChild=true会导致A向B发送一个CANCEL事件，本来可是一个Move事件
                          if (dispatchTransformedTouchEvent(ev, cancelChild,
                                  target.child, target.pointerIdBits)) {
                              handled = true;
                          }
                          //2. mFirstTouchTarget及整条事件链清空！
                          if (cancelChild) {
                              if (predecessor == null) {
                                  mFirstTouchTarget = next;
                              } else {
                                  predecessor.next = next;
                              }
                              target.recycle();
                              target = next;
                              continue;
                          }
                      }
                      predecessor = target;
                      target = next;
                  }
              }
  ```

  小结：B消费`Down`事件后，之后的`Move、Up`A依旧有机会拦截，一旦拦截就会向B发送一个`Cancel`事件，并清空`mFirstTouchTarget`。之后的事件就只会传递给A。

------

## requestDisallowInterceptTouchEvent

> 子View来调用该函数，禁止**所有父view**拦截`Move`、`Up`事件。

```java
if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {//关键的条件
   final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
   if (!disallowIntercept) {
       intercepted = onInterceptTouchEvent(ev);
    } else {
        intercepted = false;
     }
} else {
  intercepted = true;
}
```

该函数生效的条件：

1. 只能是后续的`move`、`up`事件——`down`事件父`View`要是拦截了事件传不到`子View`，子View就没机会调该函数了。
2. 子View必须消费`down`事件，才能使`mFirstTouchTarget != null`。

------

## 解决滑动冲突两种方法

**外部拦截法**

> Move事件得经过`父View`的拦截监控，一旦符合`父View`的逻辑，`父View`就把事件拦截下来。决定权在`父View`的手里

- 父View

```java
override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        var intercept = false
        when (ev.action) {
            MotionEvent.ACTION_MOVE -> {
                if ("我要拦截啦") {
                    intercept = true 
                }
            }
        }
        return intercept
    }

 override fun onTouchEvent(ev: MotionEvent): Boolean {
        when (ev.action) {
            MotionEvent.ACTION_DOWN -> {
                return true
            }
            MotionEvent.ACTION_MOVE -> {
              //此处为拦截后开始我的事件逻辑
            }
        }
        return super.onTouchEvent(ev)
    }
```

**内部拦截法**

> Move事件一开始就被Parent拦下，但是`子View`在`Down`事件就得让`父View`的拦截失效(`requestDisallowInterceptTouchEvent(true)`)。一旦不符合`子View`的事件逻辑，`子View`就重新让`父View`去拦截事件。决定权在`子View`的手里

- 子View

```java
override fun dispatchTouchEvent(ev: MotionEvent): Boolean {
        when (ev.action) {
            MotionEvent.ACTION_DOWN -> {
                //父View拦截失效
                parent.requestDisallowInterceptTouchEvent(true) 
            }
            MotionEvent.ACTION_MOVE -> {
                 //一旦不符合自己的事件逻辑就交给父View去处理
                 if ("还是交给父View处理吧") {
                     parent.requestDisallowInterceptTouchEvent(false) 
                }
            }
        }
        return super.dispatchTouchEvent(ev)
    }

    override fun onTouchEvent(ev: MotionEvent): Boolean {
        when (ev.action) {
            MotionEvent.ACTION_DOWN -> {
                return true
            }
            MotionEvent.ACTION_MOVE -> {
                 //子View的事件逻辑
            }
        }
        return super.onTouchEvent(ev)
    }
```

- 父View

```java
override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        //一开始就拦截move、up事件 
        return ev.action != MotionEvent.ACTION_DOWN  
    }

    override fun onTouchEvent(ev: MotionEvent): Boolean {
        when (ev.action) {
            MotionEvent.ACTION_DOWN -> {
                return true
            }
            MotionEvent.ACTION_MOVE -> {
              //父View的事件逻辑处理
            }
        }
        return super.onTouchEvent(ev)
    }
```

------

## 注意点

1. 设置`setOnTouchLisener()`弹出`performClick()`警告？
   原因在于`onTouchListener`会覆盖`onTouchEvent()`，导致View的点击事件没法处理（点击事件逻辑在`onTouchEvent.ACTION_UP中`）

   ```java
   button.setOnTouchListener(new View.OnTouchListener() {
               @Override
               public boolean onTouch(View v, MotionEvent event) {
                   switch (event.getAction()){
                       case MotionEvent.ACTION_DOWN:
                           break;
                       case MotionEvent.ACTION_MOVE:
                           break;
                       case MotionEvent.ACTION_UP:
                           button.performClick();//加上点击事件
                           break;
                   }
                   return false;
               }
           });
   ```

2. 重写`onTouchEvent`中该如何返回true和false？

   ```kotlin
    override fun onTouchEvent(event: MotionEvent): Boolean {
           when (event.action) {
               MotionEvent.ACTION_DOWN -> {
                   return true//若View想要处理事件ACTION_DOWN一定要返回true
               }
               MotionEvent.ACTION_UP -> {
               }
               MotionEvent.ACTION_MOVE -> {
                 // 其他事件返回true和false都不影响事件传递给该View
               }
           }
           return super.onTouchEvent(event)
       }
   ```

   原因：只要`View`消费`Down`事件就会被添加到`TouchTarget`链。后续`move、up`事件就会根据`TouchTarget`中的`View`来传递，它们的返回值不影响事件传递，只会影响Activity.onTouchEvent执行

------

## scroll

> 平移View的6种方式：https://www.cnblogs.com/fuly550871915/p/4985053.html

scroll和translate的区别：

- `translate`：是画布的平移。

- `scroll`：内容（还是画布）滚动。本质上scroll也是canvas的平移，只不过**应用场景不同**，一般用于手指滑动场景。

scrollBy和scrollTo的区别：

- `ScrollBy`和`ScrollTo`都是内容滚动。

- `scrollTo`相对于`View`初始位置滚动；`scrollBy`相对于最新位置滚动。

**和translate方向相反问题**：

直接上源码：

```java
//View.java
boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
      int sx = 0;
      int sy = 0;
      if (!drawingWithRenderNode) {
            computeScroll();//计算mScrollX，mScrollY，invalidate触发
            sx = mScrollX;
            sy = mScrollY;
      }
     canvas.translate(mLeft - sx, mTop - sy);//scroll为正值，canvas向反方向平移
 }
```







