# Browser-download-mechanism

最近在研究一项浏览器下载机制，或许我的研究不会成功，但是也在尝试中。。。

主要目的是领导要求各种浏览器下载时候的默认弹框去掉，拦截去掉。我说了做不到，但是领导显然不喜欢这个答案，所以就有了下面的研究begin...

//浏览器下载文件代码

downloadContract = (params) => {

    fetch.DOWNLOAD_CONTRACT(params).then(res => {
    
        if (res.status) {
        
            this.openDownloadDialog(`http://${res.data.zipFilePath_arr}`,`${res.data.zipFilePath_arr}`)
            （该方法等同于window.open(`http://${res.data.zipFilePath_arr}`)）
            
        }
        
    })
    
}

//下载文件（浏览器下载文件源码，但这还不够，还没深入到浏览器内部）

     openDownloadDialog=(url, saveName) =>{
     
        if (typeof url == 'object' && url instanceof Blob) {
        
            url = URL.createObjectURL(url); // 创建blob地址
            
        }
        
        var aLink = document.createElement('a');
        
        aLink.href = url;
        
        aLink.download = saveName || '';
        
        var event;
        
        event = document.createEvent('MouseEvents');
        
        event.initMouseEvent('click', true, false, window, 0, 0, 0, 0, 0, false, false, false, false, 0, null);
        
        aLink.dispatchEvent(event);
        
    }

对于HTTP协议，向服务器请求某个文件时，只要发送类似如下的请求即可：

GET /Path/FileName HTTP/1.0

Host: www.server.com:80

Accept: */*

User-Agent: GeneralDownloadApplication

Connection: close
每行用一个“回车换行”分隔，末尾再追加一个“回车换行”作为整个请求的结束。

第一行中的GET是HTTP协议支持的方法之一，方法名是大小写敏感的，HTTP协议还支持OPTIONS、HAED、POST、PUT、 DELETE、TRACE、CONNECT等方法，而GET和HEAD这两个方法通常被认为是“安全的”，也就是说任何实现了HTTP协议的服务器程序都 会实现这两个方法。对于文件下载功能，GET足矣。GET后面是一个空格，其后紧跟的是要下载的文件从WEB服务器根开始的绝对路径。该路径后又有一个空 格，然后是协议名称及协议版本。

除第一行以外，其余行都是HTTP头的字段部分。Host字段表示主机名和端口号，如果端口号是默认的80则可以不写。Accept字段中的*/* 表示接收任何类型的数据。User-Agent表示用户代理，这个字段可有可无，但强烈建议加上，因为它是服务器统计、追踪以及识别客户端的依据。 Connection字段中的close表示使用非持久连接。

关于HTTP协议更多的细节可以参考RFC2616（HTTP 1.1）。因为我只是想通过HTTP协议实现文件下载，所以也只看了一部分，并没有看全。

如果服务器成功收到该请求，并且没有出现任何错误，则会返回类似下面的数据：

HTTP/1.0 200 OK
Content-Length: 13057672
Content-Type: application/octet-stream
Last-Modified: Wed, 10 Oct 2005 00:56:34 GMT
Accept-Ranges: bytes
ETag: "2f38a6cac7cec51:160c"
Server: Microsoft-IIS/6.0
X-Powered-By: ASP.NET
Date: Wed, 16 Nov 2005 01:57:54 GMT
Connection: close

不用逐一解释，很多东西一看几乎就明白了，只说我们大家都关心内容吧。

第一行是协议名称及版本号，空格后面会有一个三位数的数字，是HTTP协议的响应状态码，200表示成功，OK是对状态码的简短文字描述。状态码共有5类：
1xx属于通知类；
2xx属于成功类；
3xx属于重定向类；
4xx属于客户端错误类；
5xx属于服务端错误类。
对于状态码，相信大家对404应该很熟悉，如果向一个服务器请求一个不存在的文件，就会得到该错误，通常浏览器也会显示类似“HTTP 404 - 未找到文件”这样的错误。Content-Length字段是一个比较重要的字段，它标明了服务器返回数据的长度，这个长度是不包含HTTP头长度的。换 句话说，我们的请求中并没有Range字段（后面会说到），表示我们请求的是整个文件，所以Content-Length就是整个文件的大小。其余各字段 是一些关于文件和服务器的属性信息。

这段返回数据同样是以最后一行的结束标志（回车换行）和一个额外的回车换行作为结束，即“/r/n/r/n”。而“/r/n/r/n”后面紧接的就是文件的内容了，这样我们就可以找到“/r/n/r/n”，并从它后面的第一个字节开始，源源不断的读取，再写到文件中了。

以上就是通过HTTP协议实现文件下载的全过程。但还不能实现断点续传，而实际上断点续传的实现非常简单，只要在请求中加一个Range字段就可以了。

假如一个文件有1000个字节，那么其范围就是0-999，则：

Range: bytes=500-      表示读取该文件的500-999字节，共500字节。
Range: bytes=500-599   表示读取该文件的500-599字节，共100字节。
Range还有其它几种写法，但上面这两种是最常用的，对于断点续传也足矣了。如果HTTP请求中包含Range字段，那么服务器会返回206（Partial Content），同时HTTP头中也会有一个相应的Content-Range字段，类似下面的格式：
Content-Range: bytes 500-999/1000
Content-Range字段说明服务器返回了文件的某个范围及文件的总长度。这时Content-Length字段就不是整个文件的大小了，而是对应文件这个范围的字节数，这一点一定要注意。

一切好像基本上没有什么问题了，本来我也是这么认为的，但事实并非如此。如果我们请求的文件的URL是类似http://www.server.com/filename.exe 这样的文件，则不会有问题。但是很多软件下载网站的文件下载链接都是通过程序重定向的，比如pchome的ACDSee的HTTP下载地址是：

http://download.pchome.net/php/tdownload2.php?sid=5547&url=/multimedia/viewer/acdc31sr1b051007.exe&svr=1&typ=0

这种地址并没有直接标识文件的位置，而是通过程序进行了重定向。如果向服务器请求这样的URL，服务器就会返回302（Moved Temporarily），意思就是需要重定向，同时在HTTP头中会包含一个Location字段，Location字段的值就是重定向后的目的 URL。这时就需要断开当前的连接，而向这个重定向后的服务器发请求。

     好了，原理基本上就是这些了。其实装个Sniffer好好分析一下，很容易就可以分析出来的。不过NetAnts也帮了我一些忙，它的文件下载日志对开发人员还是很有帮助的。

本文引自：http://hi.baidu.com/chinessnetstone/blog/item/603d20094009468ad0581b23.html
