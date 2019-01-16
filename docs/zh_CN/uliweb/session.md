# Session使用说明

## 修改说明

* (0.1.7)添加session的某个key可以单独设置超时时间的机制
* (0.1.7)当session[key]找不到时会抛出异常，以前会返回None。

## session简介

什么是session？它就是用来控制会话的一种手段。因为HTTP处理是无状态的，因此需要一
种办法记住当前用户的状态，比如：是否登录，以及记录一些额外的信息。在uliweb中，
session的实现是通过：cookie和后端的session存储来实现的。让我们考虑一下session
的处理过程：


1. 用户访问一个网站，如果以前没访问过，这时cookie中没有session要的东西(如:session_id）。
    后台在接收到一个请求时，如
    果发现没有对应的cookie值，则会自动创建一个session对象，并自动生成一个session_id。
    然后在响应时，向浏览器返回SET_COOKIE，将这个session_id保存到前端。
保存时，session对象和cookie对象的失效期要分别设置，它们可能一样，也可能不一样。
1. 用户再次访问，如果cookie失效了，自然无对应的Session_id的值。如果没失效，则
    会将其上传。后端在处理时，如果session_id没有上送，则按1的步骤来处理。如果有
    值，则从session的后端装入相关的数据。如果没找到，则同样认为session失效，则按
    1的session新建方式进行处理。如果找到，则要检查是否失效，如果失效，按session
    新建的方式进行处理。如果没有失效，则将相关的session信息取出来。


## 功能说明

uliweb中的session目前支持几种设置模式：


1. remember me 功能实现。这种模式一般支持时间比较长的session有效期。
1. 浏览器生存期cookie的设置。这种模式，session还是固定的有效期，但是cookie本身
    会随着浏览器关闭才会失效。但在这种情况下，如果后台Session失效，则session仍
    然会重新创建。
1. 普通session设置。这是最常用的一种设置。cookie和session有效期是一致的。


## uliweb session结构

在uliweb中session功能的实现是由一系统的组件组成的，分别为：


weto/session 模块 --
    它用来完成session类，sesscion对应的cookie类，及底层的存储调用框架，并且提
    供了几个预定义的session存储后端。

uliweb.contrib.session APP --
    它用来自动根据request来初始化session类及session对应cookie的相关参数，创建
    session对象及对应的session cookie对象。当应答时，对session对象进行保存，同
    时向前端发送cookie信息。

前端处理 --
    这里应由用户来实现。在plugs项目中的userman中有一个已经实现的实例。它的作用
    是生成login界面，比如増加remember me的checkbox，然后在views.py中向request.session.member
    设置相应的值。



## session配置说明

在apps/settings.ini中安装session app:


```
[GLOBAL]
INSTALLED_APPS = [
...
'uliweb.contrib.session',
...]
```

在uliweb.contrib.session的settings.ini中已经预设了一些值，简单介绍一下，这些值
你都可以在apps/settings.ini中进行重定义:


```
[SESSION]
type = 'file'
#if set session.remember, then use remember_me_timeout timeout(second)
remember_me_timeout = 30*24*3600
#if not set session.remember, then use timeout(second)
timeout = 3600
force = False
```


type --
    表示session存储后端的类型，目前可用的值有：

    1. file 文件系统
    1. database 数据库
    1. redis redis数据库


remember_me_timeout --
    只在设置了session.remember = True时生效。它是以秒为单位计算的值。一旦session.remember
    为True时，将同时将session.cookie的有效期设置为此值。

timeout --
    适用于一般的session设置。单位为秒。

force --
    session保存时的模式。如果为True，则只要session有修改，有值就会保存。适合于
    每次访问都保存，这样，失效期会向后推迟。如果为False，则只有当有修改时才保存，
    因此，在用户频繁访问的情况下，如果session没有变化，有效期不会向后推迟。



```
[SESSION_STORAGE]
data_dir = './sessions'
```

针对不同的后端要设置的参数。对于不同的后端类型，需要设置的参数不同，上面为'file'
方式的后端参数。下面根据不同类型分别列出所需要的参数:


1. file

    data_dir --
    是session文件保存的目录

    file_dir --
    是session数据文件保存的目录。每个session对象将保存到一个文件中。如果没
        有指定，它将是data_dir + '/session_files'目录。

    lock_dir --
    是读写session文件时所使用的文件锁目录。如果没有指定，它将是data_dir + '/session_files_lock'。


1. datebase

    url --
    sqlalchemy数据库连接串，和ORM一致。详情可以看ORM的文档或sqlalchemy的文档。

    table_name --
    session的表名。缺省为'session'。

    auto_create --
    是否自动创建。缺省为True。


1. redis

    unix_socket_path --
    redis socket 文件名。这是使用socket方式通用时需要进行设置。

    connection_pool --
    使用host, port方式连接。值为一个dict，例如: {'host':'localhost', 'port':6379}




```
[SESSION_COOKIE]
cookie_id = 'uliweb_session_id'
#only enabled when user not set session.cookie.expiry_time and session.remember is False
#so if the value is None, then is means browser session
timeout = None
domain = None
path = '/'
secure = None
```

这个配置是针对session对应的cookie来设的。大部分参数不用太关心，只有timeout。它
可以为None或一个单位为秒的超时时间。timeout缺省为None。

当session.remember=True时，这个timeout将失效，uliweb会使用SESSION下的remember_me_timeout
来替換。当session.remember=False时，如果timeout为None时，表示浏览器会话方式，即
当浏览器关闭时，cookie才会失效。注意，cookie失效并不表示后台的session对象也失效
了，但是由于cookie失效，再访问时，session仍然是失效的。如果timeout为一个整数，则
将按这个时间来设置cookie的失效时间。这个时间一般与SESSION.timeout的时间一致。


```
[GLOBAL]
MIDDLEWARE_CLASSES = [
 'uliweb.contrib.session.middle_session.SessionMiddle',
]
```

这段配置将在安装了uliweb.contrib.session app之后自动生效，将添加一个session处理
的Middleware。它将会创建session和对应的cookie对象，并且在向浏览器应答时保存
session和设置cookie。


## remember me的实现

uliweb的session已经实现了remember的后续处理，剩下的就是如何让其生效。因此对于
用户想要使用uliweb.contrib.session来实现remember me的功能主要要完成以下工作:


1. 前端的界面
1. view的处理

界面很简单，就是添加一个remember之类的checkbox。在views.py的代码如:


```
flag = form.validate(request.params)
if flag:
    request.session.remember = form.rememberme.data
```

先对上传数据进行校验，然后根据form的rememberme的值来设置request.session.remember即可。
上面的代码是用户登录的处理，不过省略了登录相关的代码，大家可以参考使用。


## session的使用

在安装完uliweb.contrib.session之后，在views中的request对象上会附带有一个session
对象，它是由session的middleware创建session后加上去的。用户可以直接通过request.session
来访问。使用时，session对象就是一个dict，因此你可以使用所有字典的方法，如:


```
request.session['a'] = 'b' #向session中添加key为'a'的值'b'
del request.session['a']   #删除key为'a'的内容
request.session.delete()   #删除整个session
```

session中的值一般为普通数据类型，因为当保存session时，会将值进行序列化处理。缺省
是使用cPickle来处理的。

session的保存是由session middleware来完成的，用户一般不用考虑。

通过request.session.cookie可以访问每个session对应的cookie对象，你可以修改它的值
以便保存到浏览器中。

{% alert class=success %}
当使用 `session[key]` 时，如果 key 不存在，则会抛出异常。如果不想抛出异常，可以
使用get方法并指定缺省值。如：

```
session.get(key, default)
```

{% end %}

## session的key超时处理

简单情况下，一个session的超时是整体来设置的。但是有时我们也需要针对session中的个
别key来设置超时，这里可以通过 `set()` 来设置：


```
session.set(key, value, timeout)
```
    
注意，如果timeout超过了session本身的超时时间将无较。
