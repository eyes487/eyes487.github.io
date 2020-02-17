---
title: 自建图床：上传图片 懒加载 瀑布流显示
categories: 前端
tag: ['上传文件','js','node']
date: 2019-11-16
---


一般大家写博客的时候，都会使用外链图片，我也使用过几个免费的图床，但是效果都不太理想。也只是偶尔需要用图，使用付费的也不合算。自己的服务器上也没多少东西，所以就在自己服务器上搭了一个。比较简单，上传图片，页面显示，部署到服务器上就行了。[查看地址](http://fs.eyes487.top:9999)

查看完整代码请戳[这里](https://github.com/eyes487/image-upload),下面会对关键性代码做一些讲解。

## 1.上传图片

### 1.1 html代码
```html
<input type="file" multiple id="file">
<input type="button" value="上传" id="upload">
```
页面是这个样子，当然可以自己加一些css样式
![文件上传](http://fs.eyes487.top:9999/uploads/1581848469566-file.png "图1")

### 1.2 js代码
```js
var fileinput = document.getElementById('file')

// 选择文件
fileinput.onchange = () => {
    var files = fileinput.files;
    let imgDOMArray = new Array(files.length)
    let reader = []
    let thumbPic = []
    for (let i = 0; i < files.length; i++) {
            reader[i] = new FileReader()
            thumbPic[i] = document.createElement('div')
            imgDOMArray[i] = document.createElement('img')
            imgDOMArray[i].file = files[i]
            thumbPic[i].className = 'thumbPic'
            thumbPic[i].appendChild(imgDOMArray[i])
            previewDOM.appendChild(thumbPic[i])
            reader[i].readAsDataURL(files[i])
            reader[i].onload = (img => {
                return e => {
                    img.src = e.target.result
                }
            })(imgDOMArray[i])
    }
}
```
选择文件，选择之后，可以把图片显示在下面，方便查看

![选择图片](http://fs.eyes487.top:9999/uploads/1581848855064-select-img.png "图2")


下面是使用ajax向后台发送上传图片请求，上传图片需要使用`FormData`格式，向后台发送数据，ajax提供了`xhr.upload.onprogress`可以监听文件上传的进度，这个必须在send之前设置
```js
var button = document.querySelector('#upload')

button.onclick = uploadFile;

function uploadFile() {
    var xhr = new XMLHttpRequest();
    var formdata = new FormData()
    var files = fileinput.files
    if (!files[0]) {
        alert('请先选择图片，再上传！')
        return
    }
    for (let i = 0; i < files.length; i++) {
        formdata.append('imgfile', files[i], files[i].name)
    }
    xhr.open('POST', BASE_URL+'/uploadimg')
    //可以查看上传的进度
    xhr.upload.onprogress = e => {
        var current = e.loaded; //当前上传多少size
        var total = e.total; //总共size
        if (e.lengthComputable) {
            //...
            if (percentComplete >= 100) {
               //...
            }
        }
    }
    xhr.send(formdata)
    xhr.onload = () => {
        if (xhr.readyState == 4 && xhr.status === 200) {
            //上传成功
        }
    }
}

```

### 1.3 服务端代码

服务端代码，使用`node + express`，可以快速的搭建服务端代码
```bash
npm install express -S
```
然后新建`server.js`,
```js
var express = require("express");

var app = express();

var server = app.listen(9999, function() {
    console.log('server is running at port 9999...');
});
```
这样就搭好服务器了，可以通过`localhost:9999`,直接访问服务器了

`Multer` 是一个 node.js 中间件，用于处理 `multipart/form-data` 类型的表单数据，它主要用于上传文件。安装 `multer`
```bash 
npm install multer -S
```
在`server.js`中加入
```js
//server.js
var multer = require("multer");

var storage = multer.diskStorage({
    destination: function(req, file, cb) {
        cb(null, './public/uploads');  //把上传的文件保存在 这个地址下面
    },
    filename: function(req, file, cb) {
        cb(null, `${Date.now()}-${file.originalname}`)  //对上传的文件，改名，可能上传的会有重名文件
    }
})

var uploadFile = multer({ storage: storage });
```
multer 提供了 `storage` 这个参数来对资源保存的路径、文件名进行个性化设置。

下面编写接口
```js
//server.js
app.post('/uploadimg', uploadFile.array('imgfile', 10), function(req, resp, next){
    resp.send({ status: 200, message: '上传成功'})
})
```
上传文件接口`uploadimg`
upload.array('imgfile', 10), 表示接收一个以`imgfile`命名的数组，长度最大为10。
这样就上传到服务器了

但是，我们不只把图片存到服务器就行了，还需要保存图片信息到数据库

这里数据库使用 `mysql`
```bash
npm install mysql -S
```
连接数据库，封装一个连接数据库的文件
```js
// db.js
var mysql = require('mysql');
var dbConn={
    configure:{
        host:"localhost",//主机地址
        port:"3306",//主机端口
        user:"root",//用户名
        password:"123456",//主机密码
        database:"image_store"//数据库名
    },
    sqlConnect:function(sql,values,callback){
        var pool=mysql.createPool(this.configure);
        pool.getConnection(function(error,connection){
            if(error){
                console.log('数据库---',error)
            }
            connection.query(sql,values,callback);
            connection.release();
        });
    }
}
module.exports={
    dbConn:dbConn
}
```
上面上传文件的时候，就可以改成这样
```js
const db = require('./db');

app.post('/uploadimg', uploadFile.array('imgfile', 10), upload)

function upload(req, resp, next) {
    var files = req.files
    var names = [];
    for(let i = 0;i < files.length; i++){
        names.push("('"+files[i].filename+"')");
    }
    var sql = "insert into img_list (imgSrc)values " + names.join(',');
    db.dbConn.sqlConnect(sql,[],function(err,data){
        if(err){
            resp.status(500).send({
                status: 500,
                message: err
            })
        }else{
            resp.send({
                status: 200,
                message: '上传成功'
            });
        }
    })
}
```
upload中插入上传的文件名到`img_list`表中，上传多张图片就插入多条数据。上传图片就算完成了。

## 2.渲染页面

使用瀑布流显示页面上的图片，做瀑布流的方法有多种，这里我使用了`绝对定位`的方法来实现，然后下拉刷新，根据滚动条的高度来实现懒加载。

### 2.1 瀑布流

查询图片列表，服务端代码
```js
const fs = require('fs');
const image = require("imageinfo");

//查询列表
const getImageList = function(req,resp){
    const pageNum = req.query.pageNum || 1;
    const pageSize = req.query.pageSize || 10;
    const sql = "select * from img_list order by id desc limit ?, ?";
    db.dbConn.sqlConnect(sql,[(pageNum-1)*pageSize,pageSize*1],function(err,data){
        if(err){
            resp.status(400).send({
                status: 400,
                message: err
            })
        }else{
            let newData = getImageFiles('public/uploads/',data)
            resp.send({
                status: 200,
                data: newData,
                message: '查询成功'
            });
        }
    })
}

//获取指定图片的尺寸
function getImageFiles(path,data) {
    if(!data.length){
        return []
    }
    var imageList = [];
    data.forEach((item) => {
        var ms = image(fs.readFileSync(path + item.imgSrc));
        ms.mimeType && (imageList.push({...item,width: ms.width, height: ms.height}))
    });
    return imageList;
}
```
从数据库按照倒叙，插叙最新的图片名称，查询之后需要对数据信息做一些处理。

用到了`fs 和 image`两个插件
```bash 
npm install fs image -S
```

`fs`是读取存在服务器上的图片文件，`image`可以获取图片的具体信息,我们会返回图片的宽和高。之后做瀑布流渲染，需要拿到图片的宽和高。

在前端请求接口，请求数据列表
```js
function getImageList() {
    canLoad = false; //请求数据时，改为不可加载
    var xhr = new XMLHttpRequest();
    xhr.open('GET', BASE_URL+`/getImageList?pageNum=${pageNum}&pageSize=${pageSize}`);
    xhr.send();
    xhr.onload = () => {
        if (xhr.readyState == 4 && xhr.status === 200) {
            let res = JSON.parse(xhr.responseText);
            let data = res.data;
            renderImage(data);
        }
    }
}
```

请求数据之后，就依据数据生成相应的dom,使用渲染函数 `renderImage`
```js
/**
 * 渲染图片
 * @param {*} list 图片数据
 */
function renderImage(list) {
    let Node = []; //定义一个数组，把创建的节点存储起来，方便后面修改这些节点的位置信息
    let Fragment = document.createDocumentFragment();
    for (let i = 0, len = list.length; i < len; i++) {
        let Div = document.createElement('div');
        Div.setAttribute('class','image-item');
        Div.innerHTML = `
            <input type="checkbox" class="delete-checkbox" onClick="selectDeleteImg(this,'${list[i].id}','${list[i].imgSrc}')">
            <img class="img" src="./uploads/${list[i].imgSrc}"/>
            <p class="desc">${list[i].imgSrc}</p>
        `;
        Node.push(Div)
        Fragment.appendChild(Div);
    }    
    //设置新增加点的位置
    change(Node,list);
    imageList.appendChild(Fragment)
    //渲染完之后
    if(!(list.length < pageSize)){
        pageNum++;
        canLoad = true;
    }
}
```
这里定义了一个Node数组来存储新创建的节点，是为了在`change`函数中遍历这个数组，来为节点设置定位信息，同时，也把节点放进了`Fragment`中，这样之后就可以一次性追加进页面中，提高性能。在渲染完页面之后，把`pageNum`和`canLoad`标记重新复制。canLoad是一个变量，用来判断现在是否正在请求列表，防止多次触发。

`change`方法
```js
/**
 * 改变指定图片尺寸
 * @param {*} Items dom元素
 * @param {*} source 图片数据，窗口改变大小时不传
 */
var windowCW = document.documentElement.clientWidth,  //窗口视口的宽度
    n = Math.floor(windowCW / 410),                     //一行能容纳多少个div，并向下取整
    center = (windowCW - n * 410) / 2,                   //居中
    arrH = [];                                       //定义一个数组存放每个item的高度

function change(Items,source) {
    if (n <= 0) { return };
    
    for (var i = 0; i < Items.length; i++) {
        var j = i % n;
        let height = source ? Math.ceil(380/(source[i].width/source[i].height))+70 : Items[i].offsetHeight; //根据图片设置宽度380，同比缩放高度
        if (arrH.length == n) {                    //一行排满n个后到下一行                    
            var min = findMin(arrH);              //从最“矮”的排起，可以从下图的序号中看得出来，下一行中序号是从矮到高排列的
            Items[i].style.left = center + min * 410 + "px";
            Items[i].style.top = arrH[min] + 10 + "px";
            arrH[min] += height + 10;
        } else {
            arrH[j] = height;
            Items[i].style.left = center + 400 * j + 10 * j + "px";
            Items[i].style.top = 0;
        }
    };
}
/**
 * 查询数组中最小的数据
 * @param {*} arr 
 */
function findMin(arr) {
    var m = 0;
    for (var i = 0; i < arr.length; i++) {
        m = Math.min(arr[m], arr[i]) == arr[m] ? m : i;
    }
    return m;
}
```
瀑布流就是图片宽度相等，但是高度不相等，这里我设置的宽度是 `380` ，图片容器会有`padding:10`
* 首先根据窗口的宽度，定义了一行能够容纳`n`张 图片，包括图片容器的宽度都要计算在内
* `center`是为了居中显示，判断了除去图片的宽度，在除以`2`，就是`left`开始的位置
* `arrH` 定义一个数组，用来存放`n`列图片中，每列图片的`高度总和`，下一张图片就会从`最矮`的那一列开始排列
* 每张图片高度，会根据后台返回的数据中带有宽高，根据定义的固定宽度(380)来等比缩放，判断高度应该是多少，但是当窗口发生改变的时候，我们不会再传入源数据，这时候直接使用`Items[i].offsetHeight`，应为节点之前已经渲染过了，所以直接会有offsetheight
* 循环所有节点，先把第一行排满，排满之后就开始通过`findMin(arrH)`寻找高度最低的那一列下标，放在这一列，在把这一列高度加上这个图片的高度，就这样循环完所有节点

当窗口的宽度改变的时候，需要重新计算所有节点的位置
```js
/**
 * 重新计算窗口宽度
 */
function resetSize(){
    windowCW = document.documentElement.clientWidth;  //窗口视口的宽度
    n = Math.floor(windowCW / 410);                     //一行能容纳多少个div，并向下取整
    center = (windowCW - n * 410) / 2;                   //居中
    arrH = [];                                       //定义一个数组存放每个item的高度
}
window.onresize = function{
    var Items = document.querySelectorAll('.image-item')
    resetSize();
    change(Items);
}
```

这样，瀑布流展示就做完了。查看效果点击[这里](http://fs.eyes487.top:9999/)


### 2.2 懒加载

懒加载就是页面开不到的地方先不加载，等页面滑到的地方在去请求数据

```js
/**
 * 懒加载
 */
window.onscroll = function (){
    let height = arrH.length && arrH[findMin(arrH)];    
    if(canLoad && height){
        let scrollTop = document.documentElement.scrollTop;
        let clientHeight = document.documentElement.clientHeight;

        if(scrollTop + clientHeight > height){
            getImageList()
        }
    }
}
```
当 滚动条高度 加上 文档可视区的高度 要大于 `arrH`中最小的高度，也就是页面上图片最小高度总和的时候，就需要加载新的数据了。
同时还需要通过 `canLoad`变量判断，防止多次加载，或者后面已经没有数据的时候，这个变量就会变为不可加载。懒加载基本就是这样了。


*

还有一些其他的功能，比如`查看大图`，`删除图片`等，这里就不细说了，如果感兴趣的可以去[查看完整代码](https://github.com/eyes487/image-upload)