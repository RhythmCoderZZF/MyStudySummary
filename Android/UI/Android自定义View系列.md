### Android自定义View系列

#### Paint

> Canvas绘制形状，通过Paint进行填充效果

##### API

- setShadowLayer(float radius, float dx, float dy, @ColorInt int shadowColor) 阴影效果：可以为文字|图形|bitmap添加阴影效果，但无法为bitmap添加纯色阴影
- setMaskFilter(MaskFilter maskfilter) 滤镜：MaskFilter 有两个派生类 BlurMaskFilter | EmbossMaskFilter。BlurMaskFilter用于对文字|图形|bitmap的边界进行高斯模糊， 其有四种模糊模式——Blur.INNER|OUTER|NORMAL|SOLID
- setShader (Shader shader)  着色器：用于对图形的填充(`LinearShader`、`BitmapShader`)。`Shader`有3种模式（`TileMode.CLAMP`|`REPEAT`|`MIRROR`）。填充空间时，先填充满y轴方向，再填充x方向。`Shader`与`Canvas`绘制多大的图形，在哪里绘制都无关，因为**Shader永远从控件原点开始填充（可以通过`Shader.setLocalMatrix`来改变起始位置）。
- setXfermode(Xfermode) 混合模式。混合两个bitmap的像素，其中有16种混合模式。使用过程：1先绘制Dst；为Src指定Paint.setXfermode；3最后绘制Src

------

#### Canvas

> 类似于Socket API，一个操作Bitmap的接口。Canvas通过API对Bitmap进行绘制。

**画布（BItmap）、图层（Layer）、Canvas**

- 图层（Layer）：每次调用 canvas.drawXXX 系列函数，都会生成一个透明图层专门来绘制这个图形。平移旋转只会影响到新的图层
- 画布 （Bitmap）: 每块画布都是一个Bitmap ，所有的图像都是画在这个 Bitmap 上的。我们知道，每次调用 canvas.drawXXX 系列函数，都会生成一个专用的透明图层来绘制这个图形，绘制完成以后，就覆盖在画布上。所以，如果我们连续调用5个draw函数，就会生成5个透明图层，画完之后依次覆盖在画布上显示。画布有两种，一种是View 的原始画布，是通过 onDraw(Canvas canvas）函数传入的，参数中的 canvas 对应的是 View 的原始画布，控件的背景就是画在这块画布上的；另一种是人造画布，通过saveLayer（）、 new Canvas(bitmap）等函数来人为地新建一块画布。尤其是 saveLayer() 函数， 一旦调用 saveLayer（）函数新建一块画布，以后所有 draw 函数所画的图像都是画在这块画布上的，只有在调用 restore （）、 resoreToCount（）函数以后，才会返回到原始画布上进行绘制。
- Canvas: Canvas 是画布的表现形式，我们所要绘制的任何东西都是利用 Canvas 来实现的。在代码中， Canvas 的生成方式只有一种一new Canvas( bitmap ），即只能通过Bitmap 生成，无论是原始画布还是人造画布，所有的画布最后都是通过 Canvas 画到Bitmap 上的。可以把 Canvas 理解成绘图的工具，利用它所封装的绘图函数来绘图，而所要绘制的内容最后是画在 Bitmap 上的。所以，如果我们利用 Canvas.clipXXX列函数将画布进行裁剪，其实就是把它对应的 Bitmap 进行裁剪，与之对应的结果是以后再利用 Canvas 绘图的区域会减小。

注意：

> 1. Canvas的3种构建方式——1：onDraw(Canvas) 2：Canvas(Bitmap) 3：SurfaceHolder.lockCanvas()
>2. saveLayer函数会创建一块新的画布（Bitmap），假如1个像素是8bit，1024*720像素的屏幕所占空间为1024 * 720 * 8=6.2MB，因此要避免使用该函数

##### API

- `drawXXX()`：**每次绘制都先创建一个空白图层（Layer），在这个图层绘制完后立即和上一个图层合并**（在XFermode中会有体现）

- `drwPath()`：绘制路径

  **Path**

  > 一个线、曲线的集合。可以使用Stroke或Fill填充模式。文字可以绘制到路径上**

  - ```java
    addCircle(float x, float y, float radius, Direction dir)
    ```

    添加圆形。Direction.CW：顺时针   Direction.CCW：逆时针。Direction影响文字的绘制方向和PathMeasure的测量计算方向

  - ```java
    setFillType(FillType ft) 
    ```

    Path内部如何填充图形相交的区域

  - ```java
    reset() //相当于new Path()。会重置FillType
    rewind() //类似于list.clear()。会保存之前的数据结构但不会保留FillType，便于快速重用。
    ```

  **PathMeasure**

    > 一个用于专门对Path做测量计算的类，用于矢量动画
  
    - 构造函数
    
      传入Path和forceClosed（Boolean）,其中forceClosed函数——代表测量计算时是否闭合，不关乎`Path`绘制，`ture`闭合，`false`不闭合，其中为true时会多计算一段Path closed闭合路径
    
    - `getSegment(float startD, float stopD, Path dst, boolean startWithMoveTo)`函数：对PathMeasure内部的Path截取
    
      startD：距离Path起始点的位置
    
      stopD：距离Path终点的位置
    
      dst：截取后的Path**添加**到dst（如果startD==stopD或者它们都不在【0~PathMeasure.getLength】范围内该函数返回false，且不改变dst）
    
      startWithMoveTo：截取的Path起始点是否调用moveTo函数。fasle——表示不调用moveTo，则起始点会和dst中上一个Path片段的终点连接，保证连续性；true——保证每个截取的Path片段的独立性（一般为true）
    
    - `getPosTan (float distance, float[] pos, float[] tan)`函数：得到路径上某一长度的位置以及该位置的正切值：
    
      distance：距离Path起始点的位置
    
      pos：得到该点的坐标赋值给pos[]
    
      tan：得到该点的正切值赋值给tan[]
    
    - `getMatrix (float distance, Matrix matrix, int flags)`：得到路径上某一长度的位置以及该位置的正切值的矩阵，属于getPosTan的另一种实现方式
    
      distance：距离 Path 起点的长度
    
      matrix： 根据 falgs 封装好的matrix
    
      flags：规定哪些内容会存入到matrix中（POSITION_MATRIX_FLAG(位置) | ANGENT_MATRIX_FLAG(正切)）

- `drawPaint(Paint)`：当前Canvas的Bitmap用指定Paint进行填充（受限于clip大小），等效于用指定Paint绘制无限大的Rect，但是更快速

- `drawBitmap()`：将指定Bitmap绘制到Canvas的Bitmap中，其中Paint参数没有作用，**除非Canvas中的Bitmap只有Alpha通道，则Paint中指定的颜色可以绘制出来（Alpha值也无法修改）**

- save()`：保存当前Canvas Matrix和Clip到一个私有栈，之后和往常一样调用平移、裁剪等操作，但是若调用restore函数，则这些操作会被清除，Canvas会恢复到save()前的状态

- `saveLayer(RectF bounds, Paint paint)`：和save函数类似，但除了保存平移、裁剪等还会创建**全新的一个空白**离屏的画布（BItmap）（这内存开销很大！），之后所有的绘制都绘制到这个画布，直到调用restore和上一个画布合并。

------

#### [Drawable](https://blog.csdn.net/feather_wch/article/details/79124608)

>1. Drawable和View的区别在于不能接收事件，无法与用户交互。
>2. Drawable中的Canvas是由View传递过来的，通过提前在Drawable中保存绘制逻辑代码，使得在View在drawBackground时绘制Drawable。
>3. `Drawable`是有边界的，边界大小是`View`的大小


自定义Drawable步骤：

1. 实现抽象方法——

   【一】getIntrinsicHeight/Width() 代表默认大小，若Drawable绘制Bitmap，则应该是bitmap的宽高；

   【二】setAlpha()/setColorFilter()设置给Paint；

   【三】getOpacity()默认返回PixelFormat.TRANSLUCENT；

   【四】setBounds() 表示Drawable的边界，注意其调用时机为 ImageView.setFrame 或者 View.drawBackground 过程中

##### API

- setBounds(Rect)：指定当前drawable在View显示的位置。

  【一】 Drawable绘制的区域不应该超过bounds，绘制前需要计算bounds的大小和在View中的坐标；

  【二】 setBounds的默认调用时机——View.drawBackground或者ImageView.setFrame中调用，默认为View的宽高；

  【三】ImageView.setImageDrawable流程——会先根据ScaleType模式计算Drawable的Bounds和Matrix（FitXY模式的Bounds是ImageView的宽高，而其他模式是Drawable的宽高），然后在调用Drawable.draw(Canvas)方法**前**对Canvas进行Matrix转换，以此来实现不同ScaleType实现不同的显示效果。

------

#### DisplayMetrics

屏幕密度相关

##### API

- density：默认160dpi的倍数
- densityDpi：像素密度

------

#### Bitmap

> Bitmap 在绘图中是 个非常重要的概念，在我们熟知的 Canvas 中就保存着 Bitmap对象，我们调用 Canvas 的各种绘图函数，最终还是绘制到其中的 Bitmap 上的。 我们知道，在自定义 View 时，一般都会重写一个函数 onDraw(Canvas canvas），在这个函数中是自带Canvas 参数的，只需要将需要画的内容调用 Canvas 的函数画出来，就会直接显示在对应的View上。 其实，真正的原因是， View 对应着一个 Bitmap ，而 onDraw（）函数中的 Canvas 参数就是通过这个 Bitmap 创建出来的。

##### 要点

- **如何加载大图？**

  利用BitmapFactory.Option inJustDecodeBounds=false先加载Bitmap参数，再计算出合适的inSampleSize（采样压缩）来加载大图。

- **不同像素密度文件下的图片加载到内存是怎么缩放的？**

  如果一张图片放到mdpi文件夹下（160dpi），当前设备dpi实际是480dpi，则利用BitmapFactory.decodeResource解析Bitmap时，会将该图片宽高放大480/160=3倍显示到屏幕，其内存也会暴涨；相反，高像素密度文件夹的图片加载到低分辨率手机，图片尺寸会减小，内存也会减小。

  如果图片不是在dpi文件夹下是没有对应的dpi的，默认dpi=160。因此一定要对BitmapFactory进行采样压缩

- **[如何对Bitmap优化？](https://mp.weixin.qq.com/s/RTRkNXOzrtb0T7FuG-vIwA)**

  1. 若Bitmap没有透明度，且对色彩要求不高，则可以使用RGB 565格式，内存比ARGB 8888小一半哦
  2. 利用BitmaFactory.Option inBitmap来复用Bitmap（且inMutable要为true），能有效减小内存开销（注意复用的Bitmap不能比被复用的Bitmap大！）
  
- **如何清空Bitmap?**

  bitmapCanvas . drawColor(Color.TRANSPARENT , PorterDuff . Mode . CLEAR)

##### API

BitmapFactory.Option（其中参数in开头的是set方法，out开头的是get方法）

- inJustDecodeBounds： 只获取图片信息（如宽高），不加载到内存

- inSampleSize：采样率——如4则表示宽高所缩小为原来的1/4

- inScaled：缩放——当图片所在资源文件所对应的屏幕分辨率与真实显示的屏幕分辨率不相同时（如手机屏幕对应的分辨率为xhdpi，但图片只存在于hdpi），是否缩放图片

- inDensity ：图片应当适配的屏幕dpi(像素密度)

- inTargetDensity：目标机器真实的屏幕dpi

  >1. 当inDensity＝inTargetDensity，设置的结果会被设置到Bitmap.density。
  >
  >2. 当inDensity≠inTargetDensity，并且inScaled=true（默认为true），则Bitmap.density为inTargetDensity（屏幕dpi）且返回创建的Bitmap前会缩放至inTargetDensity的屏幕dpi。
  >3. 如果inDensity=0，则inDensity的值会根据资源目录（hdpi、xhdpi）来自动匹配（若不在，则默认为160dpi），inTargetDensity为屏幕dpi。
  >4. **Bitmap.density修改后不会改变Bitmap的尺寸及大小；BitmapFactory.Option的inDensity修改后会修改Bitmap的尺寸和大小**

-  inPreferredConfig：设置图片像素存储格式——有 ALPHA | RGB 565 | ARGB 4444 | RGB 8888 默认使用 ARGB 8888，如RGB 565，一张1200*700的图片所占内存=1200 * 700 * 2 Byte（5+6+5=16位）
- outHeight/outWidth：Bitmap宽高
- inBitmap：在解析Bitmap时重用该Bitmap，但是必须相同大小的Bitmap & inMutable = true 才可重用；

Bitmap

- Bitmap createBitmap(int width, int height,  Config config) :创建一张指定宽高的空白Bitmap，config为像素存储格式
- Bitmap createBitmap(Bitmap src)/ createBitmap(Bitmap source, int x, int y, int width, int height)：copy src Bitmap/copy 起始点开始指定宽高的Bitmap
- Bitmap createBitmap( Bitmap source, int x, int y, int width, int height, Matrix m, boolean filter) ：和上面一样，多两个参数——Matrix：裁剪后的图像添加矩阵；filter：对应paint.setFilterBitmap(filter），滤波效果、
- Bitmap createScaledBitmap( Bitmap src, int dstWidth, int dstHeight,boolean filter)：copy src Bitmap并缩放到指定宽高。当createScaledBitmap创建的Bitmap与src Bitmap的缩放比一样时会返回src Bitmap

- Bitmap copy(Config config, boolean isMutable)：copy src Bitmap，isMutable表示像素是否可以修改

  > 像素可更改的含义：
  >
  > 如Canvas(Bitmap)创建Canvas想要绘制到Bitmap上，则必须要求该Bitmap是可以修改的，及Bitmap.isMutable=true，否则会报错。
  >
  > 其中BitmapFactory加载的Bitmap都是不可修改的，Bitmap.createBitmap创建的Bitmap是可以修改的。
  >
  > 注意当createScaledBitmap创建的Bitmap与src Bitmap的缩放比一样时会返回src Bitmap

- Bitmap extractAlpha()：copy src Bitmap，只保留src Alpha

- Bitmap extractAlpha(Paint paint, int[] offsetXY) ：copy src Bitmap，只保留src Alpha。其中参数Paint具有MaskFilter效果（一般用BlurMaskFilter）；参数offsetXY是返回的IntArray，offsetXY 只是建议的绘制起始位置，其取值并不一定与 BlurMaskFilter 的模糊半径一致。当模糊半径较大 ，一般 offsetXY 值会偏小。因为当模糊半径比较大的时候，边缘的效果就已经不明显了，所以起始位置就不必按照模糊半径来计算了。

- getAllocationByteCount：获取设备为该Bitmap分配的内存

- getByteCount()：获取设备为该Bitmap分配的**最小**内存

  >getAllocationByteCount和getByteCount的区别：
  >
  >一般情况两者相同，只有当Bitmap发生复用（第二张Bitmap用了第一张Bitmap的内存空间），则getAllocationByteCount会大于等于getByteCount()——**第二张Bitmap所占内存必须小于第一张，否则复用会崩溃**

- setDensity()：详见BitmapFactory.Option inDensity

- setPixel/getPixel：修改像素颜色/获取像素颜色

- compress(CompressFormat format, int quality, OutputStream stream)：图片压缩。p1：压缩格式；p2：画质（0~100）；p3：输出流

  > JPEG：有损压缩，文件较小
  >
  > PNG：无损压缩，文件较大
  >
  > WEBP：无损压缩，文件比PNG小26%，但是牺牲压缩时间
  >
  > 注意：只要图片宽高不变，再怎么压缩，图片显示所占的内存大小不变

其他

-  int getAllocationByteCount()：Bitmap所占内存字节数
- recycle()：回收Bitmap，释放内存。Android 10以前，Bitmap的像素数据是在Native中，必须手动调用recycle()释放内存；而在10以上版本，这些数据是在虚拟机堆中，因此不需要调用，gc会自动回收。
- setDensity()/getDensity()：设置图像建议的屏幕尺寸，对应BitmapFactory.Option中的inDensity。inDensity用于表示该 Bitmap 适合的屏幕 dpi， 当目标屏幕的 dpi ( inTargetDensity ）不等于它时，将会缩放图像以适应目标机器。

------

#### 动画

##### 视图动画

> 视图动画包括4个基本的补间动画（android.view.animation）和1个帧动画，只能所用于View上，且动画的过程无法真实修改View的属性值

**自定义Animation**
继承`Animation`需要实现`Transformation`。该对象保存图形变化的`Matrix`和`Alpha`

##### 属性动画

> 属性动画（android.animation）脱离于View之外，作用于特定的属性值

**开发事项**

1. ofXXX构造函数——

   【一】若只写了一个参数，则ObjectAnimator 会通过查找对应属性的 getProperty 函数来获得初始值，若没有则会取基本数据类型的缺省值；

   【二】指定的ofFloat参数在动画过程中只会大概执行到，在INFINITE模式下，初始值和结束值也不会执行到；

   【三】ofInt、ofFloat是有默认的evaluator，而ofObject需要开发者提供对应的TypeEvaluator

2. xml 写法需要写明valueType

3. propertyName是如 setTranslationX 除去set之后的单词，无论首字母大小写都可以



###### PropertyValuesHolder和Keyframe

1. PropertyValuesHolder保存了动画操作的属性和对应的值，其中值用Keyframe封装（动画完成度＋该完成度对应的属性值）
2. Keyframe至少有2帧，起实帧和结束帧不必为0或1
3. Keyframe可以添加独自的Interpolator，作用在当前帧和上一帧之间。第一帧添加Interpolator无效

------

##### SVG动画

> SVG：全称是 Scalable Vector Graphics （可缩放矢量图形）

###### Vector XML标签

- width height : 表示该 SVG 形的具体大小

- viewportWidth viwportHeight ： 表示 SVG 图形划分的比例。

- path：Path路径

- trimPathStart：取值为0~1表示路径开始位置的百分比。当取值为0时， 表示从头部开始；当取值为1时，整条路径不可见（trimPathEnd类似）

- trimPathOffset ：该属性用于指定结果路径的位移距离 ，取值为0~1。当取值为0时， 不进行位移；当取

  值为1时，位移整条路径的长

- group：

###### animated-vector XML标签

> animated-vector：也是Vector，可以当作资源文件添加到IamgeView中。其包含两部分：Vector图片和Animator，其中Animator专门对Vector中的Path的相关属性做动画

- drawable属性：Vector图片
- target属性：其中name指向Vector中Path的name；animation指向一个animator

------

#### Theme与Style

**Style**

- Style定义在values.styles.xml中。它是指具体某一类控件的风格，如TextView、Button。

```xml
<style name="BtnStyle">
    <item name="android:textStyle">bold</item>
    <item name="android:textSize">35sp</item>
    <item name="android:textColor">#FFFFFFFF</item>
</style>
```

- Style被使用在控件的Style属性和被Theme引用

- Style属性值设置的三种方式

  ```xml
  <item name="android:textColor">#FFFFFFFF</item>
  
  <!--text_color的数值已经定义在了别的地方-->
  <item name="android:textColor">@color/text_color</item>
  
  <!--使用与android:textColorLink属性相同的值，而不关心这个数值是到底是多少,
  而该属性必须是在当前Theme中定义了的。-->
  <item name="android:textColor">?android:textColorLink</item>//引用Android定义的属性当前Theme下的值
  <item name="android:textColor">?textColorLink</item>//引用自定义的属性当前Theme下的值
  ```

**Theme**

- Theme定义在values.themes.xml 中。它是一个全局风格的集合。

  theme的定义与style的定义完全一样，一样的标签、一样的写法。但是Theme还能引用Style类型。如：

  ```xml
   <style name="Theme.ExpandedMenu" parent="Theme.Holo">
          <item name="itemTextAppearance">?attr/textAppearanceLarge</item>
          <item name="listViewStyle">@style/Widget.ListView.Menu</item>
          <item name="windowAnimationStyle">@style/Animation.OptionsPanel</item>
          <item name="background">@null</item>
   </style>
  ```

- Theme被使用在Application和Activity中。

  ```xml
  <application
      <!--指定应用的theme-->
      android:theme="@style/MyTheme">
  <activity android:name=".MainActivity"
      <!--指定应用的theme-->
      android:theme="@style/MyTheme">
  ```

  ```java
  this.setTheme(R.style.MyTheme);//Alpplication和Activity的onCreate()中
  ```

- Theme的属性值设置和Style一样

------

#### 封装控件

- **attrs文件 declare-styleable的标签和属性 与代码获取对应属性值**

  属性定义：

  1. reference：资源id——R.dimen.xxx
  2. dimension：尺寸——dp px dip
  3. fraction：百分数——50%
  4. enum：枚举——如系统属性orientation
  5. flag：位运算——如系统属性gravity（enum只有一种类型，flag也是类型但是是可以利用位运算组合叠加）

  属性获取：

  ```java
      context.obtainStyledAttributes(attrs, R.styleable._MyPackageView).apply {
              try {
                  val mColor = getColor(R.styleable._MyPackageView_fillColor, Color.BLACK)
                  mSize = getDimension(R.styleable._MyPackageView_r, 200f)
              } finally {
                  recycle()//注意要手动回收
            }
          }
  ```
  

------

#### Measure

> onMeasure()的主要职责是测量自身，当然ViewGroup还必须测量子View，根据子View的测量结果评估自身的宽高。这个所评估的宽高并非最终的宽高，还需要交给layout去决定。

##### 测量流程

**自定义View的测量流程：**

1. 计算控件自身逻辑的内容宽高（基于控件算法）

2. 计算完毕的宽高要加上内边距（内边距也属于内容一部分！）

3. 内容宽高要和getSuggestedMinimumWidth()/Height()进行Max比较，取最大值。实际上就是保留【内容宽高】、【最小宽高】、【背景宽高】中的最大值

   ```java
   protected int getSuggestedMinimumWidth() {
       return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
   }
   ```

4. 和父View为子View计算的MeasureSpec进行一次修正。**wrap_content就取上述的计算结果；EXACTLY就取父view计算的结果，本身已经是我们定义的确切宽高**

   ```java
   public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
           final int specMode = MeasureSpec.getMode(measureSpec);
           final int specSize = MeasureSpec.getSize(measureSpec);
           final int result;
           switch (specMode) {
               case MeasureSpec.AT_MOST:
                   if (specSize < size) {//AT_MOST：超过父边界，取SoecSize
                       result = specSize | MEASURED_STATE_TOO_SMALL;
                   } else {
                       result = size;//AT_MOST：未超过父边界，取自己期望的尺寸
                   }
                   break;
               case MeasureSpec.EXACT父View://EXACTLY：取父View为自己计算的SpecSize
                   result = specSize;
                   break;
               case MeasureSpec.UNSPECIFIED:
               default:
                   result = size;
           }
           return result | (childMeasuredState & MEASURED_STATE_MASK);
       }
   ```

5. 调用setMeasuredDimension()保存宽高

   以ImageView为例的onMeasure()实现：

   ```java
           w += pleft + pright;//【一】计算w、h的内容宽高；【二】内容宽高需要加上内边距
           h += ptop + pbottom;
      
           w = Math.max(w, getSuggestedMinimumWidth());//【三】取内容、最小宽高、背景的最大值
           h = Math.max(h, getSuggestedMinimumHeight());
      
           widthSize = resolveSizeAndState(w, widthMeasureSpec, 0);//【四】修正——取父View计算的值还是自己wrap_content计算的值
           heightSize = resolveSizeAndState(h, heightMeasureSpec, 0);
              
           setMeasuredDimension(widthSize, heightSize);//【五】调用setMeasuredDimension()保存宽高
   ```

   

**自定义ViewGroup的测量流程：**

1. 遍历子View

2. 为每个子View计算其MeasureSpec（子View根据这个SpecMode取SpecSize还是自己计算的尺寸）

3. 将MeasureSpec传递给子View，子View自己再计算其自身的宽高

4. 父View再根据每个子View已经计算完毕的宽高再得出其自身宽高（这一部分重复View的测量流程）。

   下面以FrameLayout的onMeasure()为例：

```java
for (int i = 0; i < count; i++) {//【一】遍历子View
    final View child = getChildAt(i);
    if (mMeasureAllChildren || child.getVisibility() != GONE) {
        measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);//【二】为每个子View计算其MeasureSpec；【三】并传递给子View
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        maxWidth = Math.max(maxWidth,
                child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);//【四】父View再根据每个子View计算其自身宽高。
        maxHeight = Math.max(maxHeight,
                child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
      //...【五】剩余部分类似View的测量流程
    }
}

//重点在第【二】步 ——为每个子View计算其MeasureSpec
 protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);//【三】MesureSpec传递给子View
    }

 public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        }
     //...
 }
```

##### 要点

- **为什么会有MeasureSpec的存在，父View要为子View计算其MeasureSpec？**

  根本原因是因为Android布局有match_parent和wrap_parent两个不确定的尺寸。

  父View和子View之间的这两种尺寸模式的组合会对尺寸产生不确定性，但是这种不确定性是完全有规律的，因此Android封装了

  ```java
  public static int getChildMeasureSpec(int spec, int padding, int childDimension) 
  ```

  算法来帮助程序员对这两种模式的组合计算。

  > 为子View计算的SpecMode为ECACTLY时，SpecSize实际上就是子View期望的大小（如layout_width="100dp"，getChildMeasureSpec（）函数就会返回子View的尺寸）；而AT_MOST模式时，SpecSize就变成父View剩余最大尺寸，子View需要自己计算该模式下其自身的尺寸，并且不要超过父View的剩余最大尺寸（超过也不显示，没有意义），关于SpecMode该如何取舍由View的的静态方法resolveSize(size，measureSpec)来辅助计算。

- **getMeasureWidth() 和getWidth()的区别？**

  measureWidth是子View自己测量得到的结果，而width是父View根据子View测量的结果再计算得到子View的真正宽度。
  
  如果layout函数传入的right-left=measureWidth，则两个值相等

------

#### Layout

##### **布局流程**

1. 计算**减去控件内边距**的parentLeft、parentTop、parentRight、parentBottom起始位置
2. 遍历子View，依据控件摆放逻辑，计算**含子View外边距**的childLeft、childRight、childTop、childBottom起始位置
3. 调用child.layout(childLeft、childRight、childTop、childBottom)将边界传给子View。

##### 要点

1. 只见控件摆放child，其自身的摆放在什么时候执行？

   onLayout()函数执行前父View就已经为自己摆放完毕，具体的顶点坐标保存在layout函数中setFrame()方法

2. 一定要记得重写generateLayoutParams()和generateDefaultLayoutParams()，否则layout_margin属性将会失效

   ```java
    override fun generateLayoutParams(attrs: AttributeSet?): LayoutParams {//xml加载时调用
           return MarginLayoutParams(context, attrs)
       }
   
    override fun generateDefaultLayoutParams(): LayoutParams {//代码addView加载时调用
           return MarginLayoutParams(MATCH_PARENT, MATCH_PARENT)
       }
   ```

------


#### View

##### API

- ```java
   void setWillNotDraw(boolean willNotDraw)
  ```

  当View不需要绘制自身时，设置true来性能优化，一般ViewGroup可以设置

------

#### SurfaceView

>**SurfaceView的双缓冲技术**
>
>这种双缓冲技术需要两个图形缓冲区，其中一个被称为**前端缓冲区**，另一个被称为**后端缓冲区**。前端缓冲区对应当前屏幕正在显示的内容(第一次初始化黑屏原因)，而后端缓冲区是接下来要渲染的图形缓冲区。我们通过surfaceHolder.lockCanvas （）函数获得的缓冲区是后端缓冲区（Bitmap）。当绘图完成以后，调用surfaceHolder. unlockCanvasAndPost( mCanvas）函数将后端缓冲区与前端缓冲区交换，后端缓冲区变成前端缓冲区，将内容显示在屏幕上；而原来的前端缓冲区则变成后端缓冲区，等待下一次 surfaceHolder.lockCanvas （）函数调用返回给用户使用，如此往复。(三缓冲也是这个原理)

##### API

- ```java
  Canvas lockCanvas()
  ```

  该函数作用是获取控件大小的后端缓冲区画布，画布上的内容全部显示到屏幕。画布绘制的内容在前后缓冲区交换过程不会清空。

- ```java
  Canvas lockCanvas(Rect dirty)
  ```

  > 1. lockCanvas()的脏区为控件大小
  >
  > 2. 当缓冲区刚初始化未绘制任何内容的时候，该函数返回的脏区是整个SurfaceView大小。因此每个缓冲区第一次绘制前都要进行“清屏”——绘制点什么，但又不影响结果。“清屏”代码如下：
  >
  >    ```java
  >    private fun clearSurface(holder: SurfaceHolder) {
  >        while (true) {//保证拿到每个Surface都先执行一遍
  >            val canvas = holder.lockCanvas(Rect(0, 0, 1, 1))
  >            if (width == canvas.clipBounds.width() && height == canvas.clipBounds.height()) {//说明未绘制过
  >                canvas.drawPaint(Paint().apply {
  >                    xfermode = PorterDuffXfermode(PorterDuff.Mode.CLEAR)
  >                })
  >                holder.unlockCanvasAndPost(canvas)//清屏 提交
  >            } else {//说明所有Surface都已清屏
  >                holder.unlockCanvasAndPost(canvas)
  >                break
  >            }
  >        }
  >    }
  >    ```

------

#### [Matrix](https://www.gcssloop.com/customview/Matrix_Basic)

**矩阵加减法**

> 两个矩阵中，相同位置元素相加减得到新的矩阵。**前提是这两个矩阵行列必须相同**。
>
> 满足交换律和结合律——A+B=B+A；(A+B)+C=A+(B+C)



**矩阵乘法**

> 两个矩阵AB相乘，**前提A的列数必须等于B的行数**。

![image-20210308161138360](pic\image-20210308161138360.png)
因此 A * B ≠ B * A

##### Android 3x3 变换矩阵

- Matrix元素：

![image-20210308175900117](pic\image-20210308175900117.png)

- 当执行旋转（Rotate）变换时，坐标元素：

![image-20210308234121877](pic\image-20210308234121877.png)

- 单位矩阵（初始状态）：
![image-20210308175748030](pic\image-20210308175748030.png)

  **单位矩阵 U与其他矩阵A相乘满足：(U * A = A * U)= A**

- 坐标计算（每一个像素点的坐标）：
  ![](pic\2086682-0f975e56a75eca12.webp)
  计算方法

  ```xml
  X1 = a * X + b * Y + c
  Y1 = d * X + e * Y + f
  L  = g * X + h * Y + i
  //可以见到 除了平移，缩放、错切都是相乘关系。
  ```
##### API

- ```kotlin
  reset()//重置
  setXXX()//设置并覆盖原有的Matrix元素
  preXXX()//矩阵的右乘 M.preXXX = M * O
  postXXX()//矩阵左乘 M.postXXX = O * M
  ```

------

#### Camera 3D效果

**坐标系**

![这里写图片描述](pic\20160518114718678)

并且旋转的方向是每个轴方向`逆时针`。（Canvas除了·Z轴是顺时针，X和Y轴是逆时针。。。）

------

### 自定义View事件篇

#### 事件传递核心原理

> 大致流程：`ViewGroup A` 嵌套`View B`。
>
> 1. 事件由A传给B，这之间A有权对事件拦截。
> 2. A若开始拦截事件（无论任何事件类型），则B永远不会再收到事件（除了cancel）；A若不拦截，则事件交给B处理。
> 3. B若处理了事件（`B.dispatchTouchEvent return true`），则事件传递：A -> B；若B没有处理，则事件传递：A -> B ->A，交给A.`onTouchEvent`

**结合开发场景分析事件传递机制**

**A不拦截，看B是否消费事件**

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

2. **A要拦截事件**

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

#### 开发注意点

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

#### ScrollBy和ScrollTo

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









