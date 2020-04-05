---
title: "Django 3 博客实例"
date: 2020-04-04T22:34:03+08:00
tags: ["django"]
categories: ["python"]
draft: false
---

学习目标：

- 安装 Django
- 创建并配置 Django 项目
- 创建 Django 应用
- 设计 models 并做数据库迁移
- 创建 models 管理后台
- 学习 QuerySets 和 managers
- 创建 视图(views), 模板(templates), 以及 路由(URLs)
- 向视图中添加分页(pagination)
- 试用 Django 的 class-based 视图

## 安装 Django

这里采用的 Django 版本为 Django 3，它新增了以下特性：

- 提供对Asynchronous Server Gateway Interface(ASGI) 支持，使 Django 具有异步功能。
- 官方提供了的对 MariaDB, PostgreSQL 等新功能的支持

Django 3.0 支持 Python 3.6, 3.7, 3.8。我们这里使用 Python 3.8.2。

官网下载地址： https://www.python.org/downloads/

作为一个简单的开发环境我们不需要使用额外的数据库，Python 3 内置了 SQLite 数据库。Django 也默认使用 SQLite作为开发数据库。如果将写好的项目部署到生产环境，那么建议使用诸如 PostgreSQL, MySQL, Oracle。

Django 官网有相关数据库配置的文档：

https://docs.djangoproject.com/en/3.0/topics/install/#database-installation

### 创建 Python 虚拟环境

从 3.3 版本开始， Python 内置了 venv 库，可以用来创建轻量级的虚拟环境。虚拟环境的好处就是能够在每个项目中使用独立环境，安装需要的包，并且不会破会系统本身 Python 的环境。

创建虚拟环境命令

```shell
python -m venv my_env
```

这会创建一个 my_venv 目录，这里包含了Python执行需要的所有环境，并且你安装的包会被放在 my_env/lib/python3.8/site-packages 目录

启动虚拟环境

```shell
source my_env/bin/activate
```

退出虚拟环境 

```shell
deactivate
```

具体关于 venv 的官方介绍： https://docs.python.org/3/library/venv.html

### 通过 pip 安装 Django

推荐使用 pip 来安装 Django 。Python 3.8 内置了 pip ，但是也可以通过文档查看其他安装方式：https://pip.pypa.io/en/stable/installing/

安装命令

```shell
pip install "Django==3.0.*"
```

检测是否安装成功

```python
>>> import django
>>> django.get_version()
'3.0.5'
```

Django 的其他安装方式可以查看官方文档： https://docs.djangoproject.com/en/3.0/topics/install/

## 创建第一个项目

这个项目的目标是搭建一个简单的博客系统，我们首先通过 Django 命令来创建项目的初始化文件以及目录

```shell
django-admin startproject mysite
```

项目目录结构如下

```shell
mysite
├── manage.py                // 用于与项目进行命令行交互的程序
└── mysite                   // 项目目录
    ├── __init__.py          // 空文件，使 mysite 成为 Python 模块
    ├── asgi.py              // 项目作为ASGI运行的配置文件
    ├── settings.py          // 包含项目的设置以及配置，包括初始默认设置
    ├── urls.py              // 存放 URL patterns， 定义每个URL到视图的映射
    └── wsgi.py              // 项目作为WSGI运行的配置文件
```

Django 的应用包含一个 models.py 的文件， 它用于定义数据模型。每个数据模型对应数据库中一张表。为了项目能够成功运行启动，我们需要创建与 `INSTALLED_APPS` 列出的应用程序的模型关联的数据库表，这时候就需要用到 Django 内置的数据库迁移系统。

运行如下命令：

```shell
cd mysite
(my_env) ➜  mysite python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying sessions.0001_initial... OK
```

上面的命令创建了Django 默认系统的数据库迁移，这之后各个应用的表被初始化到数据库中。

### 运行开发环境

Django 内置了一个轻量级的web服务器，用于我们调试程序。在开发模式下，它会检测代码的实时变化并热重启。不过它也不是万能的，比如新建文件到项目中就需要我们手动重启服务。

启动 Django 项目

```shell
(my_env) ➜  mysite python manage.py runserver
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
April 04, 2020 - 16:03:32
Django version 3.0.5, using settings 'mysite.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

这时候打开 http://127.0.0.1:8000/ 就能看到官方的启动界面了。

开发环境项目的启动也可以定制绑定IP，端口以及Django 需要加载的配置文件：

```shell
python manage.py runserver 127.0.0.1:8001 \--settings=mysite.settings
```

这个服务仅适合开发环境，如果要将Django 部署到生产环境，需要将它作为 WSGI 应用运行，使用到的 web server 有 Apache, Gunicorn, uWSGI ；也可以作为 ASGI 应用，使用到的 web server 为 Uvicorn, Daphne。具体使用情况可以参考官方文档：

 https://docs.djangoproject.com/en/3.0/howto/deployment/wsgi/

https://docs.djangoproject.com/en/3.0/howto/deployment/asgi/

### 项目配置

可以打开 settings.py 查看以下具体内容。这里只是众多Django配置的一些实用部分，所有的配置信息以及默认值可以查看官方文档： https://docs.djangoproject.com/en/3.0/ref/settings/

下面是一些值得关注的部分：

- `DEBUG` 用于开关项目的DEBUG模式。如果设置为 `True` ,Django 会在程序崩溃的时候显示详细的错误信息。如果是生产环境，强烈建议设置为 `False`。否则可能后暴露项目相关的信息。

- `ALLOWED_HOSTS`  当 debug 模式启动或者在跑测试的时候不生效。如果不是上述情况，你需要向它添加允许的 domain/host，这样才能访问Django 网站。这么做主要是为了防止  [HTTP Host header attacks](https://docs.djangoproject.com/en/3.0/topics/security/#host-headers-virtual-hosting)。

- `INSTALLED_APPS` 这个选项告诉 Django 这个网站中的哪些应用需要激活。默认情况下 Django 包含了如下应用

  - django.contrib.admin: 管理站点
  - django.contrib.auth: 认证框架

  - django.contrib.contenttypes: 处理内容类型框架

  - django.contrib.sessions: session 框架
  - django.contrib.messages: 信息框架

  - django.contrib.staticfiles: 静态文件管理框架

- `MIDDLEWARE` 这里包含了所用到的中间件

- `ROOT_URLCONF`  定义项目的根 URL路径

- `DATABASES` 一个字典，包含所有项目中能够用到的数据库的配置，但是每次必须有一个默认数据库。

- `LANGUAGE_CODE` 定义 Django 网站的默认语言

- `USE_TZ` 告诉 Django 是否激活时区支持。

### 项目以及应用

在 Django中，你需要经常根 `项目(project)` `应用(application)` 这两个术语打交道。我们来了解一下的他们的关系。

在 Django 中，项目是指包含某些配置文件的安装目录；而一个应用则是包含 models, views, templates，URLs 的一组目录。应用与项目进行交互从而提供一系列的特殊功能，这些应用可以在其他项目中重新使用。你可以将项目想象成一个网站，这个网站中包含很多提供不同服务的应用，比如博客、wiki、论坛等等。

一个Django 项目的大概结构为：

```shell
DJANGO PROJECT
  ├── APP 1             
  ├── APP 2                           
  ├── APP 3         
  ├── ...             
  ├── APP N 
```

### 创建应用

接下来我们创建一个博客应用，切换到项目根目录，执行以下命令：

```shell
python manage.py startapp blog
```

Django 会为我们生成应用的基本结构，文件的作用见后面的注释

```shell
blog
├── __init__.py 
├── admin.py             // 在这里可以将定义好的 models 注册到 Django 自带的后台管理系统
├── apps.py              // 这里包含应用blog的主要配置
├── migrations           // 这个目录存放应用所用到的数据库迁移文件，它可以追踪model的改变并同步到数据中
│   └── __init__.py
├── models.py            // 定义应用的数据模型
├── tests.py             // 为应用添加测试
└── views.py             // 应用的处理逻辑，每一个视图接收 HTTP 请求，处理请求，返回相应的响应。
```

## 设计博客数据模型

Django 中定义一个model就是从 `django.db.models.Model`中继承实现的子类，它的每一个标识符代表数据库表中的一个字段。Django 将会为每一个在 `models.py` 中定义的 model 创建一个数据库表。同时 Django 也会提供一些API让你方便的对数据库进行操作。

接下来定义一个 `Post` model，修改 blog 应用中的 models.py 文件

```python 
from django.db import models
from django.utils import timezone
from django.contrib.auth.models import User


class Post(models.Model):
    STATUS_CHOICES = (
        ('draft', 'Draft'),
        ('published', 'Published'),
    )
    title = models.CharField(max_length=250)   # 文章的标题, CharField字段对应数据库中VARCHAR类型
    slug = models.SlugField(max_length=250,    # 博客的短标签，使用它能够优化搜索引擎SEO。unique_for_date确保
                            unique_for_date='publish') # 同一时间多个文章不会出现同一个标签
    author = models.ForeignKey(User,                      # 数据库多对一外键，每个文章对应一个作者，每个作者可以有多个文章。
                              on_delete=models.CASCADE,   # on_delete字段表示当相关作者被删除了，他相应的文章也会被删除。
                              related_name='blog_posts')  # 定义反向关系名称，这样方便从User反查他的 blogs 。
    body = models.TextField()                             # 文章主体内容，对应SQL中 TEXT 类型。
    publish = models.DateTimeField(default=timezone.now)  # 文章发布时间，这个时间会自动转换为当前时区的时间。
    created = models.DateTimeField(auto_now_add=True)     # 文章创建时间，auto_now_add 表示时间会在创建文章时被自动添加。
    updated = models.DateTimeField(auto_now=True)         # 文章更新时间，记录最后一次文章更新时间。auto_now 作用同上。
    status = models.CharField(max_length=10,              # 表示文章状态，使用了choices选项，这样具体的值就能在给定中选择。
                              choices=STATUS_CHOICES,
                              default='draft')

    class Meta:                   # 这里定义模型的元数据
        ordering = ('-publish',)  # ordering 表示查询数据的结果排列方式，这里根据publish时间倒序排列，从而使最新的文章在最上面。

    def __str__(self):            # 定义对象的名称，在使用时更具有可读性。
        return self.title
```

更多的字段类型可以查看官方文档：https://docs.djangoproject.com/en/3.0/ref/models/fields/

### 激活应用

要想使得Django能够识别并对应用进行操作，你需要在配置中激活它。这就需要修改 settings.py 中的 INSTALLED_APPS字段

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'blog.apps.BlogConfig',
]
```

其中 `BlogConfig` 就是blog应用 `apps.py` 中定义的类

```python
from django.apps import AppConfig

class BlogConfig(AppConfig):
    name = 'blog'
```

### 创建数据库迁移

定义好模型之后，我们就可以对它进行数据库迁移操作了。前面我们用到了 migrate 指令，它作用于 INSTALLED_APPS  中所有的应用，会将新加的models 以及已经存在的迁移操作同步到数据库中。

这里我们只需要对 Post 这一个 model 进行操作

```shell
(my_env) ➜  mysite python manage.py makemigrations blog
Migrations for 'blog':
  blog/migrations/0001_initial.py
    - Create model Post
```

这个操作仅仅是在 migrations 文件夹下面生成了 0001_initial.py文件，这是migration对数据库操作的一些实现，你可以打开文件看看具体的信息。

我们也可以通过这个文件查看它具体执行的SQL语句。这里可以利用 `sqlmigrate` 命令查看，这个命令仅仅返回具体的SQL语句而不执行。

```shell
(my_env) ➜  mysite python manage.py sqlmigrate blog 0001
BEGIN;
--
-- Create model Post
--
CREATE TABLE "blog_post" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "title" varchar(250) NOT NULL, "slug" varchar(250) NOT NULL, "body" text NOT NULL, "publish" datetime NOT NULL, "created" datetime NOT NULL, "updated" datetime NOT NULL, "status" varchar(10) NOT NULL, "author_id" integer NOT NULL REFERENCES "auth_user" ("id") DEFERRABLE INITIALLY DEFERRED);
CREATE INDEX "blog_post_slug_b95473f2" ON "blog_post" ("slug");
CREATE INDEX "blog_post_author_id_dd7a8485" ON "blog_post" ("author_id");
COMMIT;
```

这里的SQL语句更具使用的不同数据库而有所差别，这里仅仅是SQLite的。需要注意到的一点是，Django 会默认将表的名字定义为 应用名称加model类名，以小写下划线连接。你也可以自定义这个名称，在 Meta 类中添加 db_table 字段。

Django 会自动为每一个model创建一个主键，定义为 id 列，并且这个字段会随着数据的增加而自增。如果有特殊需求，可以在自定的其他字段中添加 `primary_key=True`来让它成为主键。

接下来进行实际的数据库迁移

```shell
(my_env) ➜  mysite python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, blog, contenttypes, sessions
Running migrations:
  Applying blog.0001_initial... OK
```

对于 models.py 的每一次增删改查，我们都需要使用 makemigrations 命令创建迁移文件，保持对model 改变的追踪，然后使用 migrate 命令完成对数据库的同步。

## 创建 models 管理后台

Django 内置了一个开箱即用的后台管理系统，可以用来管理model创建的数据。这个系统我们已经在 `INSTALLED_APPS` 中看到了，就是 `django.contrib.admin` 这个应用。

### 创建用户

使用系统之前，我们需要创建一个管理用户

```shell
(my_env) ➜  mysite python manage.py createsuperuser
Username (leave blank to use 'admin'): admin
Email address: admin@admin.com
Password:
Password (again):
This password is too short. It must contain at least 8 characters.
This password is too common.
This password is entirely numeric.
Bypass password validation and create user anyway? [y/N]: y
Superuser created successfully.
```

### 后台管理系统

现在启动 diango 环境，登录

```shell
python manage.py runserver 
```

打开 http://127.0.0.1:8000/admin/ ，输入刚创建的用户，就可以登录到管理系统了。

默认页面显示的 `Group` , `User` models 是Django 认证框架 `django.contrib.auth` 提供的。点开 Users 就可以看到我们添加的 admin 用户。

### 向管理站点添加 models

接下来我们将 blog models 添加到后台。编辑 blog 应用的 admin.py 文件

```python
from django.contrib import admin
from .models import Post

# Register your models here.
admin.site.register(Post)
```

刷新页面就可以看到我们的Posts 模型，这样我们便可以方便对这个模型内容进行增删改查了。试着使用这个工具添加一篇文章，标题为 'Who was Django Reinhardt'

### 定制模型的显示页面

我们可以在现有基础上对这个页面进行定制，同样是编辑 admin.py 文件

```python 
from django.contrib import admin
from .models import Post

# Register your models here.
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'slug', 'author', 'publish', 'status')
```

这里我们通过继承ModelAdmin类来创建我们自定的Post显示类，并通过装饰器将这个类注册到管理后台。刷新页面查看效果。

接下来我们在这基础上添加更多的改变：

```python
from django.contrib import admin
from .models import Post

# Register your models here.
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'slug', 'author', 'publish', 'status')

    list_filter = ('status', 'created', 'publish', 'author') # 添加右侧的过滤选项
    search_fields = ('title', 'body')           # 添加搜索框
    prepopulated_fields = {'slug': ('title',)}  # 在添加页面中，根据title 自动生成slug
    raw_id_fields = ('author', )                # 添加页面的作者栏，从下拉改为查找，并使用id作为显示
    date_hierarchy = 'publish'                 # 导航链接
    ordering = ('status', 'publish')           # 默认排列依据
```



## 学习 QuerySets 和 managers

Django 内置了强大的对象关系映射，即 object-relational mapper(ORM)，它支持众多市面上主流的数据库系统。

我们可以在 settings.py 中的 DATABASES 字段中定义这些数据库，Django 支持同时使用多个数据库系统。

一旦你定义好了数据模型，Django 提供一个强大的API来对这些模型进行操作。你可以在官方文档中查找相关的操作：

https://docs.djangoproject.com/en/3.0/ref/models/

Django ORM 基于QuerySets，QuerySet是数据查询的集合，可以用它获取数据库中的对象。你可以对QuerySet使用过滤器，通过给定的参数来缩小查询结果的范围。

### 创建对象

我们通过Python shell 来进行操作实验

```shell
python manage.py shell

>>> from django.contrib.auth.models import User 
>>> from blog.models import Post
>>> user = User.objects.get(username='admin')
>>> post = Post(title='Another post',
...             slug='another-post',
...             body='Post body',
...             author=user)
>>> post.save()
```

我们通过用户名获取到了用户对象，具体代码为：

```shell
user = User.objectjs.get(username='admin')
```

get 方法可以从数据库中查询到一个匹配对象，如果数据库中没有匹配，它会报 `DoesNotExist` 异常；如果数据有多个结果，它也会报异常 `MultipleObjectsReturned`。

然后我们通过具体的参数创建了一个 Post 实例，并将获取到的 user 传递进去。

```shell
post = Post(title='Another post',slug='another-post',body='Post body', author=user)
```

需要注意的是，这个时候创建的实例对象仅仅是存在与内存当中，还没有被保存到数据库中；保存到数据库中，我们需要执行 save() 方法。

以上两个操作也可以一步完成，使用 create() 方法：

```shell
Post.objects.create(title='One more post', slug='one-more-post', body='Post body', author=user)
```

### 更新对象

我们对post的标题进行更改，这背后执行的是 UPDATE SQL操作。

```shell
>>> post.title = 'New title'
>>> post.save()
```

### 获取对象

前面我们使用 `Post.objects.get()` 来从数据库中获取一个对象。每一个Django的数据模型至少有一个 `manager`，默认的 `manager` 就是这个 `objects`。通过这个 manager 我们能够获取到QuerySet 。如果需要获取所有的结果，那么我们就需要在这个manager 上执行 all()方法

```shell
>>> all_posts = Post.objects.all()
```

需要注意的是，这行命令并没有真正的去数据库中执行，由于QuerySets的 `lazy` 特性，只有我们强制调用时它才真正的执行。获取结果，我们需要调用这个接收参数：

```shell
>>> all_posts
<QuerySet [<Post: One more post>, <Post: New title>, <Post: Who was Django Reinhardt>]>
```

#### 使用 filter() 方法

如果我们想要获取特定的结果，可以使用 manager 提供的 filter() 方法。比如说我们想要查询在2020年发表的文章：

```shell
>>> Post.objects.filter(publish__year=2020)
<QuerySet [<Post: One more post>, <Post: New title>, <Post: Who was Django Reinhardt>]>
```

也可以添加多个条件，比如查找2020发表的文章，作者是 admin:

```shell
>>> Post.objects.filter(publish__year=2020, author__username='admin')
<QuerySet [<Post: One more post>, <Post: New title>, <Post: Who was Django Reinhardt>]>
```

也可以通过链式查询：

```shell
>>> Post.objects.filter(publish__year=2020).filter(author__username='admin')
<QuerySet [<Post: One more post>, <Post: New title>, <Post: Who was Django Reinhardt>]>
```

#### 使用 exclude() 方法

可以使用 exclude() 方法排除特定的选项，比如：获取所有2020发表的文章并且标题不能以'Who'开头：

```shell
>>> Post.objects.filter(publish__year=2020).exclude(title__startswith='Who')
<QuerySet [<Post: One more post>, <Post: New title>]>
```

#### 使用 order_by() 方法

可以通过不同的字段来排列获取的结果：

```shell
>>> Post.objects.order_by('title')
<QuerySet [<Post: New title>, <Post: One more post>, <Post: Who was Django Reinhardt>]>
```

逆序排列在字段前面添加 '-'

```shell
>>> Post.objects.order_by('-title')
<QuerySet [<Post: Who was Django Reinhardt>, <Post: One more post>, <Post: New title>]>
```

### 删除对象

删除对象很简单，只需要在对象实例上面调用 delete() 方法。

```shell
>>> post = Post.objects.get(id=1)
>>> post.delete()
```

### 自定义 model managers

前面我们提到过，每一个model 都有一个默认的manager，它可以从数据库中获取所有的对象。我们也可以自定义一些特殊的manager。比如接下来我们创建一个自定的manager，用来获取所有 published 状态的文章。

有两种方法可以添加自定义manager:

- 在现有manager 上添加额外的 manager 方法，比如说 Post.objects.my_manager()
- 通过修改manager 返回的初始 QuerySet 来新建一个manager，比如说 Post.my_manager.all()

我们定义的这个manager 可以实现通过 Post.published.all() 来获取文章。

编辑blog 应用中的models.py 文件

```python
class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status='published')
      
class Post(models.Model):
  # ....
  objects = models.Manager() # The default manager.
  published = PublishedManager() # Our custom manager.
```

第一个定义的manager就是默认的的manager，你可以通过在Meta 中添加 `detault_manager_name` 来修改默认manager。如果没有特殊指定，Django会自动创建默认的 manager `objects`。如果你需要保留 `objects` manager，那么就需要在代码中特别指出，比如我们上面的代码，就将默认和我们新建的都添加到了Post 模型中。

接下来我们使用这个自定义的manager

```shell
python manage.py shell
```

导入Post 并使用manager 查询以'Who'开头的文章

```shell
>>> from blog.models import Post
>>> Post.published.filter(title__startswith='Who')
<QuerySet [<Post: Who was Django Reinhardt>]>
```

查询之前请确保文章的状态为 publised

## 创建文章列表以及详情页视图

学会了ORM操作之后，我们就能够使用它来创建blog应用的视图了。Django的视图说白了就是一个接收网络请求，返回网络响应的Python函数。所有的你想要让它返回特定结果的处理逻辑都在视图中创建。

整个逻辑的编写过程分三步走：

- 创建应用视图
- 定义对应每个视图的 URL 匹配
- 创建用于渲染视图的HTML模板

每一个视图都会渲染一个模板，并将相关的参数传递给它，最后将渲染好的结构通过HTTP response 封装返回给客户端。

### 创建 list and detail 视图

首先来创建显示所有文章的视图，编辑blog应用的 views.py 文件：

```python
from django.shortcuts import render, get_object_or_404
from .models import Post

def post_list(request):
    posts = Post.published.all()
    return render(request, 'blog/post/list.html', {'posts': posts})
```

这里 `post_list`就是一个视图函数，它接收的唯一参数就是request，并且这个参数是所有视图函数都必须的。这个视图中，我们获取所有为 published 状态的文章。

最后使用 render() 函数将数据与模板文件合并渲染，它返回一个 HttpResponse 对象，这个对象包含了渲染好的文本（通常来说是 HTML 代码）。render() 函数获取了 请求上下文，所以模板上下文提供的所有变量都能被指定的模板接收使用。

接着我们创建第二个视图，用来显示文章详情。

```python
def post_detail(request, year, month, day, post):
    post = get_object_or_404(Post, slug=post,
                                   status='published',
                                   publish__year=year,
                                   publish__month=month,
                                   publish__day=day)
    return render(request,
                  'blog/post/detail.html',
                  {'post': post})
```

这个视图除了 request 还接收了 year, month, day, post 等参数，通过这些参数来获取满足条件的文章。这里我们使用了Django提供的便捷函数 `get_object_or_404()`，这个函数的结果是要么获取到我们需要的对象，要么返回一个HTTP 404。最后我们同样使用 render 函数渲染了结果。

### 对视图函数添加 URL 匹配

URL 匹配可以让不同的URLs 对应到相应的视图函数。URL模式由一个字符串，一个视图，以及一个可选的能够在整个项目中使用的URL命名组成。Django 会遍历整个URL匹配模式，直到第一次匹配到请求的URL停止。然后Django 会执行这个URL对应的视图函数，这个过程中会传递HttpRequest实例，以及关键参数或者位置参数。

在blog应用中创建一个 urls.py 文件，添加如下代码：

```python
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    # post views
    path('', views.post_list, name='post_list'),
    path('<int:year>/<int:month>/<int:day>/<slug:post>/',
        views.post_detail,
        name='post_detail'),
]
```

上诉代码中，我们使用变量 `app_name` 定义了应用的命名空间，这可以方便我们通过应用名组织URLs，以及可以反向推导URLs的时候使用这个名字。上面使用path()函数定义了两个URL匹配，一个没有匹配参数，另一个匹配四个参数并传递给 post_detail 视图。

注意看第二个匹配中，我们使用<>来捕获URL中的参数；如果直接用类似<parameter>这样的模式匹配参数，捕获的数值都会是字符串类型。为了获取特定类型的参数，我们需要使用类似<int:year>这种方式，它会捕获整数类型。而<slug:post> 会捕获到一个 slug 。具体其他能够使用的类型，我们可以查看官方文档： https://docs.djangoproject.com/en/3.0/topics/http/urls/#path-converters

如果path()不能满足你的需求，Django 还提供了 re_path()，它可以使用Python正则表达式来定义复杂的URL匹配规则。

具体可以查看官方文档：https://docs.djangoproject.com/en/3.0/ref/urls/#django.urls.re_path

如果你对正则表达式也不是很了解的话，也可以先看看官方的HOWTO 文档： https://docs.python.org/3/howto/regex.html

blog应用的URl工作完成之后，我们接下来需要将它添加到项目的主URL中。

编辑 mysite 目录下的 urls.py 文件

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('blog/', include('blog.urls', namespace='blog')),
]
```

这里使用了 include 函数来包含blog应用中的urls，并将它放在了 'blog/'路径下。同时我们使用了命名空间 blog，这个命名空间的名称需要在整个项目中都是独一无二的。它的使用很简单，比如需要获取blog应用定义的url时，可以使用 blog:post_list， blog:post_detail。更多有关URL 命名空间详细内容可以在官方文档：https://docs.djangoproject.com/en/3.0/topics/http/urls/#url-namespaces

### 在model中使用绝对URl

我们可以通过在模型中添加 get_absolute_url() 方法，来为每一个特定对象生成匹配它的URL.这里我们将用到官方提供的reverse() 方法，这个方法通过视图函数的名称和传递的参数来生成相应的URls。官方文档在：https://docs.djangoproject.com/en/3.0/ref/urlresolvers/

修改 blog 应用的 models.py 文件

```python
from django.urls import reverse

def get_absolute_url(self):
    return reverse('blog:post_detail',
                args=[self.publish.year,
                      self.publish.month,
                      self.publish.day, self.slug])
```

这个函数会在模板中使用到。

## 创建视图模板

通过前面的学习我们创建了视图以及它对应的URL，视图会根据逻辑计算出返回给用户的数据，而模板的作用就是接收这些数据，并定义如何展示这些数据。它们通过Django 的模板语言来与HTML结合。你可以在官方文档中了解详细的模板语言定义：https://docs.djangoproject.com/en/3.0/ref/templates/language/

在blog应用项目中创建如下目录结构：

```shell
templates
└── blog
    ├── base.html
    └── post
        ├── detail.html
        └── list.html
```

base.html 包含了网站的主要的HTML结构，并且将内容页分成了主内容区以及侧边栏。list.html 和 detail.html 从 base.html 中继承内容，分别渲染文章的列表视图以及详细页面视图。

Django 内置强大的模板语言，能够让数据根据不同的需求来显示。它基于三中模板形式：

- 模板标签(Template tags) 用来控制渲染，它的形式是 {% tag %}
- 模板变量(Template variables) 用来占位并使用变量在渲染时候替换内容， 它的形式是 {{ variable }}
- 模板过滤(Template filters) 用来修改变量的显示格式，它的形式是 {{ variable | filter }}

更多内容在官方文档： https://docs.djangoproject.com/en/3.0/ref/templates/builtins/

编辑 base.html 文件：

```html
{% load static %}
<!DOCTYPE html>
<html>
<head>
  <title>{% block title %}{% endblock %}</title>
  <link href="{% static "css/blog.css" %}" rel="stylesheet">
</head>
<body>
  <div id="content">
    {% block content %}
    {% endblock %}
  </div>
  <div id="sidebar">
    <h2>My blog</h2>
    <p>This is my blog.</p>
  </div>
</body>
</html>
```

其中 {% load static %} 告诉Django可以使用由 `django.contrib.staticfiles` 提供的 static 模板标签，这样就可以使用 {% static %} 标签了。这个标签用来加载位于blog应用 static 目录下的静态文件，比如说这里加载的 blog.css。

这个静态目录在blog目录，结构如下:

```shell
static
└── css
    └── blog.css
```

blog.css 内容;

```css
body { 
    margin:0;
    padding:0;
    font-family:helvetica, sans-serif; 
}

a { 
    color:#00abff;
    text-decoration:none; 
}

h1 { 
    font-weight:normal;
    border-bottom:1px solid #bbb;
    padding:0 0 10px 0;
}

h2 {
    font-weight:normal;
    margin:30px 0 0;
}

#content { 
    float:left;
    width:60%;
    padding:0 0 0 30px; 
}

#sidebar { 
    float:right;
    width:30%;
    padding:10px;
    background:#efefef; 
    height:100%;
}

p.date { 
    color:#ccc;
    font-family: georgia, serif;
    font-size: 12px;
    font-style: italic; 
}

/* pagination */
.pagination { 
    margin:40px 0; 
    font-weight:bold;
}

/* forms */
label { 
    float:left;
    clear:both;
    color:#333;
    margin-bottom:4px; 
}
input, textarea { 
    clear:both;
    float:left;
    margin:0 0 10px;
    background:#ededed;
    border:0;
    padding:6px 10px;
    font-size:12px;
}
input[type=submit] {
    font-weight:bold;
    background:#00abff;
    color:#fff;
    padding:10px 20px;
    font-size:14px;
    text-transform:uppercase; 
}
.errorlist { 
    color:#cc0033;
    float:left;
    clear:both;
    padding-left:10px; 
}

/* comments */
.comment {
    padding:10px;
}
.comment:nth-child(even) {
    background:#efefef;
}
.comment .info {
    font-weight:bold;
    font-size:12px;
    color:#666;
}
```

然后编辑 post/list.html 文件

```python
{% extends "blog/base.html" %}

{% block title %}My Blog{% endblock %}

{% block content %}
  <h1>My Blog</h1>
  {% for post in posts %}
    <h2>
      <a href="{{ post.get_absolute_url }}">
        {{ post.title }}
      </a>
    </h2>
    <p class="date">
      Published {{ post.publish }} by {{ post.author }}
    </p>
    {{ post.body|truncatewords:3|linebreaks }}
  {% endfor %}
{% endblock %}
```

其中 {% extends %} 标签告诉Django该文件继承 blog/base.html 模板，然后我们用特定的代码去填充base 模板中的 `title` 以及 `content` 块。

post 主体中，我们使用了两个 template filters:

- truncatewords  将字段截取出特定字数，这里是30个
- linebreaks 将输出转化为HTML换行

这里的 template filters 可以根据需求想使用多少就用多少，每一个filter 都会处理前一个的输出。

编辑post/detail.html 文件

```html
  
{% extends "blog/base.html" %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
  <h1>{{ post.title }}</h1>
  <p class="date">
    Published {{ post.publish }} by {{ post.author }}
  </p>
  {{ post.body|linebreaks }}
{% endblock %}
```

最后我们通过 python manage.py runserver 来启动服务，并验证整个工作是否符合预期。

注意请手动添加足够多的状态为 published 的文章。

## 添加分页(pagination)

当我们有了足够多的文章之后，一个页面全部显示显得不够友好。这时候我们需要通过分页将数据分散到多个页面上。

Django 内置了一个pagination 类来帮助我们简单的处理分页需求。

编辑 blog应用中的 views.py 文件，导入 paginator 相关类

```python
from django.core.paginator import Paginator, EmptyPage,\
                                  PageNotAnInteger

def post_list(request):
    object_list = Post.published.all()
    paginator = Paginator(object_list, 3) # 3 posts in each page
    page = request.GET.get('page')
    try:
        posts = paginator.page(page)
    except PageNotAnInteger:
        # If page is not an integer deliver the first page
        posts = paginator.page(1)
    except EmptyPage:
        # If page is out of range deliver last page of results
        posts = paginator.page(paginator.num_pages)
    return render(request,
                 'blog/post/list.html',
                 {'page': page,
                  'posts': posts})
```

在 templates 目录下创建分页相关的模板，pagination.html

```html
<div class="pagination">
  <span class="step-links">
    {% if page.has_previous %}
      <a href="?page={{ page.previous_page_number }}">Previous</a>
    {% endif %}
    <span class="current">
      Page {{ page.number }} of {{ page.paginator.num_pages }}.
    </span>
    {% if page.has_next %}
      <a href="?page={{ page.next_page_number }}">Next</a>
    {% endif %}
  </span>
</div>
```

这个模板需要一个Page对象来渲染前一页，后一页，当前页等相关数据。然后在 blog/post/list.html 中添加对 pagination.html 的支持

```html
{% block content %}
  ...
  {% include "pagination.html" with page=posts %}
{% endblock %}
```



## Class-based 视图

Class-based views 将视图通过Python 对象来实现，这是除了Python方法实现的另一种途径。因为一个视图就是可执行的功能，它接收request 返回 response ，所以可以将视图定义为类方法。

Class-based views 相对于函数视图的优点是：

- 可以根据HTTP 的方法（GET, POST, PUT）来组织相应的代码，每一个动作对应一个实现方法，这样就不用通过条件分支来判断了。
- 可以通过多继承来创建视图类（比如 mixins 类）

更多相关介绍在官方文档：https://docs.djangoproject.com/en/3.0/topics/class-based-views/intro/

接下来我们通过修改 post_list 视图来看一下 class-based 类的使用。

编辑blog 应用中的 view.py 文件

```python
from django.views.generic import ListView

class PostListView(ListView):
    queryset = Post.published.all()
    context_object_name = 'posts'
    paginate_by = 3
    template_name = 'blog/post/list.html'
```

上面的代码主要做了以下事情：

- 使用了特定的QuerySet而不是获取所有的数据对象。除了自定义一个 queryset 参数，我们也可以使用 model = Post 这样指定了数据源之后，Django 会使用Post.objects.all() 来获取QuerySet
- 查询结果被存储到上下文变量 posts 中。如果我们不定义 context_object_name， 那么默认变量为 object_list
- Paginate 结果设置为每页显示3个
- 使用自定义的模板来渲染页面，如果不设置，ListView类会使用默认的 blog/post_list.html

接下来修改 blog 应用 urls.py 文件：

```python
urlpatterns = [
    # post views
    # path('', views.post_list, name='post_list'),
    path('', views.PostListView.as_view(), name='post_list'),
    path('<int:year>/<int:month>/<int:day>/<slug:post>/',
        views.post_detail,
        name='post_detail'),
]
```

将原来的注释掉，换成新的。同时为了pagination 能够正常工作，我们需要修改post/list.html 中相关的代码

```html
  {% include "pagination.html" with page=page_obj %}
```

这是因为 Django的 ListView 通用视图将选定页面放在了变量 page_obj 中。

重新启动服务，查看更换了之后程序是否还能正常运行。

## 总结

通过上面的介绍，我们使用 Django 框架创建了一个简单的博客应用。我们设计数据模型并学会了数据库的迁移，同时我们也学会了创建博客相关的 视图，模板，以及路由，当然还有分页系统的简单使用。