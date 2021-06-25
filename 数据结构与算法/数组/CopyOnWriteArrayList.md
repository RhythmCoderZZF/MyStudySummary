参考：

[CopyOnWriteArrayList你都不知道，怎么拿offer？](https://www.jianshu.com/p/ce7f203731af)

# CopyOnWriteArrayList

##  概述

```java
// Android-changed: Removed javadoc link to collections framework docs
/**
 * A thread-safe variant of {@link java.util.ArrayList} in which all mutative
 * operations ({@code add}, {@code set}, and so on) are implemented by
 * making a fresh copy of the underlying array.
 *
 * <p>This is ordinarily too costly, but may be <em>more</em> efficient
 * than alternatives when traversal operations vastly outnumber
 * mutations, and is useful when you cannot or don't want to
 * synchronize traversals, yet need to preclude interference among
 * concurrent threads.  
 
 * The "snapshot" style iterator method uses a
 * reference to the state of the array at the point that the iterator
 * was created. This array never changes during the lifetime of the
 * iterator, so interference is impossible and the iterator is
 * guaranteed not to throw {@code ConcurrentModificationException}.
 * The iterator will not reflect additions, removals, or changes to
 * the list since the iterator was created.  Element-changing
 * operations on iterators themselves ({@code remove}, {@code set}, and
 * {@code add}) are not supported. These methods throw
 * {@code UnsupportedOperationException}.
 *
 * <p>All elements are permitted, including {@code null}.
 *
 * <p>Memory consistency effects: As with other concurrent
 * collections, actions in a thread prior to placing an object into a
 * {@code CopyOnWriteArrayList}
 * <a href="package-summary.html#MemoryVisibility"><i>happen-before</i></a>
 * actions subsequent to the access or removal of that element from
 * the {@code CopyOnWriteArrayList} in another thread.
 *
 * @since 1.5
 * @author Doug Lea
 * @param <E> the type of elements held in this list
 */
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {}
```

> 1. ArrayList的变种，在增量操作上是**线程安全**的。
> 2. 使用改集合代价非常昂贵，但是非常适合**读多写少的并发操作**。
> 3. 集合的迭代器**不会抛ConcurrentModificationException异常**（即允许遍历的过程对List修改），但是不允许利用迭代器进行增、删、改操作。

## 源码

```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    
    final transient Object lock = new Object();
    private transient volatile Object[] array;
    
    final Object[] getArray() {
        return array;
    }

    final void setArray(Object[] a) {
        array = a;
    }
    //1.读，不加锁
    public E get(int index) {
        return get(getArray(), index);
    }
    
    //2.写，加锁
    public E set(int index, E element) {
         synchronized (lock) {
            Object[] elements = getArray();
            E oldValue = get(elements, index);

            if (oldValue != element) {
                int len = elements.length;
                //copy出新的数组，将新元素替换至这个新数组中，并不在原数组上操作，
                Object[] newElements = Arrays.copyOf(elements, len);
                newElements[index] = element;
                //将旧数组替换为新数组
                setArray(newElements);
            } else {
                // Not quite a no-op; ensures volatile write semantics
                setArray(elements);
            }
            return oldValue;
        }
    } 
    
    //迭代器
    public Iterator<E> iterator() {
        return new COWIterator<E>(getArray(), 0);
    }
    
     static final class COWIterator<E> implements ListIterator<E> {
        private final Object[] snapshot;
        private int cursor;

        COWIterator(Object[] elements, int initialCursor) {
            cursor = initialCursor;
            //迭代器创建的时候传入的array为snapshot，由于增量操作会创建新的array，不会影响snapshot，因此迭代器不会报ConcurrentModificationException异常
            snapshot = elements;
        }

        public boolean hasNext() {
            return cursor < snapshot.length;
        }

        public boolean hasPrevious() {
            return cursor > 0;
        }

        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }

       
        public E previous() {
            if (! hasPrevious())
                throw new NoSuchElementException();
            return (E) snapshot[--cursor];
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor-1;
        }

     
        public void remove() {
            throw new UnsupportedOperationException();//禁止
        }

       
        public void set(E e) {
            throw new UnsupportedOperationException();//禁止
        }

        
        public void add(E e) {
            throw new UnsupportedOperationException();//禁止
        }
    }
}
```

## 用途

在Android的ViewTreeObserver中大量使用CopyOnWriteArrayList

```java
private CopyOnWriteArrayList<OnWindowFocusChangeListener> mOnWindowFocusListeners;
private CopyOnWriteArrayList<OnWindowAttachListener> mOnWindowAttachListeners;
private CopyOnWriteArrayList<OnGlobalFocusChangeListener> mOnGlobalFocusListeners;

private CopyOnWriteArrayList<OnTouchModeChangeListener> mOnTouchModeChangeListeners;
private CopyOnWriteArrayList<OnEnterAnimationCompleteListener> mOnEnterAnimationCompleteListeners;
```

```java
final void dispatchOnGlobalFocusChange(View oldFocus, View newFocus) {
    final CopyOnWriteArrayList<OnGlobalFocusChangeListener> listeners = mOnGlobalFocusListeners;
    if (listeners != null && listeners.size() > 0) {
        //遍历CopyOnWriteArrayList,回调onGlobalFocusChanged
        for (OnGlobalFocusChangeListener listener : listeners) {
            listener.onGlobalFocusChanged(oldFocus, newFocus);
        }
    }
}
```

利用CopyOnWriteArrayList遍历可修改的特性，可以写出如下代码

```kotlin
class UIGlobalFocusActivity : BaseActivity() {
    private lateinit var listener: ViewTreeObserver.OnGlobalFocusChangeListener
    init {
        listener=ViewTreeObserver.OnGlobalFocusChangeListener{_,_->
            //回调onGlobalFocusChanged，即使在遍历过程也可以删除
            window.decorView.viewTreeObserver.removeOnGlobalFocusChangeListener(listener)
        }
    }

    override fun initViewAndData(savedInstanceState: Bundle?) {
        window.decorView.viewTreeObserver.addOnGlobalFocusChangeListener (listener)
    }
}
```