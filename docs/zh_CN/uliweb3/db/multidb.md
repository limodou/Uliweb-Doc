# 多数据库连接处理

## 介绍

从 0.1 版本开始，uliorm 就开始支持多数据库连接了，不过在 0.3 版本中又进行了优化。
多数据库连接在这里有两种涵义:


* 同类数据库的不同连接
* 不同类的数据库的不同连接

所以这里没有简单地使用多数据库的说法，而是采用多数据库连接的说法。

在uliorm中多数据库连接的支持分为以下几方面的内容:


* 数据库连接的定义，涉及到settings.ini的配置
* Model如何指定数据库连接，涉及到settings.ini的配置和Model的定义以及执行
* 语句执行以及事务的多数据库连接的支持，包括中间件的支持，线程连接的处理等
* 命令行多数据库的支持


## 数据库连接的定义

首先为了区分不同的数据库连接，并且方便地引用它们，每个连接都需要定义一个名字。
在没有特殊定义的情况下，总是会有一个 `default` 的连接存在。它就是使用原来的
数据库连接的定义。

比如原来我们定义数据库连接就是：

```
CONNECTION = 'mysql://root:limodou@localhost/test2?charset=utf8'
```

它就是 default 连接。当需要定义其它的数据库连接时，可以在 ORM 下定义 CONNECTIONS
如:


```
CONNECTIONS = {
    'test': {
        'CONNECTION':'mysql://root:limodou@localhost/test2?charset=utf8',
        'CONNECTION_TYPE':'short',
    }
}
```

上面定义了一个名为 `test` 的连接。原来的连接还可以保留。

也可以将原来的 `CONNECTION` 注释掉，在 `CONNECTIONS` 中添加一个 `default` 的定义。

## engine_manager

定义好连接，在启动 Uliweb 项目时，系统会自动根据配置创建相应的引擎对象。并且在
`orm` 中会自动创建一个管理对象，名为: `engine_manager` ，它可以象一个dict一样
使用，是用来管理连接的。我们可以通过它得到每个连接的信息，包括配置信息和创建的
相关对象的信息，主要包含:


```
options         连接参数：
                    connection_string:  连接串
                    connection_args:    连接参数
                    debug_log:          是否调试
                    connection_type:    连接类型， long or short
engine          引擎实例。对应实际的数据库引擎对象，比如通过
                sqlalchemy的 ``create_engine()`` 创建的对象
metadata        对应的MetaData对象，可以通过它获得对应的表信息
models          与之相关的所有的Model对象信息
```

比如想要获得 default 的连接对象:


```
engine = engine_manager['default'].engine
```

或者直接使用 `get_connection`


```
engine = get_connection(engine_name='default')
```

如果只是访问缺省的连接，可以:


```
engine = get_connection()
```

因此我们可以了解，一旦项目启动，定义的数据库的引擎对象将直接被创建。但是此时真
正用来与数据库通讯的连接对象还没有创建，它们将随着请求被自动创建和管理。这个连
接对象就是下面的 Session。


### Model的连接设置

在Uliweb中，有多数据库配置需求的可以增加 MODELS_CONFIG 配置，例子可以参考测试项目中的[代码](https://github.com/limodou/uliweb/blob/master/test/test_multidb/apps/blog/settings.ini)，例子：

```
[MODELS]
blog = '#{appname}.models.Blog'
category = '#{appname}.models.Category'

[MODELS_CONFIG]
blog = {'engines':['default', 'b']}
category = {'engines':'b'}
```

当一个Model设置了多个连接名，要么在运行时动态指定，要么uliweb会抛出异常。
所以为了动态指定，uliorm的许多函数和方法都添加了 `engine_name` 参数，比如:


```
Model.connect(engine_name)
Result.connect(engine_name)
```

其中Model类上可以直接调用 `connect()` 来切換连接，它会直接影响后面的结果处理，包括
结果集的处理。这里 `engine_name` 还可以是 `Engine` 对象或 `Connection` 对象。
同时，当返回一个结果集时，在没有获得数据之前，也可以使用结果集的 `connect()` 来切換连接。
这种做法只会影响执行结果。


{% alert class=info %}
原来想实现隐式的连接切換功能，即不要显示地使用象 `connect()` 这样的方法。但是
发现很难做到。
{% endalert %}


### 多数据库下的语句执行与事务处理

在数据库处理中，所有的语句都需要在连接上被执行，事务也是在连接上被处理。不同的
连接意味着不同的处理。考虑到web处理和批处理的方式不同，我们可以考虑以下的场景:


* web处理一般是按请求来执行的，因此一个请求过来，创建一个连接，处理完毕后释放。
    连接可以是长连接或短连接。长连接意味着将使用连接池，因此所谓的释放就是放回池
    子里供下一次使用。而短连接就没有池子，释放就是真正的关闭，下次请求将再次创建。
    而不管长连接还是短连接，处理模式都基本相同。
    为了简化处理，我们可以每次当请求进入时自动创建一个连接，然后启动事务，并且把
    这个连接放到线程环境中，这样所有使用 `do_` 就可以直接利用这个共享的连接和事务
    了。这样的处理只是为了简化。因为有可能一个请求并没有事务处理，甚至不涉及到数据
    操作，这样做有些过头了，不过目前为了简化，uliweb就是这样设计的。
    当支持多数据库连接时，情况有了一些变化。原来可以只自动建一个连接，但是现
    在有可能是有多个连接。那么我们要为所有的连接创建实例，并启动事务吗？因此，
    现在的策略就是只为缺省的连接创建连接实例，并启动事务。对于其它的连接，
    Uliorm 増加了一个名为 `AUTO_DOTRANSACTION` 的配置项，缺省为 `True`.
    它的作用就是当你执行 `do_` 时自动创建连接并启动事务。另一种做法就是使用
    `Begin(ec=engine_name)` 来手工创建连接和事务。目前只要是基本的 SQL 语句
    ，包括： select, update, insert, delete 都是封装到了 `do_` 中了。而象
    `create` 之类的是直接绑定到某个 engine 上，无法直接使用 `do_` , 所以
    自动创建连接和事务一般还是可行的。象建表目前不建议自动创建，所以都是在命
    令行上来执行的，它们都有特殊的处理。
    同时在处理完毕后，也不能只关闭和提交缺省的连接了，需要对所有创建的连接（包括
    自动创建的连接）执行事务提交和关闭。
    不过这些已经通过修改middle_transaction完成了。所以在简单情况下用户不用过份关心
    这些细节。并且这种做法是兼容只有一个数据库连接的情况。
* 命令行和批处理情况有简单的也有复杂的。简单的情况和web请求的处理类似，也可以在
    开始创建相应的连接和事务，在处理完毕后关闭。复杂情况下也可以自已手工创建连接和
    启动事务。目前在命令行处理时有几个关键点：连接获取，Model的获取。Uliorm是完全
    支持脱离WEB环境来使用的。因此我们可以象test_orm.py中那样，自已去创建连接，
    创建Model，然后创建表。在这种情况下，Uliweb启动时做的自动化处理全部无效了，比
    如缺省的 `AUTO_DOTRANSACTION` 的设置, 缺置的 `Begin` 启动事务等。所以我们要自已去
    启动事务。缺省情况下是自动提交的，所以每执行完一条SQL语句就会生效。
    同时uliweb还支持通过调用 make_application 或 make_simple_application 来启动
    应用的实例。后者是专门为命令行准备了，除了个别的参数不能设外，如：debug，其它的
    都一样。一旦启动，你的开发就和WEB区别不大了。所以缺省情况下 `AUTO_DOTRANSACTION`
    是为 `True` 的。因此你执行 `do_` 时会自动启动事务。但是因为它没有 middleware_transaction
    的封装，所以无法在处理完成后自动提交或回滚。这样如果你自已不处理，结果将无法
    保存。对于这种情况，要么我们直接手工启动事务，以明确的事务方式来工作。要么执行
    `set_auto_dotransaction(False)` 来关闭自动生成事务，从而进入 autocommit 状态。
    所以这点要比较注意。建议在命令行处理时，都主动使用事务。

    {% alert class=info %}
    现在在 `make_simplae_application` 中増加了启动时自动将 `AUTO_DOTRANSACTION`
    关闭的设置。所以使用它来启动应用环境直接就是 `autocommit` 的状态。
    {% endalert %}


前面说了，在使用 `do_` 和 `Begin` 时可以自动在创建线程共享的连接。在Uliorm
中维护着一个Local的对象，它上面有 `conn` 和 `trans` 对象，它们各是一个dict
分别保存着线程相关的连接和事务对象。在调用 `do_` 和 `Begin` 时会先检查是否
存在相应的连接和事务对象，如果存在，则直接使用，如果不存在，则创建。这里，还可以
分别传入 engine_name 参数，用来指明检查某个连接名相关的对象是否存在。线程相关的
连接和事务对象存在的目的是为了编程方便。如果所有的SQL都使用 `do_` 会比较简单。
但是因为 Model 把底层SQL的执行封装到了不同的方法中，所以要么它会自动使用 Model
配置的连接名来获得线程连接对象，要么你通过 connect(engine_name) 切換到其它的连接
名上，以便可以获得其它的线程连接对象，目前也可以传入一个真正的连接对象或Engine对象。


### 命令行对多数据库的支持

为了支持多数据库，在所有数据库相关的命令上都増加了 `--engine` 参数，可以用来
切換连接名。缺省是使用 `default` 。影响较大的是 `dump*` 和 `load*` 系列的函数.
原来数据库的数据文件是缺省放在 `./data` 目录下的。现在为了支持多数据库，将会在
它的下面按连接分别创建子目录， 如： `default` 等。所以一旦使用了多数据库支持的
版本，原来的备份和数据装入的路径就发生了变化。


