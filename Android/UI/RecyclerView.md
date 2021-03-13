#### RecyclerView

##### DiffUtil

> 用`Myers差分算法`对`oldList`和`newList`的数据进行差异比较，实现`RecyclerView`的增量更新。

**DiffUtil.ItemCallback：**

> 提供`Item`是否相同的业务逻辑，差分算法依此来区分新旧`Item`是否相同

- ```kotlin
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

  

  