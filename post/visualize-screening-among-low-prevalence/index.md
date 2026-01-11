
{{% admonition abstract 引子 %}}

在低流行人群中做普遍筛查，整体效益很低。在教科书里，这只是干巴巴一句结论。形象地理解，在治安很好的地方搞拉网式排查，一定抓不到几个贼，抓到的也多半是冤枉的，综合下来看，不合算。正好今天碰到一个例子，借此把这些关系具象地量化出来。

{{% admonition question 问题是这样的 %}}

假设女性乳腺癌患病率为1‰，再假设乳腺癌检测敏感性是 90%（即100个真病人里，有90个会检出为阳性），假阳性率是 9%（即100个健康人去查，有9个会被误诊为阳性），那么一位女性检测呈阳性，她患乳腺癌的概率是多大？

{{% /admonition %}}

{{% /admonition %}}

## 1 

用下表很容易算出来。1‰的患病率，那么假定有100个病人，对应有99900个健康人。这100个病人里，90个被检出为阳性，10个漏诊；而余下99900个健康人里，有8991个（9%）被误诊为阳性。最后合计9081个阳性，里面真正的病人还是90个（只占0.99%）。这个0.99%，术语叫阳性预测值（PPV），在机器学习里叫精确度或查准率（precision）。

| | D+(有病) | D-(没病) | 合计 |
|:---|---:|----:| ---: |
|T+(阳性) | **90** | 8991 | **9081** | 
|T-(阴性) | 10 | 90909 | 90919 | 
|合计 | 100 | 99900 | 100000 | 

<!--more-->

90%灵敏度（真阳性率）和91%特异度（1-假阳性率），算是性能相当不错的筛查方法了，但最后的阳性检出者里，99%以上都是虚惊一场。因为整个人群的患病率太低了。这非常直观地印证了一开始的结论。

## 2

进一步扩展，如果真阳性率和假阳性率一起变化，阳性预测值会怎么办？

用下面的贝叶斯公式可以很简明地算出上面的结果。

$$ 后验概率 = \frac{ 先验概率 \times 似然度 }{标准化常量} $$

那么

$$ Pr(患癌 | 阳性) = \frac{ Pr(阳性 | 患癌) \times Pr(患癌) }{ Pr(阳性) } $$

Pr(阳性)又可以拆成 $Pr(阳性 | 患癌) \times Pr(患癌) + Pr(阳性 | 没患癌) \times Pr(没患癌)$。这些数都是现成的：$Pr(阳性 | 患癌)$ 就是灵敏度，$Pr(患癌)$ 就是患病率，$Pr(阳性 | 没患癌)$ 就是假阳性率。

假定患病率1‰不变，由低到高设定一系列真阳性率（P(T | D) ）和假阳性率（P(T | D') ）指标，在Excel里就能跑出一个模拟运算表来。你会发现，假阳性率对PPV影响非常大，只要升到5‰，无论灵敏度多高，PPV都好不到哪里去。一种灵敏度99%、特异度99.5%的超级筛查方法，都只能得到16.5%的PPV，信誓旦旦查出100个阳性，结果80%以上都是误诊。

{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2024/05/stimulate-tp-fp.jpg" width="100%" title="图 | 真阳性率和假阳性率模拟运算" %}}

## 3

再扩展一步，真阳性、假阳性对PPV的影响，在不同患病率水平下有什么不同？假设在5类人群里开展筛查，他们的患病率分别是1‰、5‰、1%、5%、10%。同样用Bayes公式跑模拟（代码如下），用pyecharts（matplotlib也行）将5层结果画在一张图里。

```python
import numpy as np
import matplotlib.pyplot as plt
from mpl toolkits.mplot3d import Axes3D
import pyecharts.options as opts
from pyecharts, charts import Surface3D
plt.rcParams['font,sans serif'] = u'Hei'
plt.rcParams['axes.unicode minus'] = False

# set x, y
x, y = np.meshgrid(np.arange(0.005,1,0.01),
                   np.arange(0.005,1,0.01))

# scenarios
z1 = (0.001 * x) / (0.001 * x + (1-0.001) * y)
z2 = (0.005 * x) / (0.005 * x + (1-0.005) * y)
z3 = (0.01 * x) / (0.01 * x + (1-0.01) * y)
z4 = (0.05 * x) / (0.05 * x + (1-0.05) * y)
z5 = (0.1 * x) / (0.1 * x + (1-0.1) * y)

# calc
z_data = [{'name':'患病率 1‰', 'z': z1},
          {'name':'患病率 5‰', 'z': z2},
          {'name':'患病率 1%'，'z': z3},
          {'name':'患病率 5%'，'z': z4},
          {'name':'患病率 10%'，'z': z5}]
# base plot
surface3d=(
  Surface3D(init_opts=opts.InitOpts(theme='white')),
  .set_global_opts(
    title_opts=opts.TitleOpts(
      title='不同患病率水平下，真阳性率、假阳性率和阳性预测值的关系'),
    visualmap_opts=opts.VisualMapOpts(
      is_show=True, dimension=2, max=1, min=0,
      range_color=['#bc4800','#fec904','#598901']),
    legend_opts=opts.LegendOpts(pos_right='5%', orient='vertical')
  )
)
# add layers iteratively
for _z in z_data:
  surface3d = surface3d.add(
    series_name= z['name'],
    data=list(zip(x.flatten(), y.flatten(), _z['z'].flatten())),
    xaxis3d_opts=opts.Axis3DOpts(
      type_='value'，name='真阳性率(灵敏度， Recall)')，
    yaxis3d_opts=opts.Axis3DOpts(
      type ='value'，name='假阳性率 (1-特异度)'),
    zaxis3d_opts=opts.Axis3DOpts(name='阳性预测值(Precision)'),
    grid3d_opts=opts.Grid3DOpts(width=100, height=100, depth=100),
    wire_frame_line_style_opts=opts.lineStyleOpts(
      opacity=0.1, width=0.1),
  )
```

把这张echarts 3D图旋转一下角度，能看得更清楚。x、y轴分别是真阳性率和假阳性率，z轴是PPV。你看到了什么？5个坡度不同的断崖，但都无一例外地向数值空间的零点陡峭滑落。最上一层代表10%患病率人群，最下一层是1‰患病率的人群——几乎已经贴到了墙壁上，也就是前面的模拟运算表在三维空间中的表现。

{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2024/05/tp-fp-ppv.jpg" width="100%" title="图 | 不同患病率水平下，真阳性率、假阳性率和阳性预测值的关系" %}}

{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/2024/05/tp-fp-ppv-rev.jpg" width="100%" title="图 | 不同患病率水平下，真阳性率、假阳性率和阳性预测值的关系(换个角度)" %}}

从这两张图，可以直观地看到PPV何等娇气。即使患病率高到10%，假阳性率也不能高于10%，真阳性率不能低于80%，否则PPV就会陡降到50%以下。

## 4

现在再回到最开始的结论，体验会更立体。吹得花好稻好，但特异度90%都不到的方法，基本就是扰民闹剧。而跑到患病率连1%都不到的低流行人群里拉网普查，基本也是扰民闹剧。推广而言，任何在低流行水平的不均衡样本空间里，用代理方法代替金标准的判别活动，都会面临以上两难、三难问题。

[完]

---

<!-- {% raw %} -->
{{% figure class="center" src="https://gh-1251443721.cos.ap-chengdu.myqcloud.com/QRcode.jpg" width="30%" title="扫码关注我的公众号" alt="扫码关注" %}}
<!-- {% endraw %} -->
