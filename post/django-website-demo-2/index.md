
### Admin后台

模型做好后，可以在[admin.py](https://github.com/madlogos/edx_cs50/blob/master/project3/orders/admin.py)里注册一下。这样就能像User类一样，在Django后台管理界面维护这些数据。

```python
from django.contrib import admin
from .models import Category, Product, Topping, Addition, Order, Item

# Register your models here.
class CategoryAdmin(admin.ModelAdmin):
    list_display = ['name', ]

admin.site.register(Category, CategoryAdmin)

class ProductAdmin(admin.ModelAdmin):
    list_display = ['id', 'name', 'category', 'size', 'price', 'n_topping', 
                    'n_addition', 'created', 'updated']
    list_filter = ['category', 'size', 'created', 'updated']
    list_editable = ['price', 'size', 'n_topping', 'n_addition', ]

admin.site.register(Product, ProductAdmin)

class ToppingAdmin(admin.ModelAdmin):
    list_display = ['name', 'price']
    list_editable = ['price', ]

admin.site.register(Topping, ToppingAdmin)

class AdditionAdmin(admin.ModelAdmin):
    list_display = ['name', 'size', 'price']
    list_filter = ['size',]
    list_editable = ['size', 'price', ]

admin.site.register(Addition, AdditionAdmin)

class OrderItemInline(admin.TabularInline):
    model = Order.item.through
    readonly_fields = ['item', 'quantity',]
    extra = 0

class OrderAdmin(admin.ModelAdmin):
    list_display = ['id', 'user', 'quantity', 'price', 'status', 'created', 'updated']
    list_editable = ['status', ]
    readonly_fields = ['user', 'quantity', 'price', ]
    inlines = (OrderItemInline, )

admin.site.register(Order, OrderAdmin)
```

<!--more-->

比如从admin.ModelAdmin继承一个ProductAdmin类出来，定义好list_display、list_filter和list_editable之类属性，再用`admin.site.register`把Product类注册到ProductAdmin上，admin页面就能管理Product模型了。非常方便。

值得多说一句的是多对多关系。比如OrderItem，需要专门从admin.TabularInline类继承出一个子类OrderItemInline来，绑定OrderItem模型。再在OrderAdmin中加个字段inlines，将OrderItemInline注册上去。这样就构建好了级联表UI视图，能通过Order来管理每个Order下面的Item了。


### 控制

接下来在[urls.py](https://github.com/madlogos/edx_cs50/blob/master/project3/orders/urls.py)里添加路由。

```python
urlpatterns = [
    path("menu", views.index, name="menu"),
    path("pick/<int:id>", views.pick_product, name="pick_product"),
    path("cart", views.cart, name="cart"),
    path("orders", views.orders, name="orders"),
    path("order/<int:id>", views.order, name="order"),
]
```

菜单、点菜、购物车、订单列表、订单明细五个路由都被定义好了。

### 视图

开始在[views.py](https://github.com/madlogos/edx_cs50/blob/master/project3/orders/views.py)里逐个写视图。

要记得把前面定义好的models都导进来。另外在[udf.py](https://github.com/madlogos/edx_cs50/blob/master/project3/orders/udf.py)里写了几个自定义函数(UDF)，也一并从导进来：

- `clean_from_data()`: 用正则表达式清理表单键值对，形成比较规整的{id: value}形式的字典；
- `to_num()`: 把数值型的文本转化为数值，不成功就转为None（为了`num()`和`parseInt()`抛Error退出）
- `show_cart()`: 显示某用户的购物车，返回{'items': items, 'price': price}字典
- `show_item()`: 显示某个Item对象的结构，包括product_id和内含的topping和addition对象字典

#### 菜单

##### 后端

菜单页是事实上的首页。后端代码并不复杂，主要用来跳转。

```python
@login_required
def index(request):
    if request.method == "POST":
        return HttpResponse({'message': ['success', 'Cart item added.']})
    else:
        products = Product.objects.all()
        return render(request, "orders/index.html", {"products": products})
```

{{% admonition tip "要点" %}}
在orders应用里，所有视图函数前都加了修饰器`@login_requied`，这就很便捷地完成了用户校验。
{{% /admonition %}}

用GET方法时，取出所有Product对象，返回给前端处理。用POST方法时（点了<kbd>Add to cart</kbd>按钮），发一个Http响应。但事实上不起什么作用，这个按钮绑了onclick事件，将触发点单页面。

[orders/index.html](https://github.com/madlogos/edx_cs50/blob/master/project3/templates/orders/index.html)模板定义了两摊东西：导航菜单和产品列表。这里用到Django的一个模板特性`regroup`，按category分组后再循环输出，就能实现分组显示了。

##### 前端

本质上是想实现一个按pivot_by size的交叉表，完全通过前端模板实现有点别扭。必要的话还是应该在后端加工好再往前端传。

<!-- {% raw %} -->
```html
{% extends "_base.html" %}
{% load static %}

{% block title %}
Menu
{% endblock %}

{% block script %}
<script src="{% static 'js/menu.js' %}"></script>
{% endblock %}

{% block nav_list %}
<li class="active"><a href="#">Menu</a></li>
<li><a href="cart">Shopping chart</a></li>
<li><a href="orders">Orders</a></li>
{% endblock %}

{% block disp %}
{% regroup products by category as prod_lists %}
{% for prods in prod_lists %}
<h3>{{ prods.grouper }}</h3>
<table class="table table-striped table-hover table-responsive" cellspacing="0">
    <thead>
        <th width="60%">Product</th>
        <th width="20%">Small</th>
        <th width="20%">Large</th>
    </thead>
    <tbody>
    {% regroup prods.list by name as prodset %}
    {% for prod in prodset %}
    <tr>
        <td width="60%">{{ prod.grouper }}</td>
        {% if prod.list.0.size != 'Small' %}
        <td width="20%"></td>
        {% endif %}

        {% for item in prod.list %}
        <td width="20%">
        ${{ item.price }}&emsp;
        <button class="btn btn-sm btn-primary" id="prod_{{ item.id }}" onclick="popupfun('{{ item.id }}')">Add to Cart</button>
        </td>
        {% endfor %}
    </tr>
    {% endfor %}
    </tbody>
</table>
{% endfor %}
{% endblock %}
```
<!-- {% endraw %} -->

<kbd>Add to cart</kbd>按钮绑定的事件叫`popupfun()`，写在[menu.js](https://github.com/madlogos/edx_cs50/blob/master/project3/static/js/main.js)里：

```javascript
function popupfun(prod_id){
    window.open("pick/" + prod_id, "Add to cart", "status=1,height:500,width:300,toolbar=0,resizeable=0");
    return false;
}
```

也就是弹一个点单页面出来，确认产品数量。这么做的原因是，需要在专门的页面里指定添加品（Topping和Addition）的品种和数量，放在同一个页面里写有点麻烦。

#### 点单

##### 前端

点单页面从[pick_product.html](https://github.com/madlogos/edx_cs50/blob/master/project3/templates/orders/pick_product.html)页面模板讲起。

它继承的是_popup.html模板，根据后端传入的product对象的n_topping数量判断，是否要加载topping和addition列表。一旦加载，就显示这些附加物，和数量录入框。这些表单是动态生成的，所以我没有用forms模板的方法来实现。（不过StackOverflow上查了查，还是有实现方案的。）

<!-- {% raw %} -->
```html
{% extends "_popup.html" %}
{% load static %}

{% block title %}
Topping / Additions
{% endblock %}

{% block script %}
<script src="{% static 'js/pick_product.js' %}"></script>
{% endblock %}

{% block disp %}
<h2>{{product.category}} - {{ product.name }} ({{ product.size }})</h2>

<form id="prod_form" name="prod_form" action="{% url 'pick_product' id=product.id %}" method="post" onsubmit="return check_toppings()">
{% csrf_token %}
<label>
    Unit price: <span style="font-size:x-large">$</span><span name="price" id="price" style="font-size:x-large">{{ product.price }}</span>
</label>
<br />
<label>Servings:
    <input name="qty" id="qty" type="number" style="text-align:center;width:50%" min="0" value="1" />
</label>
<button id="btn_submit" type="submit" class="btn btn-md btn-primary">Confirm</button>
<hr>
{% if product.n_topping > 0 %}
<table class="table table-striped table-hover table-responsive" cellspacing="0">
    <caption>You need to choose <span id="n_topping">{{ product.n_topping }}</span> pizza topping(s)</caption>
    <thead>
        <th>Topping</th>
        <th>Price</th>
        <th>Quantity</th>
    </thead>
    <tbody>
        {% for topping in toppings %}
        <tr>
            <td>{{ topping.name }}</td>
            <td id="topping_price_{{ topping.id }}">{{ topping.price }}</td>
            <td>
                <input name="topping_{{ topping.id }}" type="number" style="text-align:center;width:50%"
                 data-class="topping" data-item="{{ topping.id }}" min="0" max="{{ product.n_topping }}" />
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% endif %}

{% if product.n_addition > 0 %}
<table class="table table-striped table-hover table-responsive" cellspacing="0">
    <caption>Feel free to choose addition(s) for subs</caption>
    <thead>
        <th>Additions</th>
        <th>Price</th>
        <th>Quantity</th>
    </thead>
    <tbody>
        {% for addition in additions %}
        <tr>
            <td>{{ addition.name }}</td>
            <td id="addition_price_{{ addition.id }}">{{ addition.price }}</td>
            <td>
                <input name="addition_{{ addition.id }}" type="number" style="text-align:center;width:50%"
                 data-class="addition" data-item="{{ addition.id }}" min="0" />
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% endif %}

</form>
{% endblock %}
```
<!-- {% endraw %} -->

表单绑定了事件`onsubmit()`，所以点击<kbd>Confirm</kbd>按钮提交表单后，[pick_product.js](https://github.com/madlogos/edx_cs50/blob/master/project3/static/js/pick_product.js)里的`check_toppings()`函数会先运行。如果topping数量之和与limit不符，就返回false。

```javascript
function check_toppings(limit){
    var qty = 0;
    if (limit == null){
        return true;
    };
    document.querySelectorAll('input').forEach(elem => {
        if (elem.dataset.class == 'topping'){
            qty += Number(elem.value);
        };
    });
    return qty == limit;
};
```

整个JS脚本在载入点单页面后就开始监听。一旦点了提交按钮就校验数量。一旦改变了文本输入框的值，就调用`udpate_price()`更新页面小计。

这里用到了`evt.target.dataset.class`属性。我比较喜欢在前端模板的控件里埋`data-xxxx`属性，比如`data-class`、`data-value`，这样JS只要用`dataset.xxx`就能引用它。也不知道这是不是好的做法。

```javascript
document.addEventListener('DOMContentLoaded', () => {
    const submit = document.querySelector('#btn_submit');
    const n_topping = document.getElementById('n_topping');
    const qty_limit = (n_topping == null) ? null : Number(n_topping.innerHTML);

    submit.addEventListener('click', evt => {
        if (! check_toppings(qty_limit)){
            evt.preventDefault();
            alert("Please select exactly " + qty_limit + " toppings.");
        }else{
            document.forms['prod_form'].submit();
        };
    });

    document.addEventListener('change', evt => {
        if (evt.target.dataset.class == 'topping' || evt.target.dataset.class == 'addition'){
            update_price();
        };
    });
});

function update_price(){
    var price = Number(document.getElementById("price").innerHTML);
    document.querySelectorAll('input').forEach(elem => {
        if (elem.dataset.class == 'topping' || elem.dataset.class == 'addition'){
            let add_price = document.getElementById(elem.dataset.class + "_price_" + elem.dataset.item).innerHTML;
            price += Number(elem.value) * Number(add_price);
        };
    });
    document.getElementById("price").innerHTML = price.toFixed(2);
}; 
```

##### 后端

通过了JS脚本的校验后，添加物品种和数量提交到后端，python要进一步做一系列处理。

```python
@login_required
def pick_product(request, id):
    if request.method == "POST":
        qty = to_num(request.POST['qty'])
        if qty is None or qty == 0:
            return HttpResponse("<script>window.close();</script>")

        toppings = clean_form_data(request.POST)
        n_topping = Product.objects.get(id=id).n_topping
        if sum(toppings.values()) != n_topping:
            return HttpResponse("<script>alert('This product should have " + str(n_topping) + " topping\(s\).')</script>")
        additions = clean_form_data(request.POST, {'ptn': r'addition_(\d+)$', 'rpl': r'\1'})

        # prepare cart
        try:
            cart = Cart.objects.get(user=request.user)
        except Cart.DoesNotExist:
            cart = Cart.objects.create(user=request.user)

        product = Product.objects.get(pk=id)
        item = Item(product=product, quantity=qty, price=product.price)
        
        # judge if duplicated
        itm_dup = False
        itm_trk = {'product': item.product.id, 'price': item.price, 'topping': dict(), 'addition': dict()}
        if len(toppings) > 0:
            for key in toppings:
                topping = Topping.objects.get(id=to_num(key))
                itm_trk['topping'][key] = toppings[key]
                itm_trk['price'] += topping.price * toppings[key]
        if len(additions) > 0:
            for key in additions:
                addition = Addition.objects.get(id=to_num(key))
                itm_trk['addition'][key] = additions[key]
                itm_trk['price'] += addition.price * additions[key]

        ## if duplicate, add 1
        for itm in cart.cartitem_set.all():
            itm_str = show_item(itm.item)
            if itm_trk == itm_str:
                itm_dup = True
                itm.quantity += qty
                itm.save()
                break
        
        ## if not duplicate, insert record
        if not itm_dup:
            item.save()
            if len(toppings) > 0:
                for key in toppings:
                    topping = Topping.objects.get(id=to_num(key))
                    ItemTopping.objects.create(item=item, topping=topping, quantity=toppings[key])
                    item.price += topping.price * toppings[key]
            if len(additions) > 0:
                for key in additions:
                    addition = Addition.objects.get(id=to_num(key))
                    ItemAddition.objects.create(item=item, addition=addition, quantity=additions[key])
                    item.price += addition.price * additions[key]
            item.save()
            CartItem.objects.create(cart=cart, item=item, quantity=qty)

        return redirect(reverse('cart'))
    else:
        products = Product.objects.get(id=id)
        toppings = Topping.objects.all()
        additions = Addition.objects.filter(size=products.size)
        return render(request, "orders/pick_product.html",
            {"product": products, "toppings": toppings, "additions": additions})
```

如为POST方法（提交表单），还是先校验topping和addition数量是否符合产品定义。万一JS没起效，后端还是拦一道。把topping和addition解析为字典格式，再获取/创建一个购物车对象cart。做好准备工作。

接下来判断当前的item是否已经和购物车里重复。所谓重复项目，就是product、topping、addition、price都相同，如果重复，自然就不需要创建一个新的购物车条目了，只要quantity +1就行。而如果不重复，就要创建item，包括item本身，以及ItemTopping和ItemAddition。全部完成后，转跳到购物车，就能看到结果。

隐隐觉得这么实现比较笨拙。不过目前水平有限，就不追求优雅了。

如为GET方法，筛选出product、topping和addition就可以，交给前端渲染。

#### 购物车

##### 前端

购物车[cart.html](https://github.com/madlogos/edx_cs50/blob/master/project3/templates/orders/cart.html)模板主要完成三个功能：展示当前购物车、修改项目数量、筛选和下单。

这一部分非常考验对ORM模型的理解。传入的对象是items，循环遍历每一个item，那么找到其下挂的topping，就要用item.item.itemtopping_item.all，其中"itemtopping_item"就是定义模型时起的related_name。

<!-- {% raw %} -->
```html
{% extends "_base.html" %}
{% load static %}

{% block title %}
Cart
{% endblock %}

{% block script %}
<script src="{% static 'js/cart.js' %}"></script>
{% endblock %}

{% block nav_list %}
<li><a href="menu">Menu</a></li>
<li class="active"><a href="#">Shopping chart</a></li>
<li><a href="orders">Orders</a></li>
{% endblock %}

{% block disp %}
<form id="place_order" action="{% url 'cart' %}" method="post">
{% csrf_token %}
{% if items|length > 0 %}
    <table class="table table-striped table-hover table-responsive" cellspacing="0">
        <thead>
            <th width="50%" colspan="2"><input type="checkbox" name="check_all" id="check_all"><span>Cart Item</span></input></th>
            <th width="10%">Servings</th>
            <th width="10%">Unit Price</th>
            <th width="15%">Created</th>
            <th width="15%">Updated</th>
        </thead>
        <tbody>
            {% for item in items %}
            <tr>
                <td width="50%" colspan="2">
                    <input type="checkbox" name="order_product_{{ item.item.id }}" data-class="order" data-item="{{ item.item.id }}" value="{{ item.item.quantity }}">
                        <span data-toggle="tooltip" title="${{ item.item.product.price }}/serving">
                            <strong>{{ item.item.product.category }}</strong> - {{ item.item.product.name}} ({{ item.item.product.size }})
                        </span>
                    </input>
                </td>
                <td width="10%"><input id="product_{{ item.item.id }}" name="product_{{ item.item.id }}" data-class="qty" data-item="{{ item.item.id }}" 
                    type="number" min="0" style="width:50%" value="{{ item.quantity }}" /></td>
                <td width="10%" id="product_price_{{ item.item.id }}" name="product_price_{{ item.item.id }}" data-class="price" data-item="{{ item.item.id }}">
                    {{ item.item.price }}
                </td>
                <td width="15%">{{ item.item.created|date:"Y/m/d H:i:s" }}</td>
                <td width="15%">{{ item.item.updated|date:"Y/m/d H:i:s" }}</td>
            </tr>
            
            {% if item.item.topping.all|length > 0 %}
                {% for topping in item.item.itemtopping_item.all %}
                <tr>
                    <td width="10%"></td>
                    <td width="40%">
                        <span data-toggle="tooltip" title="${{ topping.topping.price }}/serving">
                            <span style="color:#bbb">+Topping:</span> {{ topping.topping.name }} x {{ topping.quantity }}
                        </span>
                    </td>
                    <td width="10%"></td>
                    <td width="10%"></td>
                    <td width="15%"></td>
                    <td width="15%"></td>
                </tr>
                {% endfor %}
            {% endif %}
            {% if item.item.addition.all|length > 0 %}
                {% for addition in item.item.itemaddition_item.all %}
                <tr>
                    <td width="10%"></td>
                    <td width="40%">
                        <span data-toggle="tooltip" title="${{ addition.addition.price }}/serving">
                            <span style="color:#bbb">+Addition:</span> {{ addition.addition.name }} ({{ addition.addition.size }}) x {{ addition.quantity }}
                        </span>
                    </td>
                    <td width="10%"></td>
                    <td width="10%"></td>
                    <td width="15%"></td>
                    <td width="15%"></td>
                </tr>
                {% endfor %}
            {% endif %}

            {% endfor %}
        </tbody>
    </table>
{% else %}
<p>No items found.</p>
{% endif %}

<hr>
<div class="container">
    <div class="row">
        <div class="col-lg-9 col-md-8"></div>
        <div class="col-lg-1 col-md-1 col-sm-4">
            <h4>Selected: </h4>
        </div>
        <div class="col-lg-2 col-md-3 col-sm-8">
            <h4 style="text-align:right;font-weight:bold">$<span id="select_price">{{ select_sum }}</span></h4>
        </div>
    </div>
    <div class="row">
        <div class="col-lg-9 col-md-8"></div>
        <div class="col-lg-1 col-md-1 col-sm-4">
            <h4>Total: </h4>
        </div>
        <div class="col-lg-2 col-md-3 col-sm-8">
            <h4 style="text-align:right;font-weight:bold">$<span id="total_price">{{ cart_sum }}</span></h4>
        </div>
    </div>
    <div class="row">
        <div class="col-lg-9 col-md-8"></div>
        <div class="col-lg-3 col-md-4 col-sm-12">
            <button id="btn_save" name="btn_save" type="submit" class="btn btn-secondary btn-block">Save my cart</button>
        </div>
    </div>
    <div class="col-12">&nbsp;</div>
    <div class="row">
        <div class="col-lg-9 col-md-8"></div>
        <div class="col-lg-3 col-md-4 col-sm-12">
            <button id="btn_submit" name="btn_submit" type="submit" class="btn btn-primary btn-block">Place an order</button>
        </div>
    </div>
</div>

</form>
{% endblock disp %}
```
<!-- {% endraw %} -->

这里只允许修改product的数量，product下挂的topping、addition就不让改了（有逻辑冲突，太麻烦）。每当修改数量，小计部分就自动更新。这需要用到[cart.js](https://github.com/madlogos/edx_cs50/blob/master/project3/static/js/cart.js)。

页面里写了tooltip value，所以JS中用jQuery定义一下tooltip，鼠标移到上方时显示数值。

```javascript
$(document).ready(function(){
  $('[data-toggle="tooltip"]').tooltip();
});
```

对页面进行持续监听，一旦文本输入框或复选框选中状态发生改动，就调用`update_price()`更改小计价格。如果一个都没选中，就`preventDefault()`，不许提交。如果点了"#check_all"复选框，那么就`batch_check()`选中所有项目。

```javascript
document.addEventListener('DOMContentLoaded', () => {
    update_price();
    document.addEventListener('change', evt => {
        if (evt.target.id == 'check_all'){
            batch_check(evt.target.checked);
            update_price();
        };
        if (evt.target.dataset.class == 'qty' || evt.target.dataset.class == 'order'){
            update_price();
        };
    });
    document.getElementById('btn_submit').addEventListener('click', evt => {
        let select = document.getElementById('select_price').innerHTML;
        if (Number(select) == 0){
            evt.preventDefault();
            alert("Please select at least 1 item."); 
        };
    });
});

function update_price(){
    var price = 0;
    var selected = 0;
    document.querySelectorAll('input').forEach(elem => {
        if (elem.dataset.class == 'qty' || elem.dataset.class == 'order'){
            let item_id = elem.dataset.item;
            let unit_price = document.getElementById("product_price_" + item_id).innerHTML;
            let qty = document.getElementById('product_' + item_id).value;
            let item_price = Number(qty) * Number(unit_price);
            
            if (elem.dataset.class == 'order'){
                if (elem.checked){
                    selected += item_price;
                };
            }else{
                price += item_price;
            };
        };
    });
    document.getElementById("total_price").innerHTML = price.toFixed(2);
    document.getElementById("select_price").innerHTML = selected.toFixed(2);
}; 

function batch_check(check=true){
    document.querySelectorAll('input').forEach(elem => {
        if (elem.dataset.class == 'order'){
            elem.checked = check
        };
    });
};
```

##### 后端

```python
@login_required
def cart(request):
    if request.method == "POST":
        # save the form
        cart = Cart.objects.get(user=request.user)
        items = clean_form_data(request.POST, {'ptn': r'product_(\d+)$', 'rpl': r'\1'}, del_val=())
        orders = clean_form_data(request.POST, {'ptn': r'order_product_(\d+)$', 'rpl': r'\1'})
        cart_items = cart.cartitem_set.all()

        for key in items:
            item = cart_items.get(item=Item.objects.get(pk=to_num(key)))
            if items[key] == 0:
                item.delete()
                Item.objects.filter(pk=to_num(key)).delete()
            else:
                if item.quantity != items[key]:
                    item.quantity = items[key]
                    item.updated = datetime.datetime.now()
                    item.save()
        cart.save()

        cart_det = show_cart(request.user)
        if 'btn_save' in request.POST:
            return render(request, "orders/cart.html", 
                {'items': cart_det['items'], 'message': ['success', 'Cart saved.'], 'cart_sum': cart_det['price']})
        elif 'btn_submit' in request.POST:
            if len(orders) > 0:
                order = Order.objects.create(user=request.user, price=0)
                for key in orders:
                    item = cart_items.get(item=Item.objects.get(pk=to_num(key)))
                    OrderItem.objects.create(order=order, quantity=item.quantity, item=Item.objects.get(pk=to_num(key)))
                    order.quantity += item.quantity
                    order.price += item.item.price * item.quantity
                    item.delete()
                order.save()

            return render(request, 'orders/orders.html', 
                {'orders': Order.objects.filter(user=request.user), 'message': None})

    else:
        cart_det = show_cart(request.user)
        return render(request, "orders/cart.html", 
            {'items': cart_det['items'], 'message': None, 'cart_sum': cart_det['price']})
```

如为POST方法（提交表单），首先保存当前购物车。这就涉及item、itemtopping、itemaddition三张表的改动。如果数量减到0，就直接删除。

接下来判断点的是哪个按钮。如果是submit，那么还要创建订单，并把选中项目的item从cart改关联到order，CartItem拷贝到OrderItem，并删除旧的CartItem（真麻烦）。

假如是GET方法，那么只要把购物车结构解析出来，传到前端渲染就行了。

#### 订单列表

##### 前端

订单列表[orders.html](https://github.com/madlogos/edx_cs50/blob/master/project3/templates/orders/orders.html)页面模板如下。跟购物车差不多，也提供了复选框和按钮，用户可以先选中对应的订单批处理操作，或直接点某个订单后的按钮进行单独操作。操作包括支付、取消、删除。利用Django模板的条件渲染能力，根据订单状态的不同，每个按钮的disabled属性各有不同。

<!-- {% raw %} -->
```html
{% extends "_base.html" %}
{% load static %}

{% block title %}
Orders
{% endblock %}

{% block script %}
<script src="{% static 'js/orders.js' %}"></script>
{% endblock %}

{% block nav_list %}
<li><a href="menu">Menu</a></li>
<li><a href="cart">Shopping chart</a></li>
<li class="active"><a href="#">Orders</a></li>
{% endblock %}

{% block disp %}
<form id="submit_orders" action="{% url 'orders' %}" method="post">
{% csrf_token %}

{% if orders|length > 0 %}
    <table class="table table-striped table-hover table-responsive" cellspacing="0">
        <thead>
            <tr>
                <th width="15%"><input type="checkbox" name="check_all" id="check_all">Order ID</input></th>
                <th width="10%">Items</th>
                <th width="15%">Price</th>
                <th width="10%">Status</th>
                <th width="15%">Created</th>
                <th width="15%">Updated</th>
                <th width="auto" class="text-right">
                    <button name="btn_pay" type="submit" id="btn_pay" class="btn btn-primary btn-sm">Pay</button>&nbsp;
                    <button name="btn_cancel" type="submit" id="btn_cancel" class="btn btn-warning btn-sm">Cancel</button>&nbsp;
                    <button name="btn_delete" type="submit" id="btn_delete" class="btn btn-danger btn-sm">Delete</button>
                </th>
            </tr>
        </thead>
        <tbody>
            {% for order in orders %}
            <tr>
                <th width="15%">
                    <input type="checkbox" name="order_{{ order.id }}" data-class="order" data-item="{{ order.id }}" value="{{ order.price }}">
                        <a href="{% url 'order' id=order.id %}"># {{ order.id }}</a>
                    </input>
                </th>
                <td width="10%" id="qty_{{ order.id }}">{{ order.quantity }}</td>
                <td width="15%">$<span id="price_{{ order.id }}">{{ order.price }}</span></td>
                <td width="10%" id="status_{{ order.id }}">
                    <span class=
                        {% if order.status == 'Paid' %}"text-success"
                        {% elif order.status == 'Completed' %}"text-info"
                        {% elif order.status == 'Failed' %}"text-danger"
                        {% elif order.status == 'Cancelled' %}"text-muted"
                        {% else %}"text-primary"
                        {% endif %}>{{ order.status }}
                    </span>
                </td>
                <td width="15%">
                    {{ order.created|date:"Y/m/d H:i:s" }}
                </td>
                <td width="15%">
                    {{ order.updated|date:"Y/m/d H:i:s" }}
                </td>
                <td width="auto" class="text-right">
                    <button name="pay" value="{{ order.id }}" class="btn btn-primary btn-sm"
                     {% if order.status == 'Paid' or order.status == 'Completed' or order.status == 'Cancelled' %}
                      disabled="disabled" 
                     {% endif %}>Pay</button>&nbsp;
                    <button name="cancel" value="{{ order.id }}" class="btn btn-warning btn-sm"
                     {% if order.status == 'Paid' or order.status == 'Cancelled' %}
                      disabled="disabled" 
                     {% endif %}>Cancel</button>&nbsp;
                    <button name="delete" value="{{ order.id }}" class="btn btn-danger btn-sm"
                     {% if order.status == 'Paid' or order.status == 'Completed' %}
                      disabled="disabled" 
                     {% endif %}>Delete</button>
                </td>
            </tr>
            {% endfor %}
        </tbody>
    </table>
{% else %}
<p>No items found.</p>
{% endif %}

<hr>
<div class="container">
    <div class="row">
        <div class="col-lg-9 col-md-8"></div>
        <div class="col-lg-1 col-md-1 col-sm-4">
            <h4>Selected: </h4>
        </div>
        <div class="col-lg-2 col-md-3 col-sm-8">
            <h4 style="text-align:right;font-weight:bold">$<span id="select_price">{{ select_sum }}</span></h4>
        </div>
    </div>
    <div class="row">
        <div class="col-lg-9 col-md-8"></div>
        <div class="col-lg-1 col-md-1 col-sm-4">
            <h4>Total: </h4>
        </div>
        <div class="col-lg-2 col-md-3 col-sm-8">
            <h4 style="text-align:right;font-weight:bold">$<span id="total_price">{{ cart_sum }}</span></h4>
        </div>
    </div>
</div>
</form>
{% endblock disp %}
```
<!-- {% endraw %} -->

前端脚本[orders.js](https://github.com/madlogos/edx_cs50/blob/master/project3/static/js/orders.js)和购物车页面的脚本一样，持续监听，实现批量选中和随时更新小计价格。有所不同的是，还加了一段监听代码，一旦批量选中后点操作按钮，先要`check_clickable()`校验一下这个按钮是否能点，然后调`confirm_submit()`跳出一个弹窗，让用户确认操作。

```javascript
document.querySelectorAll('button').forEach(elem => {
	const btn_dict = {'btn_pay': 'pay', 'btn_cancel': 'cancel', 'btn_delete': 'delete'};
	if (['btn_pay', 'btn_cancel', 'btn_delete'].indexOf(elem.id) >= 0){
		elem.addEventListener('click', evt => {
			if (! check_clickable(btn_dict[elem.id])){
				evt.preventDefault();
				alert('No order is selected or \r\nnot all the selected orders can ' + btn_dict[elem.id] + '.');
			};
		});
	}else if (['pay', 'cancel', 'delete'].indexOf(elem.name) >= 0){
		elem.addEventListener('click', confirm_submit);
	};
});

function check_clickable(state){
    /* check if the pay_all, cancel_all, delete_all buttons are clickable */
    var o = true;
    var any_checked = false;
    const dict = {'pay': ['Pending', 'Failed'], 
                  'cancel': ['Pending', 'Failed'],
                  'delete': ['Failed', 'Cancelled']};
    document.querySelectorAll('input').forEach(elem => {
        if (elem.dataset.class == 'order'){
            if (elem.checked){
                any_checked = true;
                let item_id = elem.dataset.item;
                let item_status = document.getElementById('status_' + item_id).innerHTML;
                if (! item_status  in dict[state]){
                    o = false;
                    return false;
                };
            };
        };
    });
    if (! any_checked){
        o = false;
    };
    return o;
};

function confirm_submit(evt){
    var win = window.confirm("Confirm with the operation?");
    if (! win) {
        evt.preventDefault();
    };
};
```

##### 后端

```python
@login_required
def orders(request):
    if request.method == "POST":
        if any(_ in request.POST for _ in ('btn_pay', 'btn_cancel', 'btn_delete')):
            orders = clean_form_data(request.POST, {'ptn': r'order_(\d+)$', 'rpl': r'\1'})
            if 'btn_pay' in request.POST:
                raise Http404('Payment function is not available now.')
            else:
                for key in orders:
                    order = Order.objects.get(pk=to_num(key))
                    if 'btn_delete' in request.POST:
                        order_items = order.orderitem_set.all()
                        for itm in order_items:
                            Item.object.get(pk=itm.item.id).delete()
                        order.delete()
                    elif 'btn_cancel' in request.POST:
                        order.status = 'Cancelled'
                        order.updated = datetime.datetime.now()
                        order.save()

        btn = [_ for _ in ('pay', 'cancel', 'delete') if _ in request.POST]
        if len(btn) > 0:
            order_id = request.POST[btn[0]]
            if 'pay' in request.POST:
                raise Http404('Payment function is not available now.')
            else:
                order = Order.objects.get(pk=to_num(order_id))
                if 'delete' in request.POST:
                    order_items = order.orderitem_set.all()
                    for itm in order_items:
                        Item.objects.get(pk=itm.item.id).delete()
                    order.delete()
                elif 'cancel' in request.POST:
                    order.status = 'Cancelled'
                    order.updated = datetime.datetime.now()
                    order.save()
        return render(request, "orders/orders.html",
            {'orders': Order.objects.filter(user=request.user),
             'message': ['success', 'Orders modified.']})
    else:
        orders = Order.objects.filter(user=request.user)
        return render(request, "orders/orders.html",
            {'orders': orders})
```

当按的是<kbd>Pay</kbd>支付按钮时，返回一个404错误（因为支付功能没做）。其他情况下，就批量更改订单状态。

#### 订单明细

##### 前端

订单明细[order.html](https://github.com/madlogos/edx_cs50/blob/master/project3/templates/orders/order.html)显示的是订单详情。这个页面上用户依然可以更改各项目的数量。

页面脚本[order.js](https://github.com/madlogos/edx_cs50/blob/master/project3/static/js/order.js)跟订单列表页脚本差不多，监听页面，动态更新小计，点按钮的话确认操作再放行提交。

##### 后端

其实可以复用`orders()`的代码。这块没有认真重构。

```python
@login_required
def order(request, id):
    if request.method == "POST":
        if any(_ in request.POST for _ in ('btn_save', 'btn_pay', 'btn_cancel', 'btn_delete')):
            order = Order.objects.get(pk=id)
            items = clean_form_data(request.POST, {'ptn': r'product_(\d+)$', 'rpl': r'\1'}, del_val=('0', ''))
            order_items = order.orderitem_set.all()
            order.quantity = order.price = 0
            for key in items:
                item = Item.objects.get(pk=to_num(key))
                order_item = order_items.get(item=item)
                if items[key] == 0:
                    order_item.delete()
                    item.delete()
                else:
                    if order_item.quantity != items[key]:
                        order_item.quantity = items[key]
                        order_item.updated = order.updated = datetime.datetime.now()
                        order_item.save()
                    order.quantity += items[key]
                    order.price += item.price * items[key]
            order.save()

            if 'btn_pay' in request.POST:
                raise Http404('Payment function is not available now.')
            elif 'btn_delete' in request.POST:
                order.delete()
                return redirect(reverse('orders'))
            elif 'btn_cancel' in request.POST:
                order.status = 'Cancelled'
                order.save()

            return render(request, 'orders/order.html',
                {'items': order.orderitem_set.all(), 'order': order, 'id': id,
                 'message': ['success', 'Modification saved.']})

    else:
        order = Order.objects.get(pk = id)
        return render(request, "orders/order.html", {'items': order.orderitem_set.all(), 'order': order})
```

#### 其他

[main.js](https://github.com/madlogos/edx_cs50/blob/master/project3/static/js/main.js)和[style.css](https://github.com/madlogos/edx_cs50/blob/master/project3/static/css/style.css)主要还是一些全局的基本配置。不展开了。

差不都就是这样。

[完]

---

<!-- {% raw %} -->
{{% figure src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/QRcode.jpg" width="50%" title="扫码关注我的的我的公众号" alt="扫码关注" %}}
<!-- {% endraw %} -->