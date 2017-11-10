---
layout: post
title: "easyUI实现类似搜索框关键词自动提示功能"
date: 2016-12-23 10:41
tags: javascript
category: 随笔
description: 使用easyUI实现搜索框信息提示功能。

---
在一个DataGrid里面，搜索行所在位置，实现的效果如下：

![img](/upload/images/easyui-search/demo.gif)


在搜索框中输入部分列名关键字，即可匹配到行的位置。

EasyUI的SearchBox组件只提供了静态搜索框，我们可以使用ComboBox来实现动态的自动关键匹配效果，并且不需要加载远程数据。当前页面的DataGrid的数据我们可以直接在本地获取之。代码如下：
## setp1 创建combobox
```html
<div style="text-align: left;background-color: #E0ECFF;padding-left: 10px;padding-top: 5px;">
       <div id="searchField"  style="width:250px"></div>
</div>
```
## step2 实现本地数据加载
```javascript
$("#searchField").combobox({
    loader: function(param,success,error){
        //获取输入的值
        var q = param.q || "";
        //此处q的length代表输入多少个字符后开始查询
        if(q.length <= 0) return false;
        var rows = $('#fieldList').datagrid('getRows');
        var items = $.map(rows, function (item, index) {
            //匹配条件，忽略大小写
            if(item.fieldName && (item.fieldName.toLowerCase().indexOf(q.toLowerCase()) != -1)){
                return {
                    "fieldName": item.fieldName
                };
            }
        });
        success(items);
    },
    onBeforeLoad:function () {
        //设置placeholder
        $("input[class='textbox-text validatebox-text textbox-prompt']").attr("placeholder","输入标注字段，定位所在行");
    },
    mode: 'remote',
    textField:'fieldName',
    valueField:'fieldName',
    onSelect : function(record){
        var $filedList = $('#fieldList');
        var rows = $filedList.datagrid('getRows');
        if(rows && rows.length > 0){
            for(var r = 0 ; r < rows.length ; r++){
                if(rows[r].fieldName == record.fieldName){
                    $filedList.datagrid('selectRow',r);
                    break;
                }
            }
        }
    }
});
```
load是一个适配器，它将本地数据转换成combobox所需的数据格式：
```javascript
 var rows = $('#fieldList').datagrid('getRows');
        var items = $.map(rows, function (item, index) {
            //匹配条件，忽略大小写
            if(item.fieldName && (item.fieldName.toLowerCase().indexOf(q.toLowerCase()) != -1)){
                return {
                    "fieldName": item.fieldName
                };
            }
        });
```

首先我们通过datagrid的getRows方法获取当前数据表的所有行，然后通过map转换,调用`success(items);`，到此就完成了数据的转换。
## step3、实现行的选中
```javascript
onSelect : function(record){
        var $filedList = $('#fieldList');
        var rows = $filedList.datagrid('getRows');
        if(rows && rows.length > 0){
            for(var r = 0 ; r < rows.length ; r++){
                if(rows[r].fieldName == record.fieldName){
                    $filedList.datagrid('selectRow',r);
                    break;
                }
            }
        }
    }
```
## end
在onSelect事件中，匹配所在行调用selectRow即可。
