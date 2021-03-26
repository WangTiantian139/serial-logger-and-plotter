# 使用 Web Serial API 在浏览器上实现基于 WEB 的串口通信

> 目前串口调试助手很难提供灵活的数据可视化功能. 有时对于感兴趣信号的表示不够直观. 使用 HTML + JavaScript 语言制作了一个网页 WEB 应用, 在浏览器上记录和展示传感器通过串口上传的数据, 并与传感器简单交互. 其中用到了 Web Serial API 实现串口通信, 使用 chart.js 绘制信号. 

  
![demo](https://img-blog.csdnimg.cn/20210326200921556.gif#pic_center)

## 引言 
串口通信简单, 稳定, 有非常多的硬件基于串口通信, 比如一些经过了简单封装的**传感器模块**. 基于STM32微处理器的各种**开发板**也经常采用串口来打印一些感兴趣的**数据**或者**调试日志**. 

以传感器为例, 一般我们使用各种各样的串口调试助手与我们的传感器通信, **设置传感器的参数, 并接受传感器量测数据**. 

然而, 这些传感器通常具有各式各样的通信协议, 目前串口调试助手很难提供灵活的**数据可视化功能**, 也就是绘图功能. 有时对于感兴趣信号的表示不够直观. 

为此, 使用 HTML + JavaScript 语言制作了一个网页 WEB 应用, **在浏览器上记录和展示传感器通过串口上传的数据, 并与传感器简单交互**. 尝试使用2021年Google在其开发者大会推广的 `Web Serial API` [^1] [^2] [^3] [^4] 和流行的图表库 `chart.js` [^5].

作为一个`HTML`新手, 还是遇到了很多坑的, 所以在这里记录一下. 

## Web Serial API
要注意的是, `Web Serial API`比较新, 所以一些比较古老的浏览器(就是IE)可能无法使用, Android 和 iOS 上的浏览器可能也不能使用. 下面的代码也只在基于Chromium内核的Edge浏览器上应用, 新版本的Chrome应该问题不大, 不确定Firefox是不是可以用. 

## 总体框架
因为是基于前端的串口通信, 所以所有代码都写在一个HTML文件中. 该文件的`<body>`包含了一个**图表**`<canvas id="myChart" height="100">`, 用来展示数据.  还包含了两个按钮: (1) `<button id="butConnect">`连接串口设备并开始通信; (2) `<button id="butEnd">` 停止通信并保存数据. 最后还有一个列表`<div id="received-data-list">`, 记录了接收到的数据. 

因为需要使用 `chart.js` 绘制图表, 因此在`<head>`中引用了`chart.js`.

```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@2.8.0"></script>
```
文件框架如下: 

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Serial Logger and Plotter</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@2.8.0"></script>
  </head>
  <body>
    <canvas id="myChart" height="100"></canvas>
    <div>
      <button id="butConnect">connect</button><span style="padding: 1%"></span
      ><button id="butEnd">end</button>
    </div>
    <div style="margin: 10px;">Received: </div>
    <div id="received-data-list" style="border: groove; margin: 10px;"></div>
    <script>
	/*** TODO: find the chart and list tag below ***/
	/*** TODO: find the chart and list tag above ***/

    /*** TODO: butConnect listener below ***/
    /*** TODO: butConnect listener below above ***/

    /*** TODO: function dealWithData below ***/
    /*** TODO: function dealWithData above ***/
    
    /*** TODO: butEnd listener below ***/
    /*** TODO: butEnd listener above ***/
    </script>
  </body>
</html>

```

## 调用 Web Serial API 进行串口通信

在框架下, 首先找到`chart.js`绘制折线图的标签`<canvas id="myChart" height="100">`和要打印的数据列表的标签`<div id="received-data-list" style="border: groove; margin: 10px;">`.

```js
/*** find the chart and list tag below ***/
var ctx = document.getElementById("myChart").getContext("2d");
var chart = new Chart(ctx, {
  // The type of chart we want to create
  type: "line",

  // The data for our dataset
  data: {
    labels: [],
    datasets: [
      {
        label: "ugpm3",
        borderColor: "rgb(255, 99, 132)",
        data: [],
        fill: false,
      },
      {
        label: "heat",
        borderColor: "blue",
        data: [],
        fill: false,
      },
      {
        label: "humidity",
        borderColor: "green",
        data: [],
        fill: false,
      },
    ],
  },
  // Configuration options go here
  options: {
    title: {
      display: true,
      text: new Date().toLocaleDateString(),
    },
  },
});

const dataList = document.getElementById("received-data-list");
/*** find the chart and list tag above***/
```

下面调用`Web Serial API`连接串口设备. 当点击`<button id="butConnect">connect</button>`按钮时, 总体的逻辑是: 
- 连接串口设备
- 设置串口的波特率
- 获得串口的`reader`和`writer`
- 设置定时向串口设备发送指令(依据需要设定), **传感器收到指令后立即上传数据**
- 串口开始读数据直到`port.readable`和`keepReading`不全为`true`, 一般情况下`port.readable`为`false`意味着设备断开连接, `keepReading`则在`butEnd`的回调函数中设置为`false`
- 在此过程中
  * 每接收一帧数据就调用`dealWithData`函数处理数据. 
- 最后解除`writeInt`并关闭串口.


**注意**: 用到了`await`异步函数的语句不能放在`<script>`标签的最顶层, 也就是必须放在其他函数的回调函数中. 因为会阻塞浏览器运行. 

```js
/*** butConnect listener below ***/
let keepReading = true;
let reader;
let writer;
// all data parsed are stored in a list ordered by received time of the data frame.
let receivedframe = [];

document.getElementById("butConnect").addEventListener("click", async () => {
  const port = await navigator.serial.requestPort();
  await port.open({ baudRate: 9600 }); // set baud rate
  keepReading = true;
  reader = port.readable.getReader();
  writer = port.writable.getWriter();

  // set how to write to device intervally
  const writeInt = setInterval(async () => {
    const commandframe = new Uint8Array([
      0x00,
      0xff /*...some bytes to be sent*/,
    ]);
    await writer.write(commandframe);
  }, 3000); // send a frame every 3000ms

  while (port.readable && keepReading) {
    try {
      while (true) {
        const { value, done } = await reader.read();
        if (done) {
          // Allow the serial port to be closed later.
          reader.releaseLock();
          // Allow the serial port to be closed later.
          writer.releaseLock();
          break;
        }
        if (value) {
          /*** TODO: deal with the data value ***/
          dealWithData(value);
        }
      }
    } catch (error) {
      // Handle non-fatal read error.
      console.error(error);
    } finally {
      console.log(port.readable, keepReading);
    }
  }
  clearInterval(writeInt);
  await port.close();
  console.log("port closed");
});
/*** butConnect listener above ***/
```
函数`dealWithData`按照与传感器约定的通信协议读取接收到的一帧数据:
- 首先校验数据, 
- 然后解析接收到的数据, 获得传感器的浓度/温度/湿度读数. 
- 记录接收到的数据, 以备后续保存数据
- 打印接收到的读数. 
- 接下来将数据绘制在图表`<canvas id="myChart" height="100">`上. 

```js
/*** function dealWithData below ***/
function dealWithData(value) {
  // check the frame
  function checkSum(buf) {
    let checksum = 0;
    buf.forEach((val, idx) => {
      if (idx > 0 && idx < 12) {
        checksum += val;
      } else if (idx == 12) {
        checksum = (~checksum & 0xff) + 1;
      }
    });
    return buf[12] == checksum;
  }

  if (checkSum(value)) {
    // parse the frame 
    let ugpm3 = (value[2] << 8) | value[3];
    let heat = ((value[8] << 8) | value[9]) / 100;
    let humidity = ((value[10] << 8) | value[11]) / 100;
    let datatime = new Date();
    let frame = {
      datatime,
      ugpm3,
      heat,
      humidity,
    };
    
    // record the frame 
    receivedframe.push(frame);

    // print data on the page
    dataList.innerHTML += `<p>[${datatime.toLocaleString()}] -> ugpm3: ${ugpm3}, heat: ${heat}, humidity: ${humidity}</p>`;
    
    // update the chart
    chart.data.labels.push(datatime.toLocaleTimeString());
    chart.data.datasets.forEach((dataset) => {
      dataset.data.push(frame[dataset.label]);
    });
    chart.update();
  }
}
/*** function dealWithData above ***/
```

当点击`<button id="butEnd">end</button>`按钮时, 逻辑是:   

 - 将`keepReading`设置为`false`, 从而跳出`<button id="butConnect">`中的循环, 停止接收数据
 - 将记录`receivedframe`处理为`json`格式的字符串并保存为文件

**注意**: 保存文件时, 要手动数据文件后缀`.json`, 否则可能无法正确读取文件. 

```js
/*** butEnd listener below ***/
document.getElementById("butEnd").addEventListener("click", async () => {
  keepReading = false;
  reader.cancel();
  // create a new handle
  const jsonHandle = await window.showSaveFilePicker();
  // create a FileSystemWritableFileStream to write to
  const writableStream = await jsonHandle.createWritable();
  // write our file
  const aBlob = new Blob([JSON.stringify(receivedframe)], {
    type: "text/plain",
  });
  await writableStream.write(aBlob);
  receivedframe = [];
  // close the file and write the contents to disk.
  await writableStream.close();
});
/*** butEnd listener above ***/
```


## 结论
- `Web Serial API` 依赖浏览器支持
- 可以用HTML+JavaScript组合快速开发上位机WEB应用, 与基于串口的下位机通信, 并直观展示数据
- 想要将数据保存下来, 除了上述的在结束通信时将数据保存成文件, 还可以经由后端应用程序或数据库存储. 
- 总而言之, 使用这个API编写的WEB应用更适合**转发和展示数据**, 而不是操作文件系统 IO **存储数据**, 








[^1]: [10. 连接硬件设备至 web 应用_哔哩哔哩 (゜-゜)つロ 干杯~-bilibili](https://www.bilibili.com/video/BV1DK41137de)
[^2]: [Web Serial API](https://wicg.github.io/serial/#open-method)
[^3]: [Read from and write to a serial port](https://web.dev/serial/)
[^4]: [Getting started with the Web Serial API](https://codelabs.developers.google.com/codelabs/web-serial/#0)
[^5]: [Getting Started · Chart.js documentation](https://www.chartjs.org/docs/latest/getting-started/)

