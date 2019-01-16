# soap

Uliweb 为了支持soap的处理，提供了 uliweb.contrib.soap app。它依赖于以下模块:


pysimplesoap --
    目前已经内置在uliweb/lib下了，不过这个版本做了一些改动，所以不能简单地使用
    原来的包进行处理。主要改动是对SoapDispatcher类的，一方面可以在构造函数中可
    以传入自定义的exception处理，另一方面是在dispatch()方法中添加了call_function
    参数，以便对方法调用进行封装。

suds --
    这个可以用来做为客户端的使用。不过uliweb中并没有内置它，需要自行安装。也可
    以使用pysimplesoap带的client类。通过导入:

    ```
    from uliweb.lib.pysimplesoap.client import SoapClient
    ```

    不过目前感觉还有一些问题。

httplib2 或 pycurl --
    用于pysimplesoap的客户端处理。



## 配置说明

使用soap服务，首先要在settings.ini中安装uliweb.contrib.soap。例如:


```
[GLOBAL]
INSTALLED_APPS = [
...
'uliweb.contrib.soap',
...
]
```

同时 uliweb.contrib.soap 的settings.ini下已经预设了一些配置项，用户可以根据需要
进行覆盖，如:


```
[DECORATORS]
soap = 'uliweb.contrib.soap.soap'

[EXPOSES]
soap_url = '/SOAP', 'uliweb.contrib.soap.views.soap'

[SOAP]
namespace = None
documentation = 'Uliweb Soap Service'
name = 'UliwebSoap'
prefix = 'pys'
```

首先它定义了一个soap的decorator，因此用户后面可以使用Decorators.soap来使用它，
后面会详细说明。

然后在EXPOSES中定义了外部访问时使用的URL，缺省为 `'/SOAP'` 。你可以根据需要进行
重定义。

然后是soap服务相关的一些描述信息。


namespace --
    用来指明服务内部定义的一些名字空间的说明地址，缺省为None，表示和服务的URL是
    一个地址。

documentation --
    服务的说明信息。

name --
    service的名字

prefix --
    名字空间的前缀，缺省为 'pys'。



## View的定义

服务已经安装好了，下面是定义相关的处理函数。通常你仍然可以定义在views.py中。但是
要注意，这里我们并不使用@expose来处理，而是使用@soap来处理，但是它仍然有view一样
的特性。不过，返回的内容不是HTML而是XML。

举例如下:


```
from uliweb import decorators
from uliweb.contrib.soap import Date, DateTime, Decimal

@decorators.soap('hello', returns={'a':str}, args={'a':str})
def hello(a):
    return 'Hello:' + a
```

在pysimplesoap中定义了一些类型，用来描述数据类型的，uliweb已经将其导到 uliweb.contrib.soap
中，因此可以直接从这里导入。pysimplesoap采用内置类型和自定义类型相结合的方式，
比如常见的有:


```
int, str, float, unicode, bool, short, byte, long,
integer, decimal, dateTime, date
```

因此，对于服务描述使用的wsdl，采用自动生成的方式。

在上面的例子中，使用@decorators.soap来定义一个soap服务。其中soap方法接受以下参
数:


name --
    方法名，如果没有给出，则会自动将所修饰的函数名作为方法名。同时支持类方法的
    方式。

returns --
    返回值描述。它是一个字典，可以进行嵌套描述。上例表示，返回值为一个简单
    类型，tag名为'a'，值是'string'类型。缺省为None。

args --
    输入参数描述。它也是一个字典，定义方式同returns。缺省为None。如果为None，则
    表示缺省为一个参数，可以是任意类型。

doc --
    方法描述说明。缺省为None。


因此在上例中，在soap中定义了三个参数:


```
name = 'hello'
returns = {'a':str}
args = {'a':str}
```

所以这个soap方法的名字叫 'hello'。

soap的下面为所修饰的函数。这里正好为hello，也可以为别的。它需要定义一个参数，名
为 'a'，与args中的定义要一致。

返回值直接为一个字符串。在返回时会自动对它进行封装。


## 类型描述说明

类型描述就是针对returns和args的声明使用的。它支持简单类型手复杂类型。复杂类型
一般指：数组和字典。

当只有一个返回值时，一般定义为:


```
{'name':type}
```

这里type可以是前面说过的对象，如: int, str等。如果只有一个值，一般soap函数直接
返回就可以了，不必是一个字典的形式。

如果是多个值，一般定义为:


```
{'name1':type1, 'name2':type2}
```

定义为字典的形式。返回时应按说明返回一个字典。

如果是一个数组，一般定义为:


```
{'name':[type]}
{'name':{'sub_name':type}}
```

有两种定义方式。第一种会自动转換为第二种形式。但是sub_name的名字是根据type的名
字自动生成的。所以:


```
{'name':[int]}
```

其实就是:


```
{'name':{'int':int}}
```

以上几种数据定义方式可以嵌套使用，从而形成更复杂的数据格式定义。


## 更复杂的一些示例

这个例子实现将上传的整数数组相加后返回:


```
@decorators.soap(returns={'a':int}, args={'a':[int]})
def add(a):
    t = 0
    for x in a:
        t += x['int']
    return t
```

这里a其实形式为: [{'int':v1}, {'int':v2}]

这个示例实现将上传字符串数组统一在后面添加 '中文' 信息:


```
@decorators.soap(returns={'a':[str]}, args={'a':[str]})
def string(a):
    t = []
    for i, x in enumerate(a):
        t.append(x['string'] + u'中文')
    return t
```

建议使用unicode进行中文处理。返回仍是一个数组。


## 客户端示例

以下以suds作为客户端来演示如何访问soap服务。以hello为例，首先准备一个服务端代码。
在某个app中的views.py中添加如下代码:


```
from uliweb import decorators
from uliweb.contrib.soap import Date, DateTime, Decimal

@decorators.soap('hello', returns={'a':str}, args={'a':str})
def hello(a):
    return 'Hello:' + a
```

然后启动服务器，等待测试。

创建一个客户端的测试文件，写入如下代码:


```
#coding=utf8
#import logging
#logging.basicConfig()
#logging.getLogger('suds.client').setLevel(logging.DEBUG)

from suds.client import Client
client = Client('http://localhost:8000/SOAP?wsdl')
print client

result = client.service.hello('limodou')
print 'test1:', result
```

前几行是用来控制日志输出的。suds提供debug状态，可以根据程序处理的中间结果。这里
我已经注释掉了，你可以根据需要来使用。

首先是创建一个Client，只要传入一个wsdl的地址。这个地址就是前面的soap_url加上 `?wsdl` 。

然后我们可以打印看一下这个client是什么东西:


```
Suds ( https://fedorahosted.org/suds/ )  version: 0.4 GA  build: R699-20100913

Service ( UliwebSoapService ) tns="http://localhost:8000/SOAP"
   Prefixes (0)
   Ports (1):
      (UliwebSoap)
         Methods (1):
            hello(xs:string a, )
         Types (0):
```

可以看到我们定义的web service的一些信息，如有什么方法，需要什么参数等。

然后通过 client.service.hello('limodou') 来调用服务，结果会是:


```
test1: Hello:limodou
```

更详细的示例，可以参考 [uliweb-doc/projects/soap_test](https://github.com/limodou/uliweb-doc) 中的代码。

另，通过设置apps/settings.ini:


> [LOG]
> level = 'debug'
也可以看到后台在接收和发送应答时的XML信息。

