# 如何使用dialog2打开编辑窗口

使用dialog2打开窗口主要有两种方式：


* 窗口内容已经在前端写好，只要打开即可
* 窗口内容将从后台动态生成，后台需要有相应的view来生成窗口


## 窗口内容从后台生成


### 引用组件


```
{{use "jqutils", ajaxForm=True}}
{{include "inc_jquery_dialog2.html"}}
```

如果你要使用ajax方式来提交dialog2中的Form，应在 `{{use "jqutils"}}` 时加入 `ajaxForm=True` 的
参数，以便自动导入 `jquery.form.js` 进行ajax提交的处理。


### 处理事件点击


```
var dialog = $('<div id="edit_plan"></div>').dialog2(
    {
        autoFocus:false,
        url:url,
        ajaxFormOptions: {
            onSuccess: common_ajaxForm_success(function(data){
                //success process
            }),
            dataType: 'json'
        }
    });
dialog.closest('.modal').addClass('wide-modal');
dialog.bind('dialog2.content-update', function(e){
    //add what you want to do after the content update from server
    dialog.find('input.rselect').each(function(){
        $(this).rselect();
    });
});
```

其中，建议给每个 `<div>` 都设置一个 `id` 属性。

dialog2在创建时时使用的参数参考它的文档 [http://nikku.github.com/jquery-bootstrap-scripting/](http://nikku.github.com/jquery-bootstrap-scripting/).
其中，ajaxFormOptions 中的onSuccess对应成功后的处理。这里 `common_ajaxForm_success`
是在 `inc_jquery_dialog2.html` 中定义的。它用来处理当ajax方式与后台交互时，出错
信息的处理。

缺省的bootstrap的宽度是一定的。如果太窄，可以象上面使用:


```
dialog.closest('.modal').addClass('wide-modal');
```

如果有些js代码需要在显示弹出窗口之后再执行，可以在:


```
dialog.bind('dialog2.content-update', function(e){})
```

中进行处理。


### 更多common_ajaxForm_success的细节

`common_ajaxForm_success` 可以接收参数对象或函数对象。上例为直接传入了函数对象。
你还可以传入一个参数对象，详细定义为:

```
{
success:null,
message: show_message,
done:null,
error:null,
field_prefix:'div_field_',
message_type:'bootstrap'
}
```

其中:


* success, done, error 为成功，完成及出错时的处理函数。缺省不提供。
* message 为应答信息使用的函数。应答信息是整个调用的结果，一般成功时也会返回。而出错
    信息是按字段分别对应的信息。缺省使用poshytip中定义的 `show_message` 。(poshytip
    已经在 `inc_jquery_dialog2.html` 被引用。
* field_prefix 用来处理当出错时来查找出错字段对应的容器元素。缺省是 `div_field_`。
    这种格式是使用uliweb的Generic View自动生成时创建的。所以 `common_ajaxForm_success`
    的处理是与Generic View配套的。其它不使用Generic View的情况下，你可能需要仿照 `common_ajaxForm_success`
    来进行相应的成功或出错处理。
* message_type 用来指明出错时提示的风格。缺省使用 `bootstrap` 。还支持 `tip`
    风格。它将使用 poshytip 控件来显示出错信息。

### Form的处理

因为使用了dialog2插件，如果在提交时，你想使用ajaxForm来处理，则需求按它的要求
在 `<form>` 中添加 `class=ajax` 属性。如果 Form 本身是手写代码很容易添加属性，
那么在自动生成 Form 内容的情况下，就需要一些技巧了。有以下方法可以在生成 Form
时添加class属性：

1. 在view中传入form参数

    一般我们会使用generic.py中的 AddView，EditView等来自动生成Form。因此我们可以
    向它们传入form_args参数，如:
    
    ```
    form_args={'action':request.path, 
    'html_attrs':{'class':'ajax form-horizontal'},
    }                    
    ```

2. 在模板中进行处理

    我们可以利用Form类提供的build属性将Form的生成分成几部分，如：
    
    begin --
        ```
        <form class=xxxx>
        ```
    body --
        Form的内容
    buttons_line --
        Form的按钮行
    end --
        ```
        </form>
        ```
        
    其中，每一部分都可以单独运行。这样我们就可以保留需要的内容，将其它的内容进
    行替換。举例如下：
    
    ```
    <form class="ajax {{=defined('form_class') or 'form-horizontal'}}" 
        method="POST" action="{{=defined('action') or functions.request_url()}}">
    {{build = form.build}}
    {{<< build.body}}
    {{<< build.buttons_line}}
    </form>
    ```
    
    这个例子自定义了 begin 和 end 。在 `<form>` 上添加了 `ajax` 的 class。
    
{% alert class=info %}
原来是使用 `request.url` 来表示请求的URL，但是在群集部署情况下， `request.url`
将是处理请求机器的IP，不是接入机的IP。所以会不正确。而 `functions.request_url()`
会只使用相对路径，避免了这一问题。
{% endalert %}