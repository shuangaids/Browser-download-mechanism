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

扩展window对象有以下方法：
open
close
alert
confirm
prompt
setTimeout
clearTimeout
setInterval
clearInterval
moveBy
moveTo
resizeBy
resizeTo
scrollBy
scrollTo
find
back
forward
home
stop
print
blur
focus
captureEvent
enableExternalCapture
disableExternalCapture
handleEvent
releaseEvent
routeEvent
scroll
　　1. open方法
　　语法格式：
window.open(URL,窗口名称,窗口风格)
　　功能：打开一个新的窗口，并在窗口中装载指定URL地址的网页。
　　说明：
open方法用于打开一个新的浏览器窗口，并在新窗口中装入一个指定的URL地址；
open方法在打开一个新的浏览器窗口时，还可以指定窗口的名称(第二个参数)；
open方法在打开一个新的浏览器窗口时，还可以指定窗口的风格(第三个参数)，
窗口风格有以下选项，这些选项可以多选，如果多选，各选项之间用逗号分隔：
toolbar：指定窗口是否有标准工具栏。当该选项的值为1或yes时，表示有标准工具栏，当该选项的值为0或no时，表示没有标准工具栏；
location：指定窗口是否有地址工具栏，选项的值及含义与toolbar相同；
directories：指定窗口是否有链接工具栏，选项的值及含义与toolbar相同；
status：指定窗口是否有状态栏，选项的值及含义与toolbar相同；
menubar：指定窗口是否有菜单，选项的值及含义与toolbar相同；
scrollbar：指定当前窗口文档大于窗口时是否有滚动条，选项的值及含义与toolbar相同；
resizable：指定窗口是否可改变大小，选项的值及含义与toolbar相同；
width：以像素为单位指定窗口的宽度，已被innerWidth取代；
height：以像素为单位指定窗口的高度，已被innerHeight取代；
outerWidth：以像素为单位指定窗口的外部宽度；
outerHeight：以像素为单位指定窗口的外部高度；
left：以像素为单位指定窗口距屏幕左边的位置；
top：以像素为单位指定窗口距屏幕顶端的位置；
alwaysLowered：指定窗口隐藏在所有窗口之后，选项的值及含义与toolbar相同；
alwaysRaised：指定窗口浮在所有窗口之上，选项的值及含义与toolbar相同；
dependent：指定打开的窗口为当前窗口的一个子窗口，并随着父窗口的关闭而关闭，选项的值及含义与toolbar相同；
hotkeys：在没有菜单栏的新窗口中设置安全退出的热键，选项的值及含义与toolbar相同；
innerHeight：设定窗口中文档的像素高度；
innerWidth：设定窗口中文档的像素宽度；
screenX：设定窗口距离屏幕左边界7a686964616fe59b9ee7ad9431333264663631的像素长度；
screenY：设定窗口距离屏幕上边界的像素长度；
titleBar：指明标题栏是否在新窗口中可见，选项的值及含义与toolbar相同；
z-look：指明当窗口被激活时，不能浮在其它窗口之上，选项的值及含义与toolbar相同。
open方法返回的是该窗口的引用。
2. close方法
语法格式：
window.close()
功能：close方法用于自动关闭浏览器窗口。
3. alert方法
语法格式：
window.alert(提示字符串)
功能：弹出一个警告框，在警告框内显示提示字符串文本。
4. confirm方法
语法格式：
window.confirm(提示字符串)
功能：显示一个确认框，在确认框内显示提示字符串，当用户单击“确定”按钮
时该方法返回true，单击“取消”时返回false。
5. prompt方法
语法格式：
window.prompt(提示字符串，缺省文本)
功能：显示一个输入框，在输入框内显示提示字符串，在输入文本框显示缺省文
本，并等待用户输入，当用户单击“确定”按钮时，返回用户输入的字符串，当
单击“取消”按钮时，返回null值。
6. setTimeout方法
语法格式：
window.setTimeout(代码字符表达式,毫秒数)
功能：定时设置，当到了指定的毫秒数后，自动执行代码字符表达式。
7. clearTimeout方法
语法格式：
window.clearTimeout(定时器)
功能：取消以前的定时设置，其中的参数是用setTimeout设置时的返回值。
8. setInterval方法
语法格式：
window.setInterval(代码字符表达式,毫秒数)
功能：设定一个时间间隔后(第二个参数)，反复执行“代码字符表达式”的内容
9. clearInterval方法
语法格式：
window.clearInterval(时间间隔器)
功能：取消setInterval设置的定时。其中的参数是setInterval方法的返回值。
10. moveBy方法
语法格式：
window.moveBy(水平位移量,垂直位移量)
功能：按照给定像素参数移动指定窗口。第一个参数是窗口水平移动的像素，第
二个参数是窗口垂直移动的像素。
11.moveTo方法
语法格式：
window.moveTo(x,y)
功能：将窗口移动到指定的指定坐标(x,y)处。
12. resizeBy方法
语法格式：
window.resizeBy(水平,垂直)
功能：将当前窗口改变指定的大小(x,y)，当x、y的值大于0时为扩大，小于0时
为缩小。
13. resizeTo方法
语法格式：
window.resizeTo(水平宽度,垂直宽度)
功能：将当前窗口改变成(x,y)大小，x、y分别为宽度和高度。
14. scrollBy方法
语法格式：
window.scrollBy(水平位移量，垂直位移量)
功能：将窗口中的内容按给定的位移量滚动。参数为正数时，正向滚动，否则反
向滚动。
15. scrollTo方法
语法格式：
window.scrollTo(x,y)
功能：将窗口中的内容滚动到指定位置。
16.find方法
语法格式：
window.find()
功能：当触发该方法时，将弹出一个“find”(查找)对话窗口，并允许用户在触
发find方法的页面中查找一个字符串。
注：该属性在IE5.5及Netscape6.0中都不支持。
17. back方法
语法格式：
window.back()
功能：模拟用户点击浏览器上的“后退”按钮，将页面转到浏览器的上一页。
说明：仅当当前页面存在上一页时才能进行该操作。
注：IE5.5不支持该方法，Netscape6.0支持。
18. forward方法
语法格式：
window.forward()
功能：模拟用户点击浏览器上的“前进”按钮，将页面转到浏览器的下一页。
说明：仅当当前页面存在下一页时才能进行该操作。
注：IE5.5不支持该方法，Netscape6.0支持。
19. home方法
语法格式：
window.home()
功能：模拟用户点击浏览器上的“主页”按钮，将页面转到指定的页面上。
注：IE5.5不支持该方法，Netscape6.0支持。
20. stop方法
语法格式：
window.stop()
功能：模拟用户点击浏览器上的“停止”按钮，终止浏览器的下载操作。
注：IE5.5不支持该方法，Netscape6.0支持。
21. print方法
语法格式：
window.print()
功能：模拟用户点击浏览器上的“打印”按钮，通知浏览器打开打印对话框打印
当前页。
22. blur方法
语法格式：
window.blur()
功能：从窗口中移出焦点。当与focus方法合用时必须小心，因为可能导致焦点
不断移进移出。
23. focus方法
语法格式：
window.focus()
功能：使窗口中得到焦点。当与blur方法合用时必须小心，因为可能导致焦点不
断移进移出。
24. captureEvent方法
语法格式：
window.captureEvent(Event)
window.captureEvent(事件1|事件2|...|事件n)
功能：捕捉指定参数的所有事件。由于能够捕获哪些由本地程序自己处理的事件
，所以程序员可以随意定义函数来处理事件。如果有多个事件需要捕捉，各事件
之间用管道符“|”隔开。可捕捉的事件类型如下：
Event.ABORT
Event.BLUR
Event.CHANGE
Event.CLICK
Event.DBLCLICK
Event.DRAGDROP
Event.ERROR
Event.FOCUS
Event.KEYDOWN
Event.KEYPRESS
Event.KEYUP
Event.LOAD
Event.MOUSEDOWN
Event.MOUSUEMOVE
Event.MOUSEOUT
Event.MOUSEOVER
Event.MOUSEUP
Event.MOVE
Event.RESET
Event.RESIZE
Event.SELECT
Event.SUBMIT
Event.UNLOAD
25. enableExternalCapture事件
语法格式：
window.enableExternalCapture(event)
功能：enableExternalCapture方法用于捕捉通过参数传入的外部事件。
26. disableExternalCapture事件
语法格式：
window.disableExternalCapture()
功能：取消enableExternalCapture方法的设置，终止对外部事件的捕捉。
27. handleEvent事件
语法格式：
window.handleEvent(event)
功能：触发指定事件的事件处理器。
28. releaseEvent事件
语法格式：
window.releaseEvent(event)
window.releaseEvent(事件1|事件2|...|事件n)
功能：释放通过参数传入的已被捕捉的事件，这些事件是由
window.captureEvent方法设置的，可释放的事件与captureEvent相同。
29. routeEvent事件
语法格式：
window.releaseEvent(event)
功能：把被捕捉类型的所有事件转交给标准事件处理方法进行处理，可转交的事
件与captureEvent相同。
30 scroll事件
语法格式：
window.scroll(X坐标,Y坐标)
功能：将窗口移动到指定的坐标位置。
