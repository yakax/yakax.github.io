# HTTP请求头和响应头部包括的信息一般有哪些

**之前在看老齐的视频，做了一个百思不得姐爬虫的后台，效果还不错，准备过段时间买个服务器放上去。不过在这个项目中最重要的就是模拟请求、爬数据，让我了解到了okhttp这个工具贼好用，这里就不介绍了这个工具了，这里先分析原理、了解请求头和响应头。**
<!--more-->
#### 请求头(Request Headers)
- Accept:浏览器能够处理的内容类型
- Accept-Charset:浏览器能够显示的字符集
- Accept-Encoding：浏览器能够处理的压缩编码
- Accept-Language：浏览器当前设置的语言
- Connection：浏览器与服务器之间连接的类型
- Cookie：当前页面设置的任何Cookie
- Host：发出请求的页面所在的域
- Referer：发出请求的页面的URL
- User-Agent：浏览器的用户代理字符串（当时我就是通过服务器增加这个请求信息模拟请求获取数据的）

#### 响应头(Response Headers)
- Date：表示消息发送的时间，时间的描述格式由rfc822定义
- server:服务器名字。
- Connection：浏览器与服务器之间连接的类型
- content-type:表示后面的文档属于什么MIME类型
- Cache-Control：控制HTTP缓存

以上请求响应不能概全，因为每个请求响应都可能不相同，这只是一般请求会有的一些信息。


