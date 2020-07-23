![](https://user-gold-cdn.xitu.io/2020/5/12/17206afbd2c34d96?w=900&h=383&f=png&s=84376)

> 前沿：自从前端和后端分家之后，前后端接口对接就成为了家常，“谁”也离不开谁，而对接接口的过程就离不开接口文档，比较主流就是Swagger(强大的API文档工具),当然今天它不是主角，顶多也就是个辅助。这篇文章旨在梳理如何在前端项目中更好的去管理跟后端“对接”的接口

聊接口管理，离不开请求库，vue技术栈中请求库谈及最多的，非axios莫属，先让我们重新梳理下axios

### 1.axios 

> axios 是一个基于Promise 用于浏览器和 nodejs 的 HTTP 客户端，树酱挑了三个觉得特别好用的特征唠唠👇

#### 1.1支持取消请求 (cancelToken)
> 应用场景：当用户重新刷新数据请求的时候，如果你之前发起的请求列表还没有响应，这时候如果你重新发起请求，会出现二次请求的情况，可以通过cancelToken可以取消上一次请求 
> [使用文档](http://www.axios-js.com/zh-cn/docs/index.html#%E5%8F%96%E6%B6%88)


![](https://user-gold-cdn.xitu.io/2020/5/12/17206d838153fb1f?w=2516&h=1684&f=png&s=479223)
那么cancelToken是如何实现的，可以阅读下源码，源码链接 [点我](https://github.com/axios/axios/blob/405fe690f93264d591b7a64d006314e2222c8727/lib/cancel/CancelToken.js)
感兴趣的同学可以看这篇 [axios 之cancelToken原理](https://www.cnblogs.com/ysk123/p/11544211.html)


#### 2.支持Promise API（axios.all、axios.spread等）
> 应用场景：当我想同时发起多个请求时，axios.all类似于(promise.all）给予我很好的体验方式，解决了并发请求的应用场景

![](https://user-gold-cdn.xitu.io/2020/5/12/17206d27c928c409?w=3360&h=1468&f=png&s=450863)

#### 3.拦截器（拦截请求和返回）
> 应用场景：当一个项目中，多个接口需要前端通过header传用户ID、校验token等等时，我们可以统一添加，同理，当接口出现异常的状态码，如401（登录过期）需要重定向到登录页面时，我们需要统一添加处理，这时候拦截器就起到很重要的作用

![](https://user-gold-cdn.xitu.io/2020/5/12/17206ded6e3138c2?w=3196&h=2476&f=png&s=637658)

好了废话不多说，进入今天的主题：如何更好管理 Api 接口。

### 2.API 管理

#### 2.1 方式一：按模块封装方法
> 通过swagger文档定义的功能模块，来定义不同模块的service，封装接口增删改查等方法

- 按swagger接口文档的模块创建目录

![](https://user-gold-cdn.xitu.io/2020/5/12/17206ef96542948d?w=1210&h=344&f=png&s=25894)

- 编写模块方法（举个用户模块的例子）
> 这里用到了之前封装的kdutil库[github链接](https://github.com/littleTreeme/kdutil)中的http方法，本质上是对axios进行二次封装，通过不同的api操作来封装不同的请求方法

![](https://user-gold-cdn.xitu.io/2020/5/12/17206f498f1f8d3f?w=3568&h=1540&f=png&s=458965)

- 导出所有编写好的模块

当我们将不同模块对应的Swagger接口文档都封装完成之后，可以将各模块导出安装为插件的形式来挂载，模块导出使用的是webpack打包的require.context的方法，引入指定的路径下匹配到的模块引用，如下所示👇

![](https://user-gold-cdn.xitu.io/2020/5/12/17206f9e847a0d50?w=3164&h=2332&f=png&s=559621)

为了让这些模块在Vue中更好地直接使用，我们将导出的模块通过“挂在”Vue.prototype的形式注入到Vue组件中,以此来为Vue对象添加了一个原型属性，而不是一个全局变量。

这里涉及到vue插件的使用，vue 插件一般来用进行如下几种操作

- 添加全局方法或者 property。如：vue-custom-element

- 添加全局资源：指令/过滤器/过渡等。如 vue-touch

- 通过全局混入来添加一些组件选项。如 vue-router

- 添加 Vue 实例方法，通过把它们添加到 Vue.prototype 上实现。（上文使用的是这种操作）

- 一个库，提供自己的 API，同时提供上面提到的一个或多个功能。如 vue-router

Vue.js 的插件需要暴露一个 install 方法。这个方法的第一个参数是 Vue 构造器，第二个参数是一个可选的选项对象，上图解析出来如下所示

![](https://user-gold-cdn.xitu.io/2020/5/12/1720797c70c3f5c0?w=2792&h=1180&f=png&s=323484)

最后在main.js中通过全局方法 Vue.use() 使用插件如向下所示👇

![](https://user-gold-cdn.xitu.io/2020/5/12/17206fadf6dd2dbd?w=2052&h=964&f=png&s=224787)


- 如何在项目中调用

因为已经挂载在vue对象的原型上，可以使用`this.$api`去调模块

![](https://user-gold-cdn.xitu.io/2020/5/12/172070112d31abae?w=2420&h=964&f=png&s=241591)

> 聊到你可能疑惑就是，你这接口路径不对啊，怎么是相对路径呢？其实是在axios.create的时候就把路径写进去了,如下所示👇

![](https://user-gold-cdn.xitu.io/2020/5/12/172079c67fa018c9?w=3028&h=1324&f=png&s=373975)

而这个`process.env.VUE_APP_URL`又是什么玩意？
> 是通过不同环境（开发、测试、生产）定义的不同环境的配置文件（请求api、其他配置等等）具体可以看下树酱的 [《基于 Vue-cli 3x的项目部署》](https://juejin.im/post/5e040bb06fb9a016194b0ecd)的介绍


![](https://user-gold-cdn.xitu.io/2020/5/12/17207a2313409c15?w=2704&h=592&f=png&s=82285)


总结：这种方式优势在于可以很直接的辨别接口增删改查对应的方法，且挂载在vue对象原型中方便调用，一目了然，劣势在于重复代码还是偏多，接下来让我们一起看看下面的这种方式

#### 2.2 方式二. 按api文档编写API

> 上一节讲完的方式一，导出的本质上是方法，那方式二又是怎么样的一种形式，答案是导出配置文件

- 先“上才艺”，先给目录结构
> 通过在配置文件夹定义api，同理以不同模块拆分，下面举user模块这个例子说明

![](https://user-gold-cdn.xitu.io/2020/5/12/17207affd80c6478?w=908&h=222&f=png&s=14282)

- 按模块编写api

![](https://user-gold-cdn.xitu.io/2020/5/12/17207c1a957012a0?w=2320&h=1396&f=png&s=295929)
- 导出所有编写好的api配置
> 跟上一节导出模块一样，都是使用require.context，然后再结合Object.defindproperty方法来修改对象的属性，返回一个新的api路径

![](https://user-gold-cdn.xitu.io/2020/5/12/17207b318c737e08?w=3836&h=2332&f=png&s=700015)

- 拓展：Object.defineProperty

关于Object.defineProperty，这里也简单讲一下

> MDN介绍：直接在一个对象上定义一个新属性，或者修改一个对象的现有属性，并返回此对象。

Object.defineProperty对应的三个数值

- obj 要在其上定义属性的对象。
- prop要定义或修改的属性的名称。
-  descriptor将被定义或修改的属性描述符

举个例子如下👇
![](https://user-gold-cdn.xitu.io/2020/5/12/17207b8e733a5eaf?w=1492&h=681&f=png&s=151620)

我们可以看到descriptor中，也就是第三个参数中有个字段enumerable，叫描述对象的enumerable属性，我们称为”可枚举性“

那可枚举性和不可枚举性有什么区别？你看看下面这个例子应该就清楚了，如果是不可枚举则不显示，反之即可，也就是当enumerable为false，只返回给定对象的自身可枚举属性

![](https://user-gold-cdn.xitu.io/2020/5/12/17207ba8bc266378?w=1492&h=777&f=png&s=153780)
同样的下面几种方式也是同样的思路（只返回给定对象的自身可枚举属性）


![](https://user-gold-cdn.xitu.io/2020/5/12/17207bc7f795fe57?w=1492&h=706&f=png&s=195262)

一不小心又聊偏了，回归正题，当我们成功导出API配置文件后，接下来就是如何使用了

- 如何使用

将配置挂载到vue对象原型上

![](https://user-gold-cdn.xitu.io/2020/5/12/17207c5c08f69cd8?w=2320&h=892&f=png&s=216734)

正确调用姿势：
![](https://user-gold-cdn.xitu.io/2020/5/12/17207c4c4e4b9ca9?w=2624&h=1036&f=png&s=269975)

That's all Thank you，如果你有更好的方式请留下你宝贵的意见，非常感谢

> 树酱希望将前端的乐趣带给大家
> 本文已收录 <a href="https://github.com/littleTreeme/frontendThings " style="word-wrap: break-word; font-weight: bold; border-bottom: 1px solid #1e6bb8; border: none; text-decoration: none; color: #dda52d;">https://github.com/littleTreeme/frontendThings</a> 喜欢就star✨

往期文章

- [数据类型转换看这篇就够了](https://juejin.im/post/5eb2bef2f265da7b9a6f0e7e)
- [前端表单数据那些事](https://juejin.im/post/5eaa2cb3f265da7ba0581c84)
- [微前端那些事](https://juejin.im/post/5e83f8ad6fb9a03c5e0ccccc)
- [前端工程化那些事](https://juejin.im/post/5e999cecf265da47cd357a24)
- [从0到1开发工具库](https://juejin.im/post/5e958d0f6fb9a03c6675cb5d)
- [从0到1开发简单脚手架](https://juejin.im/post/5ea1263d6fb9a03c73799a25)
- [前端运维部署那些事](https://juejin.im/post/5e88904bf265da47f517837c)

![](https://user-gold-cdn.xitu.io/2020/4/25/171b06eba70a93fe?w=1280&h=467&f=png&s=223701)