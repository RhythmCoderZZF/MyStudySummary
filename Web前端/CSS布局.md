# CSS布局

## 盒子模型

![image-20210317152847409](pic\image-20210317152847409.png)

**Block块**

1. `padding`、`border`、`margin`是**另外叠加**到宽高。
2. 子元素`margin`会并到父元素的`margin`上，而`border`、`padding`只会叠加自己。。。

**Inline**

1. `padding`、`border`、`margin`在水平方向叠加至宽高，**垂直方向失效**（类似于outline，只会显示但不会排挤其他元素）

## Flex布局

<img src="pic\image-20210318141654601.png" alt="image-20210318141654601" style="zoom:67%;" />

**flex-container**

- flex-direction：`flex-item`沿`main axis`从`main start`到`main end`方向排布
  <img src="pic\image-20210318142336634.png" alt="image-20210318142336634" style="zoom: 33%;" />

- flex-wrap：默认情况下，`flex -item`都排在一条线（又称"轴线"）上。`flex-wrap`属性定义，如果一条轴线排不下，如何换行。

- justify-content：决定`flex-item`在`main caxis`上的对齐方式

  <img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Web前端\pic\image-20210318153211353.png" alt="image-20210318153211353" style="zoom: 25%;" />

- align-items：决定`flex-item`在`cross axis`上的对齐方式

- align-content：`align-content`属性定义了多根轴线的对齐方式（flex-item超过一行）。如果项目只有一根轴线，该属性不起作用。

- flex-flow：`flex-flow`属性是`flex-direction`属性和`flex-wrap`属性的简写形式，默认值为`row nowrap`。

**flex-item**

- `order`：`order`属性定义项目的排列顺序。数值越小，排列越靠前，默认为0。

- `flex-grow`：定义`flex-item`的放大比例，默认为`0`，即如果存在剩余空间，也不放大。

  <img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Web前端\pic\image-20210318145356450.png" alt="image-20210318145356450" style="zoom:33%;" />

- `flex-shrink`:定义`flex-item`的缩小比例，默认为1，即如果空间不足，该项目将缩小。

  <img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Web前端\pic\image-20210318150247326.png" alt="image-20210318150247326" style="zoom:33%;" />

- `align-self`：`align-self`属性允许单个项目有与其他项目不一样的对齐方式，可覆盖`align-items`属性。默认值为`auto`，表示继承父元素的`align-items`属性，如果没有父元素，则等同于`stretch`。

  <img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Web前端\pic\image-20210318145213594.png" alt="image-20210318145213594" style="zoom: 33%;" />

- `flex-basis`：`flex-basis`属性定义了在分配多余空间之前，项目占据的主轴空间（main size）。浏览器根据这个属性，计算主轴是否有多余空间。它的默认值为`auto`，即项目的本来大小。

- `flex`：`flex`属性是`flex-grow`, `flex-shrink` 和 `flex-basis`的简写，默认值为`0 1 auto`。后两个属性可选。



## API

- outLine：轮廓
- box-shadow：阴影

