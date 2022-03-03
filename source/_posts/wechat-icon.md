---
title: 微信小程序Icon图标的几个实现方案
date: 2022-03-02 18:41:21
categories: Wechat
tags:
  - Icon
---

## 小程序Icon图标的几种实现方案

### 小程序原生提供的[Icon组件 ](https://developers.weixin.qq.com/miniprogram/dev/component/icon.html)

##### 属性

| 属性  | 类型          | 默认值 | 必填 | 说明                                                         |
| ----- | ------------- | ------ | ---- | ------------------------------------------------------------ |
| type  | string        |        | 是   | icon的类型，有效值：success, success_no_circle, info, warn, waiting, cancel, download, search, clear |
| size  | number/string | 23     | 否   | icon的大小                                                   |
| color | string        |        | 否   | icon的颜色，同css的color                                     |

##### 说明

###### 组件size属性的长度单位默认为px，[2.4.0](https://developers.weixin.qq.com/miniprogram/dev/framework/compatibility.html)起支持传入单位(rpx/px)

1. PX 数值类型，默认使用，什么单位都不填，只写一个数值就可以
2. RPX（Responsive Pixel）屏幕自适应单位，他将屏幕分为750个单位，每个单位是1/750。

<img src="https://i.loli.net/2021/11/15/JFTBEusxnKaQ8D4.png" style="zoom:50%;" />

比方说：iphone6的屏幕宽度是350px，每个rpx就是0.5px。也就是说如果我们在iphone6机器上将size的值设置为60rpx，他与设置为30或者30px的效果是一样的。

###### 组件color属性是改变图标所有像素点的颜色

#### 常见问题

##### 图标与文本能否放在同一行中？

可以的，图标本身就是为了更好的布局和更方便使用而诞生的。代码如下：

```html
<view style="font-size: 17px;margin-top: 20px;">
    我是一行文字，<icon type="success" size="15"></icon>我里面包含了图标！
</view>
```

##### 有时候真机上Icon显示空白

首先此问题肯定不是由于字体文件链接没有加入小程序的安全域名，WXSS加载图片及字体允许外域！如果图标是自定义实现的，要检查一下机型及嵌入的字体文件类型，可能是兼容性引起的，在小程序中推荐使用TTF和WOFF格式的字体。如果使用的是这两种字体，情况依然存在，可以考虑换SVG格式的数据嵌入。

##### weui组件库里的icon组件的图标如何取出来，保存在本地？

直接可以打开[weui官网](https://weui.io/)，然后通过浏览器开发者工具查看源码，找到资源地址然后下载。或者在[微信的官方文档上下载](https://developers.weixin.qq.com/miniprogram/design/#%E5%9B%BE%E6%A0%87)。

##### 优点

开箱即用。

##### 缺点

只支持success, success_no_circle, info, warn, waiting, cancel, download, search, clear这几种类型，远远不能满足开发需求。

### 自定义实现图标 

#### 直接使用图片

##### 优点

简单粗暴，每个图标对应一个图片。

##### 缺点

1. 图片在文本中不方便布局。不方便修改颜色。
2. 图片不能升缩，放大之后会变虚、有锯齿。
3. 图片需要在本地或者网络上存储，这样将导致大量HTTP请求，使得页面加载速度变慢。
4. 使用起来不如图标只使用一个名称那么方便。

#### 使用精灵图

Sprite，连续图片集，以非重叠、最小化分布的方式排列成一张图片。每次使用的时候通过纵横显示的起始坐标及区域大小，以达到动态切换的效果。

<img src="https://i.loli.net/2021/11/15/7BH5gdkbLynrfM3.png" alt="2" style="zoom: 33%;" />

##### 使用示例

通过精灵图实现一个爆炸效果。图片大小为（650x650） px；所以每一个小图标大小为（130x130）px；这是css样式设置的width和height为130px的原因，也是js代码移动step设置为130的原因。js中left和top均为负数，这是由于这里不是显示图标的坐标，而是背景图片所要向左上方移动的距离。

**注意：在wxss中只可以使用网络图片，不能使用本地图片！**

代码如下：

```html
<!--icon.wxml-->
<view>
<icon class="sprite scale" style="background-position: {{left}} {{top}};"></icon>
</view>
```

```css
/* icon.wxss */
.sprite{
    display: block;
    width: 130px;
    height: 130px;
    background: url("https://i.loli.net/2021/11/15/7BH5gdkbLynrfM3.png") no-repeat;
}
.scale{
    transform-origin: 0 0 0;
    transform: scale(2,2);
}
```

```javascript
// icon.js
Page({

    /**
     * 页面的初始数据
     */
    data: {
        left:'0px',
        top:'0px',
    },

    /**
     * 生命周期函数--监听页面加载
     */
    onLoad: function () {
        var that = this;
        var left = 0;
        var top = 0;
        const step = 130;
        const stop = (650-130);
        var i = setInterval(function() {
             if (left >= stop && top >=stop) {
                  clearInterval(i)
             } else {
                left += step;
                if(left >= 650){
                    left = 0;
                    top += step;
                }
                that.setData({
                    left: '-' + left +'px',
                    top: '-' + top +'px'
                  })
             }
        }, 100)
    },
})
```



##### 优点

1. 在加载的时候，只加载一次。减少了HTTP请求。

#### 使用CSS样式绘制

##### 使用示例

```html
<view>
    <icon class="icon-close"></icon>
</view>
```

```css
.icon-close{
    display: inline-block;
    width: 17px;
    height: 2px;
    background: red;
    transform: rotate(45deg);
}
.icon-close::after {
    content: '';
    display: block;
    width: 17px;
    height: 2px;
    background: red;
    transform: rotate(-90deg);
}
```



##### 缺点

1. 每个图标都需要写CSS样式代码，劳动量大。
2. 这种图标不是字符，每个图标在绘制时要统一一个中心点，不然使用起来控制位置会比较麻烦。
3. 大小与颜色也不方便控制。所以这并不是一种好的图标方案。

#### 使用矢量字体 （推荐使用）

当浏览器渲染一个字符的时候，首先看font-family样式，确定使用字体名是哪一个。接着以此字符的Unicode在字体文件里查找对应的字符信息。

字体类型有两种，一种是点阵字体，一种是矢量字体。现在使用最广泛的是矢量字体。矢量字体大概分成三类：Adobe主导的Type1、Apple和Microsoft主导的TrueType、Adobe，Apple和Microsoft共同主导的开源字体OpenType。

在矢量字体里面每个Unicode只是每个字符的一个索引，每个字符描述信息是一个几何矢量绘图描述信息。以Type1为例，它使用三次贝塞尔曲线来绘制字形。TrueType则使用二次贝塞尔曲线描述字形。正是由于矢量字体是绘制出来的，所以它可以实时填充任何颜色，并且可以无极缩放而没有锯齿。

[阿里巴巴的图标网站](https://iconfont.cn/)，我们可以在此网站上搜索到任何图片在线编辑，并下载样式文件，在小程序里面使用。

字体源说明：

1. EOT是微软IE浏览器专用的OpenType字体类型。
2. TTF是TrueType字体。
3. WOFF与WOFF2是移动开发专用的矢量字体格式。是对三种矢量字体格式的再封装。

链接各种字体文件源可以兼容不同浏览器宿主环境。浏览器会选择自己支持的格式，从列表中的第一个开始尝试加载。一旦获得一个可以使用的，就不会再加载剩下的字体格式了。小程序里面建议使用TTF和WOFF这两个格式。WOFF2在低版本的IOS设备上会有不兼容的问题。

##### 使用示例可以参考[此文章](https://www.jianshu.com/p/25db60f77531)

#### 使用SVG矢量文件

很多作图软件都可以导出SVG格式的矢量文件，比方说[Sketch](https://www.sketch.com/)，但是它导出的SVG格式的矢量文件有没有用的垃圾信息。可以到[阿里巴巴的图标网站](https://iconfont.cn/)编辑好之后下载SVG格式的矢量文件，它不带什么垃圾信息。然后我们拿这个文件找一个Image2base64工具，将文件内容转化为base64的字符串。然后就可以在小程序里使用这个base64的字符串作为图片源，实现自定义图标了。

##### 示例

1. 准备SVG图片

<svg t="1636971106558" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="1338" width="200" height="200"><path d="M326.2 429.7L710 428 525.4 892.1z" fill="#83A4FF" p-id="1339"></path><path d="M370.2 271.1l292.4 2.6 51.7 113-379.5-2.6z" fill="#FF7E71" p-id="1340"></path><path d="M296.1 380.7L64.9 284.1l124.2-92.3 148.4 76.7z" fill="#A4BEFF" p-id="1341"></path><path d="M64.9 330.5L284 428l235.5 460.6zM755.6 427.1L960.9 321 528.8 886z" fill="#5B79FB" p-id="1342"></path><path d="M751.3 379.8l-57.8-119 132-73.4 113.8 95.8z" fill="#A4BEFF" p-id="1343"></path><path d="M365.8 233.4l-50-12.9-94-52.7 110.4-39.6h360.6l109.5 45.7-105.2 50.9-35.4 8.6z" fill="#C7D8FF" p-id="1344"></path></svg>



2. 使用[线上Image2base64](https://www.sojson.com/image2base64.html)转换图片为：

   ```svg
   data:image/svg+xml;base64,PD94bWwgdmVyc2lvbj0iMS4wIiBzdGFuZGFsb25lPSJubyI/PjwhRE9DVFlQRSBzdmcgUFVCTElDICItLy9XM0MvL0RURCBTVkcgMS4xLy9FTiIgImh0dHA6Ly93d3cudzMub3JnL0dyYXBoaWNzL1NWRy8xLjEvRFREL3N2ZzExLmR0ZCI+PHN2ZyB0PSIxNjM2OTcwNTk4NjAyIiBjbGFzcz0iaWNvbiIgdmlld0JveD0iMCAwIDEwMjQgMTAyNCIgdmVyc2lvbj0iMS4xIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHAtaWQ9IjI2MDAiIHhtbG5zOnhsaW5rPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L3hsaW5rIiB3aWR0aD0iMjAwIiBoZWlnaHQ9IjIwMCI+PGRlZnM+PHN0eWxlIHR5cGU9InRleHQvY3NzIj48L3N0eWxlPjwvZGVmcz48cGF0aCBkPSJNMzI2LjIgNDI5LjdMNzEwIDQyOCA1MjUuNCA4OTIuMXoiIGZpbGw9IiM4M0E0RkYiIHAtaWQ9IjI2MDEiPjwvcGF0aD48cGF0aCBkPSJNMzcwLjIgMjcxLjFsMjkyLjQgMi42IDUxLjcgMTEzLTM3OS41LTIuNnoiIGZpbGw9IiNGRjdFNzEiIHAtaWQ9IjI2MDIiPjwvcGF0aD48cGF0aCBkPSJNMjk2LjEgMzgwLjdMNjQuOSAyODQuMWwxMjQuMi05Mi4zIDE0OC40IDc2Ljd6IiBmaWxsPSIjQTRCRUZGIiBwLWlkPSIyNjAzIj48L3BhdGg+PHBhdGggZD0iTTY0LjkgMzMwLjVMMjg0IDQyOGwyMzUuNSA0NjAuNnpNNzU1LjYgNDI3LjFMOTYwLjkgMzIxIDUyOC44IDg4NnoiIGZpbGw9IiM1Qjc5RkIiIHAtaWQ9IjI2MDQiPjwvcGF0aD48cGF0aCBkPSJNNzUxLjMgMzc5LjhsLTU3LjgtMTE5IDEzMi03My40IDExMy44IDk1Ljh6IiBmaWxsPSIjQTRCRUZGIiBwLWlkPSIyNjA1Ij48L3BhdGg+PHBhdGggZD0iTTM2NS44IDIzMy40bC01MC0xMi45LTk0LTUyLjcgMTEwLjQtMzkuNmgzNjAuNmwxMDkuNSA0NS43LTEwNS4yIDUwLjktMzUuNCA4LjZ6IiBmaWxsPSIjQzdEOEZGIiBwLWlkPSIyNjA2Ij48L3BhdGg+PC9zdmc+
   ```

   3. 编写代码

   ```css
   .svg-icon{
       display: block;
       width: 200px;
       height: 200px;  
       background-repeat: no-repeat;
       background: url("data:image/svg+xml;base64,PD94bWwgdmVyc2lvbj0iMS4wIiBzdGFuZGFsb25lPSJubyI/PjwhRE9DVFlQRSBzdmcgUFVCTElDICItLy9XM0MvL0RURCBTVkcgMS4xLy9FTiIgImh0dHA6Ly93d3cudzMub3JnL0dyYXBoaWNzL1NWRy8xLjEvRFREL3N2ZzExLmR0ZCI+PHN2ZyB0PSIxNjM2OTcwNTk4NjAyIiBjbGFzcz0iaWNvbiIgdmlld0JveD0iMCAwIDEwMjQgMTAyNCIgdmVyc2lvbj0iMS4xIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHAtaWQ9IjI2MDAiIHhtbG5zOnhsaW5rPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L3hsaW5rIiB3aWR0aD0iMjAwIiBoZWlnaHQ9IjIwMCI+PGRlZnM+PHN0eWxlIHR5cGU9InRleHQvY3NzIj48L3N0eWxlPjwvZGVmcz48cGF0aCBkPSJNMzI2LjIgNDI5LjdMNzEwIDQyOCA1MjUuNCA4OTIuMXoiIGZpbGw9IiM4M0E0RkYiIHAtaWQ9IjI2MDEiPjwvcGF0aD48cGF0aCBkPSJNMzcwLjIgMjcxLjFsMjkyLjQgMi42IDUxLjcgMTEzLTM3OS41LTIuNnoiIGZpbGw9IiNGRjdFNzEiIHAtaWQ9IjI2MDIiPjwvcGF0aD48cGF0aCBkPSJNMjk2LjEgMzgwLjdMNjQuOSAyODQuMWwxMjQuMi05Mi4zIDE0OC40IDc2Ljd6IiBmaWxsPSIjQTRCRUZGIiBwLWlkPSIyNjAzIj48L3BhdGg+PHBhdGggZD0iTTY0LjkgMzMwLjVMMjg0IDQyOGwyMzUuNSA0NjAuNnpNNzU1LjYgNDI3LjFMOTYwLjkgMzIxIDUyOC44IDg4NnoiIGZpbGw9IiM1Qjc5RkIiIHAtaWQ9IjI2MDQiPjwvcGF0aD48cGF0aCBkPSJNNzUxLjMgMzc5LjhsLTU3LjgtMTE5IDEzMi03My40IDExMy44IDk1Ljh6IiBmaWxsPSIjQTRCRUZGIiBwLWlkPSIyNjA1Ij48L3BhdGg+PHBhdGggZD0iTTM2NS44IDIzMy40bC01MC0xMi45LTk0LTUyLjcgMTEwLjQtMzkuNmgzNjAuNmwxMDkuNSA0NS43LTEwNS4yIDUwLjktMzUuNCA4LjZ6IiBmaWxsPSIjQzdEOEZGIiBwLWlkPSIyNjA2Ij48L3BhdGg+PC9zdmc+");
   }
   
   ```

   ```html
   <view>
       <icon class="svg-icon"></icon>
   </view>
   ```

   ##### 说明

   此种方法仍旧需要一张图片处理一次，然后在页面中引用。注意：样式文件中的width和height属性的值需要和下载的SVG文件的width和height保持一致的（在svg标签中可以看到）。

#### 使用Canvas绘制SVG绘制

这种绘制用于制作动画还是可以的，但是用来做图标有点大材小用了。[腾讯的将SVG绘制成图像的 Cax 引擎](https://developers.weixin.qq.com/community/develop/article/doc/000ca493bc09c0d03a8827b9b5b013)

