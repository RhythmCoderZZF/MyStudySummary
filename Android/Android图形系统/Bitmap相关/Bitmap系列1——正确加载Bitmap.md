# Bitmap系列1——正确加载Bitmap

## 正确加载Bitmap

```kotlin
class ImageResizer {
    fun decodeBitmapFromBitmap(bytes: ByteArray, reqWidth: Int, reqHeight: Int): Bitmap? {
        return BitmapFactory.Options().run {
            inJustDecodeBounds = true//只计算宽高，不生成bitmap
            BitmapFactory.decodeByteArray(bytes, 0, bytes.size, this)
            inSampleSize = calculateInSampleSize(this, reqWidth, reqHeight)//根据bitmap宽高，iv的宽高计算缩放倍数
            inJustDecodeBounds = false//生成bitmap
            BitmapFactory.decodeByteArray(bytes, 0, bytes.size, this)
        }
    }

    fun decodeSampleBitmapFromFileDescriptor(
        fd: FileDescriptor,
        reqWidth: Int,
        reqHeight: Int
    ): Bitmap? {
        return BitmapFactory.Options().run {
            inJustDecodeBounds = true//只计算宽高，不生成bitmap
            BitmapFactory.decodeFileDescriptor(fd, null, this)
            inSampleSize = calculateInSampleSize(this, reqWidth, reqHeight)//根据bitmap宽高，iv的宽高计算缩放倍数
            inJustDecodeBounds = false//生成bitmap
            BitmapFactory.decodeFileDescriptor(fd, null, this)
        }
    }

    fun decodeSampleBitmapFromStream(bytes: InputStream, reqWidth: Int, reqHeight: Int): Bitmap? {
        return BitmapFactory.Options().run {
            inJustDecodeBounds = true//只计算宽高，不生成bitmap
            BitmapFactory.decodeStream(bytes, null, this)
            inSampleSize = calculateInSampleSize(this, reqWidth, reqHeight)//根据bitmap宽高，iv的宽高计算缩放倍数
            inJustDecodeBounds = false//生成bitmap
            BitmapFactory.decodeStream(bytes, null, this)
        }
    }

    /**
     * 计算要缩放的inSampleSize大小：
     * 1. 长和宽各缩放的大小（2的倍数）
     */
    private fun calculateInSampleSize(
        options: BitmapFactory.Options,
        reqWidth: Int,
        reqHeight: Int
    ): Int {
        val (w, h) = options.run { outWidth to outHeight }//Pair(w;h)
        var inSampleSize = 1
        if (w > reqWidth || h > reqHeight) {
            val halfW = w / 2
            val halfH = h / 2
            while (halfW / inSampleSize > reqWidth || halfH / inSampleSize > reqHeight) {
                inSampleSize *= 2
            }
        }
        return inSampleSize
    }
}
```
