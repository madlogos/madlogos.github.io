
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

<!--more-->

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

[_base.html](https://github.com/madlogos/edx_cs50/blob/master/project2/templates/_base.html)是框架模板，后续其他页面模板都会套用(extend)它。


- 样式主要靠bootstrap
- body部分放了几个通用块(block)，head, flash, disp, control, misc。用jinja2结构<!-- {% raw %} -->`{% block xxx %}{% endblock %}`<!-- {% endraw %} -->来占位。
    - 块里面基本都没有进一步定义。只是给导航条加了点功能，如果当前线程有用户登着，就显示个注销按钮，否则就没有。
    - flash块比较特别，定义了一个比较通用的flash渲染宏，到时候只需要在后台的 .py文件里套用`flash`函数就能实现告警框。

<!-- {% raw %} -->
```html
<!-- templates/_base.html -->
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
                    <button type="button" class="close" data-dismiss="alert">
                      &times;
                    </button>
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
  '<user n>': '<last visited channel of user n>'
}
```

channels是比较复杂的嵌套字典，每个频道都绑一个字典，包含'created'、'max_id'和双层字典'chats'：

```
{
  '<channel 1>': 
    {
      'created': '<creation time of channel 1>',
      'max_id': <max id of chats>,
      'chats': 
        {
          <id 1>: 
              {'user': '<post 1 user>', 'time': '<post 1 time>', 
               'msg': '<post 1 msg>'
              },
          <id 2>: 
              {'user': '<post 2 user>', 'time': '<post 2 time>', 
               'msg': '<post 2 msg>'
              },
          ...,
          <id n>: 
              {'user': '<post n user>', 'time': '<post n time>', 
               'msg': '<post n msg>'
              }
        }
    },
  ...
}
```

chats里包含的就是一条条消息，以id为键，包起'user'、'time'和'msg'。

主路由"/"绑定函数`index`。假如当前`session`里有'act_user'，那么调用`get_channels()`函数（也就是跑去频道列表），否则跳转去login页。

```python
# -*- coding: UTF-8 -*-
# application.py
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

最后在__main__里加一点代码，配置日志输出。运行`python application.py`时，会自动运行这部分。如果继续用`flask run`，这部分不会自动运行。还会报警告，WebSocket无法启用，用Workzeug跑Flask-SocketIO。这是因为新版的Flask在服务端功能做了简化，不再支持WebSocket。

{{% admonition tip "注意" %}}
部署到生产环境时，要记得把`app.debug`设为False。
{{% /admonition %}}

```python
# application.py
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

一般，功能由html模板和python函数配合完成。由于这次的登录功能很简单，当前线程给自己随便起个用户名就行，所以把channels.html当成事实上的首页，在上面套个悬浮页来实现登录。

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/display_name.png" title="图 | 创建用户" %}}

<a name="channels_html"></a>[channels.html](https://github.com/madlogos/edx_cs50/blob/master/project2/templates/channels.html)模板代码：

<!-- {% raw %} -->
```html
<!-- templates/channels.html -->
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
                <div class="modal-header">
                  <label for="displayName">Your display name</label>
                </div>
                <div class="modal-body">
                    <input type="text" id="displayName" name="displayName"
                     class="form-control validate" placeholder="Your display name">
                </div>
                <div class="modal-footer">
                  <button type="submit" class="btn btn-lg btn-primary">
                    Submit
                  </button>
                </div>
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

现在到application.py看看login路由定义了些什么。

```python
# application.py
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

<!-- {% raw %} -->
{{% admonition note "flash()函数" false %}}
上面的Python代码用到了`flash(<text>, <type>)` 函数，它会发送一个`flash`请求到Flask前端，产生一个Bootstrap风格的告警。type只能是Bootstrap认识的"danger", "warning", "success", "info"这类。

为了让告警显示图标，用到了FontAwesome（_base.html模板里已经引入）。直接把`<i class="xxx">` flash到前端，无法解析出图标，需要包一个`Markup()`，以markup对象的形式传递，前端解析后自动交给fontawesome.js处理。
{{% /admonition %}}
<!-- {% endraw %} -->

### 注销

有登录就有注销。_base.html里注销按钮已经绑定了logout路由，所以只要定义logout路由的后台绑定函数就行了。这里的注销也很简单，登出后清空`session`中的`act_user`对象，转跳回登录页。

```python
# application.py
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

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/channels.png" title="图 | 频道列表" %}}

看一下后台python代码。分别对channels路由的GET和POST方法定义了两个函数`get_channels()`和`set_channels()`。

```python
# application.py
@app.route("/channels", methods=['GET'])
def get_channels():
    app.logger.info(str(channels))
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
            channels[new_channel] = {"created": datetime.datetime.now(), 
                "max_id": 0, "chats":{}}
    else:
        flash(Markup(
            """<i class='fa fa-2x fa-warning'></i>
            Channel name cannot be empty."""), 'warning')
    return redirect(url_for("get_channels"))
```

GET方法下，从全局对象`users`里取到上次访问的频道名，存入last_visit。假如last_visit不在`channels`里，last_visit覆写为None。把全局channels的键和created值取出来拼成列表channels，再把这个channels连同last_visit都发送给[channels.html模板](#channels_html)解析渲染。回顾模板里的代码，如果last_visit或channels为None，对应的表格就只显示表头。

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/last_visit.png" title="图 | 上次访问的频道" %}}

POST方法下，服务器从表单里提取"new_channel"。假如new_channel在全局对象channels里已经存在，就`flash`甩个错误警报。假如不存在，那就往channels里插入一个新字典（包含'created'、'max_id'和'msg'字典)，新频道就生成了。最后转跳回channels.html，实现刷新。

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/new_channel.png" title="图 | 创建频道" %}}

### 频道明细


点击频道名，就进到频道明细。其实就是聊天室应用。和常规网络应用相比，它要解决两个特殊问题：

1. 提交的信息要实时广播给其他所有用户，否则就不能算聊天室。
1. 不能每次提交信息都刷新一次页面，否则用户就会崩溃，服务器负担也重。

这就需要使用socket和ajax。socket实现进程间通信，不再需要整包整包地收发HttpResponse，并刷新页面来响应。而ajax实现异步数据交换，基于socket交换数据后直接用Javascript在客户端更新网页中的特定内容。速度快，服务器负荷小。

<a name="channel_html"></a>channel.html模板：

<!-- {% raw %} -->
```html
<!-- templates/channel.html -->
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
                <span data-class="post_user" style='color:dodgerblue'>
                  {{ post_user }}
                </span>
            {{ else }}
                <span data-class="post_user" style='color:lightsalmon'>
                  {{ post_user }}
                </span>
            {{/if}}
        </td>
        <td width="20%">
            {{#if same_user }}
                <span data-class="post_time" style='color:dodgerblue'>
                  {{ post_time }}
                </span>
            {{ else}}
                <span data-class="post_time" style='color:lightsalmon'>
                  {{ post_time }}
                </span>
            {{/if}}
        </td>
        <td width="auto">
            {{#if same_user }}
                <span data-class="post_msg" style='color:dodgerblue'>
                  {{ post_msg }}
                </span>
                <button data-id="{{ post_id }}" data-class="del" style="float:right"
                 class="btn btn-sm btn-danger"">Delete</button>
            {{ else }}
                <span data-class="post_msg" style='color:lightsalmon'>
                  {{ post_msg }}
                </span>
            {{/if}}
        </td>
        {%- endraw %}
    </tr>
</script>

<script type="text/javascript">
    var act_user = decodeURI("{{ act_user }}");
    var act_channel = decodeURI("{{ channel }}");
    document.addEventListener('DOMContentLoaded', () => {
        document.querySelector("#msgTbl").innerHTML = format_chats({{ chats|tojson }}, '{{ act_user }}');
    });
</script>
<script type="text/javascript" src="{{ url_for('static', filename='js/chat.js') }}"></script>
{% endblock %}

{% block control %}
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
        <textarea id="msg" name="msg" class="form-control" placeholder="Input your message" 
         rows="4" width="75%"></textarea>
        <label for="send" class="sr-only">Send</label>
        <button id="send" type="submit" class="btn btn-md btn-primary">
          Send (Shift+Enter)
        </button>
        <a href="/channels">&nbsp;&nbsp;&gt;&gt;&gt;Go back to channel list.</a>
    </div>
</div>
{% endblock %}
```
<!-- {% endraw %} -->

在channel.html模板中，script块里引入一大堆JS。比如用来动态生成HTML内容的handlebars.js，和处理socket的socket.io.js。disp块里定义了显示对话的表格"tbl_chat"，其中tbody留空，赋个id="msgTbl"。

<!-- {% raw %} -->
handlebars模板有一些特殊的语法规范，比如要转义的部分需要加{% raw -%}...{%- endraw %} 而标签、控制结构用{{#if}}...{{else}}...{{/if}}。它能解析变量，动态合成HTML。上面代码里的handlebars模板主要是根据act_user和发帖人是否为同一人，显示为不同的颜色。
<!-- {% endraw %} -->

html模板里直接嵌入一段JS监听代码，当加载页面时，提取chats和act_user，填充到tbody（也就是#msgTbl）。就是这段：

<!-- {% raw %} -->
```javascript
var act_user = decodeURI("{{ act_user }}");
var act_channel = decodeURI("{{ channel }}");
document.addEventListener('DOMContentLoaded', () => {
    document.querySelector("#msgTbl").innerHTML = format_chats({{ chats|tojson }}, '{{ act_user }}');
});
```
<!-- {% endraw %} -->

{{% admonition tip "注意" %}}
chats用tojson函数处理，把序列化的文本转成json。在Flask模板里，调用函数的形式是`对象|方法`，而不是传统的`函数(参数)`形式。
{{% /admonition %}}

这里定义了两个变量：act_user和act_channel，把当前线程的用户名和当前频道从html模板传到后面引入的javascript里，也就是[chat.js](https://github.com/madlogos/edx_cs50/blob/master/project2/static/js/chat.js)。

这段JS代码用到了`format_chats()`函数。这是个自定义函数，从chat.js里加载：

```javascript
/* static/js/chat.js */
// template for chatPost
const template = Handlebars.compile(document.querySelector('#chatPost').innerHTML);

function format_chats(json_data, act_user=act_user){
    console.log(JSON.stringify(json_data)); 
    /* json_data is a dict */
    var output = '';
    Object.keys(json_data).forEach(function(key) {
        const rslt_date = new Date(json_data[key]['time']);
        const rslt = template({
            'post_id': key,
            'post_user': decodeURI(json_data[key]['user']),
            'post_time': format_date(rslt_date), 
            'post_msg': decodeURI(json_data[key]['msg']),
            'same_user': decodeURI(json_data[key]['user']) == act_user});
        output += rslt;
    });
    return output;
};
```

template对象得先用Handlebars编译一下，绑定handlebars模板对象chatPost的innerHTML，把不同的变量(post_id, post_user...)json喂给template，就能编译产生一个个实例。

`format_chats()`负责将json数据套入[channel.html模板](#channel_html) 中的handlebars模板"chatPost"里，解析参数后生成相应的html代码。这个输入参数json结构是固定的，包含post_user、post_time、post_msg，也就是全局对象channels里每个channel中的chats字典。

{{% admonition note "要点" false %}}
非常英明地用了`decodeURI()`和`encodeURI()`函数，发到服务器的数据都先编码，接到服务器数据都先解码，这样用中文时就不会乱码了。
{{% /admonition %}}

在格式化日期时，用到了另外两个函数`format_date()`和`lead_zero()`。也定义在chat.js里。

```javascript
/* static/js/chat.js */
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
```

不这样处理也可以，Javascript会按默认的格式渲染日期。

回过头再看'channel/<channel>'路由的代码：

```python
# application.py
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

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/chatting.png" title="图 | test1和test2在频道里聊天" %}}

当点击<kbd>send</kbd>，客户端就发一个"send msg"请求，把json`{'user': encodeURI(act_user), 'time': post_time, 'msg': encodeURI(msg), 'channel': encodeURI(act_channel)}` "发射"(`socket.emit`)到服务器，交给flask_socketio处理。

<a name="chatjs"></a>
```javascript
/* static/js/chat.js */
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

    socket.on('emit msg', data => {
        console.log('emit msg: ' + JSON.stringify(data));
        if (act_channel == data.channel){
            const posttime = new Date(data.time);
            const content = template(
                {'post_id': data.id, 'post_user': decodeURI(data.user), 
                 'post_time': format_date(posttime), 
                 'post_msg': decodeURI(data.msg), 
                 'same_user': decodeURI(data.user)==act_user});
            document.querySelector("#msgTbl").innerHTML += content;
            document.querySelector("#msg").value = '';
            /* scroll to the page bottom */
            window.scrollTo(0, document.body.scrollHeight);
        };
    });
});
```

服务器接到这个"send msg"请求后，怎么处理呢？看application.py：

```python
# application.py
@socketio.on("send msg")
def emit_msg(data):
    # if msg is blank, do not emit
    if data['msg'] != '':
        channel = urllib.parse.unquote(data['channel'])
        chats = channels[channel]['chats']
        id = channels[channel]['max_id']
        chats[str(id)] = {'user': data['user'], 'time': data['time'], 
            'msg': data['msg']}
        channels[channel]['max_id'] += 1
        if len(chats) > 100:
            chats = chats[(len(chats)-100):]
        channels[channel]['chats'] = chats

        emit('emit msg', 
             {'id': id, 'user': data['user'], 'time': data['time'], 
              'msg': data['msg'], 'channel': channel}, broadcast=True)
```

假如"send msg"请求数据不为空，那就各种解析：解析出channel、chats（只保留最后100条），打包成`{'id': id, 'user': data['user'], 'time': data['time'], 'msg': data['msg'], 'channel': channel}`这样格式的json，发射(emit)回客户端。这里设`broadcast=True`，其他聊天室的客户端也都会通过广播机制接到这些数据，并通过JS实现页面内更新。

而JS中，`socket.on('emit msg')`部分的[代码](#chatjf)会将从服务器收到的广播数据套进handlebars模板里解析，再拼合成HTML填充到'#msgTbl'里，同时清空输入文本框，自动定位到页面底部。

{{% admonition note "要点" false %}}
这里，加了一个判断。只有act_channel和从前端收到的data['channel']相同，才渲染handlebars模板，更新页面。不加这条判断的话，就会发生灾难性“串台”现象，任何其他频道的新增消息，都会被广播到其他频道里。
{{% /admonition %}}

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/dif_channel.png" title="图 | 不同频道不会'串台'" %}}

为了方便输入，设置为<kbd>Shift+Enter</kbd>发送消息。这需要一段键盘事件监听代码，只要msg文本框里出现shift+enter，就阻断默认动作，触发<kbd>send</kbd>的点击事件。

```javascript
/* static/js/chat.js */
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

{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/191101/del_msg.png" title="图 | 只能删除自己发的消息" %}}

删除自己的消息是通过另一端监听代码实现的，原理很简单，定位target的父元素，调用`remove()`方法：

```javascript
/* static/js/chat.js */
document.addEventListener("click", evt => {
    var socket = io.connect(location.protocol + "//" + document.domain + ":" + location.port);
    const tgt = evt.target;
    if (tgt.dataset.class === 'del'){
        const elem = tgt.parentElement.parentElement;
        elem.remove();
        socket.emit("del msg", {'channel': act_channel, 'id': tgt.dataset.id});
    };
});
```

代码先通过parentElement定位到删除按钮所对应的这条消息，从页面中删除。同时，`socket.emit`一个字典`{'channel': xxx, 'id': xxx}`给服务器，告诉它要删除的消息是哪个频道、id是多少。

服务器端收到"del msg"请求后，收到的data就是个长度为2的字典。这样处理：

```python
# application.py
@socketio.on("del msg")
def del_msg(data):
    channel = urllib.parse.unquote(data['channel'])
    chats = channels[channel]['chats']
    app.logger.info(str(chats) + "\nDel: " + str(data))
    channels[channel]['chats'].pop(data['id'])
```

先解码channel（因为发过来前先进行了`encodeURI`），直接从channels对象的当前频道字典里，把键等于data['id']的字典整个pop掉。


### 其他

`flash`自动消失需要专门适配一些javascript。在main.js里，用jQuery实现。

```javascript
/* static/js/main.js */
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
