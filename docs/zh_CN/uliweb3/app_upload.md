# upload(文件上传及显示处理)


## 使用werkzeug进行处理

在Uliweb中对上传有一定的封装处理，下面先介绍一下，不使用Uliweb提供的功能，如何
使用底层的werkzeug来处理文件上传。

当用户在前端通过 `<input type="file">` 来上传一个文件时，在request对象中的request.files中可以得到相应的上传文件。其中是一个类dict的对象，存放多个文件对象属性，例如，
`<input type="file" name="docfile">` 定义了一个docfile字段，当上传后，可以通过:


```
filename = request.files['docfile'].filename
file_obj = request.files['docfile'].stream
```

来分别处理上传的文件名和文件对象。后续你要保存到哪个目录，同时考虑到中文的问题，
你可能需要将上传的unicode文件名转为与操作系统对应的本身编码。因此，为了解决这些
问题，Uliweb提供了upload app来处理上传文件。同时upload app还提供上传后文件的访
问处理，包括X-Sendfile的支持。


## upload 的安装与配置

在apps/settings.ini中的INSTALLED_APPS中添加:


```
'uliweb.contrib.upload'
```

upload缺省提供以下配置:


```
[UPLOAD]
TO_PATH = './uploads'
BUFFER_SIZE = 4096
FILENAME_CONVERTER =
BACKEND =

#X-Sendfile type: nginx, apache
X_SENDFILE = None

#if not set, then will be set a default value that according to X_SENDFILE
#nginx will be 'X-Accel-Redirect'
#apache will be 'X-Sendfile'
X_HEADER_NAME = ''
X_FILE_PREFIX = '/files'

[EXPOSES]
file_serving = '/uploads/<path:filename>', 'uliweb.contrib.upload.file_serving'
```

其中：


TO_PATH --
    为文件要保存的目录。缺省为当前路径下的uploads子目录。一般，当前路径就是你的项目目录。

BUFFER_SIZE --
    保存文件时的块大小。

FILENAME_CONVERTER --
    文件名转換类，它用来处理当文件上传后，在保存文件时使用的文件名。没有给出此配置时缺省是使用UUID来生成文件名，以保证文件名不重复。同时upload还定义了其它的几种转換类，可以根据需要来使用。更详细的说明参见下面具体的说明。

BACKEND --
    upload中文件上传和下载的类定义。如果没有给出，则缺省使用 `FileServing` 类来处理。


目前upload还支持X-Sendfile的处理方式，这是目前apache, nginx中都有的一种方法，不过细节上有所差异。关于X-Sendfile这里有一篇 [文章](http://www.kuigg.com/xiazai-kongzhi) 可以参考。简单地说就是在下载文件时可以对下载的过程进行控制，详情可参加下面的Nginx配置示例。


X_SENDFILE --
    X-Sendfile处理类型，目前只支持Nginx和Apache。根据需要可以输入'nginx'或'apache'。缺省为None，则表示不启动，则文件读取及下载是由Uliweb本身提供的。

X_HEADER_NAME --
    当X_SENDFILE生效时，此选项用于指明将返回web server的头。目前已经知道Nginx和Apache要使用的头标识，其它的web server可以在这里指定。当保持为空时，则根据X_SENDFILE的值自动使用相应的头标识，对于Nginx则为 X-Accel-Redirect ，对于Apache则为 X-Sendfile。

X_FILE_PREFIX --
    传给web server的头中新的URL的前缀。因为这个URL与原始的URL将不一样，所以利用这个前缀可以方便生成新的URL。这个前缀需要与web server中的内部URL相对应。不过对于Nginx和Apache的处理机制不完全相同，对于Nginx的方式，是通过返回一个新的URL，而对于Apache来说，则需要返回一个文件路径，不是一个URL。所以这个项的设置要根据所使用的web server而有所不同。对于Apache则可以认为是目录的前缀。对于Nginx则可以认为是新的URL的前缀。

EXPOSES/file_serving --
    用于定义一个缺省的View函数，以处理上传后的文件的下载。


其实这个配置中，真正与上传有关的就是前两项。后几项都是和下载有关的。在upload中，不仅处理了上传还为了方便处理了下载。只不过，它与静态文件的下载不同，它的下载是可以在非static目录下，并且可以有view的控制参与的。而静态文件是不需要进行控制处理的。


## 文件上传的处理

其实在Uliweb有不同级别的文件上传处理，比如最原始的就是手工处理、然后就是利用upload来处理、再有就是通过generic.py来处理。在处理时，有使用Form来处理的，也可以不使用Form来处理，而是使用request，它们之间有一些差异。generic.py会专门在generic.py的文档进行讲解，这里将根据几种情景进行说明。


### 上传Form的定义

其实使用手工HTML或利用Uliweb提供的Form类来生成Form代码都没有太大关系，基本上是一样的。
简单的话，就是使用Form类了，例如:


```
from uliweb.form import *

class F(Form):
    file = FileField(label='file')

@expose('/show_upload')
def show_upload():
    form = F(action='/upload')
    return {'form':form}
```

上面定义了一个Form类，然后我们在show_upload()中将返回一个dict，用于模板的渲染。这个view方法只处理了显示，上传还没有处理。在F创建时，我们传入action的值用于指定上传文件后的处理URL。


### 不使用upload app进行上传处理

当用户选择了文件，并提交上传后，信息将提交到/upload来。则对应的处理代码示例为:


```
import os

@expose('/upload')
def upload():

    form = F(action='/upload')
    if form.validate(request.params, request.files):
        filename = request.files['file'].filename
        target = os.path.join('./uploads', filename)
        with open(target, 'wb') as f:
            f.write(request.files['file'].stream.read())
        return redirect('/ok')
    else:
        #指定将要使用的模板文件名
        response.template = 'show_upload.html'
        #如果校验失败，则再次返回Form，将带有错误信息
        return {'form':form}
```

先生成保存目标的文件名，然后手工将上传的内容进行保存。不过，这里如果文件名有中文有可能会报错。request中得到的文件名是unicode，你需要将其转为与操作系统相匹配的编码。在Uliweb的全局配置项中提供了一个:


```
[GLOBAL]
FILESYSTEM_ENCODING = None
```

你可以考虑先对其进行配置，然后使用它来处理文件的编码。因此，你需要做的处理主要就是:


1. 生成目标文件名（可能要处理文件名编码的问题）
1. 保存文件

下面再看一看使用upload app的做法


### 使用upload app进行上传处理

首先安装upload app。

然后设置配置项，比如TO_PATH的值，缺省是./uploads。

将上面的代码修改一下:


```
import os

@expose('/upload')
def upload():
    from uliweb import functions

    form = F(action='/upload')
    if form.validate(request.params, request.files):
        functions.save_file(form.file.data.filename, form.file.data.file)
        return redirect('/ok')
    else:
        #指定将要使用的模板文件名
        response.template = 'show_upload.html'
        #如果校验失败，则再次返回Form，将带有错误信息
        return {'form':form}
```

这里使用了upload中提供的save_file函数，它的原型为:


```
save_file(filename, fobj, replace=False, convert=True)
```

这里只提供了两个参数，一个是文件名，一个是文件对象。第三个没有提供，因此如果存在
同名的文件，将不会覆盖，而是自动添加象(1), (2)这样的内容。在save_file中会自动根
据相关的配置项：文件系统编码、保存目录信息来自动生成目标文件名并转換成合适的编码，
然后保存。第4个参数用来控制是否自动进行文件名转換，缺省会转換。其目的是为了让文件
名不重复，没有乱码。

这里save_file是通过functions对象来引用的。在upload中还定义了其它类似的通用方法，
后面我们会看到。

为了方便处理Form字段，upload app还提供了save_file_field函数，具体使用参见下面的
函数说明。


### 放在一起的处理方式

我们可以考虑把显示和上传后的处理放在一起，也可以象这个例子一样，分开不同的URL。如果放在一起，逻辑可以是:


```
def upload():
    from uliweb import functions

    form = F()
    #GET是显示用，POST是提交用
    if request.method == 'GET':
        return {'form':form}
    else:
        #如果提交，则先进行校验，这里是使用Form的方式
        #form有一个validate方法，可以传入多个值，这里将request.files传入
        #以便形成完整的数据集，如果validate返回True，表示校验成功，并且
        #上传的数据将按照Form字段定义的类型已经做了转換
        if form.validate(request.params, request.files):
            functions.save_file(form.file.data.filename, form.file.data.file)
            return redirect('/ok')
        else:
            #如果校验失败，则再次返回Form，将带有错误信息
            return {'form':form}
```


## FileServing 类

upload 把文件和下载的管理组织成了类的形式。这个类就是FileServing，你可以根据需要
从这个类进行派生。在缺省情况下，upload app会自动创建一个default_fileserving，而
前面所看到的UPLOAD的配置项就是这个缺省的文件服务类。同时，基于这个缺省的实例，提
供了下面的一些方法。在简单的情况下，你可以只使用缺省的文件服务对象就够了。

FileServing的说明:


```
class FileServing(object):
    default_config = 'UPLOAD'
    options = {
        'x_sendfile' : ('X_SENDFILE', None),
        'x_header_name': ('X_HEADER_NAME', ''),
        'x_file_prefix': ('X_FILE_PREFIX', '/files'),
        'to_path': ('TO_PATH', './uploads'),
        'buffer_size': ('BUFFER_SIZE', 4096),
        '_filename_converter': ('FILENAME_CONVERTER', None),
    }

    #每个FileServing类有相应的settings配置项。因此FileServing的所有方法
    #都是根据这些配置项计算来的
    #options中的每个值后面都可以使用 obj.xxx 来访问
    #
    #default_config表示会从哪里读配置信息。它会和下面的配置项合成，如，可以
    #和TO_PATH合成为 UPLOAD/TO_PATH 。如果配置项中间有 '/' ，则认为不要进行合成。

    def __init__(self, default_filename_converter_cls=UUIDFilenameConverter, config=None):
        """
        初始化函数。第一个参数表示文件名转換类，缺省使用UUID方式生成文件名
        第二个参数表示读取配置项的名字，与Settings.ini中的section对应
        """
        
    def filename_convert(self, filename):
        """
        对文件名进行转換
        """

    def get_filename(self, filename, filesystem=False, convert=False)
        """
        用于获得一个文件的实际路径。它是根据to_path计算得到的。如果
        filesystem为True，则会将生成的文件名按settings中配置的文件
        系统编码来进行转換。convert参数用于处理是否要进行文件名的转換。
        因此根据参数的不同，它有几种用法:

        1. 根据传入的filename得到对应的实际路径，但文件名不转換为文件系统
           的编码:

           get_filename(filename)

        2. 根据传入的filename得到对应的实际路径，但是文件名转換为文件系统
           的编码:

           get_filename(filename, filestystem=True)

        3. 得到filiename的实际路径，同时进行文件名转換，这样得到的文件名将
           不是原来的文件名:

           get_filename(filename, convert=True)

        前两种主要是用在上传文件后的显示上，这时一般使用的是转換后的文件名。
        第三种是用在上传后保存文件时，先对文件名进行转換。
        """

    def download(self, filename, action='download', x_filename='', real_filename='')
        """
        提供下载处理，支持X-Sendfile的处理。action取值为'download'或
        'inline'，它们分别对应不同的应答头:

        download
            Content-Disposition:attachment; filename=<filename>
        inline
            Content-Disposition:inline; filename=<filename>

        如果action为None，则不显示上面的头信息。

        在这里，我们看到有三个文件名，都有什么用？

        filename一般是从数据库中取出来的文件名，比如我们将文件名保存到
        FileProperty中，当取出来时是Unicode格式的，并且是相对于上传路径
        的相对路径，所以我们要进行转換。

        如果不考虑X-Sendfile的情况，一般我们只提供filename就足够了，因
        为可以自动根据to_path来计算出实际文件路径。不过当文件名并不存在
        于to_path所指定的目录下时，我们还可以提供real_filename参数来指
        明文件实际的路径。

        对于使用了X-Sendfile的情况，又复杂了一些。我们可能还需要指出
        x_filename参数，比如在nginx下，它用来指明X-Accel-Redirect中的
        文件名，而这个文件路径是一个URL，提供Nginx可以找到真正的文件。
        所以x_sendfile其实是一个中间路径。

        所以x_sendfile和real_filename其实不会同时使用。在更底层的filedown
        函数中会进行确实的处理。对于用户来说，如果想实现根据配置不同，
        使用不同的下载方式，则么这些参数最好都提供。
        """

    def save_file(self, filename, fobj, replace=False, convert=True)
        """
        将文件保存在to_path路径下。

        使用convert可以设置要不要转換文件名。
        """

    def save_file_field(self, field, replace=False, filename=None, convert=True)
        """
        根据文件字段来保存。路径处理同save_file
        """

    def save_image_field(self, field, resize_to=None, replace=False, filename=None, convert=True)
        """
        根据图片字段来保存。路径处理同save_file
        """

    def delete_filename(self, filename)
        """
        删除保存在to_path下的文件。
        """

    def get_href(self, filename)
        """
        获取filename对应的URL地址，不是真正的URL信息
        """

    def get_url(self, filename, query_para=None, **url_args)
        """
        获取filename对应的URL。注意，这是一个真正的URL，如果只是想得到URL的
        地址，要使用get_href(filename)

        如果url_args中传入了 title 和 text，则生成的URL形式为

        <a href='xxx' title='title'>text</a>

        如果没有传入，则使用filename代替title和text。

        如果传入了query_para，则它的值将写在href对应的链接后面。query_para
        是一个dict值，如： ``query_para={'alt':'filename.txt'}``

        那么生成的URL可能为:

        <a href='xxx?alt=filename.txt' title='title'>text</a>

        它有什么用，在后面的download你会看到
        """
```

## 自定义FileServing类

如果需要自定义FileServing类，可以从这个类派生。如果没有新増的配置项，只要定义
`default_config` 即可。它表示一个完整的section，用来定义options中所有的值。因此
我们需要在settings.ini中定义完整的配置信息，可以直接拷贝UPLOAD的定义。如果有些值
想采用UPLOAD的值，并且希望和UPLOAD值一起变化，那么可以在定义值的时候引用UPLOAD的值，如：

```
[UPLOAD_TEST]
TO_PATH = UPLOAD.TO_PATH
BUFFER_SIZE = UPLOAD.BUFFER_SIZE
FILENAME_CONVERTER =
BACKEND =
X_SENDFILE = None
X_HEADER_NAME = ''
X_FILE_PREFIX = '/files'
```

上面前两项使用了UPLOAD中的定义值。

简单示例:

```
class FileServingTest(FileServing):
    default_config = 'UPLOAD_TEST'
```

## 如何获取FileServing对象

想要获得FileServing实例，可以先导入这个类，然后创建它的实例。创建时可以传入对应的
配置section的名字，以获得完整的配置，否则使用类中定义的配置。

另一种方法是使用 `get_fileserving(config)` 函数。它会使用FileServing作为BACKEND类，但
是配置项使用你指定的section的名字。这样不会创建新的类，但是实例的参数是可以不同。

## upload app提供方法说明

以下方法都是基于缺省的default_fileserving对象来处理的。


get_fileserving(config) --
    获得缺省的文件上传下载对象。缺省使用UPLOAD的配置。如果传入其它的配置名，将
    使用这个配置来创建FileServing对象。

file_serving(filename) --
    缺省的文件下载函数。它是通过在 settings.ini 中配置了:

    ```
    [EXPOSES]
    file_serving = '/uploads/<path:filename>', 'uliweb.contrib.upload.file_serving'
    ```

    这样所有以 `/uploads` 开头的 URL都会被 `file_serving` 处理，从而提供服务。

    {% alert class=info %}
    在这里还有特殊的扩展处理。在缺省情况下，上传后的文件为了保证唯一性会自动
    对文件名进行转換，具体用什么要看使用哪个文件名生成器处理的。详见下面
    `FilenameConverter` 的有关说明。因此，当下载文件名还是使用转換后的文件
    名会非常不方便。所以这里有一个扩展，就是在传入的URL上添加一个特殊的 query_string，如:

    ```
    xxxxxxxxx.txt?alt=中文.txt
    ```

    这样alt对应的就是想另存为的文件名。这样只要 `<a>` 标签加上 `alt` 信息就可以
    以想要的文件名来保存。
    {% endalert %}

get_filename(filename, filesystem=False, convert=False) --
    用于获得目标文件，即将TO_PATH与filename进行连接。同时，如果给出filesystem为
    True，则将文件名转为文件系统的编码。否则返回的将是unicode。
    convert=False 表示不对文件名进行转換

save_file(filename, fobj, replace=False, convert=True) --
    用于保存一个文件。需要传入文件名和文件对象，这些都可以从request或form字段中
    获得。如果replace设置为True，则表示当存在同名文件时自动覆盖，否则将自动添加
    (1), (2)等内容，以保证文件不重名。save_file会把文件保存到指定的目录下，并根
    据配置项进行相应的文件名编码的转換。

save_file_field(field, replace=False, filename=None, convert=True) --
    用于处理Form中的FileField字段。将自动从FileField中获得对应的文件名和文件对象。
    也可以将文件保存为filename参数指定的文件名。

save_image_field(field, resize_to=None, replace=False, filename=None, convert=True) --
    和save_file_field类似，是用来处理ImageField(图像字段)的。不过，如果你设置了
    resize_to参数的话，它还可以自动对图像进行缩放处理。

delete_filename(filename) --
    删除上传目录下的某个文件。

get_url(filename, query_para=None, **url_args) --
    获得上传目录下某个文件的URL，以便可以让浏览器进行访问。
    query_para 将传入到href属性后面成为query_string.

get_href(filename, **kwargs) --
    获取filename对应的URL地址，不是真正的URL信息



{% alert class=info %}
如果上面的文件名使用的是相对路径，则会根据当前的FileServing对象来决定使用
什么配置信息，比如文件保存的路径。但是如果使用绝对路径，则将使用绝对路径进
行处理。

{% endalert %}

## FilenameConverter类的说明

upload提供了几个用于文件名上传后转换的类，并且可以在settings.ini进行配置，分别说明如下:


1. FilenameConverter:

    ```
    class FilenameConverter(object):
        @staticmethod
        def convert(filename):
            return filename
    ```

最基本的类，对文件名不作任何转換
1. UUIDFilenameConverter
使用UUID方法生成文件名。文件名将保证唯一。
1. MD5FilenameConverter
使用MD5算法生成文件名。具体算法:

    ```
    f = md5(
                md5("%f%s%f%s" % (time.time(), id({}), random.random(),
                                  getpid())).hexdigest(),
            ).hexdigest()
    ```



## BACKEND配置说明

upload在启动时会缺省按照 `UPLOAD/BACKEND` 的定义来生成缺省的fileserving类。如果
没给出，则使用FileServing。通过修改 `BACKEND` ，用户就可以定义自已缺省的文件上传处理类。


## X-Sendfile Nginx配置说明

简单的处理流程可以表示为:


![image](_static/upload_01.png)

以上的处理可以理解为：


1. 用户请求的url在后台经过处理后，由后台处理添加一个内部的头信息，头信息带有一个新的URL，并且返回内容为空，因此真正的内容将由Nginx完成，所以只要添加相应的头信息即可。同时你也可能会返回其它的头信息，如: `'Content-Disposition'`, `'Content-Type'` 等。
1. Nginx在发现 `'X-Accel-Redirect'` 头之后会自动删除，并且根据URL的信息去对应的目录下查找相应的文件，然后返回。
1. 因此用户看到的文件路径有可能和真正存放文件的路径不同。并且，允许后台处理根据需要来决定返回 `'X-Accel-Redirect'` 还是其它的信息，从而可以控制是否真正进行文件下载。一方面可以进行下载控制，另一方面可以对后台文件进行保护。

Nginx的配置如下:


```
location /files {
    internal;
    alias /path/to/files;
}
```

在Nginx的conf文件中添加上面的内容，需要根据需要进行修改。其中:


/files --
    为你将在后台处理中要重新生成的URL的前缀。

internal --
    表示内部使用，用户将无法直接通过URL来访问这个路径。

alias --
    指明/files后对应的文件信息存放的路径。这里还可以考虑使用root，它们的区别就是:

    > 例如URL为 /files/filename，如果配置为 alias /download，则将要读取的文件应该是 /download/filename，而如果配置为 root /download，则将要读取的文件将是 /download/files/filename


启用Nginx进行文件下载处理的配置项应设置为:


```
[UPLOAD]
X_SENDFILE = 'nginx'
```

## functions定义说明

为了方便使用，在settings.ini中的FUNCTIONS中定义了一些全局函数，可以方便使用：

* get_filename
* save_file
* save_file_field
* save_image_field
* delete_filename
* get_url
* get_href
* download
* filename_convert
* get_fileserving
