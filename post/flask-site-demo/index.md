
{{% admonition abstract 摘要 %}}
本文记录了一则慕课作业。如果你也注册了这门课，千万不要抄。
{{% /admonition %}}

## 其实这是edx慕课的作业

出于不可自拔的技能焦虑，跑到edx上撸课，结果撸到一门偏前端的纯码农培训课。说出来吓死你，**哈佛【继续教育学院】**（对，就是范玮琪读的那个哈佛）开的[用Python和Javascript撸网络编程](https://courses.edx.org/courses/course-v1:HarvardX+CS50W+Web/course/)。它的主要卖点是用了Flask框架，就是号称一个 .py + 一个 .html就能欢快地hello world的小快灵建站利器。我朴素地想，学会多快好省地做一个网站，将来再学一点小程序啥的，起码搞起数据科学工程产品来，或许能派上一丢丢的用场。再不济，也能在简历上自吹全栈工程师嘛。

事实很打脸。这个作业只是整个课程的五大作业里的一个，我拿出所有业余时间埋头苦干，做了足足两个礼拜。以这个效率去搬砖，你猜老板会用什么武功揍我？

<!--more-->

## 作业的要求

要做个图书查询网络应用，首先，自己想辙把压缩包里的[book.csv](https://cdn.cs50.net/web/2019/x/projects/1/project1.zip) 5000条图书信息导进数据库里

1. 数据库自己到[heroku](https://www.heroku.com)上注册创建，订阅一个乞丐版
2. 要用Postgresql

然后，要有以下功能

1. 能注册
2. 能登录
3. 能注销
4. 能根据ISBN、书名、作者查询书籍
5. 能点进具体一本书里
    1. 除了固有信息，还要利用Goodreads的API获取平均评分
    2. 能看到其他用户发的书评和评级
6. 能发书评和评级，但一个用户只能发一次
7. 能暴露API给人家用

## 准备

### 数据库

heroku已经被salesforce.com买了，以傻瓜式建站和贵著称。我也建了一个实例，但是要翻墙。反正就是个作业，哪家SQL不是SQL？所以我就本地弄了个sqlite数据库`db.db`。

建三张表，`mbr`，`book`和`review`。`review`表里`mbr_id`和`book_id`分别外键关联到`mbr`和`book`。为啥不用`user`？因为postgresql不同意我用这个表名，所以在sqlite里也这么干。

```sql
CREATE TABLE IF NOT EXISTS book (
    id INTEGER PRIMARY KEY,
    isbn TEXT NOT NULL,
    title TEXT NOT NULL,
    author TEXT NOT NULL,
    year INTEGER NOT NULL
);
CREATE TABLE IF NOT EXISTS mbr (
    id INTEGER PRIMARY KEY,
    username TEXT NOT NULL,
    pwd TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS review (
    id INTEGER PRIMARY KEY,
    rev_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%d %H:%M:%S', 'now')),
    mbr_id INTEGER NOT NULL,
    book_id INTEGER NOT NULL,
    rating INTEGER,
    review TEXT
);
```

再把`books.csv`里的数据导进去。我是坚定的pandas粉，所以直接把csv读进pandas再一口气灌进sqlite里。面对postgresql我也这么干。传统的方法是用csv包一行一行扫描，再写入数据库。我是向量运算的刀山火海里捶打出来的，轻易不用循环。

```python
import sqlite3
import pandas as pd
conn = sqlite3.connect('db.db')
cur = conn.cursor()
if cur.execute('select count(*) from book;').fetchone()[0] == 0:
    df = pd.read_csv(r'books.csv')
    df.to_sql(name='book', con=conn, if_exists='append', index=False)
cur.close()
conn.close()
```

### 其它

当然，python那边得把python装好。我就直接用anaconda了。此外，需要把`Flask`，`Flask-Session`，`psycopg2-binary`和`SQKAlchemy`几个包都装上，`conda`/`pip`爱谁谁。 

[Goodreads](www.goodreads.com)是个书评网站（也被墙了）。需要自己上去注册账号，申请API开发密钥。


## 开工

### 项目结构

```
- application.py
- /static
    - /css
        - style.css
        - star-rating.min.css
    - /js
        - main.js
        - star-rating.min.js
- /templates
    - base.html
    - book.html
    - index.html
    - login.html
    - register.html
- db.db
```

这个应用比较简单，所以结构很扁平。

- application.py是后端，所有后台功能代码都写在上面。
- static文件夹放静态文件，css和js这种。叫assets也行。
- templates文件夹放各类html模板，这些模板都是用html+jinja2语法写的宏。

### 基础模板

base.html是框架模板。简单写一下。

<!-- {% raw %} -->
- 样式主要靠bootstrap
- body部分放了几个通用块，head, flash, disp, control, misc之类。用jinja2结构`{% block xxx %}{% endblock %}`来占位。
    - 块里面基本都没有进一步定义。只是给导航条加了点功能，如果当前线程有用户登着，就显示个注销按钮，否则就没有。
    - flash块比较特别，定义了一个比较通用的flash渲染宏，到时候只需要在后台.py里套用`flash`函数就能实现告警框。
    - 后续写其他模板时，引用base.html就行了。
<!-- {% endraw %} -->
      
<!-- {% raw %} -->
```html
<!DOCTYPE html>
<html lang='en'>
    <head>
        <meta CharacterSet='UTF-8'>
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>{% block title %}{% endblock %}</title>
        <link rel="stylesheet" href="https://cdn.staticfile.org/twitter-bootstrap/3.3.7/css/bootstrap.min.css">
        <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@3.3.7/dist/css/bootstrap-theme.min.css">
        <link rel="stylesheet" href="https://cdn.staticfile.org/font-awesome/4.7.0/css/font-awesome.css">
        <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
        <link rel="stylesheet" href="{{ url_for('static', filename='css/star-rating.min.css') }}">
        <script src="https://code.jquery.com/jquery-3.4.1.min.js"></script>
        <script src="https://cdn.staticfile.org/twitter-bootstrap/3.3.7/js/bootstrap.min.js"></script>
        <script src="{{ url_for('static', filename='js/main.js') }}"></script>
        <script src="{{ url_for('static', filename='js/star-rating.min.js') }}"></script>
    </head>
    <body>
        <!-- head -->
        <nav id="navbar" class="navbar navbar-default" role="navigation">
            <div class="container-fluid">
                <div class="navbar-header">
                    <a class="navbar-brand mb-0" href="#">Book Review</a>
                </div>
                {% if act_user is not none %}
                <form class="navbar-form navbar-right" action="{{ url_for('sign_off') }}" method="get">
                    <span>Welcome, {{ act_user['username'] }}.&nbsp;&nbsp;</span>
                    <button id="logout" class="btn btn-default btn-sm">Log out</button>
                </form>
                {% endif %}
            </div>
        </nav>
        <!-- flash -->
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                <div class="alert alert-{{ category }} alert-dismissable">
                    <button type="button" class="close" data-dismiss="alert">&times;</button>
                    {{ message }}
                </div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        <!-- body -->
        <div class="container">            
            {% block control %}
            {% endblock %}            
        </div>
        <div class="container">            
            {% block disp %}
            {% endblock %}            
        </div>
        <div class="container">            
            {% block misc %}
            {% endblock %}            
        </div>
        <!-- bottom -->
        <div class="container">
            <footer class="footer fixed-bottom">
                <hr>
                <p>&copy; 2019 madlogos</p>
            </footer>
        </div>
    </body>
</html>
```
<!-- {% endraw %} -->

对应地，在application.py里定义一些基本代码。"db.db"被export到环境变量`DATABASE_URL`，application.py被export到环境变量`FLASK_APP`，方便后面直接`flask run`。

```python
# -*- coding: UTF-8 -*-
import os
import requests
from flask import Flask, flash, jsonify, render_template, request, \
    redirect, session, url_for
from jinja2 import Markup
from flask_session import Session
from sqlalchemy import create_engine
from sqlalchemy.orm import scoped_session, sessionmaker

app = Flask(__name__, static_folder='static')


# Check for environment variable
if not os.getenv("DATABASE_URL"):
    raise RuntimeError("DATABASE_URL is not set")

# Configure session to use filesystem
app.config["SESSION_PERMANENT"] = False
app.config["SESSION_TYPE"] = "filesystem"
Session(app)

# Set up database
engine = create_engine(os.getenv("DATABASE_URL"))
db = scoped_session(sessionmaker(bind=engine))
sess = db()


@app.teardown_request
def remove_session(ex=None):
    db.remove()
```

### 登录

登录页login.html很简单，首先继承base.html的元素，然后在control块里放一个`form-signin`控件。套了一些bootstrap的元素，对前端技术不熟，丑得很。

<!-- {% raw %} -->
```html
{% extends "base.html" %}

{% block title %}
Sign In
{% endblock %}

{% block control %}
<form class="form-signin" action="{{ url_for('sign_in') }}" method="post">
    <h2 class="form-signin-heading">Please Sign In</h2>
    <label for="username" class="sr-only">Enter Your Username</label>
    <input type="text" name="username" class="form-control" placeholder="Username">
    <label for="password" class="sr-only">Enter Your Password</label>
    <input type="password" name="password" class="form-control" placeholder="Password">
    <label for="signIn" class="sr-only">Click</label>
    <button id="signIn" class="btn btn-lg btn-primary btn-block" >Sign In</button>        
</form>
<form class="form-signin" action="{{ url_for('sign_up') }}" method="get">
    <button id="signUp" class="btn btn-lg btn-default btn-block">Sign up now!</button>
</form>
{% endblock %}

{% block disp %}
&nbsp;
{% endblock %}

{% block misc %}
&nbsp;
{% endblock %}
```
<!-- {% endraw %} -->

后台部分写两个路由函数。主路由下，如果当前session没有用户登录，就转跳去登录页/login，否则直接进书籍列表页/index。如果进登录页，那么'GET'方法下跟主路由差不多逻辑，'POST'方法下（点按钮触发POST），就要校验用户名密码了。成功就进书籍列表，假如不对，就`flash`一个错误来。利用application.py里定义的`sign_in`函数和模板form中的`url_for`函数，就把后端功能绑定到前端了。

Flask是用SQLAlchemy的，SQLAlchemy是很高效的ORM工具，坊间一直认为它好过Django自带的ORM。不过这门课要求不用ORM，直接硬写SQL。


```python
@app.route("/", methods=['GET'])
def home():
    """Home page
    """
    if session.get('act_user') is None:
        return redirect(url_for("sign_in"))
    else:
        return render_template("index.html", act_user=session.get('act_user'))


@app.route("/login", methods=['GET', 'POST'])
def sign_in():
    """Sign in
    """
    if request.method == 'GET':
        if session.get('act_user') is None:
            return render_template(
                "login.html", act_user=session.get("act_user"))
        else:
            return render_template(
                "index.html", act_user=session.get("act_user"))
    elif request.method == 'POST':
        username = request.form.get('username')
        pwd = request.form.get('password')
        mbrinfo = sess.execute(
            """SELECT id, username FROM mbr WHERE username = :username
            AND pwd = :pwd;""",
            {"username": username, "pwd": pwd}).fetchone()
        if mbrinfo is None:
            flash(Markup(
                """<i class='fa fa-2x fa-exclamation-circle'></i>
                User not exist or wrong password."""), 
                'danger')
        else:
            session['act_user'] = {'id': mbrinfo[0], 'username': mbrinfo[1]}
        return home()
```

### 注销

有登陆就有注销。反正base.html里注销按钮已经绑定了logout路由，所以只要定义logout路由的后台绑定函数就行了。登出后，清空`session['act_use']`对象，回到登录页。

```python
@app.route('/logout', methods=['GET'])
def sign_off():
    session.pop('act_user', None)
    flash(Markup(
        """<i class='fa fa-2x fa-check-square-o'></i>
        You have logged out."""), 'success')
    return home()
```

效果如下。

<video width="760" height="440" controls>
  <source src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/190916/flask01-login.mp4" type="video/mp4">
</video>



### 注册

注册页和登录页差不多。

<!-- {% raw %} -->
```html
{% extends "base.html" %}

{% block title %}
Sign Up
{% endblock %}

{% block control %}
<form class="form-signin" action="{{ url_for('sign_up') }}" method="post">
    <h2 class="form-signin-heading">Register An Account</h2>
    <label for="username" class="sr-only">Enter Your Username</label>
    <input type="text" name="username" class="form-control" placeholder="Username" required>
    <label for="password" class="sr-only">Enter Your Password</label>
    <input type="password" name="password" class="form-control" placeholder="Password" required>
    <label for="repassword" class="sr-only">Confirm Your Password</label>
    <input type="password" name="repassword" class="form-control" placeholder="Re-input Password" required>
    <button id="signUp" class="btn btn-lg btn-primary btn-block">Sign Up</button>
</form>
<form class="form-signin" action="{{ url_for('sign_in') }}" method="get">
    <button id="signIn" class="btn btn-lg btn-default btn-block">Sign In Now</button>
</form>
{% endblock %}

{% block disp %}
&nbsp;
{% endblock %}

{% block misc %}
&nbsp;
{% endblock %}
```
<!-- {% endraw %} -->

后段部分分别对'GET'和'POST'两种方法做了定义。GET的话，渲染注册页模板而已。POST的话，就比较pwd和repwd的值是否相同，然而提交执行INSERT。继续转眺回登录页。

考究点的话当然还要有反机器人的措施。反正作业没这要求，就不贴金了。

```python
@app.route("/signup", methods=['GET', 'POST'])
def sign_up():
    """Sign up
    """
    if request.method == 'GET':
        return render_template("register.html", act_user=None)
    elif request.method == 'POST':
        username = request.form.get('username')
        pwd = request.form.get('password')
        repwd = request.form.get('repassword')
        mbrinfo = sess.execute(
            """SELECT id, username FROM mbr WHERE username = :username;""", 
            {"username": username}).fetchone()
        if mbrinfo is not None:
            flash(Markup(
                """<i class='fa fa-2x fa-check-square-o'></i>
                The username has been registered. Please change one."""), 
                'warning')
            return redirect(url_for('sign_up'))
        else:
            if pwd == repwd:
                sess.execute(
                    'INSERT INTO mbr (username, pwd) VALUES (:username, :pwd);',
                    {'username': username, 'pwd': pwd})
                sess.commit()
                flash(Markup(
                    """<i class='fa fa-2x fa-check-square-o'></i>
                    You have successfully created a new account. Now sign in."""), 
                    'success')
                return redirect(url_for("sign_in"))
            else:
                flash(Markup(
                    """<i class='fa fa-2x fa-exclamation-circle'></i>
                    You did not input the same password."""), 'danger')
                return redirect(url_for('sign_up'))
```


效果如下。

<video width="760" height="440" controls>
  <source src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/190916/flask02-signup.mp4" type="video/mp4">
</video>


### 检索书籍

登录进去后，进入真正的index页。通过三个文本框联合查询。在block disp部分，写一个jinja2宏循环，把books这个对象逐个解析出来填进表格里。如果什么条件都不给，那就会一口气查出5000条来。

此处应有分页，但是要去配插件。作为一个懒人，就不去多事了。5000条也崩不了，是吧。

<!-- {% raw %} -->
```html
{% extends "base.html" %}

{% block title %}
Books
{% endblock %}

{% block control %}
<form class="form-inline" action="{{ url_for('index') }}" method="post">
    <h2 class="form-group-heading">Search Books</h2>
    <div class="form-group">
        <label for="isbn" class="sr-only">ISBN</label>
        <input type="text" name="isbn" class="form-control" placeholder="ISBN">
        <label for="title" class="sr-only">Book title</label>
        <input type="text" name="title" class="form-control" placeholder="Book title">
        <label for="author" class="sr-only">Book author</label>
        <input type="text" name="author" class="form-control" placeholder="Book author">
        <label for="search" class="sr-only">Search</label>
        <button id="search" class="btn btn-lg btn-primary">Search</button>
    </div>       
</form>
{% endblock %}

{% block disp %}
<table id="tbl_books" class="table table-striped table-hover table-responsive" 
 cellspacing="0" width="80%">
<caption>Books at a glance</caption>
<thead>
    <tr>
        <th class="th-sm">ID</th>
        <th class="th-sm">ISBN</th>
        <th class="th-sm">Title</th>
        <th class="th-sm">Author</th>
        <th class="th-sm">Year</th>
    </tr>
</thead>
<tbody>
    {% if books|length > 0 %}
        {% for book in books%}
        <tr>
            <td><a href="{{ url_for('review', book_id=book[0]) }}">
                {{ book[0] }}</a></td>
            <td>{{ book[1] }}</td>
            <td>{{ book[2] }}</td>
            <td>{{ book[3] }}</td>
            <td>{{ book[4] }}</td>
        </tr>
        {% endfor %}
    {% endif %}
</tbody>
</table>
{% endblock %}

{% block misc %}
&nbsp;
{% endblock %}
```
<!-- {% endraw %} -->

后端主要是读取控件的值，然后执行查询，把books返回到网页。'GET'方法主要是直接访问网页，或者从主路由直接登入（未退出账号）。这时候干脆给books赋个空列表，减小点开销。

我不太会写查询条件的复合拼接，用了生成式。这是python里我最喜欢的语法糖。

```python
app.route('/index', methods=['GET', 'POST'])
def index():
    if request.method == 'GET':
        books = []
    elif request.method == 'POST':
        # run the query
        isbn = request.form.get('isbn')
        title = request.form.get('title')
        author = request.form.get('author')
        qry = ["%s LIKE '%%%s%%'" % (x, y) for x, y in
            (('isbn', isbn), ('title', title), ('author', author))
            if y is not None and y != '']
        if len(qry) == 0:
            books = sess.execute(
                'SELECT id, isbn, title, author, year FROM book;').fetchall()
        else:
            books = sess.execute(
                'SELECT id, isbn, title, author, year FROM book WHERE '
                + ' AND '.join(qry) + ';').fetchall()
        flash(Markup(
            """<i class='fa fa-2x fa-info-circle'></i>
            A total of %i books found.""" % len(books)), 'info')

    return render_template(
        'index.html', act_user=session.get('act_user'), books=books)
```

效果如下。

<video width="760" height="440" controls>
  <source src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/190916/flask04-search.mp4" type="video/mp4">
</video>


### 书籍明细

生成的书籍列表，可以点id访问明细。这里包含三部分：

1. books.csv自带的信息 (book对象)
1. 通过ISBN到Goodreads上查询的信息 (gr_data对象)
1. 用户发布的评级评论 (review对象)

由于作业要求一个用户只能对一本书作评价，所以还有一个用来判断当前用户对此书评论数量的my_review对象。在模板里写一个条件，一旦my_review>0，就禁用提交按钮。

<!-- {% raw %} -->
```html
{% extends "base.html" %}

{% block title %}
{{ book[1] }}
{% endblock %}


{% block control %}
&nbsp;
{% endblock %}

{% block disp %}
<table id="book_info" class="table table-responsive" width="80%">
    <caption><h4>Info of book <strong>{{ book[1] }}</strong></h4></caption>
    <tr>
        <td class="info" style="width:20%">Title</td>
        <td style="width:30%">{{ book[1] }}</td>   
        <td class="info" style="width:20%">Author</td>
        <td style="width:30%">{{ book[2] }}</td>
    </tr>
    <tr>
        <td class="info" style="width:20%">ISBN</td>
        <td style="width:30%">{{ book[3] }}</td>
        <td class="info" style="width:20%">Year</td>
        <td style="width:30%">{{ book[4] }}</td>
    </tr>
    <tr>
        <td class="info" style="width:20%">Number of ratings</td>
        <td style="width:30%">{{ gr_data['work_ratings_count'] }}</td>
        <td class="info" style="width:20%">Average ratings</td>
        <td style="width:30%">{{ gr_data['average_rating'] }}</td>
    </tr>
</table>
<hr>
<table id="review" class="table table-striped table-hover table-responsive" 
 cellspacing="0" width="80%">
 <caption>Reviews of the book <strong>{{ book[1] }}</strong></caption>
<thead>
    <tr>
        <th class="th-sm">ID</th>
        <th class="th-sm">Time</th>
        <th class="th-sm">Reviewer</th>
        <th class="th-sm">Rating</th>
        <th class="th-sm">Review</th>
    </tr>
</thead>
<tbody>
    {% for review in reviews %}
    <tr class='d-flex'>
        <td style="width:10%">{{ review[0] }}</td>
        <td style="width:20%">{{ review[1] }}</td>
        <td style="width:10%">{{ review[2] }}</td>
        <td style="width:10%">{{ review[3] }}</td>
        <td style="width:50%">{{ review[4] }}</td>
    </tr>
    {% endfor %}
</tbody>
</table>
{% endblock %}

{% block misc %}
<form class="form-group" action="{{ url_for('review', book_id=book[0]) }}" method="post">
    <h5>Submit your review comments.</h5>
    <label for="comment" class="sr-only">Input your review comments.</label>
    <textarea name="comment" class="form-control" rows="4" 
        placeholder="Input your review comments for {{ book[1] }}. You can at most submit one comment for a book"></textarea>
    <label for="submit" class="sr-only">Submit</label>
    
    <label for="rating">Rating</label>
    <input id="rating" name="rating" class="rating"  min="0" max="5" step="1" 
        data-size="sm" value='0'>
    
    <button id="submit" class="btn btn-lg btn-primary" 
        {% if my_reviews > 0 %} disabled {% endif %}>Submit</button>
</form>
{% endblock %}
```
<!-- {% endraw %} -->

后端做了很多工作

1. 定义一个`get_gr()`函数，用来读取Goodreads API。我用了Lantern，所以调用了Lantern的SOCKS代理翻墙访问。它能返回json串或者None。
2. 路由写成"/book/<int:book_id>"这种动态形式。
3. 分别拿到book，gr_data和review三处数据，丢到index函数处理。


```python
def get_gr(isbn, api='review_counts', proxy='http://127.0.0.1:55205'
           , success_code=200):
    """Goodreads API data
    """
    urls = {'review_counts': 'https://www.goodreads.com/book/review_counts.json'}
    url = urls[api]
    res = requests.get(
        url, params={'key': '不能告诉你', 'isbns': isbn},
        proxies={'http': proxy, 'https': proxy}, timeout=10)
    if res.status_code == success_code:
        return res.json()
    else:
        return None

@app.route('/book/<int:book_id>', methods=['GET', 'POST'])
def review(book_id):
    """Book detail
    """
    book = sess.execute(
        """SELECT id, title, author, isbn, year FROM book WHERE
        id = :book_id""", {"book_id": book_id}).fetchone()
    gr_data = get_gr(isbn=book[3])
    if gr_data is not None:
        gr_data = gr_data['books'][0]
    else:
        gr_data = {'work_ratings_count': '', 'average_rating': ''}
    my_reviews = sess.execute(
        """SELECT count(*) FROM review WHERE mbr_id = :mbr_id;""",
        {'mbr_id': session.get('act_user')['id']}).fetchone()[0]
    if engine.name == 'sqlite':
        reviews_qry = """SELECT review.id, 
            datetime(review.rev_at, 'localtime'),
            mbr.username, review.rating, review.review FROM
            review LEFT JOIN mbr ON review.mbr_id = mbr.id
            WHERE review.book_id = :book_id"""
    elif engine.name == 'postgresql':
        reviews_qry = """SELECT review.id, 
            review.rev_at at time zone 'utc' at time zone 'cst',
            mbr.username, review.rating, review.review FROM
            review LEFT JOIN mbr ON review.mbr_id = mbr.id
            WHERE review.book_id = :book_id"""
    reviews = sess.execute(
        reviews_qry, {"book_id": book_id}).fetchall()
    # methods
    if request.method == 'GET':
        return render_template(
            'book.html', act_user=session.get('act_user'), book=book
            , reviews=reviews, gr_data=gr_data, my_reviews=my_reviews)
    elif request.method == 'POST':
        comment = request.form.get('comment')
        rating = request.form.get('rating')
        if (comment is not None and comment != '') or \
            (rating is not None and rating != ''):
            sess.execute(
                """INSERT INTO review (mbr_id, book_id, rating, review)
                VALUES (:mbr_id, :book_id, :rating, :review);""",
                {'mbr_id': session.get('act_user')['id'],
                 'book_id': book_id, 'rating': rating, 'review': comment})
            sess.commit()
            flash(Markup(
                """<i class='fa fa-2x fa-check-square-o'></i>
                You have successfully submitted the comment."""), 'success')
        else:
            flash(Markup(
                """<i class='fa fa-2x fa-warning'></i>
                You cannot submit empty rating and comments."""), 'warning')
        return redirect(url_for("review", book_id=book_id))
```

效果如下。

<video width="760" height="440" controls>
  <source src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/190916/flask05-book.mp4" type="video/mp4">
</video>


### 评分评级

如果是提交评分评级，就调用'POST'方法。这里用到了一个评星插件star-rating。我吧它的css和js都下载到了static。


效果如下。

<video width="760" height="440" controls>
  <source src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/190916/flask06-rate.mp4" type="video/mp4">
</video>


### API

自己定义一个API方法。当然，验证key和secret这种专业操作我就不弄了。

在这里，我用request.args.get()方法取id，调用起来就变成xxxx/api/book?id=xx，而不再是xxx/api/book/xx。

```python
@app.route('/api/book', methods=['GET'])
def api():
    book_id = request.args.get('id')
    book = sess.execute(
        """SELECT id, title, author, isbn, year FROM book WHERE
        id = :book_id""", {"book_id": book_id}).fetchone()
    if book is None or len(book) == 0:
        rslt = {'result': False}
    else:
        gr_data = get_gr(isbn=book[3])
        if gr_data is not None:
            gr_data = gr_data['books'][0]
        else:
            gr_data = {'work_ratings_count': '', 'average_rating': ''}
        rslt = {'result': True, 'book': 
            {'title': book[1], 'author': book[2], 'year': book[4]
            , 'isbn': book[3], 'review_count': gr_data['work_ratings_count']
            , 'average_score': gr_data['average_rating']}}
    return jsonify(rslt)
```

效果如下。

<video width="760" height="440" controls>
  <source src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/190916/flask07-API.mp4" type="video/mp4">
</video>


### 响应式布局

参考bootstrap案例写了几个可有可无的@media选择器，概念上有响应式布局的意思了。效果如下。

```css
.form-signin .form-control {
    margin: 10px auto 10px auto;
}
.form-signin .btn {
    margin: 10px auto 10px auto;
}

@media (min-width: 1200px){
    .form-signin {
        width: 20%;
        margin: 10px 40% 10px 40%;
        padding: 10px;
    }
}

@media (min-width: 992px){
    .form-signin {
        width: 30%;
        margin: 10px 35% 10px 35%;
        padding: 5px;
    }
}

@media (min-width: 768px){
    .form-signin {
        margin: 10px auto 10px auto;
        padding: 5px;
    }
}
```

<video width="760" height="440" controls>
  <source src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/190916/flask03-responsive.mp4" type="video/mp4">
</video>


### 其他

`flash`自动消失和star-rating需要专门适配一些javascript。在main.js里。

```javascript
$(document).ready(function () {
    /* alert-dismissable dismiss automatically in 4s */
    window.setTimeout(function() {
        $(".alert-dismissable").fadeTo(1000, 0).slideUp(1000, function(){
            $(this).remove(); 
        });
    }, 4000);  
});

$(document).ready(function(){
    $("#rating").rating();           
});
 
```

就是这样了。

[完]

---

<!-- {% raw %} -->
{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/QRcode.jpg" width="50%" title="扫码关注我的的我的公众号" alt="扫码关注" %}}
<!-- {% endraw %} -->
