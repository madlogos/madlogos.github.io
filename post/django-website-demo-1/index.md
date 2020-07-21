
{{% admonition abstract 摘要 %}}
还是edx的作业。今次要换用Django框架实现一个Pizza点单系统。<br/>
【honor code警告】如果你刚巧也注册了这门课，千万不要抄。
{{% /admonition %}}

{{% admonition warning 注意 %}}
如无法显示视频，可能被作为不安全脚本屏蔽。在浏览器地址栏里点击安全提示图标，允许运行不安全的脚本。
{{% /admonition %}}

[成品效果视频](https://v.youku.com/v_show/id_XNDQzNzY0NTEwNA==.html?spm=a2hzp.8244740.0.0) @ 优酷：

<iframe height=498 width='100%' src='https://player.youku.com/embed/XNDQzNzY0NTEwNA==' frameborder=0 'allowfullscreen'></iframe>

这是哈佛**继续教育学院**开的的[用Python和Javascript撸网络编程](https://courses.edx.org/courses/course-v1:HarvardX+CS50W+Web/course/) 第四个作业项目。

## [作业要求](https://docs.cs50.net/web/2019/x/projects/3/project3.html)

做一个仿[Pinocchio Pizza](http://www.pinocchiospizza.net/menu.html)的Pizza预订系统。

{{% admonition bug "可以看到" %}}
很明显，这个网站做得很渣。但是据说在哈佛所在的坎布里奇特别受欢迎，以特色潜艇堡（subs）著称。<b>技术还是不如业务重要。</b>
{{% /admonition %}}


要实现以下功能：

1. 分析样品菜单，构建模型
2. 用Django admin或者写Python命令，添加菜单内容
3. 用户注册、登录、登出
4. 虚拟购物车
5. 下订单
6. 浏览订单和订单明细
7. 延伸功能：比如系统管理员在后台更新订单状态、用[Strip API](https://stripe.com/docs) 完成结算等

## 准备

- 先要有Python（装了Anaconda）
- 要装`Django`包(`pip`)，这里用的是Django 2.2。

<!-- {% raw %} -->

{{% admonition tip "提醒" %}}
开发要锁定工具链版本，否则后患无穷。virtualenv或者Docker都可以。
{{% /admonition %}}

<!-- {% endraw %} -->

<!--more-->

### 项目结构

{{% admonition info "源代码托管于Github" %}}
<a href="https://github.com/madlogos/edx_cs50/tree/master/project3">戳这里看源码</a>
{{% /admonition %}}

```
project3
|-- application.py
|-- db.sqlite3
|-- django.log
|-- manage.py
|
|--+ accounts
|  |--+ migrations
|  |-- __init__.py
|  |-- admin.py
|  |-- apps.py
|  |-- forms.py
|  |-- models.py
|  |-- tests.py
|  |-- urls.py
|  `-- views.py
|
|--+ orders
|  |--+ migrations
|  |-- __init__.py
|  |-- admin.py
|  |-- apps.py
|  |-- forms.py
|  |-- models.py
|  |-- tests.py
|  |-- udf.py
|  |-- urls.py
|  `-- views.py
|
|--+ pizza
|  |-- __init__.py
|  |-- settings.py
|  |-- urls.py
|  `-- wsgi.py
|
|--+ static
|  |--+ css
|  |  `-- style.css
|  `--+ js
|     |-- cart.js
|     |-- main.js
|     |-- menu.js
|     |-- order.js
|     |-- orders.js
|     `-- pick_product.js
|
`--+ templates
   |-- _base.html
   |-- _popup.html
   |
   |--+ accounts
   |  |-- login.html
   |  `-- register.html
   |
   `--+ orders
	  |-- cart.html
	  |-- index.html
	  |-- order.html
	  |-- orders.html
	  `-- pick_product.html
```

<!-- more -->

Django框架比Flask要复杂得多。整个应用就是一个工程(project)，而子应用(application)模块则相当于内含的一个个包(package)：

- 通过`django-admin startproject pizza`命令，生成一个骨架，包括pizza文件夹及内含的3个 .py文件，以及django命令行工具manage.py。
- 进入pizza根目录，运行`python manage.py startapp accounts`和`python manage.py startapp orders`，分别生成accounts和orders两个具体应用。两个文件夹都包含\_\_init\_\_.py，这就标志着它们是包。此外，都包括admin.py（Django管理后台配置）、apps.py（应用打包设置）两个设置脚本，以及实现MVC设计的models.py（模型）、views.py（视图）和urls.py（控制）。
	- accounts用来管理账户信息、登录和注册等
	- orders用来管理菜单、订单和购物车等

除了上面这些后台脚本之外，再建两个必要的资源文件夹：

- 静态文件所在的static，例行包括css和js两个文件夹。
- .html模板文件所在的templates。为了便于管理，框架模板放在根目录，accounts和orders两个应用分别开一个文件夹。

### 配置

#### 全局配置

首先，要设置一下超级管理员，控制台运行`python manage.py createsuperuser`，设置用户名、密码和邮件。这样，后续就可以用这个账户登到Django自带的管理后台，在图形界面上管理数据。

##### settings.py

pizza/settings.py里已经预置了很多配置项。要做一些调整：

- INSTALLED_APPS列表增加两项: 'accounts.apps.AccountsConfig', 'orders.apps.OrdersConfig'
- 增加LOGGING

	```python
	LOGGING = {
		'version': 1,
		'disable_existing_loggers': False,
		'formatters': {
			'verbose': {
				'format': '{asctime} {module}.{funcName} {lineno:3} {levelname:7} => {message}',
				'style': '{',
			},
		},
		'handlers': {
			'console': {
				'class': 'logging.StreamHandler',
				'formatter': 'verbose',
			},
			'file': {
				'class': 'logging.handlers.RotatingFileHandler',
				'formatter': 'verbose',
				'filename': 'django.log',
				'maxBytes': 4194304,  # 4 MB
				'backupCount': 10,
				'level': 'DEBUG',
			},
		},
		'loggers': {
			'': {
				'handlers': ['console', 'file'],
				'level': os.getenv('DJANGO_LOG_LEVEL', 'INFO'),
			},
			'django': {
				'handlers': ['console', 'file'],
				'level': os.getenv('DJANGO_LOG_LEVEL', 'INFO'),
				'propagate': False,
			},
		},
	}
	```
	
- TIME_ZONE 改成自己所在的时区，比如'Asia/Shanghai'
- STATIC_URL 改为 '/static/'
- STATICFILES_DIRS 改为 [os.path.join(BASE_DIR, "static"), '/static/']，在这个应用中，生效的是前者

##### urls.py

pizza/urls.py要更新一下urlpatterns：

```python
urlpatterns = [
    path("", include("accounts.urls")),
    path("", include("orders.urls")),
    path("admin/", admin.site.urls),
]
```

这样，accounts和orders两个子应用中的路由，都被安排到整个应用的根路由上。即：accounts里的'/'，就等价于整个应用的主页。这当然有隐患，好在应用架构不复杂。推荐的做法是把其中一个子应用映射到主路由，其他应用都丢进下一级。

admin.site.urls要映射进去，这样，后面才能通过"<domain name>/admin"去访问Django后台。

#### 分应用配置

- accounts/apps.py定义应用名称

	```python
	class AccountsConfig(AppConfig):
		name = 'accounts'
	```

- orders/apps.py定义应用名称

	```python
	class OrdersConfig(AppConfig):
		name = 'orders'
	```

这样，pizza/settings.py的INSTALLED_APPS才能识别accounts和orders这两个应用。将来，这两个包也可以剥离出去给其他项目复用。

### 基础模板

[_base.html](https://github.com/madlogos/edx_cs50/blob/master/project3/templates/_base.html)和[_popup.html](https://github.com/madlogos/edx_cs50/blob/master/project3/templates/_popup.html)是框架模板，后续其他页面模板都会套用它。后者是前者的简化版。

<!-- {% raw %} -->
- 要记得{% load static %}，载入静态文件。这样定义好之后，Django就知道上哪里动态地找到`href="{% static 'css/style.css' %}"`了。
- "elem_cont"部分添加了通用的message代码。后端传到前端的message对象必须是一个长度为2的列表，其中message.0是"info"、"warning"、"success"、"danger"这几个Bootstrap认识的类别，message.1则是信息框的具体内容。事实上Django有自己的信息组件，这里没有用到。
<!-- {% endraw %} -->

## 账户管理(accounts)应用

配置部分结束，开始做功能。

首先进到[accounts](https://github.com/madlogos/edx_cs50/blob/master/project3/accounts)目录，构建账户管理模块。

### 模型

如果要自己设计一套User体系，可以在[models.py](https://github.com/madlogos/edx_cs50/blob/master/project3/accounts/models.py)里定义。由于这个作业里对用户信息的要求已经被Django自带的User类涵盖，所以直接导进来就可以用。

```python
# accounts/models.py
from django.contrib.auth.models import User
```

在[admin.py](https://github.com/madlogos/edx_cs50/blob/master/project3/accounts/admin.py)里，导入下面几个包：

```python
# accounts/admin.py
from django.contrib.auth.admin import UserAdmin
from django.contrib.auth.models import User
from django.db import models
```

控制台运行`python manage.py runserver`，启动Django开发服务器，浏览器访问127.0.0.1:8000/admin。

{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2019/1106/admin_entry.png" title="图 | admin登录页" %}}

用前面创建的超级管理员账号登录，即可看到Site administration界面，Groups和Users表已经可以直接访问、维护了。

{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2019/1106/admin_ui.png" title="图 | admin管理界面" %}}

当然，我们并不希望通过后台来添加用户，还是由用户自己从前端注册。所以后面会进一步完善前端视图。

### 控制

Django通过urlpatterns来控制路由。在[urls.py](https://github.com/madlogos/edx_cs50/blob/master/project3/accounts/urls.py)中修改：

```python
# accounts/urls.py
from . import views

urlpatterns = [
    path("", views.index, name="index"),
    path("login", views.login_view, name="login"),
    path("signup", views.signup, name="signup"),
    path("logout", views.logout_view, name="logout")
]
```

这样，就把四个路由绑定到了views.py中对应的函数上，并且都给了别名（可以用`reverse()`函数快速解析）。

### 视图

有了模型，定义好路由绑定，接下来就在视图[views.py](https://github.com/madlogos/edx_cs50/blob/master/project3/accounts/views.py)中写具体功能。

#### 导入一堆包

```python
# accounts/views.py
from django.contrib.auth import authenticate, login, logout
from django.contrib.auth.models import User
from django.http import HttpResponse, HttpResponseRedirect
from django.shortcuts import render, redirect
from django.urls import reverse
from .forms import RegisterForm, LoginForm
```

这里，直接使用了Django.contrib.auth模块里的authenticate, login, logout功能，导入了User类。此外，专门在[forms.py](https://github.com/madlogos/edx_cs50/blob/master/project3/accounts/forms.py)里编了两套表单模板，也一并导入。

#### 首页

```python
def index(request):
    if not request.user.is_authenticated:
        return login_view(request)
    return HttpResponseRedirect(reverse("menu"), content={"user": request.user})
```

如果request中的user实例并没有通过认证，就返回`login_view()`，也就是显示登录页。否则，就跳转去别名为"menu"的页面，也就是orders模块的的首页。

{{% admonition tip "要点" %}}
Django的视图函数，必须返回一个Http响应，要么是HttpResponse，要么Http404之类。否则就会报内部错误。
{{% /admonition %}}

#### 登录

##### 后端

views.py里定义`login_view()`函数。

```python
# accounts/views.py
def login_view(request):
    if request.user.is_authenticated:
        return HttpResponseRedirect(reverse("menu"), content={"user": request.user})
    try:
        if request.method == "POST":
            login_form = LoginForm(request.POST)
            if login_form.is_valid():
                username = login_form.cleaned_data.get("username")
                password = login_form.cleaned_data.get("password")
            else:
                return render(request, "accounts/login.html", 
                    {"message": ["danger", str(login_form.errors.values())], 
                     "form": LoginForm()})
            user = authenticate(request, username=username, password=password)
            if user and user.is_active:
                login(request, user)
                return HttpResponseRedirect(reverse("index"), 
                    content={"user": request.user})
            else:
                return render(request, "accounts/login.html", 
                    {"message": ["danger", "Invalid credentials."], 
                     "form": login_form})
        else:
            login_form = LoginForm()
            return render(request, "accounts/login.html", {"message": None, 
                "form": login_form})
    except Exception as e:
        return render(request, "accounts/login.html", {"message": ["danger", str(e)]})
```

- 如果user已经认证，就跳去orders首页
- 如果没认证，那么
	- 假如是POST方法（提交登录验证表单），就从login_form里提信息出来验证。通过验证就`login()`，否则跳转回去。
	- 假如是其他方法，那就渲染登录界面

在[forms.py](https://github.com/madlogos/edx_cs50/blob/master/project3/accounts/forms.py)里定义了登录表单模板。

```python
# accounts/forms.py
class LoginForm(forms.Form):
    username = forms.CharField(
        label="Username", max_length=128, required=True,
        widget=forms.TextInput(attrs={'class': 'form-control'}))
    password = forms.CharField(
        label="Password", max_length=256, required=True,
        widget=forms.PasswordInput(attrs={'class': 'form-control'}))
    
    def clean_username(self):
        username = self.cleaned_data.get('username')

        filter_result = User.objects.filter(username__exact=username)
        if not filter_result:
            raise forms.ValidationError(
                "This username does not exist. Please register first.")
        return username
```

LoginForm类只定义了username和password两个文本型字段。Django会自动理解这些参数，渲染出对应的表单字段。在这个类里，还额外写了个`clean_username()`方法，验证用户名是否存在。这样，就不需要在views.py里单独写校验代码了，直接绑定在表单模板里，更便于维护和复用。很方便。

##### 前端

对应的[login.html](https://github.com/madlogos/edx_cs50/blob/master/project3/templates/accounts/login.html)页面模板写成这样：

<!-- {% raw %} -->
```html
{% extends "_base.html" %}

{% block title %}
Sign In
{% endblock %}

{% block control %}
<form class="form-signin" action="{% url 'login' %}" method="post">
    {% csrf_token %}
    <h2 class="form-signin-heading">Sign In</h2>
    <div class="form-group">
        <div class="fieldWrapper">
            {{ form.username.errors }}
            {{ form.username.label_tag }}
            {{ form.username }}
        </div>
        <div class="fieldWrapper">
            {{ form.password.errors }}
            {{ form.password.label_tag }}
            {{ form.password }}
        </div>
    </div>
    <label for="signIn" class="sr-only">Click</label>
    <button id="signIn" class="btn btn-lg btn-primary btn-block" >
        Sign In
    </button>        
</form>
<form class="form-signin" action="{% url 'signup' %}">
    <button id="signUp" class="btn btn-lg btn-default btn-block">
        Sign up now!
    </button>
</form>
{% endblock %}
```
<!-- {% endraw %} -->

<!-- {% raw %} -->

{{% admonition tip "要点" %}}
Django表单内都必须加个'{% csrf_token %}' 解决跨域问题。模板内部解析form对象，组装出表单。
{{% /admonition %}}

<!-- {% endraw %} -->

后端传到前端的form对象，其实就是login_form。通过这套语法，分离了校验逻辑和样式，前端表单写起来更简明。

{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2019/1106/sign_in.png" title="图 | 用户登录界面" %}}


#### 注册

##### 后端

views.py里定义 `signup()` 函数。

```python
# accounts/views.py
def signup(request):
    if request.user.is_authenticated:
        return HttpResponseRedirect(reverse("menu"), content={"user": request.user})
    try:
        if request.method == "POST":
            reg_form = RegisterForm(request.POST)
            if reg_form.is_valid():
                username = reg_form.cleaned_data.get("username")
                password = reg_form.cleaned_data.get("password")
                first_name = reg_form.cleaned_data.get("first_name")
                last_name = reg_form.cleaned_data.get("last_name")
                email = reg_form.cleaned_data.get("email")
            else:
                return render(request, "accounts/register.html", 
                    {"message": ["danger", str(reg_form.errors.values())], 
                     "form": RegisterForm()})
            
            user = User.objects.create_user(
                username=username, password=password, first_name=first_name,
                last_name=last_name, email=email)
            user.save()
            user.is_active = True
            user.success = True

            return render(request, "accounts/login.html", 
                {"message": ["success", """New account %s has been created. 
                    Log in now.""" % (username)], "form": LoginForm()})
        else:
            return render(request, "accounts/register.html", 
                {"message": None, "form": RegisterForm()})
    except Exception as e:
        return render(request, "accounts/register.html", 
            {"message": ["danger", str(e)], "form": RegisterForm()})
```

原理跟登陆差不多。主要区别在于出现了ORM操作。当通过校验后，Django就把reg_form表单字段拿过去，创建一个新的User对象。ORM操作语句很直观，`<类名>.objects.<操作方法>(<参数列表>)`。

reg_form表单模板也定义在[forms.py](https://github.com/madlogos/edx_cs50/blob/master/project3/accounts/forms.py)里。

```python
# accounts/forms.py
from django import forms
from django.contrib.auth.models import User

class RegisterForm(forms.Form):
    username = forms.CharField(
        label="Username", max_length=128, required=True,
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    password = forms.CharField(
        label="Password", max_length=256, required=True,
        widget=forms.PasswordInput(attrs={'class': 'form-control'})
    )
    password_cfm = forms.CharField(
        label="Confirm Password", max_length=256, required=True,
        widget=forms.PasswordInput(attrs={'class': 'form-control'})
    )
    first_name = forms.CharField(
        label="First Name", max_length=30, 
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    last_name = forms.CharField(
        label="Last Name", max_length=150, 
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    email = forms.CharField(
        label="Email Address", max_length=128, 
        widget=forms.EmailInput(attrs={'class': 'form-control'})
    )

    def clean_username(self):
        username = self.cleaned_data.get('username')

        filter_result = User.objects.filter(username__exact=username)
        if len(filter_result) > 0:
            raise forms.ValidationError("Your username already exists.")
        return username

    def clean_password_cfm(self):
        pwd1 = self.cleaned_data.get("password")
        pwd2 = self.cleaned_data.get("password_cfm")
        if pwd1 and pwd2 and pwd1 != pwd2:
            raise forms.ValidationError("Password mismatch. Please enter again.")
        return pwd2
```

##### 前端

对应的[register.html](https://github.com/madlogos/edx_cs50/blob/master/project3/templates/accounts/register.html)页面模板写成这样：

<!-- {% raw %} -->
```html
<!-- templates/accounts/register.html -->
{% extends "_base.html" %}

{% block title %}
Sign Up
{% endblock %}

{% block control %}
<form class="form-signin" action="{% url 'signup' %}" method="post">
    {% csrf_token %}
    <h2 class="form-signin-heading">Sign Up</h2>
    <div class="form-group">
        <div class="fieldWrapper">
            {{ form.username.errors }}
            {{ form.username.label_tag }}
            {{ form.username }}
        </div>
        <div class="fieldWrapper">
            {{ form.password.errors }}
            {{ form.password.label_tag }}
            {{ form.password }}
        </div>
        <div class="fieldWrapper">
            {{ form.password_cfm.errors }}
            {{ form.password_cfm.label_tag }}
            {{ form.password_cfm }}
        </div>
        <div class="fieldWrapper">
            {{ form.first_name.errors }}
            {{ form.first_name.label_tag }}
            {{ form.first_name }}
        </div>
        <div class="fieldWrapper">
            {{ form.last_name.errors }}
            {{ form.last_name.label_tag }}
            {{ form.last_name }}
        </div>
        <div class="fieldWrapper">
            {{ form.email.errors }}
            {{ form.email.label_tag }}
            {{ form.email }}
        </div>
    </div>
    <button id="signUp" class="btn btn-lg btn-primary btn-block">Sign Up</button>
</form>
<form class="form-signin" action="{% url 'login' %}">
    <button id="signIn" class="btn btn-lg btn-default btn-block">Sign In</button>
</form>
{% endblock %}
```
<!-- {% endraw %} -->

同样，直接把RegisterForm对象传到前端，很容易就能写出数据驱动的页面来。

{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2019/1106/sign_up.png" title="图 | 用户注册界面" %}}

#### 注销

注销操作比Flask更好些，直接用内置的 `logout()` 方法。

```python
# accounts.views.py
def logout_view(request):
    logout(request)
    return HttpResponseRedirect(reverse("login"), content={"message": 
        ["success", "Logged out."]})
```

退出后直接转跳登录页，所以也不必费劲专门写网页模板了。这里用了`HttpResponseRedirect()`而不是`redirect()`，因为除了转跳以外，还要传一个content回去，用来渲染一个告警。

到此，整个账号管理的功能就写好了。实际使用，还有必要加功能，比如反机器人、密码找回等。

## 订单管理(orders)

接下来，进[orders](https://github.com/madlogos/edx_cs50/blob/master/project3/orders)目录，构建购物车和订单管理模块。这块内容比账号管理复杂一些。

### 模型

从定义ORM模型开始。在[models.py](https://github.com/madlogos/edx_cs50/blob/master/project3/orders/models.py)：


#### 选择项元组

先定义选择项，结构是key-value元组。后续控件限定合法值，直接绑上去就行。

```python
# orders/models.py
from django.db import models
from django.contrib.auth.models import User

# Create your models here.
SIZE_CHOICES = (
    ('Small', 'Small'),
    ('Large', 'Large'),
    ('Regular', 'Regular')
)
ORDER_STATUS = (
    ('Pending', 'Pending'),
    ('Paid', 'Paid'),
    ('Completed', 'Completed'),
    ('Failed', 'Failed'),
    ('Cancelled', 'Cancelled'),
)
```

#### 模型和元参数

model的定义跟表单有点像。以品类(Category)和产品(Product)为例，都继承自models.Model类。定义好字段参数后，可以设定元数据class Meta，定义verbose_name之类元参数。对于Product，我希望实现category、name和size三个字段构成一个复合主键，只要定义进unique_together就可以了。

另外，比较推荐单独定义`__str__()`方法，这样在Django后台管理数据时，屏显记录名更人性化。

Category和Product是通过Category id连接的，所以Product里要设置一个外键字段category，设置related_name="products"。这样将来就可以通过Category.objects.filter(products="xxx")来反查xxx产品的类型。

```python
# orders/models.py
class Category(models.Model):
    name = models.CharField(max_length=128, unique=True)
    
    class Meta:
        verbose_name = "category"
        verbose_name_plural = "categories"
        db_table = "shop_category"

    def __str__(self):
        return self.name


class Product(models.Model):
    category = models.ForeignKey(Category, on_delete=models.PROTECT, related_name="products")
    name = models.CharField(max_length=128)
    size = models.CharField(max_length=8, choices=SIZE_CHOICES, default='Regular')
    n_topping = models.IntegerField(default=0)
    n_addition = models.IntegerField(default=0)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = "product"
        verbose_name_plural = "products"
        db_table = "shop_product"
        unique_together = (('category', 'name', 'size'), )

    def __str__(self):
        return f"{self.category} - {self.name} ({self.size})"
```

#### 多对多关系

要特别提一下的是`ManyToManyField`，比如作为订单组件的Item，可以绑一个或多个Topping或Addition。传统做法是专门建一张Item_Topping_Mapping表，将ItemTopping的ID关联起来，实现多对多关系。Django的做法是：

```python
# orders/models.py
class Item(models.Model):
    product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name="products")
    quantity = models.IntegerField(default=0)
    topping = models.ManyToManyField(Topping, related_name="toppings", through="ItemTopping")
    addition = models.ManyToManyField(Addition, related_name="additions", through="ItemAddition")
    price = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = "item"
        verbose_name_plural = "items"

    def __str__(self):
        return f"{self.product} x {self.quantity}"

class ItemTopping(models.Model):
    item = models.ForeignKey(Item, on_delete=models.CASCADE, related_name="itemtopping_item")
    topping = models.ForeignKey(Topping, on_delete=models.CASCADE, related_name="itemtopping_topping")
    quantity = models.IntegerField(default=0)

    class Meta:
        verbose_name = "item_topping"
        verbose_name_plural = "item_toppings"

    def __str__(self):
        return f"{self.item} - {self.topping} x {self.quantity}"
```

Item的topping字段是个ManyToManyField，外键连接到Topping，而through参数则指定了多对多映射表"ItemTopping"。在ItemTopping里，除了item和topping外，又额外扩展了一个字段quantity。如果不需要扩展字段，ItemTopping甚至不用写。Django会自动生成这张表（但名字不一定是这个）。

#### migrate

写完所有的model后，控制台运行命令：

- `python manage.py makemigrations`
- `python manage.py migrate`

Django会自动产生migrate脚本，将这些ORM模型翻译成对应的DDL，对后台数据库进行创建/删除/修改操作。如果用sqlite连接到后台去看，就会发现里面已经把表都创建好了。修改model后，再次migrate，Django会直接修改表结构来适配，而不用自己手动写ALTER。

各表的关系实际上如下图。Item成为各表关联的中枢，因为一个典型的item包含了product和附加品，如topping和addition。

{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2019/1106/db3.svg" title="图 | orders应用的表结构" %}}

{{% admonition note "设计缺陷" false %}}
这个设计不算完美，Cart也可以用客户端缓存来管理，不需要大费周章地放服务器上。不过存服务器也有跨设备同步的好处。作为天然支持键值对的数据库，Cart表完全也可以写成键值对表，下订单时再解析出来，那么设计上可以简单很多。
{{% /admonition %}}

[待续](https://madlogos.github.io/post/django-website-demo-2/)

---

<!-- {% raw %} -->
{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/QRcode.jpg" width="50%" title="扫码关注我的的我的公众号" alt="扫码关注" %}}
<!-- {% endraw %} -->