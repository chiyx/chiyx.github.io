---
title: java2d合图系列-绘制圆形图像图层
date: 2017-03-12 23:38:30
tags: image-process
categories: java2d
---

在当前潮流的趋势中，对于用户头像的展示一般都是设计成圆形图层的展现形式，如果是在网页上进行展示，我们可以通过css3的将头像包在一个方形的**div**中，然后设置 **border-radius**的值为头像的边长相等便可。

```css
div {
    width: 92px;
    height: 92px;
    border-radius: 92px;
}
```

但是如果是在图片合成的时候，需要将头像合到一个预先挖好一个圆坑的底图中，又该如何做呢? 比如有如下的底图和用户头像:

![image](/images/avatar_re.png)

![image](/images/mytaobao_avart.jpeg)

现在需要将后面的头像图片替换掉底图中被框起来的部分，采用的步骤基本如下:

1. 将头像缩放到底图中的头像框大小
2. 将缩放后的头像进一步变成圆形图片
3. 将图像头像覆盖到底图的头像框上

抛开缩放和覆盖(即打水印)不表，那么问题来了：如何将方形的图像变为一个圆形图像呢？

<!--more-->

## 思路一: 将矩形头像超出其内接椭圆的部分都设置为透明即可在视觉上营造出椭圆型图像的概念

> 正方形和圆形分别为长方形和椭圆的特例，为更好的适配不同场景，在处理上应该采用更广的定义

1. 将原头像转化为支持透明度设置的图片格式如: png
2. 设计一个代表对应图像矩形框的内接椭圆的结构，其有一个 `contain` 方法，用于判断某个点是否处于其中.
3. 遍历图像的各个像素，如果没有落在椭圆内则将其透明度设置为0

具体`scala`代码如下:

```java
val (width, height) = (image.getWidth, image.getHeight)
// 1. 如果源图不支持透明度如：jpg，则将其重绘到一个支持透明的画布上
val target = 
    if (image.getTransparency == Transparency.TRANSLUCENT) image
    else {
       val tmp = new BufferedImage(width, height, BufferedImage.TYPE_INT_ARGB)
       tmp.createGraphics().drawImage(image, 0, 0, null)
       tmp
    }
// 2. 内接椭圆, java2d已为我们备好的大餐，省了很多回顾几何的麻烦
val ellipse = new Ellipse2D.Double(0, 0, width, height)
// 3. 遍历所有像素点，如果不在椭圆里，将该像素设置为全透明
0 until width foreach { x =>
    0 until height foreach { y =>
        if (!ellipse.contains(x, y)) {
            val rgb = target.getRGB(x, y)
            target.setRGB(x, y, rgb & 0x00ffffff)
        }
    }
}
// 返回新图像
target
```

效果如下:

![image](/images/mohe_avatar_ellipse_2.png)

可以发现得到的圆形图像的边缘很不光滑，毛刺很多，这是因为在内接椭圆的边缘划分其实是浮点值，无法直接映射到像素点上，
这就造成了在边缘点的处理上并不均匀，导致毛刺。

## 思路二: 在一个透明画布上将画笔设置为头像图片，用该画笔在画布上绘制一个椭圆

> 因为java2d在绘制各类图形时，会有一些边缘光滑处理的算法，只要我们指定特定的绘制hints就行

1. 创建一个源图像同等大小的透明画布
2. 设置绘制时的边缘处理的hints
3. 将原图像设置为该画布的画笔，这样在绘制图形时就会使用该图像对应像素点的rgba值
4. 在画布中绘制一个内接椭圆

具体`scala`代码如下:

```java
val (width, height) = (image.getWidth, image.getHeight)
// 创建画布
val target = new BufferedImage(width, height, BufferedImage.TYPE_INT_ARGB)
// 设置hints
val rh = new RenderingHints(
  RenderingHints.KEY_ANTIALIASING,
  RenderingHints.VALUE_ANTIALIAS_ON)
rh.put(RenderingHints.KEY_RENDERING,
  RenderingHints.VALUE_RENDER_QUALITY)
val g = target.createGraphics()
g.setRenderingHints(rh)
// 设置画笔
g.setPaint(new TexturePaint(image, new Rectangle(0, 0, width, height)))
// 绘制内接椭圆
g.fill(new Ellipse2D.Double(0, 0, width, height))
target
```

效果如下:

![image](/images/mohe_avatar_ellipse_1.png)

可以看到这次得到的圆形图像边缘已经如丝般柔滑，Done！