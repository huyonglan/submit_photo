<a href="https://996.icu"><img src="https://img.shields.io/badge/link-996.icu-red.svg" alt="996.icu" /></a>

# submit_photo
a project about submit_photo

## 图片提交异步问题
每次提交完成要显示照片日期，循环提交图片， 因为请求是异步的，如何保证每次请求回来的结果和当前提交图片一一对应？
在每次调提交接口前new Date 一个时间戳塞到每个图片对象里，在提交结果返回时，找到对应的时间，将返回结果塞到对应图片对象，即可


## 图片压缩
用canvas将图片画出来，用canvas.toDataUrl(type, 0.1)来压缩图片再将得到的base64上传到服务器

## ios或者安卓手机上传竖拍照片旋转90度显示问题
利用exif.js读取照片的拍摄信息，详见  [exif.js官网](http://code.ciaoca.com/javascript/exif-js/)

## 图片预览
用了w-previewimg 插件


![示例图片](./show.jpg)
![示例图片](./show1.jpg)


