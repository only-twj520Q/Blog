

近在做图片上传，接触了一些图片相关的转化操作。主要涉及blob，canvas，dataurl。做一下笔记，以免时间长了忘记。



## 名词解释

### Blob

> Blob对象表示不可变的类似文件对象的原始数据。Blob表示不一定是JavaScript原生形式的数据。File接口基于Blob，继承了blob的功能并将其扩展使其支持用户系统的文件。
>
> 摘自MDN

和**File对象**的区别

File对象就是一个文件，当我们使用`input type="file"`标签来上传文件，我们在代码里得到的对象就是一个File对象。

Blob对象就是二进制数据，用来包装二进制文件的容器，可以通过`new Blob()`创建的对象就是Blob对象。又比如，在XMLHttpRequest里，如果指定responseType为blob，那么得到的返回值也是一个blob对象。

File继承于Blob

### fileReader

> fileReader对象允许Web应用程序异步读取存储在计算机上的文件（原始数据缓冲区）的内容，使用File或Blob对象指定要读取的文件或数据。
>
> 摘自MDN

`FileReader`是用来读取内存中的文件的API，支持`File`和`Blob`两种格式。



## 转化关系

这边从使用场景出发，来整理一下。

### file转化为DataURL

**场景**： 获取到一个file类型的图片，直接在页面中预览。如果等图片上传完服务器返回图片的地址，再赋值给img的src属性，无疑浪费了时间和带宽。如果直接在html中预览？这里就是利用html5的新特性，将图片转换为Base64的形式显示出来。

下面的方法可以用于在浏览器上预览本地图片或者视频。

#### 方法1：URL.createObjectURL

该方法会根据传入的参数创建一个指向该参数对象的URL，这个URL是一个基于当前文件并且存储在内存中的`URL`，直到document触发了unload事件或者执行revokeObjectURL来释放。

可以用于在浏览器上预览本地图片或者视频

**使用方法如下**

```javascript
objectURL = URL.createObjectURL(param);
```

* 参数：可以为File或者Blob对象
* 返回：一段带hash的url，类似`blob:null/2c5e93e3-f06e-4bf2-8942-1708759bf7a7`

**注意**：在每次调用 `createObjectURL() `方法时，都会创建一个新的 URL 对象。当不再需要这些 URL 对象时，每个对象必须通过调用`URL.revokeObjectURL()`方法来释放，让浏览器知道这个URL已经不再需要指向对应的文件。`URL.revokeObjectURL()`方法会释放一个通过`URL.createObjectURL()`创建的对象URL。浏览器关闭后自动释放这个对象，但是为了获得最佳性能和内存使用状况，你应该在安全的时机主动释放掉它们。

代码如下

```javascript
let img = document.createElement("img");
img.src = window.URL.createObjectURL(files[i]);
img.width = 200;
img.onload = function() {
	window.URL.revokeObjectURL(this.src);
}
document.body.appendChild(img);
```

#### 方法2：FileReader.readAsDataURL

该方法会读取指定的 Blob 或 File 对象。读取操作完成的时候，readyState 会变成已完成（DONE），并触发 loadend 事件，同时 result 属性将包含一个data:URL格式的字符串（base64编码）以表示所读取文件的内容。

**使用方法如下**

```javascript
FileReader.readAsDataURL(blob);
```

- 参数：可以为File或者Blob对象
- 返回：图片的base64编码。类似`data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEASABIA`

代码如下

```javascript
let reader = new FileReader();
reader.onload = function(e) {
  // result属性是包含data:URL格式的字符串（base64编码）表示读取的文件内容
  img.src = e.target.result;
  img.onload = function() {
    dosomething()
  }
}
// 读取file对象
reader.readAsDataURL(files[i]);
```

#### 两种方法的区别

1、执行时机

- `createObjectURL`是同步执行（立即的）
- `FileReader.readAsDataURL`是异步执行

2、内存使用

- `createObjectURL`返回一段带`hash`的`url`，并且一直存储在内存中。
- `FileReader.readAsDataURL`则返回包含很多字符的`base64`，并会比`blob url`消耗更多内存，但是在不用的时候会自动从内存中清除（通过垃圾回收机制）

3、兼容性

- `createObjectURL`支持从IE10往上的所有现代浏览器
- `FileReader.readAsDataURL`同样支持从IE10往上的所有现代浏览器

从上面答案不难看出，两者的优劣势

- 使用`createObjectURL`可以节省性能并更快速，只不过需要在不使用的情况下手动释放内存

- 如果不太在意设备性能问题，并想获取图片的`base64`，则推荐使用

  

## canvas 转化为DataURL

**场景**： canvas画出来的图片，需要在其它地方预览或者作为组件之间通信的参数。

**使用方法如下**

```javascript
canvas.toDataURL(type, encoderOptions);
```

- 参数：
  - type：图片格式，默认为 `image/png`
  - encoderOptions：在指定图片格式为 `image/jpeg`或`image/webp`的情况下，可以从 0 到 1 的区间内选择图片的质量。如果超出取值范围，将会使用默认值 `0.92`。其他参数会被忽略。
- 返回：图片的base64编码。类似`data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEASABIA`

使用

* 当一个内容画到canvas上时，我们可以将它生成任何一个格式支持的图片文件。
* 这个方法也可以用来对图片进行格式转化。比如，file(png格式) -> canvas -> base64(webp格式)，参考下图。
* 这个方法还可以用来压缩图片。

![](https://ws3.sinaimg.cn/large/006tKfTcly1g1jvojadcqj30b702kjrn.jpg)



## canvas 转化为blob

**场景**： canvas画出来的图片，或者利用canvas完成截图功能之后，我们需要把对应的文件上传到服务器。

**使用方法如下**

```javascript
canvas.toBlob(callback, type, encoderOptions);
```

- 参数：
  - callback：回调函数，可获得一个单独的`Blob`对象参数
  - type：`DOMString`类型，指定图片格式，默认格式为`image/png`
  - encoderOptions：`Number`类型，值在0与1之间，当请求图片格式为`image/jpeg或者``image/webp时用来指定图片展示质量。如果这个参数的值不在指定类型与范围之内，则使用默认值，其余参数将被忽略。`
- 返回：无返回值

得到Blob对象以后，可以利用XML2的`FormData`对象上传，模拟表单控件，异步上传这个二进制文件。

代码如下

```javascript
canvas.toBlob && canvas.toBlob(function(blob) {
　let myForm = new FormData();
	myForm.append('image', blob); //向表单中添加一个键值对
};
```



## DataURL或img转canvas

**场景**： 想要对图片想要进行一些操作，比如加蒙层，修饰，裁剪等等，这些操作可以利用canvas的特性去完成。

**使用方法如下**

```javascript
context.drawImage(img,sx,sy,swidth,sheight,x,y,width,height);
```

* 参数
  * image：绘制到上下文的元素。
  * sx：需要绘制到目标上下文中的，`image`的矩形（裁剪）选择框的左上角 X 轴坐标。
  * sy：需要绘制到目标上下文中的，`image`的矩形（裁剪）选择框的左上角 Y 轴坐标。
  * swidth：需要绘制到目标上下文中的，`image`的矩形（裁剪）选择框的宽度。如果不说明，整个矩形（裁剪）从坐标的`sx`和`sy`开始，到`image`的右下角结束。
  * sheight：需要绘制到目标上下文中的，`image`的矩形（裁剪）选择框的高度。



![drawImage](https://mdn.mozillademos.org/files/225/Canvas_drawimage.jpg)

代码如下

```javascript
function dataURLToCanvas(dataurl){
	let canvas = document.createElement('canvas');
	let ctx = canvas.getContext('2d');
	let img = new Image();
	img.onload = function(){
		canvas.width = img.width;
		canvas.height = img.height;
		ctx.drawImage(img, 0, 0);
	};
	img.src = dataurl;
}
```



