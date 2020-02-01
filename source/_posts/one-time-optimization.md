---
title: 记一次性能优化，从六七秒优化到1.35s
categories: 前端
tag: js 性能优化
date: 2019-12-07
---

这是在还没毕业做的一个项目，一直放在之前的服务器上，不久之前服务器到期了。趁着双十一的机会，优惠很大就又买了一个，把项目迁移到新的服务器上。首页加载的速度有六七秒左右，以前也优化过一些，但以前技术有限，就没怎么管它了。这次趁着换新服务器，就想着来优化一下网站。`网速`、`电脑响应速度`等 都会对页面加载速度有影响，基本可以稳定在 1秒多(未使用CDN)，先上网站地址 [花间道](http://47.106.187.172/),下面从我所用到的优化方法介绍。


## 1. 减少http请求数

众所周知，减少 `http` 请求数是缩短页面加载时间最有效的方法
* 资源压缩和合并，尽可能的将外部的脚本、样式进行合并，多个合为一个。
* 使用 `CSS Sprites`，通过背景定位获取具体图像
* 合理的设置http缓存

### 1.1 合并压缩
合并压缩的构建工具有，`gulp`，`webpack`，`grunt`等，我当时使用的是gulp，gulp不能处理ES6代码，所以还是得用webpack，是否要合并代码还是要具体看自己的需求。这个就自己去查看webpack了。

下面是使用gulp处理less转化为css，并且会压缩代码
```js
gulp.task('less',function () {
    gulp.src('public/less/*.less')
    .pipe(gulp_less())
    .pipe(gulp_minify_css())
    .pipe(gulp.dest('public/stylesheets'))
});
```
### 1.2 CSS Sprites
推荐自动生成雪碧图的工具: [https://www.toptal.com/developers/css/sprite-generator](https://www.toptal.com/developers/css/sprite-generator)
webpack 中有一款生成雪碧图的插件，[webpack-spritesmith](https://www.npmjs.com/package/webpack-spritesmith),会自动帮你生成调用雪碧图的css样式

### 1.3 http缓存

http缓存分为 **强制缓存** 和 **协商缓存**

强制缓存在响应头中会有`（Expires/Cache-Control）`,Expires是1.0的东西，在http 1.1都是用Cache-Control代替了

Cache-Control
private 客户端可以缓存
public 客户端和代理服务器都可以缓存
max-age=60 缓存内容将在60秒后失效
no-cache 需要使用对比缓存验证数据,强制向源服务器再次验证. 禁用强制缓存
no-store 所有内容都不会缓存，强制缓存和对比缓存都不会触发。


```js
let http = require('http');
let url = require('url');
let path = require('path');
let fs = require('fs');
let mime = require('mime');

http.createServer(function (req, res) {
   let { pathname } = url.parse(req.url, true);
   let filepath = path.join(__dirname, pathname);
   console.log(filepath);
   fs.stat(filepath, (err, stat) => {
       if (err) {
           return sendError(req, res);
       } else {
           send(req, res, filepath);
        }
    });
}).listen(8080);
function sendError(req, res) {
   res.end('Not Found');
}
function send(req, res, filepath) {
   res.setHeader('Content-Type', mime.getType(filepath));
   //expires指定了此缓存的过期时间，此响应头是1.0定义的，在1.1里面已经不再使用了
   res.setHeader('Expires', new Date(Date.now() + 30 * 1000).toUTCString());
   res.setHeader('Cache-Control', 'max-age=30');  //设置缓存时间为30s
   fs.createReadStream(filepath).pipe(res);
}
```

协商缓存使用 `Etag/If-None-Match` , `Last-Modified/If-Modify-Since`

浏览器第一次请求数据时，服务器会将缓存标识与数据一起返回给客户端，客户端将二者备份至缓存数据库中。再次请求数据时，客户端将备份的缓存标识发送给服务器，服务器根据缓存标识进行判断，判断成功后，返回304状态码，通知客户端比较成功，可以使用缓存数据。

使用 `Last-Modified/If-Modify-Since`方法
```js
http.createServer(function (req, res) {
    let {pathname} = url.parse(req.url);
    let filepath = path.join(__dirname,pathname);
    console.log(filepath);
    fs.stat(filepath,function (err, stat) {
          if (err) {
              return sendError(req,res)
          } else {
              // 再次请求的时候会问服务器自从上次修改之后有没有改过
              let ifModifiedSince = req.headers['if-modified-since'];
              let LastModified = stat.ctime.toGMTString();
              if (ifModifiedSince == LastModified) {
                  res.writeHead('304');
                  res.end('')
              } else {
                  return send(req,res,filepath,stat)
              }
          }
    })

}).listen(8080)

function send(req,res,filepath,stat) {
    res.setHeader('Content-Type', mime.getType(filepath));
    // 发给客户端之后，客户端会把此时间保存下来，下次再获取此资源的时候会把这个时间再发给服务器
    res.setHeader('Last-Modified', stat.ctime.toGMTString());
    fs.createReadStream(filepath).pipe(res)
}

function sendError(req,res) {
    res.end('Not Found')
}
```
使用 `Etag/If-None-Match`方法, Etag 的优先级高于 Last-Modified
```js

```

## 2.图片优化

我的网站访问起来太慢，最主要的原因还是由于使用了大量图片，除了压缩图片，上面说了可以使用雪碧图来减少图片请求，还可以使用字体图标代替一些小图片，其次使用webp格式，可以减少图片的大小。

>使用图片的时候，尽量指定图片大小，不要缩放图片,如果网页中需要什么尺寸的图片，就设计什么尺寸的图片。因为浏览器下载到原始图片之后，如果尺寸与目标尺寸不合，浏览器就会去处理图片(拉伸或者缩小)，造成浏览器负担。

### 2.1 使用字体图标

我使用的是`fontawesome`,但是文件体积有点大，我请求 `fontawesome-webfont.woff2?v=4.7.0`这个文件的时候，要花1秒多，里面有很多的图标其实我并没有用到。推荐一个网站，[http://fontello.com/](http://fontello.com/),可以到里面选择自己需要的图标，然后下载下来，替换掉fontawesome-webfont.woff 这个文件。当然也可以用 `IconFont`，只是我之前是fontawesome，我懒得换了。

### 2.2 使用webp格式
推荐一个网址，可以转换图片格式为webp，[https://www.upyun.com/webp](https://www.upyun.com/webp)

webp使用方法
```html
//html中
<picture>
    <source type="image/webp" srcset="123.webp"> >
    <img src="123.jpg" alt="">
</picture>
```

```js
//首先判断浏览器是否支持webp格式，给文档加上data-webp属性
function isSupportWebp() {
    var flag = '0';
    var canvasEL = document.createElement('canvas');
    var docEl = document.documentElement || document.getElementsByTagName('html')[0];
    if (canvasEL.getContext && canvasEL.getContext('2d')) {
        flag = canvasEL.toDataURL('image/webp').indexOf('image/webp') > -1 ? '1': '0'
    };
    docEl.setAttribute('data-webp', flag);
    return flag
};

// 然后在背景图片中引用,css文件中
[data-webp="0"] div{
    background: ur("./123.jpg")
}
[data-webp="1"] div{
    background: ur("./123.webp")
}
```

> 推荐谷歌的性能检测工具---`PageSpeed`,在谷歌网上商店可以找到，里面会帮你提供有哪些需要改进的地方，在优化图片中会帮你把图片压缩一下，可以直接下载下来使用

![pagespeed测试](http://fs.eyes487.top:9999/uploads/1575897621817-request.png  "图1")

## 3. Gzip传输压缩

在服务器开启`Gzip`压缩，可以将文件在传输的时候体积压缩`70%` 左右，所有浏览器都支持gzip解压，都是自动的。

> 不要对图片进行压缩，首先http压缩是需要成本的，其次，采用HTTP压缩已经被过压缩的东西并不能使它更小。图片压缩不仅浪费了CPU，还有可能增大图片的体积。

我使用的是 `nginx`,通过在 `nginx.config` 配置下面代码
```js
gzip on;   //打开gzip
gzip_min_length 1k;   //当返回内容大于此值时才会使用gzip进行压缩,以K为单位,当值为0时，所有页面都进行压缩。
#gzip_http_version 1.0;   //用于识别http协议的版本，早期的浏览器不支持gzip压缩，用户会看到乱码，所以为了支持前期版本加了此选项。默认在http/1.0的协议下不开启gzip压缩。
gzip_comp_level 2;   //设置gzip压缩级别，级别越底压缩速度越快文件压缩比越小，反之速度越慢文件压缩比越大
gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php application/vnd.ms-fontobject font/ttf font/opentype font/x-woff image/svg+xml;   //设置需要压缩的MIME类型,如果不在设置类型范围内的请求不进行压缩
gzip_vary off;  //是否启用应答头"Vary: Accept-Encoding"
```

设置成功之后，请求的响应头中会带有gzip字段
![开启压缩](http://fs.eyes487.top:9999/uploads/1575968710080-gzip.png  "图2")

## 4. CDN加速

CDN加速意思就是在用户和我们的服务器之间加一个缓存机制，通过这个缓存机制动态获取IP地址根据地理位置，让用户到最近的服务器访问。我们是不可能自己搭建CDN的，所以只有从阿里云，腾讯云之类的购买。因为我暂时不需要，所以就没有使用。(￣ˇ￣)

## 5. 代码优化

* css应该放在页面首部，在页面生成Dom tree的时候，就可以同时渲染页面，不会发生闪屏，白屏和布局混乱。
* js是会阻塞页面加载的，所以js可以放在页面尾部，或者使用defer等延迟加载。
* 使用渐进式加载图片，先用分辨率的图片，等空闲时间在切换高清图片。
* 减少操作dom的次数，把多次操作变为一次操作，比如脱离文档流，隐藏元素等
* 重排一定会重绘，更应该减少重排的发生

下面是我用PageSpeed测试的结果，查询数据的接口，我认为没有做缓存的必要，所以就没有做缓存
![pageSpeed测试](http://fs.eyes487.top:9999/uploads/1575970832530-test-performance.png  "图3") 


参考链接： [https://juejin.im/post/5b6fa8c8...](https://juejin.im/post/5b6fa8c86fb9a0099910ac91#heading-10)