# RecyclerView基本使用

参考：

启舰——《自定义控件高级进阶与精彩实例》

## DiffUtil

> 用`Myers差分算法`对`oldList`和`newList`的数据进行差异比较，实现`RecyclerView`的增量更新。

**DiffUtil.ItemCallback：**

> 提供`Item`是否相同的业务逻辑，差分算法依此来区分新旧`Item`是否相同

 ```kotlin
   fun areItemsTheSame(oldItem, newItem):Boolean
  //两个Item是否相同（一般用id区分，类似hash值的效果，但切记不要真用hash值，因为会对getChangePayload产生影响...）
   
   fun areContentsTheSame(oldItem, newItem):Boolean
  //两个Item中的字段是否相同
  
   fun getChangePayload(oldItem, newItem): Any? 
  //将两个Item不相同的属性值封装成map返回（一般用Bundle）
 ```

  **注意：**`getChangePayload`方法只会在`areItemsTheSame`返回true，`areContentsTheSame`返回false才会触发（这就是为什么areItemsTheSame不能用hash）。并且，返回的`Any`会被封装到

  ```kotlin
  fun onBindViewHolder(holder: BaseViewHolder<ItemBaseBinding>, position: Int, payloads: MutableList<Any>)
  //若getChangePayload()返回null（默认就是null），则参数payload不等于null且size=0
  ```

  方法的参数`payloads`中，最重要的一点是**getChangePayload()函数一旦返回非空对象，RecyclerView刷新Item将不会再重新构造一个ViewHolder，新旧的ViewHolder会是同一个对象！这对执行Item动画提供基础**

------

## ItemDecoration

> An ItemDecoration allows the application to add a **special drawing** and layout **offset** to specific item views from the adapter's data set. This can be useful for drawing dividers between items, highlights, visual grouping boundaries and more.
>
> 译：ItemDecoration 用于装饰item和对item布局。

### 概述

```kotlin
public abstract static class ItemDecoration {
   	//1.在item onDraw之前调用
    public void onDraw(@NonNull Canvas c, @NonNull RecyclerView parent, @NonNull State state) {
        onDraw(c, parent);
    }
    //2.在item onDraw之后调用（及绘制会覆盖item）
    public void onDrawOver(@NonNull Canvas c, @NonNull RecyclerView parent,
            @NonNull State state) {
        onDrawOver(c, parent);
    }
    //3.将item限制在一个outRect大小的矩形中，outRect的坐标相当于margin。该方法在item显示到屏幕时调用。
    public void getItemOffsets(@NonNull Rect outRect, @NonNull View view,
            @NonNull RecyclerView parent, @NonNull State state) {
        getItemOffsets(outRect, ((LayoutParams) view.getLayoutParams()).getViewLayoutPosition(),
                parent);
    }
}
```

### 自定义

```kotlin
inner class MyLinearItemDecoration : RecyclerView.ItemDecoration() {
    private val mPaint = Paint()
    private lateinit var mBitmap: Bitmap

    init {
        mBitmap = decodeBitmapFromBitmap(R.mipmap.ic_launcher_round, 80, 80)
    }
    
    override fun getItemOffsets(
        outRect: Rect,
        view: View,
        parent: RecyclerView,
        state: RecyclerView.State
    ) {
        outRect.set(200, 5, 20, 5)
    }

    override fun onDrawOver(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        //绘制图标
        for (index in 0 until parent.childCount step 3) {
            val view = parent.getChildAt(index)
            val top=view.top
            val left =view.left - mBitmap.width / 2
            c.drawBitmap(mBitmap,left.toFloat(),top.toFloat(),mPaint)

        }
        val gradient = LinearGradient(
            parent.width / 2f,
            0f,
            parent.width / 2f,
            200f,
            Color.argb(125, 0, 0, 0),
            Color.TRANSPARENT,
            Shader.TileMode.CLAMP
        )
        //绘制阴影
        mPaint.shader = gradient
        c.drawRect(0f,0f,parent.width.toFloat(),parent.height.toFloat(),mPaint)
    }
}
```

### 源码

`getItemOffsets()`被调用的地方：

```java
Rect getItemDecorInsetsForChild(View child) {
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
   
    final Rect insets = lp.mDecorInsets;
    insets.set(0, 0, 0, 0);
    final int decorCount = mItemDecorations.size();
    for (int i = 0; i < decorCount; i++) {
        mTempRect.set(0, 0, 0, 0);
        mItemDecorations.get(i).getItemOffsets(mTempRect, child, this, mState);
        insets.left += mTempRect.left;
        insets.top += mTempRect.top;
        insets.right += mTempRect.right;
        insets.bottom += mTempRect.bottom;
    }
    lp.mInsetsDirty = false;
    return insets;
}
```

该矩形被添加到每一个child的LayoutParams中保存，变量为`mDecorInsets`

看一下child如何响应矩形区域的限制：

```java
public void layoutDecorated(@NonNull View child, int left, int top, int right, int bottom) {
    final Rect insets = ((LayoutParams) child.getLayoutParams()).mDecorInsets;
    child.layout(left + insets.left, top + insets.top, right - insets.right,
            bottom - insets.bottom);
}
```

在LayoutManager中，当对child布局时，会加上矩形的限制。矩形的四个参数相当于child的margin值

  ## LayoutManager

> A LayoutManager is responsible for **measuring and positioning item views** within a RecyclerView as well as **determining the policy for when to recycle item views that are no longer visible to the user**
>
> 译：LayoutManager有两个作用——1.对item的测量、布局；2.对不再显示到屏幕的item进行处理

### 无回收复用机制

```kotlin
inner class MyLayoutManager : RecyclerView.LayoutManager() {
    private var mTotalHeight = 0//item总高度，当item不足以填满recyclerView时为recyclerView的高度
    private var mSumDy = 0//记录dy的累加量，有上下界限制[0 ~ itemHeights-recyclerView.height]

    //1.为child设置layoutParams
    override fun generateDefaultLayoutParams() = RecyclerView.LayoutParams(
        ViewGroup.LayoutParams.WRAP_CONTENT,
        ViewGroup.LayoutParams.WRAP_CONTENT
    )

    //2.该方法用于对children进行measure、layout（进调用一次）
    override fun onLayoutChildren(recycler: RecyclerView.Recycler, state: RecyclerView.State?) {
        super.onLayoutChildren(recycler, state)
        var offsetY = 0
        repeat(itemCount) {
            val itemView = recycler.getViewForPosition(it)
            measureChildWithMargins(itemView, 0, 0)
            addView(itemView)
            val childWidth = getDecoratedMeasuredWidth(itemView)
            val childHeight = getDecoratedMeasuredHeight(itemView)
            layoutDecorated(itemView, 0, offsetY, childWidth, childHeight + offsetY)
            offsetY += childHeight
        }
        mTotalHeight = Math.max(offsetY, getVerticalSpace())
    }
    //计算recyclerView内容区的高度
    private fun getVerticalSpace() = height - paddingTop - paddingBottom

    //3.是否允许垂直滚动，只有返回true，scrollVerticallyBy函数才会在RecyclerView发生scroll时被调用。
    override fun canScrollVertically() = true

    //4.该方法由recyclerView move事件、fling事件触发，参数dy为触摸差值
    override fun scrollVerticallyBy(
        dy: Int,
        recycler: RecyclerView.Recycler?,
        state: RecyclerView.State?
    ): Int {
        var travel = dy
        if (mSumDy + dy <= 0) {//滑到顶部处理
            travel = -mSumDy//修正travel
        } else if (mSumDy + dy >= mTotalHeight - getVerticalSpace()) {//滑倒底部处理
            travel = mTotalHeight - getVerticalSpace() - mSumDy//修正travel
        }
        
        mSumDy += travel
        //对child进行offsetTopAndBottom
        offsetChildrenVertical(-travel)
        return dy
    }
}
```

  在`onLayoutChildren`函数中，对children进行统一的测量、布局。当RecyclerView接收到Touch事件时，触发`scrollVerticallyBy|scrollHorizontallyBy`函数，从而对child进行偏移滑动。

### 单向回收复用

```kotlin
inner class MyLayoutManager : RecyclerView.LayoutManager() {
    private var mTotalHeight = 0//item总高度，当item不足以填满recyclerView时为recyclerView的高度
    private var mSumDy = 0//记录dy的累加量，有上下界限制[0 ~ itemHeights-recyclerView.height]

    private var mItemWidth = 0//每个item的宽，由于这里只有一个单一布局，所以对一个item的测量结果代表了所有item
    private var mItemHeight = 0
    private var mItemRects = SparseArray<Rect>()//记录每一个item在RecyclerView中的坐标

    
    override fun generateDefaultLayoutParams() = RecyclerView.LayoutParams(
        ViewGroup.LayoutParams.WRAP_CONTENT,
        ViewGroup.LayoutParams.WRAP_CONTENT
    )

    
    override fun onLayoutChildren(recycler: RecyclerView.Recycler, state: RecyclerView.State?) {
        super.onLayoutChildren(recycler, state)
        //1.剥离所有child
        detachAndScrapAttachedViews(recycler)
        //没有child就不显示
        if (itemCount == 0) {
            return
        }

        //2.根据第一个child 获取item宽、高、一屏可见的child个数
        val view = recycler.getViewForPosition(0)
        measureChildWithMargins(view, 0, 0)
        mItemWidth = getDecoratedMeasuredWidth(view)
        mItemHeight = getDecoratedMeasuredHeight(view)
        val visibleCount = if (getVerticalSpace() % mItemHeight == 0) {
            getVerticalSpace() / mItemHeight
        } else {
            getVerticalSpace() / mItemHeight + 1
        }

        //3.将所有child的宽高保存起来
        var offsetY = 0
        repeat(itemCount) {
            val rect = Rect(0, offsetY, mItemWidth, offsetY + mItemHeight)
            mItemRects.put(it, rect)
            offsetY += mItemHeight
        }

        //4.⭐只为可见的child进行布局
        repeat(visibleCount) {
            val rect = mItemRects.get(it)
            val view = recycler.getViewForPosition(it)
            measureChildWithMargins(view, 0, 0)
            addView(view)
            layoutDecorated(view, rect.left, rect.top, rect.right, rect.bottom)
        }
        mTotalHeight = Math.max(offsetY, getVerticalSpace())
    }

    private fun getVerticalSpace() = height - paddingTop - paddingBottom

    override fun canScrollVertically() = true

    override fun scrollVerticallyBy(
        dy: Int, //注意：dy>0时 列表是上滑
        recycler: RecyclerView.Recycler,
        state: RecyclerView.State?
    ): Int {
        var travel = dy
        if (mSumDy + dy <= 0) {
            travel = -mSumDy
        } else if (mSumDy + dy >= mTotalHeight - getVerticalSpace()) {
            travel = mTotalHeight - getVerticalSpace() - mSumDy
        }

        //5.⭐对当前可见的child进行遍历，预判child经过travel的平移之后是否屏幕不可见，依此来进行[回收]
        for (i in childCount - 1 downTo 0) {//childCount、getChild获取的都是当前屏幕可见的child
            val view = getChildAt(i)!!
            if (travel > 0) {//上滑
                //判断child是否会滑出屏幕上边界，滑出则需要回收
                if (getDecoratedBottom(view) - travel < 0) {
                    //6.⭐[回收]
                    removeAndRecycleView(view, recycler)
                    continue
                }
            }
        }

        //7.⭐对不可见的child进行遍历，预判child经过travel平移之后是否屏幕可见，依此来进行[复用]
        if (travel >= 0) {
            val lastView = getChildAt(childCount - 1)!!
            val minPos = getPosition(lastView) + 1
            for (i in minPos until itemCount) {
                val rect = mItemRects.get(i)
                //当item所占的rect即将在屏幕显示区域，则要添加该item并对其测量布局
                if (Rect.intersects(rect, getVisibleArea(travel))) {
                    //8.⭐从缓存中获取一个child[复用]
                    val child = recycler.getViewForPosition(i)
                    addView(child)
                    measureChildWithMargins(child, 0, 0)
                    layoutDecorated(
                        child,
                        rect.left,
                        rect.top - mSumDy,
                        rect.right,
                        rect.bottom - mSumDy
                    )
                } else {
                    break
                }
            }
        }
        mSumDy += travel
        //对child进行offsetTopAndBottom
        offsetChildrenVertical(-travel)
        return travel
    }

    private fun getVisibleArea(travel: Int) = Rect(
        paddingLeft,
        paddingTop + mSumDy + travel,
        width+paddingRight,
        getVerticalSpace() + mSumDy + travel
    )
}
```

### 双向回收复用

```kotlin
inner class MyLayoutManager : RecyclerView.LayoutManager() {
    private var mTotalHeight = 0
    private var mSumDy = 0

    private var mItemWidth = 0
    private var mItemHeight = 0
    private var mItemRects = SparseArray<Rect>()

    
    override fun generateDefaultLayoutParams() = RecyclerView.LayoutParams(
        ViewGroup.LayoutParams.WRAP_CONTENT,
        ViewGroup.LayoutParams.WRAP_CONTENT
    )

    
    override fun onLayoutChildren(recycler: RecyclerView.Recycler, state: RecyclerView.State?) {
        super.onLayoutChildren(recycler, state)
        detachAndScrapAttachedViews(recycler)
        if (itemCount == 0) {
            return
        }

        //根据第一个child 获取item宽高、一屏可见的child个数
        val view = recycler.getViewForPosition(0)
        measureChildWithMargins(view, 0, 0)
        mItemWidth = getDecoratedMeasuredWidth(view)
        mItemHeight = getDecoratedMeasuredHeight(view)

        val visibleCount = if (getVerticalSpace() % mItemHeight == 0) {
            getVerticalSpace() / mItemHeight
        } else {
            getVerticalSpace() / mItemHeight + 1
        }

        //将所有child的宽高保存起来
        var offsetY = 0
        repeat(itemCount) {
            val rect = Rect(0, offsetY, mItemWidth, offsetY + mItemHeight)
            mItemRects.put(it, rect)
            offsetY += mItemHeight
        }

        //只为可见的child进行布局
        repeat(visibleCount) {
            val rect = mItemRects.get(it)
            val view = recycler.getViewForPosition(it)
            measureChildWithMargins(view, 0, 0)
            addView(view)
            layoutDecorated(view, rect.left, rect.top, rect.right, rect.bottom)
        }
        mTotalHeight = Math.max(offsetY, getVerticalSpace())
    }

    private fun getVerticalSpace() = height - paddingTop - paddingBottom

    override fun canScrollVertically() = true

    override fun scrollVerticallyBy(
        dy: Int,//dy>0时 列表是上滑
        recycler: RecyclerView.Recycler,
        state: RecyclerView.State?
    ): Int {
        var travel = dy
        if (mSumDy + dy <= 0) {//滑到顶部处理
            travel = -mSumDy//矫正travel
        } else if (mSumDy + dy >= mTotalHeight - getVerticalSpace()) {//滑倒底部处理
            travel = mTotalHeight - getVerticalSpace() - mSumDy
        }


        //对可见的child进行遍历，预判child经过travel的平移之后是否屏幕不可见，依此来进行[回收]
        for (i in childCount - 1 downTo 0) {
            val view = getChildAt(i)!!
            if (travel > 0) {//上滑，判断child是否会滑出屏幕上边界，滑出则需要回收
                if (getDecoratedBottom(view) - travel < 0) {
                    removeAndRecycleView(view, recycler)//回收
                    continue
                }
            } else if (travel < 0) {//下滑，判断child是否会滑出屏幕下边界，滑出则需要回收
                if (getDecoratedTop(view) - travel > height - paddingBottom) {
                    removeAndRecycleView(view, recycler)
                    continue
                }
            }
        }

        //对不可见的child进行遍历，预判child经过travel的平移之后是否屏幕可见，依此来进行[复用]
        if (travel >= 0) {
            val lastView = getChildAt(childCount - 1)!!
            val minPos = getPosition(lastView) + 1
            for (i in minPos until itemCount) {
                val rect = mItemRects.get(i)
                if (Rect.intersects(rect, getVisibleArea(travel))) {
                    val child = recycler.getViewForPosition(i)
                    addView(child)
                    measureChildWithMargins(child, 0, 0)
                    layoutDecorated(
                        child,
                        rect.left,
                        rect.top - mSumDy,
                        rect.right,
                        rect.bottom - mSumDy
                    )
                } else {
                    break
                }
            }
        } else {
            val firstView = getChildAt(0)!!
            val maxPos = getPosition(firstView) - 1
            for (i in maxPos downTo 0) {
                val rect = mItemRects.get(i)
                if (Rect.intersects(getVisibleArea(travel), rect)) {
                    val child = recycler.getViewForPosition(i)
                    addView(child, 0)
                    measureChildWithMargins(child, 0, 0)
                    layoutDecorated(
                        child,
                        rect.left,
                        rect.top - mSumDy,
                        rect.right,
                        rect.bottom - mSumDy
                    )
                } else {
                    break
                }
            }
        }
        mSumDy += travel
        //对child进行offsetTopAndBottom
        offsetChildrenVertical(-travel)
        return travel
    }

    /**
     * 获取下一帧可见区域（）
     */
    private fun getVisibleArea(travel: Int) = Rect(
        paddingLeft,
        paddingTop + mSumDy + travel,
        width + paddingRight,
        getVerticalSpace() + mSumDy + travel
    )
}
```

### 双向回收复用2

```kotlin
inner class MyLayoutManager : RecyclerView.LayoutManager() {
    private var mTotalHeight = 0//item总高度，当item不足以填满recyclerView时为recyclerView的高度
    private var mSumDy = 0//记录dy的累加量，有上下界限制[0 ~ itemHeights-recyclerView.height]

    private var mItemWidth = 0
    private var mItemHeight = 0
    
    private var mItemRects = SparseArray<Rect>()
    private var mHasAttachedItems = SparseBooleanArray()

    override fun generateDefaultLayoutParams() = RecyclerView.LayoutParams(
        ViewGroup.LayoutParams.WRAP_CONTENT,
        ViewGroup.LayoutParams.WRAP_CONTENT
    )

    override fun onLayoutChildren(recycler: RecyclerView.Recycler, state: RecyclerView.State?) {
        super.onLayoutChildren(recycler, state)
        //剥离所有child
        detachAndScrapAttachedViews(recycler)
        //没有child就不显示
        if (itemCount == 0) {
            return
        }

        mHasAttachedItems.clear()
        mItemRects.clear()

        //根据第一个child 获取item宽高、一屏可见的child个数
        val view = recycler.getViewForPosition(0)
        measureChildWithMargins(view, 0, 0)
        mItemWidth = getDecoratedMeasuredWidth(view)
        mItemHeight = getDecoratedMeasuredHeight(view)

        val visibleCount = if (getVerticalSpace() % mItemHeight == 0) {
            getVerticalSpace() / mItemHeight
        } else {
            getVerticalSpace() / mItemHeight + 1
        }

        //将所有child的宽高保存起来
        var offsetY = 0
        repeat(itemCount) {
            val rect = Rect(0, offsetY, mItemWidth, offsetY + mItemHeight)
            mItemRects.put(it, rect)
            offsetY += mItemHeight
        }

        //只为可见的child进行布局
        repeat(visibleCount) {
            val rect = mItemRects.get(it)
            val view = recycler.getViewForPosition(it)
            measureChildWithMargins(view, 0, 0)
            addView(view)
            layoutDecorated(view, rect.left, rect.top, rect.right, rect.bottom)
        }
        mTotalHeight = Math.max(offsetY, getVerticalSpace())
    }

    private fun getVerticalSpace() = height - paddingTop - paddingBottom

    override fun canScrollVertically() = true

    override fun scrollVerticallyBy(
        dy: Int,//dy>0时 列表是上滑
        recycler: RecyclerView.Recycler,
        state: RecyclerView.State?
    ): Int {
        if (childCount <= 0) return 0
        var travel = dy
        if (mSumDy + dy <= 0) {//滑到顶部处理
            travel = -mSumDy//矫正travel
        } else if (mSumDy + dy >= mTotalHeight - getVerticalSpace()) {//滑倒底部处理
            travel = mTotalHeight - getVerticalSpace() - mSumDy
        }

        //直接加上travel的值
        mSumDy += travel
        val visibleRect = getVisibleArea()

        //对可见的child进行遍历，预判child经过travel的平移之后是否屏幕不可见，依此来进行[回收]
        for (i in childCount - 1 downTo 0) {
            val child = getChildAt(i)!!
            val position = getPosition(child)
            val rect = mItemRects.get(position)
            //如果child即将移出可见区域，则回收child，否则就对child进行布局
            if (!Rect.intersects(rect, visibleRect)) {
                removeAndRecycleView(child, recycler)
                mHasAttachedItems.put(position, false)
            } else {
                layoutDecorated(
                    child,
                    rect.left,
                    rect.top - mSumDy,
                    rect.right,
                    rect.bottom - mSumDy
                )
                child.rotationY += 1
                mHasAttachedItems.put(position, true)//记录还在屏幕区域的child的position
            }
        }
        val lastView = getChildAt(childCount - 1)!!
        val firstView = getChildAt(0)!!

        //对不可见的child进行遍历，预判child经过travel的平移之后是否屏幕可见，依此来进行[复用]
        if (travel >= 0) {
            val minPos = getPosition(firstView)
            for (i in minPos until itemCount) {
                insertView(i, visibleRect, recycler, false)
            }
        } else {
            val maxPos = getPosition(lastView)
            for (i in maxPos downTo 0) {
                insertView(i, visibleRect, recycler, true)
            }
        }
        return travel
    }

    /**
     * 获取下一帧可见区域（）
     */
    private fun getVisibleArea() = Rect(
        paddingLeft,
        paddingTop + mSumDy,
        width + paddingRight,
        getVerticalSpace() + mSumDy
    )

    private fun insertView(
        pos: Int,
        visibleRect: Rect,
        recycler: RecyclerView.Recycler,
        firstPos: Boolean
    ) {
        val rect = mItemRects.get(pos)
        //添加即将出现在视区的child，如果本身就已经在的，就不要再添加了
        if (Rect.intersects(rect, visibleRect)&&!mHasAttachedItems.get(pos)) {
            val child = recycler.getViewForPosition(pos)
            if (firstPos) {
                addView(child, 0)
            } else {
                addView(child)
            }
            measureChildWithMargins(child, 0, 0)
            layoutDecorated(
                child,
                rect.left,
                rect.top - mSumDy,
                rect.right,
                rect.bottom - mSumDy
            )
            child.rotationY = (child.rotationY + 1)
        }
    }
}
```

### 总结

LayoutManager将RecyclerView作为一个ViewGroup应当做的一些职责——对children测量、布局等全部抽离出来，独立来负责这一块的工作。另外利用Recycler封装的回收机制对child进行回收和复用。

## 回收复用机制

RecyclerView一共有4个缓存：

1. mAttachedScrap：不参与回收复用，只在重新布局保存正在显示的Holder
2. mCachedViews：保存最新被移除的Holder，先进先出。精准匹配是否为刚刚移除的Holder，是才会返回。如来回滑动匹配该缓存的效率最高，不会调用binderViewHolder
3. mRecyclerPool：真正回收复用的池子。从这里拿到的Holder是会调用binderViewHolder的。

4. 自定义缓存——mViewCacheExtension

### Api

- deacheAndScrapAttachedView(Recycler)：将当前显示的Holder从RecyclerView剥离，添加到mAttachedScrap中
- Recycler.getViewForPosition(position)：从四个缓存中获取一个Holder
- removeAndRecyclerView(child，Recycler)：回收Holder到mCachedViews，满了则回收到mRecyclerPool
- getItemCount()：获取Adapter总共的数据条数
- getPosition(View)：得到Item在当前Adapter中的索引
- getChildCount()：获取当前RecyclerView正显示的Item的个数
- getChildAt(position)=View：根据索引获取当前正在显示的Child

## ItemTouchHelper

