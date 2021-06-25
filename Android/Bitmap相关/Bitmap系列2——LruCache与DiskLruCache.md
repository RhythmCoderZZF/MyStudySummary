# Bitmap系列2——LruCache与DiskLruCache

 ## LruCache

在内存中存储

### 使用

```kotlin
	//1.指定缓存最大阈值
    private val MAX_SIZE = Runtime.getRuntime().maxMemory().toInt / 1024 / 8
	//2.初始化Cache
    private val mCache = object : LruCache<String, Bitmap>(MAX_SIZE) {
        override fun sizeOf(key: String, value: Bitmap): Int {
            return value.byteCount//返回每一个Bitmap所占内存的大小
        }
    }
  
   //3.存取
   mCache.put(hashKeyFromUrl(url), bitmap)
   mCache.get(hashKeyFromUrl(url))

   mCache.snapshot() //cache快照

/**
  将url转为hash，以key值存储
*/
 private fun hashKeyFromUrl(url: String): String {
        val cacheKey: String
        cacheKey = try {
            val messageDigest = MessageDigest.getInstance("MD5")
            messageDigest.update(url.toByteArray())
            bytesToHexString(messageDigest.digest())
        } catch (e: NoSuchAlgorithmException) {
            url.hashCode().toString()
        }
        return cacheKey
    }

    private fun bytesToHexString(bytes: ByteArray): String {
        val stringBuilder = StringBuilder()
        for (i in bytes.indices) {
            val hex = Integer.toHexString(0xFF and bytes[i].toInt())
            if (hex.length == 1) {
                stringBuilder.append('0')
            }
            stringBuilder.append(hex)
        }
        return stringBuilder.toString()
    }
```

### 源码

*put*

```java
   private final LinkedHashMap<K, V> map;

    public final V put(@NonNull K key, @NonNull V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;
            
            //1.累加容积
            size += safeSizeOf(key, value);
            
            //2.put
            previous = map.put(key, value);
            if (previous != null) {
                
                //3.如果key存在，则移除之前key所对应的value占的容量
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, value);
        }
		
        //4.触发容积检查
        trimToSize(maxSize);
        return previous;
    }   

    public void trimToSize(int maxSize) {
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize || map.isEmpty()) {
                    break;
                }

                //1. 若超过最大容积，则移除最早添加的元素
                Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }
```

*get*

```java
public final V get(K key) {
    if (key == null) {
        throw new NullPointerException("key == null");
    }

    V mapValue;
    synchronized (this) {
        
        //1.获取value
        mapValue = map.get(key);
        if (mapValue != null) {
            hitCount++;
            return mapValue;
        }
        missCount++;
    }

...
}
```

### 小结

put元素会累计元素容量，每一次添加都会触发对容器的检查。当元素容量大于最大限制，则开始移除最早添加的元素，直到小于阈值 

## DiskLruCache

缓存在磁盘中，每次读取都是IO操作。

### 使用

*添加元素*

```kotlin
val myCacheFile = File("${dataDir.absolutePath + "/myDiskCacheDir"}")
if (!myCacheFile.exists()) {
    myCacheFile.mkdirs()
}
/**
此处DiskLruCache为Glide中的实现
1.
p1:缓存路径；
p2:app版本，当app升级时会删除缓存文件夹
p3:一个key对应几个File,一般为1，即key value一一对应，否则需要index指定时第几个value
p4:阈值
*/
val cache = DiskLruCache.open(myCacheFile, 1, 1, 4)
datas.forEachIndexed { i, d ->
    
    //2.生成edit，参数为key
    val editor = cache.edit("$i")
    
    //3.向cache添加一个key=i，value=d的entry（0标识index，由于之前设置一个key对应一个file，所以设置为0）
    editor.set(0, "$d")
    
    //4.提交
    editor.commit()
    CmdUtil.v("put:$d")
}

//5.使用完记得关闭
cache.flush()
cache.close()
```

*获取*

```kotlin
//1.构建cache
val cache1 = DiskLruCache.open(myCacheFile, 1, 1, 4)

//2.获取key对应的Value对象
 val value = cache1.get("$i")

//3.通过Value对象获取value（0标识index，由于之前设置一个key对应一个file，所以设置为0）
 val res = value.getString(0)
```
