## 人脸智能识别系统

请在最新版`chrome浏览器`测试


-------
### 目录索引
> 人脸识别大概设计实现思路
- [思路分析](#人脸识别大概设计实现思路)

> 主要核心逻辑包括：开启摄像头、人脸识别、关闭摄像头
- [开启摄像头](#开启摄像头)
- [人脸识别](#人脸识别)
- [关闭摄像头](#关闭摄像头)

> 其他
- [选择上传图片](#选择上传图片)
- [事件委托代理](#事件委托代理)
- [自定义loading](#自定义loading)

> 最后
- [线上环境](#线上环境)
- [后记](#后记)

-------

### 人脸识别大概设计实现思路
```javascript
/**
 * node sdk 人脸识别大概思路
 * 
 * 1.前端页面
 *    1.js调用本地摄像头，获取视频流  video
 *       getUserMedia
 *    2.视频流解码，用video展示出来 file  blob
 *       window.URL.createObjectURL(buffer)
 *    3.canvas截取video视频流生成图片
 *       创建canvas对象，绘制截取图片
 *    4.图片生成base64编码
 *       cxt.drawImage    canvas.toDataURL
 *    5.base64传输到后台
 *    6.反馈信息交互
 *    
 * 2.后端解析
 *    1.获取前端传入的图像数据
 *       faceData:左侧图片数据  scanData:右侧扫描数据
 *    2.调用 baidu-ai 处理人脸对比数据
 *    3.返回 score 给前端
 *       {"result":[{"index_i":"0","index_j":"1","score":92.184844970703}],"result_num":1,"log_id":2309353456092517}
 */
```

### 主要核心逻辑包括：开启摄像头、人脸识别、关闭摄像头

#### 开启摄像头

```javascript
/**
 * 开启摄像头
 * 相应h5 API 查看地址
 * https://developer.mozilla.org/en-US/docs/Web/API/Navigator/getUserMedia
 */
function openCamera() {
    console.log(`开启摄像头`);
    //访问用户媒体设备的兼容方法
    function getUserMedia(constraints, success, error) {
        if (navigator.mediaDevices.getUserMedia) {
            //最新的标准API
            navigator.mediaDevices.getUserMedia(constraints).then(success).catch(error);
        } else if (navigator.webkitGetUserMedia) {
            //webkit核心浏览器
            navigator.webkitGetUserMedia(constraints, success, error)
        } else if (navigator.mozGetUserMedia) {
            //firfox浏览器
            navigator.mozGetUserMedia(constraints, success, error);
        } else if (navigator.getUserMedia) {
            //旧版API
            navigator.getUserMedia(constraints, success, error);
        }
    }

    // let video = document.getElementById('video');
    // let canvas = document.getElementById('canvas');
    // let context = canvas.getContext('2d');

    function success(stream) {
        //兼容webkit核心浏览器
        let CompatibleURL = window.URL || window.webkitURL;
        //将视频流设置为video元素的源
        console.log(stream);
        buffer = stream
        video.srcObject = buffer;
        video.play();
    }

    function error(error) {
        console.log(`访问用户媒体设备失败${error.name}, ${error.message}`);
    }

    if (navigator.mediaDevices.getUserMedia || navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia) {
        //调用用户媒体设备, 访问摄像头
        getUserMedia({ video: { width: 480, height: 320 } }, success, error);
    } else {
        alert('不支持访问用户媒体');
    }
}


```

#### 人脸识别

- 前端处理
```javascript
/**
 * 开始人脸识别
 * 将视频截屏，用canvas显示，然后再转为base64码传给后台
 * ps: 这里相似度默认在 90% 以上就算着识别成功，可设置精准度。最高 100% 最低 80%
 */
function scanFace() {
  console.log('开始人脸识别');
  if (!buffer) {
    console.log('没有视频流');
    return
  }

  // 精准度
  var accuracy = Util._$('accuracy').value;

  // 视频流解码，用video展示出来 file  blob
  var oCanvas = Util._$('canvas_l');
  var ctx = oCanvas.getContext('2d');
  ctx.drawImage(video, 0, 0, 300, 230); // 实际上是对视频截屏
  var faceData = Util._$('facedata').src;
  var load = new Loading();
  load.show();
  $.ajax({
    method: 'POST',
    url: '/scanface',
    data: {
      faceData: /^data:image\/(\w*);base64,/.test(faceData) ? faceData : null,
      scanData: oCanvas.toDataURL()
    },
    success: function (res) {
      console.log(res);
      load.hide();
      res.score > (!isNaN(Number(accuracy)) ? accuracy : 90) ? changeStatus('success') : changeStatus('error');
    },
    error: function (res) {
      console.log(res);
      load.hide();
    }
  })
}
```
- 后端处理
```javascript
app.post('/scanface', function (req, res) {
  // 数据库源数据
  var imageBase64;
  // 上传需要识别的图片
  var faceData = req.body.faceData;
  if (faceData) {
    imageBase64 = faceData;
  } else {
    // 没有传图片就默认系统图片
    var path = `${rootPath}static/images/myavatar.png`;
    var imageData = fs.readFileSync(path);
    imageBase64 = new Buffer(imageData).toString('base64');
  }
  // console.log(base64);

  // 人脸识别传入的数据
  var scanData = req.body.scanData;
  scanData = scanData.replace(/^data:image\/(\w*);base64,/g, '');
  // console.log(scanData)

  client.match([imageBase64, scanData]).then(function(result) {
    console.log(JSON.stringify(result));
    res.json({score: result.result[0] ? result.result[0].score : 0});
  });
});
```
- 启动后端服务

在终端执行如下命令：
```bash
node app.js
```

#### 关闭摄像头

```javascript
/**
 * 将视频流关闭
 */
function closeCamera() {
  console.log('关闭摄像头')
  buffer && buffer.getVideoTracks()[0].stop(); // 暂停当前播放的音频/视频
  buffer = null;
  changeStatus();
}
```

在这里核心功能已经实现了，是不是很（hen）牛（jian）逼（dan）。另外针对界面及功能的实现附带了其他效果以及实现方法。

#### 选择上传图片

```javascript
/**
 * 选择上传图片。
 * 其实并没有向服务器上传任何文件，只是利用了H5的api将图片转为base64，然后给要显示的image的src赋值为转化后的base64码
 * @returns {boolean}
 */
Util._$('uploadfile').onchange = function () {
  var file = this.files && this.files[0];
  //判断是否是图片类型
  if(!/image\/(png|jpg|jpeg)/.test(file.type)){
    this.value = null;
    alert("只能选择(png,jpg)格式图片");
    return false;
  }
  var reader = new FileReader();
  reader.readAsDataURL(file);
  reader.onload = function(e){
    Util._$('facedata').src = this.result;
  }
}
```

#### 事件委托代理

```javascript
/**
 * 基于 querySelectorAll的事件绑定，从 Array.prototype中剽窃了 forEach 方法来完成遍历
 * 按钮监听，设置事件代理
 */
Array.prototype.forEach.call(document.querySelectorAll('.control-btn'), function (el) {
  el.addEventListener('click', function (e) {
    e = e || window.event;
    var id = e.target.id;
    switch (id) {
      case 'open': // 开启摄像头
        openCamera();
        break;
      case 'close': // 关闭摄像头
        closeCamera();
        break;
      case 'scanface': // 人脸识别
        scanFace();
        break;
      default:
        break;
    }
  })
});
```

### 自定义loading

```javascript
/**
 * 请求正在加载中。。。
 * @constructor
 */
function Loading() {
  var loadingDiv, queryDom = document.querySelector('.loading');
  if (queryDom) {
    loadingDiv = queryDom;
  } else {
    loadingDiv = document.createElement('div');
    loadingDiv.classList.add('loading');
    var loadingImg = document.createElement('img');
    loadingImg.setAttribute('src', 'static/images/loading.gif');
    loadingDiv.appendChild(loadingImg);
    document.querySelector('body').appendChild(loadingDiv);
  }
  this.show = function () {
    loadingDiv.style.display = 'flex';
  }
  this.hide = function () {
    loadingDiv.style.display = 'none';
  }
}
```

