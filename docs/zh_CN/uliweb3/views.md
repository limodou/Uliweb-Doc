# 视图(View)

在Uliweb中，视图(view)相当于MVC框架中的Controller。


## view模块的定义

在Uliweb中，在一个app的目录下，所有以views开头的文件都将被视为视图模块。当你不使用集中的
URL管理的时候，Uliweb会自动将所有有效的app的视图文件在启动时进行导入，其目的就是为了搜集
所有的URL的定义。因此，只要你按规则进行定义文件名，在其中定义的URL就可以被自动发现。因此像：
views.py, views_about.py都是合法的view模块。


## 基于函数的View方法


### view函数的定义

在Uliweb中，一个view函数可以简单地定义为:


def index(): --
    pass


可以看到，它就是一个普通的函数。目前对于view函数，你只能使用普通的函数，而不能使用类。
每个view函数都应与一个或多个URL定义相匹配，一个完整的view定义如下:


```
@expose('/index')
def index():
    pass
```

如果一个view函数没有使用expose来修饰的话，它将不会被用户所访问。

expose后面是可以没有参数的，如:


```
@expose
def index():
    pass
```

那么这个时候，一个view函数的URL将被定义为:


```
/Appname/view_module_name/view_function_name
```

它是由app的名字，view模块名和view函数名组成的。这就是缺省的URL映射，已经很象是MVC的缺
省映射了。


### view函数的参数

view函数是可以有参数的，但首先你需要先在它的URL中定义参数，如果URL中有参数，则view中就
要定义参数，如果没有则view中一般也没有。带参数的例子:


```
@expose('/documents/<lang>/<path:filename>')
def show_document(filename, lang):
    return _show(request, response, filename, env, lang, False)
```

关于URL的参数定义，参见 [URL映射](url_mapping.html) 的文档。这里可以看出，定义了两个参数：
lang和filename，所以在下面的view函数中也定义了两个参数。

但如果你的URL中没有那么多的参数怎么办？这个可以在URL的定义中解决，如:


```
@expose('/documents/<path:filename>', defaults={'lang':''})
@expose('/documents/<lang>/<path:filename>')
def show_document(filename, lang):
    return _show(request, response, filename, env, lang, False)
```

则在第一个URL的定义中，只有一个filename参数，因此可以使用defaults来定义缺省参数。


### view的环境

在Uliweb中，一个view函数是运行在某种环境中的，当需要调用view函数时，在调用前，我会向
函数的func_globals属性中注入一些对象，这些对象就可以直接在函数中使用了，你不再需要导入。
目前可用的对象有：


* application 这是Uliweb的实例，你可以用它来访问应用的各种属性，如：application.debug
    表示是否处于调用状态，还可以通过它来调用一些方法，如：application.template()来处理模
    板。当然直接导入template也是可以的。不过application.template()已经预设了环境进去。
* request 请求对象。
* response 应答对象。这个对象在传入时是一个空对象，你可以使用它，也可以自行构造一个Response
    的对象进行返回。
* url_for 它是与expose是相反的，它用来生根据view函数生成反向的URL。详情见 [URL映射](url_mapping.html) 的文档。
* redirect 用于重定义处理，后面为一个URL信息。
* error 用于输出错误信息，它将自动查找出错页面。你只要在任何app下的templates中增加
    error.html，然后出错信息可以自已来定制。它也不需要前面加return，也将抛出一个异常。
* settings 是定义在所有有效的app settings.py文件中的配置项。注意，一个配置项的名称必须是
    大写的。
* json 用于将dict对象包装成json格式并返回。


{% alert class=info %}
要注意，以上的环境只能用在view函数中，当view调用其它的方法时，还是需要传入相应的参数。
有些全局性的对象将放在 uliweb/__init__.py 中，因此可以直接导入。详情见 [全局环境](globals.html)
的文档。

{% endalert %}

### view环境的扩展

如果你认为上面的环境还不够，那么你可以直接向env中增加新的对象，然后在view方法中可以通过
env.object的方式来使用它。你需要在某个app的settings.py文件中增加相应的插件处理。如:


```
from uliweb.core.dispatch import bind
@bind('prepare_default_env')
def prepare_default_env(sender, env):
    from uliweb.utils.textconvert import text2html
    env['text2html'] = text2html
```

Uliweb中已经定义了 `prepare_default_env` 这个plugin的插入点，你可以直接使用它。它的
作用就是向env中增加新的对象，如上面是增加了一个新的函数可以用来将文本转为HTML代码。


### view的返回

在Uliweb中，view函数可以返回多种类型的结果。可能为：


* dict 变量。如果返回一个dict的变量，说明你希望由Uliweb自动套用一个模板，这个模板需要在
    templates目录下，并且模板的文件名需要与view函数名相同，后缀为.html。如果你希望使用指
    定的模板文件，则需要利用response对象，将指定的模板名赋给response.template属性就行了。
* response 对象。记得上面说过的吗？你可以直接使用response对象，比如调用它的response.write()
    方法来写入返回的内容。
* 字符串。你可以直接返回一个字符串，这样将被封装为一个普通的文本返回。
* json 对象，使用前面讲的json函数对dict对象进行包装。
* Reseponse实例。你可以主动创建一个Response的实例并返回。

在某些情况下，你可以调用象redirect, error来中止view的运行。


### view模块的入口处理

我建议将不同的view函数按照功能和处理分为不同的文件来存放。

Uliweb支持一种view模块的入口和出口的处理。即你可以在view模块中定义名为 `__begin__` 和
`__end__` 的特殊的方法，它没有参数，但是就象普通的view函数一样，也是在view环境中运行的。
一旦view模块中存在这个特殊的方法，在执行每个view函数之前都会先调用这个函数。因此你可以
把它理解为初始化处理，比如给一些对象赋值。举例如下:


```
def __begin__():
    from uliweb.contrib.auth.views import login

    if not request.user:
        return redirect(url_for(login) + '?next=%s' % url_for(doto_index))
```


## 基于类的View方法

目前Uliweb也支持类的方式来定义view，这样可以有更好的封装性。举个例子:


```
@expose('/user')
class UserView(object):
    def __begin__(self):
        if not request.user:
            return redirect('/login?next=%s' % request.path)

    @expose('/login')
    def login(self):
        #login process
        #URL = /login

    def register(self, name):
        #register process
        #URL = /user/register/<name>

    @expose('add')
    def add_user(self):
        #add process
        #URL = /user/add

    @expose('')
    def list(self):
        #URL = /user here '' will just user UserView class URL prefix '/user'

    def _common(self):
        #this function will not be exposed
```

以上只是一个示例。使用基于类的view除了是以类的方式进行组织外，与一般的view方法
没有本质的区别，但同时又有一些其它的特性。


* 建议使用New Style Class，即从object派生。不需要从特殊的基类派生。
* Class-View也支持类似模块级别的__begin__的处理，但它是一个方法。Uliweb在处理
    Class-View会自动调用。
* 不要使用staticmethod对类方法进行处理。因此类方法可以是一般的方法或classmethod。
* 如果方法名开始为'_'，则这个方法将不会被自动exposed，客户端将不能进行访问。这种
    方式比较适合定义内部的函数。
* 如果使用Class-View并且要在Class上使用expose，你需要安装Python 2.6。
* 在简单情况下，你在类上使用expose('/url')，而类的方法上不使用expose，则会自动对有效的
    方法生成形如：/url/method_name 的链接形式，如果还带参数，则自动生成字符串形式
    的参数。例如上面的register函数。在注释中可以看到。
* 在上面的例子中已经演示了大部分的情况：

    * 类上使用expose
    * 覆盖自动URL的生成，如login()方法。因为这里使用了'/login'，相当于绝对路径。
    * 定义相对URL，如add()方法。
    * 使用缺省expose方式，同时定义了参数，如register()方法。
    * expose('')将直接使用类上的URL。
    * 内部函数，不会被客户端访问到，如_common()方法。
    * 定义了__begin__()方法，可以在执行类方法前先被处理。



{% alert class=info %}
注意，一般不要定义__init__()。因为Uliweb在调用Class-View时，会自动创建类的
实例，如果定义__init__()则不要带参数或全部使用缺省值。

{% endalert %}

{% alert class=info %}
注意，如果在方法上还想使用decorator来进行修饰，如果方法无参数，则顺序不影响
最终的URL生成。如果方法有参数，建议不要使用缺省URL的生成方式，而是主动定义
expose，并且将expose放在所有其它的decorator之上。

{% endalert %}
