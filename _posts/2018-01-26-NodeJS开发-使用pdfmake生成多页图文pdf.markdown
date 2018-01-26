---
title:  "NodeJS开发-使用pdfmake生成多页图文pdf"   
date:   2018-01-25 09:35:23  
categories: [NodeJS]  
tags: [NodeJS]  

---

<script type="text/javascript"
   src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>
<script type="text/x-mathjax-config"> MathJax.Hub.Config({ TeX: { equationNumbers: { autoNumber: "all" } } }); </script>


## PDFMAKE


---
简介：项目需要，需要支持后台导出pdf格式图文档案，且要支持中文和多页同时导出以及表格。因此选择使用PDFMAKE插件完成。
---


### 1.PDFMAKE安装以及使用

**PDFMAKE : Client/server side PDF printing in pure JavaScript**  
首先可以去[playground](http://pdfmake.org/playground.html)查看pdfmake支持生成的格式以及生成效果，界面如下所示。

![](http://owvctf4l4.bkt.clouddn.com/pdfmake_img1)

[项目地址](https://github.com/bpampuch/pdfmake)

PDFMAKE是纯JS实现，在node里npm install pdfmake 即可安装，详细的安装使用可在项目地址里查询。这里需要注意两点：  

**1.pdfmake本身不可使用中文，如果需要使用中文则需要导入字体文件，如黑体、微软雅黑等，从中选择字体文件最小的，否则会出现打开前端页面时出现卡顿的情况**  
**2.pdfmake不支持插入图片url,只支持使用NodeJS插入本地图片或者使用图片的base64编码**  
如何导入中文字体文件这里不再详述，给出一篇博客作为参考 [导入中文字体参考](http://blog.csdn.net/qq_35056292/article/details/75783392)

### 2.实现思路

生成pdf档案的总体思路如下：  
> 1.前端页面发起生成pdf请求，使用ajax提交uid组必要信息。  
> 2.后台收到请求之后，从数据库查询对应文本内容，图像url等信息，返回给前端界面。
> 3.前端界面解析返回内容，按照格式需要生成pdf档案。  

  中文字体导入后，主要面临的问题有两个，如何插入图片以及如何生成多页pdf。


### 3.插入图片

pdfmake插入图片只支持两种方案，图像的base64编码或者nodejs服务器使用本地文件。其中使用本地文件经测试在一些框架限制下(如thinkJS)不能工作，因此现在网上最多的解决方案为：    
**根据图像URL地址，下载图像后使用canvas进行base64转码，主要应用到canvas.toDataURL()这个函数。**  
大致实现思路如下：   
1.首先在js文件中定义函数getBase64Image(img),使用canvas生成base64编码。  
2.申请newImage()，图像加载完成后执行base64编码，onload函数体内实现后续逻辑。  

``` js
function getBase64Image(img) {
    
    var canvas = document.createElement("canvas");
    canvas.width = img.width;
    canvas.height = img.height;

    var ctx = canvas.getContext("2d");
    ctx.drawImage(img, 0, 0, img.width, img.height);
    var ext = img.src.substring(img.src.lastIndexOf(".")+1).toLowerCase();
    var dataURL;
    
    dataURL = canvas.toDataURL("image/"+ext); //base64编码
    return dataURL;
}
```

使用：

``` js
             var tempImage = new Image();
             tempImage.src = img_url; // image url(http)
             tempImage.crossOrigin = "*";
             tempImage.onload = function(){
             		var img =  getBase64Image(tempImage);
             		//Your Code Here
             	}
```
####  SCRIPT5022：Security Error
这样做会引来两个问题:   
1.跨域的问题：CHROME、SAFAIR、SOGOU浏览器等支持这种方式生成图片base64编码，但由于项目开发中图像实体通常存储在第三方平台或云存储上，这时候使用图像便要跨域下载，windows自带的ie和最新的edge均会面临**安全性策略问题，输出错误信息SCRIPT5022：Security Error**，无法下载图像，这两个浏览器还是有庞大用户群体的，方案不可取。  

2.异步问题：在这种实现方式下，采用传统的ajax链式写法，每张图片使用都要使用onload()，一两张是还好，图片过多时便会出现延时过长，逻辑书写困难等问题。若改为同步下载，便会阻塞服务器主线程，明显不可取。  


为了解决跨越的问题以及异步的问题，考虑如下方式。
> 1.将所需图像存储在云存储之上，生成图像唯一URL存储至数据库。
> 2.前端界面点击生成pdf档案按钮之后，使用ajax提交所需uid，传至后台。  
> 3.创建一个新的本地文件夹用来保存数据，最后是包含在项目文件夹里，在此选择文件缓存文件夹下新建download文件夹。后台查询数据库获取所有展示信息及图片URL，在需要导出pdf时，首先清空文件夹，然后根据URL下载图片到本地文件夹。  
> 4.在后台进行base64转码，和其它文本信息一起传人前端界面进行显示。


由于每次都需要下载图片文件，这里可以使用文件缓存机制设置缓存时间，这种方案较优先，可以减少下载时间，具体实现方式取决于后台所用框架，也可以用fs模块清除文件夹，这里给出参考。

``` js
      /**
       * 首先删除download文件夹下所有内容
       */
      var fs = require('fs'); // 引入fs模块
        var files = [];
        var path = "runtime/download/";
        if(fs.existsSync(path)) {
          files = fs.readdirSync(path);
          files.forEach(function(file, index) {
            var curPath = path + file;
              fs.unlinkSync(curPath);
         //     fs.rmdirSync(curPath);
          });
       }
```

定义图片下载函数downloadFile(),使用request下载文件，这里需要注意下载图像是耗时操作，取决于网络情况，因此在这里使用deferred作为回调函数解决方案， [详细的deferred参考链接](http://www.ruanyifeng.com/blog/2011/08/a_detailed_explanation_of_jquery_deferred_object.html)


``` js
    /*
    * url 网络文件地址
    * filename 文件名
    * callback 回调函数
    */
    async downloadFile(uri,filename)
    {
        var request = require('request');
        var fs = require('fs');
        var stream = fs.createWriteStream(filename);
   //     request(uri).pipe(stream).on('close', callback); 
        function downloadimage(url,filename) {
            let deferred = think.defer();
            request(uri).pipe(stream).on('close',function(){
                deferred.resolve("OK");
            });
            return deferred.promise;
        }
        return await downloadimage(uri,filename);
    }
```

定义文件转base64编码函数base64_encode():

``` js
// function to encode file data to base64 encoded string
    async base64_encode(file) {
    var fs = require('fs');
    // read binary data
    var bitmap = fs.readFileSync(file);
    // convert binary data to base64 encoded string
    return new Buffer(bitmap).toString('base64');
}
```

由于图像url通常为http://xxxxx/xxx/xxx.jpg/jpeg/png, 因此需要字符串切分保留,下载完成后图片文件名更改为最后一个'/'后的内容

``` js
      for(let v of media_list){
        i++;
        var index = v.img_live .lastIndexOf("/");
        v.img_live_path  =  "runtime/download/"+v.img_live.substring(index + 1, v.img_live.length);
        var index2 = v.img_location .lastIndexOf("/");  
        v.img_location_path  =  "runtime/download/" + v.img_location.substring(index + 1, v.img_location.length);
        let res = await this.downloadFile(v.img_live,v.img_live_path)
        let res2 = await this.downloadFile(v.img_location,v.img_location_path);
        v.url_1 = 'data:image/jpeg;base64,' + await this.base64_encode(v.img_live_path);
        v.url_2 =  'data:image/jpeg;base64,' + await this.base64_encode(v.img_location_path);
      }
```

至此，对于每一页需要展示的图像信息均已编码为base64,传至前端显示即可。


### 4.多页PDF生成
pdfmake官网给出的例子均为直接定义好docDefinition显示，而要生成多页pdf信息，在实际的项目需求中，大多数据都是从后台传输过来，因此需要使用代码控制格式生成。给出参考链接：
[多页pdf生成](https://stackoverflow.com/questions/26731494/pdfmake-how-to-create-multiple-page-pdf-with-different-orientation)

按照这个回答，pdfmake支持多页pdf生成,要生成多页，只需要在content里需要分页的地方插入分页内容即可,pageOrientation可以调整横向或者竖向，如下所示：

``` js
  { 
            text: '\n\nNested lists', 
            style: 'header', 
            pageBreak: 'before', 
            pageOrientation: 'portrait' 
  }
```

而要在for循环中插入分页符和内容，择要使用dd.content.push({//your style}),其中dd为定义的文档名,其中push进的内容和正常定义时的一样，只需要减去换行符、空格等即可。实现如下：
  


``` js
	 var temp = 0;
    var dd = {
        pageOrientation: 'landscape',
        content: [
            ],
            styles: {
                header: {
                    bold: true,
                    fontSize: 15
                }
            },
            defaultStyle: {
                font: 'simhei',
                fontSize: 10,
                columnGap:20
            } 

    }
    
   $.ajax({
        url:"merge",
        type: "post",
        async: false,
        data: {
            ids:JSON.stringify(ids)
        },
         dataType: 'JSON',
         success:function(list){
             for(v of list.data)
             {
                temp++;
                dd.content.push({image:img_top,width:765});
                dd.content.push({fontSize:12,margin: [0, 5, 0, 5],text:'媒体简介：'});
                dd.content.push({columns: [{text: v.data,width: 455,height:300},{image: v.url_2,width:120,height:160,margin:[85, 0, 0, 0]}]});
            
                //尾部
                dd.content.push({image:img_bottom,width:765});
                if(temp < list.data.length)
                {
                    dd.content.push({text:'', style: 'header', pageBreak:'before', pageOrientation: 'landscape'});
                }
             }
                             /*设置字体*/
            pdfMake.fonts = {
                Roboto: {
                    normal: 'Roboto-Regular.ttf',
                    bold: 'Roboto-Medium.ttf',
                    italics: 'Roboto-Italic.ttf',
                        bolditalics: 'Roboto-Italic.ttf'
                    },
                    /*这里是加入的微软雅黑字体*/
                simhei: {
                    normal: 'simhei.ttf',
                    bold: 'simhei.ttf',
                    italics: 'simhei.ttf',
                    bolditalics: 'simhei.ttf',
                    }  
            }
            //下载
            pdfMake.createPdf(dd).download("档案.pdf");             
            },
            error:function(data){
                alert("获取数据失败！");
            },
            
    })
```

#### 总结：

1. 使用后台deferred对象来处理下载图片的延迟操作，避开传统的ajax链式写法。
2. 在后台进行图片下载base64编码，解决ie、edge浏览器对跨越的安全性策略限制。
3. 利用pdfmake中的dd.content.push()和分页符,利用循环体解决多页pdf显示以及代码控制生成pdf.

#### 最终效果
一次点击之后，经过短暂延时，可以下载多页pdf。

![](http://owvctf4l4.bkt.clouddn.com/pdfmake_img2)


