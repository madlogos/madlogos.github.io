
## 离题万里的开场

码农界有个著名的“三次法则 (rule of three)”：**如果一段代码重复出现了三次，就要考虑抽出来写一个子程序，以便复用**。这是条宝贵的法则，可以衍生出更多的强迫症版本，随随便便就能举出很多喜闻乐见的例子：

{{% admonition example 比如 %}}

- 一个词在同一句话里出现三次就不能忍，必须换近义词；
- 一件事手动做三次就不能忍，必须写程序自动化；
- 一顿饭重复吃三口就不能忍，必须开发一个喂饭机；
- 同一处空气重复呼吸三口就不能忍，必须装一台呼吸机

……
{{% /admonition %}}

发人深省，对不对？这些正是当今最严肃而真实的信仰，有着最为坚定的践行者。~~在古代~~2015年全球最大的雄性交友平台GitHub上出了个网红毛子码农、脚本狂魔[Narkoz](https://github.com/NARKOZ/hacker-scripts)，他的人生信条是：**如果一件事要耗费自己90秒以上，那就写个脚本**。这些奇葩的脚本包括：

{{% admonition example 脚本 %}}

- 如加班到21点以后就自动给老婆发马屁短信；
- 收到蠢货DBA的任何求助邮件后自动恢复数据库的最近备份
- 让咖啡机等待17秒然后煮杯咖啡并等待24秒再灌入杯子（正好是作者起身走到咖啡机前的耗时）
- ……

{{% /admonition %}}

从时间效益的经济学评价来讲，这个准则烂透了。这好比为了节约每天通勤的公交车钱，去买了一辆跑车。但跑车本身还是很拉风的。若能竖立起极客死宅的品牌形象，还是可能会产生某些**潜在的**溢价——比如说，会有更多的人请你修电脑。

判断一个人是否天生适合当码农，很重要的一点就是看他/她有没有这种懒癌强迫症。而这种强迫症的形上学本质是：**对自由王国的无尽向往**。——重复劳动太蠢了，它侮辱人类的尊严，阻碍我们证法悟道，必须消灭。这就是自动化的诞生。

所以——尽管技术上跌跌撞撞，也并不妨碍我隔三差五搞个三脚猫爬虫出来。这回的主题又跳跃了：爬取现代化支付行号。

## 这就带来了第一个问题

{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2017/1128/yilianmengbi.jpg" title="蛤？这是啥？" %}}

<!--more-->

{{% admonition info 定义 %}}
所谓现代化支付行号 (CNAPS)   

也叫联行号。
{{% /admonition %}}

就是中国人民银行搞的一套12位银行代码，用来做自动清算的。在通过手机银行app转账时，选择收款方账户时，也会看到这个代码。它的结构如下：

```text
** *    ****    ****           *
行别码  地区码  分支机构序号   校验码
```

所以如果要搭一个跟支付清算相关的系统，就很有必要把CNAPS作为基建纳入考虑。这套代码并不公开，但是公开渠道仍能从一些银行的官网查到。比如[河北银行](https://www.hebbank.com/corporbank/otherBankQueryWeb.do)、[浙商银行](https://corbank.czbank.com/CORPORBANK/query_unionBank_index.jsp)。

<!-- {% raw %} -->
{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2017/1128/hebbank.png" title="图 | 河北银行CNAPS查询页" alt="河北银行" %}}

{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2017/1128/zsbank.png" title="图 | 浙商银行CNAPS查询页" alt="浙商银行" %}}
<!-- {% endraw %} -->

随手上去试了两把，还挺好用。于是思路比较清楚了：穷尽所有查询策略，把返回的结果提出来存好。

## 怎么爬咧？

### 首先，选型要精准

上面提到的这两家都要输校验码，攻起来有门槛。所以退而求其次，发现一家[中原银行](http://www.zybank.com.cn/zyb/zh_CN/jshj/lhhquery.html)。

<!-- {% raw %} -->
{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2017/1128/zybank.png" title="图 | 中原银行CNPAS查询页" alt="中原银行" %}}
<!-- {% endraw %} -->

这就比较好对付一些，而且信息更多，连网点地址也提供。

### 其次，办法要对路

- [x] 要理智。当然不能手工复制粘贴。平均查一个结果需要点行名、省、市，输验证码、按确认，超过10次交互动作。几千次这么做下来，非死亦残。更不要提错误率了。<kbd>放弃</kbd>
- [x] 我比较熟悉的`rvest`这类简单的R爬虫包，只能对付静态网页，无法模拟网页交互行为。<kbd>放弃</kbd>
- [x] Python的`scrapy`框架很牛，也能对付验证码、AJAX异步之类问题。次一点，R里面也有`Rcurl`。但我都不会。<kbd>放弃</kbd>
- [x] 要模拟用户交互行为最好的办法是`Selenium`这类框架(代码版按键精灵)，但是当时我还不会。<kbd>放弃</kbd>。
- [x] 其他。<kbd>不明</kbd>

所谓“夫未战而庙算者，得算多也”。经过一番严谨的分析，我发现：**技术上搞不定**——再次落入了经典的“看得上的买不起，买得起的看不上”窘境。

### 曲线救国

办法还是有的，但就是要先停下来，离题万里去讲另一个故事。

在OA领域，我们经常会把一系列小操作对应的系统指令录下来，套个循环再复用，那就是VBA宏语言。所谓宏，实质上就是定义一组模式替换规则，套用到一组命令上进行**批量批处理**。这不就是“录制-修改-复用”自动化党的福音吗？看看我们平时用的最多的宏，通常就是VBA，SPSS，SAS，以及各种各样的游戏作弊器。那就是宏天生的战场。

对于本地的任务，其实最合适的工具是Selenium IDE (或Katalon之类替代品)，一样从录制宏开始。但我当时并不会。好在Firefox里有一个历史悠久的代替品：iMacros。看，名字里就有一个“宏”。它也有Chrome和IE的版本，通过浏览器扩展商店装好后，模样长这样：

<!-- {% raw %} -->
{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2017/1128/imacros.png" title="图 | iMacros for Chrome界面" alt="iMacros" %}}
<!-- {% endraw %} -->

如果自己录一段宏，打开后长这样：

<!-- {% raw %} -->
{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2017/1128/imacros_demo1.png" title="图 | iMacros脚本代码" alt="iMacros脚本" %}}
<!-- {% endraw %} -->

由于完全是一堆动作指令，所以很容易读懂。无非是关闭其他标签页，打开一个网址，依次在几个文本控件里填入内容，最后点按钮提交。

双击录好的iim脚本运行，就把刚才的录入工作重复执行一遍。就跟自己动手一毛一样。

<!-- {% raw %} -->
{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2017/1128/imacros_demo2.gif" title="图 | iMacros demo" alt="demo" %}}
<!-- {% endraw %} -->

想象一下你有几千条记录要在网上填入。用imacros，只要在脚本同目录下准备一个csv，整理好数据，然后修改一下上面的代码，读入csv，逐行扫描，按布局顺序提取数据`{{!COL1}}`、`{{!COL2}}`、...，填入对应的控件，提交。接下来边喝茶边看屏幕飞滚，繁复的录入任务就自行完成了。

## 节外生枝的JavaScript

按说imacros已经给了很完美的循环方案：在iim脚本里给内置变量{{!LOOP}}赋一个初始值，然后在执行面板里设置一个（{{!LOOP}}的）Max值，点<kbd>Play Loop</kbd>即可。但这只能对付单层循环。在中原银行这个案例上，单层循环是不够的。理论上需要三层循环：

```js
for i in 所有银行名:
    for j in 所有省份:
        for k in 所有城市:
            查询
            提取结果表格
            存入一个文件
        next k
    next j
next i
```

大约80个银行类目，34个省份，650个城市，这样完整运行一遍要有17.7万个组合。但实际上34个省份和650个城市不会完整组合，每个省份只可能匹配其属下的那几个城市。省份选择“江苏”后，硬要让脚本到城市下拉框找“杭州”，只能逼它报错。所以省份、城市的两层循环(j, k)实际上应当合并。

饶是如此，仍有两层循环。假如硬是压成单层，就得在csv里把所有组合罗列一遍，imacros表示无力。所幸这个工具还有不错的延展性，可以执行vbs和js脚本。查询、提取、存储之类还是可以用imacros宏来完成，外面搭一个js的循环壳调用`iimPlay`就行。伪代码看起来大约是这样：

```javascript
for (var i=0; i < 银行数; i++){
    for (var j=0; j < 省份和城市对数; j++){
        iimPlay(一段imacros宏，选好银行、省、市，查询结果)
        iimPlay(另一段imacros宏，返回查询结果的页数，以便逐页去点)
        iimPlay(另一段imacros宏，抽取银行、省、市的标签名，用来给文件命名)
        iimPlay(最后一段imacros宏，循环点查询结果页，把每一页结果存入csv)
    }
}
```

心里有底了。开工。

## 提取银行和省份+城市的控件值

Chrome里<kbd>F12</kbd>，可以看到银行下拉菜单的载入值齐齐整整。

<!-- {% raw %} -->
{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2017/1128/banksite.png" title="图 | 银行下拉菜单载入值" %}}
<!-- {% endraw %} -->

选中'banksite'这个节点，右键copy element复制下来。到R里，赋值给banks。

```html
<select name="select" class="select_170" id="banksite">
<option value="">==请选择银行==</option>
<option value="100">中原银行</option><option value="102">中国工商银行</option><option value="103">中国农业银行</option><option value="104">中国银行</option><option value="105">中国建设银行</option><option value="201">国家开发银行</option><option value="202"> ....
```

用R快速处理一下：

```r
banks <- unlist(banks, strsplit(banks, "</option>"))
paste(sapply(banks, function(v)
    gsub("^<option value=\"(\\d+)\">\\D+$", "\\1", v)), collapses=",")
```

这样直接就提取成了一个文本向量。如果在Windows，可以复制outerHTML，用`readClipboard`来直接读剪贴板。所以定义一个`getCbId`函数，直接处理成js列表。

```r
library(jsonlite)
getCbId <- function(grpId=NA){
  # Copy XML nodes from ZYB page by F12
  a <- readClipboard(format=13)
  a <- unlist(strsplit(a, "\""))
    a <- a[seq(2, length(a), 2)]
    if (is.na(grpId)){
        toJSON(as.numeric(a))
    }else{
        a <- matrix(as.numeric(c(rep.int(grpId, length(a)), a)), ncol=2)
        toJSON(a)
    }
}
```

所以这个scrapy_cnaps.js的起头部分就变成了：

```javascript
var banks=[100,102,103,104,105,201,202,203,301,302,303,304,305,306,307,308,309,
    310,313,314,315,316,317,318,319,401,402,403,501,502,503,504,505,506,507,509,
    510,512,531,532,533,561,562,563,564,565,591,593,594,595,596,597,621,622,623,
    631,641,651,652,661,662,671,672,691,692,693,695,711,712,713,714,715,716,732,
    741,742,751,752,761,771,775,776,781,782,783,785,787];
var nBanks=banks.length;
```

以此类推，把所有省份和城市的组合都找了出来。

```javascript
var cities=[
    [11,1000],
    [12,1100],
    [13,1210],[13,1240],[13,1260],[13,1270],[13,1310],[13,1340],[13,1380],
    [13,1410],[13,1430],[13,1460],[13,1480],
    [14,1610],[14,1620],[14,1630],[14,1640],[14,1680],[14,1690],[14,1710],
    [14,1730],[14,1750],[14,1770],[14,1810],
    [15,1910],[15,1920],[15,1930],[15,1940],[15,1960],[15,1980],[15,1990],
    [15,2010],[15,2030],[15,2050],[15,2070],[15,2080],
    [21,2210],[21,2220],[21,2230],[21,2240],[21,2250],[21,2260],[21,2270],
    [21,2280],[21,2290],[21,2310],[21,2320],[21,2330],[21,2340],
    [22,2410],[22,2420],[22,2430],[22,2440],[22,2450],[22,2460],[22,2470],
    [22,2490],[22,2520],
    [23,2610],[23,2640],[23,2650],[23,2660],[23,2670],[23,2680],[23,2690],
    [23,2710],[23,2720],[23,2740],[23,2760],[23,2780],[23,2790],
    [31,2900],
    [32,3010],[32,3020],[32,3030],[32,3040],[32,3050],[32,3060],[32,3070],
    [32,3080],[32,3090],[32,3110],[32,3120],[32,3140],
    [33,3310],[33,3320],[33,3330],[33,3350],[33,3360],[33,3370],[33,3380],
    [33,3410],[33,3420],[33,3430],[33,3450],
    [34,3610],[34,3620],[34,3630],[34,3640],[34,3650],[34,3660],[34,3670],
    [34,3680],[34,3710],[34,3720],[34,3740],[34,3750],[34,3760],[34,3790],
    [35,3910],[35,3930],[35,3940],[35,3950],[35,3960],[35,3970],[35,3990],
    [35,4010],[35,4030],[35,4050],
    [36,4210],[36,4220],[36,4230],[36,4240],[36,4260],[36,4270],[36,4280],
    [36,4310],[36,4330],[36,4350],[36,4370],
    [37,4510],[37,4520],[37,4530],[37,4540],[37,4550],[37,4560],[37,4580],
    [37,4610],[37,4630],[37,4650],[37,4660],[37,4680],[37,4710],[37,4730],
    [37,4750],
    [41,4910],[41,4920],[41,4930],[41,4950],[41,4960],[41,4970],[41,4980],
    [41,5010],[41,5020],[41,5030],[41,5040],[41,5050],[41,5060],[41,5080],
    [41,5110],[41,5130],[41,5150],
    [42,5210],[42,5220],[42,5230],[42,5260],[42,5280],[42,5310],[42,5320],
    [42,5330],[42,5350],[42,5360],[42,5370],[42,5410],
    [43,5510],[43,5520],[43,5530],[43,5540],[43,5550],[43,5570],[43,5580],
    [43,5590],[43,5610],[43,5620],[43,5630],[43,5650],[43,5670],[43,5690],
    [44,5810],[44,5820],[44,5840],[44,5850],[44,5860],[44,5880],[44,5890],
    [44,5910],[44,5920],[44,5930],[44,5950],[44,5960],[44,5970],[44,5980],
    [44,5990],[44,6010],[44,6020],[44,6030],
    [45,6110],[45,6140],[45,6170],[45,6210],[45,6230],[45,6240],[45,6330],
    [46,6410],[46,6420],
    [50,6530],
    [51,6510],[51,6550],[51,6560],[51,6570],[51,6580],[51,6590],[51,6610],
    [51,6620],[51,6630],[51,6650],[51,6670],[51,6690],[51,6710],[51,6730],
    [51,6750],[51,6770],[51,6790],[51,6810],[51,6840],[51,6870],
    [52,7010],[52,7020],[52,7030],[52,7050],[52,7070],[52,7090],[52,7110],
    [52,7130],[52,7150],
    [53,7310],[53,7340],[53,7360],[53,7380],[53,7410],[53,7430],[53,7450],
    [53,7470],[53,7490],[53,7510],[53,7530],[53,7540],[53,7550],[53,7560],
    [53,7570],[53,7580],
    [54,7700],[54,7720],[54,7730],[54,7740],[54,7750],[54,7760],[54,7770],
    [54,7790],[54,7810],[54,7830],
    [61,7800],[61,7910],[61,7920],[61,7930],[61,7950],[61,7970],[61,7990],
    [61,8010],[61,8030],[61,8040],[61,8060],
    [62,8210],[62,8220],[62,8230],[62,8240],[62,8250],[62,8260],[62,8270],
    [62,8280],[62,8290],[62,8310],[62,8330],[62,8340],[62,8360],[62,8380],
    [63,8510],[63,8520],[63,8540],[63,8550],[63,8560],[63,8570],[63,8580],
    [63,8590],
    [64,8710],[64,8720],
    [65,8810],[65,8820],[65,8830],[65,8840],[65,8850],[65,8870],[65,8880],
    [65,8910],[65,8930],[65,8940],[65,8960],[65,8980],[65,9010],[65,9020]]
var nCities=cities.length;
```

## 构建循环

由于解析返回结果的页码需要用到正则，里面一大堆转义符，如果通过js来拼接脚本源文本，会让难度雪上加霜。所以我决定把最难的脚本单元提出来，最后拼到主脚本里。

### 获取返回结果的页码

点了查询之后，结果是分页显示的：

```text
首页    上一页  1 2 3 下一页    末页
```

所以只要探一下下一页和末页的链接有没有绑`onclick`事件，就知道是不是有效。

- 如果搜出结果，“下一页”能点，说明不止一页，那么就可以往下点开页面，直到末页。
- 如果点不了，就不用费劲循环了。

于是写了个脚本getNextLastPage.iim，用`SEARCH`正则表达式的方法从页面源代码里提取js绑定。

```text
VERSION BUILD=844
SEARCH SOURCE=REGEXP:"<a href=\"javascript:;\" onclick=\"goPage\\('(\\d+)'\\)\"[^>]*>下一页" EXTRACT="$1"
SEARCH SOURCE=REGEXP:"<a href=\"javascript:;\" onclick=\"goPage\\('(\\d+)'\\)\"[^>]*>末页" EXTRACT="$1"
```

imacros提供了一个`iimGetLastExtract`的接口，能把iim脚本提取的结果以数组形式拿出来交给javascript或者vbs。

### 获得控件标签键

脚本交互时用到的是控件值，所以js工作脚本里只提供了100、101这类值列表。但我希望最终下载的csv以“银行_省份_城市_页码”，而不是“银行value_省份value_城市value_页码”的纯编码方式命名，所以得再写个getBankProvCity.iim，从三个下拉框控件属性标签里把“北京”、“天津”这些键取回来。

```text
VERSION BUILD=844
Tab T=1

TAG POS=1 TYPE=SELECT ATTR=ID:banksite EXTRACT=TXT
SET !VAR1 {{!EXTRACT}}
SET !EXTRACT NULL
SEARCH SOURCE=REGEXP:"<option value=\"{{!VAR1}}\">([^\"<]+)<" EXTRACT="$1"
SET !VAR2 {{!EXTRACT}}
SET !EXTRACT NULL

TAG POS=1 TYPE=SELECT ATTR=ID:province EXTRACT=TXT
SET !VAR1 {{!EXTRACT}}
SET !EXTRACT NULL
SEARCH SOURCE=REGEXP:"<option value=\"{{!VAR1}}\">([^\"<]+)<" EXTRACT="$1"
SET !VAR3 {{!EXTRACT}}
SET !EXTRACT NULL

TAG POS=1 TYPE=SELECT ATTR=ID:city EXTRACT=TXT
SET !VAR1 {{!EXTRACT}}
SET !EXTRACT NULL
SEARCH SOURCE=REGEXP:"<option value=\"{{!VAR1}}\">([^\"<]+)<" EXTRACT="$1"
SET !VAR4 {{!EXTRACT}}

TAG POS=1 TYPE=SELECT ATTR=ID:city CONTENT=%{{!VAR1}}

SET !EXTRACT {{!VAR2}}
ADD !EXTRACT {{!VAR3}}
ADD !EXTRACT {{!VAR4}}
```

事后想想，直接在预处理时把键和值都解析出来写进js里岂不是最方便。真是智商捉急。

### 功能集成

写好两个工作脚本，就可以着手搭集成脚本了。

```javascript
//上面已述。var banks和cities的定义
//主工作脚本：
for (var i=0; i < nBanks; i++){  // 遍历所有银行
    for (var j=0; j < nCities; j++){  //遍历所有省份+城市
        /* 拼接iim脚本，分别点击银行、省、市下拉框，点查询
        注意wait seconds，不能太短，否则页面来不及反应*/
        var m = "CODE:";
        m += "VERSION BUILD=844   " + "\n";
        m += "TAG POS=1 TYPE=SELECT ATTR=ID:banksite CONTENT=%"
            + banks[i] + "\n";
        m += "TAG POS=1 TYPE=SELECT ATTR=ID:province CONTENT=%"
            + cities[j][0] + "\n";
        m += "WAIT SECONDS=3" + "\n";
        m += "TAG POS=1 TYPE=SELECT ATTR=ID:city CONTENT=%"
            + cities[j][1] + "\n";
        m += "TAG POS=1 TYPE=A ATTR=ID:search" + "\n";
        m += "WAIT SECONDS=4";
        iimPlay(m);  //执行
        // 执行外部iim脚本，获取最末一页的页码
        iimPlay("getNextLastPage.iim");
        var nextPge = Number(iimGetLastExtract(1));
        var lastPge = Number(iimGetLastExtract(2));
        // 执行另一个iim脚本，获取省份和城市的取值，用来给文件起名
        iimPlay("getBankProvCity.iim");
        var bankName = iimGetLastExtract(1);
        var provName = iimGetLastExtract(2);
        var cityName = iimGetLastExtract(3);
        /* 如果有下一页，则继续点下一页链接，知道最后一页，
        并将表格内容存入‘银行_省_市_页码_时间’.csv*/
        if (nextPge > 0) {
            for (var k=1; k <= lastPge; k++){
                m = "CODE:VERSION BUILD=844          " + "\n";
                m += "TAG POS=1 TYPE=A ATTR=TXT:" + k + "\n";
                m += "TAG POS=2 TYPE=TABLE ATTR=TXT:* EXTRACT=TXT" + "\n";
                m += "SAVEAS TYPE=EXTRACT FOLDER=* FILE=" + bankName + "_"
                    + provName + "_" + cityName + "_" + k
                    + "_{{!NOW:yymmddhhnnss}}.csv" + "\n";
                m += "WAIT SECONDS=1";
                iimPlay(m);   //执行
            }
        }
    }
}
```

说时容易做时难。因为技术三脚猫，调试这些玩意儿花了很多时间。最坑爹的环节就是正则转义。

### 运行

工程搭好后，把脚本放在虚拟机上跑了三天，幸运的是网站似乎没有反爬虫策略，夜间也不关服务，于是让我顺顺利利地跑完了。

看到上千个文件静静躺在文件夹里，感觉到了巅峰愉悦。

<!-- {% raw %} -->
{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2017/1128/cnpas_tbl.png" title="图 | 爬下来的csv文件" %}}
<!-- {% endraw %} -->

里面的文件纷纷长这样：

<!-- {% raw %} -->
{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2017/1128/cnaps_raw.png" title="图 | csv的原始面目" %}}
<!-- {% endraw %} -->

### R合并csv

为啥之前费尽周章地格式化命名下载结果呢？因为我需要把文件名里的信息提取出来，合并到数据集里。下面这个R脚本定义了一个`parseCsvTbl`函数，就负责把csv文件名里的银行、省份、城市解析出来合并到数据集中。

```r
# -------载库，配环境----------
Sys.setlocale("LC_CTYPE", "Chs")
library(readr)
library(parallel)
library(compiler)
library(magrittr)
# -------读数据---------------
files <- list.files(".")
## 剔除.zip和.bat
files <- files[!grepl("zip$|bat$", files)]
## 某些文件莫名奇妙没有拿到后缀名，强制重命名
if (!all(grepl("csv$", files))){
  file.rename(files[!grepl("csv$", files)],
            paste0(files[!grepl("csv$", files)], ".csv"))
}
## 文件名全名显示
files <- list.files(getwd(), pattern="\\.csv$", full.names=FALSE)
stopifnot(all(grepl("csv$", files)))
message("A total of ", length(files), " files read.")
# ---------------自定义函数-----------------
parseCsvTbl <- cmpfun(function(
    csvfile, header=c("CNAPS", "Bank", "Tel", "Addr")){
    # for parallel use, duplicate the global settings
    Sys.setlocale("LC_CTYPE", "Chs")
    stopifnot(length(header) == 4)
    stopifnot(file.exists(csvfile))
    dat <- readr::read_csv(csvfile, col_types="nccc")
    names(dat) <- header
    dat$bank <- sub("^([^_]*?)_([^_]*?)_([^_]*?)_.+$", "\\1", csvfile)
    dat$prov <- sub("^([^_]*?)_([^_]*?)_([^_]*?)_.+$", "\\2", csvfile)
    dat$city <- sub("^([^_]*?)_([^_]*?)_([^_]*?)_.+$", "\\3", csvfile)
    return(dat)
})
# --------------调用解析函数，解析csv并合并---------------------
## 如果文件超过1000个，调用并行计算
if (length(files) > 1000){
    cl <- makeCluster(detectCores())
    df <- do.call("rbind", parLapply(cl, files, parseCsvTbl))
    stopCluster(cl)
}else{
    df <- do.call("rbind", lapply(files, parseCsvTbl))
}
## 合并后的结果输出到一个大csv
write_csv(df, paste0("cnaps_", format(Sys.time(), "%Y%m%d%H%M%S"), ".csv"))
message("output cnaps csv is generated.")
```

跑完脚本，数据变成了：

<!-- {% raw %} -->
{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2017/1128/cnpas.png" title="图 | 成品数据效果" %}}
<!-- {% endraw %} -->

一共40多万条记录。谢天谢地，老泪纵横。

## 但是！

你以为这是一篇萌蠢拙计、热情洋溢的imacros软文？那你错了。我只是为了说明一件事：

> 为了偷懒，我勤奋到废寝忘食。

但至于imacros，我绝不推荐。这是一段技术弯路，所有有志于自动化的，应该直接去学Python和Selenium，不要给无良付费软件一毛线的活路。

首先，今年的11月份，Firefox发布了新版Quantum，运行速度快了很多，但是API全变了。一众网红扩展纷纷蒙圈，统统跑不起来了。其中就包括了imacros。官网很快发了一则通告，大意是“用不了啦，大家赶紧降级Firefox。要不给Mozilla写个万民伞，让他们重新把废掉的那些API弄回来吧。”

开历史倒车还开出情怀来了。

随即，就在前两天，这家“全球最流行的浏览器自动化解决方案”提供者，升级了imacros浏览器插件，样子少许好看了点。然而免费版的功能却从猛犸缩水到野猪。

- 免费版最多只能录50个动作
- 免费版不再能读取本地文件
- 免费版不能回放iim脚本
- 只有IE版插件能用csv作输入源，不能超过100行、3列
...

要达到以往免费版的功能，需要升级到个人版。价钱比企业版可便宜多了，大约只需要……700大洋吧。

瞧这点出息，难怪欧洲软件企业而今越来越排不上号。市占率套现的玩法千千万，它却选择直接杀鸡取卵。这应该是我第一次，也是最后一次吹imacros。写完此文，我就把imacros彻底卸了。

工具没那么重要，重要的是理念。

Save your life. Automate everything.

[完]

----

<!-- {% raw %} -->
{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/QRcode.jpg" width="30%" title="扫码关注我的公众号" alt="扫码关注" %}}
<!-- {% endraw %} -->
