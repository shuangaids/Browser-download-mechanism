# Browser-download-mechanism

最近在研究一项浏览器下载机制，或许我的研究不会成功，但是也在尝试中。。。

downloadContract = (params) => {

    fetch.DOWNLOAD_CONTRACT(params).then(res => {
    
        if (res.status) {
        
            this.openDownloadDialog(`http://${res.data.zipFilePath_arr}`,`${res.data.zipFilePath_arr}`)
            
        }
        
    })
    
}

//下载文件

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
