![Android 图形处理 —— Matirx 方法详解及应用场景](assets/02b849040dcd4b2aaab2bcae0b9878f9~tplv-k3u1fbpfcp-zoom-crop-mark:1304:1304:1304:734.awebp)

> 摘要：上一篇文章《Matrix 原理剖析》 介绍了 Matrix 的基础原理，本文介绍 Matrix 一些常用方法以及具体的使用场景

上一篇文章[《Matrix 原理剖析》](https://juejin.cn/post/7038557734517604382) 介绍了 Matrix 的基础原理，本文介绍 Matrix 一些常用方法以及具体的使用场景

## Matrix 方法详解

文中部分内容及图片参考自：https://blog.csdn.net/gb702250823/article/details/53526149

| 方法类别 | 相关 API                                                   | 摘要                           |
| -------- | ---------------------------------------------------------- | ------------------------------ |
| 数值操作 | set、reset、setValues、getValues                           | 设置、重置、设置数值、获取数值 |
| 数值计算 | mapPoints、mapRadius、mapRect、mapVectors                  | 计算变换后的数值               |
| 设置     | setConcat、setRotates、setScale、setSkew、setTranslate     | 设置变换                       |
| 前乘     | preConcat、preRotate、preScale、preSkew、preTranslate      | 前乘变换                       |
| 后乘     | postConcat、postRotate、postScale、postSkew、postTranslate | 后乘变换                       |
| 特殊方法 | setPolyToPoly、setRectToRect                               | 一些特殊操作                   |

### 数值操作

#### void set(Matrix src)

深拷贝一份 `src` 中的数据到当前 Matrix 中, 如果 `src` 为 `null`, 则相当于 `reset`

#### void reset()

将当前 Matrix 重置成一个单位矩阵

#### void setValues(float[] values)

将一组浮点数值的前 9 位数据拷贝到 Matrix 中，如果数组长度小于 9，调用该方法会抛出异常

#### void getValues(float[] values)

从 Matrix 中拷贝数据到 `values` 浮点数组中

### 数值计算

`mapXXX` 有很多签名版本，具体区别可以参考源码注释，这里据其中一个常用的签名方法作为说明，下同

#### void mapPoints(float[] dst, float[] src)

把当前 Matrix 应用到 `src` 所指示的所有坐标上，然后将变换后的坐标复制到 `dst` 数组上

数组中每两个相邻的点表示一个坐标`（x,y）`，因此数组长度一般都是偶数，否则最后一个数值不参与计算

#### float mapRadius(float radius)

把当前 Matrix 应用到半径为 `radius` 所指示的圆上，然后返回变换之后的圆的半径，由于圆可能会因为画布变换变成椭圆，所以此处测量的是平均半径

#### boolean mapRect(RectF dst, RectF src)

和 `mapPoints` 类似，把当前 Matrix 应用到 `src` 所指示的四个顶点上，然后将变换后的四个顶点值写入 `dst` 中，返回值是判断矩形经过变换后是否仍为矩形

#### void mapVectors(float[] dst, float[] src)

和 `mapPoinst`、`mapRect` 类似，这里测试计算的是向量，不再赘述

### 设置

#### setRotates、setScale、setSkew、setTranslate

这几个方法比较好理解，就是将变换设置给当前 Matrix，得到一个变换后的 Matrix 

#### boolean setConcat(Matrix a, Matrix b)

相当于计算两个矩阵相乘 `a × b`，并将结果赋值给当前矩阵，即 `c.setConcat(a, b)` 表示 `c = a ×  b`

### 前乘后乘

这两类计算也比较好理解，就是对应了数学中的矩阵前乘和后乘

### 特殊方法 

#### boolean setPolyToPoly

```java
boolean setPolyToPoly (
        float[] src, 	// 原始顶点数组 src [x, y]
        int srcIndex, 	// 原始顶点数组开始位置
        float[] dst, 	// 目标顶点数组 dst [x, y]
        int dstIndex, 	// 目标顶点数组开始位置
        int pointCount)	// 测控顶点的数量 取值范围是: 0 到 4
```

`Poly` 全称是 `Polygon`，多边形的意思。

调用这个方法后，会计算从原始顶点和到目标顶点的变换（意味着 `src` 和 `dst` 要一一对应），把这种变换信息存储到当前 Matrix 中；将得到 的 Matrix 应用到任意图形上，可以实现把这个图形进行 Matrix 所表示的形状变换，效果如下图所示

![](assets/cedd2c5ba34e4307a9b3b181fea7d507~tplv-k3u1fbpfcp-zoom-1-20220330113225043.image)

其中 `pointCount` 不同数值也代表了不同能力的变换，一般我们最常用的 `pointCount` 都是 4，0~3 我们都可以用更加简单的方法来代替实现

| pointCount | 摘要                                        |
| ---------- | ------------------------------------------- |
| 0          | 相当于 **reset**                            |
| 1          | 相当于 **Translate**                        |
| 2          | 可以进行 缩放、旋转、平移 变换              |
| 3          | 可以进行 缩放、旋转、平移、错切 变换        |
| 4          | 可以进行 缩放、旋转、平移、错切以及任何形变 |

**测控点的选取**

测控点可以选择任何你认为方便的位置，只要 `src` 与`dst`一一对应即可。不过为了方便，通常会选择一些特殊的点： 图形的四个角，边线的中心点以及图形的中心点等。不过有一点需要注意，测控点选取都应当是不重复的 (`src` 与 `dst` 均是如此)，如果选取了重复的点会直接导致测量失效，这也意味着，你不允许将一个方形 (四个点) 映射为三角形(四个点，但其中两个位置重叠)，但可以接近于三角形。

**作用范围**

作用范围是设置了 Matrix 的全部区域，如果你将这个 Matrix 赋值给了 Canvas，它的作用范围就是整个画布，如果你赋值给了 Bitmap，它的作用范围就是整张图片

####  boolean setRectToRect(RectF src, RectF dst, ScaleToFit stf)

了解了 `setPolyToPoly` 之后，这个方法就比较好理解，简单来说就是将源矩形的内容填充到目标矩形中，然而在大多数的情况下，源矩形和目标矩形的长宽比是不一致的，到底该如何填充呢，这个填充的模式就由第三个参数 `stf` 来确定

`ScaleToFit` 是一个枚举类型，共包含了四种模式:

| 模式   | 效果                                                         |
| ------ | ------------------------------------------------------------ |
| CENTER | 居中，对 `src` 等比例缩放，并最大限度的填充变换后的矩形，将其居中放置在 `dst` 中 |
| START  | 顶部，对 `src` 等比例缩放，并最大限度的填充变换后的矩形，将其放置在 `dst` 的左上角，左上对齐 |
| END    | 底部，对 `src` 等比例缩放，并最大限度的填充变换后的矩形，将其放置在 `dst` 的右下角，右下对齐 |
| FILL   | 充满，拉伸 `src` 的宽和高，使其完全填充满 `dst`              |

一图胜千言：

![img](assets/f95e22af951a40e49cd69a593c20d0dc~tplv-k3u1fbpfcp-zoom-1-20220330113224744.image)

## Matrix 在 Android 中的使用场景

其实我们日常开发中或多或少已经接触了 Matrix，只是大部分我们都还不知道，比如我们使用的 `ImageView` 的 `ScaleType`，实际上内部就是通过 Matrix 实现的

<img src="assets/a22652fe585e4a2787f943b8b6f652e5~tplv-k3u1fbpfcp-zoom-1-20220330113224752.image" alt="image-20211206173802202" style="zoom:50%;" />

除此之外，Matrix 的应用十分广泛，这里没法一一列举，大家在学习了基本原理和常用 api 之后可以自行想象和实践，网上也有很对 Matrix 应用的有趣的例子。

这里笔者分享一下自己在实际开发中用到 Matrix 的例子 —— 相机扫描识别二维码

当我在开发这个功能的时候，遇到一个棘手的问题：当相机实时预览识别到二维码之后，需要将当前帧截取下来当成静态背景图，然后在识别到二维码的位置上显示一个小黄点，类似微信扫码的效果：

<img src="assets/04cd8cbbd9934e758fd0e5b87c70be3e~tplv-k3u1fbpfcp-zoom-1-20220330113224977.image" alt="image-20211206174645851" style="zoom: 25%;" />

这里首先需要介绍下，相机识别二维码的大致流程

相机取景框实时取景 -> 图像帧预处理（包括裁剪、灰度化等）-> 扫码 SDK 分析预处理后的帧图像 -> 识别到二维码，返回二维码信息（包括在图中的位置等） -> 将当前图像原始帧设置为背景图 -> 在图上二维码位置出绘制小黄点

![image-20211206175515558](assets/26d9d386b58a49118c339b66a96dc6cb~tplv-k3u1fbpfcp-zoom-1-20220330113224971.image)

由于 SDK 分析的是裁剪灰度化过后的图像，因此返回的二维码位置信息也是基于裁剪过后的坐标系，如果我们直接把这个坐标绘制在屏幕上，必然会发现二维码位置不对

因此这里就涉及到**坐标映射**： 我们需要把裁剪后的坐标映射回手机屏幕坐标

先看看我们当前有哪些数据：

1. 裁剪后的图像
2. 二维码位置信息，是一组顶点（上下左右四个位置的点 x,y ）
3. 取景框尺寸

我们可以分析出，这里发生了变化的是两个矩形：取景框和裁剪后的图像

![image-20211206180200549](assets/7c2a22bca39f491c850088665eac8a7c~tplv-k3u1fbpfcp-zoom-1-20220330113224748.image)

根据之前学到的内容，我们可以使用 `setPolyToPoly` 或者 `setRectToRect` 来描述这一变换，这里我们以 `setPolyToPoly` 举例，伪代码如下：

```kotlin
// 实例化一个原始矩阵（单位矩阵）
val matrix = Matrix()

// cropRect 是裁剪后的图像的矩形
val source = floatArrayOf(
    cropRect.left.toFloat(),
    cropRect.top.toFloat(),
    cropRect.right.toFloat(),
    cropRect.top.toFloat(),
    cropRect.right.toFloat(),
    cropRect.bottom.toFloat(),
    cropRect.left.toFloat(),
    cropRect.bottom.toFloat()
)

// previewView 是取景框视图
val destination = floatArrayOf(
    0f,
    0f,
    previewView.width.toFloat(),
    0f,
    previewView.width.toFloat(),
    previewView.height.toFloat(),
    0f,
    previewView.height.toFloat()
)

// 使用 setPolyToPoly 描述这种变换，得到一个矩阵 Matrix
// 这里默认裁剪后的图像没有旋转，否则还需要处理旋转
matrix.setPolyToPoly(source, 0, destination, 0, 4)
```

得到 Matrix 之后，我们就可以使用 mapPointsToPoints 来计算，按照此矩阵变换后的二维码位置坐标了

```kotlin 
val srcArray = it.position.toFloatArray() // 表示二维码原始坐标数据
val destArray = FloatArray(srcArray.size) // 表示计算出来的二维码坐标数据，即在相机取景框上的位置
matrix.mapPoints(destArray, srcArray)
```

这样，我们就可以实现准确地在相机屏幕上标出二维码的位置了
