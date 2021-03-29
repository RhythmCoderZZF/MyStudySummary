# 自定义View事件篇

## 事件传递核心原理8图

### 中途不拦截

1. 都不消费事件

   <img src="pic\image-20210329085653526.png" alt="image-20210329085653526" style="zoom:67%;" />

2. 中途有控件消费所有事件

   <img src="pic\image-20210329085719062.png" alt="image-20210329085719062" style="zoom:67%;" />

3. 中途有控件只消费Down

   <img src="pic\image-20210329085832685.png" alt="image-20210329085832685" style="zoom:67%;" />

### 中途拦截Down

1. 中途有控件拦截但最后没有控件消费

   <img src="pic\image-20210329090147326.png" alt="image-20210329090147326" style="zoom:67%;" />

2. 中途有控件拦截并消费所有事件

   <img src="pic\image-20210329090642614.png" alt="image-20210329090642614" style="zoom:67%;" />

3. 中途有控件拦截只消费了Down

   <img src="pic\image-20210329090804101.png" alt="image-20210329090804101" style="zoom:67%;" />

### 中途拦截Move

1. 拦截并消费Move

   <img src="pic\image-20210329091026143.png" alt="image-20210329091026143" style="zoom:67%;" />

2. 拦截但不消费Move

   <img src="pic\image-20210329091054565.png" alt="image-20210329091054565" style="zoom:67%;" />



**结合开发场景分析事件传递机制**

**A不拦截**

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

  小结：A拦截`Down`事件导致不会遍历child，`mFirstTouchTarget`为空，就轮到A.`onTouchEvent()执行`。之后的`Move`和`Up`事件不会再进入A.`onInterceptTouchEvent`并且是否传给A取决于A.`onTouchEvent`是否消费`Down`事件。

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

  小结：B消费`Down`事件后，之后的`Move、Up`A依旧有机会拦截，一旦拦截就会向B发送一个`Cancel`事件，并清空mFirstTouchTarget。之后的事件就只会传递给A。

------

## requestDisallowInterceptTouchEvent

> 禁止**所有父view**拦截事件

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

1. 只能是后续的`move`、`up`事件——`down`事件父`View`要是拦截了都不会给自己调用该函数的机会。
2. 自身必须消费`down`事件，否则`mFirstTouchTarget = null`。

------

## 滑动冲突

**外部拦截法**

> 所有事件都先经过父`View`的拦截，由父`View`决定是否拦截事件

```java

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

   原因：只要`View`消费`Down`事件就会被添加到`TouchTarget`链，没有消费则该`View`和该`View`的子`View`都不在这条链上，后续`move、up`事件就会根据`TouchTarget`中的`View`来传递，它们的返回值不影响事件传递。

------

## ScrollBy和ScrollTo

> 1. `ScrollBy`和`ScrollTo`都是`跟随手势滑动方向的滚动`，和`Canvas`平移方向相反。
>
> 2. `ScrollTo(100,100)`——`View`内容向右下方移动（100，100）；`ScrollBy(100,100)`——`View`内容**再次**向右下方移动（100，100）
>
>    ```java
>    public void scrollBy(int x, int y) {
>        scrollTo(mScrollX + x, mScrollY + y);
>    }
>    ```

**源码**

```java
//View
boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
      int sx = 0;
      int sy = 0;
      if (!drawingWithRenderNode) {
            computeScroll();//计算mScrollX，mScrollY，invalidate触发
            sx = mScrollX;
            sy = mScrollY;
      }
     canvas.translate(mLeft - sx, mTop - sy);//例如手势滑动向右，scroll值越大，canvas反方向平移越多，内容就向右移动越多
 }
```









