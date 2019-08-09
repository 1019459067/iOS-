# 一、简要说一下常用的动画库。


我之前在动画方面用的较少。

最常用的是 `FaceBook` 的 [Pop](https://github.com/facebook/pop)。

目前 `AirBnb` 的 [Lottie](https://github.com/airbnb/lottie-ios) 也很好，感兴趣可以去看看

# 二、请说一下对 CALayer 的认识。

`layer` 层是涂层绘制、渲染、以及动画的完成者，它无法直接的处理触摸事件（也可以捕捉事件）

`layer` 包含的方面非常多，常见的属性有 `Frame`、`Bounds`、`Position`、`AnchorPoint`、`Contents` 等等。

想详细了解 `CALayer` 以及动画的，可以看看这本书 - [Core-Animation](https://legacy.gitbook.com/book/zsisme/ios-/details)

# 三、`CALayer`  的 `Contents` 有几下几个主要的属性：

###### ContentsRect

单位制（0 - 1），限制显示的范围区域

###### ContentGravity

类似于 ContentMode，不过不是枚举值，而是字符串

###### ContentsScale

决定了物理显示屏是 几@X屏

###### ContentsCenter

跟拉伸有关的属性
