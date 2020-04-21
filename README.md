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

。。。。emmmmm。。。接到消息，让去研究网站登录的问题去了。。。。暂停更新。。。
