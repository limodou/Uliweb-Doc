# 全局环境

Uliweb提供了必要的运行环境和运行对象，因此我称之为全局环境。


## 对象

有一些全局性的对象可以方便地从 uliweb 中导入，如:


```
from uliweb import (application, request, response,
    settings, Request, Response)
```

### CONTENT_TYPE_JSON (0.5)

相当于 `application/json`

### CONTENT_TYPE_TEXT (0.5)

相当于 `text/html`

### application

它是用来记录整个Uliweb项目的运行实例，全局唯一。application是 `uliweb.core.SimpleFrame.Dispatcher`
的实例，它有一些属性和方法可以让你使用，例如：


apps --
    将列举出当前application实例所有有效的App名称。它是一个list，比如： `['Hello', 'uliweb.contrib.staticfiles']`

apps_dir --
    当前application的apps的目录。

template_dirs --
    缺省为当前application所有有效的App的template搜索目录的集合。

get_file(filename, dir='static') --
    从所有App下的相应的目录，缺省是从static目录下查找一个文件。并且会先在当前请求对应
    的App下先进行查找，如果没找到，则去其它的App下的相应目录进行查找。

get_config(config_filename) --
    从所有App下的相应的目录,查找指定的ini文件,最后合成一个Ini对象并返回.

parse_tag(xml) (0.5) --
    解析tag的XML文本.输出生成的结果.

parse_tag_xml(xml) (0.5) --
    解析tag的XML文本,输出解析后的字典结构.

template(filename, vars=None, env=None, layout=None) --
    渲染一个模板，会先从当前请求对应的App下先进行查找模板文件。vars是一个dict对象。env
    不提供的话会使用缺省的环境。如果想向模板中注入其它的对象，但不是以vars方式提供，不用
    直接修改env，而是通过dispatch功能，绑定： `prepare_view_env` 主题就可以了。
    它会返回渲染后的结果，是字符串。

    0.5中添加 `layout` 参数, 可以在渲染模板时动态引用父模板.

render(filename, vars, env=None) --
    它很象template，不过它是直接返回一个Response对象，而不是字符串。



### Request

而这里的Request类是基于werkzeug的Request来派生的，区别在于：

{% alert class=info %}
增加了一些兼容性的内容。原来的werkzeug的Request是没有象GET, POST, params, FILES这
样的属性的，它们分别是：args, form, values, files，为了与其它的Request类兼容，我
添加了GET, POST, params, FILES属性。
{% endalert %}

### Response

它也对werkzeug提供的Response进行了派生，区别在于：

{% alert class=info %}
添加了一个write方法。而原werkzeug的Response类有一个stream属性，它有write方法。经过
扩展，可以直接使用write方法，会更方便一些。
{% endalert %}

### request

request 是上面 Request 类的实例的一个代理对象，并不是一个真正的 Request 对象，
response 也是。但是你可以把它当成真正的 Request 和 Response 一样来使用。那么为什
么要这样，为了方便。真正的 Request 和 Response 对象会在收到一个请求时被创建
，然后存放到 local 中，这样不同的线程将有不同实例。为了方便使用，采用代理方式，
这样用户就不用直接调用 local.reuqest 和 local.response ，而是简单使用 request 和
response 就可以根据不同的线程使用不同的对象了。


{% alert class=info %}
request和response是有生存周期的，就是在收到请求时创建，在返回后失效。因此在使用它们
时，要确保你是在它们的生存周期中进行使用的。
{% endalert %}

在讲View的环境时提到过：写一个view方法时有一些对象可以认为是全局的，其中就包括request和
response，但是这两个对象与其它的不同就是因为它是线程相关并且有生存周期的，其它的则是全局唯
一，并且生存周期是整个运行实例的生存周期。这样，在非view函数中想要使用request和response
对象，一种方式就是在view中传入，但是可能有些麻烦，另一种方式就是通过uliweb来导入，这样就
很方便。

request在行为上和Request一样。

request在处理过程中还有其它的一些属性可以使用，分别为：


* `request.appname` 表示当前请求对应的view方法的appname名称
* `request.rule` 表示解析出来的Rule对象。它是Rule的实例。其中 request.rule.endpoint 为对应的view方法路径。
* `request.function` 表示view函数名
* `request.session` 如果安装了session App，则会自动将session对象绑定到request上。
* `request.user` 如果安装了auth App，则会自动将用户对象绑定到request上。


### response

和request一样是一个代理对象。

特殊属性说明(非Werkzeug的属性)：


* `request.template` 用于重新指派渲染使用的模板


### settings

配置信息对象，这个没什么好说的。


## 方法

如:


```
from uliweb import (redirect, json, POST, GET,
    url_for, expose, get_app_dir, get_apps, function,
    functions, decorators, NotFound
    )
```


### redirect


```
def redirect(location, code=302):
```

返回一个Response对象，用于实现URL跳转.


### Redirect(0.1.4)


```
Redirect(location)
```

直接抛出异常，Uliweb捕获后将实现跳转。同时支持在跳转前session的保存。可以作为
redirect的替換。


### json


```
def json(data, **json_kwargs):
```

将一个data处理成json格式，并返回一个Response对象。json_kwargs目前可以允许用户传
入 `content_type` 值。这样在特殊情况下可以将生成的json数据生成:


```
content_type = 'text/html; charset=utf-8'
```

{% alert class=info %}
Ver0.5 增加对content_type的默认处理.当请求头中的 `Accept` 为 `'*/*'` 时, content_type
值为 `application/json`,当 `Accept` 的值中不含有 `application/json` 时, 则值为 `text/plain`,
否则为 `application/json`
{% endalert %}

### expose

详见 [URL映射](url_mapping.html)


### POST

和expose一样，不过限定访问方法为 POST。


### GET

和expose一样，不过限定访问方法为 GET。


### url_for


```
def url_for(endpoint, **values):
```

根据endpoint可以反向获得URL，endpoint可以是字符串格式，如: `Hello.view.index` ， 也可以
是真正的函数对象。

(0.5)如果指定 `_format=True` 则会将URL中的参数转为 `{name}` 的形式.如: URL 为 `/edit/<id>` 执行
`url_for(endpoint, _format=True)` 的结果为 `/edit/{id}`

### get_app_dir


```
def get_app_dir(app):
```

根据一个app名字取得它对应的目录。


### get_apps


```
def get_apps(apps_dir, include_apps=None):
```

根据一个apps目录，分析出所有可用的App的名字列表。


### function(Deprecated)

_建议使用functions_


```
func = function('function_name')
```

用户可以在settings.ini中配置供外部使用的函数路径，通过function可以获得这个函数
的对象。例如在settings.ini中如下配置:


```
[FUNCTIONS]
has_role = 'uliweb.contrib.rbac.has_role'
has_permission = 'uliweb.contrib.rbac.has_permission'
```

这是uliweb.contrib.rbac中的定义的两个方法，key为方法名，value为方法的路径。
通过:


```
has_role = function('has_role')
```

就可以导入真正的函数来使用。


### functions

这是一个对象，它的作用类似于function，不过它是以属性引用的方式来从settings.ini
中的FUNCTIONS中导入方法，如:


```
from uliweb import functions

func = functions.hello
```

相当于:


```
from uliweb import function

func = function('hello')
```


### decorators

它同functions类似的使用方法，但是需要在settings.ini中定义DECORATORS内容，如:


```
[DECORATORS]
check_role = 'uliweb.contrib.rbac.check_role'
check_permission = 'uliweb.contrib.rbac.check_permission'
```

使用方法:


```
from uliweb import decorators

@decorators.check_role('superuser')
@expose('/hello')
def hello():
    #...
```


### json_dumps

用于将Python的数据结构转为json格式的方法。


> json_dumps(obj, unicode=False, encoding='utf-8')
unicode为False时，将会把obj中的unicode值转为encoding编码的串。否则转为unicode
描述形式的串。


### NotFound

404对应的异常类。如果某个链接不存在，将引发这个异常。如果在你的处理中，发现有
不存在的对象，建议使用error来返回。因为NotFound会把当前访问的URL显示出来，可能
不是你想显示的内容。


### HTTPException

通用的HTTP错误异常类。


### Middleware

中间件基类，所有 Middleware 类可以从它派生。


### UliwebError

Uliweb提供了一个通用的异常类 - UliwebError，你可以考虑使用它。

### Storage

一个将字典对象进行封装后，可以使用 `.attr` 形式来使用key的类。

### get_endpoint(0.2.2)

可以获得一个函数表现为URL对应的endpoint值。对于一般函数，它是 `appname.views_XXX.functionname`.
对于类方法，它是 `appname.views_XXX.classname.functionname`。

它主要是用在以类的方式继承另一个类时，对原有方法进行替換时使用。

## 全局对象配置

我们已经可以从 `from uliweb import *` 中获得许多的对象和方法，如果我们也想将
其它的对象可以通过 `from uliweb import xxx` 的形式来导入，要如何做呢？可以
在 `settings.ini` 中添加以下内容:


```
[GLOBAL_OBJECTS]
your_object_name = 'your.object.path'
```

其中，每项中的 key 是对象的名字，值是对象的路径。

