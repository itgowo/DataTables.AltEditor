# DataTables.AltEditor
Support for free editing components for multiple JQuery.DataTables


### 前言
Datatables是一款jquery表格插件。它是一个高度灵活的工具，可以将任何HTML表格添加高级的交互功能。

[DataTables中文网](http://www.datatables.club/)
[DataTables英文网（资料更多）](https://datatables.net/)
[IT狗窝](http://itgowo.com)
### Editor
[Editor](https://editor.datatables.net/)是官方提供的表格编辑宽展组件。
优点：一大堆；
缺点：付费，破解版一般是老版本
今天主角不是官方的组件，所以跳过。

### DataTables.AltEditor（免费，开源）

DataTables最低兼容版本1.10.8

原作者[Github](https://github.com/KasperOlesen/DataTable-AltEditor)
[bug修复版](https://github.com/itgowo/DataTables.AltEditor)

提供简单的添加、编辑和删除Buttons，只需要简单的设置即可：
```
var availableButtons = [
    {text: '添加', name: 'add'},
    {extend: 'selected', text: '编辑', name: 'edit'},
    {extend: 'selected', text: '删除', name: 'delete'}
  ];
```
```  
$(tableId).dataTable({
      columnDefs: columnHeader,
      data: data.data,
      altEditor: true,     // 必须设置为true
      dom: "Bfrtip",
      buttons: availableButtons,
      onAddRow: function (datatable, rowdata, success, error) {
          //点击添加Button触发事件
      },
      onDeleteRow: function (datatable, rowdata, success, error) {
         /点击删除Button触发事件
      },
      onEditRow: function (datatable, rowdata, success, error) {
          //点击编辑Button触发事件
      }
    })
```


### Bug分析
原作者的AltEditor写的很好了（爱动手造轮子的都是好孩子），在一个DataTable情况下，没发现Bug，如果是初始化了多个DataTable，其中任意一个DataTable点击添加（编辑或删除），所有DataTable都会触发onAddRow（onEditRow或onDeleteRow），这点我开始没注意，后来才发现的。

这个Bug可是够大了，于是开始各种找，最后花了两天各种调试才找到。

首先点击添加（编辑或删除）都会弹出窗口（div），但是，每个DataTable弹出的窗口中button都相同，做事件绑定时就会把所有相同id的button绑定上这个事件。也就是说，初始化DataTable很多，一次触发次数也会很多。

### 继续找问题
以一段代码为例
 ```
 $('#altEditor-modal').on('show.bs.modal', function () {
          var btns = '<button type="button" data-content="remove" class="btn btn-default" data-dismiss="modal">关闭</button>' +
            '<button type="button" data-content="remove" class="btn btn-primary" id="editRowBtn" >提交更改</button>';
          $('#altEditor-modal').find('.modal-title').html('编辑记录');
          $('#altEditor-modal').find('.modal-body').html(data);
          $('#altEditor-modal').find('.modal-footer').html(btns);
        });
```
id="editRowBtn" 这个东西万年不变。源代码那些操作提交按钮都是写死的ID。

### 动手解决

好了，问题就是id重复了，那怎么办？在AltEditor库中为了区分不同DataTable，有一个叫做 **namespace** 的属性，一个叫做 **_instance** 每绑定一个DataTable就会有一个全局变量自加1，**namespace** 就是字符串 **“.altEditor"+_instance**。

``` 
  var _instance = 0;
  var altEditor = function (dt, opts) {
    //省略一万字
    this.s = {
      dt: new DataTable.Api(dt),
      namespace: '.altEditor' + (_instance++)
    };
   //省略一万字
  }
```
这样能保证他知道哪个是当前操作的DataTable。
我们把id="editRowBtn"变成id="editRowBtn"+namespace，这样就保证唯一性了。
在不同的方法内找到namespace的变量不太一样， this.s.namespace或者 that.s.namespace。
替换所有需要id的地方十几处。

### 大功告成？
**namespace:".altEditor"+_instance**，这个有问题，namespace字符串虽然拼在了id后面，但是因为包含一个点，导致提交事件不触发，所以最后应该是**namespace:“altEditor"+_instance**就行了。
### 部分改造代码
```
var btns = '<button type="button" data-content="remove" class="btn btn-default" data-dismiss="modal">关闭</button>' +
            '<button type="button" data-content="remove" class="btn btn-primary" id="editRowBtn' + that.s.namespace + '" >提交更改</button>';

 _editRowCallback: function (response, status, more) {
        $("div#altEditor-modal").find("button#editRowBtn" + this.s.namespace).prop('disabled', true);
      }
```

[开发中的开源项目：远程数据调试](https://www.jianshu.com/p/75c2e4f9fde3)
![1.png](https://upload-images.jianshu.io/upload_images/3213604-871579b0e8c90820.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![9.png](https://upload-images.jianshu.io/upload_images/3213604-c257a3d999d561ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
