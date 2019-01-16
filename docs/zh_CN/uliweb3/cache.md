# Cache使用说明


## cache简介

cache是用来临时保存数据的一种机制，在处理某些计算复杂或比较耗时的数据时，如果数
据变动不频繁，如：博客，论坛等以读为主的内容，可以考虑将处理好的数据存在缓存中，
需要时读出来，以提高性能。目前Uliweb的cache支持多种后端，它采用和session相同的
存储机制，所以在uliweb中cache也是放在weto模块中。主要的配置方式与session非常接近，
目前主要支持的后端有：文件、数据库、Redis。


## cache配置说明

在使用前需要先定装 `uliweb.contrib.cache` app。如:


```
[GLOBAL]
INSTALLED_APPS = [
...
'uliweb.contrib.cache',
...]
```

然后根据需要可以覆盖缺省的cache选项:


```
[CACHE]
type = 'file'
expiretime = 3600
```


type --
    表示session存储后端的类型，目前可用的值有：

    1. file 文件系统
    1. database 数据库
    1. redis redis数据库


expiretime --
    超时时间。在使用cache时，用户可以手工设置每项key的超时时间，如果没有给出，
    则使用缺省值。



```
[CACHE_STORAGE]
data_dir = './caches'
file_dir = './caches/cache_files'
lock_dir = './caches/cache_files_lock'
```

针对不同的后端要设置的参数。对于不同的后端类型，需要设置的参数不同，上面为'file'
方式的后端参数。下面根据不同类型分别列出所需要的参数:


1. file

    data_dir --
    是cache文件保存的目录

    file_dir --
    是cache数据文件保存的目录。每个cache对象将保存到一个文件中。如果没
        有指定，它将是data_dir + '/session_files'目录。

    lock_dir --
    是读写cache文件时所使用的文件锁目录。如果没有指定，它将是data_dir + '/session_files_lock'。


1. datebase

    url --
    sqlalchemy数据库连接串，和ORM一致。详情可以看ORM的文档或sqlalchemy的文档。

    table_name --
    cache的表名。缺省为'session'。注意，因为cache使用的是和session一样的底层存储
        所以如果使用数据库后端，一定要设置新的表名。

    auto_create --
    是否自动创建。缺省为True。


1. redis

    unix_socket_path --
    redis socket 文件名。这是使用socket方式通用时需要进行设置。

    connection_pool --
    使用host, port方式连接。值为一个dict，例如: {'host':'localhost', 'port':6379}


1. memcache

    connection --
    格式为 ['host:port']。格式为字符串的list，每个字符串为 'host:port' 的形式。
        缺省为: ['localhost:11211']




## Cache使用说明

cache不象session有一个Middleware，使用cache就象api一样。所以一般使用流程如下:


```
from uliweb import functions

cache = functions.get_cache()
```

通过 `get_cache()` 可以获得一个cache对象。然后使用这个对象可以进行相应的操作。

在调用 `get_cache` 时还可以传入与配置相关的参数，以动态覆盖配置的缺省值，它们是:


storage_type --
    配置项： `'CACHE/type'` ，对应存储类型。改变它，你可以根据需要来指定后端存储方式。缺省为 `'file'` 。

options --
    配置项： `'CACHE_STORAGE'` ，对应 `CACHE_STORAGE` 整个section的内容，它的值应该是一个dict。

expiry_time --
    配置项： `'CACHE/expiretime'` ，对应超时时间，缺省为 `3600` 。

serial_cls --
    配置项： `'CACHE/serial_cls'` ，序列化数据时使用的类，缺省为 `'weto.cache.Serial'` 。



## Cache对象方法


cache.get(key, default=None, creator=None, expire=None) --
    从cache中读取键值

    * key 为要读取的键值
    * default 为缺省值。当key不存在时，将返回此值。
    * creator 当key不存在时，如果给出creator则会调用creator得到一个值，并且将
        此值保存到cache中。如果指定了此参数，default值将会被忽略。
    * expire 与creator联用，用来设置超时时间。

    如果key不存时，并且没有指定default或creator，则会引发异常。

cache.set(key, value, expire=None) --
    向cache设置键值

    * key 为要设置的键值
    * value 为要设置的值
    * expire 为超时时间，如果不设，则使用配置中的缺省值


cache.delete(key) --
    删除某一键值


cache对象也可以象字典一样使用，如:


```
cache['name'] = value
del cache['name']
```

使用字典形式来获取key时，如果不存在，则会引发异常。

