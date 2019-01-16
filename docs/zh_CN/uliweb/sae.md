# SAE 部署及开发指南

Sina Application Engine(简称sae)是新浪发布的类似于GAE的云环境，目前已经支持PHP
和Python。因为sae是一个受限环境，因此uliweb在它上面的部署和开发有一些特殊的地方，
甚至有一些app是专门为sae开发的。下面让我由浅入深地向你介绍sae环境的部署和使用。
因为sae的python环境也在不断完善中，所以本文档也会不断完善。


## uliweb的安装

你需要安装uliweb 0.1以上版本或svn中的版本，简单的安装可以是:


```
easy_install Uliweb
```

安装后在Python环境下就可以使用uliweb命令行工具了。

目前Uliweb支持Python 2.6和2.7版本。3.X还不支持。


## Hello, Uliweb

让我们从最简单的Hello, Uliweb的开发开始。首先假设你已经有了sae的帐号.


1. 创建一个新的应用，并且选择Python环境。
1. 从svn环境中checkout一个本地目录
1. 进入命令行，切換到svn目录下
1. 创建Uliweb项目:

    ```
    uliweb makeproject project
    ```

会在当前目录下创建一个 `project` 的目录。这个目录可以是其它名字，不过它是和后面要使用的 `index.wsgi` 对应的，所以建议不要修改。
1. 创建 `index.wsgi` 文件，Uliweb提供了一个命令来做这事:

    ```
    uliweb support sae
    ```

这样会在当前目录下创建一个 `index.wsgi` 的文件和 `lib` 目录。注意执行时是在svn的目录，即project的父目录中。
`index.wsgi` 的内容是:

    ```
    import sae
    import sys, os
    
    path = os.path.dirname(os.path.abspath(__file__))
    project_path = os.path.join(path, 'project')
    sys.path.insert(0, project_path)
    sys.path.insert(0, os.path.join(path, 'lib'))
    
    from uliweb.manage import make_application
    app = make_application(project_dir=project_path)
    
    application = sae.create_wsgi_app(app)
    ```

其中 `project` 和 `lib` 都已经加入到 `sys.path` 中了。所以建议使用上面
    的路径，不然就要手工修改这个文件了。
1. 然后就可以按正常的开发app的流程来创建app并写代码了，如:

    ```
    cd project
    uliweb makeapp simple_todo
    
    这时一个最简单的Hello, Uliweb已经开发完毕了。
    ```

1. 如果有静态文件，则需要放在版本目录下，Uliweb提供了命令可以提取安装的app的静态文件:

    ```
    cd project
    uliweb exportstatic ../static
    ```

1. 如果有第三方源码包同时要上传到sae中怎么办，Uliweb提供了export命令可以导出已经
    安装的app或指定的模块的源码到指定目录下:

    ```
    cd project
    uliweb export -d ../lib #这样是导出全部安装的app
    uliweb export -d ../lib module1 module2 #这样是导出指定的模块
    ```

为什么还需要导出安装的app，因为有些app不是放在uliweb.contrib中的，比如第三方
    的，所以需要导出后上传。但是因为export有可能导出已经内置于uliweb中的app，所以
    通常你可能还需要在 `lib` 目录下手工删除一些不需要的模块。
1. 提交代码
访问 `http://<你的应用名称>.sinaapp.com` ，就可看到项目的页面了。


## 数据库配置

Uliweb中内置了一个对sae支持的app，还在不断完善中，目前可以方便使用sae提供的MySql
数据库。它是需要同时安装sqlalchemy才可以运行的。因此我们第一步先在本地安装好
sqlalchemy，然后在版本目录中，导出sqlalchemy到lib下:


```
uliweb export -d ../lib sqlalchemy
```

然后修改 `project/apps/settings.ini` 在 `GLOBAL/INSTALLED_APPS` 最后添加:


```
[GLOBAL]
INSTALLED_APPS = [
...
'uliweb.contrib.sae'
]
```

然后为了支持每个请求建立数据库连接的方式，还需要添加一个Middleware在settings.ini中:


```
[MIDDLEWARES]
transaction = 'uliweb.orm.middle_transaction.TransactionMiddle'
```

 不仅是用来控件事务，同时也可以用来控制是否使用短连接。因为
SAE不支持长连接，所以需要在  中启动短连接的配置项:


```
[ORM]
CONNECTION_TYPE = 'short'
```

这样就配置好了。而相关的数据库表的创建维护因为sae不能使用命令行，所以要按sae的
文档说明通过phpMyAdmin来导入。以后Uliweb会増加相应的维护页面来做这事。


## SAE的受限环境说明

具体内容请参见下面的开发文档，因为它是一个受限环境，所以一些常用的使用方式可能有变化，下面列出我写的一些补充:


1. 数据库连接不能是长连接，超时时间目录为30s，所以才需要安装db_connection middleware，它就是用来保证每个请求创建数据库连接和关闭数据库连接。
1. PIL模块目前还没有预装。PHP环境下的GD还不能用。
1. 虽然有临时目录可以写文件，但是os模块不能执行mkdirs，因此无法创建目录。
1. 文件上传目录只能使用数据库，sae提供的storage目录还无法使用。


## SAE的开发文档

[http://readthedocs.org/docs/sae-python/en/latest/index.html](http://readthedocs.org/docs/sae-python/en/latest/index.html)

