
{{% admonition abstract 摘要 %}}
记载了「通往全栈之路上」的一则edx慕课作业：用flask_socketio实现一个粗糙的聊天室。<br/>
【honor code警告】如果你刚巧也注册了这门课，千万不要抄。
{{% /admonition %}}

这是哈佛**继续教育学院**开的的[用Python和Javascript撸网络编程](https://courses.edx.org/courses/course-v1:HarvardX+CS50W+Web/course/) 第三个作业项目。

## [作业要求](https://docs.cs50.net/web/2019/x/projects/2/project2.html)

做一个仿[Slack](www.slack.com)概念的聊天应用。这次不需要用数据库了，但得

要实现以下功能：

1. 能自定义用户名（不能和已有的重复）
2. 能创建频道
3. 能看频道列表
4. 进频道能看到所有消息，但最多显示100条
5. 能在频道中发送消息
6. 能记住频道，下次回来能立即进去
7. 延伸功能：比如删除自己的消息，上传附件等

## 开工

### 准备

- 先要有Python（装了Anaconda），然后要装`Flask`、`Flask-Socketio`包(`pip`)。
- 在环境变量里定义一个"SECRET_KEY"，否则flask_socketio会出错。

### 项目结构

```
|-- application.py
|-- flask.log
|-- + static
|   |-- + css
|   |   \-- style.css
|   \-- + js
|       |-- main.js
|       \-- chat.js
\-- + templates
    |-- _base.html
    |-- channel.html
    \-- channels.html
```

结构不复杂：

- application.py是后端，所有后台功能代码都写在上面。也可以把自定义函数另外写在一个 .py里，import进来。
	- 记得export "application.py" 到环境变量`FLASK_APP`，便于后面直接运行`flask run`。在Windows cmd里，用`set`，PowerShell里用`$Env:FLASK_APP=...`。
- static文件夹放静态文件，css和js。
- templates文件夹放各类html模板（html+jinja2语法写的宏）。


### 基础模板

[_base.html]()是框架模板，后续其他页面模板都会套用(extend)它。


- 样式主要靠bootstrap
- body部分放了几个通用块(block)，head, flash, disp, control, misc。用jinja2结构<!-- {% raw %} -->`{% block xxx %}{% endblock %}`<!-- {% endraw %} -->来占位。
    - 块里面基本都没有进一步定义。只是给导航条加了点功能，如果当前线程有用户登着，就显示个注销按钮，否则就没有。
    - flash块比较特别，定义了一个比较通用的flash渲染宏，到时候只需要在后台的 .py文件里套用`flash`函数就能实现告警框。

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
        <script src="https://code.jquery.com/jquery-3.4.1.min.js"></script>
        <script src="https://cdn.staticfile.org/twitter-bootstrap/3.3.7/js/bootstrap.min.js"></script>
        <script src="{{ url_for('static', filename='js/main.js') }}"></script>
        {% block script %}{% endblock %}
    </head>
    <body>
        <!-- head -->
        <nav id="navbar" class="navbar navbar-default" role="navigation">
            <div class="container-fluid">
                <div class="navbar-header">
                    <a class="navbar-brand mb-0" href="#">Chat</a>
                </div>
                {% if act_user is not none %}
                <form class="navbar-form navbar-right" action="{{ url_for('logout') }}" method="get">
                    <span>Welcome, {{ act_user }}.&nbsp;&nbsp;</span>
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
        <div class="container" id="elem_cont">
            {% block control %}
            {% endblock %}            
        </div>
        <div class="container" id="elem_disp">
            {% block disp %}
            {% endblock %}            
        </div>
        <div class="container" id="elem_misc">
            {% block misc %}
            {% endblock %}            
        </div>
        <!-- footer -->
        <nav class="navbar navbar-fixed-bottom" id="nav_footer" role="navigation">
            <div class="container" id="elem_foot">
                <footer class="navbar" id="footer">
                    <hr>
                    {% block botm_cont %}
                    {% endblock %}
                    <p>&copy; 2019 madlogos</p>
                </footer>
            </div>
        </nav>
    </body>
</html>
```
<!-- {% endraw %} -->

对应地，在application.py里定义一些基本代码，包括导入必要的包、设定好全局对象app和socketio。另外，再定义users和channels两个全局字典对象，用来将聊天相关信息存在服务端备用。由于不配置数据库，这两个全局字典就成了整个应用的存储底层。服务一关，所有数据就烟消云散了。

user是个比较扁平的字典，存用户名和最后一次访问的频道:

```
{
  '<user 1>': '<last visited channel of user 1>',
  '<user 2>': '<last visited channel of user 2>', 
  ...,
  '<user n>': '<last visited channel of user n>', 
}
```

channels是比较复杂的嵌套字典，每个频道都绑一个字典，包含'created'和双层列表'chats'：

```
{
  '<channel 1>': 
    {
      'created': '<creation time of channel 1>',
      'chats': 
        [
          ['<post 1 user>', '<post 1 time>', '<post 1 msg>'],
          ['<post 2 user>', '<post 2 time>', '<post 2 msg>'],
          ...,
          ['<post n user>', '<post n time>', '<post n msg>'],
        ]
    },
  ...
}
```

chats里包含的就是一条条消息。讲究点的话，应该也存成字典，毕竟time和user、msg类型不同。不过偷懒天经地义，简化处理也没啥不可以。

主路由"/"绑定函数`index`。假如当前`session`里有'act_user'，那么调用`get_channels()`函数（也就是跑去频道列表），否则跳转去login页。

```python
# -*- coding: UTF-8 -*-

import os
import datetime
import urllib.parse
from flask import Flask, flash, render_template, redirect, session, request, url_for, jsonify
from flask_socketio import SocketIO, emit
from jinja2 import Markup
import logging


app = Flask(__name__, static_folder="static")
app.config["SECRET_KEY"] = os.getenv("SECRET_KEY")
app.config['SESSION_TYPE'] = 'filesystem'
socketio = SocketIO(app)


users = dict()
channels = dict()

@app.route("/")
def index():
    if session.get("act_user") is None:
        return redirect(url_for("login"))
    else:
        return get_channels()
```

最后在__main__里加一点代码，配置日志输出。部署到生产环境时，要记得把`app.debug`设为False。

```python
if __name__ == '__main__':
    app.debug = True
    handler = logging.FileHandler("flask.log", encoding="UTF-8")
    handler.setLevel(logging.DEBUG)
    logging_format = logging.Formatter(
        '%(asctime)s - %(levelname)s - %(filename)s - %(funcName)s - %(lineno)s - %(message)s')
    handler.setFormatter(logging_format)
    app.logger.addHandler(handler)
    socketio.run(app)
```

### 登录

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/landing.png" title="图 | 登陆页" %}}

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/display_name.png" title="图 | 创建用户" %}}

一般，功能由html模板和python函数配合完成。由于这次的登录功能很简单，当前线程给自己随便起个用户名就行，所以把channels.html当成事实上的首页，在上面套个悬浮页来实现登录。

<a name="channels_html"></a>channels.html模板代码：

<!-- {% raw %} -->
```html
{% extends "_base.html" %}

{% block title %}
Channels
{% endblock %}

{% block control %}
{% if act_user is none %}
<div>
    <button id="addName" class="btn btn-lg btn-primary" data-toggle="modal" data-target="#inputName">
        Enter your display name
    </button>
</div>
{% else %}
<form class="form-inline" action="{{ url_for('set_channels') }}" method="post">
    <div class="form-group">
        <label for="new_channel" class="sr-only">Add a channel</label>
        <input type="text" name="new_channel" class="form-control" placeholder="Create a new channel">
        <label for="add_channel" class="sr-only">Submit</label>
        <button id="add_channel" class="btn btn-md btn-primary">Submit</button>
    </div>
</form>
{% endif %}
{% endblock %}

{% block disp %}
{% if act_user is none %}
<div id="inputName" class="modal fade" role="dialog">
    <div class="modal-dialog">
        <!-- Modal content-->
        <div class="modal-content">
            <form action="{{ url_for('login') }}" method="post">
                <div class="modal-header"><label for="displayName">Your display name</label></div>
                <div class="modal-body">
                    <input type="text" id="displayName" name="displayName" class="form-control validate" placeholder="Your display name">
                </div>
                <div class="modal-footer"><button type="submit" class="btn btn-lg btn-primary">Submit</button></div>
            </form>
        </div>
    </div>
</div>
{% else %}
<table id="last_channels" class="table table-striped table-hover table-responsive" cellspacing="0" width="80%">
    <thead>
        <tr><th class="th-sm">The latest channel you accessed</th></tr>
    </thead>
    <tbody>
        {% if last_visit is not none %}
            <tr><td><a href="channel/{{ last_visit }}">{{ last_visit }}</a></td></tr>
        {% endif %}
    </tbody>
</table>

<table id="tbl_channels" class="table table-striped table-hover table-responsive" cellspacing="0" width="80%">
    <thead>
        <tr>
            <th class="th-sm">Channel</th>
            <th class="th-sm">Created at</th>
        </tr>
    </thead>
    <tbody>
        {% if channels|length > 0 %}
            {% for channel in channels %}
            <tr>
                <td><a href="{{ url_for('get_channel', channel=channel['name']) }}">
                    {{ channel['name'] }}</a></td>
                <td>{{ channel['created'].strftime('%Y-%m-%d %H:%M:%S') }}</td>
            </tr>
            {% endfor %}
        {% endif %}
    </tbody>
</table>
{% endif %}
{% endblock %}
```
<!-- {% endraw %} -->

页面根据act_user的状态显示为两套。如果act_user是None，即当前用户没登录，那就显示一颗id为"addName"的按钮，target指向锚点"inputName"。inputName是当前页面上的一个modal对话框，里面包了个表单，套住一个名叫"displayName"的输入框。当用户填完屏显名，点<kbd>submit</kbd>按钮提交，触发login路由的POST事件。

如果act_user不是None，即当前用户已经登录，那就显示另一套内容：本user的前次访问频道，和全部频道列表。具体在[频道列表](#频道列表)里讲。

到application.py看看login路由定义了些什么。

```python
@app.route("/login", methods=['GET', 'POST'])
def login():
    if request.method == "GET":
        return get_channels()
    elif request.method == "POST":
        usernm = request.form.get("displayName")
        if usernm not in users.keys():
            flash(Markup(
                """<i class='fa fa-2x fa-info-circle'></i>
                New username %s created.""" % (usernm)), 'info')
            users[usernm] = None

        session['act_user'] = usernm
        return(redirect(url_for("get_channels")))
```

GET方法下，调用`get_channels()`函数，显示频道列表。如果用户没登陆，直接GET login呢？不要紧，这种情况下[channels.html](#channels_html)只会展示<kbd>addName</kbd>按钮和inputName表单。

而POST方法下（也就是提交了inputName表单后），判断一下displayName里填的名字是否已经在全局对象users里，没有的话就创建一个。完事后重定向到get_channels绑定的路由，也就是频道列表。

### 注销

有登录就有注销。_base.html里注销按钮已经绑定了logout路由，所以只要定义logout路由的后台绑定函数就行了。这里的注销也很简单，登出后清空`session`中的`act_user`对象，转跳回登录页。

```python
@app.route("/logout")
def logout():
    session.pop('act_user', None)
    flash(Markup(
        """<i class='fa fa-2x fa-check-square-o'></i>
        You have logged out."""), 'success')
    return redirect(url_for("index"))
```

### 频道列表

完成登录后，就进入频道列表，其实就是channels.html换了套内容呈现，包括前次访问的频道和全部频道列表。

看一下后台python代码。分别对channels路由的GET和POST方法定义了两个函数`get_channels()`和`set_channels()`。

```python
@app.route("/channels", methods=['GET'])
def get_channels():
    last_visit=users.get(session.get("act_user"))
    if last_visit not in channels.keys():
        last_visit = None
    return render_template(
        "channels.html", act_user=session.get("act_user"), 
        channels=[{'name':k, 'created':channels[k]['created']} for k in channels], 
        last_visit=last_visit)


@app.route("/channels", methods=['POST'])
def set_channels():
    new_channel = request.form.get("new_channel").strip()
    if new_channel != "":
        if new_channel in channels.keys():
            flash(Markup(
                """<i class='fa fa-2x fa-warning'></i>
                Channel %s already exists.""" % (new_channel)), 
                'warning')
        else:
            flash(Markup(
                """<i class='fa fa-2x fa-check-square-o'></i>
                The new channel %s has been created.""" % (new_channel)),
                'success')
            channels[new_channel] = {"created": datetime.datetime.now(), "chats":[]}
    else:
        flash(Markup(
            """<i class='fa fa-2x fa-warning'></i>
            Channel name cannot be empty."""), 'warning')
    return redirect(url_for("get_channels"))
```

GET方法下，从全局对象`users`里取到上次访问的频道名，存入last_visit。假如last_visit不在`channels`里，last_visit覆写为None。把全局channels的键和created值取出来拼成列表channels，再把这个channels连同last_visit都发送给[channels.html模板](#channels_html)解析渲染。回顾模板里的代码，如果last_visit或channels为None，对应的表格就只显示表头。

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/last_visit.png" title="图 | 上次访问的频道" %}}

POST方法下，服务器从表单里提取"new_channel"。假如new_channel在全局对象channels里已经存在，就flash甩个错误警报。假如不存在，那就往channels里插入一个新列表，新频道就生成了。最后转跳回channels.html，实现刷新。

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/new_channel.png" title="图 | 创建频道" %}}

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/channels.png" title="图 | 频道列表" %}}

### 频道明细

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/chatting.png" title="图 | 在频道里聊天" %}}

点击频道名，就进到频道明细。其实就是聊天室应用。和常规网络应用相比，它要解决两个特殊问题：

1. 提交的信息要实时广播给其他所有用户，否则就不能算聊天室。
1. 不能每次提交信息都刷新一次页面，否则用户就会崩溃，服务器负担也重。

这就需要使用socket和ajax。socket实现进程间通信，不再需要整包整包地收发HttpResponse，并刷新页面来响应。而ajax实现异步数据交换，基于socket交换数据后直接用Javascript在客户端更新网页中的特定内容。速度快，服务器负荷小。

<a name="channel_html"></a>channel.html模板：

<!-- {% raw %} -->
```html
{% extends "_base.html" %}

{% block title %}
Channel {{ channel }}
{% endblock %}

{% block script %}
<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/handlebars.js/4.0.11/handlebars.min.js"></script>
<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/1.3.6/socket.io.min.js"></script>

<!-- handlebars template -->
<script id="chatPost" type="text/x-handlebars-template">
    <tr>
        {% raw -%}
        <td width="10%">
            {{#if same_user }}
                <span data-class="post_user" style='color:dodgerblue'>{{ post_user }}</span>
            {{ else }}
                <span data-class="post_user" style='color:lightsalmon'>{{ post_user }}</span>
            {{/if}}
        </td>
        <td width="20%">
            {{#if same_user }}
                <span data-class="post_time" style='color:dodgerblue'>{{ post_time }}</span>
            {{ else}}
                <span data-class="post_time" style='color:lightsalmon'>{{ post_time }}</span>
            {{/if}}
        </td>
        <td width="auto">
            {{#if same_user }}
                <span data-class="post_msg" style='color:dodgerblue'>{{ post_msg }}</span>
                <button data-class="del" style="float:right" class="btn btn-sm btn-danger">Delete</button>
            {{ else }}
                <span data-class="post_msg" style='color:lightsalmon'>{{ post_msg }}</span>
            {{/if}}
        </td>
        {%- endraw %}
    </tr>
</script>

<script type="text/javascript" src="{{ url_for('static', filename='js/chat.js') }}"></script>
<script type="text/javascript">
    document.addEventListener('DOMContentLoaded', () => {
        document.querySelector("#msgTbl").innerHTML = format_chats({{ chats|tojson }}, '{{ act_user }}');
    });
</script>
{% endblock %}

{% block disp %}
<table id="tbl_chat" class="table table-striped table-hover table-responsive" 
 cellspacing="0" width="80%">
    <caption>You are now in channel [{{ channel }}].</caption>
    <thead>
        <tr>
            <th class="th-sm">User</th>
            <th class="th-sm">Time</th>
            <th class="th-sm">Message</th>
        </tr>
    </thead>
    <tbody id="msgTbl">
    </tbody>
</table>
{% endblock %}

{% block botm_cont %}
<div class="container">
    <div class="form-group" id="inputMsg">
        <label for="msg" class="sr-only">Input your message</label>
        <textarea id="msg" name="msg" class="form-control" placeholder="Input your message" rows="4" width="75%"></textarea>
        <label for="send" class="sr-only">Send</label>
        <button id="send" type="submit" class="btn btn-md btn-primary">Send (Shift+Enter)</button>
        <a href="/channels">&nbsp;&nbsp;&gt;&gt;&gt;Go back to channel list.</a>
    </div>
</div>
{% endblock %}
```
<!-- {% endraw %} -->

在channel.html模板中，script块里引入一大堆JS。比如用来动态生成HTML内容的handlebars.js，和处理socket的socket.io.js。disp块里定义了显示对话的表格"tbl_chat"，其中tbody留空，赋个id="msgTbl"。

handlebars模板有一些特殊的语法规范，比如要转义的部分需要加<!-- {% raw %} -->{% raw -%}{%- endraw %}<!-- {% endraw %} -->标签、控制结构用{{#if}}{{else}}{{/if}}。它能解析变量，动态合成HTML。

html模板里直接嵌入一段JS监听代码，当加载页面时，提取chats和act_user，填充到tbody（也就是msgTbl）。chats要加管道函数tojson，把文本转成json。代码如下：

<!-- {% raw %} -->
```javascript
document.addEventListener('DOMContentLoaded', () => {
	document.querySelector("#msgTbl").innerHTML = format_chats({{ chats|tojson }}, '{{ act_user }}');
});
```
<!-- {% endraw %} -->

这段代码用到了`format_chats()`函数。这是个自定义函数，从chat.js里加载：

```javascript
// template for chatPost
const template = Handlebars.compile(document.querySelector('#chatPost').innerHTML);
var act_user = "";
var act_channel = "";

function format_chats(json_data, act_user=act_user){
    var output = '';
    json_data.forEach(function(sub_json) {
        const rslt_date = new Date(sub_json[1]);
        const rslt = template({'post_user': decodeURI(sub_json[0]),
            'post_time': format_date(rslt_date), 
            'post_msg': decodeURI(sub_json[2]),
            'same_user': decodeURI(sub_json[0]) == act_user});
        output += rslt;
    });
    return output;
};

function format_date(date){
    const yr = date.getYear() + 1900;
    const mo = date.getMonth() + 1;
    const dt = date.getDate();
    const hr = date.getHours();
    const mi = date.getMinutes();
    const se = date.getSeconds(); 
    const ms = date.getMinutes();
    return yr + "-" + lead_zero(mo) + "-" + lead_zero(dt) + " " +
        lead_zero(hr) + ":" + lead_zero(mi) + ":" + lead_zero(se) + 
        "." + lead_zero(ms, 3);
};

function lead_zero(num, digits=2){
    /* put zeros in front the num */
    return (Array(digits).join(0) + num).slice(-digits);
};

function format_tz_offset(offset_min){
    const sign = (offset_min < 0) ? '+' : '-';
    const hr = Math.abs(offset_min) / 60;
    const mi = Math.abs(offset_min) % 60;
    return sign + lead_zero(hr) + ":" + lead_zero(mi);
};
```

template对象得先用Handlebars编译一下，绑定handlebars模板对象chatPost的innerHTML，把不同的变量json喂给template，就能产生一个个实例。act_user和act_channel指当前用户和频道，起始时置空。

- `format_chats()`负责将json数据套入[channel.html模板](#channel_html) 中的handlebars模板"chatPost"里，解析参数后生成相应的html代码。这个输入参数json结构是固定的，包含post_user、post_time、post_msg，也就是全局对象channels里每个channel中的chats字典。
	- 非常英明地用了`decodeURI()`和`encodeURI()`函数，发到服务器的数据都先编码，接到服务器数据都先解码，这样用中文时就不会乱码了。
- `format_date()`是一个整形函数，把日期输出为mmmm-yy-dd hh:mm:ss的标准格式。
	- `lead_zero()`是另一个整形函数，在数字前加零补位
- `format_tz_offset()`生成时区，以+/-hh:mm的形式输出，比如北京时间就是+08:00，后面有用。


先创建socket连接。和服务器实现连接(`on('connect')`)后，服务器会发一个"send username"请求，把当前用户名和当前频道发到客户端。客户端接到"send username"请求，就能获得act_user和act_channel，用于后续的解析计算。

```python
@socketio.on("connect")
def send_username():
    emit('send username', {'act_user': session.get('act_user'), 
         'act_channel': users.get(session.get('act_user'))})
```

前端JS里，页面监听代码里有一段用来处理"send username"请求：

```javascript
document.addEventListener('DOMContentLoaded', () => {
	/* Some other codes here */
	
	socket.on('send username', data => {
        // console.log(JSON.stringify(data));
        act_user = decodeURI(data.act_user);
        act_channel = decodeURI(data.act_channel);
    });
	
	/* Some other codes here */
}
```

channel/<channel>路由的代码：

```python
@app.route("/channel/<channel>", methods=['GET'])
def get_channel(channel):
    if session.get('act_user') is None:
        flash(Markup(
            """<i class='fa fa-2x fa-exclamation-circle'></i>
            You are not logged in."""), 'danger')
        return redirect(url_for("index"))
    
    users[session.get('act_user')] = channel
    return render_template(
        "channel.html", act_user=session.get("act_user"), channel=channel,
        chats=channels[channel]['chats'])
```

先校验当前用户是否登录。没问题的话就渲染channel.html模板了。服务器从全局对象channels里提取该频道的对话消息chats发给客户端。客户端用`format_chats()`把chats解析后，套进handlebars模板，再呈现到表格里。


### 发消息

当点击<kbd>send</kbd>，客户端就发一个"send msg"请求，把json`{'user': encodeURI(act_user), 'time': post_time, 'msg': encodeURI(msg), 'channel': encodeURI(act_channel)}`发射(socket.emit)到服务器，交给flask_socketio处理。

```javascript
document.addEventListener('DOMContentLoaded', () => {
    /* connect to socket */
    var socket = io.connect(location.protocol + "//" + document.domain + ":" + location.port);

    socket.on('connect', () => {
        document.querySelector("#send").onclick = () => {
            const msg = document.querySelector('#msg').value;
            const post_time = new Date();
            // console.log(msg);
            socket.emit('send msg', {'user': encodeURI(act_user), 'time': post_time, 
                'msg': encodeURI(msg), 'channel': encodeURI(act_channel)});
        };
    });
    
	/* 'send username' codes */

    socket.on('emit msg', data => {
        const posttime = new Date(data.time);
        // console.log(JSON.stringify(data));
        const content = template(
            {'post_user': decodeURI(data.user), 'post_time': format_date(posttime), 
             'post_msg': decodeURI(data.msg), 'same_user': decodeURI(data.user)==act_user});
        document.querySelector("#msgTbl").innerHTML += content;
        document.querySelector("#msg").value = '';
        /* scroll to the page bottom */
        window.scrollTo(0, document.body.scrollHeight);
    });
});
```

服务器接到这个"send msg"请求后，怎么处理呢？看application.py：

```python
@socketio.on("send msg")
def emit_msg(data):
    if data['msg'] != '':
        channel = urllib.parse.unquote(data['channel'])
        chats = channels[channel]['chats']
        chats.append([data['user'], data['time'], data['msg']])
        if len(chats) > 100:
            chats = chats[(len(chats)-100):]
        channels[channel]['chats'] = chats
        emit('emit msg', 
             {'user': data['user'], 'time': data['time'], 'msg': data['msg']},
             broadcast=True)
```

假如"send msg"请求数据不为空，那就各种解析：解析出channel、chats（只保留最后100条），打包成`{'user': data['user'], 'time': data['time'], 'msg': data['msg']}`这样格式的json，发射(emit)回客户端。这里设`broadcast=True`，其他聊天室的客户端也都会通过广播机制接到这些数据，并通过ajax更新页面内容。

而JS中，`socket.on('emit msg')`代码会将从服务器收到的广播数据套进handlebars模板里解析，再拼合成HTML填充到msgTbl里，同时清空输入文本框，自动定位到页面底部。

为了方便输入，设置为<kbd>Shift+Enter</kbd>发送消息。这需要一段键盘事件监听代码，只要msg文本框里出现shift+enter，就阻断默认动作，触发<kbd>send</kbd>的点击事件。

```javascript
window.onload = function(){
    const msgArea = document.getElementById("msg");
    msgArea.addEventListener('keypress', evt => {
        if (evt.keyCode === 13 && evt.shiftKey) {
            evt.preventDefault()
            document.querySelector('#send').click();
        };
    });
};
```

### 删除消息

由于定义了`same_user`变量，因此handlebars在拼装时，会根据消息作者是否与当前用户相同，在对应的消息后加<kbd>删除</kbd>按钮。这就避免了误删。

删除自己的消息是通过另一端监听代码实现的，原理很简单，定位target的父元素，调用remove()方法：

```javascript
document.addEventListener("click", evt => {
    var socket = io.connect(location.protocol + "//" + document.domain + ":" + location.port);
    const tgt = evt.target;
    if (tgt.dataset.class === 'del'){
        const tz_offset = new Date().getTimezoneOffset();
        const elem = tgt.parentElement.parentElement;
        const elem_user = elem.cells[0].innerText;
        const elem_time = new Date(elem.cells[1].innerText.replace(/\s/g, 'T') +
            format_tz_offset(tz_offset));
        const output = {'user': encodeURI(elem_user), 
            'time': elem_time, 'channel': encodeURI(act_channel)};
        // console.log(JSON.stringify(output));
        elem.remove();
        socket.emit("del msg", output);
    };
});
```

因为没有数据库，所以并没有考虑设置主键。这带来一个问题，前端删除一条消息，怎么同步到服务器呢？如果不能从服务器同步删除，下次回来，这条消息还在，多尴尬。我构造了一个字典`{'user': encodeURI(elem_user), 'time': elem_time, 'channel': encodeURI(act_channel)}`，回传给服务器。服务器校验用户、时间和频道条件，从channels里删掉对应的消息。

服务器端的处理代码：

```python
@socketio.on("del msg")
def del_msg(data):
    channel = urllib.parse.unquote(data['channel'])
    chats = channels[channel]['chats']
    app.logger.info(str(chats))
    for i in range(len(chats)):
        app.logger.info([data['user'], data['time']])
        if chats[i][0] == data['user'] and chats[i][1] == data['time']:
            channels[channel]['chats'].pop(i)
            break
```

每次删除都要遍历整个字典，多字段匹配。感觉效率堪忧。还是有必要考虑一下主键设计。


### 其他

`flash`自动消失和star-rating需要专门适配一些javascript。在main.js里。

```javascript
$(document).ready(function () {
    /* alert-dismissable dismiss automatically in 3s */
    window.setTimeout(function() {
        $(".alert-dismissable").fadeTo(1000, 0).slideUp(1000, function(){
            $(this).remove(); 
        });
    }, 3000);  
});
```

这就实现了这则应用的主要功能。

[完]

---

<!-- {% raw %} -->
{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/QRcode.jpg" width="50%" title="扫码关注我的的我的公众号" alt="扫码关注" %}}
<!-- {% endraw %} -->
