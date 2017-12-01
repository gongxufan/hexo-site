---
layout: post
title: "基于Jtopo的网络拓扑编辑器初探"
date: 2016-12-07 10:17
tags: [javascript]
category: 工作日志
description: 基于Jtopo实现的在线网络拓扑设计和编辑，可以创建复杂网络并对网络和设备进行各种操作，提供拓扑的序列化和反序列以及分层编辑操作。
top: 999
---
## 写在前面

本文实现了基于Jtopo的在线网络拓扑设计和编辑，可以创建复杂网络并对网络和设备进行各种操作，提供拓扑的序列化和反序列操作。

为了方便演示，我已经把一个静态DEMO部署到github，[传送门](https://gongxufan.github.io/web-topology/topologyEditor.html)

关于项目访问,因为读取远程json数据,直接用浏览器打开会有安全限制.请将项目放到tomcat启动访问或者直接用idea/webstorm打开项目直接右键打开,如下图所示:
![img](/upload/images/topology/how-to-open.png)

完整代码单独放到了github：https://github.com/gongxufan/web-topology

## 功能简介

基本的操作可以看下面的GIF图，节点从左边图标栏拖拽至编辑区，通过单击节点图标，然后松开鼠标按键则可拖动一条连线。在目标节点单击，怎完成一次连线操作。左栏可以选择多种连线方式。
![img](/upload/images/topology/demo.gif)

这个拓扑编辑器UI是easyUI做的布局，节点的编辑和连线则是基于jTopo二次开发的。jTopo提供了Stage、Scene、Node、Container以及对动画的支持，API使用相对简单易用。缺点是文档缺乏，但是我们可以通过作者提供的DEMO进行熟悉。一旦熟悉其源码，则十分便于二次开发。用来实现一些动画也是轻而易举的。由于是基于canvas的绘画，所以每次更改和操作其实都会重绘整个画布。当中还是有不少可以优化的地方。其实想D3.js也可以走到这样的效果，只是近期工作都在后端就没去探究了。
由于只是摘取个人开发项目的前端部分，由于公司许可原因只能开发这一部分了，有兴趣的可以去我的github上clone下来参考。核心部分代码都在editor.js，提供了节点拖拽、节点连线、布局等方法支持。这个是一个初始版本。代码粗糙，仅供参考。
下面摘取几个重要的地方稍微讲解：
### 编辑器初始化代码
```javascript
//创建JTOP舞台屏幕对象  
   var canvas = document.getElementById('drawCanvas');  
   canvas.width = $("#contextBody").width();  
   canvas.height = $("#contextBody").height();  
   //加载空白的编辑器  
   if(stageJson == "-1"){  
       this.stage = new JTopo.Stage(canvas);  
       this.stage.topoLevel = 1;  
       this.stage.parentLevel = 0;  
       this.modeIdIndex = 1;  
       this.scene=  new JTopo.Scene(this.stage);  
       this.scene.totalLevel = 1;  
   }else{  
       this.stage = JTopo.createStageFromJson(stageJson, canvas);  
       this.scene = this.stage.childs[0];  
   }  
```
我们可以调用JTopo.Stage(canvas)来初始化一个画图区域，接下来就可以使用API进行节点和动画的操作了。整个画图对象的JSON层次结构如下所示：
```json
{  
  "version": "0.4.8",  
  "deviceNum": "19",  
  "wheelZoom": 0.95,  
  "width": 864,  
  "height": 569,  
  "id": "ST172.19.105.52015100809430700001",  
  "topoLevel": "1",  
  "parentLevel": "0",  
  "nextLevel": "0",  
  "childs": [  
    {  
      "elementType": "scene",  
      "id": "S172.19.105.52015100809430700002",  
      "topoLevel": "1",  
      "parentLevel": "0",  
      "nextLevel": "0",  
      "translateX": 106.5,  
      "translateY": 20,  
      "scaleX": 1,  
      "scaleY": 1,  
      "totalLevel": "1",  
      "childs": [  
        {  
          "elementType": "node",  
          "id": "",  
          "topoLevel": 1,  
          "parentLevel": "0",  
          "nextLevel": "0",  
          "x": 211.5,  
          "y": 135,  
          "width": 32,  
          "height": 32,  
          "visible": true,  
          "rotate": 0,  
          "scaleX": 1,  
          "scaleY": 1,  
          "zIndex": 3,  
          "deviceId": "1404683827351.4666",  
          "dataType": "VR",  
          "nodeImage": "tpIcon_9.png",  
          "text": "CS路由器",  
          "textPosition": "Bottom_Center",  
          "templateId": undefined  
        }  
      ]  
    }  
  ]  
}  
```
其结构为：
![img](/upload/images/topology/json.png)

通常我们只需要一个 Scene对象即可管理所有的对象，当然如果要实现更复杂的分组对象管理则可以创建多个Scene对象进行单独管理。同时我们可以调用JTopo.createStageFromJson(stageJson, canvas)方法来讲一个保存好的拓扑结构重新渲染。

### 节点的拖拽
节点的拖拽使用的原生H5的drag&drop来实现
```javascript
/** 
 * 图元拖放功能实现 
 * @param modeDiv 
 * @param drawArea 
 */  
networkTopologyEditor.prototype.drag = function (modeDiv, drawArea, text) {  
    if (!text) text = "";  
    var self = this;  
    //拖拽开始,携带必要的参数  
    modeDiv.ondragstart = function (e) {  
        e = e || window.event;  
        var dragSrc = this;  
        var backImg = $(dragSrc).find("img").eq(0).attr("src");  
        backImg = backImg.substring(backImg.lastIndexOf('/') + 1);  
        var datatype = $(this).attr("datatype");  
        try {  
            //IE只允许KEY为text和URL  
            e.dataTransfer.setData('text', backImg + ";" + text + ";" + datatype);  
        } catch (ex) {  
            console.log(ex);  
        }  
    };  
    //阻止默认事件  
    drawArea.ondragover = function (e) {  
        e.preventDefault();  
        return false;  
    };  
    //创建节点  
    drawArea.ondrop = function (e) {  
        e = e || window.event;  
        var data = e.dataTransfer.getData("text");  
        var img, text,datatype;  
        if (data) {  
            var datas = data.split(";");  
            if (datas && datas.length == 3) {  
                img = datas[0];  
                text = datas[1];  
                datatype = datas[2];  
                var node = new JTopo.Node();  
                node.fontColor = self.config.nodeFontColor;  
                node.setBound((e.layerX ? e.layerX : e.offsetX) - self.scene.translateX - self.config.defaultWidth / 2, (e.layerY ? e.layerY : e.offsetY) - self.scene.translateY - self.config.defaultHeight / 2,self.config.defaultWidth,self.config.defaultHeight);  
                //设备图片  
                node.setImage(context + 'post/web-topology/icon/' + img);  
                //var cuurId = "device" + (++self.modeIdIndex);  
                var cuurId = "" + new Date().getTime() * Math.random();  
                node.deviceId = cuurId;  
                node.dataType = datatype;  
                node.nodeImage = img;  
                ++self.modeIdIndex;  
                node.text = text;  
                node.layout = self.layout;  
                //节点所属层次  
                node.topoLevel = parseInt($("#selectLevel").find("option:selected").val());  
                //节点所属父层次  
                node.parentLevel = $("#parentLevel").val();  
                //子网连接点的下一个层,默认为0  
                node.nextLevel = "0";  
                self.scene.add(node);  
  
                //加载属性面板  
                /* if(self.currDataType) 
                 self.clearOldPanels(self.currDataType) 
                 self.currDeviceId = cuurId; 
                 self.createNewPanels(datatype,self.templateId,self.currentModeId);*/  
                //self.currDataType = datatype;  
                self.currentNode = node;  
            }  
        }  
        if (e.preventDefault()) {  
            e.preventDefault();  
        }  
        if (e.stopPropagation()) {  
            e.stopPropagation();  
        }  
    }  
}  
```
在ondragstart回调方法中传递底图以及必要参数，然后在ondrop进行节点的创建新建节点使用JTopo.Node()构造，设置好相关属性然后通过scene.add(node)加入到Stage。为何执行add操作在界面上就可以看到新的节点了呢？

原因是Stage有一个frames属性，它定义了画布重绘频率1000/frames

#### frames属性
设置当前舞台播放的帧数/秒

默认为:24

frames可以为0，表示：不自动绘制，由用户手工调用Stage对象的paint()方法来触发。

如果小于0意味着：只有键盘、鼠标有动作时才会重绘，例如：stage.frames = -24。

默认画面帧数为24帧，也就是每1000/24ms就会重绘屏幕。后台刷新的代码如下：
```javascript
function() {  
             0 == stage.frames ? setTimeout(arguments.callee, 100) : stage.frames < 0 ? (stage.repaint(),  
                        setTimeout(arguments.callee, 1e3 / -stage.frames)) : (stage.repaint(),  
                        setTimeout(arguments.callee, 1e3 / stage.frames))  
        } ()  
```
setTimeout会调用下面的重绘函数，
```javascript
this.paint = function() {  
      null  != this.canvas && (this.graphics.save(),  
          this.graphics.clearRect(0, 0, this.width, this.height),  
          this.childs.forEach(function(a) {  
                  1 == a.visible && a.repaint(stage.graphics)  
              }  
          ),  
      1 == this.eagleEye.visible && this.eagleEye.paint(this),  
          this.graphics.restore())  
  }  
  ,  
  this.repaint = function() {  
      0 != this.frames && (this.frames < 0 && 0 == this.needRepaint || (this.paint(),  
      this.frames < 0 && (this.needRepaint = !1)))  
  }  
```
paint对遍历所有可见对象 ，依次调用repaint方法。

### 节点连线
这里采用的连线方法是在节点按下鼠标左键，然后松开鼠标则创建一个连线，起点是被点击的节点，终点则随鼠标移动而动态更新。因此单机一个节点松开鼠标则可以看到随鼠标移动的一条连线。然后在某个节点点击左键松开怎完成了两个节点的连线。效果如下：
![img](/upload/images/topology/line.gif)

部分代码实现如下：
![img](/upload/images/topology/code.png)

jTopo支持支线、折线、曲线等的常见，但是折线的拐角处长度现在只能在创建的时候指定。如需动态的秒点创建需要二次开发。代码如下：
```javascript
if(self.lineType == "line"){
    self.link = new JTopo.Link(self.tempNodeA, self.tempNodeZ);
    self.link.lineType = "line";
}else if(self.lineType == "foldLine"){
    self.link = new JTopo.FoldLink(self.tempNodeA, self.tempNodeZ);
    self.link.lineType = "foldLine";
    self.link.direction =  self.config.direction;
}else if(self.lineType == "flexLine"){
    self.link = new JTopo.FlexionalLink(self.tempNodeA, self.tempNodeZ);
    self.link.direction =  self.config.direction;
    self.link.lineType = "flexLine";
}else if(self.lineType == "curveLine"){
    self.link = new JTopo.CurveLink(self.tempNodeA, self.tempNodeZ);
    self.link.lineType = "curveLine";
}  xxxxx
```

## 关于拓扑的保存和加载

因为本文只是拓扑编辑器的前端，后端部分因为商业限制原因暂未开源。而且因为整体项目很大，单独剥离也不容易。其实我们只要知道拓扑的加载和保存逻辑，实现拓扑结构的序列化操作也是很简单的。下面占用一个篇幅集中讨论：

### 加载现有的拓扑图
现在我们拥有一个拓扑结构，也就是文本传送门地址所展示的拓扑图，其序列化结构如下：
```json
{
  "errorInfo": "ok",
  "topologyJson": {
    "version": "0.4.8",
    "wheelZoom": "0.95",
    "deviceNum": "18",
    "width": "1098",
    "height": "671",
    "id": "ST172.19.105.52015100809430700001",
    "topoLevel": "1",
    "parentLevel": "0",
    "nextLevel": "0",
    "childs": [
      {
        "id": "S172.19.105.52015100809430700002",
        "elementType": "scene",
        "translateX": "106.5",
        "translateY": "20",
        "scaleX": "1",
        "scaleY": "1",
        "totalLevel": "1",
        "parentLevel": "0",
        "nextLevel": "0",
        "topoLevel": "1",
        "childs": [
          {
            "id": "L172.19.105.52015100811153300001",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "1136300297928.587",
            "deviceZ": "1406176935353.6848",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811153300002",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "1394105924967.951",
            "deviceZ": "1136300297928.587",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811153300004",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "564241624829.4331",
            "deviceZ": "318016674603.1266",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811153300005",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "1062340763210.1544",
            "deviceZ": "564241624829.4331",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811153300006",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "564241624829.4331",
            "deviceZ": "488323736331.2087",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811153300007",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "564241624829.4331",
            "deviceZ": "827722371280.0199",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811153300008",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "564241624829.4331",
            "deviceZ": "1045863334277.662",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811153300009",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "1262070648024.5728",
            "deviceZ": "564241624829.4331",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811153300010",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "1262070648024.5728",
            "deviceZ": "1136300297928.587",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811214600001",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "253209434749.73596",
            "deviceZ": "564241624829.4331",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811214600002",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "253209434749.73596",
            "deviceZ": "1136300297928.587",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811214600003",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "520008948580.8119",
            "deviceZ": "564241624829.4331",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811214600004",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "520008948580.8119",
            "deviceZ": "1136300297928.587",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811512000002",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "809054608657.865",
            "deviceZ": "1394105924967.951",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100811512000004",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "1250687739831.9912",
            "deviceZ": "1062340763210.1544",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "L172.19.105.52015100911051100001",
            "elementType": "link",
            "x": "0",
            "y": "0",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "undefined",
            "deviceA": "564241624829.4331",
            "deviceZ": "597645745716.1871",
            "lineType": "line",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "2"
          },
          {
            "id": "N172.19.105.52015100809464700002",
            "elementType": "node",
            "x": "198",
            "y": "315",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "attack-network",
            "textPosition": "Bottom_Center",
            "deviceId": "1136300297928.587",
            "dataType": "EC",
            "nodeImage": "tpIcon_5.png",
            "templateId": "NK172.19.105.52015100809464700001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100809465000001",
            "elementType": "node",
            "x": "196",
            "y": "242",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "attackr-router",
            "textPosition": "Bottom_Center",
            "deviceId": "1394105924967.951",
            "dataType": "VR",
            "nodeImage": "tpIcon_9.png",
            "templateId": "RT172.19.105.52015100811241800001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100809500700002",
            "elementType": "node",
            "x": "104",
            "y": "383.5",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "attack-client",
            "textPosition": "Bottom_Center",
            "deviceId": "1406176935353.6848",
            "dataType": "VM",
            "nodeImage": "tpIcon_2.png",
            "templateId": "VT172.19.105.52015100910413200001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100810295700002",
            "elementType": "node",
            "x": "539",
            "y": "321.5",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "intert-networker",
            "textPosition": "Bottom_Center",
            "deviceId": "564241624829.4331",
            "dataType": "EC",
            "nodeImage": "tpIcon_5.png",
            "templateId": "NK172.19.105.52015100810295700001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100810320100002",
            "elementType": "node",
            "x": "719",
            "y": "160.5",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "ws-manage",
            "textPosition": "Bottom_Center",
            "deviceId": "318016674603.1266",
            "dataType": "VM",
            "nodeImage": "mypc.png",
            "templateId": "VT172.19.105.52015100810320100001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100810402700002",
            "elementType": "node",
            "x": "725",
            "y": "225.5",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "ws-browser",
            "textPosition": "Bottom_Center",
            "deviceId": "488323736331.2087",
            "dataType": "VM",
            "nodeImage": "tpIcon_2.png",
            "templateId": "VT172.19.105.52015100810402700001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100810461200002",
            "elementType": "node",
            "x": "726",
            "y": "293.5",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "internal-mssql",
            "textPosition": "Bottom_Center",
            "deviceId": "827722371280.0199",
            "dataType": "VM",
            "nodeImage": "tpIcon_6.png",
            "templateId": "VT172.19.105.52015100810461200001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100810485100002",
            "elementType": "node",
            "x": "726",
            "y": "361.5",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "internal-ossec",
            "textPosition": "Bottom_Center",
            "deviceId": "1045863334277.662",
            "dataType": "VM",
            "nodeImage": "tpIcon_6.png",
            "templateId": "VT172.19.105.52015100810485100001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100810523500002",
            "elementType": "node",
            "x": "369",
            "y": "245.5",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "internet-www",
            "textPosition": "Bottom_Center",
            "deviceId": "1262070648024.5728",
            "dataType": "ECVR",
            "nodeImage": "vr-selfdefined.png",
            "templateId": "RT172.19.105.52015100810523500001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100811153300003",
            "elementType": "node",
            "x": "536",
            "y": "237.5",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "intert-router",
            "textPosition": "Bottom_Center",
            "deviceId": "1062340763210.1544",
            "dataType": "VR",
            "nodeImage": "tpIcon_9.png",
            "templateId": "RT172.19.105.52015100811281200001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100811163200002",
            "elementType": "node",
            "x": "367.5",
            "y": "320",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "internet-blog",
            "textPosition": "Bottom_Center",
            "deviceId": "253209434749.73596",
            "dataType": "ECVR",
            "nodeImage": "vr-selfdefined.png",
            "templateId": "RT172.19.105.52015100811163200001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100811204100002",
            "elementType": "node",
            "x": "370.5",
            "y": "403",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "internet-vpn",
            "textPosition": "Bottom_Center",
            "deviceId": "520008948580.8119",
            "dataType": "ECVR",
            "nodeImage": "vr-selfdefined.png",
            "templateId": "RT172.19.105.52015100811204100001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100811512000001",
            "elementType": "node",
            "x": "194.5",
            "y": "166",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "attack-firewall",
            "textPosition": "Bottom_Center",
            "deviceId": "809054608657.865",
            "dataType": "FW",
            "nodeImage": "tpIcon_4.png",
            "templateId": "undefined",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100811512000003",
            "elementType": "node",
            "x": "534.5",
            "y": "163",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "intert-firewall",
            "textPosition": "Bottom_Center",
            "deviceId": "1250687739831.9912",
            "dataType": "FW",
            "nodeImage": "tpIcon_4.png",
            "templateId": "undefined",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          },
          {
            "id": "N172.19.105.52015100911045400002",
            "elementType": "node",
            "x": "727.5",
            "y": "430",
            "width": "32",
            "height": "32",
            "rotate": "0",
            "scaleX": "1",
            "scaleY": "1",
            "text": "defend-client",
            "textPosition": "Bottom_Center",
            "deviceId": "597645745716.1871",
            "dataType": "VM",
            "nodeImage": "tpIcon_2.png",
            "templateId": "VT172.19.105.52015100911045400001",
            "topoLevel": "1",
            "parentLevel": "0",
            "nextLevel": "0",
            "zindex": "3"
          }
        ]
      }
    ]
  }
}
```
我们在页面上只需要调用
>editor.loadTopology("images/backimg.png",'${templateId}','${topologyId}',"");

其中templateIdtemplateId和topologyId是涉及到后端数据表的主键id，在此完全可以不用在意。loadTopology方法代码如下：
```javascript
/**
 * 加载环境模板ID对应的拓扑图JSON数据结构
 * @param backImg 拓扑图的背景图片
 * @param templateId 环境模板ID
 * @param topologyId 拓扑 表记录ID
 */
propertyPanel.prototype.loadTopology = function (backImg,templateId,topologyId,topoLevel) {
    if(!topoLevel) topoLevel = "";
    var self = this;
    self.showLoadingWindow();
    if (!templateId) {
        templateId = editor.templateId;
    }
    $.ajax({
        url: './topology.html',
        async: false,
        type: "GET",
        dataType: "html",
        data: {
            "templateId":templateId,
            "topologyId":topologyId,
            "topoLevel":topoLevel
        },
        error: function () {
            self.closeLoadingWindow();
            jAlert("服务器异常，请稍后重试..");
        },
        success: function (response) {
            response = JSON.parse(response);
            var err = response.errorInfo;
            // 错误处理
            if (err && err != "ok") {
                if(err == "-1"){
                    editor.init(backImg, templateId, topologyId,"-1","");
                }else if (err == "logout") {
                    handleSessionTimeOut();
                    return;
                } else {
                    self.closeLoadingWindow();
                    jAlert(err);
                }
            } else {
                var topologyJson = response.topologyJson;
                editor.init(backImg, templateId, topologyId, topologyJson,"");
            }
        }
    });
    };
```
首先发送请求获取json数据，然后处理响应结构，真正实现拓扑加载的逻辑在editor.init方法中。这里如果发现请求的拓扑不存在，获取创建一个空白的拓扑图。
下面再看init的实现逻辑：
```javascript
/**
 * 编辑器初始化方法,根据请求返回结果加载空白的或者指定结构的拓扑编辑器
 * @param backImg     背景图片
 * @param templateId  环境模板ID
 * @param topologyId  拓扑记录ID
 * @param stageJson    拓扑JSON结构
 */
networkTopologyEditor.prototype.init = function (backImg,templateId,topologyId,stageJson,templateName) {
    if(!stageJson){
        jAlert("加载拓扑编辑器失败!");
        return;
    }
    this.templateId = templateId;
    this.topologyId = topologyId;
    //创建JTOP舞台屏幕对象
    var canvas = document.getElementById('drawCanvas');
    canvas.width = $("#contextBody").width();
    canvas.height = $("#contextBody").height();
    //加载空白的编辑器
    if(stageJson == "-1"){
        this.stage = new JTopo.Stage(canvas);
        this.stage.topoLevel = 1;
        this.stage.parentLevel = 0;
        this.modeIdIndex = 1;
        this.scene=  new JTopo.Scene(this.stage);
        this.scene.totalLevel = 1;
    }else{
        this.stage = JTopo.createStageFromJson(stageJson, canvas);
        this.scene = this.stage.childs[0];
    }
    $("#parentLevel").val(this.stage.parentLevel);
    //拓扑层次切换
    var options = "";
    for(var i = 1; i <= this.scene.totalLevel ;i++){
        options += '<option value="' + i +'" ';
        if( i == this.stage.topoLevel){
            options += 'selected="selected" ';
        }
        options += '>编辑第' + i + '层</option>';
    }
    $("#selectLevel").append(options);
    //滚轮缩放
    this.stage.frames = this.config.stageFrames;
    this.stage.wheelZoom = this.config.defaultScal;
    this.stage.eagleEye.visible = this.config.eagleEyeVsibleDefault;

    this.scene.mode = "edit";
    //背景由样式指定
    //this.scene.background = backImg;

    //用来连线的两个节点
    this.tempNodeA = new JTopo.Node('tempA');
    this.tempNodeA.setSize(1, 1);
    this.tempNodeZ = new JTopo.Node('tempZ');
    this.tempNodeZ.setSize(1, 1);
    this.beginNode = null;
    this.link = null;
    var self = this;

    //初始化菜单
    this.initMenus();
    //事件处理逻辑在此省略...
    //第一次进入拓扑编辑器,生成stage和scene对象
    if(stageJson == "-1"){
        this.saveToplogy(false);
    }
    //编辑器初始化完毕关闭loading窗口
    this.closeLoadingWindow();
}
```
这里的代码看似复杂，其实就是做了两件事: 
1) 构建基本的stage和scene
2) 调用JTopo.createStageFromJson(stageJson, canvas)即可创建整个拓扑的结构
3) 初始化编辑其菜单
4) 最后绑定各种事件的处理逻辑
5) 最后在获取拓扑数据的时候如果发生错误则创建一个空白的拓扑
```javascript
 //第一次进入拓扑编辑器,生成stage和scene对象
    if(stageJson == "-1"){
        this.saveToplogy(false);
    }
```
至此拓扑的加载和创建一个空白拓扑任务就完成了
### 保存拓扑图
保存拓扑就是把编辑好的拓扑序列化成json，然后上传到服务端保存数据库。按照最简单的思路，我们需要按照Node,Scene,Stage。因此我们定义以下实体对象：

Node
```java
package com.bjhit.cncert.common.beans.models;

import org.codehaus.jackson.map.annotate.JsonSerialize;

/**
 * Created by gongxufan on 2014/11/20.
 */
@JsonSerialize(include=JsonSerialize.Inclusion.NON_NULL)
public class Node
{
    private String id;
    private String elementType;
    private String x;
    private String y;
    private String width;
    private String height;
    private String alpha;
    private String rotate;
    private String scaleX;
    private String scaleY;
    private String strokeColor;
    private String fillColor;
    private String shadowColor;
    private String shadowOffsetX;
    private String shadowOffsetY;
    private String zIndex;
    private String text;
    private String font;
    private String fontColor;
    private String textPosition;
    private String textOffsetX;
    private String textOffsetY;
    private String borderRadius;
    private String deviceId;
    private String dataType;
    private String borderColor;
    private String offsetGap;
    private String childNodes;
    private String nodeImage;
    private String templateId;
    private String deviceA;
    private String deviceZ;
    private String lineType;
    private String direction;
    private String vmInstanceId;
    private String displayName;
    private String vmid;
    private String topoLevel;
    private String parentLevel;
    private String nextLevel;
    //getter/setter
}
```
Scene
```java
package com.bjhit.cncert.common.beans.models;

import org.codehaus.jackson.map.annotate.JsonSerialize;
import java.util.List;

/**
 * Created by gongxufan on 2014/11/20.
 */
@JsonSerialize(include=JsonSerialize.Inclusion.NON_NULL)
public class Scene
{
    private String id;
    private String elementType;
    private String background;
    private String backgroundColor;
    private String mode;
    private String translateX;
    private String translateY;
    private String alpha;
    private String scaleX;
    private String scaleY;
    private String totalLevel;
    private String parentLevel;
    private String nextLevel;
    private String topoLevel;
    //getter/setter
}
```
Stage
```java
package com.bjhit.cncert.common.beans.models;

import org.codehaus.jackson.map.annotate.JsonSerialize;
import java.util.List;
/**
 * 拓扑编辑器舞台对象
 * Created by gongxufan on 2014/11/20.
 */
@JsonSerialize(include=JsonSerialize.Inclusion.NON_NULL)
public class Stage
{
    private String version;
    private String frames;
    private String wheelZoom;
    private String deviceNum;
    private String width;
    private String height;
    private String id;
    private String totalLevel;
    private String topoLevel;
    private String parentLevel;
    private String nextLevel;
    private List<Scene> childs;
    //getter/setter
}
```
定义好了这三个结构，我们在前后端就有了对照关系。接着再看前端如何序列化?
先看前端如何序列化拓扑结构：

```javascript
/**
 * 保存序列化的JSON数据到服务器,为减少请求参数长度,进行了字符串的替换
 */
propertyPanel.prototype.saveToplogy = function (showAlert) {
        editor.mainMenu.hide();
        var self = this;
        this.showLoadingWindow();
        //先删除标尺线
        editor.utils.clearRuleLines();
        //保存container状态
        var containers = editor.utils.getContainers();
        for(var c=0 ; c < containers.length ; c++){
              var temp = [];
              var nodes = containers[c].childs;
              for(var n =0 ; n < nodes.length ; n++){
                  if(nodes[n] instanceof JTopo.Node){
                      temp.push(nodes[n].deviceId);
                  }
              }
            containers[c].childNodes = temp.join(",");
        }
        //设置拓扑当前层次和最大层次数
        var selectLevel = $("#selectLevel");
        var levels = selectLevel.find("option:selected");
        editor.stage.topoLevel = parseInt(levels.eq(0).val());
        if(editor.stage.topoLevel == -1) editor.stage.topoLevel = 1;
        editor.stage.parentLevel = $("#parentLevel").val();
        editor.scene.totalLevel = selectLevel.find("option").size() - 1;
        topologyJSON = editor.stage.toJson();
        if(topologyJSON){
            for(var key in this.jsonCode){//字符压缩
                topologyJSON = topologyJSON.replace(new RegExp('"'+ key +'"',"gm"),this.jsonCode[key]);
            }
            topologyJSON = topologyJSON.replace(new RegExp('",',"gm"),';');
            topologyJSON = topologyJSON.replace(new RegExp('"',"gm"),'@');
            topologyJSON = topologyJSON.replace(new RegExp('undefined',"gm"),'#');
        }
        $.ajax({
            url: context + "topologyManage/saveTopologyJSON",
            async: true,
            type: "POST",
            dataType: "json",
            data: {
                "topologyJSON": topologyJSON,
                "templateId": editor.templateId,
                "topologyId":editor.topologyId
            },
            error: function () {
                self.closeLoadingWindow();
                jAlert("服务器异常，请稍后重试..");
            },
            success: function (response) {
                var err = response.errorInfo;
                // 错误处理
                if (err && err != "ok") {
                    if (err == "logout") {
                        handleSessionTimeOut();
                        return;
                    } else {
                        self.closeLoadingWindow();
                        jAlert(err);
                    }
                } else {
      s/**
 * 保存序列化的JSON数据到服务器,为减少请求参数长度,进行了字符串的替换
 */
propertyPanel.prototype.saveToplogy = function (showAlert) {
        editor.mainMenu.hide();
        var self = this;
        this.showLoadingWindow();
        //先删除标尺线
        editor.utils.clearRuleLines();
        //保存container状态
        var containers = editor.utils.getContainers();
        for(var c=0 ; c < containers.length ; c++){
              var temp = [];
              var nodes = containers[c].childs;
              for(var n =0 ; n < nodes.length ; n++){
                  if(nodes[n] instanceof JTopo.Node){
                      temp.push(nodes[n].deviceId);
                  }
              }
            containers[c].childNodes = temp.join(",");
        }
        //设置拓扑当前层次和最大层次数
        var selectLevel = $("#selectLevel");
        var levels = selectLevel.find("option:selected");
        editor.stage.topoLevel = parseInt(levels.eq(0).val());
        if(editor.stage.topoLevel == -1) editor.stage.topoLevel = 1;
        editor.stage.parentLevel = $("#parentLevel").val();
        editor.scene.totalLevel = selectLevel.find("option").size() - 1;
        topologyJSON = editor.stage.toJson();
        if(topologyJSON){
            for(var key in this.jsonCode){//字符压缩
                topologyJSON = topologyJSON.replace(new RegExp('"'+ key +'"',"gm"),this.jsonCode[key]);
            }
            topologyJSON = topologyJSON.replace(new RegExp('",',"gm"),';');
            topologyJSON = topologyJSON.replace(new RegExp('"',"gm"),'@');
            topologyJSON = topologyJSON.replace(new RegExp('undefined',"gm"),'#');
        }
        $.ajax({
            url: context + "topologyManage/saveTopologyJSON",
            async: true,
            type: "POST",
            dataType: "json",
            data: {
                "topologyJSON": topologyJSON,
                "templateId": editor.templateId,
                "topologyId":editor.topologyId
            },
            error: function () {
                self.closeLoadingWindow();
                jAlert("服务器异常，请稍后重试..");
            },
            success: function (response) {
                var err = response.errorInfo;
                // 错误处理
                if (err && err != "ok") {
                    if (err == "logout") {
                        handleSessionTimeOut();
                        return;
                    } else {
                        self.closeLoadingWindow();
                        jAlert(err);
                    }
                } else {
                    self.replaceStage(editor.templateId,editor.topologyId,showAlert,editor.stage.topoLevel);
                    self.closeLoadingWindow();
                }
            }
        });
    };              self.replaceStage(editor.templateId,editor.topologyId,showAlert,editor.stage.topoLevel);
                    self.closeLoadingWindow();
                }
            }
        });
    };
```
1. 删除标尺线（编辑区域显示的横向和竖向直线），这个对象需要从我们的结构中剔除，调用clearRuleLines方法即可。
2. containers区域的节点对象需要提取出来，因为这些节点在调用序列化的时候已经包含进去了，所以入库的时候只需要把他们的节点id和容器关联即可
3. 如果支持分层编辑，还要保存当前拓扑结构的层级数量
4. 调用editor.stage.toJson()就获取了整个拓扑的结构
5. json替换，仅仅为了减少传输数据
6. 最后将解析的json发送给后台处理      
          
其余细节在此不做赘述，这个项目需要读者具备：H5、easyUI、jTopo、canvas、JSON等基本知识。至于jTopo只需看看其作者提供的DEMO和很少的API就可以很快上手。最好的学习方法是打断点不同的调试跟踪，查看整个工作机制。这里演示的拓扑编辑也是一个很简单不完整的例子，其实还是有很多可以定制化的东西，比如连线以及连线方式都可以进一步定制。

## end
参考资料：
http://www.jtopo.com/

http://www.jeasyui.com/documentation/

http://www.w3school.com.cn/html5/

